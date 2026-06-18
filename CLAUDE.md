# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of **GitHub Copilot Skills** — reusable skill definitions consumed by GitHub Copilot coding agents. Each skill is a markdown file that provides specialized instructions, workflows, and domain knowledge for Copilot to follow when invoked.

Skills are defined in `.github/skills/<skill-name>/SKILL.md` and use YAML frontmatter to declare metadata (name, description, scope, whether users can invoke it directly, etc.).

## Skill Format

Every skill file at `.github/skills/<skill-name>/SKILL.md` follows this structure:

```yaml
---
name: <kebab-case-name>
description: "<one-line description>"
scope: workspace          # optional
user-invocable: true      # optional — whether users can call it with /<name>
argument-hint: "<hint>"   # optional — shown when user invokes the skill
disable-model-invocation: false  # optional
license: MIT              # optional
---
# Skill content (markdown body)
```

## How to Add a New Skill

1. Create a directory: `.github/skills/<skill-name>/`
2. Add `SKILL.md` with YAML frontmatter and markdown body
3. Commit and push to the `main` branch

No build step or validation is required — GitHub Copilot reads these files directly from the repository.

## Existing Skills

- **karpathy-guidelines** (`code-guideline/SKILL.md`) — Behavioral guidelines to reduce common LLM coding mistakes (simplicity first, surgical changes, goal-driven execution). Invoked when writing, reviewing, or refactoring code.
- **sql-optimization** (`sql-optimization/SKILL.md`) — Reviews Go/GORM SQL queries for correctness: validates field names against table schemas, checks GORM method semantics (Pluck/First/Scan/Find), and flags deep-pagination performance risks with keyset pagination recommendations.
- **excel-upload-to-s3** (`excel-upload-to-s3/SKILL.md`) — Go module design for generating Excel files (`excelize` library) and uploading to S3 with presigned URLs. Covers three input formats (rows/maps/structs), column definitions, stream writing for large datasets, and injectable S3 client for testability.

## Repository Hosting

Remote: `https://github.com/koockty/my_copilot_skills.git`
Default branch: `main`
