# Infrastructure

Excellent — let’s lock this down as a complete, real-world, enterprise-grade Data Engineering Platform with DEV / PPD / PROD, built using Terraform + GitOps, and covering all required services for ingestion, transformation, orchestration, and loading.

Below is a clean, scalable blueprint you can directly implement and confidently explain in interviews.

## 1️⃣ Target Architecture (Big Picture)

Environments

DEV – feature development & experimentation

PPD (Pre-Prod) – integration, load testing, validation

PROD – production workloads

Each environment is:

- Separate AWS account
- Separate Terraform state
- Separate PKI / Secrets
- Same Terraform modules (DRY)

## 2️⃣ Services Required (Data Engineering)

Layer | Service | Purpose
--- | --- | ---
Ingestion | Apache NiFi | Streaming + batch ingestion
Orchestration | Apache Airflow | Workflow orchestration
Processing | Spark (EMR / Databricks-ready) | Transformations
Storage | S3 (Bronze / Silver / Gold) | Data lake
Metadata | Glue Data Catalog | Table metadata
Security | ACM Private CA | mTLS & user auth
Secrets | AWS Secrets Manager | Keystores, DB creds
Networking | VPC, ALB, ASG | HA & scalability
Observability | CloudWatch + ALB logs | Monitoring
CI/CD | GitHub Actions | GitOps infra + certs

## 3️⃣ Final Repo Structure (Full)

terraform-aws-infra/

- modules/                        # Reusable building blocks
  - networking/
    - main.tf
    - variables.tf
    - outputs.tf
    - versions.tf
    - README.md
  - s3-datalake/
    - main.tf
    - variables.tf
    - outputs.tf
    - lifecycle.tf           # bucket lifecycle / retention
    - README.md
  - pki-acm-ca/
    - main.tf
    - variables.tf
    - outputs.tf
    - policy.tf              # PCA IAM policies
    - README.md
  - nifi/
    - main.tf
    - variables.tf
    - outputs.tf
    - asg.tf                 # autoscaling group
    - alb.tf                 # ALB + target groups
    - user-data.tpl          # cloud-init / bootstrap
    - README.md
  - airflow/
    - main.tf
    - variables.tf
    - outputs.tf
    - rds.tf                 # Postgres RDS config
    - dags_sync.tf           # DAGs sync (S3)
    - ecs_or_ec2.tf          # optional runtime
    - README.md
  - iam/
    - main.tf
    - variables.tf
    - outputs.tf
    - policies.tf            # managed & inline policies
    - README.md
  - monitoring/
    - main.tf
    - variables.tf
    - outputs.tf
    - alarms.tf              # CloudWatch alarms
    - dashboards.tf          # optional dashboards
    - README.md

- environments/                  # Per-account/per-env stacks
  - dev/
    - provider.tf               # provider + region
    - backend.tf                # remote state backend config
    - main.tf                   # root module composition
    - variables.tf              # environment variables
    - terraform.tfvars          # env-specific values (gitignored)
    - versions.tf               # required_providers & terraform block
    - outputs.tf                # exported outputs
  - ppd/                         # same structure as dev
    - provider.tf
    - backend.tf
    - main.tf
    - variables.tf
    - terraform.tfvars
    - versions.tf
    - outputs.tf
  - prod/                        # same structure as dev
    - provider.tf
    - backend.tf
    - main.tf
    - variables.tf
    - terraform.tfvars
    - versions.tf
    - outputs.tf

- cert-requests/                  # GitOps user onboarding
  - dev/
    - alice.yaml
    - bob.yaml
  - ppd/
    - ci-user.yaml
  - prod/
    - ops-user.yaml

- .github/workflows/
  - terraform.yml               # Infra deployment (per-branch env selection)
  - cert-issuer.yml             # Cert issuance automation
  - promote.yml                 # Promotion workflow (manual approval for PROD)
  - security-scan.yml           # optional scanners (tfsec, checkov)

## 4️⃣ Environment Flow (DEV → PPD → PROD)

### Git Strategy

Action | Result
--- | ---
`feature/*` branch | Deploys to DEV
`release/*` branch | Deploys to PPD
`vX.Y.Z` tag | Deploys to PROD

### Terraform Deployment Workflow

```yaml
on:
  push:
    branches:
      - feature/*
      - release/*
    tags:
      - v*

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Select environment
        run: |
          if [[ "${GITHUB_REF}" == refs/heads/feature/* ]]; then
            echo "ENV=dev" >> $GITHUB_ENV
          elif [[ "${GITHUB_REF}" == refs/heads/release/* ]]; then
            echo "ENV=ppd" >> $GITHUB_ENV
          else
            echo "ENV=prod" >> $GITHUB_ENV
          fi

      - name: Terraform Apply
        run: |
          cd environments/$ENV
          terraform init
          terraform apply -auto-approve
```

## 5️⃣ Networking Module (Shared)

modules/networking

- VPC
- Public + Private subnets
- NAT Gateway
- ALB
- Security Groups

Used identically across dev / ppd / prod.

## 6️⃣ Data Lake (S3)

modules/s3-datalake

```hcl
resource "aws_s3_bucket" "bronze" {
  bucket = "dl-${var.env}-bronze"
}

resource "aws_s3_bucket" "silver" {
  bucket = "dl-${var.env}-silver"
}

resource "aws_s3_bucket" "gold" {
  bucket = "dl-${var.env}-gold"
}
```

## 7️⃣ NiFi (EC2 + ASG + ALB)

Architecture

- NiFi nodes in private subnet
- ALB in public subnet
- mTLS enforced
- Auto Scaling enabled

Certificate handling

- Truststore → S3
- Keystore → Secrets Manager

NiFi user data (simplified)

```bash
aws s3 cp s3://nifi-${ENV}-certs/truststore.jks /opt/nifi/conf/

aws secretsmanager get-secret-value \
  --secret-id nifi-keystore-${ENV} \
  --query SecretString \
  --output text > keystore.p12
```

## 8️⃣ Airflow (Latest Version)

Architecture

- EC2 or ECS (start EC2 for learning)
- Metadata DB → RDS Postgres
- DAGs → S3
- IAM role for NiFi trigger

## 9️⃣ Certificate Management (Critical)

Central CA (per environment)

Environment | CA
--- | ---
dev | dev-acm-pca
ppd | ppd-acm-pca
prod | prod-acm-pca

User cert via Git

cert-requests/dev/alice.yaml

```yaml
user: alice
env: dev
subject:
  common_name: alice
  organizational_unit: DataEng
  organization: Platform
expiry_days: 180
```

Flow

- PR approved
- Cert issued
- Stored in Secrets Manager
- User imports into browser

## 🔐 How Users Login (NiFi)

- User opens NiFi URL
- Browser presents client certificate
- NiFi validates against truststore
- DN mapped to role
- Access granted (no passwords, no IAM users)

## 🔄 DEV → PPD → PROD Promotion

Promotion rules

- Infra identical
- Only instance size, scaling, retention, approvals differ

Promote using tag-based deployment with manual approval for PROD.

## 🔍 Monitoring & Ops

- CloudWatch logs
- ALB access logs
- Auto Scaling alarms
- NiFi provenance tuning
- Airflow SLA alerts

## 🎯 Why This Is Real-World & Interview-Ready

- Multi-account AWS
- GitOps infra + certs
- Secure mTLS auth
- Scalable ingestion
- Production-grade orchestration
- Clean promotion strategy

## 🧭 Recommended Implementation Order

1. Networking + S3  
2. ACM PCA module  
3. NiFi single node (DEV)  
4. Cert GitOps flow  
5. Airflow integration  
6. Scale to ASG  
7. Add PPD & PROD