# Kora Analytics API — DeployReady

A production-ready containerised deployment of the Kora Analytics Node.js API,
built as part of the AmaliTech DevOps engineering challenge.

---

## Note on AWS Deployment

AWS account registration requires a bank card for identity verification, which
was not available during this challenge. This was communicated to the AmaliTech
team (Kevin Rukundo), who confirmed that submitting completed parts is
acceptable.

All infrastructure code is fully written and ready — the pipeline is configured
to deploy to EC2 automatically once `EC2_HOST`, `EC2_USER`, and `EC2_SSH_KEY`
secrets are provided.

---

## What Was Built

| Part | Deliverable                                                       | Status |
| ---- | ----------------------------------------------------------------- | ------ |
| 1    | `Dockerfile` — multi-stage, non-root user, healthcheck            | ✅     |
| 1    | `docker-compose.yml` — local orchestration via `.env`             | ✅     |
| 2    | `.github/workflows/deploy.yml` — Test → Build → Push → Deploy     | ✅     |
| 2    | Image pushed to GitHub Container Registry (GHCR)                  | ✅     |
| 2    | Automatic rollback if `/health` fails after deploy                | ✅     |
| 3    | EC2 + Security Group + Docker setup (documented in DEPLOYMENT.md) | ✅     |

---

## Architecture

```
Developer pushes to main
        │
        ▼
┌─────────────────────────────────┐
│     GitHub Actions Pipeline     │
│                                 │
│  1. Test  → npm test (Jest)     │
│     fail  → pipeline stops      │
│     pass  ▼                     │
│  2. Build → Docker image + SHA  │
│     ▼                           │
│  3. Push  → ghcr.io/<owner>/    │
│             kora-analytics-api  │
│     ▼                           │
│  4. Deploy → SSH into EC2       │
│     ├── pull new image          │
│     ├── restart container       │
│     ├── check /health → 200 ✓  │
│     └── 200 ✗ → rollback        │
└─────────────────────────────────┘
        │
        ▼
┌──────────────────────┐
│   AWS EC2 t2.micro   │
│   Amazon Linux 2023  │
│   Docker             │
│   Port 80 → 3000     │
└──────────────────────┘
```

- **Registry:** GitHub Container Registry (GHCR)
- **Image tags:** Git commit SHA (immutable) + `latest` (convenience)
- **Rollback:** Automatic if `/health` doesn't return HTTP 200 after deploy

---

## Quick Start (Local)

```bash
# 1. Clone the repo
git clone https://github.com/PattyWambere/AmaliTech-DEG-Project-based-challenges-fork.git
cd AmaliTech-DEG-Project-based-challenges-fork/dev-ops/DeployReady

# 2. Create your .env file
cp .env.example .env

# 3. Build and start
docker compose up --build
```

Test the three endpoints:

```bash
# Health check
curl http://localhost:3000/health
# {"status":"ok"}

# Metrics
curl http://localhost:3000/metrics
# {"uptime_seconds":12,"memory_mb":42,"node_version":"v20.x.x"}

# Data echo
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
│   ├── index.js              # Express API (unchanged)
│   ├── index.test.js         # Jest + Supertest tests
│   └── package.json
├── .env.example              # Environment variable template (PORT=3000)
├── .gitignore                # Excludes .env, .pem, terraform state
├── docker-compose.yml        # Local development orchestration
├── Dockerfile                # Multi-stage build (deps → test → production)
├── DEPLOYMENT.md           # AWS setup and operations guide
└── README.md                 # This file
```

---

## CI/CD Pipeline

The GitHub Actions workflow runs automatically on every push to `main`:

1. **Test** — Runs `npm test` via Jest + Supertest. Any failure stops the
   pipeline immediately — nothing gets built or deployed.
2. **Build** — Builds the Docker image using the multi-stage Dockerfile,
   tagged with the Git commit SHA for full traceability.
3. **Push** — Pushes the image to `ghcr.io/pattywambere/kora-analytics-api`
   using the built-in `GITHUB_TOKEN` — no extra credentials needed.
4. **Deploy** — SSHs into EC2, pulls the new image, restarts the container,
   verifies `/health` returns 200, and rolls back automatically if it doesn't.

### GitHub Secrets Required

| Secret        | Value                                        |
| ------------- | -------------------------------------------- |
| `EC2_HOST`    | EC2 public IP address                        |
| `EC2_USER`    | `ec2-user`                                   |
| `EC2_SSH_KEY` | Full contents of the `.pem` private key file |

---

## Key Decisions

**Multi-stage Dockerfile** — Three stages: `deps` (install), `test` (run Jest
inside the build so a failing test aborts the image), and `production` (lean
final image with only what's needed to run). This keeps the production image
small and free of test tooling.

**Non-root container user** — A dedicated `appuser` is created and the
container switches to it before starting. Running as root inside a container
is a known security risk.

**Commit SHA image tags** — Every image is immutable and tied to a specific
commit. This makes rollbacks deterministic — you always know exactly what code
is running.

**GHCR over ECR** — GitHub Container Registry integrates natively with
`GITHUB_TOKEN`, requiring zero additional AWS permissions and reducing the
overall secret surface area.

**Automatic rollback** — After every deploy the pipeline checks `/health`. A
non-200 response immediately restores the previous image, keeping downtime
under 30 seconds without any human intervention.

---

## Documentation

See [DEPLOYMENT.md](./DEPLOYMENT.md) for the full AWS setup guide, Docker
installation steps, and operational runbook.
