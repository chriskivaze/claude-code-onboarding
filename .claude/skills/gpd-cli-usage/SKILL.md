---
name: gpd-cli-usage
description: Guidance for using the Google Play Developer CLI (gpd) — flags, output formats, authentication, pagination, safety conventions, and command discovery. Use when asked to run or design gpd commands, set up service account auth, configure GPD_SERVICE_ACCOUNT_KEY, understand gpd output formats, or learn gpd flag conventions. Triggers: "gpd command", "gpd CLI", "Google Play CLI", "gpd auth", "gpd flags", "gpd output", "Play Developer CLI", "GPD_SERVICE_ACCOUNT_KEY".
allowed-tools: Bash, Read
metadata:
  triggers: gpd command, gpd CLI, Google Play CLI, gpd auth, gpd flags, gpd output, Play Developer CLI, GPD_SERVICE_ACCOUNT_KEY, gpd help, gpd pagination
  related-skills: gpd-id-resolver, gpd-release-flow, gpd-submission-health
  domain: mobile-deployment
  role: specialist
  scope: deployment
  output-format: commands
last-reviewed: "2026-03-15"
---

# GPD CLI Usage

## Iron Law

**NEVER GENERATE GPD COMMANDS WITHOUT RUNNING `gpd --help` OR SUBCOMMAND `--help` FIRST**

Always use `--help` to discover current flags and subcommands. The gpd CLI evolves — never assume flags from memory.

## Command discovery
- Always use `--help` to confirm commands and flags.
  - `gpd --help`
  - `gpd publish --help`
  - `gpd monetization --help`

## Flag conventions
- Use explicit long flags (for example: `--package`, `--track`, `--status`).
- No interactive prompts; destructive operations require `--confirm`.
- Use `--all` when the user wants all pages.

## Output formats
- Default output is minified JSON.
- Use `--pretty` for readable JSON during debugging.

## Authentication and defaults
- Service account auth via `GPD_SERVICE_ACCOUNT_KEY` is required.
- Validate access for a package:
  - `gpd auth check --package com.example.app`

## Safety
- Use `--dry-run` when available before destructive operations.
- Prefer edit lifecycle (`gpd publish edit create`) for multi-step publishing.

## Documentation Sources

| Source | How to Access | Purpose |
|--------|--------------|---------|
| gpd CLI help | `gpd --help`, `gpd <command> --help` | Discover current flags, subcommands, and options |
| gpd subcommand help | `gpd publish --help`, `gpd monetization --help` | Command-specific flags and usage |
