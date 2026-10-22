# 🐳 Docker Day 1: Complete Beginner-to-Hands-On Guide

[![Docker](https://img.shields.io/badge/Docker-24.0%2B-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Docker Compose](https://img.shields.io/badge/Compose-v2-blue?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Level](https://img.shields.io/badge/Level-Beginner%20to%20Intermediate-green)](#)
[![Track](https://img.shields.io/badge/Track-DevOps%20%26%20Cloud%20Engineering-orange)](#)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](#)

> **Core Philosophy:** Package an application and all its runtime dependencies into a portable, immutable **image**, then instantiate and run that image as an isolated **container** anywhere.

---

## 📑 Table of Contents

1. [What is Docker?](#1-what-is-docker)
   - [A Real-World Analogy](#a-real-world-analogy)
   - [Why DevOps Engineers Use Docker](#why-devops-engineers-use-docker)
2. [Virtual Machines vs. Containers](#2-virtual-machines-vs-containers)
3. [Docker Architecture](#3-docker-architecture)
4. [Install and Verify Docker](#4-install-and-verify-docker)
5. [Lab 1 — Run Your First Container](#5-lab-1--run-your-first-container)
6. [Lab 2 — Container Lifecycle Management](#6-lab-2--container-lifecycle-management)
7. [Image vs. Container vs. Registry](#7-image-vs-container-vs-registry)
8. [Command Cheat Sheet](#8-command-cheat-sheet)
9. [What is a Dockerfile?](#9-what-is-a-dockerfile)
10. [Lab 3 — Build Your Own Web Application Image](#10-lab-3--build-your-own-web-application-image)
11. [Image Tags and Versions](#11-image-tags-and-versions)
12. [Ports and Networking (Lab 4)](#12-ports-and-networking-lab-4)
13. [Volumes and Persistent Data (Lab 5)](#13-volumes-and-persistent-data-lab-5)
14. [Docker Compose — Run Multi-Container Apps (Lab 6)](#14-docker-compose--run-multi-container-apps-lab-6)
15. [Docker Troubleshooting & Common Errors](#15-docker-troubleshooting--common-errors)
16. [Production-Style Best Practices](#16-production-style-best-practices)
17. [Day 1 Revision Checklist](#17-day-1-revision-checklist)
18. [Test Yourself — Interactive Quiz](#18-test-yourself--interactive-quiz)
19. [Learning Roadmap](#19-learning-roadmap)
20. [Official Resources & Next Steps](#20-official-resources--next-steps)

---

## 1. What is Docker?

**Docker** is an open-source containerization platform used for **building, packaging, shipping, and running applications inside isolated environments called containers**.

### The Classic Problem
> *"It works on my machine!"*  
> — Every developer before Docker.

A typical web application requires an interconnected stack:
- Programming runtime (e.g., Python 3.12, Node.js 22, Go 1.22)
- Application code & internal modules
- Third-party packages and pinned libraries
- System-level shared libraries (`libc`, `openssl`, etc.)
- Configuration files and environment variables
- Web servers and reverse proxies (e.g., Nginx)

When deploying across different environments (local laptops, staging servers, CI/CD runners, and production clouds), differences in operating systems, kernel utilities, and library versions cause subtle, hard-to-debug crashes. Docker eliminates this divergence by bundling everything the application needs into a single self-contained artifact.

### A Real-World Analogy

| Standard Shipping Container | Docker Container |
|:---|:---|
| Goods of any shape and size are packed into standardized steel shipping boxes. | Software code and dependencies are bundled into standardized, portable units. |
| Transported via trucks, trains, or ships without modifying the cargo inside. | Deployed on local laptops, on-prem servers, AWS, GCP, or Azure without modification. |
| Handled by universal cranes and docks regardless of the contents. | Handled by universal container runtimes (`dockerd`, `containerd`, `CRI-O`, Kubernetes). |

### Why DevOps Engineers Use Docker

- **Consistency across stages:** Eliminates environment drift between Dev, QA, Staging, and Production.
- **Lightning-fast spin-up:** Containers launch in milliseconds compared to minutes for VMs.
- **Process isolation:** Multiple applications run side-by-side on the same host without conflicting dependencies (e.g., Python 2.7 legacy and Python 3.12 modern).
- **Lightweight resource footprint:** Multiple containers share the same host OS kernel.
- **CI/CD streamlining:** Ephemeral test containers can be created, tested, and destroyed automatically in pipelines.
- **Stepping stone to Kubernetes:** Containerization is the prerequisite foundation for modern container orchestration.

> ⚠️ **Important Reality Check:** While Docker drastically improves cross-platform portability, containers share the host kernel. Differences in underlying CPU architecture (e.g., `x86_64` vs. `arm64`/Apple Silicon), host kernel configurations, capabilities, and system permissions must still be accounted for.

---

## 2. Virtual Machines vs. Containers

To understand Docker, compare how virtual machines (VMs) and containers utilize hardware and operating systems:

```
+-----------------------------------+      +-----------------------------------+
|       VIRTUAL MACHINES (VMs)      |      |         CONTAINERS (DOCKER)       |
+-----------------------------------+      +-----------------------------------+
|  App A      |  App B      | App C |      |  App A      |  App B      | App C |
|  Bins/Libs  |  Bins/Libs  | B/L   |      |  Bins/Libs  |  Bins/Libs  | B/L   |
|  Guest OS   |  Guest OS   | G-OS  |      +-------------+-------------+-------+
+-------------+-------------+-------+      |       Docker Engine Runtime       |
|            Hypervisor             |      +-----------------------------------+
+-----------------------------------+      |          Host OS Kernel           |
|         Host Operating System     |      +-----------------------------------+
+-----------------------------------+      |        Underlying Hardware        |
|        Underlying Hardware        |      +-----------------------------------+
+-----------------------------------+
```

### Feature Comparison

| Feature | Virtual Machine (VM) | Container (Docker) |
|:---|:---|:---|
| **Operating System** | Dedicated guest OS per VM (heavyweight) | Shares the host operating system kernel |
| **Startup Speed** | Minutes (boots a full operating system) | Milliseconds to seconds (starts a process) |
| **Storage & Memory** | Gigabytes per VM (GBs of OS files and RAM overhead) | Megabytes (only app code + minimal delta layers) |
| **Isolation Boundary** | Hardware-level virtualization (hypervisor) | OS-level isolation via Linux cgroups & namespaces |
| **Performance Overhead** | Hypervisor instruction translation tax | Direct near-native execution speed |
| **Ideal For** | Running different OS kernels, deep multi-tenant isolation | Microservices, API services, CLI utilities, CI/CD jobs |

> ℹ️ **Note on Docker Desktop:** On macOS and Windows, Docker Desktop automatically runs a lightweight Linux VM in the background (using Hyper-V, WSL2, or Apple Hypervisor) to provide the Linux kernel required to execute Linux containers.

---

## 3. Docker Architecture

Docker uses a client-server architecture consisting of four foundational components:

```
                   CLIENT                       DOCKER ENGINE / HOST                 REGISTRY
            +------------------+         +-------------------------------+      +---------------+
            |                  |         |  Docker Daemon (dockerd)      |      |               |
            |  docker run      |=======> |  +-------------------------+  |      |  Docker Hub   |
            |  docker build    | (REST)  |  | Containers              |  | <==> |  AWS ECR      |
            |  docker pull     |         |  |   [App 1]   [App 2]     |  |      |  Google GAR   |
            |                  |         |  +-------------------------+  |      |               |
            +------------------+         |  | Images                  |  |      +---------------+
                                         |  |   nginx     redis       |  |
                                         |  +-------------------------+  |
                                         |  | Networks & Volumes      |  |
                                         |  +-------------------------+  |
                                         +-------------------------------+
```
### Core Terminology Breakdown

| Component | Role & Function |
|:---|:---|
| **Docker CLI** | The terminal client used by developers to issue commands (`docker run`, `docker build`). Communicates via Unix sockets or REST API with the daemon. |
| **Docker Daemon (`dockerd`)** | The background service responsible for building images, running containers, managing storage volumes, and configuring virtual networks. |
| **Image** | An immutable, read-only template built from layered filesystems containing the runtime, code, binaries, and dependencies. |
| **Container** | A runnable, isolated runtime instance of an image. It includes a thin read-write layer on top of the image's immutable layers. |
| **Registry** | A hosted repository service for distributing images. Public default is Docker Hub; cloud equivalents include AWS ECR, Azure ACR, and Google Artifact Registry (GAR). |
| **Network** | Virtual software-defined bridge, host, or overlay network facilitating inter-container and host-to-container communication. |
| **Volume** | A persistent storage directory managed by Docker on the host filesystem that survives container deletion. |

---

## 4. Install and Verify Docker

Ensure your environment has a functioning Docker installation.

### Verification Commands

Open your shell and run:

```bash
# 1. Verify Docker CLI installation and client version
docker --version

# 2. Verify Docker Compose plugin availability
docker compose version

# 3. Check connectivity to Docker Daemon and runtime details
docker info
```

### Expected Output & Troubleshooting

| Command | What it Confirms | If It Fails |
|:---|:---|:---|
| `docker --version` | Docker client binary is present in `$PATH` | Reinstall Docker Engine or Docker Desktop |
| `docker compose version` | Compose V2 CLI plugin is available | Install `docker-compose-plugin` |
| `docker info` | CLI can communicate with `dockerd` over `/var/run/docker.sock` | Daemon is stopped or user lacks permissions (add user to `docker` group) |

> 💡 **Linux Permission Tip:** If `docker info` returns `permission denied while trying to connect to the Docker daemon socket`, grant your user access:
> ```bash
> sudo usermod -aG docker $USER
> newgrp docker
> ```

---

## 5. Lab 1 — Run Your First Container

**Objective:** Pull an official web server image, run it as an isolated container in background mode, expose it to a host port, and verify HTTP connectivity.

### Step 1 — Download the Image
Fetch the lightweight Alpine Linux-based Nginx image from Docker Hub:

```bash
docker pull nginx:alpine
```

### Step 2 — Verify Downloaded Images
List local images stored in your local Docker cache:

```bash
docker images
```

*Output should display repository `nginx`, tag `alpine`, Image ID, creation date, and size (~23MB).*

### Step 3 — Launch the Container
Run the container in detached mode with port mapping and a friendly name:

```bash
docker run -d \
  --name day1-nginx \
  -p 8080:80 \
  nginx:alpine
```

#### Dissecting the Flag Syntax:

```
docker run   -d              --name day1-nginx     -p 8080:80              nginx:alpine
│            │               │                     │                       │
Command      Run detached    Assign custom name    Map Host Port 8080      Target Image:Tag
             (background)    instead of random     to Container Port 80
```

### Step 4 — Verify the Container State
Inspect the running status, process logs, and allocated port:

```bash
# View active containers
docker ps

# Stream initial server access/error logs
docker logs day1-nginx

# Inspect port bindings
docker port day1-nginx
```

### Step 5 — Test HTTP Connectivity
Send an HTTP request to the exposed port:

```bash
curl -I http://localhost:8080
```

**Expected Response:**
```http
HTTP/1.1 200 OK
Server: nginx/...
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
```

🎉 **Congratulations!** Your first Docker web server is running.

---

## 6. Lab 2 — Container Lifecycle Management

Containers transition through states: **Created** ➔ **Running** ➔ **Paused** ➔ **Stopped (Exited)** ➔ **Removed**.

```
                +-------------------+
                |      docker       |
                |      create       |
                +-------------------+
                          │
                          ▼
+------------+       +---------+       +---------+
|   docker   | ====> | RUNNING | <==== | docker  |
|    stop    |       +---------+       |  start  |
+------------+        │       │        +---------+
       │              │       │             ▲
       ▼              │       ▼             │
+------------+        │  +---------+        │
|   EXITED   | <──────┘  | PAUSED  | ───────┘
+------------+           +---------+
       │
       ▼ (docker rm)
 [ Destroyed ]
```

Execute these lifecycle commands in sequence to master state transitions:

```bash
# 1. List only currently running containers
docker ps

# 2. Gracefully stop the running container (sends SIGTERM, then SIGKILL after 10s)
docker stop day1-nginx

# 3. Confirm it is no longer listed in active processes
docker ps

# 4. View all containers including stopped/exited ones
docker ps -a

# 5. Start the stopped container back up
docker start day1-nginx

# 6. Verify it is running again
docker ps

# 7. Restart the container process in-place
docker restart day1-nginx

# 8. Forcibly stop and remove the container
docker rm -f day1-nginx

# 9. Verify container list is clean
docker ps -a
```

> ⚠️ **Key Takeaway:** Stopping a container preserves its configuration and writable layer. Running `docker rm` permanently destroys the container and any unsaved data in its writable layer. The underlying image (`nginx:alpine`) remains intact in your local cache.

---

## 7. Image vs. Container vs. Registry

Understanding the boundaries between these three concepts is essential for technical interviews and day-to-day operations:

| Concept | What It Is | State | Analogy in Programming |
|:---|:---|:---|:---|
| **Image** | A packaged, immutable filesystem snapshot with metadata, environment configs, and entrypoint instructions. | Static / Read-Only | A **Class** definition in OOP |
| **Container** | An active, isolated OS-level process instance running an image, augmented with a temporary writable layer. | Dynamic / State-changing | An **Object instance** instantiated from that Class |
| **Registry** | A distributed storage and versioning repository for sharing images (Docker Hub, Quay, ECR, Artifact Registry). | Central Artifact Storage | **GitHub / npm / PyPI** for container images |

### Hands-On Proof: Multiple Containers from One Single Image

Demonstrating that images are immutable blueprints:

```bash
# Spin up two completely independent containers using the same base image:
docker run -d --name web1 -p 8081:80 nginx:alpine
docker run -d --name web2 -p 8082:80 nginx:alpine

# Confirm both are running concurrently:
docker ps

# Test both independent endpoints:
curl -I http://localhost:8081
curl -I http://localhost:8082

# Cleanup:
docker rm -f web1 web2
```

---

## 8. Command Cheat Sheet

Bookmark this comprehensive reference table for Day 1 and beyond:

### Inspect & Information Commands

```bash
docker --version                  # Show client version
docker info                       # Detailed system, engine, and driver info
docker system df                  # Disk space usage breakdown (images, containers, volumes)
docker system prune -f            # Remove stopped containers, dangling images, unused networks
```

### Image Management Commands

```bash
docker pull <image>:<tag>         # Pull image from registry
docker images                     # List all local images
docker image inspect <image>      # Display detailed JSON metadata of an image
docker rmi <image>                # Remove one or more images from local cache
docker image prune                # Delete unused/untagged dangling images
```

### Container Lifecycle & Execution

```bash
docker run -d -p <H>:<C> <img...> # Run new container in detached background mode
docker ps                         # List active running containers
docker ps -a                      # List all containers (running + stopped)
docker stop <container>           # Gracefully stop container (SIGTERM)
docker start <container>          # Start an existing stopped container
docker restart <container>        # Restart container
docker rm <container>             # Delete stopped container
docker rm -f <container>          # Force delete container even if running
docker logs <container>           # View container stdout/stderr output
docker logs -f <container>        # Stream container logs in real time
docker stats                      # Live stream of container CPU, RAM, Network I/O
docker exec -it <container> sh    # Open an interactive shell inside a running container
docker inspect <container>        # View full JSON configuration and network settings
```

---

## 9. What is a Dockerfile?

A **Dockerfile** is a plain-text configuration file containing an ordered sequence of instructions used by the Docker Engine to automatically assemble a custom container image.

### Essential Dockerfile Directives

| Instruction | What It Does | Execution Phase | Example |
|:---|:---|:---|:---|
| `FROM` | Declares the parent base image to start from | Build-time | `FROM node:22-alpine` |
| `WORKDIR` | Sets the default working directory for subsequent instructions | Build-time | `WORKDIR /app` |
| `COPY` | Copies files from the host build context into image filesystem | Build-time | `COPY . .` |
| `ADD` | Like `COPY`, but can unpack tar archives and fetch URLs *(prefer `COPY` unless unpacking)* | Build-time | `ADD archive.tar.gz /data/` |
| `RUN` | Executes commands inside the container and commits a new layer | Build-time | `RUN npm install --production` |
| `ENV` | Sets persistent environment variables inside the container | Build & Runtime | `ENV NODE_ENV=production` |
| `EXPOSE` | Documents the ports the application listens on *(informational)* | Metadata | `EXPOSE 3000` |
| `USER` | Switches execution user from root to a non-privileged account | Build & Runtime | `USER node` |
| `ENTRYPOINT` | Defines the default executable command that always runs | Runtime | `ENTRYPOINT ["nginx"]` |
| `CMD` | Supplies default arguments or default command *(overridable)* | Runtime | `CMD ["-g", "daemon off;"]` |

> 🧠 **Rule of Thumb:**
> - `RUN` = Executes **during image build** to install software and configure files.
> - `CMD` = Executes **at container runtime** when the container launches.

---

## 10. Lab 3 — Build Your Own Web Application Image

**Objective:** Author a custom HTML document, write a clean Dockerfile, build a versioned Docker image, and run it as an isolated web service.

```bash
mkdir -p ~/Docker/my-web
cd ~/Docker/my-web

cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Docker Day 1 Custom App</title>
</head>
<body>
  <h1>Hello from Custom Container!</h1>
  <p>Built using Dockerfile on Day 1.</p>
</body>
</html>
EOF

cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

docker build -t my-web:v1 .
docker run -d --name my-web-container -p 8080:80 my-web:v1
curl http://localhost:8080
```

---

## 11. Image Tags and Versions

Docker image identifiers follow this structure: `[registry/][namespace/]repository[:tag]`.

- Always pin specific, deterministic tags in production (e.g., `python:3.12-slim`, `nginx:1.27-alpine`).
- Avoid relying on mutable `:latest` tag for mission-critical deployments.

---

## 12. Ports and Networking (Lab 4)

```bash
# Create custom user-defined network
docker network create my-network

# Attach container web-a
docker run -d --name web-a --network my-network nginx:alpine

# Verify internal DNS resolution by container name:
docker run --rm --network my-network busybox:1.36 wget -qO- http://web-a

# Cleanup:
docker rm -f web-a
docker network rm my-network
```

---

## 13. Volumes and Persistent Data (Lab 5)

```bash
# Create managed volume
docker volume create my-data

# Write data from an ephemeral container
docker run --rm --mount source=my-data,target=/data busybox:1.36 sh -c 'echo "Docker persistence works" > /data/message.txt'

# Verify data persists into a completely new container
docker run --rm --mount source=my-data,target=/data busybox:1.36 cat /data/message.txt

# Cleanup:
docker volume rm my-data
```

---

## 14. Docker Compose — Run Multi-Container Apps (Lab 6)

```bash
mkdir -p ~/Docker/compose-demo
cd ~/Docker/compose-demo

cat > compose.yaml <<'EOF'
services:
  web:
    build: .
    ports:
      - "8080:80"
    restart: unless-stopped
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    restart: unless-stopped
EOF

docker compose up -d --build
docker compose ps
docker compose down
```

---

## 15. Docker Troubleshooting & Common Errors

| Issue | Resolution |
|:---|:---|
| Port 8080 already allocated | Stop conflicting process or change host port: `-p 8085:80` |
| Container name already in use | Remove stale container: `docker rm -f <name>` |
| Cannot connect to Docker daemon | Start Docker service (`sudo systemctl start docker` or start Docker Desktop) |

---

## 16. Production-Style Best Practices

1. **Enforce Non-Root Users (`USER`):** Prevent container breakout privileges.
2. **Never Store Secrets in Images:** Pass API keys and passwords via runtime environments.
3. **Use Minimal Base Images:** Prefer Alpine Linux to minimize image size and CVE surface.
4. **Leverage `.dockerignore`:** Keep build context clean from node_modules, .git, and .env files.
5. **Implement Native Health Checks:** Add `HEALTHCHECK` instructions for orchestrators.
6. **Protect Docker Socket:** Never mount `/var/run/docker.sock` in untrusted containers.

---

## 17. Day 1 Revision Checklist

- [ ] Explain what Docker is and why containers prevent environment drift
- [ ] Understand architecture differences between Virtual Machines and Containers
- [ ] Grasp Docker CLI, Daemon, Images, Containers, Registries, Volumes, and Networks
- [ ] Verify Docker Engine installation (`docker --version`, `docker info`)
- [ ] Run an Nginx container and verify HTTP 200 via `curl`
- [ ] Master lifecycle commands: `ps`, `stop`, `start`, `restart`, `logs`, `exec`, `rm`
- [ ] Author a Dockerfile and build an image via `docker build -t`
- [ ] Explain the difference between build-time `RUN` and runtime `CMD`
- [ ] Connect containers using a custom network and test DNS resolution
- [ ] Persist data across container lifecycles using a named volume
- [ ] Run a multi-service stack using Docker Compose (`docker compose up -d`)

---

## 18. Test Yourself — Interactive Quiz

1. **What is a Docker image?** &rarr; An immutable read-only template used to create containers.
2. **In `-p 8080:80`, which is the host port?** &rarr; Port 8080.
3. **Which Docker feature preserves data after a container is removed?** &rarr; Docker Volume or Bind Mount.
4. **What is the difference between `RUN` and `CMD`?** &rarr; `RUN` executes during build; `CMD` defines runtime executable.
5. **What command launches a compose stack with a fresh rebuild?** &rarr; `docker compose up -d --build`.

---

## 19. Learning Roadmap

- **Day 1:** Core Foundations (Images, Containers, Commands, Dockerfiles, Ports, Volumes, Compose)

---

## 20. Official Resources

- Docker Docs: https://docs.docker.com/
- Dockerfile Reference: https://docs.docker.com/reference/dockerfile/
- Compose Spec: https://docs.docker.com/compose/
