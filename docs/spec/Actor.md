# Actor.js — Spec

**File:** `src/Actor.js`  
**Purpose:** Base class for all animated characters. Manages the command bytecode processor, per-frame position data (`fdx`/`fdy`/`fcheck`), and sprite frame display.

---

## Constructor: `PrinceJS.Actor(game, x, y, charName, direction)`

Extends `Phaser.Sprite`. Initial Phaser position always `(0, 0)`.

**Instance variables:**

```
charX          — logical x position (0–140 per room)
charY          — logical y position
charFace       — 1 (right) or -1 (left)
charName       — sprite atlas key (e.g. "kid", "guard-1")
charFrame      — current animation frame index (undefined initially)
charFdx        — x offset from animation data (updated per frame)
charFdy        — y offset from animation data (updated per frame)
charFcheck     — collision/floor check flag (from animation bitmask)
charFfoot      — foot offset (bits 0–4 of fcheck bitmask)
charFood       — half-pixel parity flag (bit 7 of fcheck bitmask)
charFthin      — thin-frame flag (bit 5 of fcheck bitmask)
scale.x        — set to -charFace (negative = flipped horizontally)
anchor         — (0, 1) — bottom-left origin
_action        — internal action string (e.g. "stand", "running")
_seqpointer    — current position in the action's command sequence
z              = 20
baseX          = 0
baseY          = 0
anims          — animation data from cache (JSON)
commands       — Array[256], pre-filled CMD_NOOP; 6 registered slots
delegate       = null
```

---

## Command Table

256-slot array. Default: all `CMD_NOOP`. Six handlers registered:

| Opcode | Hex  | Name          | Behaviour |
|--------|------|---------------|-----------|
| 0x00   | 0    | CMD_FRAME     | Set `charFrame`, call `updateCharFrame()`, set `processing = false` (only loop exit) |
| 0xF2   | 242  | CMD_TAP       | No-op in base Actor (overridden in Kid) |
| 0xFA   | 250  | CMD_CHY       | `charY += data.p1` |
| 0xFB   | 251  | CMD_CHX       | `charX += data.p1 * charFace` |
| 0xFE   | 254  | CMD_ABOUTFACE | Calls `changeFace()` |
| 0xFF   | 255  | CMD_GOTO      | `_action = data.p1`; `_seqpointer = data.p2 - 1`; continues processing immediately |

---

## processCommand()

```javascript
processing = true
while (processing):
    data = anims[_action][_seqpointer]
    commands[data.cmd](data)
    _seqpointer++
```

Loop only exits when `CMD_FRAME` sets `processing = false`.

---

## action Setter

```javascript
set action(value):
    _action = value
    _seqpointer = 0    // ALWAYS resets to start of sequence
```

---

## updateCharFrame()

Called by `CMD_FRAME` after setting `charFrame`. Updates `frameName` and reads per-frame animation data:

```javascript
frameName = charName + "-" + charFrame
// reads from animation JSON:
charFdx   = framedef.fdx
charFdy   = framedef.fdy
// fcheck bitmask decode:
charFfoot = fcheck & 0x1F          // bits 0–4
charFthin = (fcheck >> 5) & 0x1   // bit 5
charFcheck= (fcheck >> 6) & 0x1   // bit 6 — floor/collision gate
charFood  = (fcheck >> 7) & 0x1   // bit 7 — half-pixel parity
```

---

## updateCharPosition()

```javascript
frameName = charName + "-" + charFrame
tempx = charX + charFdx * charFace
// half-pixel correction:
if (charFood && faceL()) || (!charFood && faceR()):
    tempx += 0.5
x = baseX + convertX(tempx)
y = baseY + charY + charFdy
```

---

## frameID(from, to)

```javascript
// Single argument:
frameID(n)        → charFrame === n
// Two arguments:
frameID(from, to) → from <= charFrame && charFrame <= to
```

---

## faceL() / faceR()

```javascript
faceL() → charFace === -1
faceR() → charFace === 1
```

---

## changeFace()

```javascript
charFace *= -1
scale.x *= -1
```

---

## Base updateActor()

Used only for cutscene actors (not Kid or Enemy):

```javascript
processCommand()
updateCharPosition()
```

---

## registerCommand(opcode, fn)

Registers a handler function at `commands[opcode]`.

---

## Animation Data Format (JSON)

Each action sequence is an array of objects:

```json
{ "cmd": 0xFF, "p1": "running", "p2": 3 }     // GOTO
{ "cmd": 0x00, "fdx": 2, "fdy": -1, "fcheck": 64, "id": 7 }  // FRAME
```

`fcheck` encodes `charFfoot` (bits 0–4), `charFthin` (bit 5), `charFcheck` (bit 6), `charFood` (bit 7).
