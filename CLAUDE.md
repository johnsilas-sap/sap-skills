# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

32 production-tested Claude Code skills (plugins) for SAP technologies: BTP, CAP, Fiori, ABAP, HANA, and Analytics. Packages skills as a marketplace that Claude Code users install via the `plugin-dev` system.

**Version**: 2.1.8 | **Plugins**: 32 | **Last Updated**: 2026-04-02

## Setup & Commands

```bash
# Run once after cloning — activates pre-commit and pre-push validation hooks
git config core.hooksPath .githooks

# Sync all plugin.json manifests and regenerate marketplace.json
./scripts/sync-plugins.sh

# Preview what sync would change without writing files
./scripts/sync-plugins.sh --dry-run

# Validate individual concerns
./scripts/validate-frontmatter.sh        # SKILL.md YAML headers
./scripts/validate-json-schemas.sh       # plugin.json and marketplace.json
./scripts/validate-reserved-words.sh     # Blocked terms in name/description
```

## Architecture

### Plugin Structure (all 32 follow this pattern)

```
plugins/{name}/
├── .claude-plugin/plugin.json       # Plugin manifest (auto-generated from SKILL.md)
└── skills/{name}/
    ├── SKILL.md                     # Source of truth — YAML frontmatter + skill content
    ├── README.md                    # Keywords for discovery
    ├── references/                  # 5–30 SAP documentation reference files
    ├── templates/                   # Code templates (optional)
    ├── agents/                      # Specialized agents (0–4 per plugin)
    ├── commands/                    # Slash commands (0–5 per plugin)
    ├── hooks/hooks.json             # Event hooks (optional)
    ├── .mcp.json                    # MCP server config (6 plugins)
    └── .lsp.json                    # LSP config (sap-cap-capire, sap-sqlscript only)
```

### Marketplace Registry

`.claude-plugin/marketplace.json` is the central registry — **auto-generated, never hand-edit it**. The `source` field for each plugin must be `"./plugins/{name}"` (not a full path) to prevent cache bloat.

`sync-plugins.sh` orchestrates three phases:
1. Read global version from `marketplace.json`
2. Run `generate-plugin-manifests.sh` — converts each `SKILL.md` YAML frontmatter → `plugin.json`
3. Run `generate-marketplace.sh` — aggregates all `plugin.json` files into `marketplace.json`

### SKILL.md Frontmatter Schema

```yaml
---
name: sap-skill-name          # lowercase, kebab-case, ≤64 chars
description: |
  Multi-line description.     # ≤1024 chars
license: GPL-3.0
metadata:
  version: "2.1.2"
  last_verified: "2026-02-22"
  cap_version: "@sap/cds 9.7.x"   # Track SAP SDK versions here
---
```

### Plugin Categories

| Category | Count |
|---|---|
| SAP BTP Platform | 14 |
| Core Technologies | 7 |
| Data & Analytics | 5 |
| UI Development | 4 |
| Tooling & Development | 2 |

## Critical Directives

### Use plugin-dev First

For all general plugin development (creating skills, commands, agents, hooks, MCP integration, YAML frontmatter syntax) — invoke the official **plugin-dev** skills before writing any code:
`skill-development`, `plugin-structure`, `command-development`, `agent-development`, `hook-development`, `mcp-integration`

### Manual Review Only — No Automated Refactoring

**Never** use Python/shell scripts, `sed`, or `awk` to batch-rewrite skills. Skills require context-aware decisions; automation introduces subtle errors that break functionality. Always use `Read`/`Edit`/`Write` tools one file at a time with human review.

### Reserved Words Policy

`name` and `description` fields in any manifest MUST NOT contain `"official"`, `"anthropic"`, or `"claude"` — the CLI blocks these to prevent marketplace impersonation. Use `"AI coding assistant"` or `"the Code CLI"` instead.

## Maintenance

**Quarterly**: Check SAP SDK/package versions, update `last_verified` dates in SKILL.md frontmatter, re-test in production.

**On major SAP releases**: Review breaking changes, update templates and examples, document migration paths.

Next review: 2026-07-02
