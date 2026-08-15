# Dockerfile Examples: Simple vs Production-Ready Node.js

A walkthrough of two Node.js Dockerfiles — a simple one for learning/development, and an advanced, production-style multi-stage build — with an explanation of why the advanced version is better for real-world deployments.

## Table of Contents

- [1. Simple Dockerfile — Node.js Application](#1-simple-dockerfile--nodejs-application)
- [2. Advanced Application — Production-Style Node.js API](#2-advanced-application--production-style-nodejs-api)
- [3. .dockerignore](#3-dockerignore)
- [4. Advanced Multi-Stage Dockerfile](#4-advanced-multi-stage-dockerfile)
- [5. Why Is This Dockerfile Better?](#5-why-is-this-dockerfile-better)
- [6. Build the Advanced Image](#6-build-the-advanced-image)
- [7. The Complete DevOps Flow](#7-the-complete-devops-flow)
- [Simple vs Advanced Comparison](#simple-vs-advanced-comparison)

---

## 1. Simple Dockerfile — Node.js Application

Project structure:

```
simple-node-app/
├── app.js
├── package.json
└── Dockerfile
```

### `app.js`

```js
const http = require("http");

const PORT = 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hello from Docker!");
});

server.listen(PORT, () => {
  console.log(`Application running on port ${PORT}`);
});
```

### `package.json`

```json
{
  "name": "simple-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
```

### Simple Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

### Build and Run

```bash
docker build -t simple-node-app .
docker run -d -p 3000:3000 --name simple-app simple-node-app
```

Test at: [http://localhost:3000](http://localhost:3000)

### What Each Line Does

| Instruction | Purpose |
|---|---|
| `FROM` | Base image |
| `WORKDIR` | Working directory inside container |
| `COPY` | Copy files into container |
| `RUN` | Execute command while building image |
| `EXPOSE` | Documents application port |
| `CMD` | Command executed when container starts |

---

## 2. Advanced Application — Production-Style Node.js API

This is closer to what you'd discuss in a DevOps interview.

### Architecture

```
                    ┌─────────────────┐
                    │      GitHub     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    CI Pipeline  │
                    │                 │
                    │ Test            │
                    │ Security Scan   │
                    │ Docker Build    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Container Image │
                    │      ECR        │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │      EKS        │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │ Node API    │ │
                    │ │ Pod         │ │
                    │ └─────────────┘ │
                    │ ┌─────────────┐ │
                    │ │ Node API    │ │
                    │ │ Pod         │ │
                    │ └─────────────┘ │
                    └─────────────────┘
```

This application includes:

- Node.js + Express
- Production dependencies only
- Multi-stage Docker build
- Non-root user
- `.dockerignore`
- Health endpoint
- Environment variables
- Graceful shutdown
- Container health check

### Project Structure

```
advanced-node-api/
│
├── src/
│   └── server.js
│
├── package.json
├── package-lock.json
├── .dockerignore
└── Dockerfile
```

### `src/server.js`

```js
const express = require("express");

const app = express();

const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get("/", (req, res) => {
  res.json({
    message: "ShopSphere API is running",
    environment: process.env.NODE_ENV || "development"
  });
});

app.get("/health", (req, res) => {
  res.status(200).json({
    status: "UP"
  });
});

app.get("/api/products", (req, res) => {
  res.json([
    {
      id: 1,
      name: "Laptop",
      price: 75000
    },
    {
      id: 2,
      name: "Mobile",
      price: 30000
    }
  ]);
});

const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

process.on("SIGTERM", () => {
  console.log("SIGTERM received. Shutting down gracefully...");

  server.close(() => {
    console.log("HTTP server closed");
    process.exit(0);
  });
});
```

### `package.json`

```json
{
  "name": "shopsphere-api",
  "version": "1.0.0",
  "description": "Production-style ShopSphere API",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js"
  },
  "dependencies": {
    "express": "^4.21.2"
  }
}
```

Run:

```bash
npm install
```

This generates `package-lock.json`. **You should commit `package-lock.json` to Git.**

---

## 3. .dockerignore

This is important.

```
node_modules
npm-debug.log
.git
.gitignore
Dockerfile
.dockerignore
.env
.env.*
README.md
coverage
.vscode
```

**Why?** Without `.dockerignore`, Docker may unnecessarily copy `node_modules`, `.git`, `.env`, logs, and IDE files into the build context.

---

## 4. Advanced Multi-Stage Dockerfile

Here's the important part.

```dockerfile
# ==========================================
# Stage 1: Dependencies
# ==========================================

FROM node:20-alpine AS dependencies

WORKDIR /app

COPY package*.json ./

RUN npm ci


# ==========================================
# Stage 2: Production dependencies
# ==========================================

FROM node:20-alpine AS production-dependencies

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev


# ==========================================
# Stage 3: Final production image
# ==========================================

FROM node:20-alpine AS production

WORKDIR /app

ENV NODE_ENV=production

ENV PORT=3000

# Create non-root user
RUN addgroup -S nodeapp && \
    adduser -S nodeapp -G nodeapp

# Copy production dependencies
COPY --from=production-dependencies /app/node_modules ./node_modules

# Copy application
COPY src ./src

# Change ownership
RUN chown -R nodeapp:nodeapp /app

# Run as non-root user
USER nodeapp

EXPOSE 3000

# Container health check
HEALTHCHECK --interval=30s \
            --timeout=5s \
            --start-period=10s \
            --retries=3 \
            CMD wget --no-verbose \
                --tries=1 \
                --spider \
                http://localhost:3000/health || exit 1

CMD ["node", "src/server.js"]
```

---

## 5. Why Is This Dockerfile Better?

The simple Dockerfile:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

is good for learning and development. The advanced Dockerfile addresses production concerns.

### ① Multi-stage build

Instead of putting everything into one image:

```
Build/Dependencies
        ↓
Production Image
```

This keeps the final image cleaner and smaller.

### ② `npm ci`

Instead of `RUN npm install`, we use `RUN npm ci`.

`npm ci` is preferred for CI/CD because it installs from the lock file and provides reproducible dependency installation.

### ③ Production dependencies

The final image uses:

```dockerfile
RUN npm ci --omit=dev
```

Development dependencies aren't included in the production dependency layer.

### ④ Non-root container

This is very important.

❌ Bad:

```dockerfile
USER root
```

✅ Better:

```dockerfile
RUN addgroup -S nodeapp && \
    adduser -S nodeapp -G nodeapp

USER nodeapp
```

If an attacker manages to exploit the application, they don't automatically get root privileges inside the container.

### ⑤ Health check

We expose `/health`, and Docker checks `http://localhost:3000/health`.

This is particularly useful with orchestration platforms:

```
                    Kubernetes
                        │
                        ▼
                  ┌───────────┐
                  │    Pod    │
                  │           │
                  │ Node API  │
                  └─────┬─────┘
                        │
                   /health
                        │
                        ▼
                  Healthy?
                   /      \
                 YES       NO
                  │         │
                  ▼         ▼
              Continue    Restart/
                          remove traffic
```

---

## 6. Build the Advanced Image

From the project directory:

```bash
docker build -t shopsphere-api:1.0 .
```

Check the image:

```bash
docker images
```

Run it:

```bash
docker run -d \
  --name shopsphere-api \
  -p 3001:30001 \
  -e NODE_ENV=production \
  -e PORT=3001 \
  shopsphere-api:1.0
```

Check running containers:

```bash
docker ps
```

View logs:

```bash
docker logs shopsphere-api
```

Test the endpoints:

- App: [http://localhost:3000](http://localhost:3000)
- Health: [http://localhost:3000/health](http://localhost:3000/health)
- Products: [http://localhost:3000/api/products](http://localhost:3000/api/products)

---

## 7. The Complete DevOps Flow

For a ShopSphere-style project, this can be taken a step further:

```
Developer
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── npm test
    ├── npm audit
    ├── Docker build
    ├── Trivy scan
    │
    ▼
Docker Image
shopsphere-api:1.0
    │
    ▼
Amazon ECR
    │
    ▼
Amazon EKS
    │
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
    │
    ▼
Kubernetes Service
    │
    ▼
AWS ALB
    │
    ▼
Users
```

This is much closer to the kind of architecture you'd explain in a 3–6 year DevOps Engineer interview.

---

## Simple vs Advanced Comparison

| Feature | Simple Dockerfile | Production Dockerfile |
|---|---|---|
| Base image | Node Alpine | Node Alpine |
| WORKDIR | ✅ | ✅ |
| Dependency install | `npm install` | `npm ci` |
| Multi-stage | ❌ | ✅ |
| Production dependencies | ❌ | ✅ |
| Non-root user | ❌ | ✅ |
| Health check | ❌ | ✅ |
| Environment variables | Basic | ✅ |
| Graceful shutdown | ❌ | ✅ |
| CI/CD ready | Basic | ✅ |
| Kubernetes ready | Basic | ✅ |

### Interview Answer

> "For development, I can use a simple Dockerfile with a Node.js Alpine image, copy the package files, install dependencies, copy the source code, expose the port, and start the application. For production, I prefer a multi-stage build, `npm ci`, production-only dependencies, a non-root user, health checks, environment variables, proper signal handling, and a minimal final image. I then push the image to ECR and deploy it to EKS using Kubernetes manifests or Helm."

