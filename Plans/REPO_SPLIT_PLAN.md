# VirtusCo Repository Split Plan
### Monorepo → Multi-Repo Migration
**Company:** Virtusco | virtusco.in
**Author:** Antony Austin
**Version:** 1.0
**Last Updated:** March 2026
**GitHub Organization:** github.com/VirtusCo (or github.com/austin207 until org created)

---

## Table of Contents

1. [Current State](#1-current-state)
2. [Why Split](#2-why-split)
3. [Proposed Repository Structure](#3-proposed-repository-structure)
4. [Repo 1 — porter-ros (Core Robot Software)](#4-repo-1--porter-ros)
5. [Repo 2 — virtusco-extensions (VS Code Extensions)](#5-repo-2--virtusco-extensions)
6. [Repo 3 — virtus-sdk (API + Clients)](#6-repo-3--virtus-sdk)
7. [Repo 4 — virtus-configs (Deployment Configs)](#7-repo-4--virtus-configs)
8. [Repo 5 — virtus-fleet (Fleet Management)](#8-repo-5--virtus-fleet)
9. [Repo 6 — virtus-docs (Plans + Internal Docs)](#9-repo-6--virtus-docs)
10. [What Goes Where — Complete File Map](#10-what-goes-where--complete-file-map)
11. [Migration Execution Steps](#11-migration-execution-steps)
12. [CI/CD Per Repo](#12-cicd-per-repo)
13. [Cross-Repo Dependencies](#13-cross-repo-dependencies)
14. [Access Control Matrix](#14-access-control-matrix)
15. [Risk Register](#15-risk-register)

---

## 1. Current State

Everything lives in a single monorepo: `github.com/austin207/Porter-ROS`

```
Porter-ROS/                          ← ~500+ source files, 8 VS Code extensions,
│                                       6 infrastructure packages, Rust backend,
│                                       deployment configs, plans, docs
├── porter_robot/                    ← ROS 2 + ESP32 firmware + Flutter GUI + AI
├── porter-vscode-extension/         ← VS Code extension #1
├── virtus-firmware-builder/         ← VS Code extension #2
├── virtus-ai-studio/                ← VS Code extension #3
├── virtus-ros2-studio/              ← VS Code extension #4
├── virtus-hardware-dashboard/       ← VS Code extension #5
├── virtus-simulation-manager/       ← VS Code extension #6
├── virtus-pcb-studio/               ← VS Code extension #7
├── virtusco-devtools-suite/         ← VS Code extension #8 (master)
├── virtus-sdk/                      ← REST API server + Python/Dart clients
├── virtus-configs/                  ← Deployment profiles (sensitive)
├── virtus-observability/            ← Off-robot monitoring tools
├── virtus-fleet-backend/            ← Rust fleet server
├── Plans/                           ← Internal strategy documents
└── .github/workflows/               ← CI/CD
```

### Problems with Monorepo at Current Scale

- CI runs ROS 2 build + npm install + cargo build + flutter build for every push
- Airport operators who `pip install virtus-sdk` download the entire ROS 2 codebase
- Deployment configs (with SSH host paths) are in the same repo as public code
- VS Code extensions (TypeScript) have zero build dependency on ROS 2 (C++/Python)
- Fleet backend (Rust) has zero shared code with robot software
- Plans/strategy docs are visible to anyone with repo access

---

## 2. Why Split

| Problem | Solution |
|---------|----------|
| CI takes too long — builds everything on every push | Each repo has focused CI: robot CI doesn't run npm, extension CI doesn't run colcon |
| SDK consumers download robot source | SDK in separate repo — `pip install` only gets the client |
| Sensitive configs in public repo | `virtus-configs` is a private repo with restricted access |
| Extension changes trigger robot CI | Separate repos — extension PR doesn't rebuild ROS 2 |
| Rust compilation in same CI as Python | Fleet backend has its own `cargo test` CI |
| Internal docs visible to public | `virtus-docs` is private |

### What Stays Together (Single Release Unit)

ROS 2 packages + ESP32 firmware + Flutter GUI + AI assistant + virtus_msgs = **one release**.

A firmware version must match the bridge version must match the message definitions. Breaking one breaks all. These MUST be in the same repo with the same version tag.

---

## 3. Proposed Repository Structure

| # | Repo | Content | Access | Stack | Release |
|---|------|---------|--------|-------|---------|
| 1 | `porter-ros` | Robot software | Public | C++/Python/Dart/C | `v0.X.Y` → Docker + .bin + Flutter + AI |
| 2 | `virtusco-extensions` | All 8 VS Code extensions | Public | TypeScript/React | `v0.X.Y` → 8 .vsix files |
| 3 | `virtus-sdk` | API server + clients | Public | Python/Dart | Semver `v1.X.Y` → PyPI + pub |
| 4 | `virtus-configs` | Deployment profiles | **Private** | YAML/Python | Git tags per deployment |
| 5 | `virtus-fleet` | Rust fleet backend + off-robot tools | Private | Rust/Python | Binary release |
| 6 | `virtus-docs` | Plans + company docs | **Private** | Markdown | No releases |

---

## 4. Repo 1 — porter-ros

**The core robot software. Everything that runs on the robot or compiles for the robot.**

```
porter-ros/
├── CLAUDE.md
├── README.md
├── CHANGES.md
├── VERSION                           ← Single version for entire robot release
│
├── src/                              ← ROS 2 colcon workspace
│   ├── virtus_msgs/                  ← VDL — message/service/action definitions
│   ├── ydlidar_driver/               ← C++17 LIDAR driver (Apache 2.0)
│   ├── porter_lidar_processor/       ← Python scan filter pipeline
│   ├── porter_lidar_processor_cpp/   ← C++17 high-performance replacement
│   ├── porter_orchestrator/          ← Python 9-state FSM
│   ├── porter_esp32_bridge/          ← C++17 motor + sensor serial bridges
│   ├── porter_ai_assistant/          ← Python AI + C++ hot path (pybind11)
│   ├── porter_gui/                   ← Flutter touchscreen UI (Dart)
│   └── porter_observability/         ← VOS on-robot nodes (log bridge, metrics, journal)
│
├── esp32_firmware/                   ← Zephyr RTOS for both ESP32s
│   ├── motor_controller/
│   ├── sensor_fusion/
│   ├── common/
│   │   ├── protocol/                ← CRC16 + binary protocol
│   │   └── hal/                     ← VHAL drivers
│   └── tests/                       ← 178+ Ztest cases
│
├── tests/                            ← VTI — three-layer test infrastructure
│   ├── unit/                         ← 129 pytest cases
│   ├── integration/                  ← launch_testing + mock ESP32
│   ├── system/                       ← Gazebo headless scenarios
│   ├── mocks/                        ← MockESP32Bridge
│   ├── scenarios/                    ← Reusable test scenarios
│   ├── hil/                          ← Hardware-in-the-loop (manual)
│   └── bags/                         ← Recorded test bags
│
├── docker/
│   ├── Dockerfile.dev
│   ├── Dockerfile.prod
│   ├── docker-compose.dev.yml
│   └── docker-compose.prod.yml
│
├── skills/                           ← ROS 2 + Zephyr reference files
│   ├── (16 ROS 2 Jazzy skill files)
│   └── zephyr/ (12 Zephyr skill files)
│
├── DevLogs/                          ← Development session logs
│
└── .github/workflows/
    ├── verify.yml                    ← 9-job CI gate
    ├── build-release.yml             ← 6-job release pipeline
    └── test.yml                      ← VTI test suite
```

### What's Included and Why

| Component | Why It's Here |
|-----------|---------------|
| virtus_msgs (VDL) | Every ROS 2 package depends on it — must be in same colcon workspace |
| ESP32 firmware | Bridge protocol version must match — same release |
| Flutter GUI | Communicates with AI server — same release |
| AI assistant + C++ hot path | Runs on same RPi, same Docker container |
| VHAL | Part of ESP32 firmware build |
| VTI tests | Tests the packages in this repo |
| VOS on-robot nodes | Deployed with the robot software |
| Docker | Builds this repo's code specifically |

### What's NOT Included

| Component | Why NOT Here |
|-----------|-------------|
| VS Code extensions | Zero build dependency on ROS 2 — different tech stack (TypeScript) |
| SDK server | Independently versioned API — airport operators don't need robot source |
| Fleet backend | Rust binary deployed to cloud, not robot |
| Deployment configs | Sensitive (SSH hosts) — needs restricted access |
| Plans/docs | Strategy documents — different audience |

### Release Process

```bash
# Tag a release
git tag v0.5.0
git push origin v0.5.0

# CI builds automatically:
# → Docker image (multi-arch: amd64 + arm64)
# → motor_controller.bin + sensor_fusion.bin
# → porter-gui-linux-x64.tar.gz
# → SHA256SUMS
# → GitHub Release with all artifacts
```

---

## 5. Repo 2 — virtusco-extensions

**All 8 VS Code extensions as an npm workspaces monorepo.**

```
virtusco-extensions/
├── package.json                      ← npm workspaces: extensions/*
├── tsconfig.base.json                ← Shared TypeScript config
├── esbuild.shared.js                 ← Shared esbuild config (lessons learned baked in)
├── .eslintrc.json                    ← Shared ESLint config
├── CLAUDE.md                         ← All lessons learned from extension development
│
├── shared/                           ← Shared code across extensions
│   ├── platform-utils/               ← PlatformUtils (OS detection, WSL, paths)
│   ├── webview-common/               ← Common React components, VS Code CSS vars
│   └── vscode-helpers/               ← acquireVsCodeApi singleton pattern, etc.
│
├── extensions/
│   ├── porter-devtools/              ← Firmware uploader + RPi deployment
│   ├── firmware-builder/             ← Zephyr visual node canvas
│   ├── ai-studio/                    ← MLOps workbench (GPU, training, export, deploy)
│   ├── ros2-studio/                  ← ROS 2 dev environment (topics, graph, FSM, launch)
│   ├── hardware-dashboard/           ← Live hardware telemetry
│   ├── simulation-manager/           ← Gazebo launch profiles, Nav2 tuning
│   ├── pcb-studio/                   ← KiCad viewer + PCB layout + Gerber + DRC
│   └── devtools-suite/               ← Master meta-extension (manages all 7)
│
└── .github/workflows/
    └── build-extensions.yml          ← Builds all 8 .vsix files
```

### Why Extensions Monorepo (Not 8 Separate Repos)

| Reason | Detail |
|--------|--------|
| Shared build config | All 8 extensions use identical esbuild config with the same lessons learned (jsx: automatic, NODE_ENV: production, external native modules) |
| Shared TypeScript patterns | PlatformUtils, vscodeApi singleton, error boundaries — duplicated 8 times currently |
| DevTools Suite depends on all 7 | The master extension installs/manages all sub-extensions — easier in monorepo |
| Cross-extension fixes | When a command ID changes, one PR fixes both the extension and the Suite |
| npm workspaces | Shared `node_modules` — saves ~500MB of duplicate React/esbuild/zustand installs |

### Release Process

```bash
# Build all extensions
npm run build:all

# Package each as .vsix
npm run package:all
# → dist/porter-devtools-0.5.0.vsix
# → dist/virtus-firmware-builder-0.5.0.vsix
# → dist/virtus-ai-studio-0.5.0.vsix
# → ... (8 total)

# Tag and release
git tag v0.5.0
git push origin v0.5.0
# CI uploads all 8 .vsix to GitHub Release
```

---

## 6. Repo 3 — virtus-sdk

**REST API server (runs on robot) + Python client (pip installable) + Dart client (for Flutter).**

```
virtus-sdk/
├── CLAUDE.md
├── README.md
│
├── server/                           ← FastAPI, deployed on RPi 5
│   ├── virtus_api_server/
│   │   ├── main.py
│   │   ├── models.py                 ← Pydantic models (the API contract)
│   │   ├── auth.py                   ← 3-tier API key auth
│   │   ├── ros2_bridge.py            ← ROS 2 ↔ FastAPI bridge
│   │   └── routers/
│   ├── requirements.txt
│   └── install.sh                    ← systemd service installer
│
├── clients/
│   ├── python/                       ← pip install virtus-sdk
│   │   ├── virtus_sdk/
│   │   ├── setup.py
│   │   └── README.md
│   └── dart/                         ← Flutter dependency
│       ├── lib/
│       └── pubspec.yaml
│
├── docs/
│   ├── openapi.yaml                  ← 580-line API spec
│   ├── quickstart.md
│   ├── integration_guide.md          ← For airport operator IT teams
│   └── examples/
│
└── tests/                            ← 81 tests (client + models + auth)
```

### Why Separate

- Airport operators do `pip install virtus-sdk` — they should NOT download ROS 2 source
- API version (`/v1/`, `/v2/`) is independently managed from robot software version
- Breaking API change requires its own deprecation cycle (6 months)
- SDK docs are the public-facing product for integrations

### Versioning

```
SDK v1.0.0 ← compatible with robot v0.3.0 through v0.9.x
SDK v1.1.0 ← adds new endpoint (non-breaking)
SDK v2.0.0 ← breaking change (v1 still served for 6 months)
```

---

## 7. Repo 4 — virtus-configs

**Deployment profiles, validation tools, audit log. PRIVATE repo.**

```
virtus-configs/                       ← PRIVATE
├── defaults/
│   ├── firmware.yaml
│   ├── ros2.yaml
│   ├── nav2.yaml
│   └── ai.yaml
├── configs/deployments/
│   ├── dev-laptop.yaml
│   ├── dev-robot.yaml
│   ├── cial-unit-001.yaml
│   └── kempegowda-unit-001.yaml
├── schema/
│   └── deployment.schema.json
├── tools/
│   ├── virtus-deploy.py
│   ├── virtus-validate.py
│   ├── virtus-diff.py
│   └── virtus-audit.py
└── audit/
    └── deployment-log.jsonl          ← Append-only audit trail
```

### Why Separate and Private

- Contains SSH hostnames, network topology, API key file paths
- Deployment audit log is a compliance artifact — needs its own Git history
- Only ops team (Antony + future DevOps) needs write access
- Robots' exact configs should not be public knowledge

---

## 8. Repo 5 — virtus-fleet

**Rust fleet management backend + Python off-robot observability tools.**

```
virtus-fleet/                         ← Private (initially)
├── backend/                          ← Rust
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs
│   │   ├── api/
│   │   ├── aggregator/
│   │   ├── alerts/
│   │   ├── db/
│   │   └── models/
│   └── tests/
│
├── observability/                    ← Python off-robot tools
│   ├── aggregator.py
│   ├── alert_rules.py
│   ├── alert_delivery.py
│   ├── query.py
│   └── dashboard/                    ← FastAPI web dashboard
│       ├── app.py
│       └── templates/
│
└── .github/workflows/
    └── build-fleet.yml               ← cargo build + cargo test
```

### Why Separate

- Completely different tech stack (Rust) — `cargo build` in same CI as `colcon build` is wasteful
- Deployed to a cloud server, not to the robot
- Different release cadence — fleet backend updates don't require robot redeployment
- Will eventually be the SaaS product — may become its own business unit

---

## 9. Repo 6 — virtus-docs

**Internal strategy documents, devlogs, company context. PRIVATE repo.**

```
virtus-docs/                          ← PRIVATE
├── Plans/
│   ├── Extensions/
│   │   ├── VIRTUS_FIRMWARE_BUILDER_PLAN.md
│   │   ├── VIRTUS_AI_STUDIO_PLAN.md
│   │   ├── VIRTUS_ROS2_STUDIO_PLAN.md
│   │   ├── VIRTUS_HARDWARE_DASHBOARD_PLAN.md
│   │   ├── VIRTUS_SIMULATION_MANAGER_PLAN.md
│   │   ├── VIRTUS_PCB_REVIEWER_PLAN.md
│   │   ├── VIRTUS_FLEET_MONITOR_PLAN.md
│   │   └── VIRTUSCO_DEVTOOLS_SUITE_PLAN.md
│   ├── Infrastructure/
│   │   ├── SDK_PLAN.md
│   │   ├── VCMS_PLAN.md
│   │   ├── VDL_PLAN.md
│   │   ├── VHAL_PLAN.md
│   │   ├── VOS_PLAN.md
│   │   └── VTI_PLAN.md
│   └── Migration/
│       └── VIRTUS_LANGUAGE_MIGRATION_PLAN.md
├── REPO_SPLIT_PLAN.md                ← This document
├── OBJECTIVES.md
├── COMPANY.md
├── DevLogs/
└── pitch-materials/
```

### Why Separate and Private

- Strategy docs reveal roadmap, pricing plans, hardware decisions — competitive intel
- Investors/advisors may need read access to docs but not code
- DevLogs are informal session notes — don't belong in a code repo

---

## 10. What Goes Where — Complete File Map

| Current Location | Target Repo | Target Path |
|-----------------|-------------|-------------|
| `porter_robot/src/*` | `porter-ros` | `src/*` |
| `porter_robot/esp32_firmware/*` | `porter-ros` | `esp32_firmware/*` |
| `porter_robot/docker/*` | `porter-ros` | `docker/*` |
| `porter_robot/tests/*` | `porter-ros` | `tests/*` |
| `porter_robot/skills/*` | `porter-ros` | `skills/*` |
| `porter_robot/DevLogs/*` | `virtus-docs` | `DevLogs/*` |
| `porter_robot/OBJECTIVES.md` | `virtus-docs` | `OBJECTIVES.md` |
| `porter_robot/COMPANY.md` | `virtus-docs` | `COMPANY.md` |
| `porter-vscode-extension/` | `virtusco-extensions` | `extensions/porter-devtools/` |
| `virtus-firmware-builder/` | `virtusco-extensions` | `extensions/firmware-builder/` |
| `virtus-ai-studio/` | `virtusco-extensions` | `extensions/ai-studio/` |
| `virtus-ros2-studio/` | `virtusco-extensions` | `extensions/ros2-studio/` |
| `virtus-hardware-dashboard/` | `virtusco-extensions` | `extensions/hardware-dashboard/` |
| `virtus-simulation-manager/` | `virtusco-extensions` | `extensions/simulation-manager/` |
| `virtus-pcb-studio/` | `virtusco-extensions` | `extensions/pcb-studio/` |
| `virtusco-devtools-suite/` | `virtusco-extensions` | `extensions/devtools-suite/` |
| `virtus-sdk/` | `virtus-sdk` | `/` (root) |
| `virtus-configs/` | `virtus-configs` | `/` (root) |
| `virtus-fleet-backend/` | `virtus-fleet` | `backend/` |
| `virtus-observability/` | `virtus-fleet` | `observability/` |
| `Plans/` | `virtus-docs` | `Plans/` |
| `.github/workflows/verify.yml` | `porter-ros` | `.github/workflows/verify.yml` |
| `.github/workflows/test.yml` | `porter-ros` | `.github/workflows/test.yml` |
| `.github/workflows/build-release.yml` | `porter-ros` | `.github/workflows/build-release.yml` |

---

## 11. Migration Execution Steps

### Phase 1 — Create GitHub Organization (1 hour)

1. Create `github.com/VirtusCo` organization
2. Add team members
3. Set up billing (free tier is fine initially)

### Phase 2 — Create Repos with Preserved History (1 day)

Use `git filter-branch` or `git-filter-repo` to split the monorepo while preserving commit history for each subset of files:

```bash
# Example: extract porter_robot/ into its own repo
git clone Porter-ROS porter-ros-split
cd porter-ros-split
git filter-repo --path porter_robot/ --path .github/workflows/ --path CLAUDE.md --path README.md
# Flatten: move porter_robot/* to root
git filter-repo --path-rename porter_robot/:
```

Repeat for each repo with its respective file paths.

### Phase 3 — Set Up CI Per Repo (1 day)

- `porter-ros`: verify.yml + test.yml + build-release.yml (already exist, minor path fixes)
- `virtusco-extensions`: new build-extensions.yml (npm workspaces build)
- `virtus-sdk`: new test-sdk.yml (pytest)
- `virtus-fleet`: new build-fleet.yml (cargo build + test)

### Phase 4 — Cross-Repo References (2 hours)

- `virtus-sdk/server/ros2_bridge.py` imports from `virtus_msgs` → document that SDK server runs inside the porter-ros Docker container (virtus_msgs is available at runtime)
- `virtusco-extensions` reference porter-ros commands → update command IDs if needed
- CI workflows that reference other repos → use GitHub Actions workflow dispatch or artifact downloads

### Phase 5 — Update CLAUDE.md Files (2 hours)

Each repo gets its own CLAUDE.md with repo-specific instructions, build commands, and cross-repo dependency documentation.

### Phase 6 — Archive Monorepo (1 hour)

```bash
# Rename original monorepo
gh repo rename Porter-ROS Porter-ROS-archive
# Mark as archived
gh repo archive Porter-ROS-archive
```

Keep the archived monorepo readable for 6 months. Delete after all teams have migrated.

---

## 12. CI/CD Per Repo

| Repo | CI Trigger | Jobs | Runtime |
|------|-----------|------|---------|
| `porter-ros` | Push + PR | ROS 2 build, colcon test, Ztest, Docker build, Flutter, integration | ~15 min |
| `virtusco-extensions` | Push + PR | npm lint, npm compile (all 8), package .vsix | ~3 min |
| `virtus-sdk` | Push + PR | pytest (81 tests), OpenAPI validation | ~30 sec |
| `virtus-configs` | Push | Schema validation, business rules check | ~10 sec |
| `virtus-fleet` | Push + PR | cargo check, cargo test (21 tests), cargo clippy | ~2 min |
| `virtus-docs` | None | No CI needed (documentation only) | — |

**Total CI time across all repos: ~20 min** (vs ~30 min in monorepo because jobs don't run unnecessarily)

---

## 13. Cross-Repo Dependencies

```
porter-ros ←──────────────────── virtus-sdk (server runs on robot, imports virtus_msgs)
     │                                │
     │                                ├── virtus-configs (deploy tool pushes config to robot)
     │                                │
     ├── virtusco-extensions ─────────┘ (extensions open robot commands, read config)
     │         │
     │         └── virtus-fleet (Fleet Monitor extension reads from fleet backend)
     │
     └── virtus-fleet (aggregator pulls logs from robot via SSH)
```

### Dependency Rules

1. `porter-ros` depends on NOTHING external (self-contained robot software)
2. `virtus-sdk` server depends on `porter-ros` at runtime (imports virtus_msgs) — resolved by running inside the same Docker container
3. `virtusco-extensions` depend on `porter-ros` and `virtus-sdk` only via command IDs and HTTP API — no build dependency
4. `virtus-fleet` depends on `porter-ros` only via SSH (pulls log files) — no build dependency
5. `virtus-configs` depends on `porter-ros` only via SSH (pushes config) — no build dependency

**No circular dependencies. No build-time cross-repo dependencies.**

---

## 14. Access Control Matrix

| Repo | Antony | Danush | Allen | Azeem | Alwin | Airport IT | Public |
|------|--------|--------|-------|-------|-------|-----------|--------|
| `porter-ros` | Admin | Write (firmware) | Read | Read | Read | — | Read |
| `virtusco-extensions` | Admin | Read | Read | Read | Read | — | Read |
| `virtus-sdk` | Admin | — | Read | — | Read | Read | Read |
| `virtus-configs` | Admin | — | — | — | — | — | — |
| `virtus-fleet` | Admin | — | — | — | Read | — | — |
| `virtus-docs` | Admin | Read | Read | Read | Read | — | — |

---

## 15. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Cross-repo breaking change (virtus_msgs schema change breaks SDK) | Medium | High | SDK server pins virtus_msgs version; SDK tests run against mock data matching pinned schema |
| Developer forgets which repo to commit to | High (initially) | Low | CLAUDE.md in each repo clearly states scope; git hooks warn if committing to wrong repo |
| CI becomes harder to debug across repos | Medium | Medium | Each repo's CI is self-contained; cross-repo integration test runs weekly (not per-push) |
| Git history lost during split | Low | Medium | Use `git-filter-repo` which preserves full commit history per file path |
| Private repos need GitHub paid plan | Certain | Low | GitHub Free allows unlimited private repos with 3 collaborators; upgrade to Team ($4/user/mo) when team grows |
| Extension monorepo npm workspaces setup is complex | Medium | Low | Use Turborepo or Nx for monorepo tooling; or keep simple npm workspaces |
| Deployment workflow spans two repos (porter-ros + virtus-configs) | Certain | Medium | `virtus-deploy.py` clones porter-ros release artifacts; configs repo only contains YAML + tools |

---

## Decision Summary

| Decision | Rationale |
|----------|-----------|
| ROS 2 + firmware + GUI + AI = one repo | Single release unit. Version coupling is real and important. |
| All 8 extensions = one repo | Shared build config, DevTools Suite dependency, npm workspaces savings |
| SDK = separate repo | Independently versioned. Airport operators shouldn't download ROS 2 source. |
| Configs = private separate repo | Sensitive deployment data. Restricted access. Audit trail. |
| Fleet backend = separate repo | Different stack (Rust), different deployment target (cloud), different cadence. |
| Docs = private separate repo | Strategy documents are not code. Different access needs. |

---

*Execute this plan when the team agrees on the repo structure. Estimated migration effort: 2 days.*
