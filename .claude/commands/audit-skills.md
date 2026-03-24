---
name: audit-skills
description: Periodic health audit of all skills in .claude/skills/. Checks Iron Law presence, last-reviewed staleness (90+ days), description quality, allowed-tools declaration, and body line count. Outputs an aggregate report with PASS/WARN/FAIL per skill. Use when: auditing skill health, reviewing skill governance, finding stale skills, checking skill compliance.
allowed-tools: Read, Glob, Grep
---

Audit all skills in `.claude/skills/` and produce an aggregate health report.

## Audit Dimensions (7 checks per skill)

For each skill, check:

| # | Check | PASS | WARN | FAIL |
|---|-------|------|------|------|
| 1 | **Iron Law** | Present in body — any of: `## Iron Law` heading, `**Iron Law:**` bold text, or `> **Iron Law:**` blockquote | — | Absent (none of the above formats found) |
| 2 | **last-reviewed** | ≤ 90 days ago | 91–180 days | Missing or > 180 days |
| 3 | **Description quality** | ≥ 2 sentences + trigger words | 1 sentence | Missing or < 10 words |
| 4 | **allowed-tools** | Declared | — | Missing |
| 5 | **Body line count** | ≤ 500 lines | 451–500 lines | > 500 lines |
| 6 | **References exist** | references/ dir present if body references it | — | Body says "see references/" but dir missing |
| 7 | **name matches dir** | SKILL.md `name:` == directory name | — | Mismatch |

## Scoring

- FAIL on any check = ❌ (needs attention)
- WARN on any check, no FAIL = ⚠️ (monitor)
- All PASS = ✅ (healthy)

## Steps

1. Glob `.claude/skills/*/SKILL.md` to get all skill files
2. For each skill file, read the frontmatter and first 50 lines of body (increase to 60 lines if no Iron Law found in first 50)
3. Apply the 7 checks above
4. Output results in the report format below

## Report Format

```
# Skill Health Report — {DATE}
Total skills: {N} | ✅ Healthy: {N} | ⚠️ Warning: {N} | ❌ Failing: {N}

## ❌ Failing Skills (action required)
| Skill | Failed Checks | Details |
|-------|--------------|---------|
| skill-name | Iron Law, allowed-tools | No Iron Law in body; allowed-tools missing |

## ⚠️ Warning Skills (monitor)
| Skill | Warnings | Details |
|-------|---------|---------|
| skill-name | last-reviewed | Last reviewed: 2025-09-10 (185 days ago) |

## ✅ Healthy Skills
{skill-name}, {skill-name}, ... (comma-separated list)

## Recommended Actions
1. [Highest priority fix — skill name + specific action]
2. ...
```

## Staleness Calculation

Today's date: use the `currentDate` from context if available, otherwise use Bash to get `date +%Y-%m-%d`.

A skill is STALE (FAIL) if `last-reviewed` is missing entirely or more than 180 days ago.
A skill is WARN if `last-reviewed` is between 91 and 180 days ago.

## After the Report

If any skills are ❌ FAIL:
- List the top 3 by impact (implementation skills > review skills > utility skills)
- For each: state exactly what to fix and in which file

Do NOT auto-fix skills. Report only. The human decides what to update.
