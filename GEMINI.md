# Agent Instructions & Project Memory (GEMINI.md)

Welcome! This repository contains KiCad hardware design projects, automation workflows, and manufacturing exports managed in Antigravity.

## Mandatory Pre-Flight Checklist

Before proposing or executing any schematic or PCB modifications in this repository, you **MUST**:

1. **Read Persistent Lessons Learned First:**
   - Review [`docs/lessons-learned.md`](docs/lessons-learned.md) to understand known KiCad MCP server quirks, component layout constraints, ERC/DRC workarounds, and calculations.

2. **Follow Persistent Engineering Rules:**
   - Strictly adhere to [`.agents/rules/kicad-pcb.md`](.agents/rules/kicad-pcb.md), which is always active:
     - Verify board dimensions (`Edge.Cuts`), courtyard sizes, and pinout documentation prior to placement/routing.
     - Enforce mandatory verification gates: `kicad-cli sch erc` after schematic changes and `kicad-cli pcb drc` after layout changes.
     - Observe high-current routing rules (0.8 mm+ traces), hot-loop priority (CIN directly to VIN/PGND on top layer without vias), and quiet isolated feedback routing.

3. **Keep Lessons Learned Up-to-Date:**
   - Whenever you debug an issue, resolve an unexpected DRC/ERC error, find a workaround for an MCP tool quirk, or refine a design rule, **you must append the new reusable findings to [`docs/lessons-learned.md`](docs/lessons-learned.md)** so future agents retain that memory.

4. **Verify Clean DRC & ERC Before Committing:**
   - Run the exact `kicad-cli` commands specified in [`docs/lessons-learned.md`](docs/lessons-learned.md) to ensure 0 violations, 0 unconnected items, and 0 warnings.
