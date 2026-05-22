# LevelBuilder.js — Spec

**File:** `src/LevelBuilder.js` (362 lines)  
**Purpose:** Parses level JSON and constructs a `PrinceJS.Level` with all rooms, tiles, links, events. Returns the fully built Level.

---

## Constructor: `PrinceJS.LevelBuilder(game, delegate)`

```
game      — Phaser.Game
delegate  — Game state (receives onPushed, onChopped, onStartFalling etc.)
layout    = []    // 2D grid [y][x] → room id (-1 = empty)
wallColor = ["#D8A858","#E0A45C","#E0A860","#D8A054","#E0A45C","#D8A458","#E0A858","#D8A860"]
wallPattern = []  // per-room procedural colour indices for palace walls
seed      — PRNG state for wall pattern generation
level     — PrinceJS.Level instance (set during buildFromJSON)
```

---

## buildFromJSON(json) → PrinceJS.Level

**Step 1 — World bounds:**
```javascript
width  = json.size.width
height = json.size.height
type   = json.type
game.world.setBounds(0, 0, WORLD_WIDTH * width, WORLD_HEIGHT * height)
```

**Step 2 — Level object:**
```javascript
level = new PrinceJS.Level(game, json.number, json.name, type)
level.delegate = delegate
```

**Step 3 — Room grid pass 1 (build room metadata):**
```javascript
for y in 0..height-1:
    for x in 0..width-1:
        index = y * width + x
        id = json.room[index].id
        layout[y][x] = id
        if id !== -1:
            if type === TYPE_PALACE: generateWallPattern(id)
            level.rooms[id] = {
                x, y,
                links: {},
                tiles: json.room[index].tile   // raw tile data array
            }
```

**Step 4 — Room grid pass 2 (build tiles, iterating bottom-to-top):**
```javascript
for y from height-1 to 0:
    for x from 0 to width-1:
        id = layout[y][x]
        if id === -1: continue

        // Set room links:
        links.left  = getRoomId(x-1, y)
        links.right = getRoomId(x+1, y)
        links.up    = getRoomId(x, y-1)
        links.down  = getRoomId(x, y+1)

        // Add left border walls if no left room:
        if links.left <= 0:
            for jj in 2..0:
                tile = new Tile.Base(game, TILE_WALL, 0, type)
                tile.back.frameName = key + "_wall_0"
                level.addTile(-1, jj, id, tile)

        buildRoom(id, startRoomId, startLocation)

        // Add top border floors if no up room:
        if links.up <= 0:
            for ii in 0..9:
                tile = new Tile.Base(game, TILE_FLOOR, 0, type)
                level.addTile(ii, -1, id, tile)
```

**Step 5 — Events:**
```javascript
level.events = json.events
return level
```

---

## buildRoom(id, startId, startLocation)

```javascript
for y from 2 to 0:
    for x from 0 to 9:
        tile = buildTile(x, y, id, startId, startLocation)
        level.addTile(x, y, id, tile)
```

---

## buildTile(x, y, id, startId, startLocation) → tile

Reads `t = rooms[id].tiles[y*10+x]` (raw `{ element, modifier }` object).

| `t.element` | Tile class created | TROB registered | Notes |
|-------------|-------------------|-----------------|-------|
| TILE_WALL | Tile.Base | No | Wall frameName set from adjacency pattern; palace gets procedural BitmapData |
| TILE_SPACE / TILE_FLOOR | Tile.Base | No | `back` child sprite uses modifier variant frame |
| TILE_STUCK_BUTTON / TILE_RAISE_BUTTON / TILE_DROP_BUTTON | Tile.Button | Yes | `onPushed → delegate.fireEvent` |
| TILE_TORCH / TILE_TORCH_WITH_DEBRIS | Tile.Torch | Yes | |
| TILE_POTION | Tile.Potion | Yes | Special potion: reads room 8, pos (0,0) for `specialModifier`; `onDrank → delegate.fireEvent` |
| TILE_SWORD | Tile.Sword | Yes | |
| TILE_EXIT_RIGHT | Tile.ExitDoor | Yes | `open = (room === startRoom && abs(tileNum - startLocation) <= 1)`; if open: drop() after 200ms |
| TILE_CHOPPER | Tile.Chopper | Yes | `onChopped → level.activateChopper` |
| TILE_SPIKES | Tile.Spikes | Yes (only if modifier===0) | modifier 0 = retracted (needs tick updates) |
| TILE_LOOSE_BOARD | Tile.Loose | Yes | `onStartFalling → delegate.floorStartFall`; `onStopFalling → delegate.floorStopFall` |
| TILE_SKELETON | Tile.Skeleton | Yes | |
| TILE_MIRROR | Tile.Mirror | Yes | |
| TILE_GATE | Tile.Gate | Yes | `onFastDrop → delegate.checkGateFastDropped`; `setCanMute(false)` if `t.mute === false` |
| TILE_TAPESTRY | Tile.Base | No | Palace variant: modifier>0 uses modifier-specific frame |
| TILE_TAPESTRY_TOP | Tile.Base | No | Palace: modifier>0 frame; if left tile is LATTICE_SUPPORT adds small lattice child |
| TILE_BALCONY_RIGHT | Tile.Base | No | Palace: adds balcony child sprite at (0,-4) |
| TILE_BOTTOM_BIG_PILLAR | Tile.Base | No | If no TOP_BIG_PILLAR above: append "_low" to frameName |
| All others | Tile.Base | No | |

---

### Wall tile detail

**Dungeon:** `front.frameName = wallType + "_" + tileSeed`

`wallType` is a 3-char string: `[W|S]W[W|S]`
- char 0: W if left tile is WALL, else S
- char 1: always W (this tile)
- char 2: W if right tile is WALL, else S

`tileSeed = tileNumber + id`

**Palace:** Draws procedural BitmapData (60×79px) using `wallPattern` colour table (8 colour bands), then overlays `"W_" + tileSeed` sprite on top.

If `wallType[2] === 'S'` (right edge wall): `back.frameName = key + "_wall_" + modifier`

---

## getTileObjectAt(x, y, id)

Returns the raw tile object at `(x, y)` in room `id`, resolving cross-room coordinates. Returns `null` if outside the world.

## getTileAt(x, y, id)

Returns `tile.element` (integer), or `TILE_WALL` if outside world.

---

## getRoomId(x, y)

```javascript
if x < 0 || x >= width || y < 0 || y >= height: return -1
return layout[y][x]
```

---

## generateWallPattern(room)

Fills `wallPattern[room][]` (array of colour indices for 3 rows × 4 sub-rows × 11 columns = 132 entries per room):

```javascript
seed = room
prandom(1)   // advance seed once

for row in 0..2:
    for subrow in 0..3:
        colorBase = (subrow % 2) ? 0 : 4
        prevColor = -1
        for col in 0..10:
            do:
                color = colorBase + prandom(3)
            while color === prevColor
            wallPattern[room][44*row + 11*subrow + col] = color
            prevColor = color
```

Alternating rows use colour index offsets 0 and 4 within the `wallColor` array.

---

## prandom(max)

LCG pseudo-random number generator:
```javascript
seed = ((seed * 214013 + 2531011) & 0xFFFFFFFF) >>> 0
return (seed >>> 16) % (max + 1)
```

Same algorithm as the original Prince of Persia source.

---

## Level JSON Format (summary)

```json
{
  "number": 1,
  "name": "Level 1",
  "type": 0,
  "size": { "width": 5, "height": 3 },
  "room": [
    {
      "id": 1,
      "tile": [
        { "element": 1, "modifier": 0 },
        ...
      ]
    },
    { "id": -1 },
    ...
  ],
  "prince": {
    "room": 1,
    "location": 14,
    "direction": -1,
    "turn": true,
    "bias": 0,
    "offset": 0,
    "reverse": 1,
    "sword": false,
    "cameraRoom": null,
    "danger": true,
    "specialEvents": true
  },
  "guards": [
    {
      "room": 2, "location": 5, "direction": -1,
      "skill": 0, "colors": 0, "type": "guard",
      "visible": true, "active": true, "sneak": true,
      "bias": 0, "reverse": 1
    }
  ],
  "events": [
    { "room": 1, "location": 5, "next": true },
    { "room": 2, "location": 3, "next": false }
  ]
}
```
