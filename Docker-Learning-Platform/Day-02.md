# 🚀 Docker Day 2: Image Building & Debugging Mastery

[![Docker](https://img.shields.io/badge/Docker-24.0%2B-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![BuildKit](https://img.shields.io/badge/BuildKit-Enabled-blue?logo=docker&logoColor=white)](https://docs.docker.com/build/buildkit/)
[![Level](https://img.shields.io/badge/Level-Intermediate-orange)](#)
[![Track](https://img.shields.io/badge/Track-DevOps%20%26%20Cloud%20Engineering-orange)](#)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](#)

> **Day 2 Core Philosophy:** An image is not a monolith; it is an immutable stack of read-only cached layers. Mastering image construction means optimizing build contexts, ordering directives for maximal cache reuse, separating build-time arguments (`ARG`) from runtime configuration (`ENV`), and wielding log telemetry and exit codes to diagnose container failures in seconds.

---

## 📑 Table of Contents

1. [Understanding Image Layers & Union Filesystems](#1-understanding-image-layers--union-filesystems)
2. [The Build Context & Docker Daemon Transfer](#2-the-build-context--docker-daemon-transfer)
3. [Mastering `.dockerignore`](#3-mastering-dockerignore)
4. [Docker Layer Caching & Directive Ordering](#4-docker-layer-caching--directive-ordering)
5. [Build Arguments (`ARG`) vs. Environment Variables (`ENV`)](#5-build-arguments-arg-vs-environment-variables-env)
6. [Container Logging Architecture & Telemetry](#6-container-logging-architecture--telemetry)
7. [Systematic Troubleshooting & Debugging Playbook](#7-systematic-troubleshooting--debugging-playbook)
8. [Hands-On Labs for Day 2](#8-hands-on-labs-for-day-2)
9. [Day 2 Production Best Practices](#9-day-2-production-best-practices)
10. [Day 2 Command Cheat Sheet](#10-day-2-command-cheat-sheet)
11. [Day 2 Revision Checklist](#11-day-2-revision-checklist)
12. [Test Yourself — Day 2 Quiz](#12-test-yourself--day-2-quiz)
13. [Looking Ahead: Day 3 Preview](#13-looking-ahead-day-3-preview)

---

## 1. Understanding Image Layers & Union Filesystems

A Docker image is an ordered stack of read-only filesystem layers unified by a storage driver such as `overlay2`. Each instruction in a `Dockerfile` that modifies files (`RUN`, `COPY`, `ADD`) creates a new layer referencing its parent.

```text
+-----------------------------------------------------------+
|      CONTAINER WRITABLE LAYER (Read-Write: ephemeral)     |  <-- Upperdir: writes, temp files, logs
+-----------------------------------------------------------+
|  Layer 4: CMD ["node", "server.js"]      (Metadata / 0 B) |  \
|  Layer 3: COPY . /app                    (~12 MB)         |   |  DOCKER IMAGE
|  Layer 2: RUN npm ci --omit=dev          (~48 MB)         |   |  (Lowerdir: Read-Only Layers)
|  Layer 1: COPY package*.json /app/       (~4 KB)          |   |  Shared across containers
|  Layer 0: FROM node:22-alpine            (~140 MB)        |  /
+-----------------------------------------------------------+
```

### The Copy-on-Write (CoW) Strategy
- Base image layers remain strictly read-only and immutable.
- When a container modifies an inherited file, the storage driver copies the file up into the private writable layer (`upperdir`) before writing changes.
- Deleted files in the container layer are masked using whiteout markers; the underlying image layer size remains unchanged on disk.

---

## 2. The Build Context & Docker Daemon Transfer

When invoking `docker build -t app:1.0 .`, the final argument (`.`) designates the **build context**.

```
[Client CLI] --- (Tar Archive of Context Path) ---> [Docker Engine Daemon / BuildKit]
```

- The client bundles every file in the directory (unless excluded) and sends it over IPC or TCP to the daemon before execution starts.
- **Common anti-patterns:** Running `docker build .` from `$HOME` or root project folders containing `.git`, local dependencies (`node_modules`, `.venv`), database dumps, and assets.
- Bloated contexts waste bandwidth, inflate build times, and bust disk caches.

---

## 3. Mastering `.dockerignore`

Place a `.dockerignore` file adjacent to your `Dockerfile` to drop files from the tar bundle sent to the engine:

```gitignore
# VCS & System Metadata
.git
.gitignore
.gitattributes
.DS_Store
Thumbs.db

# Dependencies & Build Artefacts
node_modules/
vendor/
__pycache__/
*.pyc
dist/
build/
target/

# Local Environment & Secrets
.env
.env.*
*.pem
*.key
*.crt
id_rsa*

# Telemetry & Logs
*.log
npm-debug.log*
yarn-debug.log*
coverage/
.nyc_output/
```

---

## 4. Docker Layer Caching & Directive Ordering

BuildKit evaluates instructions sequentially from top to bottom. If any instruction's checksum changes, its cache is invalidated along with **all subsequent downstream steps**.

### ❌ Inefficient Ordering (Breaks cache on every code edit)
```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
CMD ["node", "server.js"]
```

###  Optimized Directive Order (Leverages cached dependencies)
```dockerfile
FROM node:22-alpine
WORKDIR /app

# Step 1: Copy manifest files only
COPY package*.json ./

# Step 2: Install dependencies (cached until package manifests change)
RUN npm ci --omit=dev

# Step 3: Copy fast-changing source code
COPY src/ ./src
COPY server.js ./

USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 5. Build Arguments (`ARG`) vs. Environment Variables (`ENV`)

| Criterion | `ARG` (Build Argument) | `ENV` (Environment Variable) |
|:---|:---|:---|
| **Lifecycle** | Build time only | Available during build and container runtime |
| **CLI Input** | `--build-arg KEY=VALUE` | `-e KEY=VALUE` or `--env-file` |
| **Persistence** | Absent in running container processes | Persisted inside image metadata and containers |
| **Layer Invalidation** | Invalidates cache starting at the declared line | Invalidates cache starting at the declared line |

```dockerfile
ARG NODE_VERSION=22
FROM node:${NODE_VERSION}-alpine

ARG BUILD_REVISION
ENV NODE_ENV=production
ENV APP_PORT=3000

RUN echo "Compiling revision: ${BUILD_REVISION}"
```

> 🚨 **Critical Security Notice:** Never supply private tokens, registry passwords, or TLS keys via `ARG` or `ENV`. Both are recorded in the manifest metadata and viewable through `docker history` or `docker inspect`. Use `RUN --mount=type=secret` for credential handling.

---

## 6. Container Logging Architecture & Telemetry

Modern containerized applications delegate log collection to the container runtime by streaming unbuffered output directly to **`stdout`** and **`stderr`**.

```
[App stdout/stderr] ---> [Docker Daemon Logging Driver (json-file, local, syslog)] ---> [Log Router]
```

### Essential Log Inspection Commands
```bash
# Stream output continuously
docker logs -f <container_name_or_id>

# View the most recent 100 lines
docker logs --tail 100 <container_name_or_id>

# Filter events emitted during the last 15 minutes
docker logs --since 15m <container_name_or_id>

# Prepend high-precision ISO-8601 timestamps
docker logs -t --tail 50 <container_name_or_id>

# Inspect structured Docker daemon log configurations
docker inspect <container_name_or_id> --format='{{.HostConfig.LogConfig.Type}}'
```

---

## 7. Systematic Troubleshooting & Debugging Playbook

### Standard Container Exit Codes

| Exit Code | Meaning | Common Culprit |
|:---:|:---|:---|
| **0** | Success / Normal Exit | Ephemeral workload finished its command |
| **1** | Application Error | Unhandled exception, syntax error, missing environment variable |
| **126** | Invocation Error | Binary lacks executable permissions (`chmod +x entrypoint.sh`) |
| **127** | Command Not Found | Executable path missing or bad shell path inside minimal base image |
| **137** | `SIGKILL` (Fatal signal 9) | Host initiated forced kill; most commonly **Out Of Memory (OOM)** |
| **139** | `SIGSEGV` (Fatal signal 11) | Memory segmentation fault in compiled binaries |
| **143** | `SIGTERM` (Fatal signal 15) | Normal shutdown signal passed by `docker stop` |

### Rapid Debugging Decision Tree
```text
Container exits unexpectedly?
 ├── 1. Check Exit Code: docker inspect <id> --format='{{.State.ExitCode}}'
 ├── 2. Was it OOM killed? docker inspect <id> --format='{{.State.OOMKilled}}'
 ├── 3. Read failure logs: docker logs --tail 100 <id>
 └── 4. Inspect runtime state: docker inspect <id>
```

---

## 8. Hands-On Labs for Day 2

### Lab 1: Inspect Layer Weights and Commands
Examine how base images are structured and identify layer sizes:
```bash
docker history --human=true --format "table {{.CreatedBy}}\t{{.Size}}" python:3.12-slim
```

### Lab 2: Benchmark Cache Invalidation
1. Build an image containing combined `COPY . .` and `npm install` instructions.
2. Edit a documentation file or single code comment.
3. Re-run `docker build` and observe whether dependencies reinstall.
4. Refactor the `Dockerfile` to split manifests and test the cache speedup.

### Lab 3: Build Context Profiling
Run a build with debug output enabled to measure the raw bytes sent to the daemon:
```bash
DOCKER_BUILDKIT=1 docker build --progress=plain -t context-benchmark .
```

### Lab 4: Runtime Argument Injection
Pass build parameters and test dynamic override defaults:
```bash
docker build --build-arg APP_ENV=staging -t demo:stg .
docker run --rm -d --name test-env -e LOG_LEVEL=debug demo:stg
docker exec -it test-env env
```

### Lab 5: Investigating CrashLoops Interactively
Override entrypoints on broken containers to inspect the filesystem:
```bash
docker run --rm -it --entrypoint /bin/sh failing-image:latest
```

### Lab 6: Diagnose OOM Termination
Run a container constrained to 32MB of RAM and force allocation to view Exit Code 137:
```bash
docker run --rm --memory=32m python:3.12-slim python3 -c 'a = [0] * (10**7)'
```

---

## 9. Day 2 Production Best Practices

- **Pin explicit, immutable base tags:** Avoid `node:latest` or `python:alpine`. Use deterministic release tags such as `node:22.12.0-alpine3.20`.
- **Chain package manager steps with cleanups:** Always remove cache indexes in the same `RUN` command to prevent temporary package tarballs from polluting layer storage:
  ```dockerfile
  RUN apt-get update && apt-get install -y --no-install-recommends \
      curl \
      ca-certificates \
   && rm -rf /var/lib/apt/lists/*
  ```
- **Run as a Non-Root User:** System images run as root by default. Always instantiate an unprivileged application user:
  ```dockerfile
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser
  ```
- **Use Exec Form for Entrypoint and CMD:** Shell form (`CMD node server.js`) starts `/bin/sh -c`, which traps system signals and stops your app from intercepting `SIGTERM` gracefully. Use exec array syntax (`CMD ["node", "server.js"]`).

---

## 10. Day 2 Command Cheat Sheet

```bash
# Build an image with a specific tag and explicit Dockerfile
docker build -t myapp:2.1.0 -f Dockerfile.prod .

# Build without reading from the local cache
docker build --no-cache -t myapp:clean .

# Prune build cache to reclaim disk space
docker builder prune -f

# Inspect intermediate layers and build commands
docker history --no-trunc <image_name>

# Interactively connect to an active container with bash/sh
docker exec -it <container_name> /bin/sh

# Stream live CPU and memory utilization statistics
docker stats --no-stream

# Format container inspection output for rapid diagnosis
docker inspect <container> --format='IP: {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} | Status: {{.State.Status}}'
```

---

## 11. Day 2 Revision Checklist

- [ ] I can explain the difference between base image layers (`lowerdir`) and the container writable layer (`upperdir`).
- [ ] I understand why moving frequently changed instructions (`COPY . .`) near the top ruins build efficiency.
- [ ] I know how to construct a `.dockerignore` file to omit dependencies and security-sensitive files.
- [ ] I can differentiate between the purpose of `ARG` and `ENV`.
- [ ] I can identify exit codes 126, 127, 137, and 143 and know how to extract `OOMKilled` metrics.
- [ ] I can fetch real-time and timestamped container logs from standard file descriptors.

---

## 12. Test Yourself — Day 2 Quiz

#### 1. Why does changing a comment in your application code cause Docker to re-run `RUN npm install` if `COPY . .` is written above it?
> **Answer:** Docker invalidates the layer cache when file checksums change. Because `COPY . .` watches the entire context, any file update creates a new layer hash, forcing all subsequent instructions to rerun.

#### 2. How can an exit code of 137 be confirmed as an Out-of-Memory event rather than an external manual kill?
> **Answer:** Run `docker inspect <container_id> --format='{{.State.OOMKilled}}'`. If the kernel OOM killer terminated the process, this property returns `true`.

#### 3. What is the fundamental difference in the runtime container between a value set via `ARG` versus one set via `ENV`?
> **Answer:** `ARG` values exist exclusively during image build and are discarded from the environment of running containers; `ENV` variables persist into the image metadata and are loaded into every instantiated container's environment.

#### 4. Why should `RUN apt-get update` and `RUN apt-get install` never be run as separate `Dockerfile` instructions?
> **Answer:** If split across two instructions, the `apt-get update` layer will remain cached even when the remote package repository updates, causing subsequent installs to fail or fetch outdated/incompatible packages.

---

- **Multi-Stage Builds:** Compile binaries in builder environments and copy only minimal runtimes into production images (<20MB).
- **Advanced Volume Management:** Managing named volumes, host bind mounts, and persistent storage lifecycles.
- **Port Mapping & Container Networking Fundamentals:** How bridge networks, DNS discovery, and NAT port forwards work under the hood.
