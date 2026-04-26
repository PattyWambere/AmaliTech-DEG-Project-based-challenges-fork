# DeployReady

This challenge is designed to test the understanding of core DevOps practices: containerisation, automated pipelines, and cloud deployment.

---

## 1. Business Context

**Client:** Kora Analytics
**Industry:** SaaS — Data dashboards for logistics companies

### The Problem

Every time the Kora team wants to deploy a new version of their app, a developer manually SSHs into the server, pulls the code, and restarts the process by hand. There are no automated tests before a release and no way to tell if a deploy broke something until a customer complains.

### Your Role

You are joining as their first DevOps engineer. The application code already works — your job is to **containerise it, automate the delivery pipeline, and get it running on AWS**.

---

## 2. The Application

A simple Node.js API is provided in the [`app/`](./app/) directory. It has three endpoints:

| Method | Route      | Description                            |
| ------ | ---------- | -------------------------------------- |
| GET    | `/health`  | Returns `{ "status": "ok" }`           |
| GET    | `/metrics` | Returns uptime and memory usage        |
| POST   | `/data`    | Accepts a JSON body and echoes it back |

Run it locally:

```bash
cd app
npm install
npm start
```

Do not change the application logic. Your work is everything around it.

---

## 3. The Assignment

### Part 1 — Containerise the App

**Deliverables:** A `Dockerfile` and a `docker-compose.yml` in the root of your repository.

**Dockerfile requirements:**

- The app must run inside a Docker container.
- The container must accept a `PORT` environment variable.
- The container must **not** run as the `root` user.

**Docker Compose requirements:**

- Define the app as a service in `docker-compose.yml`.
- Map port `3000` on the host to the container.
- Pass the `PORT` variable via an `.env` file (include a `.env.example` with placeholder values).
- Running the fo# Kora Analytics API — DeployReady

A production-ready deployment of the Kora Analytics Node.js API, built as part
of the AmaliTech DevOps engineering challenge.

**Live endpoint:** `http://<EC2_PUBLIC_IP>/health`

---

## What Was Built

| Part  | Deliverable                                   | Status |
| ----- | --------------------------------------------- | ------ |
| 1     | `Dockerfile` + `docker-compose.yml`           | ✅     |
| 2     | `.github/workflows/deploy.yml` CI/CD pipeline | ✅     |
| 3     | AWS EC2 deployment + `DEPLOYMENT.md`          | ✅     |
| Bonus | Automatic rollback on failed health check     | ✅     |

---

## Architecture

```
GitHub push → Actions (Test → Build → Push → Deploy) → AWS EC2 (Docker)
```

- **Registry:** GitHub Container Registry (GHCR)
- **Images tagged** with the Git commit SHA for full traceability
- **Rollback:** If `/health` doesn't return 200 after deploy, the previous
  image is automatically restored

---

## Quick Start (Local)

```bash
# 1. Clone the repo
git clone https://github.com/<your-username>/DeployReady.git
cd DeployReady

# 2. Create your .env file
cp .env.example .env

# 3. Build and start
docker compose up --build
```

Test the endpoints:

```bash
curl http://localhost:3000/health
# {"status":"ok"}

curl http://localhost:3000/metrics
# {"uptime_seconds":12,"memory_mb":42,"node_version":"v20.x.x"}

curl -X POST http://localhost:3000/data \
  -H "Content-Type: application/json" \
  -d '{"shipment_id":"KOR-001","status":"in_transit"}'
# {"received":{"shipment_id":"KOR-001","status":"in_transit"}}
```

---

## Project Structure

```
DeployReady/
├── .github/
│   └── workflows/
│       └── deploy.yml        # CI/CD: test → build → push → deploy
├── app/
│   ├── index.js              # Express API (do not modify)
│   ├── index.test.js         # Jest + Supertest tests
│   └── package.json
├── .env.example              # Environment variable template
├── .gitignore
├── docker-compose.yml        # Local development orchestration
├── Dockerfile                # Multi-stage build (deps → test → production)
├── DEPLOYMENT.md             # AWS setup and operations guide
└── README.md                 # This file
```

---

## CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/deploy.yml`) runs on every
push to `main`:

1. **Test** — `npm test` via Jest. Failure stops the pipeline.
2. **Build** — Docker multi-stage build, tagged with the commit SHA.
3. **Push** — Image pushed to `ghcr.io/<owner>/kora-analytics-api`.
4. **Deploy** — SSH into EC2, pull new image, restart container, verify
   `/health`, rollback if unhealthy.

### Required GitHub Secrets

| Secret        | Description                        |
| ------------- | ---------------------------------- |
| `EC2_HOST`    | EC2 public IP address              |
| `EC2_USER`    | SSH username (`ec2-user`)          |
| `EC2_SSH_KEY` | Private key contents (`.pem` file) |

---

## Decisions Made

**Multi-stage Dockerfile** — Separates dependency installation, test execution,
and the final production image. The production image contains only the app
code and production `node_modules`, keeping it lean and without test tooling.

**Non-root container user** — A dedicated `appuser` is created inside the
container. Running as root inside a container is a security risk even with
namespace isolation.

**Commit SHA image tags** — Every image is immutable and tied to a specific
commit. This makes rollbacks deterministic and debugging straightforward.

**GHCR over ECR** — GitHub Container Registry requires no additional AWS
permissions and integrates natively with `GITHUB_TOKEN`, reducing the secret
surface area.

**Automatic rollback** — The deploy script checks `/health` after every deploy.
A non-200 response triggers an immediate rollback to the last known-good image,
minimising downtime without human intervention.

---

## Documentation

See [DEPLOYMENT.md](./DEPLOYMENT.md) for full AWS setup instructions,
operational runbooks, and the architecture diagram.llowing must start a working API:

```bash
docker compose up --build
```

---

### Part 2 — Automate the Pipeline

**Deliverable:** A `.github/workflows/deploy.yml` GitHub Actions workflow.

The pipeline must run these steps **in order** on every push to `main`:

1. **Test** — Run `npm test`. If tests fail, the pipeline stops. Nothing gets deployed.
2. **Build** — Build the Docker image and tag it with the Git commit SHA.
3. **Push** — Push the image to a container registry (GitHub Container Registry or AWS ECR).
4. **Deploy** — Pull the new image on the EC2 server and restart the container.

Additional requirements:

- Secrets (SSH key, registry token) must be stored as **GitHub repository secrets** — never in the code.
- Add a short comment above each step in the YAML explaining what it does.

---

### Part 3 — Deploy to AWS

**Deliverable:** A running service on AWS and a short `DEPLOYMENT.md` explaining your setup.

Provision the following manually (via the AWS Console is fine):

- An **EC2 instance** (`t2.micro`, Amazon Linux 2023) with Docker installed.
- A **Security Group** that allows:
  - HTTP on port 80 from anywhere
  - SSH on port 22 **from your IP only** — not `0.0.0.0/0`
- An **IAM user or role** for the pipeline with only the permissions it needs.

At submission time, `GET http://<your-ec2-ip>/health` must return `{ "status": "ok" }`.

Document in `DEPLOYMENT.md`:

- How you set up the EC2 instance
- How you installed Docker and pulled your image
- How to check if the container is running
- How to view the application logs

---

## 4. Bonus (Optional)

Pick **one** of the following if you want to go further:

- **Use Terraform** to provision the EC2 instance and Security Group instead of the console.
- **Add a CloudWatch alarm** that triggers if `/health` stops responding.
- **Implement a rollback step** in the pipeline that re-deploys the previous image if the health check fails after deploy.

Describe what you added and why in your `DEPLOYMENT.md`.

---

## 5. Submission Instructions

1. **Fork** this repository.
2. Complete all three parts in your fork.
3. **Replace this README** with your own documentation (architecture overview, setup steps, decisions made).
4. Submit your repo link via the [online form](https://forms.cloud.microsoft/e/f3FF83LVz3).

---

## ⚠️ Pre-Submission Checklist

- [ ] `docker compose up --build` starts the app locally
- [ ] A `.env.example` file is committed (the real `.env` is not)
- [ ] At least one successful pipeline run is visible in the GitHub Actions tab
- [ ] `GET /health` on your EC2 public IP returns 200
- [ ] No secrets or `.pem` files committed to the repository
- [ ] SSH port 22 is **not** open to `0.0.0.0/0`
- [ ] `DEPLOYMENT.md` is present and covers the four points in Part 3
- [ ] This README has been replaced with your own documentation
- [ ] Commit history shows progress over time (not a single upload commit)
