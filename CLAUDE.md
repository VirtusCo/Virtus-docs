# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working on Virtus Docs.

## Project Overview

Internal documentation repository for VirtusCo. Contains development plans, architecture documents, extension specifications, infrastructure designs, migration guides, and session development logs. This is a documentation-only repository -- no code, no builds, no tests.

**Repo**: `github.com/austin207/virtus-docs` | **Branch**: `main` | **License**: Proprietary (VirtusCo)

## Repository Structure

```
virtus-docs/
├── Plans/
│   ├── Extensions/             # 8 extension development plans
│   │   ├── porter-devtools.md
│   │   ├── firmware-builder.md
│   │   ├── ai-studio.md
│   │   ├── ros2-studio.md
│   │   ├── hardware-dashboard.md
│   │   ├── simulation-manager.md
│   │   ├── pcb-studio.md
│   │   └── devtools-suite.md
│   ├── Infrastructure/         # 6 infrastructure plans
│   │   ├── VDL.md              # Virtus Description Language (message types)
│   │   ├── VTI.md              # Virtus Test Infrastructure
│   │   ├── VCMS.md             # Virtus Config Management System
│   │   ├── VHAL.md             # Virtus Hardware Abstraction Layer
│   │   ├── VOS.md              # Virtus Observability Stack
│   │   └── SDK.md              # Virtus SDK (REST API + clients)
│   ├── Migration/              # Language migration plans
│   │   └── language-migration.md
│   └── REPO_SPLIT_PLAN.md     # Multi-repo migration plan
├── DevLogs/                    # Session-by-session development logs
│   ├── session-001.md
│   ├── session-002.md
│   └── ...
├── Architecture/               # System architecture documents
│   ├── hardware-stack.md
│   ├── ros2-topology.md
│   ├── ai-pipeline.md
│   └── firmware-protocol.md
├── OBJECTIVES.md               # Project objectives and timeline
└── COMPANY.md                  # VirtusCo context, team, vision
```

## Working with This Repo

This repository contains only Markdown files. There are no build steps, no tests, and no CI pipelines.

### Common Tasks

- **Adding a new plan**: Create a new `.md` file in the appropriate `Plans/` subdirectory
- **Adding a dev log**: Create `DevLogs/session-NNN.md` with the next sequential number
- **Updating architecture docs**: Edit files in `Architecture/`
- **Cross-referencing**: Use relative links between documents: `[SDK Plan](../Infrastructure/SDK.md)`

### Document Format

All documents follow this structure:

```markdown
# Title

## Overview
Brief description of what this document covers.

## Status
Current status: Draft | In Progress | Complete | Superseded

## Content
Main content sections...

## References
Links to related documents and external resources.
```

## Critical Rules

- **No code files** -- this repo is documentation only
- **No CI/CD** -- no workflows needed for a docs-only repo
- **Markdown only** -- all documents are `.md` files
- DevLogs are append-only -- never edit a previous session's log, create a new one
- Plans may be updated as implementation progresses -- note the date of each update
- Use relative links for cross-references within this repo
- Use full GitHub URLs when linking to other VirtusCo repos
- Keep documents focused -- one topic per file, split if it grows beyond ~500 lines

## Git Conventions

- Conventional commits: `<type>(<scope>): <description>`
- Types: `docs` (primary), `chore`, `fix`
- Scopes: `plans`, `devlogs`, `architecture`, `objectives`
- Example: `docs(plans): add fleet monitor extension spec`

## Related Repositories

| Repo | Description |
|------|-------------|
| `porter-ros` | Main robot software (ROS 2, firmware, AI) |
| `virtusco-extensions` | VS Code extension suite (8 extensions) |
| `virtus-sdk` | REST API server + Python/Dart clients |
| `virtus-configs` | Deployment profiles and config management |
| `virtus-fleet` | Fleet management backend + observability |
