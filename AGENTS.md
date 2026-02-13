# Agent Instructions

Productivity skills for Microsoft ecosystem professionals — designed for AI coding agents (GitHub Copilot, Claude Code, etc.).

## Purpose

This repository contains reusable skills that help with real enterprise tasks in the Microsoft tech stack:
- Azure Data Explorer/Kusto querying
- Azure-branded PowerPoint generation
- Systematic skill creation from docs/codebases

See [README.md](README.md) for:
- Available skills and installation commands
- Repository structure
- Known limitations

## Skill Structure

Each skill in `skills/` follows a standard format:

```
skills/<skill-name>/
├── SKILL.md                       # Main skill file with frontmatter
├── feature-area-skill-resources/  # Detailed resources by topic (optional)
├── investigation-reports/         # Research artifacts (optional)
└── submodules/                    # Git submodules with source docs (optional)
```

### SKILL.md Format

```yaml
---
name: skill-name
description: |
  Description with trigger phrases.
  Use when: (1) condition, (2) condition, (3) keyword triggers.
---

# Skill Title

Content organized by feature area...
```

## Using Skills

Install a skill:
```bash
npx skills add johnsonshi/skills365 --skill <skill-name>
```

## Creating New Skills

Use the `deep-knowledge-skill-creator` skill to systematically create new skills from documentation repositories through a multi-phase workflow.
