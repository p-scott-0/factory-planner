# Factory Planner — Design Brief

## Purpose

A tool for players of factory and automation games (e.g. Factorio, Satisfactory, Shapez, Mindustry) to answer two practical questions:

1. **How many of each machine do I need** to reach a target production rate?
2. **How should those machines be arranged** in physical space?

The tool is fully configurable — users define their own machines, resources, and recipes for whatever game they're playing, rather than having a single game hard-coded in.

---

## Core Concepts

### Machines
A machine is a building that processes resources. Machines have:
- A **name**
- One or more **tiers** — named variants with a speed multiplier relative to the base (e.g. Mk1 ×1.0, Mk2 ×2.5). Tier selection affects how many machines the planner recommends.
- A **footprint** — width and height in grid cells, for layout purposes
- **I/O ports** — each port has a type (input or output), a side (top/right/bottom/left), and a position offset along that side. Used for visual belt routing in the layout editor.
- A **build cost** — optional list of resources and quantities required to construct one machine. Used for the layout cost tracker.

### Resources
A resource is anything that can be produced, consumed, or mined. Resources have:
- A **name**
- An optional **category** for organisational grouping
- Zero or more **recipes** — any resource can have multiple alternative recipes, reflecting different production paths
- A **default recipe** — the recipe the planner uses unless the user overrides it

### Recipes
A recipe represents one way to produce a resource. Recipes have:
- An optional **machine** assignment (no machine = raw/mined resource)
- A **cycle time** in seconds
- An **output amount** per cycle
- Zero or more **input resources**, each with an amount per cycle

A resource with no recipe is treated as a raw input — something that must be supplied externally.

### Categories
Categories are user-defined labels for grouping resources. They affect visual organisation in the resource list and the flow diagram colour-coding. Categories have no mechanical effect.

---

## Game Profiles

All machine, resource, recipe, and category data is scoped to a named **profile**. Profiles let users maintain separate configurations for different games without interference. Users can:
- Create, rename, and delete profiles
- Duplicate a profile as a starting point
- Export a profile to a file and import it on another device or share it

The planner state (tier selections, recipe overrides, layout) is also profile-scoped.

---

## Production Chain Calculator

### Target Mode
The user specifies a target resource and a desired output rate (per second, minute, or hour). The planner works backwards through the dependency graph to produce a complete bill of machines and raw material inputs.

Key behaviours:
- Uses the full dependency tree — if Iron Plates require Iron Ore, and Gears require Iron Plates, a target of Gears will surface all three stages.
- Accounts for the selected tier's speed multiplier when calculating machine counts.
- **Machine count is always ceiled** to a whole number. Input provisioning is then based on that ceiled count so every built machine is fully fed and runs at 100% capacity.
- Diamond dependencies (same resource demanded by multiple downstream stages) accumulate correctly — the tool does not double-count machines.
- The user can override which recipe is used for any resource that has multiple alternatives.

### From-Inputs Mode
The user specifies a set of input resources and rates, and a target resource. The planner finds the maximum achievable output rate given the supply constraint, identifies the bottleneck resource, and shows each machine stage's efficiency (what fraction of full capacity it runs at).

Key behaviours:
- Machine counts are fractional/exact — machines are not ceiled because the constraint is the input supply, not a round-number target.
- Efficiency is displayed per stage so the user can see which machines are underutilised.
- All stages that can't be satisfied by the given inputs are surfaced as an error.

### Flow Diagram
Results are displayed as a visual graph alongside the machine list. The diagram shows:
- One node per recipe stage, labelled with machine, recipe name, count, and actual output rate
- Edges between nodes labelled with the resource name and flow rate
- Raw input nodes at the far end of the graph
- Nodes grouped left-to-right by dependency depth (target on the left, raw inputs on the right)
- Nodes within each column sorted by category then name
- Edges that skip columns (long-range dependencies) routed via horizontal highway lanes above or below the node area so they don't overlap node boxes
- Category-based colour coding on node borders
- Efficiency bands on nodes in from-inputs mode

---

## Layout Editor

A 2D grid-based canvas where users plan the physical placement of machines.

### Grid
- Infinite, pannable and zoomable canvas
- Machines snap to a fixed cell grid — fractional placement is not supported (matches how most factory games work)
- A coordinate HUD shows the current grid position and zoom level

### Tools
- **Select** — click to select a machine instance; drag to move it; R to rotate 90°; Delete/Backspace to remove
- **Place** — choose a machine type from the palette, then click to place instances; ghost preview shows the machine footprint and whether the placement is valid; R to rotate before placing
- **Belt** — click an output port to start routing; click cells to add waypoints; click an input port on a different machine to finish; right-click or Escape to cancel; belt endpoints snap to the external cell adjacent to the port
- **Erase** — click a machine instance to delete it (and all belts connected to it); click a belt path cell to delete that belt

### Placement Rules
- Machines cannot overlap — placement is blocked if any cell in the footprint is occupied by another instance
- Move/drag respects the same collision rule (a machine can't be dragged into another)
- Rotation is blocked if the rotated footprint would overlap another machine

### Ports and Belts
- Ports are rendered as coloured dots on machine edges (input = red, output = green)
- Belts are manual paths connecting one output port to one input port, via user-defined waypoints
- Belts are purely visual/organisational — they don't enforce flow constraints or validate resource compatibility
- Belt paths store grid-cell waypoints; the first cell is the one adjacent to the output port, the last is the one adjacent to the input port

### Machine Palette
The sidebar shows all defined machines with their footprint size. Clicking a machine selects it for placement. Clicking the same machine again deselects it.

### Build Cost Tracker
The sidebar accumulates the total build cost of all placed machines. It shows:
- Per-machine-type subtotals (name × count, with cost breakdown)
- A total belt segment count
- Grand totals across all resource types

The cost panel updates in real time as machines are placed, moved, or deleted.

---

## User Interface Structure

### Navigation
Four tabs: **Machines**, **Resources**, **Planner**, **Layout**. A profile selector and profile management panel are persistent in the nav bar.

### Machines Tab
A form for defining and editing machines (name, tiers, footprint, ports, build cost) alongside a list of all defined machines with tier badges and port counts.

### Resources Tab
Two-column layout:
- Left: form for adding/editing a resource (name, category, optional inline recipe for new resources)
- Right: categorised list of all resources. Each resource item shows badges for its recipes and has a button to open the recipe editor.

The **recipe editor** is an inline panel (same page, not a modal) that shows all recipes for the selected resource, with the ability to add, edit, delete, and set the default recipe. The editor closes when saved or cancelled.

Resources are grouped under collapsible category headers. The uncategorised group appears last if named categories also exist.

Each resource item can show a collapsible **dependency tree diagram** — a visual tree of the resource's recipe inputs, recursively expanded to raw resources.

### Planner Tab
Two sub-panels:
- Left: mode switcher (target / from-inputs), inputs for the chosen mode, tier selectors per machine, recipe override selectors for resources with multiple recipes, and the machine result list
- Right: the flow diagram

### Layout Tab
Full-height canvas with a sidebar on the left containing the tool buttons, rotation control, machine palette, and build cost panel.

---

## Data Persistence

All data is stored locally in the browser across sessions. No account, server, or network connection is required. Data is never sent anywhere.

---

## Design Principles

- **Game-agnostic** — no game-specific logic; all data is user-defined
- **Single-page, no install** — opens directly in a browser, no build step or server
- **Profiles as the top-level organiser** — switching profiles feels like switching games
- **Planner and layout are independent** — the layout editor does not auto-generate from planner results (by design, for flexibility); users decide how to arrange machines
- **Visual but not simulation** — belts and ports are for planning/communication, not for running a simulation
- **Snap-to-grid always** — fractional positions are not a design goal; factory games use grids
