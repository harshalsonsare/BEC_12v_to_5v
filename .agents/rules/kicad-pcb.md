---
trigger: always_on
---
# KiCad PCB & Schematic Engineering Rules

1. **Verify Before Placing or Wiring:**
   - **Sheet Centering:** Always center the `Edge.Cuts` board outline on the A4 sheet center (`X = 148.5 mm, Y = 105.0 mm`) when creating or modifying board outlines.
   - **Text Boundary Check:** Keep all `F.SilkS` labels inside `Edge.Cuts` with at least 0.3 mm margin from the board edge; move long part numbers/values (e.g., `"XT30PW-M (12V IN)"`) onto `F.Fab` and use concise silkscreen labels (`"12V IN"`) on `F.SilkS`.
   - Always inspect board dimensions (`Edge.Cuts`), courtyard sizes, and exact component footprints via MCP or CLI before placing footprints.
   - Cross-check every IC/connector pin number and orientation against datasheets and pinout documentation before wiring.

2. **Mandatory DRC/ERC Gates:**
   - Run `kicad-cli sch erc --severity-all <schematic>` after schematic edits; resolve all errors and warnings before layout.
   - Run `kicad-cli pcb drc --severity-all --refill-zones <board>` after every layout and routing edit; ensure 0 DRC violations, 0 unconnected items, and 0 footprint/clearance errors before proceeding.

3. **Board & Fab Constraints:**
   - Define `Edge.Cuts` outline and mounting holes first before component placement.
   - Trace widths:
     - Minimum signal trace: 0.2 mm.
     - Power rails / high-current paths: 0.5 mm – 1.0 mm (use 0.8 mm+ for 3A+ paths).
     - Minimum clearance: 0.15 mm – 0.25 mm.
   - Pour solid GND copper zones on bottom and top layers, tied together with an array of thermal and RF stitching vias.
   - Keep silkscreen text >0.5 mm inside `Edge.Cuts` and away from exposed solder pads to eliminate clearance warnings.

4. **Switching Regulator & BEC Layout Rules:**
   - **Hot-Loop Priority:** The critical high-frequency AC loop—input decoupling capacitor (`CIN` / `C2`), buck IC (`VIN` and `PGND` pins), catch diode/low-side FET return—must be kept as tight as physically possible on the top layer (`F.Cu`) directly connected with wide copper without passing through vias first.
   - **Switching Node (`SW`):** Keep the `SW` copper trace/pour short and wide directly connecting the IC SW pin to the inductor and bootstrap capacitor (`CBOOT`). Minimize surface area to avoid capacitive EMI while handling peak switching currents.
   - **Quiet Feedback (`FB`):** Remote-sense the output voltage cleanly past the output filter capacitors. Never route the voltage-divider feedback trace underneath the inductor or parallel to the `SW` switching node. Route `FB` on the bottom layer shielded beneath the ground plane or maintain >3 mm physical separation.
   - **Bootstrap Capacitor:** Place bootstrap capacitor immediately adjacent to `BOOT` and `SW` pins with minimal parasitic trace inductance.
