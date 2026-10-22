# Docker Day 1 — Complete Beginner-to-Hands-On Guide
  - Today, we'll learn Docker from zero, like a real Cloud & DevOps Engineer. We'll cover the concepts, commands, hands-on labs, a mini-project, troubleshooting, and interview questions.
  - Since you're learning practical DevOps tools, we'll focus on commands you can actually run in a terminal, including Google Cloud Shell.
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/b88877e1-87a6-44b8-9ed4-4e3599441588" />

# 1. What is Docker?
  -  Docker is a platform for building, packaging, shipping and running applications in containers.
  -  Imagine you develop an application on your laptop. It works perfectly, but when you deploy it to a server, it fails because the server has different software versions or missing dependencies.
  -  Docker helps solve this problem by packaging the application and its required dependencies into a consistent environment.

        - For example, a web application might need:
        - Python 3.12
        - Required Python libraries
        - Application source code
        - Configuration settings
        - A web server
  - You can package these into a Docker image and run that image on different supported environments.
## Why do DevOps engineers use Docker?
- Consistent development, testing, and production environments.
- Faster application deployment.
- Isolation between applications.
- Easier CI/CD automation.
- Convenient scaling and replacement of application instances.
- A foundation for container orchestration with Kubernetes.
  
**Important: Docker improves consistency, but it doesn't guarantee that every application will run identically everywhere. CPU architecture, kernel capabilities, permissions, networking, secrets, and configuration can still matter.**

# 2. Virtual machines vs. Docker containers.

**Virtual machines (VMs)**
  - Each VM normally includes its own guest operating system on virtualized hardware. This can provide a strong isolation boundary but generally uses more resources.
    
**Docker containers**
  - Containers isolate applications and their processes while sharing the host kernel. They are usually lighter and faster to start than full VMs.

# 3. Understand the Docker architecture.
- **Docker CLI:** The command-line client you use.
- **Docker daemon (dockerd):** A background service that manages Docker objects.
- **Docker image:** A read-only template used to create containers.
- **Docker container:** An isolated runtime instance created from an image.
- **Registry:** A service that stores and distributes images, such as Docker Hub or a private registry.
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/e89992b8-1c12-4002-9b81-49239f27408e" />

# 4. Install and verify Docker.
- You can use Docker on a local Linux machine, Docker Desktop, or a suitable cloud environment.
  
 **How to interpret the results:**
- [**docker --version**] — confirms that the Docker CLI is installed.
- [**docker compose version**] — confirms that the Compose plugin is available.
- [**docker info**] — checks whether the CLI can communicate with the Docker daemon.

# 🐳 Docker Day 1 — Complete Beginner-to-Hands-On Guide

A practical, command-first introduction to Docker for aspiring Cloud & DevOps Engineers. This guide covers concepts, hands-on labs, a Compose mini-project, troubleshooting, best practices, and a self-quiz.

> **Core idea:** package an application and its dependencies into an **image**, then run that image as a **container**.

---

## 📑 Table of Contents

1. [What is Docker?](#1-what-is-docker)
2. [Virtual Machines vs. Containers](#2-virtual-machines-vs-containers)
3. [Docker Architecture](#3-docker-architecture)
4. [Install and Verify Docker](#4-install-and-verify-docker)
5. [Lab 1 — Run Your First Container](#5-lab-1--run-your-first-container)
6. [Lab 2 — Container Lifecycle](#6-lab-2--container-lifecycle)
7. [Image vs. Container vs. Registry](#7-image-vs-container-vs-registry)
8. [Command Cheat Sheet](#8-command-cheat-sheet)
9. [What is a Dockerfile?](#9-what-is-a-dockerfile)
10. [Lab 3 — Build Your Own Image](#10-lab-3--build-your-own-web-application-image)
11. [Image Tags and Versions](#11-image-tags-and-versions)
12. [Ports and Networking (Lab 4)](#12-container-ports-and-networking)
13. [Volumes and Persistent Data (Lab 5)](#13-docker-volumes-and-persistent-data)
14. [Docker Compose (Lab 6)](#14-docker-compose--run-applications-with-one-file)
15. [Troubleshooting](#15-docker-troubleshooting)
16. [Production-Style Best Practices](#16-production-style-best-practices)
17. [Revision Checklist](#17-day-1-revision-checklist)
18. [Quiz](#18-test-yourself--quiz)
19. [Learning Roadmap](#19-learning-roadmap)
20. [Official Resources](#20-official-resources)

---

## 1. What is Docker?

Docker is a platform for **building, packaging, shipping, and running applications in containers**.

Imagine you develop an application on your laptop and it works perfectly, but on the server it fails because of different software versions or missing dependencies. Docker solves this by packaging the application and its dependencies into a consistent environment.

A web application might need:

- Python 3.12
- Application source code
- Required Python libraries
- Configuration settings
- A web server

You package these into a Docker **image** and run that image on different supported environments.

### A real-world analogy

| Shipping container | Docker container |
|---|---|
| Goods are packed into a standard container so they can be transported between locations. | An application is packaged with its runtime and dependencies so it can run consistently. |

### Why DevOps engineers use Docker

- Consistent development, testing, and production environments
- Faster application deployment
- Isolation between applications
- Easier CI/CD automation
- Convenient scaling and replacement of application instances
- A foundation for container orchestration with Kubernetes

> ⚠️ **Important:** Docker improves consistency, but it doesn't guarantee every application runs identically everywhere. CPU architecture, kernel capabilities, permissions, networking, secrets, and configuration still matter.

---

## 2. Virtual Machines vs. Containers

**Virtual machines (VMs)** normally include their own guest operating system on virtualized hardware. This provides a strong isolation boundary but generally uses more resources.

**Containers** isolate applications and processes while **sharing the host kernel**. They are usually lighter and faster to start than full VMs.

| Feature | Virtual Machine | Container |
|---|---|---|
| Operating system | Separate guest OS | Shares host kernel |
| Startup | Usually slower | Usually faster |
| Resource use | Generally higher | Generally lower |
| Isolation | Hardware virtualization boundary | OS-level isolation |
| Typical use | Full OS environments | Applications and services |

> ℹ️ Docker Desktop and some cloud development environments use a Linux VM behind the scenes to run Linux containers. On Windows, macOS, or Cloud Shell, the underlying architecture may differ from a native Linux server.
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/f7904122-0195-48f0-aea6-e2f594e5cc44" />

---

## 3. Docker Architecture

Learn these terms first:

| Component | Description |
|---|---|
| **Docker CLI** | The command-line client you use (e.g. `docker run`) |
| **Docker daemon (`dockerd`)** | Background service that manages images, containers, networks, and volumes |
| **Image** | A read-only template used to create containers |
| **Container** | An isolated runtime instance created from an image |
| **Registry** | A service that stores and distributes images (Docker Hub, private registries) |
| **Network** | Enables container communication |
| **Volume** | Persistent data storage |

```
 You ──► Docker CLI ──► Docker Daemon ──► Images ──► Containers
                                │                        │
                                ├── Networks ◄───────────┤
                                └── Volumes  ◄───────────┘
```

Modern Docker Engine installations may also use `containerd` and additional runtime components behind the scenes.

---

## 4. Install and Verify Docker

You can use Docker on a local Linux machine, Docker Desktop, or a suitable cloud environment.

> **Google Cloud Shell note:** Cloud Shell does not necessarily have a working Docker daemon available by default. Verify before trying to build anything.

```bash
docker --version
docker compose version
docker info
```

| Command | What it confirms |
|---|---|
| `docker --version` | The Docker CLI is installed |
| `docker compose version` | The Compose plugin is available |
| `docker info` | The CLI can communicate with the Docker daemon |

If `docker info` fails with a daemon or socket error, don't assume Docker is working. You need an environment with a supported, running Docker Engine or another supported container runtime.

📖 [Docker getting-started documentation](https://docs.docker.com/get-started/)

---

## 5. Lab 1 — Run Your First Container

**Goal:** Start a web server and verify that it responds.

**Step 1 — Download the image**

```bash
docker pull nginx:alpine
```

**Step 2 — Check downloaded images**

```bash
docker images
```

You should see an image named `nginx` with the `alpine` tag.

**Step 3 — Run the container**

```bash
docker run -d \
  --name day1-nginx \
  -p 8080:80 \
  nginx:alpine
```

| Part | Meaning |
|---|---|
| `docker run` | Create and start a container |
| `-d` | Run in the background (detached) |
| `--name day1-nginx` | Give it a memorable name |
| `-p 8080:80` | Map host port 8080 to container port 80 |
| `nginx:alpine` | Image and tag to use |

**Step 4 — Verify the container**

```bash
docker ps
docker logs day1-nginx
docker port day1-nginx
```

**Step 5 — Test the web server**

```bash
curl -I http://localhost:8080
```

A successful response normally contains `HTTP/1.1 200 OK`.

> 💡 In Google Cloud Shell, `localhost` refers to the Cloud Shell environment. Use Cloud Shell's **Web Preview** to view the site in your browser.

---

## 6. Lab 2 — Container Lifecycle

Run these commands **one at a time**:

```bash
docker ps
docker ps -a
docker stop day1-nginx
docker ps
docker start day1-nginx
docker ps
docker restart day1-nginx
docker rm -f day1-nginx
```

**What you learned:**

- `docker ps` lists running containers
- `docker ps -a` lists running **and** stopped containers
- `docker stop` stops a running container
- `docker start` starts an existing stopped container
- `docker restart` restarts a container
- `docker rm -f` forcibly removes the container

> ⚠️ `docker rm -f` removes the container, **not its image**. Any data stored only in the container's writable layer is lost.

---

## 7. Image vs. Container vs. Registry

One of the most important distinctions for interviews and real work.

| Concept | Description |
|---|---|
| **Image** | A template from which containers are created. Typically immutable once built; identified by tags or content digests. |
| **Container** | A runtime instance of an image, with its own configuration and writable layer. |
| **Registry** | A location to store and distribute images — Docker Hub, Amazon ECR, Azure Container Registry, Google Artifact Registry. |

**Try it — two containers from one image:**

```bash
docker run -d --name web1 -p 8081:80 nginx:alpine
docker run -d --name web2 -p 8082:80 nginx:alpine

docker ps
curl -I http://localhost:8081
curl -I http://localhost:8082
```

Both containers use the same image but have different names and host ports.

**Clean up:**

```bash
docker rm -f web1 web2
```

---

## 8. Command Cheat Sheet

| Command | Purpose |
|---|---|
| `docker --version` | Check CLI version |
| `docker info` | Inspect Engine and runtime information |
| `docker pull IMAGE` | Download an image |
| `docker images` | List local images |
| `docker run IMAGE` | Create and start a container |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker logs CONTAINER` | Read container logs |
| `docker exec -it CONTAINER sh` | Open a shell in a running container (if available) |
| `docker inspect CONTAINER` | Inspect detailed container configuration |
| `docker stats` | View live resource usage |
| `docker stop CONTAINER` | Stop a container |
| `docker start CONTAINER` | Start an existing container |
| `docker rm CONTAINER` | Remove a stopped container |
| `docker rmi IMAGE` | Remove an unused image |
| `docker system df` | Check Docker disk usage |

**Inspect an image:**

```bash
docker pull nginx:alpine
docker image inspect nginx:alpine
```

**Inspect a running container and open a shell:**

```bash
docker run -d --name inspect-demo nginx:alpine
docker inspect inspect-demo
docker exec -it inspect-demo sh
```

Inside the container, try:

```sh
hostname
cat /etc/os-release
exit
```

`exit` leaves the shell; it does not stop the container because Nginx continues running as the main process.

**Remove the lab container:**

```bash
docker rm -f inspect-demo
```

---

## 9. What is a Dockerfile?

A **Dockerfile** is a text file containing instructions to build your own Docker image.

| Instruction | Purpose |
|---|---|
| `FROM` | Select a base image |
| `WORKDIR` | Set the working directory |
| `COPY` | Copy files into the image |
| `RUN` | Execute a command during the build |
| `ENV` | Define environment variables |
| `EXPOSE` | Document the intended listening port |
| `USER` | Select the runtime user |
| `CMD` | Define the default command |
| `ENTRYPOINT` | Define the main executable |

> 🧠 **Remember:** `RUN` executes **while building** an image; `CMD` defines the default command **at runtime**.

---

## 10. Lab 3 — Build Your Own Web Application Image

We'll create a small static website — no programming language needed.

**Step 1 — Create a project directory**

```bash
mkdir -p ~/docker-day1/my-web
cd ~/docker-day1/my-web
```

**Step 2 — Create an HTML page**

```bash
cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Docker Day 1</title>
</head>
<body>
  <h1>Hello from my Docker container!</h1>
  <p>My first custom Docker image is running.</p>
</body>
</html>
EOF
```

**Step 3 — Create a Dockerfile**

```bash
cat > Dockerfile <<'EOF'
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
EOF
```

Project structure:

```
my-web/
├── Dockerfile
└── index.html
```

**Step 4 — Build your image**

```bash
docker build -t my-web:v1 .
```

The `.` means Docker uses the current directory as the **build context**.

**Step 5 — Verify the image**

```bash
docker images
docker image inspect my-web:v1
```

**Step 6 — Run your application**

```bash
docker run -d \
  --name my-web-container \
  -p 8080:80 \
  my-web:v1
```

> If port 8080 is still in use from an earlier lab, remove that container first or choose another host port.

**Step 7 — Test the application**

```bash
curl http://localhost:8080
```

You should see HTML containing `Hello from my Docker container!`.

Verify the status code:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

Expected output:

```
200
```

If you don't get `200`, check `docker ps -a`, `docker logs my-web-container`, and whether port 8080 is already in use.

**What happens during `docker build`?**

```
Dockerfile + source files ──► Docker build process ──► Image: my-web:v1 ──► Container: my-web-container
```

The image can be reused to create many containers. Docker can also reuse cached build layers to speed up subsequent builds.

---

## 11. Image Tags and Versions

An image name often looks like `nginx:alpine`:

- `nginx` → repository name
- `alpine` → tag

Other examples: `python:3.12-slim`, `redis:7-alpine`, `nginx:1.28`.

> ⚠️ Tags identify intended versions or variants, but **tags can be mutable**. For repeatable production deployments, pin approved image digests where appropriate, scan images, and maintain a controlled update process.

**Practice:**

```bash
docker pull nginx:alpine
docker image ls
docker image tag my-web:v1 my-web:day1
docker image inspect my-web:day1
```

Remove the original tag:

```bash
docker image rm my-web:v1
```

This removes only that tag if another tag still references the same image. Don't remove an image a running container needs.

---

## 12. Container Ports and Networking

Port mapping is a key concept for cloud and DevOps engineers.

```
-p HOST_PORT:CONTAINER_PORT
```

```bash
docker run -d --name nginx-demo -p 8080:80 nginx:alpine
docker run -d --name site -p 9090:80 nginx:alpine
```

Test the second one at `http://localhost:9090` on the Docker host.

**Important distinctions:**

- `EXPOSE 80` in a Dockerfile **documents** a container port; it does **not** publish it on the host.
- `-p 8080:80` **publishes** container port 80 through host port 8080.
- Containers on the same suitable Docker network can generally communicate using container ports.
- On a user-defined network, containers can reach each other by container name or network alias.

### Lab 4 — Create a Docker network

```bash
docker network create day1-network

docker run -d \
  --name web-a \
  --network day1-network \
  nginx:alpine

docker run --rm \
  --network day1-network \
  busybox:1.36 \
  wget -qO- http://web-a
```

The final command should return Nginx's HTML response. The temporary BusyBox container resolves `web-a` through the user-defined network and connects on port 80.

Inspect the network:

```bash
docker network ls
docker network inspect day1-network
```

Clean up:

```bash
docker rm -f web-a
docker network rm day1-network
```

---

## 13. Docker Volumes and Persistent Data

A container's writable layer is **not** a reliable place for important data. If the container is removed, changes stored only in that layer disappear. A **Docker volume** stores data separately from an individual container's lifecycle.

### Lab 5 — Save data using a volume

```bash
docker volume create day1-data

docker run --rm \
  --mount source=day1-data,target=/data \
  busybox:1.36 \
  sh -c 'echo "Docker persistence works" > /data/message.txt'
```

Now read it from a **new** container using the same volume:

```bash
docker run --rm \
  --mount source=day1-data,target=/data \
  busybox:1.36 \
  cat /data/message.txt
```

Expected output:

```
Docker persistence works
```

The first container was removed, yet the data remains in the volume.

```bash
docker volume inspect day1-data
```

When finished practising (only if you no longer need the data):

```bash
docker volume rm day1-data
```

> **Bind mounts** connect a specific host directory to a container directory. They're useful for local development and sharing config or source files. Volumes are often a better default for application data managed by Docker.

---

## 14. Docker Compose — Run Applications with One File

Docker Compose lets you define and manage multiple services using YAML. Instead of running many commands manually, you declare the desired services in a Compose file.

### Lab 6 — Build a small Compose project

```bash
mkdir -p ~/docker-day1/compose-demo
cd ~/docker-day1/compose-demo
```

**`index.html`**

```bash
cat > index.html <<'EOF'
<h1>Hello from Docker Compose!</h1>
<p>My first multi-service setup.</p>
EOF
```

**`Dockerfile`**

```bash
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
```

**`compose.yaml`**

```bash
cat > compose.yaml <<'EOF'
services:
  web:
    build: .
    ports:
      - "8080:80"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped
EOF
```

**Start the project**

```bash
docker compose config
docker compose up -d --build
```

**Verify**

```bash
docker compose ps
docker compose logs
curl http://localhost:8080
```

You should see two services: `web` and `redis`. The `web` service builds from your Dockerfile; Redis uses a published image.

**Stop the project**

```bash
docker compose down
```

> ⚠️ This example is for learning, not production: Redis has no persistence, authentication, or health check configured.

---

## 15. Docker Troubleshooting

When something fails, don't randomly restart everything. Diagnose systematically.

| Problem | Commands to investigate |
|---|---|
| Container is not running | `docker ps -a` |
| Application exits immediately | `docker logs CONTAINER` |
| Port is already occupied | `docker ps` and inspect host processes |
| Cannot access web application | `docker port CONTAINER` and `curl` |
| Cannot pull an image | `docker pull IMAGE` and check connectivity |
| Container cannot resolve another service | `docker network inspect NETWORK` |
| Disk space running low | `docker system df` |
| Docker daemon unavailable | `docker info` |

### Common errors

**Error 1 — Container name already exists**

```
Conflict. The container name is already in use.
```

```bash
docker ps -a
docker rm -f day1-nginx
```

Only remove the old container if you no longer need it.

**Error 2 — Port already allocated**

```
Bind for 0.0.0.0:8080 failed: port is already allocated
```

Use a different host port:

```bash
docker run -d --name nginx-alt -p 8082:80 nginx:alpine
```

**Error 3 — Cannot connect to Docker daemon**

```
Cannot connect to the Docker daemon
```

```bash
docker info
```

The Docker Engine may not be installed, running, or accessible to your user. On a Linux machine using systemd:

```bash
sudo systemctl status docker
```

> This systemd command isn't appropriate for every environment, including some cloud shells. Don't try to install or start services without checking your environment's restrictions first.

---

## 16. Production-Style Best Practices

1. **Run as a non-root user where possible** — avoid unnecessary privileges; choose a user and file permissions appropriate to the app.
2. **Use trusted, maintained base images** — use supported versions, scan for vulnerabilities, update through a controlled process.
3. **Never bake secrets into images** — no passwords, tokens, or private keys in Dockerfiles, image layers, or source control. Use a proper secret-management mechanism.
4. **Persist important data** — use volumes or managed external storage for data that must survive container replacement.
5. **Monitor application health** — use health checks, logs, metrics, resource limits, and app-level monitoring. A running container isn't necessarily a healthy application.
6. **Automate image builds** — use CI/CD to build, test, scan, version, and publish images to an approved registry.

> 🔒 **Never expose the Docker daemon's privileged control socket to untrusted containers.** Access to it can effectively grant control over the Docker host.

---

## 17. Day 1 Revision Checklist

- [ ] Explain Docker and why containers are useful
- [ ] Understand images, containers, registries, and the daemon
- [ ] Run Nginx and test its HTTP response
- [ ] Start, stop, inspect, and remove containers
- [ ] Build a custom image using a Dockerfile
- [ ] Explain host ports versus container ports
- [ ] Connect containers using a Docker network
- [ ] Persist a file using a named volume
- [ ] Run a multi-service Compose project
- [ ] Diagnose one deliberately created error

---

## 18. Test Yourself — Quiz

**1. What is a Docker image?**
- A running process
- A template used to create containers
- A virtual machine

**2. What does `-p 8080:80` mean?**
- Host port 80 maps to container port 8080
- Both ports are on the host
- Host port 8080 maps to container port 80

**3. Which file defines instructions for building an image?**
- `Dockerfile`
- `compose.yaml`
- `README.md`

**4. Which Docker feature helps preserve data after a container is removed?**
- Image tag
- Named volume
- Container name

**5. What does `docker compose up -d --build` do?**
- Builds images if needed and starts services in the background
- Deletes all containers
- Uploads images to a registry

<details>
<summary><b>Show answers</b></summary>

1. **A template used to create containers** — an image is the template; a container is the runtime instance.
2. **Host port 8080 maps to container port 80** — host port first, container port second.
3. **`Dockerfile`** — contains image build instructions.
4. **Named volume** — exists independently of an individual container.
5. **Builds images if needed and starts services in the background** — detached mode.

</details>

---

## 19. Learning Roadmap

| Day | Topic | Focus |
|---|---|---|
| **Day 1** ⭐ | Docker fundamentals | Images, containers, basic commands, Dockerfiles, ports, volumes, networks, Compose |
| **Day 2** | Image building & debugging | Image layers, caching, build context, `.dockerignore`, environment variables, logs, troubleshooting |
| **Day 3** | Docker Compose project | Web service + database, networks and volumes, verify persistence |
| **Day 4** | Production image practices | Multi-stage builds, non-root users, health checks, resource limits, image scanning, secrets |
| **Day 5+** | Real DevOps workflows | Push to a registry, CI/CD automation, cloud deployment, Kubernetes fundamentals |

---

## 20. Official Resources

- [Docker Getting Started](https://docs.docker.com/get-started/)
- [Docker CLI Reference](https://docs.docker.com/reference/cli/docker/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## ✅ Today's Task

Complete **Labs 1–6 in order**. If a command fails, note the exact command and error output rather than assuming the lab succeeded.
