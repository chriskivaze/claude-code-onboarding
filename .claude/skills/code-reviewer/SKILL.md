---
name: code-reviewer
description: General-purpose code review skill. Provides checklists for security, code quality, performance, and best practices. Use when reviewing code changes, PRs, or performing quality audits.
allowed-tools: Read, Grep, Glob, Bash
agent: code-reviewer
context: fork
metadata:
  triggers: code review, review code, PR review, pull request review, code quality, code audit, review this code
  related-skills: security-reviewer, dedup-code-agent, test-driven-development
  domain: quality
  role: specialist
  scope: review
  output-format: report
last-reviewed: "2026-03-15"
---

**Iron Law:** Never approve code without reading the actual implementation; always provide severity-bucketed findings with file:line evidence.

# Code Reviewer

General-purpose code review skill covering security, quality, performance, and best practices.

## When to Use

- After writing or modifying code
- During PR reviews
- When auditing code quality

## Process

1. Identify changed files via `git diff` or user request
2. Read [reference/code-review-checklist.md](reference/code-review-checklist.md) for review categories, severity levels, and output format
3. Review each file against the checklist
4. Report findings by severity (Critical > High > Medium > Low)

## Reference Files

| File | Contents | Load When |
|------|----------|-----------|
| `reference/code-review-checklist.md` | Security checks, code quality, performance, best practices, output format | Reviewing any code change, pre-PR checklist, security audit |

## Error Handling

If no changes are found, report "No changes detected" and list the files/paths searched.
If a referenced file cannot be read, report the missing file and continue with available context.
