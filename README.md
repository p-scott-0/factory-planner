# Factory Planner

A browser-based production chain calculator for factory/automation games. Define machines, resources, and recipes for any game, then calculate exactly how many machines you need to hit a target output rate.

**Live version: https://p-scott-0.github.io/factory-planner/**

## Usage

Open `index.html` in any browser. No server or build step required.

## Features

- **Machines** with named tiers and speed multipliers (e.g. Mk1 x1.0, Mk2 x2.0)
- **Resources** organised into collapsible categories, with inline dependency tree diagrams
- **Recipes** assigned to machines, with multiple recipe alternatives per resource and a configurable default
- **Production planner** with two modes:
  - **Target output** — set a desired rate and get a full machine count + flow diagram
  - **From available inputs** — specify what you have and see the max achievable output with bottleneck analysis
- **Flow diagram** — SVG visualisation with category-grouped nodes, highway routing for long-distance connections, and per-stage efficiency indicators
- **Game profiles** — save, switch, duplicate, rename, export/import profiles so you can maintain separate setups for different games (Satisfactory, Factorio, Shapez, etc.)

## Profiles

All game data (machines, resources, categories, recipes, planner settings, layout) lives inside a named profile. Use the profile selector in the top-right of the nav bar to switch between games. Profiles can be exported as JSON files and shared.

Existing data from before profiles were added is automatically migrated into a "Default" profile on first load.

## Sample profiles

The [`profiles/`](profiles/) folder ships two ready-made profiles that double as reference examples for building your own:

- [`satisfactory.json`](profiles/satisfactory.json) — Satisfactory Tiers 0–8: nine machines (Miner Mk1–3, Smelter, Constructor, Assembler, Foundry, Water/Oil Extractors, Refinery, Manufacturer), 41 resources up to Space Elevator Phase 3 parts (Assembly Director System), belt tiers Mk1–Mk5, research-tier gating throughout, and four alternate recipes shipped toggled off (Cast Screw, Steel Screw, Iron Wire, Caterium Wire).
- [`factorio.json`](profiles/factorio.json) — Factorio early game: burner/electric drills, stone/steel/electric furnaces, assembling machines 1–3, yellow/red/blue belts, up to Automation Science Packs.

On the hosted version (or any local server) load them with the **Samples** buttons in the Profiles panel. When opening `index.html` directly from disk, use **Import Profile** and pick the JSON file instead (browsers block `fetch` on `file://`).

Numbers are approximate where the games don't map 1:1 onto this tool's model (e.g. footprints are in meters/tiles, miner rates assume normal-purity nodes).

### Profile JSON structure

A profile file is what **Export Profile** produces:

```jsonc
{
  "name": "My Game",
  "researchTiers": [ { "id": "rt0", "name": "Tier 0" } ],  // ordered progression ladder
  "unlockedTierId": null,                            // highest unlocked tier (null = all)
  "beltTiers": [
    { "id": "belt-x", "name": "Belt Mk1", "speed": 1, "researchTierId": "rt0" } // speed in items/sec
  ],
  "recToggles": { "rec-alt": false },                // recipe id → false when disabled
  "categories": [ { "id": "cat-x", "name": "Ores" } ],
  "machines": [
    {
      "id": "mach-x", "name": "Smelter",
      "researchTierId": "rt0",                       // optional gate (null = always)
      "tiers": [
        {
          "id": "tier-x", "name": "Mk1", "speedMult": 1,
          "researchTierId": "rt0",                   // tiers can be gated individually
          "buildCost": [ { "resourceId": "res-x", "amount": 5 } ]   // per tier
        }
      ],
      "width": 2, "height": 3,                       // footprint in grid cells
      "ports": [ { "type": "input", "side": "top", "offset": 0 } ]
    }
  ],
  "resources": [
    {
      "id": "res-x", "name": "Iron Plate", "categoryId": "cat-x",
      "researchTierId": "rt0",                       // optional gate (null = always)
      "recipes": [
        {
          "id": "rec-x", "name": "Smelt iron",
          "machineId": "mach-x",                     // null = raw/mined resource
          "cycleTime": 1.6,                          // seconds per cycle
          "outputAmount": 1,                         // output per cycle
          "inputs": [ { "resourceId": "res-y", "amount": 1 } ]
        }
      ],
      "defaultRecipeId": "rec-x"                     // preferred on cost ties
    }
  ],
  "selTiers": {},                                    // machineId → selected tier id
  "layout": { "instances": [], "belts": [] }         // layout editor contents
}
```

The planner picks recipes automatically: for each resource it uses the cheapest enabled + unlocked recipe chain, measured in raw inputs per unit of output. Disable recipes you haven't found (e.g. alternates) via `recToggles` or the checkboxes in the Planner tab.

IDs just need to be unique strings within the profile — the readable slugs in the samples are a convention, not a requirement. A resource with no recipes (or whose recipe has no machine) is treated as a raw input.

## Data storage

Everything is stored in `localStorage` under the keys `fp_profiles` and `fp_active`. No data leaves your browser.
