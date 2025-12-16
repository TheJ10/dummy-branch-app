# Branch Loan API – DevOps Setup

This repository contains a production-ready DevOps setup for the Branch Loan API.
The goal of this project is to containerize the existing API, enable secure HTTPS access, support multiple environments, and automate build and release using CI/CD.

The API is a backend-only service (no frontend UI) that exposes REST endpoints for managing microloans. This setup focuses on reliability, security, and reproducibility across environments.

---

## Architecture
The application is composed of three main services running in Docker containers:
```text
         Browser / curl
               |
           HTTPS (443)
               v
        +-------------+
        |    Nginx    |
        | (TLS Proxy) |
        +-------------+
               |
           HTTP (8000)
               v
+-------------+ +--------------------+
| Flask API | <----> | PostgreSQL DB |
| (Gunicorn) | | (Persistent)        |
+-------------+ +--------------------+
```

### Flow
- Requests are made to `https://branchloans.com`
- Nginx terminates HTTPS using a self-signed certificate (local development)
- Traffic is proxied to the Flask API container
- The API communicates with PostgreSQL over Docker’s internal network
- Database schema is managed using Alembic migrations

---

## Prerequisites
Ensure the following are installed:

- Docker (v20+)
- Docker Compose v2
- Git
- OpenSSL

Tested on Windows/Linux using Docker Desktop.

---

## Local Setup (HTTPS Enabled)

### 1. Clone the repository
```bash
git clone https://github.com/<your-username>/dummy-branch-app.git
cd dummy-branch-app	
```

### 2. Configure Local Domain
Add the following entry to your hosts file:
```bash
127.0.0.1 branchloans.com
```

**Locations:**
- **Windows:** `C:\Windows\System32\drivers\etc\hosts`
- **Linux/macOS:** `/etc/hosts`

This allows accessing the API via [https://branchloans.com](https://branchloans.com).

---

### 3. Generate Self-Signed TLS Certificate (Local Only)
Create a directory for certificates:
```bash
mkdir -p certs
```

Generate the certificate:
```bash
openssl req -x509 -nodes -days 365
-newkey rsa:2048
-keyout certs/branchloans.key
-out certs/branchloans.crt
-subj "/C=IN/ST=KA/L=Bangalore/O=Branch/OU=Dev/CN=branchloans.com"
```

> **Note:** Browsers will show a warning for self-signed certificates. This is expected for local development.

---

### 4. Start the Application
Build and start services with Docker Compose:
```bash
docker compose up -d --build
```

This starts the following containers:
- **PostgreSQL database**
- **Flask API (via Gunicorn)**
- **Nginx reverse proxy (HTTPS)**

Verify containers:
```bash
docker compose ps
```

---

### 5. Apply Database Migrations
```bash	
docker compose exec api alembic upgrade head
```

---

### 6. Verify API Access

#### Browser
- [https://branchloans.com/health](https://branchloans.com/health)
- [https://branchloans.com/api/loans](https://branchloans.com/api/loans)

#### Curl
```bash
curl -k https://branchloans.com/api/loans
```

---

### Environment Configuration
The project supports multiple environments using environment variables.

**Files:**
- `.env.example` – sample variables (committed)
- `.env.dev` – local development (not committed)
- `.env.staging` / `.env.prod` – future environments

**Switching environments:**
```bash
ENV_FILE=.env.dev docker compose up -d
```

**Approach:**
- Avoids hardcoding secrets 
- Keeps one compose file 
- Matches production practices 

---

### Database Migrations & Seeding (Demo)

#### Migrations
```bash
docker compose exec api alembic upgrade head
```

#### Seeding (Demo Only)
Insert sample loan data:
```bash
docker compose exec api python scripts/seed.py
```

> This is not used in CI or production. 
> In real deployments, data is created via API calls.

---

### CI/CD Pipeline
Continuous integration and deployment are implemented using **GitHub Actions**.

**Trigger conditions:**
- Push to `main`
- Pull requests

**Pipeline stages:**
1. Checkout code 
2. Install Python dependencies 
3. Run tests (if present) 
4. Build Docker image 
5. Scan image for **CRITICAL** vulnerabilities (via Trivy) 
6. Push image to **GitHub Container Registry (GHCR)** 

Images are tagged using the **Git commit SHA** for traceability.
Images are pushed only on successful pushes to main; pull requests do not publish images.

---

### Security
- No secrets stored in code 
- Uses GitHub-provided `GITHUB_TOKEN` 
- Pipeline fails on **CRITICAL** vulnerabilities 
- HTTPS enabled via **Nginx** 
- Backend API not directly exposed 
- Secrets managed via environment variables 
- Database credentials not committed 
- Vulnerability scanning enforced in CI 

---

### Monitoring & Observability

-At this stage, observability is provided through application logs and container logs
available via Docker.

-In a production environment, this setup could be extended with centralized logging,
metrics collection, and alerting using tools such as Prometheus, Grafana, ELK stack,
or cloud-native monitoring services.

-Health check endpoints (`/health`) are exposed to support uptime monitoring and
service readiness checks.


---

### Troubleshooting
**Containers not running**
```bash
docker compose ps
docker compose up -d
```

**Database errors**
```bash
docker compose exec api alembic upgrade head
```

**HTTPS issues**
- Ensure `branchloans.com` is mapped to `127.0.0.1`
- Accept self-signed certificate warning in browser

**Empty `/api/loans` response**
- Run seed script for demo data
- Empty list is expected on fresh databases

---

### Design Decisions & Trade-offs

**Docker & Docker Compose**
- Simplifies setup and ensures portability.

**Nginx for HTTPS**
- Simulates real-world TLS termination used in production (ALB / Ingress).

**Environment-based configuration**
- Enables the same setup for dev, staging, and production.

**GitHub Container Registry (GHCR)**
- Provides integrated authentication and clean deployment workflow.

**Trade-offs**
- Self-signed certificates used for local development only
- No frontend UI (API-only service)

---

### Future Improvements
- Production TLS via managed certificates 
- Centralized logging and metrics 
- Kubernetes deployment 
- Automated database migrations in release pipeline 

---

### Summary
This project demonstrates:

- **Production-ready containerization** 
- **Secure HTTPS setup** 
- **Multi-environment configuration** 
- **Automated CI/CD with security scanning** 
- **Clear, reproducible documentation**




