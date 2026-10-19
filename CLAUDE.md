# LED Calculators

Planning tools for addressable LED builds (WLED on ESP32). Each tool is a single self-contained HTML file that opens straight from disk in a browser. There is no build step, no package manager and no server.

## Files

- `configs/`: saved configurations, one subfolder per tool (see Conventions). The JSON files are git-ignored.

- `LED Matrix Planner.html`: plans either a 2D strip matrix or a straight strip run. It covers LED count, strip to buy, power supply size, power injection points, wire gauge, voltage drop, a canvas wiring preview, and the values to type into WLED.
- `Hexaleaf Planner.html`: a Nanoleaf-style layout of 3D-printed hex tiles. Click to add or remove tiles on a hex grid, give each tile a depth (25/30/35/40 mm wall plates), and set which tile takes data in. A Measure tool gives the distance between any two points, and a Shapes tool places custom obstacles (wall plugs, switches) to plan around. It covers the print list (walls by depth, lids, Support 1/2/3), LEDs per tile, strip length, power supply, injection feeds, the data path between tiles and per-tile WLED segments.

## Conventions

- Keep each tool in one file: inline CSS and JS, no frameworks. The only external resource is Google Fonts, and every face has a system fallback, so the page works offline.
- Colors live as CSS custom properties on `:root`, with a dark palette under `prefers-color-scheme: dark` and `[data-theme="dark"]`. Use the tokens and never hard-code a color in a rule. The canvas reads the tokens with `getComputedStyle`.
- Inputs are saved to `localStorage`, wrapped in try/catch, under a key for each tool: `ledplanner` for the matrix planner and `hexplanner` for the hexaleaf planner (it also saves the tile layout). Each tool keeps an `ids` array, and a new input needs its id added to it so it is saved and restored.
- Saved configurations: both tools save named setups as JSON files in `configs/<app>/` (`configs/matrix/`, `configs/hexaleaf/`).
  - The code is `setupConfigs(app, getState, applyState, afterLoad)`, and an identical copy sits in each file.
  - Chrome and Edge write to the folder directly through the File System Access API. The user picks `configs` (or the tools folder, and `configs` is made inside it) once.
  - The folder handle is kept in IndexedDB (`ledcalc` / `handles` / `configsDir`), shared by both tools. After a browser restart it may ask to reconnect.
  - Other browsers fall back to downloading the file and loading it with a file picker.
  - A file looks like `{ app, version: 1, name, saved, state }`, where `state` is the same object the tool keeps in `localStorage`.
  - Overwriting and deleting take a second click (an armed button) instead of `confirm()`.
  - Each tool's `getState()` and `applyState()` are the single place its saved state is defined, so a new input only needs adding to `ids`.
- Visibility is toggled with the `hidden` attribute. `[hidden] { display: none !important; }` is in the stylesheet, so keep it.
- Keep the layout working at phone width (about 400px).
- The Hexaleaf Planner fills the whole window width (no max-width). The measurements and shapes sidebar is 290 px wide. From 1500 px wide, the result panels flow into columns (`auto-fill, minmax(420px, 1fr)`) under the full-width layout panel and stats row. The canvas grows with the window: tiles up to 110 px across, and a canvas up to 72% of the window height.

## How the planner works

There are two build modes, `build = "matrix" | "strip"`, switched by `setBuild()`. `applyVis()` shows and hides the inputs for each mode. `update()` handles matrix mode and hands strip mode to `updateStrip()`.

### Strip types (`chips` object)

| Key | Voltage | Default mA/LED (full white) | Default max LEDs per feed | WLED LED type |
|---|---|---|---|---|
| `ws2812b` | 5V | 60 | 150 | WS281x |
| `sk6812` | 5V | 80 (RGBW) | 120 | SK6812/WS2814 RGBW |
| `sk6812_12` | 12V | 20 (RGBW) | 300 | SK6812/WS2814 RGBW |
| `ws2815` | 12V | 15 | 400 | WS281x (WS2815) |

These are typical figures. The real value is `mA per pixel = W/m ÷ V ÷ pixels per m × 1000`, taken from the strip's listing. Many 12V strips group 3 LEDs per IC, so the density to enter is pixels per metre, not LEDs per metre.

### Power rules

- Supply current = full-white current × brightness limit × 1.2 headroom, rounded up to a common supply size (`roundPsu`).
- The suggested WLED current limit is 85% of the supply rating.
- Wire gauge for each feed comes from `awg()`. Voltage drop uses `ohmPerM` (ohms per metre for one conductor) × 2 for the round trip.
- Data refresh is estimated at about 30 µs per LED plus a 300 µs reset. The planner suggests one output per 800 LEDs.

### Injection rules (these were deliberate requests; keep them)

- **Matrix:** every feed must land on the **same edge** as LED 0, so a single power bus runs down one side. Serpentine wiring returns to that edge every 2 runs and straight wiring every run, so the spacing is rounded **down** to a multiple of that period. If even the minimum period exceeds the requested maximum, the planner warns instead of breaking the rule.
- **Strip:** feeds go at the start and then every N LEDs. Each feed powers the N LEDs after it. There is no automatic extra feed at the far end; an earlier "add the far end if the leftover is over half the interval" rule over-counted and was removed.

## Hardware context

- Intended controller: GLEDOPTO Elite 4-channel ESP32 WLED controller (Ethernet). It takes 5–24V and handles **15A total** with **10A per output**. The 20A figure is the fuse rating, not a continuous load. It is advertised for about 1,600 ICs.
- For larger builds, feed injection points straight from the supply so the controller only carries data and the first section's power.
- Reference build: Chris Maher's 50×50 (2,500 LED) panel. It uses 12V SK6812 at 30 LEDs/m in a vertical serpentine, split across 4 data outputs, with one controller and one supply.

## WLED setup notes

- Config → LED Preferences: length, LED type, color order (GRB for these chips), max current, outputs.
- Config → 2D Configuration: for a matrix, choose 2D Matrix and set width, height, first-LED corner, orientation and serpentine. For a straight strip, leave the default 1D setting.
- If the build also uses xLights, set up the xLights models before WLED's 2D configuration.

## How the hexaleaf planner works

- Tiles are stored in axial hex coordinates (`q,r`), with geometry computed for flat-top hexes. Point-top is the same layout rotated 90° (`rot()`).
- The strip runs around all 6 inner walls, which have rounded corners. There are two ways to fit it (`wrap`):
  - `loop` (default): one piece bent round the corners. LEDs per tile = floor((6 × straight wall + 2π × corner radius) ÷ pitch).
  - `sides`: 6 pieces, cut per side. LEDs per tile = 6 × floor(straight wall ÷ pitch).
- Size inputs: `inner` is the straight part of one inside wall and `outer` is the outer side length (flat to flat = side × √3). Saved state carries `v: 2`; size values saved before v2 meant something else and are ignored on load.
- "Randomize depths" (under the depth picker) gives each tile a random depth, visiting tiles in random order. It picks a depth none of its already-assigned neighbours has where possible, so adjacent tiles differ. One level of Undo restores the depths from before the first randomize in a run.
- Parts and connectors:
  - Support 3 (edge joint): one per shared edge.
  - Support 2 (3-way corner): one per corner where 3 tiles meet.
  - Support 1 (wall mount): tiles × the "wall mounts per tile" input.
  - Lids: one per tile.
- Data path (`solveChain`): a branch-and-bound search over tile orders from the data-in tile.
  - It minimises the number of jump wires first, then their total length (cost = 1000 per jump + centre-to-centre distance).
  - It steps to neighbours in Warnsdorff order and only jumps when stuck, trying the 5 nearest unvisited tiles plus up to 3 chain ends.
  - Bounds:
    - Every unvisited region the chain can't reach needs a jump.
    - Dead-end bound: an unvisited tile with k < 2 unvisited neighbours needs 2 − k chain ends, each further chain supplies 2, and the current chain supplies 1.
    - When a jump is due, the nearest unvisited tile's distance is added.
  - A memo of (current tile, visited set) skips states already reached at no higher cost.
  - The node budget is 150k. The result is cached by the layout and start, so editing other inputs doesn't re-solve.
  - "Best data-in tile" only tries dead-end tiles (plus the current start) when the layout has any, at up to 40k nodes each, since every chain end must sit on a dead end. Otherwise it tries every tile.
  - `anyOrderWithin(K)` is an exact feasibility search: any start, chains may end anywhere, same bound, memoised, budget 250k. `exactMinJumps` uses it to tell the user when the jump count is the true minimum for the shape.
  - Jump wires are listed in "Strip and wiring" with centre-to-centre lengths.
  - BehindMonitor config (47 tiles, 8 dead ends): the old version gave 5 jumps and about 2.2 m of jump wire. The current one gives 4 jumps and about 1.0 m, and the exact search proves 4 is the minimum.
  - Over 300 random layouts the search cut jumps by 42% from the same start and by 77% with best start.
  - The "Data path" entry in the legend is a toggle (`pathShow`, saved with the state and configs) that hides the path arrows on the canvas. The tile numbers stay.
- Power injection lands at the start of a tile: every floor(inject spacing ÷ LEDs per tile) tiles, at least 1. Supply sizing and the 85% WLED limit follow the same power rules as the matrix planner.
- Measure tool (`tool = "measure"`):
  - Click two points to measure between them. Clicks snap to tile corners, edge midpoints and centres within about 16 px; anywhere else gives a free point.
  - It reports the straight-line distance plus its horizontal (↔) and vertical (↕) parts in mm, in screen orientation, so point-top layouts swap them.
  - Measurements scale by the outer side length (the circumradius) and assume tiles butt together with no gap.
  - They are stored as `measures: [{ a, b, legs }]` in unrotated unit coordinates and are saved with the state and configs. Older saves stored them as `[[ax, ay], [bx, by]]`, and `applyState` still reads that form.
  - When a measurement's `legs` flag is on, it draws the full dashed rectangle, so both right triangles show. Both horizontals are labelled ↔ and both verticals ↕, with right-angle marks at the two corners off the hypotenuse. The sidebar row also shows the angle from level.
  - A Measurements sidebar sits to the right of the canvas (below it under 1100 px). Each row has a Legs toggle and a ✕.
  - Below the list are "All legs on/off", Undo last and Clear all. A Shown/Hidden toggle in the header (`measShow`, saved) hides every measurement on the canvas.
  - Hovering or focusing a row highlights that measurement on the canvas (`mHi`) with its legs, even while all are hidden. Highlights use the violet `--hl` / `--hl-soft` tokens, never the amber `--accent`, so they stand out from the tiles. The same goes for the pending point, the snap cursor and highlighted shapes.
- Shapes tool (`tool = "shape"`): custom obstacles such as a wall plug.
  - Each shape is `{ name, kind: "rect" | "circle", w, h, x, y }` in mm, where x, y is the centre in screen orientation measured from tile (0,0)'s centre. They are stored as `shapes` and saved with the state and configs.
  - The form (`shName`, `shKind`, `shW`, `shH`) sets the next shape. It defaults to a 70 × 115 mm wall plug, a North American single-gang cover plate.
  - Clicking empty canvas places a shape, and dragging moves one. The view is held still while dragging so the shape tracks the pointer.
  - Shapes are drawn translucent over the tiles. They turn red, with "Overlaps tile #n" in the sidebar, when they intersect a tile (separating-axis test against the hex polygons).
  - Shape corners, edge midpoints and centres are snap points for the Measure tool.
  - Shapes are physical sizes in mm, so changing the outer side length rescales the tiles but not the shapes. "Clear all" in Edit layout keeps the shapes.
- Filament and print time:
  - Each part has slicer figures (`g_<part>`, `t_<part>`: grams and model minutes for one part). They sit in the "Per-part slicer figures" panel and are saved with configs.
  - Lids and walls print one per plate, so each gets model time plus the "setup per plate" overhead (default 6 min, Bambu's prepare time).
  - Supports share plates, so they count model time only, as the user asked.
  - Cost = grams × price per kg. Metres are estimated at 3.175 g/m (1.75 mm PLA, from the slicer figures).
  - Default figures from Bambu Studio (model weight, model time):

    | Part | Grams | Model time |
    |---|---|---|
    | Lid | 14.73 g | 37m39s |
    | 25 mm wall | 41.19 g | 1h10m |
    | 30 mm wall | 48.86 g | 1h23m |
    | 35 mm wall | 56.53 g | 1h36m |
    | 40 mm wall | 64.20 g | 1h49m |
    | Support 1 | 6.93 g | 17m42s |
    | Support 2 | 2.61 g | 9m21s |
    | Support 3 | 3.25 g | 10m46s |

  - The slicer's cost works out to about $20/kg.
- Measured tile, from the slicer:
  - Outer side 65.08 mm, so 130.16 mm corner to corner and about 112.7 mm flat to flat.
  - Straight inner wall 53.7 mm, with 51.4 mm on the inner lip.
  - 109.8 mm between the left and right corner pockets.
  - The corner radius of about 7 mm is estimated from a screenshot, not measured.
  - At 60 px/m that gives 21 LEDs per tile as one loop, or 18 cut per side.

## Version control

- The folder is a git repo on branch `main`. Commit after each change to a tool.
- `.gitattributes` keeps text files LF. `core.autocrlf` and `core.filemode` are off for this repo, so the Windows and OneDrive sides do not show false changes.
- Saved configs in `configs/` are git-ignored. Only the folder structure is tracked, through `.gitkeep` files in `configs/`, `configs/matrix/` and `configs/hexaleaf/`.
- Remote: `origin` = https://github.com/benaminc/led-calculator (public). The computer has no GitHub credentials, so Claude pushes from its cloud copy: commit here, `git bundle create .git/led-<sha>.bundle origin/main..main` (a fresh file name each time, since the file bridge can serve stale copies of a reused name), copy the bundle to the cloud, fetch it into the clone there and push, then delete the bundle.
