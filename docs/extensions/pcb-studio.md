# Virtus PCB Studio

A KiCad schematic viewer and visual PCB builder integrated into VS Code. Designed for Porter's electronics design workflow — view schematics, lay out PCBs, check design rules, and manage the bill of materials.

## Features

### KiCad Schematic Viewer

Open and view KiCad `.kicad_sch` files directly in VS Code. Navigate hierarchical sheets, search for components, and inspect net connections.

### Visual PCB Builder

Drag-and-drop PCB layout tool with a 14-component library tailored to Porter's hardware:

| Category | Components |
|----------|-----------|
| **MCU** | ESP32-WROOM, ESP32-S3-WROOM |
| **Power** | LM7805, AMS1117-3.3, buck converter module |
| **Motor** | BTS7960 H-bridge, screw terminal blocks |
| **Sensors** | VL53L0x breakout, HC-SR04 header, RCWL-0516 |
| **Connectors** | USB-C, JST-XH, pin headers (2.54 mm) |
| **Passives** | Resistor, capacitor, LED |

### Pinout Sync Checker

Validates that the pin assignments in the PCB schematic match the Zephyr DTS overlay and firmware source code. Reports mismatches between hardware design and software configuration.

```
[PASS] GPIO_25 -> BTS7960_RPWM (motor_controller overlay.dts line 12)
[PASS] GPIO_26 -> BTS7960_LPWM (motor_controller overlay.dts line 13)
[FAIL] GPIO_18 -> VL53L0x_XSHUT (not found in sensor_fusion overlay.dts)
```

### Bill of Materials (BOM)

Auto-generated BOM with:

- Component values and footprints
- LCSC part numbers with direct purchase links
- Quantity and unit cost
- Total board cost estimate

### Git-Based Schematic Diff

Visual diff for KiCad schematic changes across Git commits. Highlights added, removed, and modified components and connections.

### Firmware Impact Analyzer

When a schematic change is detected, the analyzer identifies which firmware source files and DTS overlays are affected and need updating.

!!! note "Revisit items"
    Builder drag-drop rendering, KiCad export, and sync checker testing are planned for a future update.
