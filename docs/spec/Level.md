# Level.js — Spec

**File:** `src/Level.js` (389 lines)  
**Purpose:** Tile map container. Stores rooms, tiles, event wiring, TROB (tile update) list, masked tile tracking, and gate visibility management.

---

## Constructor: `PrinceJS.Level(game, number, name, type)`

```javascript
this.number = number        // 1–16
this.name   = name          // display name
this.type   = type          // TYPE_DUNGEON or TYPE_PALACE

this.rooms  = []            // indexed by room number (1-based)
this.back   = game.add.group()   // z=10 — background tile sprites
this.front  = game.add.group()   // z=30 — foreground tile sprites
this.trobs  = []            // tiles needing per-tick update
this.maskedTiles = {}       // keyed by actor.id → masked tile
this.dummyWall   = new PrinceJS.Tile.Base(game, TILE_WALL, 0, type)
this.exitDoorOpen = false
this.activeGates  = []      // gates currently visible (for muting)
```

---

## Level Type Constants

```javascript
PrinceJS.Level.TYPE_DUNGEON = 0
PrinceJS.Level.TYPE_PALACE  = 1
```

---

## Tile Element ID Constants

```javascript
TILE_SPACE           = 0
TILE_FLOOR           = 1
TILE_SPIKES          = 2
TILE_PILLAR          = 3
TILE_GATE            = 4
TILE_STUCK_BUTTON    = 5
TILE_DROP_BUTTON     = 6
TILE_TAPESTRY        = 7
TILE_BOTTOM_BIG_PILLAR = 8
TILE_TOP_BIG_PILLAR  = 9
TILE_POTION          = 10
TILE_LOOSE_BOARD     = 11
TILE_TAPESTRY_TOP    = 12
TILE_MIRROR          = 13
TILE_DEBRIS          = 14
TILE_RAISE_BUTTON    = 15
TILE_EXIT_LEFT       = 16
TILE_EXIT_RIGHT      = 17
TILE_CHOPPER         = 18
TILE_TORCH           = 19
TILE_WALL            = 20
TILE_SKELETON        = 21
TILE_SWORD           = 22
TILE_BALCONY_LEFT    = 23
TILE_BALCONY_RIGHT   = 24
TILE_LATTICE_PILLAR  = 25
TILE_LATTICE_SUPPORT = 26
TILE_SMALL_LATTICE   = 27
TILE_LATTICE_LEFT    = 28
TILE_LATTICE_RIGHT   = 29
TILE_TORCH_WITH_DEBRIS = 30
TILE_DEBRIS_ONLY     = 31
TILE_NULL            = 32
```

---

## Potion Type Constants

```javascript
POTION_RECOVER = 1   // restore 1 HP
POTION_ADD     = 2   // +1 max HP
POTION_BUFFER  = 3   // float / slow fall
POTION_FLIP    = 4   // invert screen
POTION_DAMAGE  = 5   // lose 1 HP
POTION_SPECIAL = 6   // cutscene-controlled
```

---

## Flash Color Constants

```javascript
FLASH_RED    = 0xff0000
FLASH_GREEN  = 0x00ff00
FLASH_YELLOW = 0xffff00
FLASH_WHITE  = 0xffffff
```

---

## Room Data Structure

Each `rooms[n]` entry has:
```javascript
{
    tiles: Array[30],   // 10 wide × 3 tall, indexed as y*10+x
    links: {
        left:  roomNumber,
        right: roomNumber,
        up:    roomNumber,
        down:  roomNumber
    },
    x: roomGridX,   // column position in world grid
    y: roomGridY    // row position in world grid
}
```

Room indices are 1-based (0 = no room / outside world).

---

## addTile(x, y, room, tile)

```javascript
if x >= 0 && y >= 0:
    rooms[room].tiles[y * 10 + x] = tile
    tile.roomX = x
    tile.roomY = y
    tile.room  = room

tile.x = rooms[room].x * ROOM_WIDTH  + x * BLOCK_WIDTH
tile.y = rooms[room].y * ROOM_HEIGHT + y * BLOCK_HEIGHT - 13   // NOTE: -13 offset

back.add(tile.back)
front.add(tile.front)
```

---

## addTrob(trob)

```javascript
trobs.push(trob)
```

Registers a tile for per-tick update calls. Called by Gate, Chopper, Loose, Spikes, etc. during level build.

---

## update()

```javascript
i = trobs.length
while (i--):
    trobs[i].update()
```

Iterates backwards to allow safe splice during update.

---

## removeObject(x, y, room)

```javascript
tile = getTileAt(x, y, room)
if tile && tile.removeObject:
    tile.removeObject()
    idx = trobs.indexOf(tile)
    if idx > -1: trobs.splice(idx, 1)
```

---

## getTileAt(x, y, room)

Resolves cross-room coordinates. Returns `dummyWall` (TILE_WALL) if room is invalid.

```javascript
if !rooms[room]: return dummyWall

// Try X traversal first, then Y:
result = getRoomX(room, x)
if result.room > 0:
    newRoom = result.room; newX = result.x
    result  = getRoomY(newRoom, y)
    newRoom = result.room; newY = result.y
else:
    result  = getRoomY(room, y)
    newRoom = result.room; newY = result.y
    if result.room > 0:
        result  = getRoomX(newRoom, x)
        newRoom = result.room; newX = result.x

if newRoom <= 0: return dummyWall
return rooms[newRoom].tiles[newX + newY * 10]
```

---

## getRoomX(room, x)

```javascript
if x < 0:  room = links.left;  x += 10
if x > 9:  room = links.right; x -= 10
return { room, x }
```

---

## getRoomY(room, y)

```javascript
if y < 0:  room = links.up;   y += 3
if y > 2:  room = links.down; y -= 3
return { room, y }
```

---

## shakeFloor(y, room)

Shakes all loose boards in row `y` of `room`:
```javascript
for x in 0..9:
    tile = getTileAt(x, y, room)
    if tile.element === TILE_LOOSE_BOARD: tile.shake(false)
```

---

## maskTile(x, y, room, actor)

Swaps a tile's sprite to show the background frame (so a climbing character appears behind the ledge):
```javascript
tile = getTileAt(x, y, room)
if maskedTiles[actor.id] === tile: return   // already masked
if maskedTiles[actor.id]: unMaskTile(actor) // unmask previous
if tile.isWalkable():
    maskedTiles[actor.id] = tile
    tile.toggleMask(actor)
```

---

## unMaskTile(actor)

```javascript
if maskedTiles[actor.id]:
    maskedTiles[actor.id].toggleMask(actor)
    delete maskedTiles[actor.id]
```

---

## floorStartFall(tile)

Called when a loose board begins falling:
```javascript
// Replace the loose board position with a SPACE tile
space = new Tile.Base(game, TILE_SPACE, 0, tile.type)
if type === TYPE_PALACE: space.back.frameName = key + "_0_1"
addTile(tile.roomX, tile.roomY, tile.room, space)

// Find the landing row (first non-SPACE below)
while getTileAt(tile.roomX, tile.roomY, tile.room).element === TILE_SPACE:
    tile.roomY++
    if tile.roomY === 3:
        tile.roomY = 0
        tile.room  = links.down
    tile.yTo += BLOCK_HEIGHT
```

---

## floorStopFall(tile)

Called when a falling loose board reaches its landing row:
```javascript
floor = getTileAt(tile.roomX, tile.roomY, tile.room)
if floor.element !== TILE_SPACE:
    tile.destroy()
    floor.addDebris()
    shakeFloor(tile.roomY, tile.room)
else:
    tile.sweep()   // tile fell into another empty space, just clean up
```

---

## fireEvent(event, type, stuck)

Triggers a level event (button press → gate/exit response). Events chain via `next` flag.

```javascript
if !events[event]: return
room     = events[event].room
x        = (events[event].location - 1) % 10
y        = Math.floor((events[event].location - 1) / 10)
tile     = getTileAt(x, y, room)
if tile.element === TILE_EXIT_LEFT: tile = getTileAt(x+1, y, room)

if type === TILE_RAISE_BUTTON:
    if tile.raise:
        tile.raise(stuck)
        if tile.element in [EXIT_LEFT, EXIT_RIGHT]: exitDoorOpen = true
else:
    if tile.drop: tile.drop(stuck)

if events[event].next:
    fireEvent(event + 1, type)   // chain to next event
```

---

## activateChopper(x, y, room)

Scans right from `x` to find the nearest chopper and activates it:
```javascript
do:
    tile = getTileAt(++x, y, room)
while (x < 9 && tile.element !== TILE_CHOPPER)

if tile.element === TILE_CHOPPER:
    delegate.handleChop(tile)
```

---

## Gate Sound/Visibility Management

### checkGates(room, prevRoom)

Updates `activeGates` list and calls `isVisible(true/false)` on each gate (controls whether gate sounds play):
```javascript
gates = getGatesAll(room, prevRoom)
activeGates.forEach(gate => { if !gates.includes(gate): gate.isVisible(false) })
gates.forEach(gate => gate.isVisible(true))
activeGates = gates
```

### getGatesAll(room, prevRoom)

Combines: all gates in `room`, all gates at x=9 in left neighbour room, plus (when prevRoom given) gates at the shared vertical edge.

### getGates(room, edgeX, edgeY)

Returns all gates in `room`, filtered by optional `edgeX` and/or `edgeY`.

### getGatesLeft(room)

Gates at `x=9` in `rooms[room].links.left`.

### getGatesRight(room)

Gates at `x=0` in `rooms[room].links.right`.

### getGatesUp(room)

Gates at `y=2` in `rooms[room].links.up`, plus gates at `x=9, y=2` in the up-room's left neighbour.

### getGatesDown(room)

Gates at `y=0` in `rooms[room].links.down`, plus gates at `x=9, y=0` in the down-room's left neighbour.

---

## recheckCurrentRoom()

```javascript
delegate.recheckCurrentRoom()
```

Delegated to Game state — forces camera/room recalculation.
