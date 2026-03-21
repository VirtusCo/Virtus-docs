# VirtusCo Repository Split Plan
### Monorepo to Multi-Repo Migration
**Company:** Virtusco | virtusco.in
**Author:** Antony Austin
**Version:** 1.0
**Last Updated:** March 2026
**GitHub Organization:** github.com/VirtusCo (or github.com/austin207 until org created)

---

## Table of Contents

1. [Current State](#current-state)
2. [Why Split](#why-split)
3. [Proposed Repository Structure](#proposed-repository-structure)
4. [Repo 1 -- porter-ros (Core Robot Software)](#repo-1-porter-ros)
5. [Repo 2 -- virtusco-extensions (VS Code Extensions)](#repo-2-virtusco-extensions)
6. [Repo 3 -- virtus-sdk (API + Clients)](#repo-3-virtus-sdk)
7. [Repo 4 -- virtus-configs (Deployment Configs)](#repo-4-virtus-configs)
8. [Repo 5 -- virtus-fleet (Fleet Management)](#repo-5-virtus-fleet)
9. [Repo 6 -- virtus-docs (Plans + Internal Docs)](#repo-6-virtus-docs)
10. [What Goes Where -- Complete File Map](#what-goes-where-complete-file-map)
11. [Migration Execution Steps](#migration-execution-steps)
12. [CI/CD Per Repo](#cicd-per-repo)
13. [Cross-Repo Dependencies](#cross-repo-dependencies)
14. [Access Control Matrix](#access-control-matrix)
15. [Risk Register](#risk-register)

---

## Current State

Everything lives in a single monorepo: `github.com/austin207/Porter-ROS`

```
Porter-ROS/
├── porter_robot/                    # ROS 2 + ESP32 firmware + Flutter GUI + AI
├── porter-vscode-extension/         # VS Code extension #1
├── virtus-firmware-builder/         # VS Code extension #2
├── virtus-ai-studio/                # VS Code extension #3
├── virtus-ros2-studio/              # VS Code extension #4
├── virtus-hardware-dashboard/       # VS Code extension #5
├── virtus-simulation-manager/       # VS Code extension #6
├── virtus-pcb-studio/               # VS Code extension #7
├── virtusco-devtools-suite/         # VS Code extension #8 (master)
├── virtus-sdk/                      # REST API server + Python/Dart clients
├── virtus-configs/                  # Deployment profiles (sensitive)
├── virtus-observability/            # Off-robot monitoring tools
├── virtus-fleet-backend/            # Rust fleet server
├── Plans/                           # Internal strategy documents
└── .github/workflows/               # CI/CD
```

### Problems with Monorepo at Current Scale

- CI runs ROS 2 build + npm install + cargo build + flutter build for every push
- Airport operators who `pip install virtus-sdk` download the entire ROS 2 codebase
- Deployment configs (with SSH host paths) are in the same repo as public code
- VS Code extensions (TypeScript) have zero build dependency on ROS 2 (C++/Python)
- Fleet backend (Rust) has zero shared code with robot software
- Plans/strategy docs are visible to anyone with repo access

---

## Why Split

| Problem | Solution |
|---------|----------|
| CI takes too long | Each repo has focused CI |
| SDK consumers download robot source | SDK in separate repo |
| Sensitive configs in public repo | `virtus-configs` is private |
| Extension changes trigger robot CI | Separate repos |
| Rust compilation in same CI as Python | Fleet backend has its own CI |
| Internal docs visible to public | `virtus-docs` is private |

### What Stays Together (Single Release Unit)

ROS 2 packages + ESP32 firmware + Flutter GUI + AI assistant + virtus_msgs = **one release**.

A firmware version must match the bridge version must match the message definitions. Breaking one breaks all. These MUST be in the same repo with the same version tag.

---

## Proposed Repository Structure

| # | Repo | Content | Access | Stack | Release |
|---|------|---------|--------|-------|---------|
| 1 | `porter-ros` | Robot software | Public | C++/Python/Dart/C | `v0.X.Y` -- Docker + .bin + Flutter + AI |
| 2 | `virtusco-extensions` | All 8 VS Code extensions | Public | TypeScript/React | `v0.X.Y` -- 8 .vsix files |
| 3 | `virtus-sdk` | API server + clients | Public | Python/Dart | Semver `v1.X.Y` -- PyPI + pub |
| 4 | `virtus-configs` | Deployment profiles | **Private** | YAML/Python | Git tags per deployment |
| 5 | `virtus-fleet` | Rust fleet backend | Private | Rust/Python | Binary release |
| 6 | `virtus-docs` | Plans + company docs | **Private** | Markdown | No releases |

---

## Repo 1 -- porter-ros

**The core robot software. Everything that runs on the robot or compiles for the robot.**

```
porter-ros/
├── src/                              # ROS 2 colcon workspace
│   ├── virtus_msgs/                  # Message/service/action definitions
│   ├── ydlidar_driver/               # C++17 LIDAR driver (Apache 2.0)
│   ├── porter_lidar_processor/       # Python scan filter pipeline
│   ├── porter_lidar_processor_cpp/   # C++17 high-performance replacement
│   ├── porter_orchestrator/          # Python 9-state FSM
│   ├── porter_esp32_bridge/          # C++17 motor + sensor serial bridges
│   ├── porter_ai_assistant/          # Python AI + C++ hot path (pybind11)
│   ├── porter_gui/                   # Flutter touchscreen UI (Dart)
│   └── porter_observability/         # On-robot log bridge, metrics
├── esp32_firmware/                   # Zephyr RTOS for both ESP32s
├── tests/                            # Three-layer test infrastructure
├── docker/                           # Dev + prod Dockerfiles
├── skills/                           # ROS 2 + Zephyr reference files
└── .github/workflows/                # 9-job CI gate + 6-job release
```

### Release Process

```bash
git tag v0.5.0
git push origin v0.5.0
# CI builds: Docker image, ESP32 .bin files, Flutter bundle, SHA256SUMS
```

---

## Repo 2 -- virtusco-extensions

**All 8 VS Code extensions as an npm workspaces monorepo.**

```
virtusco-extensions/
├── package.json                      # npm workspaces: extensions/*
├── tsconfig.base.json                # Shared TypeScript config
├── shared/                           # PlatformUtils, webview-common
└── extensions/
    ├── porter-devtools/
    ├── firmware-builder/
    ├── ai-studio/
    ├── ros2-studio/
    ├── hardware-dashboard/
    ├── simulation-manager/
    ├── pcb-studio/
    └── devtools-suite/
```

### Why Extensions Monorepo

- Shared esbuild config with identical lessons learned baked in
- Shared TypeScript patterns (PlatformUtils, vscodeApi singleton)
- DevTools Suite depends on all 7 sub-extensions
- npm workspaces saves ~500 MB of duplicate node_modules

---

## Repo 3 -- virtus-sdk

**REST API server + installable Python/Dart clients.**

- Airport operators do `pip install virtus-sdk` without downloading ROS 2 source
- API version (`/v1/`, `/v2/`) is independently managed
- 81 tests covering client, models, and auth

---

## Repo 4 -- virtus-configs

**Deployment profiles, validation tools, audit log. PRIVATE repo.**

- Contains SSH hostnames, network topology, API key paths
- Deployment audit log is a compliance artifact
- Only ops team needs write access

---

## Repo 5 -- virtus-fleet

**Rust fleet management backend + Python off-robot observability tools.**

- Different tech stack (Rust) from robot software
- Deployed to cloud, not robot
- Different release cadence

---

## Repo 6 -- virtus-docs

**Internal strategy documents, devlogs, company context. PRIVATE repo.**

- Strategy docs reveal roadmap, pricing, hardware decisions
- Investors may need read access to docs but not code

---

## What Goes Where -- Complete File Map

| Current Location | Target Repo | Target Path |
|-----------------|-------------|-------------|
| `porter_robot/src/*` | `porter-ros` | `src/*` |
| `porter_robot/esp32_firmware/*` | `porter-ros` | `esp32_firmware/*` |
| `porter_robot/docker/*` | `porter-ros` | `docker/*` |
| `porter-vscode-extension/` | `virtusco-extensions` | `extensions/porter-devtools/` |
| `virtus-firmware-builder/` | `virtusco-extensions` | `extensions/firmware-builder/` |
| `virtus-sdk/` | `virtus-sdk` | `/` (root) |
| `virtus-configs/` | `virtus-configs` | `/` (root) |
| `virtus-fleet-backend/` | `virtus-fleet` | `backend/` |
| `Plans/` | `virtus-docs` | `Plans/` |

---

## Migration Execution Steps

### Phase 1 -- Create GitHub Organization (1 hour)

1. Create `github.com/VirtusCo` organization
2. Add team members
3. Set up billing (free tier initially)

### Phase 2 -- Create Repos with Preserved History (1 day)

Use `git-filter-repo` to split while preserving commit history:

```bash
git clone Porter-ROS porter-ros-split
cd porter-ros-split
git filter-repo --path porter_robot/ --path .github/workflows/
git filter-repo --path-rename porter_robot/:
```

### Phase 3 -- Set Up CI Per Repo (1 day)

### Phase 4 -- Cross-Repo References (2 hours)

### Phase 5 -- Update CLAUDE.md Files (2 hours)

### Phase 6 -- Archive Monorepo (1 hour)

---

## CI/CD Per Repo

| Repo | Jobs | Runtime |
|------|------|---------|
| `porter-ros` | ROS 2 build, colcon test, Ztest, Docker, Flutter | ~15 min |
| `virtusco-extensions` | npm lint, compile, package .vsix | ~3 min |
| `virtus-sdk` | pytest (81 tests), OpenAPI validation | ~30 sec |
| `virtus-configs` | Schema validation, business rules | ~10 sec |
| `virtus-fleet` | cargo check, cargo test, clippy | ~2 min |
| `virtus-docs` | None | -- |

---

## Cross-Repo Dependencies

```
porter-ros <--- virtus-sdk (server imports virtus_msgs at runtime)
     |               |
     |               +--- virtus-configs (deploy tool pushes config)
     |
     +--- virtusco-extensions (reads command IDs, HTTP API)
     |
     +--- virtus-fleet (pulls logs via SSH)
```

**No circular dependencies. No build-time cross-repo dependencies.**

---

## Access Control Matrix

| Repo | Antony | Danush | Allen | Azeem | Alwin | Public |
|------|--------|--------|-------|-------|-------|--------|
| `porter-ros` | Admin | Write | Read | Read | Read | Read |
| `virtusco-extensions` | Admin | Read | Read | Read | Read | Read |
| `virtus-sdk` | Admin | -- | Read | -- | Read | Read |
| `virtus-configs` | Admin | -- | -- | -- | -- | -- |
| `virtus-fleet` | Admin | -- | -- | -- | Read | -- |
| `virtus-docs` | Admin | Read | Read | Read | Read | -- |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Cross-repo breaking change | Medium | High | SDK pins virtus_msgs version; tests run against pinned schema |
| Developer commits to wrong repo | High (initially) | Low | CLAUDE.md in each repo states scope; git hooks warn |
| CI harder to debug across repos | Medium | Medium | Each repo's CI is self-contained |
| Git history lost during split | Low | Medium | `git-filter-repo` preserves full history per path |
| Private repos need paid plan | Certain | Low | GitHub Free allows unlimited private repos (3 collaborators) |

---

## Decision Summary

| Decision | Rationale |
|----------|-----------|
| ROS 2 + firmware + GUI + AI = one repo | Single release unit with version coupling |
| All 8 extensions = one repo | Shared build config, npm workspaces savings |
| SDK = separate repo | Independently versioned, airport operators don't need robot source |
| Configs = private separate repo | Sensitive deployment data, restricted access |
| Fleet backend = separate repo | Different stack (Rust), different target (cloud) |
| Docs = private separate repo | Strategy documents, different access needs |

---

*Execute this plan when the team agrees on the repo structure. Estimated migration effort: 2 days.*
