# Scene.js — Spec

**File:** `src/Scene.js` (91 lines)  
**Purpose:** Cutscene room renderer. Builds a static princess's-room background with animated torches, stars, and pillars. Used by `PrinceJS.Cutscene`.

---

## Constructor: `PrinceJS.Scene(game)`

```javascript
back  = game.add.group()   // z=10
front = game.add.group()   // z=30
trobs = []                 // tiles needing per-tick updates
flash = false
tick  = 0
_build()
```

---

## _build()

Constructs the princess's room:

**Background images (added to `back` group):**
```
image(0,   0,   "cutscene", "room")       // main room bg
image(0,   142, "cutscene", "room_bed")   // bed area
```

**Torches** (two, added as Tile.Torch):
```
{ x: 53,  y: 81 }
{ x: 171, y: 81 }
```
Each: `new PrinceJS.Tile.Torch(game, TILE_TORCH, 0, TYPE_PALACE)` with `back.frameName = "palace_0"`.

**Stars** (six, added as Tile.Star):
```
{ x: 20, y: 97  }
{ x: 16, y: 104 }
{ x: 23, y: 110 }
{ x: 17, y: 116 }
{ x: 24, y: 120 }
{ x: 18, y: 128 }
```

**Foreground pillars (added to `front` group):**
```
image(59,  120, "cutscene", "room_pillar")
image(240, 120, "cutscene", "room_pillar")
```

---

## addObject(object)

```javascript
back.add(object.back)
addTrob(object)
```

---

## addTrob(trob)

```javascript
trobs.push(trob)
```

---

## update()

```javascript
i = trobs.length
while (i--): trobs[i].update()

if flash:
    if tick === 7: flash = false; return
    if tick % 2: stage.backgroundColor = "#FFFFFF"
    else:        stage.backgroundColor = "#000000"
    tick++
```

Flash cycles white/black 4 times (ticks 0–6), then stops.

---

## effect()

```javascript
flash = true
// tick is NOT reset here — starts from wherever it was
```

Triggers flash effect (used by `EFFECT` opcode in Cutscene program).
