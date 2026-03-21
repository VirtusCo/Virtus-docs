# VS Code Extension Suite

VirtusCo maintains a suite of 8 VS Code extensions that provide a complete development environment for the Porter robot. Each extension is a standalone `.vsix` package managed by the master DevTools Suite.

## Extension Overview

| Extension | Purpose | Stack |
|-----------|---------|-------|
| [Porter DevTools](../extensions/firmware-builder.md) | Firmware flashing, RPi deployment | TypeScript |
| [Firmware Builder](firmware-builder.md) | Visual node-based Zephyr firmware dev | TypeScript, React Flow |
| [AI Studio](ai-studio.md) | MLOps workbench: train, benchmark, export, deploy | TypeScript, React |
| [ROS 2 Studio](ros2-studio.md) | Node graph, FSM viewer, launch builder | TypeScript, React |
| [Hardware Dashboard](hardware-dashboard.md) | Live hardware telemetry and power monitoring | TypeScript, React |
| [Simulation Manager](simulation-manager.md) | Gazebo launch profiles, Nav2 tuning | TypeScript, React |
| [PCB Studio](pcb-studio.md) | KiCad viewer, PCB layout, BOM, DRC | TypeScript, React |
| **DevTools Suite** | Master meta-extension managing all 7 above | TypeScript |

## Installation

All extensions are distributed as `.vsix` files via GitHub Releases. The DevTools Suite can install and manage the sub-extensions automatically.

```bash
# Install the master suite
code --install-extension virtusco-devtools-suite-0.5.0.vsix

# Or install individually
code --install-extension virtus-firmware-builder-0.5.0.vsix
```

## Architecture

All extensions share:

- **TypeScript strict mode** with no `any` types
- **React** webview panels with VS Code CSS variable theming
- **esbuild** bundling with shared configuration
- **`acquireVsCodeApi` singleton** pattern for webview-to-extension communication
- **PlatformUtils** for OS detection (Windows, Linux, macOS, WSL)

## Build from Source

The extensions live in an npm workspaces monorepo:

```bash
cd virtusco-extensions
npm install
npm run compile     # Build all 8 extensions
npm run lint        # ESLint across all
npm run package:all # Package each as .vsix
```

!!! tip "Development"
    Press F5 in VS Code with any extension folder open to launch an Extension Development Host for debugging.
