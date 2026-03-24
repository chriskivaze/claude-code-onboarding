# Terraform Module Development

> **When to use**: Creating or extending reusable Terraform modules for GCP — Cloud Run, Cloud SQL, Artifact Registry, VPC, Secret Manager, or Workload Identity Federation
> **Time estimate**: 1–2 hours for a new module with tests; 30 min for extending an existing module
> **Prerequisites**: `terraform-skill` loaded, Terraform 1.9+ installed, `terraform-module-library` skill loaded

## Overview

GCP-focused workflow for authoring reusable, tested Terraform modules following workspace conventions. Uses `terraform-skill` for standards and `terraform-module-library` for GCP-specific patterns. The `terraform-specialist` agent provisions; this workflow produces the modules it consumes.

---

## Skills and Agents

| Skill / Agent | When to Use |
|--------------|-------------|
| `terraform-skill` skill | Load FIRST — naming, testing strategy, code structure |
| `terraform-module-library` skill | GCP module patterns, reference HCL |
| `terraform-specialist` agent | Apply/provision the modules after authoring |
| `security-reviewer` agent | Security review after module is written |

**Load order:** `terraform-skill` first, then `terraform-module-library`.

---

## Phases

### Phase 1 — Module Scaffolding

**Load**: `terraform-skill` + `terraform-module-library` skills

**Create module directory structure** (from `terraform-module-library/SKILL.md`):

```
terraform/modules/<module-name>/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── README.md
└── tests/
    └── main.tftest.hcl
```

**versions.tf (required for all GCP modules):**

```hcl
terraform {
  required_version = "~> 1.9.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}
```

---

### Phase 2 — Variable and Resource Design

**Reference**: `terraform-module-library/references/gcp-modules.md` for the target resource

**Variable checklist** (from `terraform-skill` naming conventions):

- [ ] Every variable has `description` and `type`
- [ ] Names are context-prefixed (`service_name`, not `name`)
- [ ] `validation {}` block for any constrained value (e.g., Cloud Run name <= 49 chars)
- [ ] `deletion_protection` variable for stateful resources (Cloud SQL, Artifact Registry)
- [ ] No hardcoded values — all configurable via variables

**Resource checklist**:

- [ ] `for_each` used (not `count`) for IAM members and multi-instance resources
- [ ] Resource arguments in correct order: count/for_each → required args → optional args → labels → depends_on → lifecycle
- [ ] `lifecycle.ignore_changes` on Cloud Run image (CI/CD manages image, not Terraform)
- [ ] `ipv4_enabled = false` on Cloud SQL (no public IP)
- [ ] No `provider {}` block in module (environment roots configure providers)

---

### Phase 3 — Outputs

**Output checklist**:

- [ ] Every important resource attribute exposed as an output
- [ ] All outputs have `description`
- [ ] Sensitive values (passwords, connection strings) marked `sensitive = true`
- [ ] Cloud SQL modules expose `connection_name` (required for Cloud Run sidecar)
- [ ] Artifact Registry modules expose `repository_url` (full URL prefix for image tagging)
- [ ] WIF modules expose `provider_name` (used in GitHub Actions workflows)

---

### Phase 4 — Testing (Mandatory)

**Reference**: `terraform-skill` § Testing Strategy

**Use native terraform test with mock providers (Terraform 1.7+)**:

```hcl
# tests/main.tftest.hcl
mock_provider "google" {
  mock_resource "google_<resource_type>" {
    defaults = {
      # Set outputs that your assertions will check
    }
  }
}

variables {
  # Provide all required variables
}

run "verify_happy_path" {
  command = apply   # Safe — mock provider, no real GCP calls

  assert {
    condition     = <resource>.<name>.<attr> == expected
    error_message = "Descriptive failure message"
  }
}

run "verify_validation_fails" {
  command = plan
  variables { <invalid_value> }
  expect_failures = [var.<variable_name>]
}
```

**Gate**: `terraform test` must pass before any PR. CI runs it automatically (see `docs/workflows/cloud-run-terraform.md` Phase 6).

---

### Phase 5 — Security Review

**Dispatch `security-reviewer` agent** to check:

- No hardcoded credentials or project IDs
- IAM roles follow least-privilege (`roles/artifactregistry.reader` not `roles/owner`)
- Cloud SQL has `ipv4_enabled = false` and `ssl_mode = "ENCRYPTED_ONLY"`
- WIF `attribute_condition` is scoped (not open to all GitHub)
- `deletion_protection = true` for production stateful resources
- Secrets stored in Secret Manager, never in `.tfvars` or state outputs

---

### Phase 6 — Integration into Environment

After module is authored and tested:

1. **Dispatch `terraform-specialist` agent** to wire the module into `environments/staging/main.tf`
2. Run `terraform plan` to validate no unexpected changes
3. Run `terraform apply` in staging
4. Run `terraform plan` in production — get human approval before applying

---

## Module Development Checklist

```
Before opening PR for a new/modified module:
[ ] terraform fmt -check passes
[ ] terraform validate passes
[ ] tflint passes (no warnings)
[ ] tfsec passes (no HIGH/CRITICAL findings)
[ ] terraform test passes (all assertions green)
[ ] README.md updated with usage example
[ ] security-reviewer agent verdict = no CRITICAL/HIGH unresolved
[ ] Module wired into staging environment and plan is clean
```

---

## Quick Reference: Which Module for Which Task

| Task | Module | Reference |
|------|--------|-----------|
| Deploy a service to Cloud Run | `modules/cloud-run` | `gcp-modules.md § Cloud Run` |
| Provision PostgreSQL database | `modules/cloud-sql` | `gcp-modules.md § Cloud SQL` |
| Docker image registry | `modules/artifact-registry` | `gcp-modules.md § Artifact Registry` |
| GitHub Actions → GCP auth (no keys) | `modules/wif` | `gcp-modules.md § Workload Identity Federation` |
| Private networking for Cloud SQL | `modules/vpc` | `gcp-modules.md § VPC` |
| Store and rotate secrets | Secret Manager pattern in cloud-sql module | `gcp-modules.md § Cloud SQL` |

---

## Common Pitfalls

- **Using `count` for IAM members** — reordering the list causes destroy/recreate. Use `for_each = toset(...)` always.
- **Hardcoding image URL in Cloud Run module** — use `lifecycle.ignore_changes = [image]`; CI/CD owns image updates.
- **No `deletion_protection`** — Cloud SQL instances deleted by `terraform destroy` in staging can mistakenly template prod config. Always add the variable even if default is `false` in dev.
- **Testing with real provider** — use `mock_provider "google"` so tests run without GCP auth in CI.
- **Sensitive outputs not marked** — `terraform output` will print passwords in plaintext. Mark with `sensitive = true`.

---

## Related Workflows

- [`cloud-run-terraform.md`](cloud-run-terraform.md) — End-to-end GCP provisioning using these modules
- [`deployment-ci-cd.md`](deployment-ci-cd.md) — Application CI/CD that deploys to provisioned Cloud Run
- [`security-audit.md`](security-audit.md) — Security review of provisioned infrastructure
