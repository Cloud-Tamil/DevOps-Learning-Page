# Docker Day 4 — Production Image Practices

Transition your container operations from development basics to production-grade security, reliability, resource governance, and optimization.

This guide walks through hardening a practical Go web service (**ShopSphere**) using multi-stage builds, non-root user execution, Docker health checks, runtime security constraints, vulnerability scanning, and secret isolation.

---

## Table of Contents
1. [Core Learning Objectives](#core-learning-objectives)
2. [Lab 0: Verify Environment & BuildKit](#lab-0-verify-environment--buildkit)
3. [Section 1: Multi-Stage Builds](#1-multi-stage-builds)
4. [Section 2: Non-Root Execution & Hardening](#2-run-containers-as-non-root-users)
5. [Section 3: Container Health Checks](#3-configure-docker-health-checks)
6. [Section 4: Resource Limits (CPU & Memory)](#4-apply-resource-limits)
7. [Section 5: Vulnerability Scanning with Trivy](#5-scan-docker-images-for-vulnerabilities)
8. [Section 6: Secrets Management](#6-manage-secrets-securely)
9. [Section 7: Full Production Workflow](#7-combine-the-practices-into-one-production-style-workflow)
10. [Lab Cleanup](#lab-cleanup)
11. [Revision Checklist](#revision-checklist)
12. [Interview Preparation](#interview-questions-to-practice)

---

## Core Learning Objectives

| # | Topic | Objective |
|---|---|---|
| **01** | **Multi-stage builds** | Decouple compilation toolchains from the lean final runtime layer. |
| **02** | **Non-root users** | Execute application processes with least-privilege UID/GID. |
| **03** | **Health checks** | Distinguish between an active PID and an actual responsive service. |
| **04** | **Resource limits** | Guard hosts against starvation by capping CPU time and memory consumption. |
| **05** | **Image scanning** | Detect known CVEs across OS packages and application dependencies. |
| **06** | **Secrets management** | Prevent credentials from ever baking into static image layers or Git history. |

---

## Lab 0: Verify Environment & BuildKit

Run these checks to confirm the Docker engine daemon is responsive and that BuildKit is supported:

```bash
docker --version
docker compose version
docker info
```

Ensure BuildKit is ready:

```bash
docker buildx version
docker buildx ls
```

> **Note**: If using Google Cloud Shell or lightweight Linux VMs, check that cgroup-v2 support is active if applying strict memory/CPU caps.

---

## 1. Multi-Stage Builds

### The Problem with Single-Stage Builds
Compiling an application requires toolchains (compilers, build headers, dev dependencies, source files). Packaging all of that into the final container bloats the image by hundreds of megabytes and exponentially expands your attack surface.

### The Solution
A **multi-stage build** uses multiple `FROM` instructions:
```
┌─────────────────────────────────────────┐
│              Build Stage                │
│ (Source Code + Compiler + Dependencies) │
└────────────────────┬────────────────────┘
                     │ Copy binary only
                     ▼
┌─────────────────────────────────────────┐
│           Final Runtime Stage           │
│   (App Binary + Minimal Rootfs/Certs)   │
└─────────────────────────────────────────┘
```

### Lab 1: Setup ShopSphere Go Application

1. **Create the workspace:**
   ```bash
   mkdir -p ~/shopsphere-day4
   cd ~/shopsphere-day4
   ```

2. **Generate `main.go`:**
   ```bash
   cat > main.go <<'EOF'
   package main

   import (
   	"fmt"
   	"log"
   	"net/http"
   )

   func main() {
   	mux := http.NewServeMux()

   	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
   		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
   		fmt.Fprintln(w, "ShopSphere API is running")
   	})

   	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
   		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
   		fmt.Fprintln(w, "healthy")
   	})

   	log.Println("Listening on :8080")
   	log.Fatal(http.ListenAndServe(":8080", mux))
   }
   EOF
   ```

3. **Create the Multi-Stage `Dockerfile`:**
   ```bash
   cat > Dockerfile <<'EOF'
   # Stage 1: Build the application
   FROM golang:1.25-alpine AS builder

   WORKDIR /src
   COPY main.go .

   RUN CGO_ENABLED=0 GOOS=linux go build \
       -trimpath \
       -ldflags="-s -w" \
       -o /out/shopsphere main.go

   # Stage 2: Final runtime image
   FROM alpine:3.22

   RUN apk add --no-cache ca-certificates \
       && addgroup -S app \
       && adduser -S -G app app

   COPY --from=builder --chown=app:app \
       /out/shopsphere /usr/local/bin/shopsphere

   USER app:app

   EXPOSE 8080

   ENTRYPOINT ["/usr/local/bin/shopsphere"]
   EOF
   ```

4. **Add a `.dockerignore` file:**
   ```bash
   cat > .dockerignore <<'EOF'
   .git
   .env
   .env.*
   *.pem
   *.key
   secrets/
   tmp/
   coverage/
   EOF
   ```

5. **Build and inspect the image:**
   ```bash
   docker build --pull -t shopsphere-api:1.0 .

   # Inspect metadata and history
   docker image ls shopsphere-api
   docker history shopsphere-api:1.0

   # Confirm non-root default user
   docker image inspect --format '{{.Config.User}}' shopsphere-api:1.0
   # Expected output: app:app
   ```

---

## 2. Run Containers as Non-Root Users

By default, Docker executes processes inside containers as `root` (UID `0`). While isolated by namespaces, container breakouts or mount-sharing can lead to full host compromise. 

### Lab 2: Runtime User and Filesystem Verification

1. **Run the container:**
   ```bash
   docker run -d \
     --name shopsphere-secure \
     -p 127.0.0.1:8080:8080 \
     shopsphere-api:1.0
   ```

2. **Verify runtime UID & GID:**
   ```bash
   docker exec shopsphere-secure id
   # Output should indicate nonzero UID/GID for user 'app'
   ```

3. **Verify endpoints:**
   ```bash
   curl -i http://127.0.0.1:8080/
   curl -i http://127.0.0.1:8080/health
   ```

4. **Verify root filesystem restrictions:**
   ```bash
   docker exec shopsphere-secure sh -c 'touch /root/test-file'
   # Expected: Permission denied
   ```

### Comprehensive Security Hardening Flags
Recreate the container with strict kernel capabilities, read-only rootfs, and no privilege escalation:

```bash
docker rm -f shopsphere-secure

docker run -d \
  --name shopsphere-secure \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --memory 128m \
  --cpus 0.5 \
  -p 127.0.0.1:8080:8080 \
  shopsphere-api:1.0
```

Verify configured security controls:
```bash
docker inspect \
  --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}' \
  shopsphere-secure
```

---

## 3. Configure Docker Health Checks

A process running inside a container does not mean the application is serving traffic correctly (e.g., deadlocks, blocked event loops, broken DB connections).

1. **Update `Dockerfile` with the `HEALTHCHECK` directive:**
   Add this directly after `EXPOSE 8080`:

   ```dockerfile
   HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
     CMD wget -q -O /dev/null http://127.0.0.1:8080/health || exit 1
   ```

2. **Rebuild the image:**
   ```bash
   docker build -t shopsphere-api:1.0 .
   ```

3. **Relaunch container:**
   ```bash
   docker rm -f shopsphere-secure

   docker run -d \
     --name shopsphere-secure \
     --read-only \
     --tmpfs /tmp:rw,noexec,nosuid,size=16m \
     --cap-drop ALL \
     --security-opt no-new-privileges:true \
     --memory 128m \
     --cpus 0.5 \
     -p 127.0.0.1:8080:8080 \
     shopsphere-api:1.0
   ```

4. **Inspect health state:**
   ```bash
   docker inspect --format '{{.State.Health.Status}}' shopsphere-secure
   # Initially 'starting', transitions to 'healthy'

   # Review full health-check probe history
   docker inspect --format '{{json .State.Health}}' shopsphere-secure
   ```

---

## 4. Apply Resource Limits

Prevent "noisy neighbor" syndrome and system-wide Out-Of-Memory (OOM) crashes by enforcing hard CPU and memory thresholds:

```bash
docker run -d \
  --name resource-demo \
  --memory 128m \
  --cpus 0.5 \
  nginx:alpine
```

Monitor live usage and inspect settings:
```bash
docker stats --no-stream resource-demo

docker inspect \
  --format 'Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}' \
  resource-demo

docker rm -f resource-demo
```

### Production Guidelines
* Always base limits on performance and load testing, not guesses.
* Monitor memory closely: if a process exceeds its limit, the Linux kernel terminates it with an **OOM (Out of Memory) Kill** (`Exit Code 137`).

---

## 5. Scan Docker Images for Vulnerabilities

Scan layers for outdated OS libraries and vulnerable application dependencies prior to registry publishing.

### Lab 3: Scanning with Trivy

1. **Run a scan via installed CLI:**
   ```bash
   trivy image --scanners vuln shopsphere-api:1.0
   ```

2. **Alternative: Run Trivy via temporary container:**
   ```bash
   docker run --rm \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v trivy-cache:/root/.cache/ \
     aquasec/trivy:latest \
     image --scanners vuln shopsphere-api:1.0
   ```

### Severity Triage Reference
| Severity | Policy Response |
|---|---|
| **CRITICAL** | Block deployment pipeline; patch immediately. |
| **HIGH** | Prioritize remediation before release window. |
| **MEDIUM** | Evaluate exploitability; schedule within sprint cycle. |
| **LOW / UNKNOWN** | Track and address during standard patch maintenance. |

### Enforcing CI/CD Failure Gates
Return a non-zero exit code if vulnerabilities above your threshold are detected:
```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  shopsphere-api:1.0
```

---

## 6. Manage Secrets Securely

> **Critical Rule**: Never embed credentials in `ENV`, `ARG`, or via static `COPY` instructions. Even if removed in subsequent layers, static layers are immutable and extractable.

### Pattern A: BuildKit Secret Mounts (Build-Time Only)
Ideal for private package registry tokens or build-time credentials that must not exist in any layer.

1. **Create `Dockerfile.build-secret`:**
   ```bash
   cat > Dockerfile.build-secret <<'EOF'
   # syntax=docker/dockerfile:1
   FROM alpine:3.22

   RUN --mount=type=secret,id=build_token \
       test -s /run/secrets/build_token

   CMD ["sh", "-c", "echo Application started"]
   EOF
   ```

2. **Build using the secret mount:**
   ```bash
   umask 077
   printf '%s' 'training-only-token' > /tmp/shopsphere-build-token

   docker build \
     -f Dockerfile.build-secret \
     --secret id=build_token,src=/tmp/shopsphere-build-token \
     -t shopsphere-secret-demo:1.0 .

   rm -f /tmp/shopsphere-build-token
   ```

### Pattern B: Docker Compose Secrets (Local Runtime)
Injects secrets as in-memory files at `/run/secrets/<name>` without passing secrets via environment variables.

1. **Create the secret file and compose manifest:**
   ```bash
   mkdir -p secrets
   umask 077
   printf '%s' 'training-only-password' > secrets/db_password.txt

   cat > compose.secrets.yaml <<'EOF'
   services:
     secret-demo:
       image: alpine:3.22
       command: ["sh", "-c", "test -r /run/secrets/db_password && echo Secret file mounted"]
       secrets:
         - db_password

   secrets:
     db_password:
       file: ./secrets/db_password.txt
   EOF
   ```

2. **Execute and test:**
   ```bash
   docker compose -f compose.secrets.yaml config
   docker compose -f compose.secrets.yaml run --rm secret-demo
   # Expected output: Secret file mounted

   rm -f secrets/db_password.txt
   ```

---

## 7. Combine the Practices into One Production-Style Workflow

Run the complete, hardened ShopSphere container:

```bash
cd ~/shopsphere-day4

# 1. Clean previous containers
docker rm -f shopsphere-secure 2>/dev/null || true

# 2. Launch hardened container
docker run -d \
  --name shopsphere-secure \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --memory 128m \
  --cpus 0.5 \
  --restart unless-stopped \
  -p 127.0.0.1:8080:8080 \
  shopsphere-api:1.0

# 3. Validate container configuration
docker inspect --format \
'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}' \
shopsphere-secure

# 4. Verify endpoints and health check status
curl -i http://127.0.0.1:8080/health

docker inspect --format \
'{{if .State.Health}}{{.State.Health.Status}}{{else}}no health check{{end}}' \
shopsphere-secure
```

---

## Lab Cleanup

```bash
docker rm -f shopsphere-secure resource-demo 2>/dev/null || true
# (Optional) Remove generated images
# docker rmi shopsphere-api:1.0 shopsphere-secret-demo:1.0
```

---

## Revision Checklist

- [ ] Built an optimized image using multiple stages
- [ ] Confirmed process runs as a non-root user (`app:app`)
- [ ] Added and validated a container `HEALTHCHECK`
- [ ] Configured hard CPU (`0.5`) and memory limits (`128m`)
- [ ] Scanned the final image with Trivy and handled severity thresholds
- [ ] Mounted a build secret with BuildKit (`--mount=type=secret`)
- [ ] Mounted runtime secrets using Docker Compose
- [ ] Ran container with `--read-only`, `--cap-drop ALL`, and `no-new-privileges`

---

## Interview Questions to Practice

1. **What is a multi-stage Docker build, and why does it reduce the final image size?**
   * *Answer:* Multi-stage builds utilize several `FROM` directives in a single `Dockerfile`. Each stage represents a discrete environment. Intermediate compilation artifacts, SDKs, and build utilities remain in earlier stages, allowing only the final static binary or runtime files to be copied into a lean base image (e.g., Alpine or scratch).

2. **What is the difference between a Dockerfile's `USER` instruction and the `--user` runtime option?**
   * *Answer:* `USER` sets the default UID/GID metadata directly within the image configuration. `--user` is a CLI runtime parameter that overrides the image's default UID/GID during container creation (`docker run`).

3. **What is the difference between a running container and a healthy container?**
   * *Answer:* A running container simply means the initial process (`PID 1`) is active from the Linux kernel's perspective. A healthy container means the user-defined `HEALTHCHECK` probe (such as an HTTP check or internal script) is executing successfully and the application is functionally operational.

4. **What happens when a container exceeds its memory limit?**
   * *Answer:* When a container attempts to allocate memory beyond its cgroup boundary (`--memory`), the Linux kernel triggers the Out-Of-Memory (OOM) killer. The main process inside the container is terminated with `Exit Code 137`.

5. **What is the difference between image scanning and runtime security monitoring?**
   * *Answer:* Image scanning is a static analysis step (shift-left) that parses package manifests and binary hashes to detect *known* CVEs before deployment. Runtime monitoring (e.g., Falco) tracks active kernel syscalls, process behaviors, and anomalous network connections while the container is executing.

6. **Why is deleting a secret in a later Dockerfile layer not sufficient?**
   * *Answer:* Docker images use an append-only, union file system. Each layer stores the delta from the layer before it. Deleting a file in a later layer merely writes a "whiteout" record masking the file, but the secret remains permanently readable in the underlying layer history.

7. **How do BuildKit secret mounts differ from Docker Compose secrets?**
   * *Answer:* BuildKit secret mounts (`RUN --mount=type=secret`) temporarily expose credentials *only* during a specific image build step without saving the secret to any filesystem layer. Compose secrets inject secrets into *running containers* as in-memory files at runtime (`/run/secrets/`).

8. **Why should a production image avoid unnecessary packages, capabilities, and root privileges?**
   * *Answer:* Applying least privilege limits the tools available to an attacker if an exploit occurs. Removing shells, package managers, root privileges, and default Linux kernel capabilities minimizes the blast radius and prevents privilege escalation to the host.
