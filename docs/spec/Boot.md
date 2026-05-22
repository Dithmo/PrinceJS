# Boot.js — Spec

**File:** `src/Boot.js`  
**Purpose:** Defines the entire `PrinceJS` global namespace, all display/layout constants, global mutable game state, gamepad mapping, and the Phaser Boot state.

---

## Display Constants

```
PrinceJS.SCALE_FACTOR   = 2
PrinceJS.SCREEN_WIDTH   = 320       // logical pixels
PrinceJS.SCREEN_HEIGHT  = 200
PrinceJS.WORLD_WIDTH    = 640       // canvas pixels (SCREEN * SCALE)
PrinceJS.WORLD_HEIGHT   = 400
PrinceJS.WORLD_RATIO    = 1.6       // WORLD_WIDTH / WORLD_HEIGHT
PrinceJS.BLOCK_WIDTH    = 32
PrinceJS.BLOCK_HEIGHT   = 63
PrinceJS.ROOM_HEIGHT    = 189       // 3 * BLOCK_HEIGHT
PrinceJS.ROOM_WIDTH     = 320
PrinceJS.UI_HEIGHT      = 8
```

---

## Global Mutable State — `PrinceJS.Init()`

Called at startup and on every restart via `PrinceJS.Restart()`.

```
PrinceJS.currentLevel   = 1
PrinceJS.maxHealth      = 3
PrinceJS.currentHealth  = null      // null means use maxHealth
PrinceJS.minutes        = 60        // timer start value
PrinceJS.startTime      = undefined
PrinceJS.endTime        = undefined
PrinceJS.strength       = 100
PrinceJS.screenWidth    = 0
PrinceJS.shortcut       = false
PrinceJS.danger         = null
PrinceJS.skipShowLevel  = false
```

---

## `PrinceJS.Restart()`

```javascript
PrinceJS.clearQuery()
PrinceJS.Init()
PrinceJS.applyQuery()
```

Resets all global state then re-applies URL parameters.

---

## Gamepad Button Mapping

```
A=0, B=1, X=2, Y=3, L=4, R=5, ZL=6, ZR=7
Minus=8, Plus=9
DPadUp=12, DPadDown=13, DPadLeft=14, DPadRight=15
```

**Axes:**
```
LeftX=0, LeftY=1, RightX=2, RightY=3
```

---

## Boot State

**preload:** Loads bitmap font atlas with key `"font"`.

**create:**
```javascript
game.world.scale.setTo(2, 2)
game.state.start("Preloader")
```
