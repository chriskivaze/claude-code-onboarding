# Cloud Run + Terraform Infrastructure

> **When to use**: Provisioning or modifying GCP infrastructure for services — Cloud Run, Cloud SQL, Artifact Registry, IAM, Secret Manager
> **Time estimate**: 2–4 hours for initial setup; 30 min per additive change
> **Prerequisites**: GCP project exists, Terraform installed, GCS bucket for remote state

## Overview

GCP infrastructure provisioning using Terraform with the `terraform-specialist` agent. Covers Cloud Run services, Cloud SQL (PostgreSQL), Artifact Registry, Secret Manager, IAM bindings, and Workload Identity Federation — all with multi-environment (staging/prod) module design and GCS remote state.

---

## Agent

**`terraform-specialist`** — use for ALL Terraform and GCP infrastructure work.

Capabilities (from agent description):
- Cloud Run services with traffic splitting
- Cloud SQL PostgreSQL with connection pooling
- Artifact Registry for Docker images
- Firebase project integration
- IAM bindings and least-privilege
- Secret Manager with version management
- Workload Identity Federation (no service account keys)
- GCS remote state, multi-environment modules
- tfsec/Checkov security scanning
- GitHub Actions Terraform pipeline

---

## Phases

### Phase 1 — Repository Structure

**Dispatch `terraform-specialist`** to scaffold:

```
terraform/
├── modules/
│   ├── cloud-run/          # Cloud Run service module
│   ├── cloud-sql/          # PostgreSQL module
│   ├── artifact-registry/  # Docker registry module
│   ├── secret-manager/     # Secrets module
│   └── wif/                # Workload Identity Federation module
├── environments/
│   ├── staging/
│   │   ├── main.tf         # Calls modules with staging vars
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf      # GCS remote state — staging bucket
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf      # GCS remote state — prod bucket
└── .github/workflows/
    └── terraform.yml       # Plan on PR, Apply on merge
```

---

### Phase 2 — Remote State (GCS)

```hcl
# terraform/environments/staging/backend.tf
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "staging"
  }
}
```

```bash
# One-time setup — create state bucket
gsutil mb -l us-central1 gs://my-project-terraform-state
gsutil versioning set on gs://my-project-terraform-state
```

**Why GCS remote state**: State file contains sensitive resource IDs; never commit to git. Versioning enables rollback.

---

### Phase 3 — Cloud Run Module

```hcl
# terraform/modules/cloud-run/main.tf
variable "service_name" {}
variable "image_url" {}
variable "region" {}
variable "min_instances" { default = 0 }
variable "max_instances" { default = 10 }
variable "secrets" {
  type = map(string)   # env_var_name → secret_id
  default = {}
}

resource "google_cloud_run_v2_service" "service" {
  name     = var.service_name
  location = var.region

  template {
    containers {
      image = var.image_url

      dynamic "env" {
        for_each = var.secrets
        content {
          name = env.key
          value_source {
            secret_key_ref {
              secret  = env.value
              version = "latest"
            }
          }
        }
      }

      liveness_probe {
        http_get { path = "/health" }
        initial_delay_seconds = 10
        period_seconds        = 30
      }
    }

    scaling {
      min_instance_count = var.min_instances
      max_instance_count = var.max_instances
    }
  }
}

output "service_url" {
  value = google_cloud_run_v2_service.service.uri
}
```

**Staging usage**:
```hcl
# terraform/environments/staging/main.tf
module "api_service" {
  source       = "../../modules/cloud-run"
  service_name = "my-api-staging"
  image_url    = "us-central1-docker.pkg.dev/${var.project_id}/my-registry/my-api:latest"
  region       = "us-central1"
  min_instances = 0    # Scale to zero in staging
  secrets = {
    DATABASE_URL = "projects/${var.project_id}/secrets/staging-db-url"
    JWT_SECRET   = "projects/${var.project_id}/secrets/staging-jwt-secret"
  }
}
```

---

### Phase 4 — Cloud SQL (PostgreSQL)

```hcl
# terraform/modules/cloud-sql/main.tf
resource "google_sql_database_instance" "postgres" {
  name             = var.instance_name
  database_version = "POSTGRES_16"
  region           = var.region

  settings {
    tier              = var.machine_type   # "db-f1-micro" (dev), "db-n1-standard-2" (prod)
    availability_type = var.ha ? "REGIONAL" : "ZONAL"

    backup_configuration {
      enabled    = true
      start_time = "03:00"
      point_in_time_recovery_enabled = var.ha
    }

    ip_configuration {
      ipv4_enabled    = false   # No public IP
      private_network = var.vpc_id
    }
  }

  deletion_protection = var.deletion_protection
}

resource "google_sql_database" "database" {
  name     = var.database_name
  instance = google_sql_database_instance.postgres.name
}

resource "google_sql_user" "app_user" {
  name     = var.db_user
  instance = google_sql_database_instance.postgres.name
  password = random_password.db_password.result
}
```

**Connection from Cloud Run**: Use Cloud SQL Auth Proxy (auto-injected as sidecar):
```hcl
# In cloud-run module, add Cloud SQL connection
containers {
  ...
  env {
    name  = "DATABASE_URL"
    value = "postgresql://${var.db_user}:PASSWORD@localhost/mydb?host=/cloudsql/${var.connection_name}"
  }
}

volumes {
  name = "cloudsql"
  cloud_sql_instance { instances = [var.connection_name] }
}
```

---

### Phase 5 — Secret Manager

```hcl
# terraform/modules/secret-manager/main.tf
resource "google_secret_manager_secret" "secret" {
  for_each  = var.secrets
  secret_id = each.key

  replication {
    auto {}
  }
}

# Allow Cloud Run service account to read secrets
resource "google_secret_manager_secret_iam_member" "cloud_run_accessor" {
  for_each  = var.secrets
  secret_id = google_secret_manager_secret.secret[each.key].id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${var.cloud_run_sa}"
}
```

**Secret values** are set outside Terraform (never store secret values in `.tfvars` or state):
```bash
echo -n "postgresql://..." | gcloud secrets versions add staging-db-url --data-file=-
```

---

### Phase 6 — Terraform CI/CD Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  plan:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.TERRAFORM_SA }}

      - name: Terraform Plan
        run: |
          cd terraform/environments/staging
          terraform init
          terraform plan -out=tfplan

      - name: Security scan (tfsec)
        uses: aquasecurity/tfsec-action@v1.0.0

  apply:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - name: Terraform Apply
        run: |
          cd terraform/environments/staging
          terraform init
          terraform apply -auto-approve
```

---

## Quick Reference

| Phase | Action | Agent | Gate |
|-------|--------|-------|------|
| 1 — Structure | Scaffold module hierarchy | `terraform-specialist` | Directory structure created |
| 2 — State | GCS backend configured | `terraform-specialist` | `terraform init` succeeds |
| 3 — Cloud Run | Service module + per-env usage | `terraform-specialist` | `terraform plan` clean |
| 4 — Cloud SQL | PostgreSQL module | `terraform-specialist` | Private IP, no public access |
| 5 — Secrets | Secret Manager + IAM | `terraform-specialist` | Secrets accessible to Cloud Run |
| 6 — CI/CD | GitHub Actions plan/apply | `deployment-engineer` | Plan on PR, apply on merge |

---

## Common Pitfalls

- **Committing `.tfstate` to git** — state contains secrets; always use GCS remote state
- **Service account JSON keys** — use Workload Identity Federation; keys are a credential leak vector
- **Public IP on Cloud SQL** — set `ipv4_enabled = false`; access via Cloud SQL Auth Proxy or private IP
- **No deletion protection** — set `deletion_protection = true` on production DB
- **All secrets in `.tfvars`** — tfvars can end up in git; put only non-secret config there; secrets via Secret Manager
- **Skipping tfsec** — catches misconfigurations (public S3, IAM over-permissions) before apply

## Skills Used in This Workflow

| Skill | Load When |
|-------|-----------|
| `terraform-skill` | Load BEFORE writing any HCL — naming, testing strategy, `for_each` vs `count`, version constraints |
| `terraform-module-library` | Load when authoring or consuming GCP modules — full HCL patterns in `references/gcp-modules.md` |

## Related Workflows

- [`terraform-module-development.md`](terraform-module-development.md) — authoring and testing reusable Terraform modules for GCP
- [`deployment-ci-cd.md`](deployment-ci-cd.md) — application CI/CD that deploys to Cloud Run infrastructure
- [`database-schema-design.md`](database-schema-design.md) — schema migrations after Cloud SQL is provisioned
- [`security-audit.md`](security-audit.md) — security posture of infrastructure after provisioning
