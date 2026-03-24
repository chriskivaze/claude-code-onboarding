---
name: gpd-id-resolver
description: Resolve Google Play identifiers (package names, track names, version codes, product IDs, subscription IDs, base plan IDs, offer IDs, permission IDs) using gpd CLI commands. Use when gpd commands require IDs or exact values, when you only know an app name or bundle, or when you need to look up track names, tester groups, or monetization product IDs. Triggers: "resolve ID", "package name", "track name", "version code", "product ID", "subscription ID", "Google Play ID", "gpd identifier".
allowed-tools: Bash, Read
metadata:
  triggers: resolve ID, package name, track name, version code, product ID, subscription ID, Google Play ID, gpd identifier, tester group, offer ID, base plan ID
  related-skills: gpd-cli-usage, gpd-release-flow, gpd-betagroups, gpd-submission-health, gpd-build-lifecycle
  domain: mobile-deployment
  role: specialist
  scope: deployment
  output-format: commands
last-reviewed: "2026-03-15"
---

# GPD ID Resolver

## Iron Law

**NEVER PASS A HUMAN-READABLE NAME TO A GPD COMMAND THAT REQUIRES AN EXACT ID OR PACKAGE NAME — RESOLVE IT FIRST**

Always pass `--package` explicitly. Use `--all` on list commands to avoid missing items.

## Package name (app ID)
- Package name is the primary identifier: `com.example.app`.
- Always pass `--package` explicitly for deterministic results.

## Track names
- Common tracks: `internal`, `alpha`, `beta`, `production`.
- List tracks:
  - `gpd publish tracks --package com.example.app`

## Version codes and release status
- Use release status to find version codes on a track:
  - `gpd publish status --package com.example.app --track production`

## Tester groups
- List testers by track:
  - `gpd publish testers list --package com.example.app --track internal`

## Monetization IDs
- Products:
  - `gpd monetization products list --package com.example.app`
  - `gpd monetization products get sku123 --package com.example.app`
- One-time products:
  - `gpd monetization onetimeproducts list --package com.example.app`
- Subscriptions:
  - `gpd monetization subscriptions list --package com.example.app`
  - `gpd monetization subscriptions get sub123 --package com.example.app`
- Base plans and offers:
  - `gpd monetization baseplans migrate-prices --package com.example.app sub123 plan456 --region-code US --price-micros 9990000`
  - `gpd monetization offers list --package com.example.app sub123 plan456`

## Permissions IDs
- Developer users:
  - `gpd permissions users list --developer-id DEV_ID`
- App grants:
  - `gpd permissions grants create --package com.example.app --email user@example.com --app-permissions CAN_REPLY_TO_REVIEWS`

## Output tips
- JSON is default; use `--pretty` for debugging.
- Use `--all` on list commands to avoid missing items.

## Documentation Sources

| Source | How to Access | Purpose |
|--------|--------------|---------|
| gpd CLI help | `gpd publish --help`, `gpd monetization --help` | Discover list/filter flags for each entity type |
| gpd output formats | `--pretty` for debugging, default JSON for scripting | Machine-parseable ID extraction |
