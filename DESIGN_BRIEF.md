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
- One or more **tiers** — named variants with a speed multiplier relative to the base (e.g. Mk1 ×1.0, Mk2 ×2.5). Tier selection affects how many machines the planner recommends. Each tier has its own optional **build cost** — resources and quantities required to construct one machine of that tier (higher tiers typically cost more). Used for the layout cost tracker.
- A **footprint** — width and height in grid cells, for layout purposes
- **I/O ports** — each port has a type (input or output), a side (top/right/bottom/left), a position offset along that side, and a flow kind (items or fluid). Used for belt/pipe routing in the layout editor and for the machine-output throughput cap.

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

### Research Tiers
Research tiers are an ordered, user-defined progression ladder (e.g. Tier 0 … Tier 8). Machines, individual machine tiers, resources, and belt tiers can each be gated behind a research tier. A profile-wide **highest unlocked tier** selector (in the nav bar) hides everything gated above it — from lists, planner selects, the layout palette, everywhere — so shared profiles don't spoil later-game content. Items with no research tier are always available. Tiers and their contents are managed in the dedicated **Tiers tab** (see UI structure), which deliberately shows everything regardless of the unlock level.

### Belt Tiers & Pipe Tiers
Belt tiers are named belt speeds (e.g. Belt Mk1 = 60/min), optionally research-gated. **Pipe tiers** are the same concept for fluids: resources marked as *fluid* flow through pipes instead of belts, and machine I/O ports declare whether they carry items or fluids (fluid ports render as squares, item ports as circles). The planner labels every flow-diagram edge and raw input with the **lowest allowed belt/pipe tier that can carry the rate**, or "N× fastest" when even the fastest tier needs parallel lanes. **Max belt tier** and **max pipe tier** settings in the planner further cap which tiers are considered, below the research unlock. In the layout editor a belts & pipes palette selects what new connections are drawn as; belts and pipes are colour-coded by kind and tier and the cost tracker counts segments per tier.

Belt speeds also cap machine throughput: a machine cannot output more than (number of output ports × best allowed belt speed) — one output port can only feed one belt. Belt-capped stages run below full speed, need proportionally more machines, consume inputs at the reduced rate, and are flagged with a warning in the results and diagram. Machines with no output ports defined are never capped.

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
- Recipes are toggled **enabled/disabled** rather than manually assigned: the planner automatically picks, per resource, the cheapest enabled and unlocked recipe chain, measured in raw-equivalent inputs per unit of output (raw resources and mining recipes count 1 per unit; ties go to the default recipe).

### From-Inputs Mode
The user specifies a set of input resources and rates, and a target resource. The planner finds the maximum achievable output rate given the supply constraint, identifies the bottleneck resource, and shows each machine stage's efficiency (what fraction of full capacity it runs at).

Key behaviours:
- Machine counts are fractional/exact — machines are not ceiled because the constraint is the input supply, not a round-number target.
- Efficiency is displayed per stage so the user can see which machines are underutilised.
- Recipe selection considers only chains that terminate at the supplied inputs — nothing may be mined or conjured from outside the supply, and the planner automatically falls back to alternate recipes that work with what's available.
- All stages that can't be satisfied by the given inputs are surfaced as an error.

### Flow Diagram
Results are displayed as a visual graph alongside the machine list. The diagram shows:
- One node per recipe stage, labelled with machine, recipe name, count, and actual output rate
- Columns aligned to the top of the diagram (matching the results list beside it)
- Each flow into or out of a node gets its own entry/exit point spread along the node's edge, so multiple flows never converge on one midpoint
- Every edge ends in a straight horizontal run so the line enters the back of its arrowhead
- Edges labelled with the flow rate, required belt tier, and resource name — labels float just above the line's approach into the consumer (never covering the line), one label block per entry point; highway edges carry a single-line label on their lane
- Raw input nodes at the far end of the graph
- Nodes grouped left-to-right by dependency depth (raw inputs on the left, target on the right, following natural reading direction)
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
- **Select** — click to select a machine instance (shift-click toggles); drag on empty space for box/marquee selection; drag any selected machine to move the whole selection; arrow keys nudge the selection by one cell; Ctrl+C / Ctrl+V copy and paste the selection (machines plus belts fully inside it) at the mouse position; R rotates a single selected machine; Delete/Backspace removes the selection. Dragged machines **soft-snap** into alignment with nearby machines when an edge is one cell off.
- **Place** — choose a machine type from the palette, then click to place instances; ghost preview shows the machine footprint and whether the placement is valid; R to rotate before placing
- **Belt** — click an output port to start routing; click cells to add waypoints; click an input port on a different machine to finish; right-click or Escape to cancel; belt endpoints snap to the external cell adjacent to the port
- **Erase** — click a machine instance to delete it (and all belts connected to it); click a belt path cell to delete that belt

### Belt behaviour
Belts and pipes are always drawn as clean orthogonal runs with 90° corners (paths are rasterised through their waypoints). They are **attached to their endpoint ports**: moving or rotating a machine re-routes every connected belt so it stays plugged into the same input/output.

### Machine rendering
Machines are colour-coded with a stable per-machine colour that also appears next to machine names in the planner. Labels scale with the machine's footprint (name + tier, with the active recipe/product on a second line for machines placed from a plan) and hide when zoomed too far out. The canvas zooms out to 3% for very large factories.

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
The sidebar shows one entry per machine tier (since tiers have different build costs) with the machine's footprint size. Clicking an entry selects it for placement. Clicking the same entry again deselects it.

### Build Cost Tracker
The sidebar accumulates the total build cost of all placed machines. It shows:
- Per-machine-type subtotals (name × count, with cost breakdown)
- A total belt segment count
- Grand totals across all resource types

The cost panel updates in real time as machines are placed, moved, or deleted. Subtotals are per machine tier, since tiers can have different build costs.

### Planner Hand-off
Planner results include a **Build in Layout** action that places the calculated machine counts into the layout editor — one column per stage, ordered raw-side-left to target-right like the flow diagram, using the tiers selected in the planner, with machines rotated so outputs face right and inputs face left. New machines are placed beside any existing layout content, never over it.

Belts are generated automatically in a **manifold style**: each producer's output stubs right into a vertical trunk that runs upward merging the column's outputs, peels off to the right along a dedicated corridor row above the factory, then drops down a feed line on the consumer column's left and taps into each machine's input. When the merged flow would exceed the fastest allowed belt/pipe tier, the manifold splits into parallel trunk/feed lanes, each tagged with the cheapest tier that carries its share. Fluid flows use pipes and prefer fluid-typed ports. Generated belts are normal attached belts and can be re-routed, erased, or redrawn manually afterwards.

If the layout already has content, Build in Layout asks whether to **append beside** it, **overwrite** the whole layout, or cancel (so the current layout can be saved as a blueprint first). It always switches to the Layout tab.

### Blueprints
The layout sidebar can save the current layout as a named **blueprint**, optionally under a user-defined category. Saving under an existing name overwrites it (load → edit → resave round-trips cleanly; loading pre-fills the save form). Blueprints can be **loaded** (replacing the layout) or **placed** — stamped into the current layout as a normal copy of machines and belts, arriving selected so it can be dragged into position (sub-blueprints by composition). Placement is non-associative: the copy does not track later edits to the blueprint.

A configurable **blueprint bounds box** (width × height in cells, e.g. Satisfactory's in-game blueprint designer limits) can be shown as a dashed rectangle at the origin to check what fits inside a buildable blueprint.

### Rendering performance
The canvas renders at most once per animation frame, culls machines and belts outside the viewport, and draws belts corner-to-corner rather than cell-by-cell, so factories with hundreds of machines and over a thousand belts pan smoothly.

---

## User Interface Structure

### Navigation
Five tabs: **Machines**, **Resources**, **Tiers**, **Planner**, **Layout**. A profile selector, profile management panel, the highest-unlocked-research-tier selector, and a global **items/min ↔ items/sec** display toggle are persistent in the nav bar. The rate-unit toggle affects every rate shown in the app.

### Machines Tab
A form for defining and editing machines (name, research tier, tiers with per-tier build costs and research gates, footprint, ports) alongside a list of all defined machines with tier badges and port counts. The belt tier editor (name, speed, research tier) also lives here.

### Resources Tab
Two-column layout:
- Left: form for adding/editing a resource (name, category, optional inline recipe for new resources)
- Right: categorised list of all resources. Each resource item shows badges for its recipes and has a button to open the recipe editor.

The **recipe editor** is an inline panel (same page, not a modal) that shows all recipes for the selected resource, with the ability to add, edit, delete, and set the default recipe. The editor closes when saved or cancelled.

Resources are grouped under collapsible category headers. The uncategorised group appears last if named categories also exist.

Each resource item can show a collapsible **dependency tree diagram** — a visual tree of the resource's recipe inputs, recursively expanded to raw resources.

### Tiers Tab
Manages the research-tier ladder: add, rename, reorder, and delete tiers. Each tier is a card showing everything gated at that tier (machines, machine tiers, resources, belts) with one-click unassignment, an assign dropdown to move any item into the tier, and a locked/unlocked indicator relative to the current unlock level. An "always available" card lists everything with no gate. This tab ignores the unlock filter by design — it is the authoring view.

### Planner Tab
Two sub-panels:
- Left: mode switcher (target / from-inputs), inputs for the chosen mode, a **Max Machine Tiers** card (tier selectors only for machines that actually have multiple unlocked tiers, plus a max-belt-tier cap), enable/disable checkboxes for resources with multiple recipes, and the machine result list
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
- **Planner seeds, layout decides** — planner results can be pushed to the layout editor as a starting arrangement via Build in Layout, but the layout never auto-syncs afterwards; users arrange machines and route belts freely
- **Visual but not simulation** — belts and ports are for planning/communication, not for running a simulation
- **Snap-to-grid always** — fractional positions are not a design goal; factory games use grids
