# Two-Tier Flask & MySQL Application with Automated CI/CD & Self-Healing Rollbacks

A production-grade, containerized two-tier web application built with Flask and MySQL, orchestrated via Docker Compose, and deployed to an AWS EC2 instance. The project incorporates an automated CI/CD pipeline utilizing GitHub Actions, DevSecOps vulnerability gating with Trivy, and an automated health verification and rollback engine.

---

## Architecture Overview

```
                                      +---------------------------------------------+
                                      |                 GitHub                      |
                                      |  1. Push / Pull Request                     |
                                      |  2. Lint & Unit Tests                       |
                                      |  3. Trivy Security Vulnerability Scan       |
                                      |  4. Build & Push Versioned Image (SHA Tag)  |
                                      +----------------------+----------------------+
                                                             |
                                                             | Docker Hub Registry
                                                             v
+------------------------------------------------------------+-------------------------------------+
| AWS EC2 Instance                                                                                 |
|                                                                                                  |
|   +-------------------------------------------------------------------------------------------+  |
|   | CD Deployment Engine (appleboy/ssh-action)                                                |  |
|   |  - Authenticate to Docker Hub                                                             |  |
|   |  - Checkpoint current running image tag ($PREV_TAG)                                       |  |
|   |  - Pull new release ($TARGET_TAG)                                                         |  |
|   |  - Targeted container restart: `docker compose up -d flask-app`                           |  |
|   |  - 60-Second Active Health Polling Loop (`/health`)                                       |  |
|   |  - Automated Self-Healing Rollback on Failure                                             |  |
|   +-------------------------------------------------------------------------------------------+  |
|                                                                                                  |
|   +--------------------------+  Bridge Network   +--------------------------------------------+  |
|   |        Flask App         | <===============> |                 MySQL 8.4                  |  |
|   |   (Stateless, Port 5000) |    `twotier`      |   (Stateful, Persistent Volume: mysql-data)|  |
|   +--------------------------+                   +--------------------------------------------+  |
+--------------------------------------------------------------------------------------------------+
```

---

## Key Features

- **Multi-Tier Orchestration:** Decoupled stateless application tier (`flask-app`) and stateful database tier (`mysql`) connected via custom Docker bridge network (`twotier`).
- **Data Persistence:** Dedicated named volume (`mysql-data`) ensures database records and seed data (`message.sql`) persist across container lifecycles.
- **DevSecOps Integration:** Static vulnerability scanning integrated into CI using Trivy, scanning dependencies and OS base layers to block critical CVEs prior to registry push.
- **Deterministic Artifact Versioning:** Container images tagged with short Git commit SHAs alongside `:latest` for traceability and auditability.
- **State-Preserving Continuous Deployment:** CD pipeline selectively recreates only the stateless application tier (`docker compose up -d flask-app`), leaving the MySQL database running without interruption.
- **Automated Self-Healing Rollback:** Post-deployment polling loop interrogates the `/health` endpoint over a 60-second window. In the event of a runtime failure or unhandled exception, the pipeline automatically redeploys the previous stable image checkpoint tag and flags the build as failed.

---

## Tech Stack

- **Application:** Python 3, Flask, PyMySQL
- **Database:** MySQL 8.4
- **Containerization:** Docker, Docker Compose v2
- **CI/CD:** GitHub Actions
- **Security:** Aqua Security Trivy
- **Cloud Infrastructure:** AWS EC2 (Ubuntu 24.04 / 22.04 LTS)
- **Scripting & Automation:** Bash, POSIX Shell

---

## Directory Structure

```text
.
├── .github/
│   └── workflows/
│       ├── ci.yml                 # Build, test, security scan, and push
│       └── cd.yml                 # SSH deployment, health probe, and rollback loop
├── app/                           # Flask application source code
│   ├── app.py
│   └── requirements.txt
├── tests/                         # Unit tests
│   └── test_app.py
├── message.sql                    # Initial database schema and seed data
├── Dockerfile                     # Multi-stage Docker build file
├── docker-compose.yml             # Service definition and orchestration
└── README.md
```

---

## CI/CD Pipeline Details

### 1. Continuous Integration (`ci.yml`)
Triggered on pull requests and pushes to `main`:
1. **Lint and Test:** Runs code checks and pytest unit tests inside a Python virtual environment.
2. **Compute Commit SHA:** Computes short Git SHA (`git rev-parse --short HEAD`) for artifact tagging.
3. **Local Container Build:** Assembles the Flask runtime image.
4. **Trivy Vulnerability Scan:** Scans the generated image for OS and library CVEs, blocking on severe issues (`CRITICAL,HIGH`).
5. **Registry Push:** Authenticates to Docker Hub and pushes two image tags:
   - `<username>/two-tier-flask-app:<short-sha>`
   - `<username>/two-tier-flask-app:latest`

### 2. Continuous Deployment (`cd.yml`)
Triggered automatically via `workflow_run` upon successful completion of the CI pipeline on `main` (also supports manual trigger via `workflow_dispatch`):
1. Establishes a secure SSH session to the AWS EC2 instance.
2. Synchronizes local repository files using `git fetch` and `git reset --hard origin/main`.
3. Checks and captures the currently active container tag (`PREV_TAG`) as a rollback target.
4. Pulls the target versioned image from Docker Hub.
5. Executes `docker compose up -d flask-app` to perform an in-place container swap.
6. **Active Health Check Loop:** Interrogates `http://127.0.0.1:5000/health` up to 12 times at 5-second intervals.
7. **Automated Rollback:** If the check fails to return `HTTP 200`, the script redeploys `$PREV_TAG`, re-verifies health, and exits with status code `1`.
8. Prunes orphaned images (`docker image prune -f`) to conserve host disk space.

---

## Setup & Deployment Instructions

### Prerequisites
- AWS EC2 Instance (Ubuntu) with inbound TCP port `5000` and `22` (SSH) open in Security Groups.
- Docker and Docker Compose v2 installed on EC2.
- Docker Hub account with an active Personal Access Token (PAT).

### 1. Required GitHub Repository Secrets

Configure the following secrets under **Settings > Secrets and variables > Actions**:

| Secret Name | Description | Example |
|---|---|---|
| `EC2_HOST` | Public IP or Public DNS of the EC2 instance | `54.210.12.34` |
| `EC2_USER` | SSH username | `ubuntu` |
| `EC2_SSH_KEY` | Private SSH Key (`.pem` file content) | `-----BEGIN RSA PRIVATE KEY-----...` |
| `DOCKERHUB_USERNAME` | Docker Hub username | `sahilsutar` |
| `DOCKERHUB_TOKEN` | Docker Hub Personal Access Token | `dckr_pat_...` |

---

### 2. EC2 Host Preparation

SSH into the EC2 instance and set up the repository and dependencies:

```bash
# 1. Update package index and install Docker
sudo apt-get update
sudo apt-get install -y docker.io curl git
sudo usermod -aG docker ubuntu
newgrp docker

# 2. Install Docker Compose v2 CLI plugin
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose

# Verify Docker Compose installation
docker compose version

# 3. Clone repository on EC2
cd /home/ubuntu
git clone https://github.com/<your-username>/two-tier-flask-app.git
cd two-tier-flask-app
```

---

### 3. Local Development & Testing

To run the full stack locally:

```bash
# Set default image build environment variables
export DOCKER_USERNAME="local"
export APP_TAG="dev"

# Build and start services in the background
docker compose up -d --build

# Verify running services
docker compose ps

# Test the health endpoint
curl -i http://localhost:5000/health

# Stop the stack (preserving volumes)
docker compose down
```

---

## Health Check & Rollback Verification

The Flask application exposes a designated health check endpoint:

- **Endpoint:** `GET /health`
- **Response:**
  ```json
  {
    "status": "healthy",
    "database": "connected"
  }
  ```

If the database is unreachable or an internal runtime exception occurs, the endpoint responds with an `HTTP 500` status code, triggering the self-healing routine during pipeline deployment.
