# PrinceJS Level JSON Format

This document specifies the JSON format used for PrinceJS level map files, located at
`assets/maps/levelN.json` (built-in levels) and `assets/maps/custom/levelN.json` (custom levels).
All values shown are real examples from `level1.json` unless otherwise noted.

---

## Top-level structure

```json
{
  "number": 1,
  "name": "Cell",
  "type": 0,
  "size": {
    "width": 9,
    "height": 3
  },
  "room": [ ... ],
  "prince": { ... },
  "guards": [ ... ],
  "events": [ ... ]
}
```

| Field    | Type    | Description                                                       |
|----------|---------|-------------------------------------------------------------------|
| `number` | integer | Level number (1–15 for built-in levels).                          |
| `name`   | string  | Human-readable level name shown in the UI.                        |
| `type`   | integer | Visual tileset type: `0` = dungeon, `1` = palace (see below).    |
| `size`   | object  | Grid dimensions: `{ "width": N, "height": N }`.                  |
| `room`   | array   | Flat room grid, `width × height` entries, row-major (see below). |
| `prince` | object  | Starting position and options for the Prince (see below).         |
| `guards` | array   | List of enemy/NPC spawn descriptors (see below).                  |
| `events` | array   | Ordered list of trigger→gate event records (see below).           |

---

## `type` values

| Value | Constant              | Description                                |
|-------|-----------------------|--------------------------------------------|
| `0`   | `Level.TYPE_DUNGEON`  | Stone dungeon tileset (levels 1–8).        |
| `1`   | `Level.TYPE_PALACE`   | Palace tileset with lattice/tapestry tiles.|

---

## `size` object

```json
"size": { "width": 9, "height": 3 }
```

The grid has `width` columns and `height` rows. Level 1 is a 9 × 3 grid = 27 cells. The `room`
array always has exactly `width × height` entries.

---

## `room` array

The `room` array is a flat, row-major list of room cells. Index `y * width + x` gives the cell at
grid column `x`, row `y`.

### Empty cell

A cell with no room uses only the `id` field set to `-1`:

```json
{ "id": -1 }
```

### Populated cell

A cell that contains a room uses an `id` ≥ 1 and a `tile` array of exactly 30 objects:

```json
{
  "id": 1,
  "tile": [
    { "element": 0, "modifier": 0 },
    ...
  ]
}
```

| Field    | Type    | Description                                               |
|----------|---------|-----------------------------------------------------------|
| `id`     | integer | Room identifier used throughout events/prince/guards. `-1` = empty cell. |
| `tile`   | array   | 30 tile objects `{element, modifier}` for this room.     |

### Tile array layout

Each room has a 10-column × 3-row grid of tiles. The `tile` array is stored row-major:

- Index 0–9: top row (row 0)
- Index 10–19: middle row (row 1)
- Index 20–29: bottom row (row 2)

Index formula: `index = row * 10 + col`

Each tile object:

```json
{ "element": 1, "modifier": 0 }
```

| Field      | Type    | Description                                                                   |
|------------|---------|-------------------------------------------------------------------------------|
| `element`  | integer | Tile type constant (see tile element table below).                            |
| `modifier` | integer | Tile variant or data (meaning depends on element — e.g. event index for buttons, potion type for potions, torch variant for torches). |

### Tile element constants

These match the `PrinceJS.Level.TILE_*` constants defined in `Level.js`:

| Value | Constant                 | Description                                       |
|-------|--------------------------|---------------------------------------------------|
| `0`   | `TILE_SPACE`             | Empty space (open air, no floor).                 |
| `1`   | `TILE_FLOOR`             | Solid floor tile.                                 |
| `2`   | `TILE_SPIKES`            | Spike trap.                                       |
| `3`   | `TILE_PILLAR`            | Pillar (blocks movement).                         |
| `4`   | `TILE_GATE`              | Gate / portcullis.                                |
| `5`   | `TILE_STUCK_BUTTON`      | Button that is stuck (already pressed).           |
| `6`   | `TILE_DROP_BUTTON`       | Pressure plate that drops (closes) a gate.        |
| `7`   | `TILE_TAPESTRY`          | Palace tapestry hanging (lower half).             |
| `8`   | `TILE_BOTTOM_BIG_PILLAR` | Bottom half of a large pillar.                    |
| `9`   | `TILE_TOP_BIG_PILLAR`    | Top half of a large pillar.                       |
| `10`  | `TILE_POTION`            | Potion pickup.                                    |
| `11`  | `TILE_LOOSE_BOARD`       | Loose floor board that falls when stepped on.     |
| `12`  | `TILE_TAPESTRY_TOP`      | Palace tapestry hanging (upper half).             |
| `13`  | `TILE_MIRROR`            | Mirror tile (level 4 special).                    |
| `14`  | `TILE_DEBRIS`            | Rubble / debris on floor.                         |
| `15`  | `TILE_RAISE_BUTTON`      | Pressure plate that raises (opens) a gate.        |
| `16`  | `TILE_EXIT_LEFT`         | Left half of level exit door.                     |
| `17`  | `TILE_EXIT_RIGHT`        | Right half of level exit door (triggers exit).    |
| `18`  | `TILE_CHOPPER`           | Guillotine / chopper blade.                       |
| `19`  | `TILE_TORCH`             | Torch (decorative, animates).                     |
| `20`  | `TILE_WALL`              | Solid wall block.                                 |
| `21`  | `TILE_SKELETON`          | Skeleton remains on floor.                        |
| `22`  | `TILE_SWORD`             | Sword pickup.                                     |
| `23`  | `TILE_BALCONY_LEFT`      | Palace balcony railing, left side.                |
| `24`  | `TILE_BALCONY_RIGHT`     | Palace balcony railing, right side.               |
| `25`  | `TILE_LATTICE_PILLAR`    | Palace lattice pillar.                            |
| `26`  | `TILE_LATTICE_SUPPORT`   | Palace lattice support beam.                      |
| `27`  | `TILE_SMALL_LATTICE`     | Palace small lattice panel.                       |
| `28`  | `TILE_LATTICE_LEFT`      | Palace lattice, left edge.                        |
| `29`  | `TILE_LATTICE_RIGHT`     | Palace lattice, right edge.                       |
| `30`  | `TILE_TORCH_WITH_DEBRIS` | Torch combined with debris on floor.              |
| `31`  | `TILE_DEBRIS_ONLY`       | Debris only (no torch).                           |
| `32`  | `TILE_NULL`              | Null / placeholder tile.                          |

> **Note:** The spec instructions list different numeric assignments for elements 7–31 — the table
> above reflects the actual assignments in `Level.js` and in the JSON data.

### Room 1 tile array (level 1 starting room)

Room id=1 sits at grid position col=6, row=0. The Prince starts here.

```
Row 0 (top):    [0,0]  [0,0]  [0,0]  [1,0]  [1,1]  [1,0]  [1,1]  [1,0]  [20,1] [20,0]
Row 1 (middle): [19,0] [19,0] [1,1]  [3,0]  [0,0]  [20,1] [20,0] [20,0] [20,0] [20,0]
Row 2 (bottom): [20,0] [20,0] [20,0] [20,0] [14,0] [3,0]  [11,0] [1,1]  [1,0]  [20,1]
```

Format: `[element, modifier]`

### Room 22 tile array (level 1, grid col=0 row=0)

```
Row 0 (top):    [20,0] [0,0]  [0,0]  [0,0]  [0,0]  [3,0]  [10,1] [19,0] [1,1]  [1,0]
Row 1 (middle): [3,0]  [0,0]  [0,1]  [0,0]  [0,0]  [20,0] [20,0] [20,0] [20,0] [20,0]
Row 2 (bottom): [3,0]  [0,0]  [0,1]  [0,0]  [0,0]  [20,0] [20,0] [20,0] [20,0] [20,0]
```

### Room 5 tile array (contains a gate and buttons)

Room id=5 sits at grid position col=5, row=0.

```
Row 0 (top):    [3,0]  [19,0] [6,11] [19,0] [15,9] [4,0]  [15,8] [0,1]  [1,0]  [4,1]
Row 1 (middle): [20,0] [20,0] [20,0] [20,0] [20,0] [20,0] [3,0]  [0,0]  [20,0] [20,0]
Row 2 (bottom): [20,0] [20,0] [19,0] [10,1] [1,1]  [11,0] [3,0]  [14,0] [20,0] [20,0]
```

Notable tiles in room 5:
- `[6, 11]` at row 0, col 2: `TILE_DROP_BUTTON` with modifier=11 (event index 11, 0-based: fires event 12)
- `[15, 9]` at row 0, col 4: `TILE_RAISE_BUTTON` with modifier=9 (event index 9, 0-based: fires event 10)
- `[4, 0]`  at row 0, col 5: `TILE_GATE`
- `[15, 8]` at row 0, col 6: `TILE_RAISE_BUTTON` with modifier=8 (event index 8, 0-based: fires event 9)
- `[4, 1]`  at row 0, col 9: `TILE_GATE`

---

## `prince` object

Describes where and how the Prince spawns at the start of the level.

```json
"prince": {
  "room": 1,
  "location": 1,
  "direction": 1,
  "turn": false,
  "offset": -7
}
```

### Required fields

| Field       | Type    | Description                                                                  |
|-------------|---------|------------------------------------------------------------------------------|
| `room`      | integer | Room id where the Prince spawns.                                             |
| `location`  | integer | Tile index (0–29) within the starting room: `row * 10 + col`.               |
| `direction` | integer | Initial facing: `1` = right, `-1` = left.                                   |

### Optional fields

| Field          | Type    | Default | Description                                                                               |
|----------------|---------|---------|-------------------------------------------------------------------------------------------|
| `turn`         | boolean | `true`  | If `true`, the Prince spawns facing direction then immediately executes a turn animation. The effective direction becomes `-direction`. If `false`, no turn is performed. |
| `offset`       | integer | `0`     | Pixel offset added to the Prince's initial X position after placement. Level 1 uses `-7`. |
| `bias`         | integer | `0`     | Added directly to `location` before placement (coarser X adjustment in tile units).       |
| `reverse`      | integer | `1`     | Multiplied with `direction`: `-1` flips the facing without triggering a turn animation.   |
| `sword`        | boolean | (unset) | If `true`, the Prince starts with a sword drawn; if `false`, starts without one. When absent the default game sword state applies. |
| `cameraRoom`   | integer | (unset) | If set, the camera starts focused on this room instead of `room`. A value of `0` means no camera movement. |
| `specialEvents` | boolean | `true` | If `false`, level-specific scripted events (e.g. the shadow, mirror scripting) are disabled. |
| `danger`       | boolean | `true`  | If `false`, the danger music cue on level 1 is suppressed.                                |

### Level 1 prince values

```json
"prince": {
  "location": 1,
  "offset": -7,
  "turn": false,
  "room": 1,
  "direction": 1
}
```

- Spawns in room 1, tile index 1 (top row, column 1).
- Faces right (`direction: 1`); `turn: false` so no turn animation.
- Pixel offset of `-7` shifts the Prince 7 px left from the tile centre.

### Location encoding

```
location = row * 10 + col
```

| location | row | col | Description             |
|----------|-----|-----|-------------------------|
| 0        | 0   | 0   | Top-left tile           |
| 1        | 0   | 1   | Top row, second column  |
| 9        | 0   | 9   | Top row, rightmost      |
| 10       | 1   | 0   | Middle row, leftmost    |
| 29       | 2   | 9   | Bottom-right tile       |

---

## `guards` array

Each element describes one enemy or NPC. Level 1 has two guards:

```json
"guards": [
  {
    "room": 21,
    "location": 7,
    "skill": 0,
    "colors": 2,
    "type": "guard",
    "direction": -1
  },
  {
    "room": 3,
    "location": 18,
    "skill": 0,
    "colors": 2,
    "type": "guard",
    "direction": -1
  }
]
```

### Required fields

| Field       | Type    | Description                                                                 |
|-------------|---------|-----------------------------------------------------------------------------|
| `room`      | integer | Room id where the guard spawns.                                             |
| `location`  | integer | Tile index (0–29) within the room: `row * 10 + col`.                       |
| `direction` | integer | Initial facing: `1` = right, `-1` = left.                                  |
| `skill`     | integer | Combat skill level (0 = weakest). Controls attack/block timing.             |
| `colors`    | integer | Palette index selecting the guard's colour scheme.                          |
| `type`      | string  | Character type (see type strings below).                                    |

### Optional fields

| Field     | Type    | Default | Description                                                                               |
|-----------|---------|---------|-------------------------------------------------------------------------------------------|
| `bias`    | integer | `0`     | Added to `location` for fine X positioning (same semantics as `prince.bias`).             |
| `reverse` | integer | `1`     | Multiplied with `direction`: `-1` flips the facing direction.                             |
| `visible` | boolean | `true`  | If `false`, the guard spawns invisible (used for scripted shadow appearances).            |
| `active`  | boolean | `true`  | If `false`, the guard spawns inactive (will not patrol or attack until activated).        |
| `sneak`   | boolean | `true`  | If `false`, the guard does not attempt to sneak up on the Prince.                         |

### Guard type strings

| Value       | Description                                               |
|-------------|-----------------------------------------------------------|
| `"guard"`   | Standard sword guard (most common enemy).                 |
| `"fat"`     | Fat guard (more health, slower).                          |
| `"skeleton"`| Skeleton (cannot be killed by sword alone).               |
| `"shadow"`  | The Prince's shadow (scripted ally/antagonist).           |
| `"princess"`| The Princess (non-combat NPC).                            |
| `"jaffar"`  | Jaffar (final boss, level 13).                            |
| `"mouse"`   | Mouse (non-combat NPC, level 8 scripted scene).           |

### Guard spawn index

Guards are indexed 1-based in the order they appear in the array (the first guard is enemy 1, the
second is enemy 2, etc.). This index is passed to `PrinceJS.Enemy` as its `index` parameter.

---

## `events` array

Events are the trigger–target records that link buttons to gates. When a button tile is stepped on,
it fires the event whose index matches the button's `modifier` value (0-based).

Each event object:

```json
{
  "number": 1,
  "room": 12,
  "location": 10,
  "next": 0
}
```

| Field      | Type    | Description                                                                           |
|------------|---------|---------------------------------------------------------------------------------------|
| `number`   | integer | 1-based sequence number (informational; the array index is used by the engine).       |
| `room`     | integer | Room id containing the gate tile to be triggered.                                     |
| `location` | integer | Tile index (0–29) of the gate within that room.                                       |
| `next`     | integer | Chain flag: `1` (truthy) = also trigger event at the next array index; `0` = terminal.|

### How LevelBuilder loads events

```js
this.level.events = json.events;
```

Events are stored as-is on the `Level` object. The engine indexes into this array using 0-based
offsets derived from a button tile's `modifier` value.

### Level 1 events

```json
"events": [
  { "number":  1, "room": 12, "location": 10, "next": 0 },
  { "number":  2, "room": 12, "location": 10, "next": 0 },
  { "number":  3, "room": 12, "location": 10, "next": 0 },
  { "number":  4, "room":  9, "location": 14, "next": 0 },
  { "number":  5, "room":  7, "location": 10, "next": 0 },
  { "number":  6, "room":  8, "location": 10, "next": 0 },
  { "number":  7, "room":  7, "location": 10, "next": 0 },
  { "number":  8, "room":  8, "location": 10, "next": 0 },
  { "number":  9, "room":  5, "location": 10, "next": 0 },
  { "number": 10, "room":  5, "location": 10, "next": 1 },
  { "number": 11, "room":  5, "location":  6, "next": 0 },
  { "number": 12, "room":  5, "location": 10, "next": 0 }
]
```

Event 10 (`next: 1`) chains to event 11 — triggering event index 9 (0-based) fires both room 5
tile 10 and room 5 tile 6 in sequence.

---

## Grid layout for level 1

The `room` array for level 1 has `size.width=9` columns and `size.height=3` rows, giving 27 cells.
The table below shows the room id at each grid position (`-1` = empty cell):

```
col:   0    1    2    3    4    5    6    7    8
row 0: 22   16   23   17   21    5    1   -1   -1
row 1: 15   12   20    7    8    6    2    3    9
row 2: 10   19    4   14   11   -1   -1   -1   -1
```

The Prince starts in room 1 (row 0, col 6). The two guards are in rooms 21 (row 0, col 4) and 3
(row 1, col 7).

### Room adjacency (derived by LevelBuilder)

After loading, each room's `links` object is computed from the grid:

| Direction | Source room neighbour |
|-----------|-----------------------|
| `left`    | `layout[y][x-1]`      |
| `right`   | `layout[y][x+1]`      |
| `up`      | `layout[y-1][x]`      |
| `down`    | `layout[y+1][x]`      |

A link value of `-1` means no adjacent room (wall boundary). Out-of-bounds grid positions also
yield `-1`.

---

## Coordinate conventions

### Room pixel dimensions

| Measure         | Value      |
|-----------------|------------|
| Room width      | 320 px logical (`WORLD_WIDTH`) |
| Room height     | 200 px logical (`WORLD_HEIGHT` / `ROOM_HEIGHT`) |
| Tile block width  | 32 px (`BLOCK_WIDTH`) |
| Tile block height | 63 px (`BLOCK_HEIGHT`) |
| Columns per room  | 10 |
| Rows per room     | 3  |

### Tile position formula

Given tile at `(col, row)` in room at grid `(roomX, roomY)`:

```
pixel_x = roomX * ROOM_WIDTH  + col * BLOCK_WIDTH
pixel_y = roomY * ROOM_HEIGHT + row * BLOCK_HEIGHT - 13
```

The `-13` offset (applied in `Level.addTile`) accounts for the tile sprite anchor offset.

### Location index encoding (summary)

```
location = row * 10 + col    (row ∈ {0,1,2}, col ∈ {0..9})
```

This applies identically to `prince.location`, `guard.location`, and `event.location`.
