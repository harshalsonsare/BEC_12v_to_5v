# Engineering Memory & Lessons Learned

This document serves as the persistent engineering memory for KiCad PCB and schematic design in this repository. It documents hard-won discoveries, tool quirks, DRC/ERC troubleshooting patterns, component selections, and validated workflows.

---

## 1. KiCad MCP Server Quirks & Solutions

### A. Net Class Creation & Persistence
- **Issue**: Invoking `create_netclass` via the KiCad MCP server often reports success, but the net assignments may fail to persist into the project configuration (`.kicad_pro`).
- **Solution**: Explicitly verify net assignments in `.kicad_pro` under `netclass_patterns`. Use `add_net` to ensure net definitions exist in the board database, and verify netclass assignments before running automated routing or DRC.

### B. Schematic-to-PCB Netlist Synchronization
- **Issue**: Synchronizing a modified schematic to an existing board (`sync_schematic_to_board`) can leave orphan copper traces or disconnected pads if pin numbers or net labels were refactored.
- **Solution**: Always run `run_erc` first. Confirm all symbol pin names match the schematic wires. After updating the board, run a connectivity check (`kicad-cli pcb drc --severity-all --refill-zones`) to verify 0 unconnected pads before manual routing.

### C. MCP Execution Environment
- **Configuration**: `KiCAD-MCP-Server` requires:
  - `KICAD_PYTHON`: Python virtual environment with KiCad module bindings.
  - `PYTHONPATH`: `/usr/lib/kicad/lib/python3/dist-packages` (or system-specific path) to load `pcbnew`.
- **Lock Files (`.lck`)**: When KiCad GUI is open, it creates lock files (e.g. `~bec_12v_to_5v.kicad_pro.lck`). Do NOT attempt to force-delete these while KiCad is active; ensure `.gitignore` excludes them.

---

## 2. Schematic Capture & ERC Lessons

### A. Power Flag (`PWR_FLAG`) Requirements
- **Rule**: In KiCad, ERC checks whether power pins (`power_in`) are actively driven by a `power_out` pin. Connectors (e.g., XT30, screw terminals, headers) have passive pins, so connecting them directly to power nets flags an ERC error: *"Pin connected to other pins, but not driven by any pin"*.
- **Fix**: Attach a `power:PWR_FLAG` symbol to both the positive power rail (`VIN_12V`) and the ground rail (`GND`).

### B. UVLO Divider & IC Enable Pin Ratings
- **Rule**: Switching regulator enable pins frequently have lower absolute maximum voltage ratings than the VIN rail.
- **Example**: For the TI TPS54302 buck IC, `EN` has an absolute maximum rating of 7.0 V, while `VIN` operates up to 28 V (30 V abs max). Wiring `EN` directly to `VIN` will destroy the IC. Leaving `EN` disconnected risks floating input noise and ERC warnings.
- **Fix**: Implement an external precision resistor divider ($R_3 = 511	ext{ k}\Omega$, $R_4 = 105	ext{ k}\Omega$) from `VIN` to `GND`. This sets turn-on threshold at ~6.9 V and turn-off threshold at ~5.7 V, protecting the pin, suppressing inrush, and providing a deterministic driven net for ERC.

### C. Precision Voltage Divider & Phase Margin
- **Calculation**: For TPS54302 with reference voltage $V_{ref} = 0.596	ext{ V}$:
  $$V_{OUT} = V_{ref} \times \left(1 + \frac{R_1}{R_2}\right) = 0.596\text{ V} \times \left(1 + \frac{93.1\text{ k}\Omega}{12.6\text{ k}\Omega}\right) = 4.9998\text{ V}$$
- **Passives**: Use 0.5% tolerance (or 0.1%) standard SMD resistors (0805) to eliminate output voltage offset.
- **Feedforward Capacitor ($C_{FF}$)**: When using all-ceramic output capacitors with low ESR, add a 75 pF C0G capacitor ($C_8$) across the top feedback resistor ($R_1$) to inject phase lead and ensure stable loop phase margin (>60°).

### D. Unconnected Pins
- **Rule**: Any unused physical pins (e.g. the Signal pin on 3-pin BEC output connector `J2`) must have an explicit KiCad `no_connect` flag attached.

---

## 3. PCB Layout, Routing & DRC Resolutions

### A. Hot-Loop Layout (Rule 4 Compliance)
- **Principle**: In a buck converter, the input high-frequency loop ($C_{IN} \rightarrow Q_{high} \rightarrow Q_{low} \rightarrow \text{PGND}$) carries high $di/dt$ pulsed currents. Parasitic inductance here causes high-frequency voltage ringing and severe radiated EMI.
- **Implementation**:
  - Place ceramic input capacitor $C_2$ (100 nF 50V 0805) directly adjacent to `U1` (spacing < 3 mm).
  - Connect $C_2$ pad 1 directly to $U_1$ pin 3 (`VIN`) with a 0.8 mm top-layer (`F.Cu`) trace.
  - Connect $C_2$ pad 2 directly to $U_1$ pin 1 (`PGND`) with a 0.8 mm top-layer (`F.Cu`) trace.
  - Neither connection may pass through vias before reaching the IC pins.

### B. Switching Node ($SW$) Optimization
- **Principle**: The $SW$ node carries high $dv/dt$ rectangular voltage waveforms. Large surface area creates capacitive EMI coupling to nearby traces.
- **Implementation**:
  - Keep the trace connecting IC pin 2, inductor $L_1$ terminal 1, and bootstrap cap $C_3$ terminal 2 wide (0.8 mm – 1.0 mm) but short (< 6 mm length).
  - Place bootstrap capacitor $C_3$ immediately adjacent to the `BOOT` pin.

### C. Quiet Remote Feedback ($FB$) Routing
- **Principle**: The feedback pin is a high-impedance node susceptible to capacitive and magnetic noise from the inductor and switching node.
- **Implementation**:
  - Pick up output voltage sensing remotely at the output connector header ($J_2$ pin 2), past output filter capacitors $C_4$ and $C_5$.
  - Route the $FB$ trace on the bottom layer (`B.Cu`), fully shielded underneath the solid top and bottom ground planes.
  - Maintain > 3 mm physical clearance from inductor $L_1$ and route orthogonally to switching tracks.

### D. Ground Planes & Thermal Stitching Vias
- **Implementation**:
  - Pour solid `GND` copper zones across both `F.Cu` and `B.Cu`.
  - Place a dense stitching via cluster (0.6 mm diameter, 0.3 mm drill) around the IC power ground ($U_1.1$), input decoupling ($C_2.2$), bulk capacitor ground, and output connector return.
  - Stitching vias tie top and bottom planes together, providing low-impedance return paths and heatsinking heat into the bottom plane.

### E. Component Selection & Capacitor DC Bias De-rating
- **Issue**: Ceramic Class II capacitors (X5R/X7R) suffer significant capacitance loss under DC voltage bias (often losing 60–80% of rated capacitance at nominal voltage).
- **Solution**:
  - For 12 V input bulk capacitance, use a 35 V-rated 1210 package ($C_1$, 22 µF 35V X5R) rather than 25 V 1206.
  - Output capacitors use two 22 µF 25V 1206 capacitors in parallel ($C_4, C_5$) to handle 3 A ripple current with minimal voltage ripple (< 20 mV).

### F. Silkscreen Clearance Fixes
- **Issue**: Silkscreen text overlapping component copper pads or placed closer than 0.5 mm to the board edge (`Edge.Cuts`) generates DRC warnings (`silk_over_copper`, `silk_edge_clearance`).
- **Solution**:
  - Hide reference designators for mounting holes.
  - Scale text size to 1.0 mm – 1.2 mm and maintain at least 0.5 mm margin inside board edges.

---

## 4. Verification & CLI Toolchain Commands

Always run these standard verification gates from the repository root:

```bash
# 1. Run full schematic Electrical Rules Check (ERC)
kicad-cli sch erc --severity-all bec_12v_to_5v/bec_12v_to_5v.kicad_sch

# 2. Run full PCB Design Rules Check (DRC) with zone refill
kicad-cli pcb drc --severity-all --refill-zones -o bec_12v_to_5v-drc.rpt bec_12v_to_5v/bec_12v_to_5v.kicad_pcb

# 3. Export manufacturing Gerbers
kicad-cli pcb export gerbers -o exports/gerbers/ --use-drill-file-origin bec_12v_to_5v/bec_12v_to_5v.kicad_pcb

# 4. Export NC drill files (PTH and NPTH)
kicad-cli pcb export drill -o exports/drill/ --format excellon --generate-map --map-format gerber bec_12v_to_5v/bec_12v_to_5v.kicad_pcb

# 5. Export Bill of Materials (BOM)
kicad-cli sch export bom -o exports/bom/bec_12v_to_5v_bom.csv bec_12v_to_5v/bec_12v_to_5v.kicad_sch

# 6. Export Component Centroid / Pick-and-Place (POS)
kicad-cli pcb export pos -o exports/pos/bec_12v_to_5v_pos.csv --format csv --units mm bec_12v_to_5v/bec_12v_to_5v.kicad_pcb

# 7. Export 3D STEP Model
kicad-cli pcb export step -o exports/3d/bec_12v_to_5v.step bec_12v_to_5v/bec_12v_to_5v.kicad_pcb
```

---

## 5. Centering PCB Layouts on Sheet & Geometric Offsets

### A. Centering PCB on Standard Sheet (A4 / Letter)
- **Problem**: When a PCB is initially placed starting at origin $(0, 0)$, it appears in the top-left corner of the schematic/PCB drawing sheet.
- **Standard A4 Coordinates**: KiCad's A4 paper size is $297\text{ mm} \times 210\text{ mm}$, with its geometric center at $(148.5\text{ mm}, 105.0\text{ mm})$.
- **Atomic Translation Method**:
  1. Determine the exact bounding box of `Edge.Cuts` drawings: $(X_{min}, Y_{min})$ to $(X_{max}, Y_{max})$.
  2. Compute board center: $(X_c, Y_c) = ((X_{min} + X_{max})/2, (Y_{min} + Y_{max})/2)$.
  3. Calculate offset: $(dx, dy) = (148.5 - X_c, 105.0 - Y_c)$.
  4. In Python `pcbnew`, invoke `board.Move(pcbnew.VECTOR2I(pcbnew.FromMM(dx), pcbnew.FromMM(dy)))`.
  5. This atomically translates all board items (`Edge.Cuts`, graphics, footprints, tracks, vias, copper zones, and board text) while preserving all relative geometries and netlist connectivity.

---

## 6. Edge Connectors, Courtyards & Silkscreen Decluttering

### A. Right-Angle Connectors & Off-Board Geometries
- **Problem**: Edge-mounted right-angle connectors (e.g., `AMASS_XT30PW-M`) feature physical bodies that protrude off-board. Library footprints often include courtyards, silkscreen lines, fab outlines, and dummy mechanical pads extending into negative coordinates ($x < 0$).
- **Issues Encountered**:
  - `copper_edge_clearance`: Pads placed near or on the edge violate default board edge clearance constraints (typically 0.5 mm).
  - `lib_footprint_mismatch`: Pruning off-board dummy pads triggers DRC library mismatch warnings.
  - Courtyard or fab text extending outside `Edge.Cuts`.
- **Standard Resolutions**:
  - **Custom DRU Rule**: Allow negative edge clearance specifically for the connector parent in `<project>.kicad_dru`:
    ```lisp
    (rule "J1 Edge Clearance"
        (condition "A.Parent == 'J1'")
        (constraint edge_clearance (min -1.0mm)))
    ```
  - **Ignore Library Mismatch**: Set `"lib_footprint_mismatch": "ignore"` in `.kicad_pro` under `drc_exclusions`.
  - **Courtyard Clamping**: Clamp courtyard segment endpoints to stay within board boundaries ($x \ge 0.05\text{ mm}$) to retain a closed, valid courtyard polygon.
  - **Prune Off-Board Geometry**: Remove off-board mechanical dummy pads and graphical outlines beyond the board edge.

### B. Silkscreen Decluttering on Dense SMD Boards
- **Rule**: On high-density boards (e.g., $25\text{ mm} \times 18\text{ mm}$), verbose footprint `Value` strings (e.g. `"XT30PW-M (12V IN)"`, `"Servo Out (S/+5V/GND)"`, `"10uH SRP7028A-100M"`) overlap solder pads or extend outside `Edge.Cuts`.
- **Solution**:
  - Hide footprint `Reference` and `Value` texts (`SetVisible(False)`) and reposition them inside the footprint centroid (`SetPosition(fp.GetPosition())`) so their bounding boxes never extend beyond the board outline.
  - Add dedicated, clean silkscreen text objects (`PCB_TEXT`) using compliant font sizing ($\ge 0.8\text{ mm}$ height, $\ge 0.15\text{ mm}$ line thickness) placed with $>0.5\text{ mm}$ clearance from board edges and solder pads (e.g., `"12V IN"`, `"+"`, `"-"`, `"5V OUT"`).
