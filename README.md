# Factory Planner

A browser-based production chain calculator for factory/automation games. Define machines, resources, and recipes for any game, then calculate exactly how many machines you need to hit a target output rate.

**Live version: https://p-scott-0.github.io/factory-planner/**

## Usage

Open `index.html` in any browser. No server or build step required.

## Features

- **Type-to-filter dropdowns** — any dropdown with 8+ options turns into a search box in place: click it and type to filter, with a tight typo-tolerant match ("iorn" finds Iron, "hmf" finds Heavy Modular Frame) that stays quiet rather than burying you in loose fuzzy hits
- **Named saved plans** — the planner holds any number of plans per profile, each with its own targets, recipe pins and overclocks; switch between them from the plan bar, and **Compare** any two as a side-by-side table of outputs, raw inputs, machine counts and total power
- **Per-plan recipe pins** — pin a specific recipe for a resource in *this plan only* (📌 next to the recipe toggles), overriding the cheapest-chain choice without flipping profile-wide toggles
- **Overclocking** — set a per-stage clock % in the results list; machines run faster (fewer needed) while power scales with clock^exponent (the exponent is a profile setting — the Satisfactory sample ships ≈1.32, matching the game)
- **Raw supply limits** — set an available rate on any raw input (e.g. "my map slice has 3900 crude") and the planner flags any plan that exceeds it
- **Share links** — copy a URL that encodes the whole plan; anyone with the same profile (e.g. the bundled sample) opens it to the exact same plan
- **Plan export** — download the flow diagram as a PNG (2×) or standalone SVG, or the stage list + raw inputs as CSV
- **Phone-friendly** — a proper mobile viewport with single-column layouts; the planner's results group by machine type (tap to expand each machine's recipes), and the recipe/tier cards collapse to one-line summaries with a filter that expands matches as you type
- **Edit undo** — machine/resource/recipe edits and deletes have a 20-step undo (the layout editor has its own Ctrl+Z)
- **Machines** with named tiers and speed multipliers (e.g. Mk1 x1.0, Mk2 x2.0)
- **Resources** organised into collapsible categories, with inline dependency tree diagrams
- **Recipes** assigned to machines, with multiple recipe alternatives per resource and a configurable default
- **Two recipe objectives** — From-Inputs mode defaults to **Optimal** (pick the chains that squeeze the most target out of the inputs you listed, searching the target's enabled recipes for the best achievable rate and listing whatever extra inputs the winner needs); switch to **Cheapest** to minimise total raw cost instead. Target mode always uses cheapest-by-raw-cost, weighted by each raw's `rawCost` scarcity (the Satisfactory sample prices hand-collected power slugs accordingly, so slug-powered generators no longer masquerade as the cheapest bulk power)
- **Diagram zoom** — −/+/Fit controls on the flow diagram (big plans auto-fit on phones), and dashed raw boxes now say *why* something is a raw input when it actually has recipes that are all disabled or research-locked
- **Read-only sample browsing** — locked sample profiles show everything (recipe lists, dependency trees with a per-recipe picker for alternates, machine/tier/port info); only the editing controls are hidden
- **Production planner** with two modes that share one engine — a priority-ordered list of targets where every target after the first can be planned entirely from an earlier target's byproducts, differing only in how the first target's rate is pinned down:
  - **Target output** — set a desired rate and get a full machine count + flow diagram
  - **From available inputs** — specify what you have and see the max achievable output with bottleneck analysis; also accepts secondary targets fed by the primary target's byproducts, exactly like Target mode
- **Flow diagram** — SVG visualisation with machines ordered to minimise belt crossings, highway routing for long-distance connections, and per-stage efficiency indicators
- **Game profiles** — save, switch, duplicate, rename, export/import profiles so you can maintain separate setups for different games (Satisfactory, Factorio, Shapez, etc.)

## Profiles

All game data (machines, resources, categories, recipes, planner settings, layout) lives inside a named profile. Use the profile selector in the top-right of the nav bar to switch between games. Profiles can be exported as JSON files and shared.

Existing data from before profiles were added is automatically migrated into a "Default" profile on first load.

## Sample profiles

The [`profiles/`](profiles/) folder ships two ready-made profiles that double as reference examples for building your own:

- [`satisfactory.json`](profiles/satisfactory.json) — Satisfactory Tiers 0–8: fifteen machines (Miner Mk1–3, Smelter, Constructor, Assembler, Foundry, Water/Oil/Well Extractors, Refinery, Manufacturer, Blender, Particle Accelerator, Coal/Fuel Generators, Nuclear Power Plant) with real per-tier power draws, and 89 resources spanning the full progression — steel, oil, quartz, sulfur/turbofuel, caterium electronics, the aluminum chain, nitrogen parts, the full nuclear chain (uranium fuel rods plus the waste-processing branch: Uranium Waste → Non-Fissile Uranium → Plutonium Pellet → Encased Plutonium Cell → Plutonium Fuel Rod, with Nitric Acid), the late-game fuel chain (Rocket Fuel and Ionized Fuel, plus Power Shards from the three power slugs), and all Space Elevator Phase 1–4 parts (through Nuclear Pasta and Thermal Propulsion Rockets). Power is plannable from seven sources (coal, fuel, turbofuel, uranium, plutonium, rocket fuel, ionized fuel). Belt tiers Mk1–Mk5, pipes Mk1–2, research-tier gating throughout, and **65 alternate recipes**, all shipped toggled off so the profile starts as a fresh save would — tick the ones whose hard drives you've actually found. They span the whole tree: ingots (Iron/Copper Alloy, Coke/Compacted Steel, Pure Aluminum), basics (Fine/Rubber Concrete, Steamed Copper Sheet, Fused Wire, Coated Iron Plate, Cheap Silica, Pure/Fused Quartz Crystal, Coated/Insulated/Quickwire Cable), frames and parts (Bolted/Stitched Iron Plate, Copper/Steel Rotor, Bolted/Steeled Frame, Encased Industrial Pipe, Heavy Encased/Flexible Frame, Rigor/Electric/Turbo Electric Motor), electronics (Silicon/Caterium Circuit Board, Crystal Computer, Super-State/OC Supercomputer, Automated Speed Wiring), the oil chain (Heavy Oil Residue, Polymer Resin, Residual Fuel/Plastic, Diluted Fuel, Turbo Heavy/Blend Fuel, Nitro Rocket Fuel, Recycled Plastic/Rubber), aluminium (Sloppy Alumina, Instant/Electrode Scrap, Alclad Casing, Heat Exchanger, Radio Connection Unit, Classic Battery, Heat-Fused Frame) and nuclear (Infused Uranium Cell, Uranium Fuel Unit, Fertile Uranium, Instant Plutonium Cell, Plutonium Fuel Unit).

Recipe values were checked against two independent sources apiece, with a `rate = amount × 60 / cycleTime` checksum against the published per-minute figures — several widely-mirrored datasets are still pre-1.0 and wrong (Compacted Steel Ingot is `2 Iron Ore + 1 Compacted Coal → 4 Steel Ingot @24s`, not `6/3/10 @16s`; Radio Connection Unit takes Quartz Crystal, not Quickwire; Uranium Fuel Unit takes Rotors, not the removed Beacon). Several recipes carry real byproducts: Fuel yields Polymer Resin (consumed by the "Alternate: Residual Plastic" recipe — a ready-made example for the planner's multi-target byproduct flow, see below), base Plastic and Rubber both yield Heavy Oil Residue (which the "Alternate: Residual Fuel" recipe turns into Fuel — so a Plastic or Rubber target can feed a second Power target through Residual Fuel + a fuel generator), Alumina Solution yields Silica and Aluminum Scrap yields Water (so an Aluminum Ingot target uses the Alumina's Silica byproduct instead of mining full Quartz — the planner credits byproducts within a single chain, not just across targets), the Nuclear Power Plant yields Uranium Waste (uranium fuel rods) or Plutonium Waste (plutonium fuel rods) — the Uranium Waste in turn feeds Non-Fissile Uranium, so a uranium power target can drive a plutonium production chain off its own waste, and Rocket Fuel yields Compacted Coal, which is an ingredient of the Turbofuel that Rocket Fuel is made from — a genuine feedback loop, and the reason the byproduct crediting iterates to a fixpoint rather than doing a single pass (100/min Rocket Fuel needs only 38/min of freshly-produced Compacted Coal, the rest coming back round from its own byproduct). Recipes are 1.0-accurate where the model allows; a handful of byproducts are still omitted, since a recipe with two co-equal primary outputs isn't modelled — only single-output-plus-byproduct is.
- [`factorio.json`](profiles/factorio.json) — Factorio early game: burner/electric drills, stone/steel/electric furnaces, assembling machines 1–3, yellow/red/blue belts, up to Automation Science Packs.

On the hosted version (or any local server) load them with the **Samples** buttons in the Profiles panel. When opening `index.html` directly from disk, use **Import Profile** and pick the JSON file instead (browsers block `fetch` on `file://`).

Sample profiles are **locked (read-only)** so they can **self-update**: each carries a `version`, and on load the app re-fetches the bundled copy and refreshes the definition (machines/resources/recipes) if a newer version has shipped — while keeping your own usage (recipe toggles, tier picks, planner query, layout). So when the bundled sample changes you get the update automatically instead of having to delete and reload the profile. To customise a sample, hit **Duplicate to edit** (or **Duplicate Current**) — the copy is a normal, editable profile. The app version is shown next to the logo in the nav bar.

Numbers are approximate where the games don't map 1:1 onto this tool's model (e.g. footprints are in meters/tiles, miner rates assume normal-purity nodes).

### Profile JSON structure

A profile file is what **Export Profile** produces:

```jsonc
{
  "name": "My Game",
  "version": 1,                                      // bundled samples only: bump to push a self-update
  "powerExponent": 1.321928,                         // optional: machine power scales with clock^exponent (default 1)
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
      "rawCost": 1,                                  // scarcity of one mined/extracted unit (default 1;
                                                     // lower for unlimited inputs, e.g. water 0.05)
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

A raw input counts as `1` by default, but resources can set **`rawCost`** to say how scarce a unit actually is — because not all raws are equal. In Satisfactory water is effectively unlimited while crude oil is node-capped, so pricing them the same made the planner prefer Residual Fuel (112.5 crude per 100 Fuel) over Diluted Fuel (37.5) — it couldn't see that diluting scarce crude with free water is the entire point of the recipe. The sample sets Water to `0.05`; with the fuel alternates enabled the planner now picks Diluted Fuel, which also drops Rocket Fuel's raw cost by ~31%. Set it low for whatever your game hands out for free (water, air), leave it at 1 for anything node-limited.

Both planner modes accept multiple targets in priority order. A target left without an amount is planned entirely from the byproducts of the targets before it (so a Fuel target that yields Polymer Resin as a byproduct can feed a second, amount-less Plastic target) — see the `satisfactory.json` sample's Fuel/Polymer Resin/Residual Plastic chain for a working example (enable the `rec-residual-plastic` toggle first, since it ships off by default). In From-Inputs mode the first target's rate comes from the supplied inputs instead of a fixed amount; every target after it works exactly like Target mode's secondary targets. The flow diagram shows each byproduct as a small satellite box branching off its producer — connected to whatever consumes it elsewhere in the same plan, or left dashed as unclaimed surplus if nothing does (also called out in a banner in the results panel).

IDs just need to be unique strings within the profile — the readable slugs in the samples are a convention, not a requirement. A resource with no recipes (or whose recipe has no machine) is treated as a raw input.

## Data storage

Everything is stored in `localStorage` under the keys `fp_profiles` and `fp_active`. No data leaves your browser.
