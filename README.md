# ESP32-S3 Mini — Custom 4-Layer High-Speed PCB Design

A custom-designed 4-layer ESP32-S3 development board with USB-C interface, impedance-controlled differential routing, and optimized power delivery. Designed in **Altium Designer** following professional PCB design practices for signal integrity, thermal management, and manufacturability.

---

## Board Overview

```
                    ┌─────────────────────────────────┐
                    │        ESP32-S3 Mini Board       │
                    │          (4-Layer PCB)            │
                    │                                   │
  USB-C ═══════════►│  FTDI Bridge ──► ESP32-S3-MINI   │◄══════ GPIO Headers
  (5V Power +       │       │              │            │        (Breadboard
   USB Data)        │   Auto-Reset     Wi-Fi/BLE        │         compatible)
                    │   Circuit        Antenna          │
                    │       │              │            │
                    │   5V ──► LDO ──► 3.3V Rail       │
                    │         TI          │             │
                    │        LDO    Decoupling Array    │
                    │               (10µF+4.7µF+100nF) │
                    └─────────────────────────────────┘
```

---

## Key Specifications

| Parameter | Value |
|---|---|
| **MCU** | ESP32-S3-MINI (dual-core Xtensa LX7, Wi-Fi + BLE 5.0) |
| **Layers** | 4 (SIG–GND–GND–SIG) |
| **USB** | USB-C (UFP sink, 5.1kΩ CC pull-downs) |
| **USB Bridge** | FTDI USB-to-UART |
| **Impedance Control** | 50Ω single-ended, 90Ω differential (USB D+/D−) |
| **Power** | 5V USB → TI LDO → 3.3V regulated |
| **Stackup** | JLC3313 (JLCPCB 4-layer standard) |
| **DRC** | 6 mil clearance, 12 mil (0.3mm) min hole |
| **Assembly** | JLCPCB SMT assembly ready (LCSC part numbers in BOM) |

---

## PCB Stackup & Impedance Control

A 4-layer stackup is critical for the ESP32's Wi-Fi performance and USB high-speed signal integrity.

```
  ┌──────────────────────────────────────────────┐
  │  Layer 1 (Top)     — Signal / Power           │  ← USB traces, MCU, components
  ├──────────────────────────────────────────────┤
  │  Layer 2 (Inner 1) — Ground Plane             │  ← Reference for top layer signals
  ├──────────────────────────────────────────────┤
  │  Layer 3 (Inner 2) — Ground Plane             │  ← Reference for bottom layer signals
  ├──────────────────────────────────────────────┤
  │  Layer 4 (Bottom)  — Signal / Power           │  ← GPIO routing, silkscreen labels
  └──────────────────────────────────────────────┘
```

### Why Dual Ground Planes?

Every high-speed signal needs a **reference plane** directly adjacent to it for the return current. With ground on both inner layers:
- Top layer signals reference Layer 2 (GND)
- Bottom layer signals reference Layer 3 (GND)
- Wi-Fi antenna performance depends on clean ground beneath the signal path
- EMI radiation is minimized — signals are shielded between ground planes

### Impedance Targets

| Signal Type | Target | Track Width | Spacing | Method |
|---|---|---|---|---|
| **Standard Digital IO** | 50Ω single-ended | 6 mil | — | Microstrip over GND plane |
| **USB D+ / D−** | 90Ω differential | 6 mil | 8 mil | Differential pair routing |

Track widths calculated using impedance calculator based on JLC3313 dielectric thickness.

---

## Schematic Design

### USB-C Configuration (UFP Sink)

The USB-C connector is configured as an **Upstream Facing Port (UFP)** — the board acts as a device, not a host.

```
         USB-C Connector
         ┌──────────┐
  VBUS ──┤          ├── 5V Power
  D+   ──┤          ├── To FTDI (90Ω differential pair)
  D−   ──┤          ├── To FTDI (90Ω differential pair)
  CC1  ──┤          ├── 5.1kΩ → GND  (identifies as UFP sink)
  CC2  ──┤          ├── 5.1kΩ → GND  (identifies as UFP sink)
  GND  ──┤          ├── Ground
         └──────────┘
```

The **5.1kΩ pull-down resistors** on CC1 and CC2 tell the USB-C source (laptop/charger) that this is a sink device requesting 5V power.

### Auto-Reset Circuit (Hands-Free Programming)

A dual-transistor (NPN) logic gate automatically toggles the ESP32's boot pins using DTR/RTS signals from the FTDI bridge.

```
  FTDI DTR ──┬── NPN Q1 ──► CHIP_PU (Reset)
             │
  FTDI RTS ──┴── NPN Q2 ──► GPIO0 (Boot Mode)
```

**How it works:**
1. IDE asserts DTR + RTS in a specific sequence
2. Q1 pulls CHIP_PU low → ESP32 resets
3. Q2 holds GPIO0 low → ESP32 enters bootloader
4. Firmware uploads via UART
5. DTR/RTS release → ESP32 boots into new firmware

**Result:** One-click flash from Arduino IDE / ESP-IDF — no physical button presses needed.

### Power Distribution

```
  USB 5V ──► TI LDO Regulator ──► 3.3V Rail
                    │
                    ├── 10µF  (bulk energy storage — handles load transients)
                    ├── 4.7µF (medium frequency filtering)
                    └── 100nF (high-frequency decoupling — placed closest to MCU pins)
```

Multiple capacitor values in parallel provide **broadband decoupling**:
- **10µF:** Stores energy for sudden current demands (Wi-Fi TX bursts)
- **4.7µF:** Filters mid-frequency ripple from the LDO
- **100nF:** Filters high-frequency switching noise — must be within 3mm of each VDD pin

---

## PCB Layout Techniques

### Differential Pair Routing (USB)

USB D+ and D− are routed as a **90Ω differential pair** with length matching:

```
  FTDI ════╤════╤════ USB-C
           D+   D−
       6 mil  8 mil  6 mil
       width  gap    width
```

- Tracks routed together with constant spacing (8 mil)
- No reference plane breaks under the differential pair
- Length matched to within ±5 mil to prevent skew

### Thermal Management (LDO)

The 3.3V regulator dissipates heat when converting 5V → 3.3V. A thermal relief structure prevents overheating:

```
  ┌────────────────────────┐
  │   LDO Thermal Pad      │  ← Top copper pour (exposed pad)
  │   ┌──┐ ┌──┐ ┌──┐      │
  │   │▓▓│ │▓▓│ │▓▓│      │  ← Thermal vias (matrix pattern)
  │   └──┘ └──┘ └──┘      │     connecting to inner GND planes
  │   ┌──┐ ┌──┐ ┌──┐      │
  │   │▓▓│ │▓▓│ │▓▓│      │  ← Heat conducts through vias
  │   └──┘ └──┘ └──┘      │     to 2 internal copper planes
  └────────────────────────┘     acting as a heat sink
```

- Top layer polygon pour under LDO thermal pad
- Matrix of thermal vias connecting to Layer 2 and Layer 3 ground planes
- Internal ground planes act as heat spreaders (large copper area)

### Ground Stitching Vias

When a signal via transitions from top to bottom layer, the return current must also transition between ground planes:

```
  Top Layer:    ──── Signal ────[VIA]
  Layer 2 GND:  ══════════════[STITCH VIA]══  ← Return current path
  Layer 3 GND:  ══════════════[STITCH VIA]══  ← provided by stitching via
  Bottom Layer: ──── Signal ────[VIA]
```

**Without stitching vias:** Return current takes a long detour around the board → creates a loop antenna → radiates EMI.

**With stitching vias:** Return current has a low-impedance path directly adjacent to the signal via → minimal loop area → minimal radiation.

### Antenna Keep-Out

The ESP32-S3-MINI has an integrated antenna. The PCB layout includes:
- **No copper** (any layer) under or near the antenna area
- **No ground plane** within the antenna keep-out zone
- Board edge aligned with module edge for unobstructed radiation pattern

---

## Design for Manufacturing (DFM)

| Feature | Implementation | Purpose |
|---|---|---|
| **Pin Labels** | Mirrored silkscreen on bottom layer | Identify pins while board is on breadboard |
| **LCSC Part Numbers** | Every component has LCSC/DigiKey PN in BOM | One-click JLCPCB assembly — no manual sourcing |
| **Design Variants** | Components marked "Not Fitted" (NF) using Altium variants | BOM reflects exact components for each prototype |
| **DRC Clearance** | 6 mil (0.15mm) | Within JLCPCB standard capability |
| **Min Hole Size** | 12 mil (0.3mm) | Standard drill — no extra cost |
| **3D DRC** | STEP models on all components | Mechanical collision check — connectors align with board edge |
| **Loop Removal** | Altium automatic loop removal enabled | Prevents redundant copper paths from acting as antennas |

---

## Design Techniques Summary

| Feature | Technique | Purpose |
|---|---|---|
| **USB Signals** | Differential pair routing (6 mil / 8 mil) | Maintain 90Ω impedance for high-speed data |
| **Heat Dissipation** | Via stitching matrix + polygon pours | Prevent LDO regulator from overheating |
| **Signal Integrity** | 4-layer dual ground planes | Clean return paths, reduced EMI for Wi-Fi |
| **Programming** | Dual-transistor auto-reset circuit | Enable hands-free firmware flashing |
| **Usability** | Bottom-layer mirrored silkscreen | Pin identification during breadboard prototyping |
| **Impedance** | Calculated from JLC3313 stackup | 50Ω single-ended, 90Ω differential — verified |
| **Assembly** | Full BOM with LCSC part numbers | Drop-in JLCPCB SMT assembly |

---

## Component Library

All components built with professional library practices:

| Method | Usage |
|---|---|
| **Symbol Wizard** | Generated ESP32-S3 73-pin symbol from datasheet PDF → Excel → Altium import |
| **Manufacturer Part Search** | LCSC/DigiKey part numbers pulled directly into schematic for BOM automation |
| **3D STEP Models** | Every component has 3D model (3D Content Central) for mechanical DRC |

---

## Project Structure

```
esp32-s3-mini/
├── Schematic/
│   ├── ESP32-S3-Mini.SchDoc          # Main schematic
│   └── ESP32-S3-Mini.SchLib          # Component symbols
│
├── PCB/
│   ├── ESP32-S3-Mini.PcbDoc          # PCB layout
│   ├── ESP32-S3-Mini.PcbLib          # Component footprints
│   └── ESP32-S3-Mini.rules           # Design rules (6 mil, impedance)
│
├── Library/
│   ├── ESP32-S3-MINI.SchLib          # MCU symbol (wizard-generated)
│   ├── ESP32-S3-MINI.PcbLib          # MCU footprint
│   └── 3D_Models/                    # STEP files for all components
│
├── Manufacturing/
│   ├── Gerbers/                      # Production Gerber files
│   ├── BOM.xlsx                      # Bill of Materials (LCSC PNs)
│   ├── Pick_Place.csv                # SMT placement file
│   └── Stackup_JLC3313.pdf          # Impedance-controlled stackup
│
├── Docs/
│   ├── Schematic_PDF.pdf             # Exported schematic for review
│   └── 3D_Render.png                 # Board 3D visualization
│
└── README.md
```

---

## Getting Started

### Prerequisites
- **Altium Designer** (21.x or later)
- **JLCPCB account** (for PCB fabrication + SMT assembly)

### Open the Project
```
1. Clone this repository
2. Open ESP32-S3-Mini.PrjPcb in Altium Designer
3. Review schematic → Run ERC
4. Review PCB → Run DRC
5. Generate manufacturing files: File → Fabrication Outputs → Gerber
```

### Order from JLCPCB
```
1. Upload Gerbers to jlcpcb.com
2. Select: 4-layer, JLC3313 stackup, impedance control
3. Enable SMT assembly → Upload BOM.xlsx + Pick_Place.csv
4. Review placement → Order
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| **Altium Designer** | Schematic capture, PCB layout, DRC, BOM generation |
| **Altium Symbol Wizard** | Auto-generate high-pin-count symbols from datasheet |
| **3D Content Central** | STEP models for mechanical DRC |
| **JLCPCB** | 4-layer PCB fabrication + SMT assembly |
| **Impedance Calculator** | Track width calculation for 50Ω / 90Ω targets |

---

## References

- [ESP32-S3-MINI Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-s3-mini-1_mini-1u_datasheet_en.pdf) — Espressif
- [ESP32-S3 Hardware Design Guidelines](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/index.html) — Antenna layout, decoupling, stackup
- [JLCPCB Capabilities](https://jlcpcb.com/capabilities/pcb-capabilities) — DRC rules aligned to standard manufacturing
- [JLC3313 Stackup](https://jlcpcb.com/impedance) — 4-layer impedance-controlled stackup

