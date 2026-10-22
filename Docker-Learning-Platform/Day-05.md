# ShopSphere: Day 5+ Real DevOps Workflows
### Docker → GitHub Actions (CI) → GHCR → Local Kubernetes (`kind`)

A practical, end-to-end DevOps lab building a real-world delivery pipeline for **ShopSphere**, a lightweight e-commerce frontend service.

---

## Architecture Overview

```text
+-----------------------+       git push        +---------------------------+
| Local Workspace       | --------------------> | GitHub Repository         |
| (Code + Manifests)    |                       | (shopsphere-cicd)         |
+-----------------------+                       +---------------------------+
                                                              |
                                                              v
                                                +---------------------------+
                                                | GitHub Actions (CI)       |
                                                | - Lint & file validation  |
                                                | - Multi-stage build       |
                                                | - Tagging with commit SHA |
                                                +---------------------------+
                                                              |
                                                              v (docker push)
                                                +---------------------------+
                                                | GitHub Container Registry |
                                                | (ghcr.io)                 |
                                                +---------------------------+
                                                              |
                                                              v (image pull)
+-------------------------------------------------------------+-------------+
| Local `kind` Kubernetes Cluster (2 nodes)                                 |
|                                                                           |
|   +-----------------------+           +-------------------------------+   |
|   | Service (ClusterIP)   | --------> | Pod 1 (ShopSphere container)  |   |
|   | Port 80               |           +-------------------------------+   |
|   +-----------------------+           | Pod 2 (ShopSphere container)  |   |
|                                       +-------------------------------+   |
+---------------------------------------------------------------------------+
```

> **Why this setup?** GitHub-hosted Actions runners cannot reach a private local machine without complex tunnels or self-hosted agents. We automate **Continuous Integration (CI)** and image publishing in GitHub Actions, and execute verified **Continuous Delivery (CD)** deployments to our local `kind` cluster.

---

## Prerequisites & Tooling Verification

Run the following commands to confirm your host engine is healthy:

```bash
docker --version
docker info
git --version
```

> **Warning:** If running in Google Cloud Shell or a constrained VM, ensure Docker Engine is active before continuing.

---

## Step 1: Install `kubectl` & `kind` (Debian / Ubuntu)

### 1.1 Install `kubectl`
```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates gettext-base

ARCH="$(dpkg --print-architecture)"
case "$ARCH" in
  amd64) KARCH=amd64 ;;
  arm64) KARCH=arm64 ;;
  *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac

curl -fsSLo /tmp/kubectl \
  "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/${KARCH}/kubectl"
sudo install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl

kubectl version --client
```

### 1.2 Install `kind`
```bash
ARCH="$(dpkg --print-architecture)"
case "$ARCH" in
  amd64) KARCH=amd64 ;;
  arm64) KARCH=arm64 ;;
  *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac

curl -fsSLo /tmp/kind \
  "https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-${KARCH}"
sudo install -o root -g root -m 0755 /tmp/kind /usr/local/bin/kind

kind version
```

---

## Step 2: Provision the Local `kind` Cluster

### 2.1 Cluster Definition (`kind-config.yaml`)
Create the cluster definition with 1 control-plane and 1 worker node:

```bash
mkdir -p ~/shopsphere-cicd/.github/workflows ~/shopsphere-cicd/k8s
cd ~/shopsphere-cicd

cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: shopsphere
nodes:
  - role: control-plane
  - role: worker
EOF
```

### 2.2 Create & Verify Cluster
```bash
kind create cluster --config kind-config.yaml --wait 180s

# Verification
kind get clusters
kubectl config current-context
kubectl get nodes -o wide
```
Ensure all nodes report `Ready` before proceeding.

---

## Step 3: Application Code & Local Container Build

### 3.1 Application Files
```bash
# index.html
cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ShopSphere</title>
</head>
<body>
  <h1>ShopSphere</h1>
  <p>Deployed using Docker, GitHub Actions, and Kubernetes.</p>
  <p>Release: Day 5+ CI/CD Lab</p>
</body>
</html>
EOF

# Dockerfile
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF

# .dockerignore
cat > .dockerignore <<'EOF'
.git
.github
k8s
kind-config.yaml
*.log
.env
EOF
```

### 3.2 Local Test Run
```bash
docker build -t shopsphere:local .
docker run -d --name shopsphere-local -p 8080:80 shopsphere:local

# Verify
curl -fsS http://localhost:8080 | grep "ShopSphere"

# Cleanup
docker rm -f shopsphere-local
```

---

## Step 4: Configure GitHub Actions CI/CD Pipeline

### 4.1 Workflow Definition (`.github/workflows/shopsphere.yml`)
```yaml
cat > .github/workflows/shopsphere.yml <<'EOF'
name: ShopSphere CI and Publish

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

permissions:
  contents: read

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Validate source files
        run: |
          test -s Dockerfile
          test -s index.html

      - name: Build test image
        run: docker build -t shopsphere:test .

      - name: Set up Docker Buildx
        if: github.event_name == 'push'
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Generate image tags
        if: github.event_name == 'push'
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,format=long
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push image
        if: github.event_name == 'push'
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
EOF
```

### 4.2 Push Code to GitHub
> **Note:** Set `YOUR_USERNAME` to your actual GitHub username (use lowercase).

```bash
git init
git add Dockerfile index.html .dockerignore kind-config.yaml k8s .github
git commit -m "feat: initial ShopSphere pipeline configuration"
git branch -M main

# Replace YOUR_USERNAME
git remote add origin https://github.com/YOUR_USERNAME/shopsphere-cicd.git
git push -u origin main
```

### 4.3 Configure Package Visibility
1. Open the GitHub repository in your browser.
2. Under **Actions**, confirm that the workflow ran and succeeded.
3. Open **Packages** on your GitHub profile or repository.
4. Go to **Package Settings** and set the visibility to **Public** (enables `kind` to pull without an `imagePullSecret`).

---

## Step 5: Kubernetes Manifests

Create `k8s/shopsphere.yaml` with production-grade probes and resource constraints:

```yaml
cat > k8s/shopsphere.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopsphere
  labels:
    app: shopsphere
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shopsphere
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: shopsphere
    spec:
      containers:
        - name: web
          image: ${IMAGE}
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: shopsphere
spec:
  type: ClusterIP
  selector:
    app: shopsphere
  ports:
    - name: http
      port: 80
      targetPort: http
EOF
```

---

## Step 6: Deploy & Verify

### 6.1 Render and Apply Manifest
```bash
# Substitute your actual GitHub username
export GITHUB_USER="your_username_here"
export IMAGE="ghcr.io/${GITHUB_USER}/shopsphere-cicd:sha-$(git rev-parse HEAD)"

# Validate substitution
envsubst < k8s/shopsphere.yaml > /tmp/shopsphere-rendered.yaml
kubectl apply --dry-run=client -f /tmp/shopsphere-rendered.yaml

# Deploy
kubectl apply -f /tmp/shopsphere-rendered.yaml
kubectl rollout status deployment/shopsphere --timeout=180s
```

### 6.2 Deployment Verification
```bash
kubectl get deployments
kubectl get pods -l app=shopsphere -o wide
kubectl get services shopsphere
kubectl get endpoints shopsphere
```

### 6.3 Test Traffic
In one terminal:
```bash
kubectl port-forward service/shopsphere 8080:80
```

In a second terminal:
```bash
curl -i http://localhost:8080
```

---

## Step 7: Operations — Scaling, Updates & Rollbacks

### 7.1 Scale Deployment
```bash
kubectl scale deployment/shopsphere --replicas=3
kubectl rollout status deployment/shopsphere
kubectl get pods -l app=shopsphere
```

### 7.2 Release an Update
```bash
# 1. Update source
sed -i 's/Day 5+ CI\/CD Lab/Day 5+ CI\/CD Lab - Release 2/' index.html

# 2. Commit and push
git add index.html
git commit -m "chore: update release version to v2"
git push origin main

# 3. Wait for GitHub Actions to complete, then deploy updated SHA
export IMAGE="ghcr.io/${GITHUB_USER}/shopsphere-cicd:sha-$(git rev-parse HEAD)"
envsubst < k8s/shopsphere.yaml > /tmp/shopsphere-rendered.yaml
kubectl apply -f /tmp/shopsphere-rendered.yaml
kubectl rollout status deployment/shopsphere --timeout=180s

# 4. Confirm image tag
kubectl get deployment shopsphere -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### 7.3 Execute a Rollback
```bash
# View deployment history
kubectl rollout history deployment/shopsphere

# Undo rollout to previous revision
kubectl rollout undo deployment/shopsphere
kubectl rollout status deployment/shopsphere

# Verify current revision
kubectl rollout history deployment/shopsphere
```

---

## Step 8: Troubleshooting Matrix

| Problem | Potential Root Cause | Diagnostic Command |
| :--- | :--- | :--- |
| `ImagePullBackOff` | Private GHCR repo / Incorrect tag / Typo in username | `kubectl describe pods -l app=shopsphere` |
| `CrashLoopBackOff` | Application crash or failed entrypoint | `kubectl logs deployment/shopsphere` |
| Nodes Not Ready | Docker resource exhaustion | `kubectl describe nodes` & `docker info` |
| Endpoints Missing | Selector mismatch between Service & Pod labels | `kubectl describe svc shopsphere` |
| Ingress / Port Forward Fails | Port binding collision on host | `netstat -tlpn \| grep 8080` or `lsof -i :8080` |

---

## Step 9: Teardown & Cleanup

```bash
# 1. Delete Kubernetes resources
kubectl delete -f /tmp/shopsphere-rendered.yaml

# 2. Destroy kind cluster
kind delete cluster --name shopsphere

# 3. Verify removal
kind get clusters
```
