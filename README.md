# KiCad Hardware Design & Automation Workspace

A complete, production-verified hardware repository featuring a high-efficiency **12V-to-5V 3A Synchronous Buck Converter (BEC)**, engineered and verified using **KiCad 10** and the **KiCad MCP (Model Context Protocol) Server** in Google Antigravity.

---

## 1. Project Overview & Specifications

### 12V-to-5V 3A Battery Eliminator Circuit (BEC)
Designed for robotics, UAV/FPV drones, RC models, and embedded computing (e.g. powering Raspberry Pi, flight controllers, and high-torque 5V servos from a 3S–4S LiPo battery or 12V DC rail).

| Parameter | Specification | Notes / Rationale |
| :--- | :--- | :--- |
| **Input Voltage ($V_{IN}$)** | **12 V nominal** (4.5 V – 18.0 V operational) | Compatible with 3S (11.1 V–12.6 V) and 4S (14.8 V–16.8 V) LiPo packs |
| **Input Transient Protection** | **SM6T18A TVS Diode** (600 W, $V_{RM} = 15.3\text{ V}$, $V_C = 25.2\text{ V}$) | Clamps voltage spikes safely below IC 30 V absolute max |
| **Output Voltage ($V_{OUT}$)** | **5.00 V DC** | Set via precision 0.5% divider ($R_1 = 93.1\text{ k}\Omega$, $R_2 = 12.6\text{ k}\Omega$) |
| **Output Current ($I_{OUT}$)** | **3.0 A continuous** | Thermally sustained on 2-layer 1 oz copper board |
| **Switching Frequency ($f_{SW}$)** | **400 kHz** fixed frequency | Optimized balance between size and switching efficiency |
| **Regulator IC** | **TI TPS54302DDCR** (SOT-23-6) | Synchronous buck converter with integrated 85 mΩ / 40 mΩ MOSFETs |
| **Power Inductor** | **Bourns SRP7028A-100M** (10 µH, shielded) | $I_{sat} = 6.0\text{ A}$, $I_{rms} = 3.5\text{ A}$, DCR = 75 mΩ |
| **Input Decoupling** | $22\ \mu\text{F}$ 35V X5R (1210) + $100\text{ nF}$ 50V X7R (0805) | High-voltage 1210 prevents severe ceramic DC bias capacitance drop |
| **Output Decoupling** | $2\times 22\ \mu\text{F}$ 25V X7R (1206) | Low ESR (< 10 mΩ), voltage ripple < 20 mV pk-pk at 3A load |
| **Phase Margin Boost** | $75\text{ pF}$ C0G feedforward cap ($C_8$) across $R_1$ | Stabilizes loop phase margin (>60°) with ceramic output capacitors |
| **UVLO Protection** | Precision divider: $R_3 = 511\text{ k}\Omega$, $R_4 = 105\text{ k}\Omega$ | Starts at ~6.9 V, stops at ~5.7 V; protects EN pin (7 V abs max) |
| **Input Connector ($J_1$)** | **AMASS XT30PW-M** (Horizontal / Right-Angle PCB) | High-reliability DC connector rated up to 30 A |
| **Output Connector ($J_2$)** | **3-Pin 2.54 mm Vertical Pin Header** | Standard RC servo pinout (Pin 1: NC/Signal, Pin 2: +5V, Pin 3: GND) |
| **PCB Dimensions** | **25.0 mm × 18.0 mm** | Ultra-compact form factor |
| **Layer Stackup** | **2-Layer FR-4**, 1.6 mm thickness, 1 oz Cu | Dual solid ground planes with thermal stitching array |
| **Design Status** | **100% Routed & Verified** | **0 DRC violations, 0 unconnected items, 0 ERC errors** |

---

## 2. Repository Structure

```text
KiCad/
├── .agents/
│   └── rules/
│       └── kicad-pcb.md           # Persistent agent engineering rules (always_on)
├── docs/
│   └── lessons-learned.md         # Persistent engineering memory, quirks & solutions
├── bec_12v_to_5v/                 # Core 12V-to-5V BEC KiCad project
│   ├── bec_12v_to_5v.kicad_pro    # Project settings & net classes
│   ├── bec_12v_to_5v.kicad_sch    # Schematic capture
│   ├── bec_12v_to_5v.kicad_pcb    # 25x18mm 2-layer routed board
│   ├── bec_12v_to_5v.kicad_dru    # Design rules
│   └── exports/                   # Project-local mirror of manufacturing outputs
├── DemoPCB/                       # MCP server interactive testing & validation board
│   ├── DemoPCB.kicad_pro
│   ├── DemoPCB.kicad_sch
│   └── DemoPCB.kicad_pcb
├── exports/                       # Production manufacturing & assembly packages
│   ├── gerbers/                   # RS-274X copper, soldermask, paste, silkscreen, edge cuts
│   ├── drill/                     # Excellon PTH & NPTH drill files and drill report
│   ├── bom/                       # Full Bill of Materials (CSV)
│   ├── pos/                       # Pick-and-place centroid coordinates (CSV)
│   ├── 3d/                        # Standard 3D CAD model (STEP AP214/AP242)
│   └── svg/                       # High-resolution vector renderings (top & bottom)
├── bec_12v_to_5v-drc.rpt          # Latest clean DRC report (0 errors, 0 warnings)
├── bec_12v_to_5v-erc.rpt          # Latest clean ERC report (0 errors, 0 warnings)
├── GEMINI.md                      # Agent pre-flight instructions & engineering guidelines
├── .gitignore                     # KiCad-optimized ignore rules (locks, backups, cache)
└── README.md                      # Project documentation and setup guide
```

---

## 3. Prerequisites & KiCad MCP Setup Guide

This project can be inspected and edited directly using the KiCad GUI or autonomously through the **KiCAD-MCP-Server**.

### Software Requirements
- **KiCad 10.0+** (`kicad`, `kicad-cli`, and `pcbnew` Python module)
- **Node.js 18+**
- **Python 3.10+** (with system `pcbnew` accessible)

### Configuring KiCAD-MCP-Server in Antigravity
The MCP server bridges Antigravity to KiCad's C++ routing and layout engine through `pcbnew` Python bindings.

Add the server to your `mcp_config.json` (typically located in `~/.gemini/config/mcp_config.json` or project root):

```json
{
  "mcpServers": {
    "kicad": {
      "command": "node",
      "args": [
        "/path/to/KiCAD-MCP-Server/dist/index.js"
      ],
      "env": {
        "KICAD_PYTHON": "/path/to/KiCAD-MCP-Server/venv/bin/python",
        "PYTHONPATH": "/usr/lib/kicad/lib/python3/dist-packages"
      }
    }
  }
}
```

> [!TIP]
> Ensure `PYTHONPATH` points to your system's KiCad Python dist-packages directory where `pcbnew.py` resides. On Ubuntu/Debian, this is `/usr/lib/kicad/lib/python3/dist-packages`.

---

## 4. Design & Verification Workflow

The design strictly adheres to a multi-gate verification pipeline:

```text
[ Schematic Capture ] ──> [ kicad-cli sch erc ] ──( 0 Errors )──> [ Netlist / PCB Sync ]
                                                                        │
[ DRC: 0 Violations ] <── [ Copper Pours & Vias ] <── [ Placement & Route ]
        │
        └───> [ Fabrication Exports (Gerber, Drill, POS, BOM, 3D STEP) ]
```

### Verification & Manufacturing Commands (`kicad-cli`)

#### 1. Electrical Rules Check (ERC)
Run ERC on the schematic to verify all pins are driven and connected:
```bash
kicad-cli sch erc --severity-all bec_12v_to_5v/bec_12v_to_5v.kicad_sch
```

#### 2. Design Rules Check (DRC)
Run DRC on the board with automatic polygon zone refill:
```bash
kicad-cli pcb drc --severity-all --refill-zones -o bec_12v_to_5v-drc.rpt bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

#### 3. Export Manufacturing Gerbers
```bash
kicad-cli pcb export gerbers -o exports/gerbers/ --use-drill-file-origin bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

#### 4. Export NC Drill Files
```bash
kicad-cli pcb export drill -o exports/drill/ --format excellon --generate-map --map-format gerber bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

#### 5. Export Bill of Materials (BOM)
```bash
kicad-cli sch export bom -o exports/bom/bec_12v_to_5v_bom.csv bec_12v_to_5v/bec_12v_to_5v.kicad_sch
```

#### 6. Export Pick-and-Place Centroid (POS)
```bash
kicad-cli pcb export pos -o exports/pos/bec_12v_to_5v_pos.csv --format csv --units mm bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

#### 7. Export 3D STEP Solid Model
```bash
kicad-cli pcb export step -o exports/3d/bec_12v_to_5v.step bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

#### 8. Export Vector SVGs
```bash
kicad-cli pcb export svg -o exports/svg/bec_12v_to_5v_top.svg --page-size-mode 2 --layers F.Cu,F.SilkS,F.Mask,Edge.Cuts bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
kicad-cli pcb export svg -o exports/svg/bec_12v_to_5v_bottom.svg --page-size-mode 2 --layers B.Cu,B.SilkS,B.Mask,Edge.Cuts bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

---

## 5. Engineering Architecture & Layout Highlights

1. **Hot-Loop Priority**: Input capacitor $C_2$ (100 nF ceramic) connects directly to $U_1.3$ (`VIN`) and $U_1.1$ (`PGND`) on the top layer (`F.Cu`) without passing through vias first. Loop distance is < 3 mm to suppress high-frequency switching EMI.
2. **Short Switching Node**: The $SW$ track between IC pin 2, inductor $L_1$, and bootstrap capacitor $C_3$ is routed with 0.8 mm width and kept under 6 mm in length.
3. **Isolated Feedback Path**: The $FB$ trace is remote-sensed at output header $J_2$, drops to the bottom layer (`B.Cu`), and runs completely shielded underneath the continuous ground plane with >3 mm clearance from inductor $L_1$.
4. **Thermal Stitching**: Dedicated arrays of 0.6 mm vias tie top and bottom ground planes together around the power IC, TVS diode, and decoupling return paths.
