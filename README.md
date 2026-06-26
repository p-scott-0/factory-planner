# Factory Planner

A browser-based production chain calculator for factory/automation games. Define machines, resources, and recipes for any game, then calculate exactly how many machines you need to hit a target output rate.

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

All game data (machines, resources, categories, recipes, planner settings) lives inside a named profile. Use the profile selector in the top-right of the nav bar to switch between games. Profiles can be exported as JSON files and shared.

Existing data from before profiles were added is automatically migrated into a "Default" profile on first load.

## Data storage

Everything is stored in `localStorage` under the keys `fp_profiles` and `fp_active`. No data leaves your browser.
