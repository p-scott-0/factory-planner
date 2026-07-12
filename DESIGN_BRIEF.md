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
- One or more **tiers** — named variants with a speed multiplier relative to the base (e.g. Mk1 ×1.0, Mk2 ×2.5). Tier selection affects how many machines the planner recommends. Each tier has its own optional **build cost** — resources and quantities required to construct one machine of that tier (higher tiers typically cost more) — and an optional **power draw in MW** per machine. Both feed the planner totals and the layout cost tracker.
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
- Zero or more **byproducts** — extra output resources produced per cycle alongside the primary output (e.g. a Fuel recipe that also yields Polymer Resin; Plastic and Rubber both yield Heavy Oil Residue; Alumina Solution yields Silica and Aluminum Scrap yields Water; the Nuclear Power Plant recipe yields Uranium Waste). Byproducts are shown as badges on the recipe and, where a plan produces them, on the corresponding machine stage in planner results and as a small satellite box in the flow diagram. **A byproduct offsets same-chain demand for that resource**: within a single target's plan, a recipe's byproduct is credited against anything else the chain needs of that resource, so its dedicated production (and raw inputs) shrink — e.g. Alumina Solution's Silica byproduct covers part of an Aluminum Ingot chain's Silica need, cutting the Quartz mined for it; Aluminum Scrap's Water byproduct offsets the water the Alumina Solution draws. Only the genuine leftover after in-chain use is surplus, and any leftover is what feeds later targets (see **Multi-Target Planning** below). Because a byproduct's producer can sit downstream of the consumer in the graph, the planner solves this by iterating to a fixpoint.

A resource with no recipe is treated as a raw input — something that must be supplied externally.

### Categories
Categories are user-defined labels for grouping resources. They affect visual organisation in the resource list and the flow diagram colour-coding. Categories have no mechanical effect.

### Research Tiers
Research tiers are an ordered, user-defined progression ladder (e.g. Tier 0 … Tier 8). Machines, individual machine tiers, resources, and belt tiers can each be gated behind a research tier. A profile-wide **highest unlocked tier** selector (in the nav bar) hides everything gated above it — from lists, planner selects, the layout palette, everywhere — so shared profiles don't spoil later-game content. Items with no research tier are always available. Tiers and their contents are managed in the dedicated **Tiers tab** (see UI structure), which deliberately shows everything regardless of the unlock level.

### Belt Tiers & Pipe Tiers
Belt tiers are named belt speeds (e.g. Belt Mk1 = 60/min), optionally research-gated. **Pipe tiers** are the same concept for fluids: resources marked as *fluid* flow through pipes instead of belts, and machine I/O ports declare whether they carry items or fluids (fluid ports render as squares, item ports as circles). The planner labels every flow-diagram edge and raw input with the **lowest allowed belt/pipe tier that can carry the rate**, or "N× fastest" when even the fastest tier needs parallel lanes. **Max belt tier** and **max pipe tier** settings in the planner further cap which tiers are considered, below the research unlock. In the layout editor a belts & pipes palette selects what new connections are drawn as; belts and pipes are colour-coded by kind and tier and the cost tracker counts segments per tier.

Belt speeds also cap machine throughput: a machine cannot output more than (number of output ports × best allowed belt speed) — one output port can only feed one belt. Belt-capped stages run below full speed, need proportionally more machines, consume inputs at the reduced rate, and are flagged with a warning in the results and diagram. Machines with no output ports defined are never capped.

### Power
Machine tiers declare a **power draw in MW**. The planner shows per-stage draw (installed: ceiled machine count × MW) and a **total power banner** for every plan, in the results and on diagram nodes; the layout cost tracker totals the draw of placed machines.

**Power generation is planned with the same recipe engine.** A resource can be flagged as **Power**: its per-second rate *is* megawatts, it displays as "X MW" regardless of the rate-unit toggle, and it travels over power lines — never belt/pipe capped or routed. Generator machines produce it through ordinary recipes that burn fuel (e.g. Coal Generator: 0.25 coal + 0.75 water per second → 75 MW), so generator counts are ceiled, fuel chains are planned recursively, alternate sources (coal vs fuel vs …) participate in recipe toggles and automatic cheapest-chain selection, research tiers gate them, from-inputs mode answers "how many MW from this much coal", and Build in Layout places generators with their fuel belts and pipes.

The total-power banner includes a **Plan generation →** button that targets the profile's Power resource at the required MW — one click from "my factory needs 528 MW" to a complete generator + fuel plan (whose own residual power draw, e.g. water extractors, is then shown for iteration). The layout cost tracker shows **Draw**, maximum **Generation** of placed generators, and the resulting **Balance**.

---

## Game Profiles

All machine, resource, recipe, and category data is scoped to a named **profile**. Profiles let users maintain separate configurations for different games without interference. Users can:
- Create, rename, and delete profiles
- Duplicate a profile as a starting point
- Export a profile to a file and import it on another device or share it

The planner state (tier selections, recipe overrides, layout) is also profile-scoped.

**Locked, self-updating sample profiles.** A profile loaded from a bundled sample is marked as a locked sample (it stores the sample key and a version). Locked profiles are read-only — the definition-editing UI (add/edit/delete machines, resources, recipes, categories, tiers, belts) is hidden and guarded, and a banner explains why — but usage (recipe toggles, tier picks, planner query, layout) stays editable. On load the app best-effort re-fetches each locked sample's bundled copy and, if it carries a newer version, refreshes the definition from it while preserving the user's usage (recipe toggles merge bundle defaults under the user's choices). This means a bundled sample can be improved and existing users pick up the change automatically, instead of having to delete and reload the profile. To customise a sample, **Duplicate** it — the copy is an ordinary editable profile with the lock cleared. There is one canonical locked copy per sample key, so re-loading a sample switches to and refreshes the existing one rather than piling up duplicates. The current app version is shown next to the logo in the nav.

---

## Production Chain Calculator

### Target Mode
The user specifies one or more target resources, each with a desired output rate (per second, minute, or hour), and the planner works backwards through the dependency graph to produce a complete bill of machines and raw material inputs.

Key behaviours:
- Uses the full dependency tree — if Iron Plates require Iron Ore, and Gears require Iron Plates, a target of Gears will surface all three stages.
- Accounts for the selected tier's speed multiplier when calculating machine counts.
- **Machine count is always ceiled** to a whole number. Input provisioning is then based on that ceiled count so every built machine is fully fed and runs at 100% capacity.
- Diamond dependencies (same resource demanded by multiple downstream stages) accumulate correctly — the tool does not double-count machines.
- Recipes are toggled **enabled/disabled** rather than manually assigned: the planner automatically picks, per resource, the cheapest enabled and unlocked recipe chain, measured in raw-equivalent inputs per unit of output (raw resources and mining recipes count 1 per unit; ties go to the default recipe).

### From-Inputs Mode
The user specifies a set of input resources and rates, and a primary target resource. The planner finds the maximum achievable output rate given the supply constraint, identifies the bottleneck resource, and shows each machine stage's efficiency (what fraction of full capacity it runs at).

Key behaviours:
- Machine counts are fractional/exact — machines are not ceiled because the constraint is the input supply, not a round-number target.
- Efficiency is displayed per stage so the user can see which machines are underutilised.
- Recipe selection prefers chains that terminate at the supplied inputs, and automatically falls back to alternate recipes that work with what's available — but a chain is never blocked just because part of it needs something the user didn't list. Only the resources actually supplied constrain the achievable rate.
- **Ingredients the chain needs beyond the supplied inputs are calculated automatically** at the optimum ratio (e.g. supplying only Steel Beams for Encased Industrial Beams computes how much Concrete — and the Limestone/Miners under it — is also required) rather than erroring. These are shown as full stages in the results and diagram like any other. A highlighted "extra requirements" banner additionally summarises just the resources at the **boundary** where the chain first needs something beyond supply: a recipe that combines something reachable from the given inputs with something that isn't (e.g. a Steel Ingot recipe mixing supplied Iron Ore with unsupplied Coal flags Coal, not the Steel Beam made from it two steps later), or, for a branch that never reconnects with the supply at all (e.g. Concrete's Limestone, when nothing supplied touches limestone), the point where that disconnected branch first attaches. This keeps the summary short even for deep chains while still naming the resource, its rate, and the recipe/machine producing it — flagging when that recipe is a non-default alternate so the user knows what to disable if they'd rather supply that ingredient directly instead.
- The plan only errors when the target itself is impossible to produce at all (research-locked or a dependency cycle) or when none of the supplied inputs are actually consumed by the chain.

### Multi-Target Planning (both modes)
Target mode and From-Inputs mode share one planning engine and are, beyond the primary target, **the same feature**: a priority-ordered list of targets where every target after the first can be planned entirely from the byproducts of the targets before it. The two modes differ only in how the *primary* (first) target's rate is determined — "different bottleneck termination":
- **Target mode**: the primary target has a fixed, user-specified rate; its machine count is ceiled.
- **From-Inputs mode**: the primary target's rate is derived from the user-supplied inputs (the achievable-rate/bottleneck computation described above); its machine counts are exact/fractional.

From that point on both modes behave identically. Secondary targets are entered as a list in **priority order**, one per row (Target mode's rows sit below the primary target; From-Inputs mode gets its own "Secondary targets" row list below its inputs list) — a fresh blank row appears automatically as soon as the current last row gets a resource. Every secondary target can either have its own fixed amount (planned independently, exactly like a primary target-mode target) or be left with **no amount**, meaning "make as much of this as possible from an earlier target's byproducts" — the classic case being: primary target = Fuel at a fixed rate (which yields Polymer Resin as a byproduct), secondary target = Plastic with no amount, computed entirely from that Polymer Resin via the "Alternate: Residual Plastic" recipe.

Processing is strictly sequential and priority governs the outcome: the primary target is planned first and its byproducts are added to a shared pool; each byproduct-driven secondary target then plans itself against whatever's currently in that pool — using the from-inputs machinery, including automatic extra-requirement calculation and alternate-recipe fallback for anything the byproduct alone doesn't cover — and draws down what it uses, so a target later in the list only sees the leftover; a fixed-amount secondary target adds its own byproducts to the pool the same way the primary target did. Swapping the order of two targets can change (or break) the outcome by design, since it changes what's in the pool when each one runs. A byproduct-driven target with nothing available in the pool yet shows its own inline error without blocking the other targets. Results, raw inputs, power total, and the flow diagram are combined across every target into one plan; **Build in Layout** places the whole thing together.

Any byproduct produced anywhere in the combined plan beyond what the plan itself consumes is reported as an **unused byproduct surplus** banner, computed quantitatively (total produced − total consumed per resource): a byproduct partly used in-chain shows only its true leftover, one fully consumed shows nothing, and a purely unused byproduct shows its full rate. This applies uniformly to a single-target plan (either mode) and to multi-target plans with leftover after every target has run.

### Flow Diagram
Results are displayed as a visual graph alongside the machine list. The diagram shows:
- One node per recipe stage, labelled with machine, recipe name, count, and actual output rate
- Columns aligned to the top of the diagram (matching the results list beside it)
- Each flow into or out of a node gets its own entry/exit point spread along the node's edge, so multiple flows never converge on one midpoint
- Every edge ends in a straight horizontal run so the line enters the back of its arrowhead
- Edges are **colour-coded per resource** (the same material keeps one colour across the whole diagram, including its raw-input box) so flows can be traced at a glance
- Each edge carries only a **compact rate pill** at its arrowhead (e.g. "360/min"); hovering a belt pops an **instant, styled tooltip card** showing the full details — resource (in its colour), exact rate, required belt/pipe tier, and producer → consumer. Rather than tracking the raw mouse, a **marker dot is placed at the nearest point along the belt to the cursor** and the card is anchored to that dot, so both slide smoothly down the belt as the mouse moves — cleaner, and it confirms exactly which belt is being examined. (Rate pills and byproduct boxes, which have no path, just anchor the card near themselves.) Every pill is rendered in its own layer above every line and node box in the diagram, so a crossing belt can never cover a label, regardless of draw order.
- When byproduct satellite boxes are present, the inter-column gap widens so the boxes have room to read instead of being squished against the next machine box.
- Raw input nodes at the far end of the graph
- Nodes grouped left-to-right by dependency depth (raw inputs on the left, target on the right, following natural reading direction)
- Nodes within each column sorted by category then name
- Edges that skip columns (long-range dependencies) routed via horizontal highway lanes above or below the node area so they don't overlap node boxes
- Category-based colour coding on node borders
- Efficiency bands on nodes in from-inputs mode
- **Byproducts** branch off their producing node as a small, visually distinct satellite box (smaller than a machine box) labelled with the resource and rate. If another stage in the same plan consumes that byproduct, a normal edge runs from the satellite box to it; if nothing in the plan claims it, the box is dashed like a raw-input box to flag it as unused surplus. When a resource is produced by **both** a primary stage and as a byproduct (e.g. Silica from a Quartz constructor and from Alumina Solution), each consumer's draw is split proportionally between the two sources, so the edge rates add up to what the stages actually make. Byproduct supply counts toward column placement just like a primary-output dependency, so a producer whose byproduct feeds another node is positioned upstream (left) of that consumer — the connection reads left-to-right like every other flow rather than looping backward, even when the producer is itself a target
- **Secondary-target banding** — when a plan has more than one target, only the first (primary) target's chain occupies the main row region. Every later target — a byproduct-driven one and its own extra requirements, or an additional fixed target — is laid out in a separate band beneath the main chain, inside a labelled dashed backdrop ("Byproduct & secondary targets"). This keeps the primary production line readable on its own while the secondary work (e.g. the Polymer Resin → Residual Plastic → Plastic chain fed by a Fuel target's byproduct) reads as a distinct group. A byproduct box that feeds the band drops into the band too, aligned with its consumer, with a dashed connector dropping from the producer up top — so the whole byproduct chain lives in one row region (byproduct boxes that are unclaimed, or that feed back into the main chain, still hang beside their producer). Nodes shared with the main chain stay up top. Single-target plans are unaffected (no band)

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
Belts and pipes are always drawn as clean orthogonal runs with 90° corners (paths are rasterised through their waypoints). They are **attached to their endpoint ports** by stable port ids: moving or rotating a machine re-routes every connected belt, and editing/reordering a machine's port list can't re-target belts (belts whose ports are deleted are removed with a warning). Belt paths are derived data and are never persisted — they're recomputed on load.

### Undo / feedback
The layout editor keeps a 50-step undo/redo history (Ctrl+Z / Ctrl+Y) covering placement, moves, rotation, deletion, belts, paste, blueprint load/place, and plan builds; the history survives tab switches and resets on profile switch. Actions that can't proceed (paste with no space, placing a blueprint whose machines were deleted) show a brief HUD message instead of failing silently. Holding Alt while dragging disables soft-snap.

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
- Left: mode switcher (target / from-inputs) with a **Clear all** button (resets every target and input in both modes to a fresh empty plan, so a many-target setup doesn't have to be cleared row by row), inputs for the chosen mode (both include a secondary-targets row list, per Multi-Target Planning above), a **Max Machine Tiers** card (tier selectors only for machines that actually have multiple unlocked tiers, plus a max-belt-tier cap), enable/disable checkboxes for resources with multiple recipes, and the machine result list
- Right: the flow diagram

### Layout Tab
Full-height canvas with a sidebar on the left containing the tool buttons, rotation control, machine palette, and build cost panel.

---

## Data Persistence

All data is stored locally in the browser across sessions. No account, server, or network connection is required. Data is never sent anywhere. If browser storage fills up, the app warns loudly instead of silently dropping changes. The planner's last query (mode, target, rate, inputs) is saved per profile and restored — with results recalculated — on the next visit, and results auto-recalculate when tiers, recipe toggles, or belt/pipe caps change. All user-entered names are HTML-escaped wherever they are displayed.

---

## Design Principles

- **Game-agnostic** — no game-specific logic; all data is user-defined
- **Single-page, no install** — opens directly in a browser, no build step or server
- **Profiles as the top-level organiser** — switching profiles feels like switching games
- **Planner seeds, layout decides** — planner results can be pushed to the layout editor as a starting arrangement via Build in Layout, but the layout never auto-syncs afterwards; users arrange machines and route belts freely
- **Visual but not simulation** — belts and ports are for planning/communication, not for running a simulation
- **Snap-to-grid always** — fractional positions are not a design goal; factory games use grids
