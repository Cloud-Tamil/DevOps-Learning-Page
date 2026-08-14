# Dockerfile Guide

A practical guide to understanding Dockerfiles — what they are, how each instruction works, why they matter, and how to use them well in production.

## Table of Contents

- [What is a Dockerfile?](#what-is-a-dockerfile)
- [Instruction Breakdown](#instruction-breakdown)
- [Benefits of a Dockerfile](#benefits-of-a-dockerfile)
- [Dockerfile in a Production Environment](#dockerfile-in-a-production-environment)
- [Best Practices](#best-practices)

---

## What is a Dockerfile?

A Dockerfile is a text file containing instructions that Docker uses to build a Docker image.

Think of it like this:

```
Dockerfile → Docker Image → Docker Container
```

### Example

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

Build the image:

```bash
docker build -t myapp:v1 .
```

Docker reads the Dockerfile and creates an image called `myapp:v1`.

Run it:

```bash
docker run -d -p 3000:3000 myapp:v1
```

---

## Instruction Breakdown

### `FROM`

```dockerfile
FROM node:20-alpine
```

Defines the base image. Here, we're using Node.js 20 on Alpine Linux.

### `WORKDIR`

```dockerfile
WORKDIR /app
```

Sets the working directory inside the container. Instead of running `cd /app` before every command, Docker automatically works from `/app`.

### `COPY`

```dockerfile
COPY package*.json ./
```

Copies files from the local application into the image.

```
Local machine
    |
    └── package.json
          ↓
Docker image
    |
    └── /app/package.json
```

### `RUN`

```dockerfile
RUN npm install
```

Executes a command during image creation — in this case, installing the application's dependencies.

### `COPY . .`

```dockerfile
COPY . .
```

Copies the application source code into the image.

### `EXPOSE`

```dockerfile
EXPOSE 3000
```

Documents that the application listens on port 3000.

> **Note:** `EXPOSE` itself does not publish the port to the host. Publishing happens with:
> ```bash
> docker run -p 3000:3000 myapp:v1
> ```

### `CMD`

```dockerfile
CMD ["npm", "start"]
```

Defines the default command that runs when the container starts.

---

## Benefits of a Dockerfile

### 1. Consistency

Without containers, an application might work on a developer's laptop but fail elsewhere.

```
Developer
    ↓
Dockerfile
    ↓
Docker Image
    ↓
Dev → Test → QA → Production
```

This reduces the classic **"it works on my machine"** problem.

### 2. Automation

Instead of manually installing Node.js, Java, Python, dependencies, system packages, and configuration, everything is defined in the Dockerfile. Then:

```bash
docker build
```

...automatically creates the environment.

### 3. Version Control

A Dockerfile can be stored in Git alongside application code:

```
GitHub
   |
   ├── Dockerfile
   ├── application code
   ├── requirements.txt
   └── README.md
```

Any change to the Dockerfile is tracked and reviewable.

### 4. Reproducibility

If production needs to be recreated months later, the same Dockerfile and image version reproduces the environment:

```
myapp:v1
myapp:v2
myapp:v3
```

Each version can be traced back to a specific source-code commit.

### 5. CI/CD Integration

A typical pipeline:

```
Developer
    ↓
GitHub
    ↓
CI Pipeline
    |
    ├── Unit Tests
    ├── Security Scan
    ├── Docker Build
    └── Docker Image Scan
    ↓
Container Registry
    ↓
Kubernetes / EKS / GKE / AKS
    ↓
Production
```

Example GitHub Actions steps:

```bash
docker build -t myapp:$VERSION .
docker push myregistry/myapp:$VERSION
```

Kubernetes then deploys that image.

---

## Dockerfile in a Production Environment

Consider an e-commerce application, **ShopSphere**, made up of multiple microservices:

```
ShopSphere
│
├── user-service
├── product-service
├── cart-service
├── order-service
├── payment-service
└── notification-service
```

Each service has its own Dockerfile:

```
user-service/
   ├── Dockerfile
   └── source code

product-service/
   ├── Dockerfile
   └── source code

order-service/
   ├── Dockerfile
   └── source code
```

Each Dockerfile produces a separate image:

```
user-service:v1
product-service:v1
order-service:v1
```

These images are pushed to a registry such as **Amazon ECR** or **Google Artifact Registry**, and Kubernetes deploys them:

```
                 Dockerfiles
                     |
                     ↓
                Docker Images
                     |
                     ↓
              Container Registry
                     |
                     ↓
                Kubernetes
              ┌──────┼──────┐
              ↓      ↓      ↓
           User    Order   Product
           Pods     Pods     Pods
```

---

## Best Practices

### Use a small base image

Instead of:

```dockerfile
FROM ubuntu
```

Prefer a minimal image:

```dockerfile
FROM node:20-alpine
```

Smaller images generally mean faster pulls and a smaller attack surface.

### Don't put secrets in the Dockerfile

❌ Avoid:

```dockerfile
ENV DB_PASSWORD=MyPassword123
```

✅ Use Kubernetes Secrets, AWS Secrets Manager, Azure Key Vault, or similar tools instead.

### Use a `.dockerignore` file

```
.git
node_modules
.env
*.log
```

This prevents unnecessary or sensitive files from being copied into the image.

### Use multi-stage builds

For compiled applications:

```dockerfile
FROM node:20 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
```

This keeps the final image lean by discarding build tools and intermediate files.
