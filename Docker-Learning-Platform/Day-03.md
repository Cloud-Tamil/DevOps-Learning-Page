# Docker Compose Project: Web Service + PostgreSQL (Day 3)

An end-to-end, production-oriented reference guide for deploying a multi-container architecture using Docker Compose. This project couples a hardened, multi-stage Python Flask web service with a persistent PostgreSQL database, complete with health checks, Docker secrets, isolated networking, and resource limits.

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Architecture & Workflow](#architecture--workflow)
3. [Project Directory Layout](#project-directory-layout)
4. [Source Files & Configuration](#source-files--configuration)
   - [docker-compose.yml](#docker-composeyml)
   - [Environment Configuration (.env)](#environment-configuration-env)
   - [Secrets Configuration](#secrets-configuration)
   - [Web Service Dockerfile](#web-service-dockerfile)
   - [Flask Application (app.py)](#flask-application-apppy)
   - [Dependencies (requirements.txt)](#dependencies-requirementstxt)
   - [Database Initialization (init.sql)](#database-initialization-initsql)
5. [Getting Started: Build & Run](#getting-started-build--run)
6. [Data Persistence Verification](#data-persistence-verification)
7. [Cleanup](#cleanup)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Security & Production Best Practices](#security--production-best-practices)
10. [Technical Comparison Tables](#technical-comparison-tables)
    - [PostgreSQL vs. MySQL](#postgresql-vs-mysql)
    - [Compose v2 vs. Compose v3 / Compose Spec](#compose-v2-vs-compose-v3--compose-spec)
11. [References](#references)

---

## Executive Summary

This project demonstrates the design and orchestration of a multi-container stack configured with standard production patterns:
* **Multi-Stage Builds & Least Privilege:** A lean Python 3.11 image executing under a non-root system user (`appuser`).
* **Declarative Secrets Management:** Database credentials provided via Docker secrets (`_FILE` injection), avoiding cleartext credential leakage in environment variables or images.
* **Resilient Lifecycle & Observability:** Custom container health checks leveraging `curl` and `pg_isready`, paired with restart policies and compute resource constraints.
* **Persistent Storage & Isolation:** Dedicated named volumes (`db-data`, `web-data`) and a bridge network (`appnet`) for internal service discovery.

---

## Architecture & Workflow

### Service Architecture

```
                       +-----------------------+
                       |      User / Client    |
                       +-----------+-----------+
                                   | HTTP :8080
                                   v
+=============================== appnet ===============================+
|                                                                      |
|  +---------------------------+       +----------------------------+  |
|  |     web (Flask App)       |       |       db (PostgreSQL)      |  |
|  |  Internal Port: 5000      |       |    Internal Port: 5432     |  |
|  |  User: appuser (non-root) | ----> |    pg_isready healthcheck  |  |
|  +-------------+-------------+  DNS  +--------------+-------------+  |
|                |                                    |                |
+================|====================================|================+
                 |                                    |
                 v Mount                              v Mount
        [ web-data Volume ]                  [ db-data Volume ]
        (/app/uploads)                       (/var/lib/postgresql/data)
```

### Lifecycle Timeline

```mermaid
flowchart LR
    A[Code & Config Files] --> B[docker compose up -d --build]
    B --> C[Build Images: Flask & Pull Postgres]
    C --> D[Initialize 'appnet' Network & Volumes]
    D --> E[Start Containers: web & db]
    E --> F[Run Healthchecks: /health & pg_isready]
    F --> G[Test Persistence: /upload & SQL INSERT]
    G --> H[Stop Containers: docker compose down]
    H --> I[Recreate Containers: docker compose up -d]
    I --> J[Verify Volume State Preserved]
```

---

## Project Directory Layout

```text
.
├── .env
├── docker-compose.yml
├── Dockerfile
├── init.sql
├── app.py
├── requirements.txt
└── secrets/
    └── db_password.txt
```

---

## Source Files & Configuration

### docker-compose.yml

```yaml
version: "3.8"

services:
  web:
    build: .
    image: flask-app:latest
    ports:
      - "8080:5000"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=${POSTGRES_DB}
      - DB_USER=${POSTGRES_USER}
      - DB_PASSWORD_FILE=/run/secrets/db_password
    depends_on:
      - db
    volumes:
      - web-data:/app/uploads
    networks:
      - appnet
    restart: unless-stopped
    secrets:
      - db_password
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M
      restart_policy:
        condition: on-failure

  db:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - appnet
    restart: unless-stopped
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 10s
      retries: 5

networks:
  appnet:

volumes:
  db-data:
  web-data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

### Environment Configuration (.env)

```dotenv
POSTGRES_USER=flaskuser
POSTGRES_DB=flaskdb
```

---

### Secrets Configuration

Generate the secret securely on your local file system:

```bash
mkdir -p secrets
echo "supersecret" > secrets/db_password.txt
chmod 600 secrets/db_password.txt
```

---

### Web Service Dockerfile

```dockerfile
# Stage 1: Dependency resolution and compilation
FROM python:3.11-slim AS builder

WORKDIR /app

# Install curl for healthcheck down the line and build tools if needed
RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Minimal runtime image
FROM python:3.11-slim

# Install runtime curl for health checks
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -u 1000 appuser

WORKDIR /app

# Copy installed Python packages from builder stage
COPY --from=builder /root/.local /home/appuser/.local
COPY app.py requirements.txt ./

# Ensure correct binary resolution and file ownership
ENV PATH=/home/appuser/.local/bin:$PATH
RUN chown -R appuser:appuser /app

USER appuser

ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --retries=5 \
  CMD curl -f http://localhost:5000/health || exit 1

CMD ["flask", "run"]
```

---

### Flask Application (app.py)

```python
import os
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Flask!", 200

@app.route('/health')
def health():
    return jsonify(status="OK"), 200

@app.route('/upload')
def upload():
    directory = '/app/uploads'
    os.makedirs(directory, exist_ok=True)
    file_path = os.path.join(directory, 'test.txt')
    with open(file_path, 'a') as f:
        f.write("This file is persisted!\n")
    return f"Wrote file to {file_path}", 200

if __name__ == '__main__':
    app.run()
```

---

### Dependencies (requirements.txt)

```text
Flask==2.3.2
```

---

### Database Initialization (init.sql)

```sql
-- Executed on initial container setup by Postgres entrypoint
CREATE TABLE IF NOT EXISTS messages (
  id SERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO messages (content) VALUES ('Hello from init script!');
```

---

## Getting Started: Build & Run

1. **Start the multi-container stack in detached mode:**
   ```bash
   docker compose up -d --build
   ```

2. **Verify container lifecycle and health states:**
   ```bash
   docker compose ps
   ```

3. **Inspect container logs:**
   ```bash
   docker compose logs --tail=20
   ```

4. **Verify HTTP Web Service Health:**
   ```bash
   curl -i http://localhost:8080/health
   ```
   *Expected Response:*
   ```http
   HTTP/1.1 200 OK
   Content-Type: application/json
   ...
   {"status":"OK"}
   ```

5. **Verify Database Initialization via `psql`:**
   ```bash
   export PGPASSWORD=$(< secrets/db_password.txt)
   psql -h localhost -p 5432 -U flaskuser -d flaskdb -c '\dt'
   psql -h localhost -p 5432 -U flaskuser -d flaskdb -c "SELECT * FROM messages;"
   ```

---

## Data Persistence Verification

### Volume Persistence Test (Web Service)

1. **Trigger file creation inside the mounted upload directory:**
   ```bash
   curl http://localhost:8080/upload
   ```
2. **Tear down running containers:**
   ```bash
   docker compose down
   ```
3. **Re-launch containers:**
   ```bash
   docker compose up -d
   ```
4. **Assert file contents within the recreation lifecycle:**
   ```bash
   docker compose exec web cat /app/uploads/test.txt
   ```
   Output: `This file is persisted!`

### Database Volume Verification

Query the database after tearing down and restarting the stack:
```bash
export PGPASSWORD=$(< secrets/db_password.txt)
psql -h localhost -p 5432 -U flaskuser -d flaskdb -c "SELECT * FROM messages;"
```
All initial records and subsequent mutations remain intact within the `db-data` volume.

---

## Cleanup

To stop containers, release internal networks, and prune named volumes:

```bash
# Stop and remove containers, networks, and persistent volumes
docker compose down -v
```

> **Warning:** The `-v` flag removes named volumes (`db-data` and `web-data`). Omit `-v` if data retention is desired across project resets.

---

## Troubleshooting Guide

| Issue / Error | Likely Root Cause | Remediation Step |
| :--- | :--- | :--- |
| **`bind: address already in use`** | Host port `8080` or `5432` is occupied by another process. | Change the host mapping in `docker-compose.yml` (e.g., `"8081:5000"`) or stop the occupying process (`lsof -i :5432`). |
| **`curl: (7) Failed to connect`** | Service still bootstrapping or container stopped. | Check logs via `docker compose logs web`. Ensure Flask is bound to `0.0.0.0`, not `127.0.0.1`. |
| **`FATAL: password authentication failed`** | Mismatch between secret file content and cached volume password. | Delete old volume (`docker volume rm <project>_db-data`) if passwords were altered after the first initialization. |
| **Permission Denied (`/app/uploads`)** | Non-root `appuser` cannot write to root-owned volume mount. | Ensure destination directory exists and is chowned to `1000:1000` or created within the application context. |
| **Container shows `(unhealthy)`** | Healthcheck tool missing or endpoint timing out. | Ensure `curl` is present in the final Dockerfile stage and verifies HTTP 200 within the 10s timeout window. |

---

## Security & Production Best Practices

* **Run as Non-Root:** Containers should never execute as `root`. We provision `appuser` (UID 1000) inside the Dockerfile.
* **Avoid Plaintext Secrets:** Use Compose secrets instead of raw environment variables to prevent credentials leaking through `docker inspect` or process table inspections (`ps aux`).
* **Pin Image Digests/Tags:** Avoid `latest` in production manifests. Pin precise versions (e.g., `postgres:15.3-alpine`) to ensure deterministic builds.
* **Limit Compute Resources:** Set strict limits on CPU (`0.50`) and memory (`512M`) to prevent cascading resource starvation across co-located containers.
* **Network Segmentation:** Never expose database ports (`5432:5432`) to the public host in production; restrict database ingress solely to the private bridge network (`appnet`).

---

## Technical Comparison Tables

### PostgreSQL vs. MySQL

| Feature / Dimension | PostgreSQL | MySQL (InnoDB) |
| :--- | :--- | :--- |
| **License** | Open Source (PostgreSQL License) | Dual-licensed (GPL / Proprietary by Oracle) |
| **SQL Standards Support** | Extremely high conformance (ANSI SQL, CTEs, Window functions) | Moderate conformance; historically permissive syntax |
| **Extensibility** | Native procedural languages, custom types, extensions (`PostGIS`, `pg_stat_statements`) | Pluggable storage engines; limited runtime extension mechanisms |
| **Concurrency Model** | Multi-Version Concurrency Control (MVCC) without table-level rollback locks | MVCC relying on Undo Logs inside the `ibdata` tablespace |
| **Complex Queries & Analytics**| Highly optimized hash joins, merge joins, and parallel query plans | Historically optimized for simpler nested-loop lookups |
| **Ideal Workloads** | Complex schemas, high analytical requirements, strict ACID integrity | Standard web CRUD workloads, LAMP/LEMP stacks, read-heavy patterns |

---

### Compose v2 vs. Compose v3 / Compose Spec

| Dimension | Docker Compose v2 (Legacy) | Compose v3 / Specification |
| :--- | :--- | :--- |
| **Target Runtime** | Single-host Docker Engine | Swarm Mode & Unified Compose Specification |
| **Resource Limits** | Native support via `mem_limit`, `cpus` directly under service | Defined under `deploy.resources.limits` (Standardized across Swarm & Compose v2+ CLI) |
| **Condition-based `depends_on`**| Supported (`service_healthy`, `service_started`) | Deprecated in classic v3; re-introduced in modern Compose Specification |
| **Secrets Implementation** | Local host mount simulation | Native Swarm secrets and local file-backed secrets engine |
| **CLI Invocation** | Python standalone binary (`docker-compose`) | Native Go Docker CLI plugin (`docker compose`) |

---

## References

* [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
* [Official PostgreSQL Docker Library](https://hub.docker.com/_/postgres)
* [Flask Production Deployment Considerations](https://flask.palletsprojects.com/en/latest/deploying/)
* [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
