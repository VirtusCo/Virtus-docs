# Virtusco DevTools Suite — Master Extension
### Detailed Development Plan
**Project:** Meta-extension that installs, manages, and orchestrates all 8 Virtusco VS Code extensions as a unified suite  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [The 8 Extensions](#2-the-8-extensions)
3. [Extension Architecture](#3-extension-architecture)
4. [Module 1 — Installer & Dependency Manager](#4-module-1--installer--dependency-manager)
5. [Module 2 — Suite Dashboard](#5-module-2--suite-dashboard)
6. [Module 3 — Cross-Extension Communication Bus](#6-module-3--cross-extension-communication-bus)
7. [Module 4 — Shared Configuration](#7-module-4--shared-configuration)
8. [Module 5 — Update Manager](#8-module-5--update-manager)
9. [Module 6 — Workspace Bootstrapper](#9-module-6--workspace-bootstrapper)
10. [Extension Dependency Graph](#10-extension-dependency-graph)
11. [Shared Component Library](#11-shared-component-library)
12. [VS Code Marketplace Strategy](#12-vs-code-marketplace-strategy)
13. [Windows & Cross-Platform Support](#13-windows--cross-platform-support)
14. [Production Quality Standards](#14-production-quality-standards)
15. [Phased Build Plan](#15-phased-build-plan)
16. [Tech Stack Summary](#16-tech-stack-summary)
17. [Risk Register](#17-risk-register)

---

## 1. Overview & Vision

### What the Master Extension Is

A single installable VS Code extension — **`virtusco.devtools-suite`** — that:

1. **Installs all 8 sub-extensions** with one click (or `code --install-extension virtusco.devtools-suite`)
2. **Provides a unified suite dashboard** — one panel that shows the status and quick-launch for all tools
3. **Runs a cross-extension communication bus** — so extensions can share data (e.g., Hardware Dashboard fires an alert that ROS 2 Studio surfaces in the topic monitor)
4. **Manages shared configuration** — SSH credentials, robot inventory, workspace paths — defined once, used by all extensions
5. **Bootstraps a new Virtusco workspace** — clones the Porter-ROS repo, installs all dependencies, sets up all configs in one command

### Why a Master Extension Matters

Without it, onboarding a new team member (or setting up a new laptop) means:
- Installing 8 extensions manually
- Configuring SSH credentials in 3 different extensions
- Setting workspace paths in 5 different places
- Discovering that Hardware Dashboard and ROS 2 Studio don't know about each other's data

With the master extension:
```bash
code --install-extension virtusco.devtools-suite
# → installs all 8 extensions
# → opens suite dashboard
# → runs workspace setup wizard
# Done. Everything configured.
```

### The Suite Identity

The master extension gives Virtusco's dev toolchain a **product identity** — not 8 random extensions, but "Virtusco DevTools Suite v1.0" — something that can be shown to investors, mentioned in pitches, and eventually open-sourced or licensed.

---

## 2. The 8 Extensions

| # | Extension ID | Name | Status | Priority |
|---|---|---|---|---|
| 1 | `virtusco.firmware-uploader` | Firmware Uploader | ✅ Built | Done |
| 2 | `virtusco.firmware-builder` | Firmware Builder | 📋 Planned | 2nd |
| 3 | `virtusco.ai-studio` | AI Studio | 📋 Planned | 3rd |
| 4 | `virtusco.ros2-studio` | ROS 2 Studio | 📋 Proposed | 4th |
| 5 | `virtusco.hardware-dashboard` | Hardware Dashboard | 📋 Proposed | 5th |
| 6 | `virtusco.simulation-manager` | Simulation Manager | 📋 Proposed | 6th |
| 7 | `virtusco.pcb-reviewer` | PCB Schematic Reviewer | 📋 Proposed | 7th |
| 8 | `virtusco.fleet-monitor` | Fleet Monitor | 🔮 Future | 8th |
| **M** | `virtusco.devtools-suite` | **DevTools Suite (Master)** | 📋 Planned | Build after #2 |

---

## 3. Extension Architecture

```
virtusco-devtools-suite/
├── package.json                          ← Manifest declaring all 8 as extensionDependencies
├── tsconfig.json
├── esbuild.config.js
│
├── src/
│   ├── extension.ts                      ← Entry: registers suite commands, creates dashboard
│   ├── panel/
│   │   └── SuiteDashboardPanel.ts        ← Master WebviewPanel
│   ├── installer/
│   │   ├── ExtensionInstaller.ts         ← Installs missing sub-extensions via VS Code API
│   │   ├── ExtensionVersionChecker.ts    ← Checks installed versions vs latest
│   │   └── DependencyChecker.ts          ← Checks system deps: uv, git, ros2, kicad, etc.
│   ├── bus/
│   │   ├── EventBus.ts                   ← Cross-extension event pub/sub via VS Code commands
│   │   └── BusEvents.ts                  ← All typed event definitions
│   ├── config/
│   │   ├── SharedConfig.ts               ← Reads/writes virtusco.json shared config
│   │   ├── SSHConfigManager.ts           ← Robot SSH credentials (VS Code SecretStorage)
│   │   └── WorkspaceConfig.ts            ← Workspace-level settings
│   ├── bootstrap/
│   │   ├── WorkspaceBootstrapper.ts      ← Clone repo, install deps, configure workspace
│   │   └── SetupWizard.ts                ← Step-by-step first-run wizard
│   └── updater/
│       └── SuiteUpdater.ts               ← Checks for updates across all extensions
│
└── webview-ui/
    └── src/
        ├── App.tsx
        ├── pages/
        │   ├── DashboardPage.tsx         ← Suite overview with all extension tiles
        │   ├── InstallerPage.tsx         ← Extension install status + one-click install
        │   ├── ConfigPage.tsx            ← Shared configuration editor
        │   ├── SetupWizardPage.tsx       ← First-run workspace bootstrapper
        │   └── UpdatesPage.tsx           ← Available updates across all extensions
        └── components/
            ├── ExtensionTile.tsx         ← Status tile per extension (installed/version/open)
            ├── DependencyRow.tsx         ← System dependency check row
            ├── SetupStep.tsx             ← Wizard step component
            └── ConfigSection.tsx         ← Shared config section editor
```

---

## 4. Module 1 — Installer & Dependency Manager

### VS Code Extension Installation API

VS Code provides `vscode.commands.executeCommand('workbench.extensions.installExtension', extensionId)` to install extensions programmatically.

```typescript
// installer/ExtensionInstaller.ts
export class ExtensionInstaller {

  static readonly SUITE_EXTENSIONS = [
    { id: 'virtusco.firmware-uploader',  name: 'Firmware Uploader',   required: true  },
    { id: 'virtusco.firmware-builder',   name: 'Firmware Builder',    required: false },
    { id: 'virtusco.ai-studio',          name: 'AI Studio',           required: false },
    { id: 'virtusco.ros2-studio',        name: 'ROS 2 Studio',        required: false },
    { id: 'virtusco.hardware-dashboard', name: 'Hardware Dashboard',  required: false },
    { id: 'virtusco.simulation-manager', name: 'Simulation Manager',  required: false },
    { id: 'virtusco.pcb-reviewer',       name: 'PCB Reviewer',        required: false },
    { id: 'virtusco.fleet-monitor',      name: 'Fleet Monitor',       required: false },
  ];

  static isInstalled(extensionId: string): boolean {
    return vscode.extensions.getExtension(extensionId) !== undefined;
  }

  static getVersion(extensionId: string): string | null {
    return vscode.extensions.getExtension(extensionId)?.packageJSON?.version ?? null;
  }

  static async install(extensionId: string): Promise<void> {
    await vscode.commands.executeCommand(
      'workbench.extensions.installExtension', extensionId
    );
  }

  static async installAll(): Promise<void> {
    for (const ext of this.SUITE_EXTENSIONS) {
      if (!this.isInstalled(ext.id)) {
        await this.install(ext.id);
        await new Promise(r => setTimeout(r, 1000)); // Brief delay between installs
      }
    }
  }
}
```

### System Dependency Checker

Checks all system-level dependencies required by the suite:

```typescript
// installer/DependencyChecker.ts
export interface DepCheck {
  name:       string;
  command:    string;           // Command to check presence
  version_re: RegExp;           // Regex to extract version from output
  required_by: string[];        // Which extensions need this
  install_url: string;          // Where to install if missing
  windows_cmd?: string;         // Alternative command on Windows
}

export const SUITE_DEPENDENCIES: DepCheck[] = [
  { name: 'git',         command: 'git --version',          version_re: /git version (.+)/,     required_by: ['all'],                  install_url: 'https://git-scm.com' },
  { name: 'uv',          command: 'uv --version',           version_re: /uv (.+)/,              required_by: ['ai-studio'],            install_url: 'https://docs.astral.sh/uv/getting-started/installation/' },
  { name: 'python3',     command: 'python3 --version',      version_re: /Python (.+)/,          required_by: ['ai-studio', 'ros2-studio'], install_url: 'https://www.python.org' },
  { name: 'ros2',        command: 'ros2 --version',         version_re: /ros2 cli (.+)/,        required_by: ['ros2-studio', 'sim-manager'], install_url: 'https://docs.ros.org/en/jazzy/Installation.html',
    windows_cmd: 'wsl ros2 --version' },
  { name: 'colcon',      command: 'colcon --version',       version_re: /colcon (.+)/,          required_by: ['ros2-studio'],          install_url: 'https://colcon.readthedocs.io',
    windows_cmd: 'wsl colcon --version' },
  { name: 'west',        command: 'west --version',         version_re: /West version (.+)/,   required_by: ['firmware-builder', 'firmware-uploader'], install_url: 'https://docs.zephyrproject.org/latest/develop/west/install.html' },
  { name: 'nvidia-smi',  command: 'nvidia-smi --version',  version_re: /NVIDIA-SMI (.+)/,      required_by: ['ai-studio'],            install_url: 'https://developer.nvidia.com/cuda-downloads',
    windows_cmd: 'C:\\Windows\\System32\\nvidia-smi.exe --version' },
  { name: 'node',        command: 'node --version',         version_re: /v(.+)/,               required_by: ['all'],                  install_url: 'https://nodejs.org' },
  { name: 'ssh',         command: 'ssh -V',                 version_re: /OpenSSH_(.+)/,        required_by: ['hardware-dashboard', 'fleet-monitor'], install_url: 'https://www.openssh.com' },
];
```

### Installer Page UI

```
┌── Virtusco DevTools Suite — Installer ───────────────────────────────────┐
│                                                                           │
│  Extensions                                                               │
│  ✓ Firmware Uploader       v1.2.0  ● Installed                          │
│  ✓ Firmware Builder        v0.3.1  ● Installed                          │
│  ✗ AI Studio               —       ○ Not installed   [Install]           │
│  ✗ ROS 2 Studio            —       ○ Not installed   [Install]           │
│  ✗ Hardware Dashboard      —       ○ Not installed   [Install]           │
│  ✗ Simulation Manager      —       ○ Not installed   [Install]           │
│  ✗ PCB Reviewer            —       ○ Not installed   [Install]           │
│  ✗ Fleet Monitor           —       ○ Not installed   [Install]           │
│                                                   [Install All Missing]  │
│                                                                           │
│  System Dependencies                                                      │
│  ✓ git              2.43.0                                               │
│  ✓ uv               0.5.1                                                │
│  ✓ python3          3.12.3                                               │
│  ✓ node             22.1.0                                               │
│  ⚠ ros2             NOT FOUND   Required by: ROS 2 Studio, Sim Manager   │
│    [Install guide →]   (Linux: sudo apt install ros-jazzy-desktop)       │
│  ✓ west             1.2.0                                                │
│  ✓ nvidia-smi       551.23.04                                            │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Module 2 — Suite Dashboard

### Dashboard Overview

The suite dashboard is the **home screen** of the Virtusco dev environment. It shows the status of all extensions and the robot system at a glance.

```
┌── Virtusco DevTools Suite v1.0 ──────────────────────────────────────────┐
│  Workspace: Porter-ROS  Branch: main  Last commit: 2h ago               │
│                                                                           │
│  ROBOT STATUS (from Hardware Dashboard + ROS 2 Studio)                  │
│  ● ROS 2: Connected (Jazzy)     ● Serial: /dev/ttyUSB2 connected        │
│  ● 12V: 11.9V ✓                 ● Motors: Idle                          │
│                                                                           │
│  EXTENSIONS                                                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐             │
│  │ 📟 Firmware    │  │ 🔧 Firmware    │  │ 🤖 AI Studio   │             │
│  │    Uploader    │  │    Builder     │  │                │             │
│  │ ✓ Installed    │  │ ✓ Installed    │  │ ✓ Installed    │             │
│  │ [Open]         │  │ [Open]         │  │ [Open]         │             │
│  └────────────────┘  └────────────────┘  └────────────────┘             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐             │
│  │ 📡 ROS 2       │  │ ⚡ Hardware    │  │ 🌐 Simulation  │             │
│  │    Studio      │  │    Dashboard   │  │    Manager     │             │
│  │ ✓ Installed    │  │ ✓ Installed    │  │ ○ Not installed│             │
│  │ [Open]         │  │ [Open]         │  │ [Install]      │             │
│  └────────────────┘  └────────────────┘  └────────────────┘             │
│  ┌────────────────┐  ┌────────────────┐                                 │
│  │ 🔌 PCB         │  │ 🛰 Fleet       │                                 │
│  │    Reviewer    │  │    Monitor     │                                 │
│  │ ✓ Installed    │  │ ○ Not installed│                                 │
│  │ [Open]         │  │ [Install]      │                                 │
│  └────────────────┘  └────────────────┘                                 │
│                                                                           │
│  ALERTS (from all extensions)                                            │
│  ⚠ Hardware Dashboard: Left motor current elevated (2.1A)               │
│  ℹ Firmware Builder: esp32.overlay not regenerated since firmware change │
│                                                                           │
│  [Open Extension →]  links each tile to its extension panel             │
└───────────────────────────────────────────────────────────────────────────┘
```

The **ROBOT STATUS** section in the dashboard aggregates live data from the Hardware Dashboard and ROS 2 Studio extensions via the event bus — giving a health-at-a-glance without opening each extension.

The **ALERTS** section shows alerts from all extensions in one place — so Antony doesn't miss a Hardware Dashboard alert just because that panel isn't open.

---

## 6. Module 3 — Cross-Extension Communication Bus

### Problem

VS Code extensions run in isolated processes. They cannot directly call each other's functions. The standard way they communicate is via **VS Code commands** (each extension can register commands that other extensions can call) and via **shared workspace state** (a file both extensions read/write).

### Event Bus Design

```typescript
// bus/BusEvents.ts — All cross-extension event types

// Hardware Dashboard → others
export const HW_MOTOR_ALERT       = 'virtusco.bus.hw.motorAlert';
export const HW_POWER_ALERT       = 'virtusco.bus.hw.powerAlert';
export const HW_TELEMETRY_UPDATE  = 'virtusco.bus.hw.telemetryUpdate';

// ROS 2 Studio → others
export const ROS_NODE_DEAD        = 'virtusco.bus.ros.nodeDead';
export const ROS_FSM_STATE_CHANGE = 'virtusco.bus.ros.fsmStateChange';
export const ROS_CONNECTED        = 'virtusco.bus.ros.connected';

// AI Studio → others
export const AI_TRAINING_STARTED  = 'virtusco.bus.ai.trainingStarted';
export const AI_TRAINING_DONE     = 'virtusco.bus.ai.trainingDone';
export const AI_DEPLOY_COMPLETE   = 'virtusco.bus.ai.deployComplete';

// Firmware Builder → others
export const FW_GENERATED         = 'virtusco.bus.fw.generated';
export const FW_FLASH_COMPLETE    = 'virtusco.bus.fw.flashComplete';

// Suite → all
export const SUITE_WORKSPACE_INIT = 'virtusco.bus.suite.workspaceInit';
export const SUITE_ROBOT_CONFIG   = 'virtusco.bus.suite.robotConfig';
```

```typescript
// bus/EventBus.ts
export class EventBus {

  // Any extension publishes an event
  static publish(eventId: string, payload: object): void {
    vscode.commands.executeCommand(eventId, payload);
  }

  // Any extension subscribes to an event
  static subscribe(eventId: string, handler: (payload: any) => void): vscode.Disposable {
    return vscode.commands.registerCommand(eventId, handler);
  }
}
```

### Cross-Extension Interactions Enabled by the Bus

| Publisher | Event | Subscribers | Effect |
|---|---|---|---|
| Hardware Dashboard | `HW_MOTOR_ALERT` | Suite Dashboard | Alert shown in suite dashboard alerts panel |
| Hardware Dashboard | `HW_MOTOR_ALERT` | ROS 2 Studio | Highlighted in topic monitor as correlated event |
| Firmware Builder | `FW_FLASH_COMPLETE` | Hardware Dashboard | Prompts hardware dashboard to reconnect serial |
| Firmware Builder | `FW_GENERATED` | PCB Reviewer | Auto-runs pinout sync check after new DTS overlay generated |
| ROS 2 Studio | `ROS_FSM_STATE_CHANGE` | Suite Dashboard | Live FSM state shown in suite dashboard robot status |
| AI Studio | `AI_DEPLOY_COMPLETE` | Fleet Monitor | Prompts fleet monitor to push new model to deployed robots |
| Suite | `SUITE_ROBOT_CONFIG` | All extensions | All extensions receive updated SSH config when user changes it in suite |

---

## 7. Module 4 — Shared Configuration

### Problem

Currently (without the suite), each extension has its own settings:
- Firmware Uploader: serial port path
- Hardware Dashboard: telemetry serial port path
- ROS 2 Studio: rosbridge port
- Fleet Monitor: SSH credentials per robot

This means the same SSH hostname is configured in 3 different places. When it changes (e.g., robot gets a new IP), it must be updated 3 times.

### Shared Config File

```json
// .virtusco/suite.config.json  — committed to workspace
{
  "workspace": {
    "ros_ws_root":    ".",
    "esp32_fw_root":  "esp32_firmware/",
    "hardware_root":  "hardware/"
  },
  "robots": [
    {
      "id":           "virtus-dev",
      "name":         "Development Robot",
      "ssh_host":     "192.168.1.100",
      "ssh_user":     "pi",
      "ssh_key_id":   "virtus-dev-key",
      "telemetry_port_linux": "/dev/ttyUSB2",
      "telemetry_port_win":   "COM6",
      "firmware_port_linux":  "/dev/ttyUSB0",
      "firmware_port_win":    "COM4"
    }
  ],
  "ros2": {
    "rosbridge_port":  9090,
    "ros_distro":      "jazzy"
  },
  "ai_studio": {
    "venv_root":    ".virtus-ai/",
    "artifacts":    ".virtus-ai/artifacts/"
  },
  "fleet": {
    "backend_url":  "https://api.virtusco.in",
    "airport_id":   "CIAL"
  }
}
```

SSH credentials (private keys) are NOT in this file — they live in VS Code SecretStorage. The config file only contains the key ID as a reference.

### SharedConfig.ts

```typescript
// config/SharedConfig.ts
export class SharedConfig {

  static async load(): Promise<SuiteConfig> {
    const configUri = vscode.Uri.joinPath(workspaceRoot, '.virtusco', 'suite.config.json');
    const bytes     = await vscode.workspace.fs.readFile(configUri);
    return JSON.parse(bytes.toString()) as SuiteConfig;
  }

  static async save(config: SuiteConfig): Promise<void> {
    const json = JSON.stringify(config, null, 2);
    const uri  = vscode.Uri.joinPath(workspaceRoot, '.virtusco', 'suite.config.json');
    await vscode.workspace.fs.writeFile(uri, Buffer.from(json));
    // Broadcast to all extensions
    EventBus.publish(SUITE_ROBOT_CONFIG, config);
  }

  // Any extension calls this to get shared config — no file reading needed
  static getRobotSSHHost(robotId: string): string {
    return this.config.robots.find(r => r.id === robotId)?.ssh_host ?? '';
  }
}
```

### Config Editor UI

```
┌── Shared Configuration ──────────────────────────────────────────────────┐
│  File: .virtusco/suite.config.json  [● Saved]                           │
│                                                                           │
│  Workspace                                                                │
│  ROS workspace root:   [.                    ]                           │
│  ESP32 firmware root:  [esp32_firmware/      ]                           │
│                                                                           │
│  Robots                                                              [+] │
│  ┌── virtus-dev (Development Robot) ────────────────────────────────┐   │
│  │  SSH Host:    [192.168.1.100]  User: [pi]                        │   │
│  │  SSH Key:     [virtus-dev-key] [🔑 Set Key]                      │   │
│  │  Telemetry:   Linux: [/dev/ttyUSB2]  Windows: [COM6]            │   │
│  │  Firmware:    Linux: [/dev/ttyUSB0]  Windows: [COM4]            │   │
│  │  [Test SSH Connection ✓]  [Delete]                               │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ROS 2                                                                   │
│  rosbridge port: [9090]  ROS distro: [jazzy]                            │
│                                                                           │
│  [Save]  Changes broadcast to all installed extensions automatically     │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Module 5 — Update Manager

```typescript
// updater/SuiteUpdater.ts
export class SuiteUpdater {

  async checkAll(): Promise<UpdateReport> {
    const updates: ExtensionUpdate[] = [];
    for (const ext of ExtensionInstaller.SUITE_EXTENSIONS) {
      const installed = ExtensionInstaller.getVersion(ext.id);
      const latest    = await this.fetchLatestVersion(ext.id);
      if (installed && latest && semver.gt(latest, installed)) {
        updates.push({ id: ext.id, name: ext.name, installed, latest });
      }
    }
    return { updates, checked_at: Date.now() };
  }

  async updateAll(): Promise<void> {
    for (const ext of await this.checkAll().then(r => r.updates)) {
      await vscode.commands.executeCommand(
        'workbench.extensions.installExtension', ext.id
      );
    }
    vscode.window.showInformationMessage('All Virtusco extensions updated. Reload VS Code to apply.');
  }
}
```

### Updates Page UI

```
┌── Suite Updates ─────────────────────────────────────────────────────────┐
│  Last checked: 5 minutes ago  [Check Now]                               │
│                                                                           │
│  Extension             Installed   Latest    Status                      │
│  Firmware Uploader     v1.2.0      v1.2.0    ✓ Up to date               │
│  Firmware Builder      v0.3.1      v0.4.0    ↑ Update available         │
│  AI Studio             v0.1.2      v0.1.2    ✓ Up to date               │
│  ROS 2 Studio          v0.2.0      v0.2.1    ↑ Update available         │
│  Hardware Dashboard    v0.1.0      v0.1.0    ✓ Up to date               │
│                                                                           │
│  [Update All ↑ (2 updates)]                                              │
│                                                                           │
│  Changelog — Firmware Builder v0.4.0:                                   │
│  - Added BLE peripheral composite node                                   │
│  - Fixed DTS generation for LEDC channel mapping                        │
│  - Windows: fixed path separator in generated overlay files              │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Module 6 — Workspace Bootstrapper

### First-Run Setup Wizard

When the suite is installed in a new VS Code workspace without a `.virtusco/` directory, it auto-opens the setup wizard:

```
Step 1/5: Clone Repository
  [✓] Repository URL: github.com/austin207/Porter-ROS
  [✓] Cloned to current workspace

Step 2/5: System Dependencies
  [✓] git 2.43.0
  [✓] uv 0.5.1
  [⚠] ros2 — not found
      [Install guide for Ubuntu ▼] or [Use WSL2 ▼]

Step 3/5: Python Environments
  Creating .virtus-ai/venv-base... [████████████] Done
  Creating .virtus-ai/venv-vision... [████████████] Done
  Creating .virtus-ai/venv-llm... [░░░░░░░░░░░░] (install when needed)

Step 4/5: Robot Configuration
  Robot name:   [Development Robot]
  SSH host:     [192.168.1.100]
  SSH user:     [pi]
  SSH key:      [Browse... or Generate new]
  [Test Connection: ✓ Connected]

Step 5/5: Review & Finish
  ✓ Extensions installed (8)
  ✓ Python environments created
  ✓ Robot configured
  ✓ .virtusco/suite.config.json created
  [Finish Setup]
```

### WorkspaceBootstrapper.ts

```typescript
// bootstrap/WorkspaceBootstrapper.ts
export class WorkspaceBootstrapper {

  async run(wizard: SetupWizardAnswers): Promise<void> {
    // Step 1: Write shared config
    await SharedConfig.save(wizard.toSuiteConfig());

    // Step 2: Create .virtusco directory structure
    await vscode.workspace.fs.createDirectory(
      vscode.Uri.joinPath(workspaceRoot, '.virtusco')
    );

    // Step 3: Create Python base venv
    await VenvManager.createVenv('base');

    // Step 4: Write VS Code workspace settings
    await this.writeWorkspaceSettings(wizard);

    // Step 5: Broadcast config to all extensions
    EventBus.publish(SUITE_WORKSPACE_INIT, wizard.toSuiteConfig());
  }

  private async writeWorkspaceSettings(wizard: SetupWizardAnswers): Promise<void> {
    // Writes .vscode/settings.json with extension-specific settings
    // pre-populated from wizard answers — so each extension
    // starts configured without manual setup
    const settings = {
      'virtusco.ros2-studio.rosbridgePort': 9090,
      'virtusco.hardware-dashboard.telemetryPort': wizard.telemetryPort,
      'virtusco.firmware-uploader.flashPort': wizard.firmwarePort,
      'virtusco.ai-studio.venvRoot': '.virtus-ai/',
    };
    // Write to .vscode/settings.json
  }
}
```

---

## 10. Extension Dependency Graph

```
virtusco.devtools-suite
    ├── virtusco.firmware-uploader    (independent)
    ├── virtusco.firmware-builder     (independent)
    │       ↕ bus: FW_GENERATED → pcb-reviewer sync check
    ├── virtusco.ai-studio            (independent)
    │       ↕ bus: AI_DEPLOY_COMPLETE → fleet-monitor push
    ├── virtusco.ros2-studio          (reads from hardware-dashboard via bus)
    │       ↕ bus: ROS_NODE_DEAD → suite dashboard alert
    ├── virtusco.hardware-dashboard   (independent)
    │       ↕ bus: HW_MOTOR_ALERT → suite dashboard + ros2-studio
    ├── virtusco.simulation-manager   (uses ros2-studio rosbridge)
    ├── virtusco.pcb-reviewer         (triggered by firmware-builder via bus)
    └── virtusco.fleet-monitor        (triggered by ai-studio via bus)

Shared by all via SharedConfig:
    → SSH credentials, robot inventory, workspace paths
```

---

## 11. Shared Component Library

To ensure visual consistency across all 8 extensions, extract common UI components into a **shared npm package**:

```
@virtusco/ui-components   (private npm package)
├── GpuMonitor            ← Used by AI Studio + suite dashboard
├── SSHStatusBadge        ← Used by Fleet Monitor + Hardware Dashboard
├── TimeSeriesChart       ← Used by Hardware Dashboard + AI Studio
├── LiveLossChart         ← Used by AI Studio
├── AlertBadge            ← Used by Hardware Dashboard + suite dashboard
├── PlatformUtils         ← Used by all 8 extensions
└── VirtusColors          ← Design tokens: colors, spacing, typography
```

This package is not published to npm public registry — it's a local package referenced via `file:../virtusco-ui-components` in each extension's `package.json`.

Shared design tokens:

```typescript
// @virtusco/ui-components/src/VirtusColors.ts
export const Colors = {
  primary:   '#1A73E8',   // Virtusco brand blue
  success:   '#34A853',
  warning:   '#FBBC04',
  error:     '#EA4335',
  surface:   '#1E1E1E',   // VS Code dark theme panel background
  border:    '#3C3C3C',
  text:      '#CCCCCC',
  textDim:   '#858585',
};
```

---

## 12. VS Code Marketplace Strategy

### Publisher Setup

1. Create Marketplace publisher: `virtusco`
2. Generate Personal Access Token (PAT) for CI/CD publishing
3. Package and publish via `vsce publish`

### Extension IDs (Final)

```
virtusco.devtools-suite          ← Install this → gets everything
virtusco.firmware-uploader
virtusco.firmware-builder
virtusco.ai-studio
virtusco.ros2-studio
virtusco.hardware-dashboard
virtusco.simulation-manager
virtusco.pcb-reviewer
virtusco.fleet-monitor
```

### package.json — extensionDependencies

```json
{
  "name": "devtools-suite",
  "publisher": "virtusco",
  "displayName": "Virtusco DevTools Suite",
  "description": "Complete development environment for Virtus autonomous robot — firmware, AI, ROS 2, hardware, and fleet management",
  "version": "1.0.0",
  "extensionDependencies": [
    "virtusco.firmware-uploader"
  ],
  "extensionPack": [
    "virtusco.firmware-builder",
    "virtusco.ai-studio",
    "virtusco.ros2-studio",
    "virtusco.hardware-dashboard",
    "virtusco.simulation-manager",
    "virtusco.pcb-reviewer",
    "virtusco.fleet-monitor"
  ]
}
```

`extensionPack` is VS Code's native mechanism for bundling extensions — when the suite is installed, VS Code automatically installs all listed extensions. The `ExtensionInstaller.ts` module handles cases where pack installation fails.

### Marketplace Listing

```
Virtusco DevTools Suite

The complete VS Code development environment for the Virtus autonomous 
airport luggage porter robot. Install once to get 8 integrated tools:

• Firmware Uploader    — Flash ESP32 devices running Zephyr RTOS
• Firmware Builder     — Visual node-based Zephyr firmware development  
• AI Studio            — Train, export and deploy AI models (vision + LLM)
• ROS 2 Studio         — Topic monitor, FSM viewer, bridge debugger
• Hardware Dashboard   — Live telemetry, motor health, power rails
• Simulation Manager   — Gazebo launch profiles, Nav2 tuning, bag files
• PCB Reviewer         — KiCad viewer, pinout sync, firmware impact analysis
• Fleet Monitor        — Multi-robot fleet management and OTA updates

Built by Virtusco — virtusco.in
```

---

## 13. Windows & Cross-Platform Support

The suite itself is fully cross-platform — it only calls VS Code APIs and the sub-extensions' own cross-platform logic. The dependency checker shows OS-specific install instructions:

```typescript
function getInstallInstructions(dep: DepCheck): string {
  if (Platform.isWindows) {
    if (dep.name === 'ros2') return 'Install WSL2, then: wsl --install Ubuntu, then install ROS 2 Jazzy inside WSL2';
    if (dep.name === 'west') return 'pip install west  (in Windows cmd)';
    return dep.install_url;
  }
  if (dep.name === 'ros2') return 'sudo apt install ros-jazzy-desktop';
  if (dep.name === 'uv')   return 'curl -LsSf https://astral.sh/uv/install.sh | sh';
  return dep.install_url;
}
```

---

## 14. Production Quality Standards

- **Graceful extension absence:** Suite dashboard works even if only 2 of 8 extensions are installed. Missing extension tiles show "Install" — not errors.
- **Config migration:** When suite config schema changes (new version), auto-migrate old `.virtusco/suite.config.json` forward. Never break existing configs.
- **Event bus reliability:** Event bus uses VS Code's built-in command system — if a subscriber extension is not installed, the command simply has no handler. No crash.
- **SecretStorage for SSH keys:** Private keys are NEVER written to disk or config files. Always VS Code `SecretStorage` API.
- **One-time wizard:** Setup wizard runs only once per workspace. If `.virtusco/suite.config.json` exists, skip to dashboard.

---

## 15. Phased Build Plan

### Phase 1 — Suite Scaffold + Installer (Week 1)
- `package.json` with `extensionPack` for all 8
- `ExtensionInstaller.ts` + `DependencyChecker.ts`
- Installer page UI
- **Deliverable:** Install suite → installs all extensions; dependency check shows gaps

### Phase 2 — Shared Config + Config Editor (Week 2)
- `SharedConfig.ts` + `.virtusco/suite.config.json` schema
- `SSHConfigManager.ts` with SecretStorage
- Config editor UI
- **Deliverable:** One-place config shared across all extensions

### Phase 3 — Suite Dashboard (Week 3)
- Extension tile grid with status
- Robot status panel (aggregated from bus)
- Alerts panel (aggregated from bus)
- Extension open links
- **Deliverable:** Full suite dashboard home screen

### Phase 4 — Event Bus (Week 4)
- `EventBus.ts` + all `BusEvents.ts` definitions
- Integration with Hardware Dashboard (`HW_*` events)
- Integration with ROS 2 Studio (`ROS_*` events)
- Integration with Firmware Builder (`FW_*` events)
- **Deliverable:** Cross-extension alerts and data sharing

### Phase 5 — Workspace Bootstrapper + Update Manager (Week 5)
- `WorkspaceBootstrapper.ts` + `SetupWizard.ts`
- Setup wizard UI (5-step flow)
- `SuiteUpdater.ts` + updates page UI
- **Deliverable:** Zero-to-working-dev-environment in one wizard

### Phase 6 — Marketplace + Shared UI Library (Week 6)
- `@virtusco/ui-components` package extracted and published privately
- Marketplace publisher account + `vsce` CI/CD pipeline (GitHub Actions)
- Marketplace listing with screenshots
- **Deliverable:** Published to VS Code Marketplace

---

## 16. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Extension pack mechanism | `extensionPack` in `package.json` | Native VS Code mechanism |
| Cross-extension comms | VS Code commands as event bus | Only supported IPC mechanism |
| Shared config storage | `.virtusco/suite.config.json` (workspace) | Git-committed, shared across team |
| SSH key storage | VS Code `SecretStorage` API | Encrypted, never on disk |
| Shared UI components | Private npm package (`@virtusco/ui`) | One source of truth for design |
| Marketplace publishing | `vsce` + GitHub Actions CI | Standard extension publishing |
| Versioning | Semantic versioning (semver) | Standard, `semver` npm package |

---

## 17. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| VS Code `extensionPack` install fails silently | Medium | Medium | `ExtensionInstaller` re-checks after pack install; installs missing manually |
| Event bus command name collision with other extensions | Low | High | All bus events prefixed `virtusco.bus.*` — unique namespace |
| Shared config schema breaks on update | Medium | High | Config versioned with `schema_version` field; auto-migration on load |
| SSH private key accidentally written to config file | Low | Critical | Code review gate: `SecretStorage` usage enforced; config file gitignored for keys |
| extensionPack pulls wrong version of sub-extension | Low | Medium | Pin sub-extension versions in pack; test matrix on publish |
| Marketplace publisher account access lost | Low | High | Two-team-member access to publisher account; PAT stored in GitHub Secrets |
| @virtusco/ui-components diverges across extensions | Medium | Medium | Single repo (monorepo) for all extensions; shared package always in sync |
