# Factory Planner

A browser-based production chain calculator for factory/automation games. Define machines, resources, and recipes for any game, then calculate exactly how many machines you need to hit a target output rate.

**Live version: https://p-scott-0.github.io/factory-planner/**

## Usage

Open `index.html` in any browser. No server or build step required.

## Features

- **Machines** with named tiers and speed multipliers (e.g. Mk1 x1.0, Mk2 x2.0)
- **Resources** organised into collapsible categories, with inline dependency tree diagrams
- **Recipes** assigned to machines, with multiple recipe alternatives per resource and a configurable default
- **Production planner** with two modes that share one engine — a priority-ordered list of targets where every target after the first can be planned entirely from an earlier target's byproducts, differing only in how the first target's rate is pinned down:
  - **Target output** — set a desired rate and get a full machine count + flow diagram
  - **From available inputs** — specify what you have and see the max achievable output with bottleneck analysis; also accepts secondary targets fed by the primary target's byproducts, exactly like Target mode
- **Flow diagram** — SVG visualisation with category-grouped nodes, highway routing for long-distance connections, and per-stage efficiency indicators
- **Game profiles** — save, switch, duplicate, rename, export/import profiles so you can maintain separate setups for different games (Satisfactory, Factorio, Shapez, etc.)

## Profiles

All game data (machines, resources, categories, recipes, planner settings, layout) lives inside a named profile. Use the profile selector in the top-right of the nav bar to switch between games. Profiles can be exported as JSON files and shared.

Existing data from before profiles were added is automatically migrated into a "Default" profile on first load.

## Sample profiles

The [`profiles/`](profiles/) folder ships two ready-made profiles that double as reference examples for building your own:

- [`satisfactory.json`](profiles/satisfactory.json) — Satisfactory Tiers 0–8: fifteen machines (Miner Mk1–3, Smelter, Constructor, Assembler, Foundry, Water/Oil/Well Extractors, Refinery, Manufacturer, Blender, Particle Accelerator, Coal/Fuel Generators, Nuclear Power Plant) with real per-tier power draws, and 83 resources spanning the full progression — steel, oil, quartz, sulfur/turbofuel, caterium electronics, the aluminum chain, nitrogen parts, the full nuclear chain (uranium fuel rods plus the waste-processing branch: Uranium Waste → Non-Fissile Uranium → Plutonium Pellet → Encased Plutonium Cell → Plutonium Fuel Rod, with Nitric Acid), and all Space Elevator Phase 1–4 parts (through Nuclear Pasta and Thermal Propulsion Rockets). Power is plannable from five sources (coal, fuel, turbofuel, uranium, plutonium). Belt tiers Mk1–Mk5, pipes Mk1–2, research-tier gating throughout, and twelve alternate recipes shipped toggled off (Cast/Steel Screw, Iron/Caterium Wire, Wet Concrete, Pure Iron/Copper Ingot, Solid Steel Ingot, Residual Plastic, Residual Fuel, Polymeric Heavy Oil Residue, Turbo Heavy Fuel). Four recipes carry real byproducts: Fuel yields Polymer Resin (consumed by the "Alternate: Residual Plastic" recipe — a ready-made example for the planner's multi-target byproduct flow, see below), base Plastic and Rubber both yield Heavy Oil Residue (which the "Alternate: Residual Fuel" recipe turns into Fuel — so a Plastic or Rubber target can feed a second Power target through Residual Fuel + a fuel generator), and the Nuclear Power Plant yields Uranium Waste (uranium fuel rods) or Plutonium Waste (plutonium fuel rods) — the Uranium Waste in turn feeds Non-Fissile Uranium, so a uranium power target can drive a plutonium production chain off its own waste. Recipes are 1.0-accurate where the model allows; most byproducts (e.g. other refinery residue) are still omitted, since a recipe with two co-equal primary outputs isn't modelled — only single-output-plus-byproduct is.
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
  "maxBeltTierId": null,                             // planner belt cap (null = fastest unlocked)
  "beltTiers": [
    { "id": "belt-x", "name": "Belt Mk1", "speed": 1, "researchTierId": "rt0" } // speed in items/sec
  ],
  "pipeTiers": [
    { "id": "pipe-x", "name": "Pipe Mk1", "speed": 5, "researchTierId": null } // belts for fluids
  ],
  "maxPipeTierId": null,
  "recToggles": { "rec-alt": false },                // recipe id → false when disabled
  "categories": [ { "id": "cat-x", "name": "Ores" } ],
  "machines": [
    {
      "id": "mach-x", "name": "Smelter",
      "researchTierId": "rt0",                       // optional gate (null = always)
      "tiers": [
        {
          "id": "tier-x", "name": "Mk1", "speedMult": 1,
          "power": 4,                                // MW drawn per machine (optional)
          "researchTierId": "rt0",                   // tiers can be gated individually
          "buildCost": [ { "resourceId": "res-x", "amount": 5 } ]   // per tier
        }
      ],
      "width": 2, "height": 3,                       // footprint in grid cells
      "ports": [ { "type": "input", "side": "top", "offset": 0, "flow": "item" } ] // flow: item|fluid
    }
  ],
  "resources": [
    {
      "id": "res-x", "name": "Iron Plate", "categoryId": "cat-x",
      "researchTierId": "rt0",                       // optional gate (null = always)
      "isFluid": false,                              // fluids flow through pipes
      "isPower": false,                              // power: per-second rate = MW, no belts/pipes
      "recipes": [
        {
          "id": "rec-x", "name": "Smelt iron",
          "machineId": "mach-x",                     // null = raw/mined resource
          "cycleTime": 1.6,                          // seconds per cycle
          "outputAmount": 1,                         // output per cycle
          "inputs": [ { "resourceId": "res-y", "amount": 1 } ],
          "byproducts": [ { "resourceId": "res-z", "amount": 0.5 } ]  // optional extra output per cycle
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

Both planner modes accept multiple targets in priority order. A target left without an amount is planned entirely from the byproducts of the targets before it (so a Fuel target that yields Polymer Resin as a byproduct can feed a second, amount-less Plastic target) — see the `satisfactory.json` sample's Fuel/Polymer Resin/Residual Plastic chain for a working example (enable the `rec-residual-plastic` toggle first, since it ships off by default). In From-Inputs mode the first target's rate comes from the supplied inputs instead of a fixed amount; every target after it works exactly like Target mode's secondary targets. The flow diagram shows each byproduct as a small satellite box branching off its producer — connected to whatever consumes it elsewhere in the same plan, or left dashed as unclaimed surplus if nothing does (also called out in a banner in the results panel).

IDs just need to be unique strings within the profile — the readable slugs in the samples are a convention, not a requirement. A resource with no recipes (or whose recipe has no machine) is treated as a raw input.

## Data storage

Everything is stored in `localStorage` under the keys `fp_profiles` and `fp_active`. No data leaves your browser.
