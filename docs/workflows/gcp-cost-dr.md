# GCP Cost Optimization & Disaster Recovery

> **When to use**: Setting up GCP billing budgets, analyzing committed use discounts, configuring Cloud SQL HA/PITR, planning multi-region DR, or responding to unexpected cost spikes
> **Prerequisites**: GCP project with billing account, Terraform workspace, `gcp-finops` skill loaded

## Overview

Cost and resilience are two sides of operational maturity. This workflow covers GCP-specific FinOps practices (CUD, SUD, labels, Recommender) and DR patterns (Cloud SQL PITR, multi-region Cloud Run, RTO/RPO tiers) for the workspace's tech stack.

---

## Skills and Agents

| Skill / Agent | When to Use |
|--------------|------------|
| `gcp-finops` skill | Cost analysis, budget alerts, CUD sizing, DR tier planning |
| `terraform-specialist` agent | Implement billing budgets, CUD reservations, Cloud SQL HA as Terraform |
| `security-reviewer` agent | IAM for billing exports, cost data access |

---

## Cost Optimization Phases

### Phase 1 — Cost Visibility

**Load `gcp-finops` skill → `reference/gcp-cost-optimization.md`**

```bash
# Run Recommender to find cost optimization opportunities
gcloud recommender recommendations list \
  --project=$PROJECT_ID \
  --location=global \
  --recommender=google.billing.CostInsightRecommender \
  --format="table(recommenderSubtype,stateInfo.state,primaryImpact.costProjection.cost.units)"

# VM right-sizing (if using Compute Engine)
gcloud recommender recommendations list \
  --project=$PROJECT_ID \
  --location=us-central1 \
  --recommender=google.compute.instance.MachineTypeRecommender
```

**Cost allocation labels** — apply to all Cloud Run services and Cloud SQL instances:
```bash
gcloud run services update SERVICE_NAME \
  --update-labels environment=prod,team=backend,service=NAME,cost-center=eng
```

### Phase 2 — Committed Use Discounts

**Agent**: `terraform-specialist`

CUD sizing checklist:
- [ ] Identify baseline vCPU/memory usage over last 30 days (Cloud Monitoring)
- [ ] Commit only to baseline — never commit to peak usage
- [ ] 1-year CUD for stable workloads; 3-year only for proven long-term services
- [ ] Spend-based CUD for variable workloads that still have a predictable floor

```hcl
# terraform/modules/finops/cud.tf
resource "google_compute_commitment" "baseline_cud" {
  name   = "${var.service_name}-cud-1yr"
  region = var.region
  plan   = "TWELVE_MONTH"  # or THIRTY_SIX_MONTH
  resources {
    type   = "VCPU"
    amount = var.committed_vcpus
  }
  resources {
    type   = "MEMORY"
    amount = var.committed_memory_mb
  }
}
```

### Phase 3 — Billing Budgets

**Agent**: `terraform-specialist`

Set per-service budgets with Pub/Sub alert routing:
```hcl
resource "google_billing_budget" "per_service" {
  billing_account = var.billing_account_id
  display_name    = "${var.service_name}-${var.environment}-budget"
  budget_filter {
    projects = ["projects/${var.project_id}"]
    labels   = { service = var.service_name }
  }
  amount {
    specified_amount {
      currency_code = "USD"
      units         = var.monthly_budget_usd
    }
  }
  threshold_rules { threshold_percent = 0.8 }
  threshold_rules { threshold_percent = 1.0 }
}
```

---

## DR Planning Phases

### Phase 4 — Assign DR Tiers

**Load `gcp-finops` skill → `reference/gcp-resilience-dr.md`**

| Service | Tier | RTO | RPO | Action |
|---------|------|-----|-----|--------|
| Auth API | Critical | < 1hr | < 15min | Multi-region Cloud Run + Cloud SQL Regional HA |
| Core APIs | Standard | < 4hr | < 1hr | Single-region + Cloud SQL PITR enabled |
| Batch jobs | Non-critical | < 24hr | < 4hr | Single-region, manual recovery |

### Phase 5 — Cloud SQL HA + PITR

**Agent**: `terraform-specialist`

```hcl
# Cloud SQL with HA + PITR — see gcp-finops reference/gcp-resilience-dr.md
settings {
  availability_type = "REGIONAL"   # HA standby in different zone
  backup_configuration {
    enabled                        = true
    point_in_time_recovery_enabled = true
    transaction_log_retention_days = 7
    backup_retention_settings {
      retained_backups = 30
    }
  }
}
```

### Phase 6 — Multi-Region Cloud Run (if Critical tier)

Deploy to primary + DR region via GitHub Actions matrix:
```yaml
strategy:
  matrix:
    region: [us-central1, us-east1]
```

Traffic: 100% → primary region. DR region is warm standby (0% traffic, min-instances=1).

---

## Quick Reference

| Phase | Action | Tool | Gate |
|-------|--------|------|------|
| 1 — Visibility | Recommender + labels | `gcp-finops` skill | Labels on all resources |
| 2 — CUD | Commit baseline vCPU/mem | `terraform-specialist` | CUD utilization >80% |
| 3 — Budgets | Billing alert per service | `terraform-specialist` | Alert fires at 80% threshold |
| 4 — DR Tiers | Assign RTO/RPO per service | `gcp-finops` skill | Tier documented in ADR |
| 5 — Cloud SQL HA | REGIONAL + PITR enabled | `terraform-specialist` | PITR tested with clone |
| 6 — Multi-region | Matrix deploy if Critical | `deployment-engineer` | Health check passes in both regions |

---

## Common Pitfalls

- **CUD over-commitment** — committing to peak usage instead of baseline; unutilized CUD is wasted money
- **No cost labels** — impossible to allocate costs per service or team without labels on every resource
- **PITR not tested** — enable PITR is not DR; test a PITR clone restore before an incident happens
- **min-instances=1 on all envs** — staging services with min=1 run 24/7; use min=0 for non-prod
- **Single-region for Critical tier** — any Cloud Run or Cloud SQL in single-region fails the Critical DR gate

## Related Workflows

- [`cloud-run-terraform.md`](cloud-run-terraform.md) — Cloud SQL HA module, Cloud Run Terraform config
- [`deployment-ci-cd.md`](deployment-ci-cd.md) — GitHub Actions deploy pipeline (matrix for multi-region)
- [`gcp-cloud-run` skill](../../.claude/skills/gcp-cloud-run/SKILL.md) — Cloud Run cold start optimization
