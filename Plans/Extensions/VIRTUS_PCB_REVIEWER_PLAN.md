# Virtus PCB Schematic Reviewer — VS Code Extension
### Detailed Development Plan
**Project:** KiCad schematic viewer, pinout sync checker, BOM viewer, and hardware-firmware diff tool for Virtusco  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Extension Architecture](#2-extension-architecture)
3. [Module 1 — KiCad Schematic Viewer](#3-module-1--kicad-schematic-viewer)
4. [Module 2 — Pinout Sync Checker](#4-module-2--pinout-sync-checker)
5. [Module 3 — BOM Viewer](#5-module-3--bom-viewer)
6. [Module 4 — Schematic Change Diff](#6-module-4--schematic-change-diff)
7. [Module 5 — Firmware Impact Analyzer](#7-module-5--firmware-impact-analyzer)
8. [Message Protocol](#8-message-protocol)
9. [KiCad File Format Notes](#9-kicad-file-format-notes)
10. [Windows & Cross-Platform Support](#10-windows--cross-platform-support)
11. [Production Quality Standards](#11-production-quality-standards)
12. [Phased Build Plan](#12-phased-build-plan)
13. [Tech Stack Summary](#13-tech-stack-summary)
14. [Risk Register](#14-risk-register)

---

## 1. Overview & Vision

### The Core Problem

Danush designs PCBs and schematics in KiCad. Antony writes Zephyr firmware that maps to specific GPIO pins, I2C addresses, and UART instances. **There is currently no automated bridge between these two worlds.** When Danush renames a net, reassigns a pin, or changes an I2C address, Antony finds out when something stops working — not before.

Specific incidents this tool prevents:
- Danush reassigns `MOTOR_LEFT_RPWM` from GPIO18 to GPIO21 → Zephyr overlay still says GPIO18 → motors behave erratically
- Danush adds a pull-up to the I2C bus → VL53L0X address changes → ToF sensor returns garbage
- Danush swaps UART0 and UART1 on the PCB → bridge communicates with wrong ESP32

### What Virtus PCB Schematic Reviewer Provides

```
KiCad Viewer        → Renders .kicad_sch schematic directly in VS Code (read-only)
Pinout Sync Checker → Compares DTS aliases vs schematic netlist → flags mismatches
BOM Viewer          → Bill of Materials from KiCad, linked to LCSC/Robu.in
Change Diff         → Human-readable diff when Danush pushes schematic changes
Firmware Impact     → When schematic changes, highlights which firmware files need updating
```

### Who Uses It

**Primary:** Danush — to review schematic before committing and catch obvious pin conflicts  
**Primary:** Antony — to instantly know when Danush's hardware changes affect firmware  
**Secondary:** Azeem — to verify mechanical mounting points match PCB footprint locations

---

## 2. Extension Architecture

```
virtus-pcb-reviewer/
├── package.json
│     activationEvents: ["onLanguage:kicad_sch", "workspaceContains:**/*.kicad_sch"]
├── tsconfig.json
├── esbuild.config.js
│
├── src/
│   ├── extension.ts                      ← Activates on .kicad_sch files
│   ├── panel/
│   │   └── SchematicPanel.ts             ← WebviewPanel
│   ├── kicad/
│   │   ├── SchematicParser.ts            ← Parses .kicad_sch (S-expression format)
│   │   ├── NetlistExtractor.ts           ← Extracts net→pin mappings from schematic
│   │   ├── BOMExtractor.ts               ← Extracts component list with references
│   │   └── SchematicRenderer.ts          ← Converts parsed schematic to SVG
│   ├── sync/
│   │   ├── DTSReader.ts                  ← Reads Zephyr .overlay files
│   │   ├── PinoutMapper.ts               ← Maps DTS aliases to GPIO numbers
│   │   └── SyncChecker.ts                ← Compares netlist vs DTS, reports mismatches
│   ├── diff/
│   │   ├── SchematicDiffer.ts            ← Computes readable diff between two .kicad_sch
│   │   └── DiffFormatter.ts             ← Formats diff as human-readable change list
│   ├── impact/
│   │   └── FirmwareImpactAnalyzer.ts     ← Maps net changes to firmware files
│   └── platform/
│       └── PlatformUtils.ts
│
└── webview-ui/
    └── src/
        ├── pages/
        │   ├── SchematicPage.tsx         ← SVG schematic viewer with pan/zoom
        │   ├── SyncPage.tsx              ← Pinout sync results table
        │   ├── BOMPage.tsx               ← BOM table with part links
        │   ├── DiffPage.tsx              ← Schematic change diff view
        │   └── ImpactPage.tsx            ← Firmware impact analysis
        └── components/
            ├── SchematicSVG.tsx          ← Pan/zoom SVG canvas
            ├── SyncRow.tsx               ← OK/mismatch/missing row
            ├── BOMRow.tsx                ← Component row with LCSC link
            ├── DiffLine.tsx              ← Added/removed/changed diff line
            └── ImpactFile.tsx            ← Firmware file with affected lines
```

---

## 3. Module 1 — KiCad Schematic Viewer

### KiCad S-Expression Format

KiCad `.kicad_sch` files are S-expression (Lisp-style) text format since KiCad 6:

```lisp
(kicad_sch (version 20231120) (generator eeschema)
  (lib_symbols ...)
  (wire (pts (xy 100 100) (xy 200 100)) (stroke (width 0) (type default)))
  (symbol (lib_id "Device:R") (at 150 150 0) (unit 1)
    (property "Reference" "R1" (at 150 140 0))
    (property "Value" "10k" (at 150 160 0))
    (pin "1" (at 150 145 270))
    (pin "2" (at 150 155 90))
  )
  (net_inspector ...)
)
```

### Parser

```typescript
// kicad/SchematicParser.ts
export function parseKicadSch(content: string): KicadSchematic {
  // S-expression parser — handles nested lists
  const sexp = parseSExpr(content);
  return {
    version:    sexpGet(sexp, 'version'),
    wires:      sexpGetAll(sexp, 'wire').map(parseWire),
    symbols:    sexpGetAll(sexp, 'symbol').map(parseSymbol),
    labels:     sexpGetAll(sexp, 'net_label').map(parseLabel),
    junctions:  sexpGetAll(sexp, 'junction').map(parseJunction),
    buses:      sexpGetAll(sexp, 'bus').map(parseBus),
  };
}
```

### SVG Renderer

Converts the parsed schematic to SVG using a coordinate transform (KiCad units → SVG pixels). Renders:
- Wires as `<line>` elements
- Symbols as `<g>` groups with `<rect>`, `<line>`, `<text>` for pins and labels
- Net labels as `<text>` with directional arrows
- Junctions as `<circle>`

The SVG is rendered in the webview with pan/zoom via `transform` CSS on a wrapper div (no heavy library needed).

### Schematic Viewer UI

```
┌── Schematic — power_management.kicad_sch ────────────────────────────────┐
│  [Fit] [100%] [+] [-]  [Search net...]  [Highlight: BTS7960_LEFT ▼]     │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                                                                     │ │
│  │   [SVG schematic rendered here]                                     │ │
│  │   Pan: click+drag  Zoom: scroll                                     │ │
│  │   Click net label to highlight all connected nets                   │ │
│  │                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│  Components: 47  Nets: 62  Sheets: 1                                     │
└───────────────────────────────────────────────────────────────────────────┘
```

**Net highlight:** Click a net label → all wires on that net highlighted in orange. Critical for tracing power rails and signal paths.

---

## 4. Module 2 — Pinout Sync Checker

### How It Works

1. `NetlistExtractor` reads the KiCad schematic and extracts `{net_name → [component_pin]}` mappings
2. `DTSReader` reads `boards/esp32.overlay` and extracts `{alias → gpio_number}` mappings
3. `PinoutMapper` applies Virtusco's mapping rules (net names → expected DTS aliases)
4. `SyncChecker` compares the two — reports OK, mismatch, or missing

### Virtusco Pin Mapping Rules

```typescript
// sync/PinoutMapper.ts
// Maps KiCad net names → expected Zephyr DTS alias names
export const VIRTUS_NET_TO_DTS: Record<string, string> = {
  'MOTOR_LEFT_RPWM':    'motor-left-rpwm',
  'MOTOR_LEFT_LPWM':    'motor-left-lpwm',
  'MOTOR_LEFT_EN':      'motor-left-en',
  'MOTOR_RIGHT_RPWM':   'motor-right-rpwm',
  'MOTOR_RIGHT_LPWM':   'motor-right-lpwm',
  'MOTOR_RIGHT_EN':     'motor-right-en',
  'TOF_SDA':            'i2c-sda',
  'TOF_SCL':            'i2c-scl',
  'ULTRASONIC_TRIG':    'ultrasonic-trig',
  'ULTRASONIC_ECHO':    'ultrasonic-echo',
  'MICROWAVE_OUT':      'microwave-out',
  'ESP1_UART_TX':       'uart-esp1-tx',
  'ESP1_UART_RX':       'uart-esp1-rx',
  'RELAY_1':            'relay-1',
  'RELAY_2':            'relay-2',
};
```

### Sync Checker UI

```
┌── Pinout Sync Check ─────────────────────────────────────────────────────┐
│  Schematic: hardware/power_mgmt.kicad_sch                                │
│  DTS:       firmware/boards/esp32.overlay                                │
│  [Run Check]  Last checked: 2 minutes ago                                │
│                                                                           │
│  Status  Net Name           DTS Alias           Schematic  DTS           │
│  ──────────────────────────────────────────────────────────────────────── │
│  ✓ OK    MOTOR_LEFT_RPWM   motor-left-rpwm      GPIO18     GPIO18        │
│  ✓ OK    MOTOR_LEFT_LPWM   motor-left-lpwm      GPIO19     GPIO19        │
│  ✗ MISMATCH MOTOR_RIGHT_EN motor-right-en       GPIO21  ≠  GPIO5         │
│            Schematic: GPIO21    DTS: GPIO5                               │
│            [View in Schematic]  [View in DTS]  [Fix DTS]                 │
│  ⚠ MISSING TOF_SDA         i2c-sda              GPIO22     NOT IN DTS   │
│            This net exists in schematic but has no DTS alias             │
│            [Add to DTS]                                                  │
│  ✓ OK    RELAY_1            relay-1              GPIO4      GPIO4         │
└───────────────────────────────────────────────────────────────────────────┘
  Summary: 12 OK  1 Mismatch  1 Missing  [Export Report]
```

**Fix DTS button:** Opens `esp32.overlay` at the mismatched alias line in VS Code editor. Does not auto-fix — Antony must confirm the correct GPIO.

**Add to DTS button:** Generates a DTS alias snippet for the missing net and shows it in a quick-pick dialog to insert.

**Auto-check on commit:** The extension registers a Git pre-commit hook (optional, user opt-in) that runs the sync check and blocks commit if mismatches exist.

---

## 5. Module 3 — BOM Viewer

### BOM Extraction

```typescript
// kicad/BOMExtractor.ts
interface BOMEntry {
  reference:    string;    // "R1", "U3"
  value:        string;    // "10k", "BTS7960"
  footprint:    string;    // "Package_TO_SOT_THT:TO-220-3_Horizontal"
  lcsc_part:    string;    // "C2890"  (from schematic property if set)
  quantity:     number;
  description:  string;
}

export function extractBOM(schematic: KicadSchematic): BOMEntry[] {
  // Group by value + footprint → merge references, sum quantities
}
```

### BOM UI

```
┌── Bill of Materials ─────────────────────────────────────────────────────┐
│  power_management.kicad_sch  [Export CSV]  [Export LCSC Cart]           │
│                                                                           │
│  Ref        Value          Qty  Footprint                  LCSC          │
│  ──────────────────────────────────────────────────────────────────────  │
│  U1, U2     BTS7960        2    TO-220-5                   [C2890 →]     │
│  U3         LM7812CT       1    TO-220-3_Horizontal        [C181241 →]   │
│  U4         LM7805CT       1    TO-220-3_Horizontal        [C55473 →]    │
│  U5         AMS1117-3.3    1    SOT-223                    [C6186 →]     │
│  U6         Arduino Nano   1    Arduino_Nano               [—]           │
│  R1–R10     10k            10   R_0805                     [C17414 →]    │
│  C1–C8      100nF          8    C_0805                     [C49678 →]    │
│  K1–K4      G5LE-1-E       4    Relay_SPDT                [—]           │
│                                                                           │
│  Total components: 47  Unique: 18  LCSC parts: 14/18                    │
└───────────────────────────────────────────────────────────────────────────┘
```

**LCSC links:** Each part with an LCSC code opens `https://www.lcsc.com/product-detail/{code}.html` in browser. **Export LCSC Cart** generates a CSV in LCSC's cart import format.

**Robu.in fallback:** For parts not on LCSC (common modules like Arduino Nano), link to Robu.in search.

---

## 6. Module 4 — Schematic Change Diff

### When Is This Used?

Danush commits an updated `.kicad_sch` to Git. The extension detects the change (via `git diff`) and renders a **human-readable change list** — not a raw S-expression diff (which is unreadable).

### Differ Logic

```typescript
// diff/SchematicDiffer.ts
export function diffSchematics(oldSch: KicadSchematic, newSch: KicadSchematic): SchematicDiff {
  return {
    nets_added:      findAddedNets(oldSch, newSch),
    nets_removed:    findRemovedNets(oldSch, newSch),
    nets_renamed:    findRenamedNets(oldSch, newSch),
    pins_moved:      findMovedPins(oldSch, newSch),
    components_added:   findAddedComponents(oldSch, newSch),
    components_removed: findRemovedComponents(oldSch, newSch),
    values_changed:  findChangedValues(oldSch, newSch),
  };
}
```

### Diff UI

```
┌── Schematic Changes — power_management.kicad_sch ────────────────────────┐
│  Comparing: HEAD~1 → HEAD  (committed 3 minutes ago by Danush)           │
│                                                                           │
│  ⚠ NET RENAMED (affects firmware!)                                       │
│    MOTOR_RIGHT_EN  →  MOTOR_R_ENABLE                                     │
│    Impact: 2 firmware files reference old net name                       │
│    [View Firmware Impact]                                                 │
│                                                                           │
│  + PIN REASSIGNED                                                         │
│    MOTOR_RIGHT_RPWM: GPIO19 → GPIO21                                     │
│    Impact: DTS alias motor-right-rpwm needs update                       │
│    [View in DTS]                                                          │
│                                                                           │
│  + COMPONENT ADDED                                                        │
│    D1: 1N4007 (freewheeling diode across relay K3)                       │
│    No firmware impact                                                     │
│                                                                           │
│  ✓ No firmware impact for 3 other changes                                │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Module 5 — Firmware Impact Analyzer

### Concept

When a schematic change is detected, the analyzer scans all firmware files for references to the changed net names, GPIO numbers, and I2C addresses, and reports which lines need updating.

```typescript
// impact/FirmwareImpactAnalyzer.ts
export class FirmwareImpactAnalyzer {

  async analyze(diff: SchematicDiff): Promise<FirmwareImpact[]> {
    const impacts: FirmwareImpact[] = [];
    const firmwareDirs = ['esp32_firmware/', 'porter_esp32_bridge/'];

    for (const change of diff.pins_moved) {
      // Search for old GPIO number in .c, .h, .overlay, .conf files
      const matches = await this.grepFirmware(firmwareDirs, `GPIO${change.old_pin}`);
      impacts.push({
        change,
        files: matches,
        severity: 'HIGH',  // Pin reassignment always HIGH
        description: `GPIO${change.old_pin} reassigned to GPIO${change.new_pin} — update DTS alias and verify PWM channel`
      });
    }

    for (const rename of diff.nets_renamed) {
      const matches = await this.grepFirmware(firmwareDirs, rename.old_name);
      if (matches.length > 0) {
        impacts.push({ change: rename, files: matches, severity: 'MEDIUM' });
      }
    }

    return impacts;
  }

  private async grepFirmware(dirs: string[], pattern: string): Promise<FileMatch[]> {
    // Uses VS Code's built-in TextSearch API — no grep dependency
    return vscode.workspace.findTextInFiles(
      { pattern, isRegExp: false },
      { include: new vscode.RelativePattern(workspaceRoot, `{${dirs.join(',')}}**/*.{c,h,overlay,conf}`) }
    );
  }
}
```

### Impact UI

```
┌── Firmware Impact — MOTOR_RIGHT_RPWM: GPIO19 → GPIO21 ──────────────────┐
│  Severity: HIGH  (Pin reassignment affects motor PWM output)             │
│                                                                           │
│  Files requiring update:                                                  │
│                                                                           │
│  esp32_firmware/motor_controller/boards/esp32.overlay                    │
│  Line 12:  motor-right-rpwm = &ledc0_ch1;  /* GPIO19 */                 │
│            Update to: ledc0_ch3  (GPIO21 → LEDC channel 3 on ESP32)     │
│            [Open File]                                                    │
│                                                                           │
│  esp32_firmware/motor_controller/src/bts7960.c                           │
│  Line 34:  #define MOTOR_RIGHT_RPWM_PIN 19                               │
│            Update to: 21                                                  │
│            [Open File]                                                    │
│                                                                           │
│  [Create GitHub Issue for Antony]                                        │
└───────────────────────────────────────────────────────────────────────────┘
```

**Create GitHub Issue:** Generates a pre-filled GitHub issue with the schematic change, affected files, and suggested fixes. Opens the browser to the Virtusco GitHub repo's new issue page.

---

## 8. Message Protocol

```typescript
type WebviewMessage =
  | { type: 'loadSchematic';   payload: { path: string } }
  | { type: 'runSyncCheck';    payload: void }
  | { type: 'openInEditor';    payload: { file: string; line: number } }
  | { type: 'exportBOM';       payload: { format: 'csv' | 'lcsc' } }
  | { type: 'openLCSC';        payload: { partCode: string } }
  | { type: 'runDiff';         payload: { oldRef: string; newRef: string } }
  | { type: 'createGHIssue';   payload: { impact: FirmwareImpact } }

type HostMessage =
  | { type: 'schematicParsed'; payload: { svg: string; stats: SchematicStats } }
  | { type: 'syncResults';     payload: SyncResult[] }
  | { type: 'bomData';         payload: BOMEntry[] }
  | { type: 'diffResult';      payload: SchematicDiff }
  | { type: 'impactResult';    payload: FirmwareImpact[] }
  | { type: 'schematicFiles';  payload: string[] }
```

---

## 9. KiCad File Format Notes

- **KiCad 6+:** S-expression format (`.kicad_sch`) — what the parser targets
- **KiCad 5 and earlier:** Legacy format (`.sch`) — different parser needed; show "legacy format not supported" if detected
- **Hierarchical sheets:** A master `.kicad_sch` may reference sub-sheets. The parser must follow `sheet` references and merge netlists across sheets.
- **Library symbols:** Component shapes defined in `lib_symbols` section of the schematic file — no need to load external `.kicad_sym` libraries for viewing

---

## 10. Windows & Cross-Platform Support

KiCad files are plain text — fully cross-platform. The parser runs in Node.js (extension host) with no OS-specific dependencies. The SVG renderer runs in the webview. Git diff calls use `child_process.exec('git diff ...')` which works on Windows natively (Git for Windows). No WSL2 needed for this extension.

---

## 11. Production Quality Standards

- **Large schematic performance:** Schematics with 500+ components parsed in < 2s. SVG rendered progressively — wires first, then symbols.
- **Read-only safety:** The viewer never modifies `.kicad_sch` files. All suggested changes are shown in the UI and must be manually applied by the developer.
- **Git dependency:** Diff and impact features require Git. Extension gracefully degrades to "diff not available — not a Git repository" if Git not found.
- **Schema version detection:** Parser checks `version` field in `.kicad_sch`. Warns if version newer than tested (may have new S-expression nodes).

---

## 12. Phased Build Plan

### Phase 1 — KiCad Parser + Schematic Viewer (Weeks 1–2)
- S-expression parser
- SVG renderer for wires, symbols, labels
- Pan/zoom viewer
- Net highlight on click
- **Deliverable:** `.kicad_sch` files open in VS Code panel

### Phase 2 — Pinout Sync Checker (Week 3)
- `NetlistExtractor` + `DTSReader`
- Virtusco net→DTS mapping table
- Sync results table with OK/Mismatch/Missing
- "View in editor" links
- **Deliverable:** Hardware-firmware pin sync check

### Phase 3 — BOM Viewer (Week 4)
- BOM extractor with grouping
- LCSC link integration
- CSV + LCSC cart export
- **Deliverable:** BOM view with part links

### Phase 4 — Schematic Diff (Week 5)
- `SchematicDiffer` comparing two schematic versions
- Human-readable change list
- Git integration (diff against HEAD~1 or any ref)
- **Deliverable:** Readable schematic change diff

### Phase 5 — Firmware Impact Analyzer + Polish (Week 6)
- `FirmwareImpactAnalyzer` using VS Code TextSearch API
- Impact severity levels
- GitHub issue creation
- **Deliverable:** Production-ready extension

---

## 13. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| S-expression parser | Custom TypeScript | KiCad format is simple enough; no deps |
| SVG rendering | Custom SVG generation | No heavy canvas lib needed for schematics |
| Pan/zoom | CSS `transform` + mouse events | Simple, performant |
| BOM grouping | Pure TypeScript | No deps |
| Git diff | `child_process.exec('git diff')` | Standard Git CLI |
| Firmware search | `vscode.workspace.findTextInFiles` | Native VS Code API, no grep |
| LCSC links | Browser `openExternal` | Opens in user's browser |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| KiCad 7/8 S-expression changes break parser | Medium | Medium | Version check at load; warn if unsupported version |
| Hierarchical sheets not followed → incomplete netlist | High | High | Implement sheet reference following in Phase 1; flag if sub-sheets found but not loaded |
| Net naming conventions differ between Danush's KiCad and Zephyr DTS | High | Medium | Mapping table editable by user; not hardcoded only |
| Large schematics (500+ parts) slow SVG rendering | Medium | Medium | Virtualize SVG — only render visible viewport; use requestAnimationFrame |
| Git not installed on Windows (rare but possible) | Low | Low | Detect; disable diff features; show install guide |
| LCSC part codes not set in schematic properties | High | Low | Show "—" for missing codes; provide instructions for adding LCSC codes in KiCad |
