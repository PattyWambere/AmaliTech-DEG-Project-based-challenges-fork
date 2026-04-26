# DEPLOYMENT.md — Kora Analytics API

---

## Note on AWS Deployment

AWS account registration requires a bank card for identity verification, which
was not available during this challenge. This was communicated directly to the
AmaliTech team (Kevin Rukundo), who confirmed that submitting completed parts
is acceptable.

The full deployment infrastructure is written and ready:

- The pipeline is fully configured to deploy to EC2 on every push to `main`
- It will work automatically once three GitHub Secrets are added:
  `EC2_HOST`, `EC2_USER`, and `EC2_SSH_KEY`
- All steps below document exactly how the EC2 instance would be set up

---

## 1. EC2 Instance Setup

### Launch via AWS Console

1. Sign in to the [AWS Console](https://console.aws.amazon.com) and go to
   **EC2 → Launch Instance**
2. Configure the instance:

   | Field         | Value                                             |
   | ------------- | ------------------------------------------------- |
   | Name          | `kora-analytics-api`                              |
   | AMI           | Amazon Linux 2023 (64-bit x86)                    |
   | Instance type | `t2.micro` (Free Tier eligible)                   |
   | Key pair      | Create new → `kora-key` → RSA → `.pem` → Download |

3. Under **Network settings → Edit**, configure the Security Group:

   | Type | Protocol | Port | Source         | Purpose                          |
   | ---- | -------- | ---- | -------------- | -------------------------------- |
   | SSH  | TCP      | 22   | `<your-ip>/32` | Your IP only — never `0.0.0.0/0` |
   | HTTP | TCP      | 80   | `0.0.0.0/0`    | Public API access                |

   > Find your IP at https://checkip.amazonaws.com

4. Click **Launch Instance**

### IAM — Least-privilege user for the pipeline

1. Go to **IAM → Users → Create user** → name it `kora-pipeline`
2. Attach this inline policy (minimum permissions needed):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

> This project uses GitHub Container Registry (GHCR) so no ECR permissions
> are needed. The policy above is included for completeness if switching
> registries in the future.

---

## 2. Installing Docker and Running the Container

### SSH into the instance

```bash
# Fix key file permissions (required on Linux/Mac)
chmod 400 ~/Downloads/kora-key.pem

# Connect
ssh -i ~/Downloads/kora-key.pem ec2-user@<EC2_PUBLIC_IP>
```

### Install Docker

```bash
# Update all packages
sudo dnf update -y

# Install Docker
sudo dnf install -y docker

# Start Docker and enable it to auto-start on reboot
sudo systemctl start docker
sudo systemctl enable docker

# Add ec2-user to the docker group (avoids needing sudo for docker commands)
sudo usermod -aG docker ec2-user

# Apply the group change in the current session
newgrp docker

# Verify Docker is working
docker --version
```

### Pull and run the image (first-time manual bootstrap)

```bash
# Log in to GitHub Container Registry
# Create a PAT at: GitHub → Settings → Developer settings →
# Personal access tokens → Fine-grained → read:packages scope
echo "<GITHUB_PAT>" | docker login ghcr.io -u <GITHUB_USERNAME> --password-stdin

# Pull the latest image
docker pull ghcr.io/pattywambere/kora-analytics-api:latest

# Run the container
# Maps EC2 port 80 → container port 3000
docker run -d \
  --name kora-analytics-api \
  --restart unless-stopped \
  -p 80:3000 \
  -e PORT=3000 \
  ghcr.io/pattywambere/kora-analytics-api:latest
```

After this one-time bootstrap, all future deploys are handled automatically
by the GitHub Actions pipeline on every push to `main`.

### Add GitHub Secrets for the pipeline

Go to your repo → **Settings → Secrets and variables → Actions** and add:

| Secret        | Value                                    |
| ------------- | ---------------------------------------- |
| `EC2_HOST`    | EC2 public IPv4 address                  |
| `EC2_USER`    | `ec2-user`                               |
| `EC2_SSH_KEY` | Full contents of the `kora-key.pem` file |

---

## 3. Checking if the Container is Running

```bash
# List all running containers
docker ps

# Check this specific container's status and health
docker inspect \
  --format='Status: {{.State.Status}} | Health: {{.State.Health.Status}}' \
  kora-analytics-api

# Quick HTTP health check directly on the server
curl http://localhost/health
# Expected response: {"status":"ok"}

# Verify from your local machine
curl http://<EC2_PUBLIC_IP>/health
# Expected response: {"status":"ok"}
```

---

## 4. Viewing Application Logs

```bash
# Stream live logs (Ctrl+C to stop)
docker logs -f kora-analytics-api

# Show the last 100 lines
docker logs --tail 100 kora-analytics-api

# Show logs with timestamps
docker logs -t kora-analytics-api

# Show logs from the last 30 minutes
docker logs --since 30m kora-analytics-api
```

---

## 5. Pipeline Architecture

```
Developer pushes to main
        │
        ▼
┌──────────────────────────────────────────────────────┐
│               GitHub Actions Pipeline                │
│                                                      │
│  1. TEST                                             │
│     └── npm test (Jest + Supertest)                  │
│         fail → pipeline stops, nothing deploys       │
│         pass ▼                                       │
│                                                      │
│  2. BUILD                                            │
│     └── docker build (multi-stage)                   │
│         tagged: ghcr.io/.../kora-analytics-api:SHA   │
│         tagged: ghcr.io/.../kora-analytics-api:latest│
│         ▼                                            │
│  3. PUSH                                             │
│     └── push both tags to GHCR                      │
│         ▼                                            │
│  4. DEPLOY                                           │
│     ├── SSH into EC2                                 │
│     ├── docker pull <new image>                      │
│     ├── docker stop + rm old container               │
│     ├── docker run <new image> -p 80:3000            │
│     ├── sleep 15s                                    │
│     ├── curl /health → HTTP 200? ✓ → done            │
│     └── HTTP 200? ✗ → rollback to previous image     │
└──────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│     AWS EC2 t2.micro     │
│     Amazon Linux 2023    │
│     Docker               │
│     Port 80  → 3000      │
│     SG: 80   0.0.0.0/0   │
│     SG: 22   your-IP only│
└──────────────────────────┘
```

---

## 6. Bonus — Automatic Rollback

The deploy job in `.github/workflows/deploy.yml` implements automatic rollback:

1. Before replacing the running container, the pipeline saves the current
   image tag using `docker inspect`
2. After starting the new container it waits 15 seconds then hits `/health`
3. If the response is not HTTP 200:
   - The broken container is stopped and removed
   - The previous image is started again automatically
   - The pipeline exits with a failure status so the team is alerted via
     GitHub notifications

This means a bad deploy can never leave the API fully down. The worst case
is a ~15 second window before the rollback completes, rather than a full
outage waiting for manual intervention.

---

## 7. Common Operational Commands

| Task                      | Command                                                     |
| ------------------------- | ----------------------------------------------------------- |
| Restart container         | `docker restart kora-analytics-api`                         |
| Stop container            | `docker stop kora-analytics-api`                            |
| Pull a specific tag       | `docker pull ghcr.io/pattywambere/kora-analytics-api:<sha>` |
| Remove stopped containers | `docker container prune -f`                                 |
| Free unused images        | `docker image prune -f`                                     |
| Check disk usage          | `docker system df`                                          |
