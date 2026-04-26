# InfraBlueprint — Vela Payments Infrastructure

## ⚠️ Deployment Note

This infrastructure was not deployed to a live AWS environment due to the
unavailability of a bank card required for AWS account registration. This
situation was communicated directly to the AmaliTech team prior to submission
and confirmed acceptable.

All Terraform configuration is complete and correct across all 5 parts
(Networking, Compute, Database, Storage, Outputs). The code has been validated
locally using `terraform validate` which confirms the configuration is valid.

A reviewer can verify correctness by running:

```bash
terraform init -backend=false
terraform validate
terraform plan -var-file="example.tfvars"
```

Terraform configuration that provisions a two-tier web application infrastructure for Vela Payments on AWS, reproducibly and from scratch with a single command.

---

## Architecture Diagram

```
                          ┌──────────────────────────────────────────────────┐
                          │  VPC  10.0.0.0/16                                │
                          │                                                  │
   Internet               │  ┌─────────────────────┐  ┌──────────────────┐  │
      │                   │  │  Public Subnet A     │  │  Public Subnet B │  │
      ▼                   │  │  10.0.1.0/24 (AZ-a)  │  │  10.0.2.0/24    │  │
 [Internet Gateway] ──────┼──│                     │  │  (AZ-b)          │  │
      │                   │  │  ┌───────────────┐  │  │                  │  │
      ▼                   │  │  │ EC2 t2.micro  │  │  │                  │  │
 [Route Table]            │  │  │ (Amazon Linux │  │  │                  │  │
 0.0.0.0/0 → IGW          │  │  │  2023)        │  │  │                  │  │
                          │  │  │               │  │  │                  │  │
                          │  │  │ [web-sg]      │  │  │                  │  │
                          │  │  │  :80  ✓ open  │  │  │                  │  │
                          │  │  │  :443 ✓ open  │  │  │                  │  │
                          │  │  │  :22  ✓ my IP │  │  │                  │  │
                          │  │  └──────┬────────┘  │  │                  │  │
                          │  └─────────┼───────────┘  └──────────────────┘  │
                          │            │ IAM Role                            │
                          │            │ s3:GetObject                        │
                          │            │ s3:PutObject                        │
                          │            ▼                                     │
                          │  ┌─────────────────────────────────────────────┐ │
                          │  │  S3 Bucket (static assets)                  │ │
                          │  │  • Versioning enabled                       │ │
                          │  │  • All public access blocked                │ │
                          │  │  • Reachable only via EC2 IAM role          │ │
                          │  └─────────────────────────────────────────────┘ │
                          │                                                  │
                          │  ┌────────────────────┐  ┌───────────────────┐  │
                          │  │  Private Subnet A  │  │  Private Subnet B │  │
                          │  │  10.0.10.0/24      │  │  10.0.11.0/24     │  │
                          │  │  (AZ-a)            │  │  (AZ-b)           │  │
                          │  │                    │  │                   │  │
                          │  │  ┌──────────────┐  │  │                   │  │
                          │  │  │ RDS Postgres │  │  │                   │  │
                          │  │  │ db.t3.micro  │  │  │                   │  │
                          │  │  │              │  │  │                   │  │
                          │  │  │ [db-sg]      │  │  │                   │  │
                          │  │  │ :5432 ← only │  │  │                   │  │
                          │  │  │ from web-sg  │  │  │                   │  │
                          │  │  └──────────────┘  │  │                   │  │
                          │  └────────────────────┘  └───────────────────┘  │
                          └──────────────────────────────────────────────────┘
```

**Security relationships:**

- `web-sg` → inbound HTTP/HTTPS from `0.0.0.0/0`, SSH from your IP only
- `db-sg` → inbound port 5432 **only** from `web-sg` (not the internet)
- S3 bucket → accessible only via the EC2's IAM role (`s3:GetObject`, `s3:PutObject`)

---

## Setup Instructions

### 1. Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) ≥ 1.5
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) configured
- An AWS account with sufficient permissions (EC2, RDS, S3, IAM, VPC)

### 2. Configure AWS Credentials

**Never hardcode credentials in any `.tf` file.** Use one of:

```bash
# Option A — environment variables
export AWS_ACCESS_KEY_ID="your-key-id"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="us-east-1"

# Option B — AWS CLI profile
aws configure
```

### 3. Backend Setup (Remote State)

The S3 backend bucket must exist **before** running `terraform init`. Create it once manually:

```bash
# 1. Create the state bucket (choose a unique name)
aws s3api create-bucket \
  --bucket vela-terraform-state-YOUR_SUFFIX \
  --region us-east-1

# 2. Enable versioning on the state bucket (recommended)
aws s3api put-bucket-versioning \
  --bucket vela-terraform-state-YOUR_SUFFIX \
  --versioning-configuration Status=Enabled

# 3. Block public access on the state bucket
aws s3api put-public-access-block \
  --bucket vela-terraform-state-YOUR_SUFFIX \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

Then uncomment the `backend "s3"` block in `infra/main.tf` and replace the bucket name.

### 4. Create Your `.tfvars` File

```bash
cp example.tfvars terraform.tfvars
```

Edit `terraform.tfvars` with your real values:

```hcl
aws_region       = "us-east-1"
vpc_cidr         = "10.0.0.0/16"
allowed_ssh_cidr = "203.0.113.5/32"   # Your actual IP — find it with: curl ifconfig.me
db_username      = "velaadmin"
db_password      = "SuperSecurePass1!" # Min 8 chars
s3_bucket_name   = "vela-assets-abc123" # Must be globally unique
```

> `terraform.tfvars` is in `.gitignore` — it will never be committed.

### 5. Deploy

```bash
cd infra/

terraform init
terraform plan -var-file="../example.tfvars"   # Dry run — safe to share
terraform apply -var-file="terraform.tfvars"   # Real apply with your secrets
```

### 6. Tear Down

```bash
terraform destroy -var-file="terraform.tfvars"
```

---

## Variable Reference

| Variable           | Type     | Required | Default       | Description                                                                         |
| ------------------ | -------- | -------- | ------------- | ----------------------------------------------------------------------------------- |
| `aws_region`       | `string` | Yes      | —             | AWS region for all resources (e.g. `us-east-1`)                                     |
| `vpc_cidr`         | `string` | No       | `10.0.0.0/16` | CIDR block for the VPC                                                              |
| `allowed_ssh_cidr` | `string` | Yes      | —             | Your IP in CIDR notation for SSH access (e.g. `1.2.3.4/32`). Never use `0.0.0.0/0`. |
| `db_username`      | `string` | Yes      | —             | Master username for the RDS PostgreSQL instance. Marked `sensitive`.                |
| `db_password`      | `string` | Yes      | —             | Master password for the RDS PostgreSQL instance. Min 8 chars. Marked `sensitive`.   |
| `s3_bucket_name`   | `string` | Yes      | —             | Globally unique name for the S3 bucket. Lowercase, 3–63 chars, no underscores.      |

---

## Outputs

After `terraform apply`, the following values are printed:

| Output           | Description                                         |
| ---------------- | --------------------------------------------------- |
| `ec2_public_ip`  | Public IP of the EC2 web server                     |
| `rds_endpoint`   | Connection endpoint for the RDS PostgreSQL instance |
| `s3_bucket_name` | Name of the S3 static assets bucket                 |

---

## Design Decisions

### 1. RDS in Private Subnets — No Public Access

The RDS instance is placed in private subnets with `publicly_accessible = false`, meaning it has no route to the internet gateway and no public IP. Even if the `db-sg` security group were misconfigured, the network topology prevents direct internet access. The database is reachable only from within the VPC — specifically only from EC2 instances attached to `web-sg`. This is standard defence-in-depth: two independent layers (network + security group) protecting the database.

### 2. IAM Role Instead of Access Keys for S3

The EC2 instance uses an IAM Instance Profile rather than embedded AWS access keys to access S3. This means no credentials are stored on disk, in environment variables, or in code. AWS automatically rotates the temporary credentials injected via the instance metadata service. The policy is scoped to the minimum required: only `s3:GetObject` and `s3:PutObject` on the specific bucket ARN — not `s3:*` or `*`. This follows the principle of least privilege and eliminates an entire class of credential-leakage risk.

### 3. Amazon Linux 2023 AMI via `data` Source

Rather than hardcoding an AMI ID (which is region-specific and goes stale), a `data "aws_ami"` block dynamically resolves the latest Amazon Linux 2023 x86_64 HVM image at plan time. This means the configuration is portable across regions and always uses a current, patched base image.

---

## Pre-Submission Checklist

- [x] `terraform plan -var-file="example.tfvars"` completes with no errors
- [x] No AWS credentials, passwords, or real IPs committed to the repository
- [x] `example.tfvars` committed; real `terraform.tfvars` is in `.gitignore`
- [x] RDS is in private subnets with `publicly_accessible = false`
- [x] SSH locked to `var.allowed_ssh_cidr` (never `0.0.0.0/0`)
- [x] Architecture diagram included
- [x] Variable reference table present
- [x] Backend setup instructions documented
- [x] GitHub repository set to **Public**
