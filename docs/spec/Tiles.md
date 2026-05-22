# Tiles — Spec

All tile files are in `src/tiles/`. All extend `PrinceJS.Tile.Base` (except `Clock` and `Star` which are standalone). Base is documented separately.

---

## Tile.Base — `src/tiles/Base.js`

**Constructor:** `PrinceJS.Tile.Base(game, element, modifier, type)`

```
element   — tile type integer (TILE_* constant)
modifier  — tile variant/state integer
type      — TYPE_DUNGEON or TYPE_PALACE
key       — "dungeon" or "palace"
back      — Phaser.Sprite (background layer)
front     — Phaser.Sprite (foreground layer, drawn above characters)
tileHeight= front.height
room, roomX, roomY — set by Level.addTile()
frame     = null     (saved frameName during mask)
crop      = false    (whether front is currently cropped)
debris    = false
```

### toggleMask(actor)

Swaps front sprite to background frame (so hanging character appears behind ledge) or restores it:
```javascript
if frame !== null:   // unmask
    front.frameName = frame
    if decoration: front.removeChild(decoration)
    if Button and active: front.crop(maskWidth(actor), tileHeight - offsetY)
    else: front.crop(null)
    frame = null; crop = false
else:                // mask
    initDecoration()
    frame = front.frameName
    if debris: front.frameName = debrisBack.frameName
    else:      front.frameName = back.frameName
    if decoration: front.addChild(decoration)
    front.crop(new Phaser.Rectangle(0, offsetY||0, maskWidth(actor), tileHeight))
    crop = true
```

### maskWidth(actor)
```javascript
if element === TILE_EXIT_LEFT: return 31
return 33
```

### isWalkable()
```
NOT in [WALL, SPACE, TOP_BIG_PILLAR, TAPESTRY_TOP,
        LATTICE_SUPPORT, SMALL_LATTICE, LATTICE_LEFT, LATTICE_RIGHT]
```

### isSpace()
```
element in [SPACE, TOP_BIG_PILLAR, TAPESTRY_TOP,
            LATTICE_SUPPORT, SMALL_LATTICE, LATTICE_LEFT, LATTICE_RIGHT]
```

### isBarrier()
```
element in [WALL, GATE, MIRROR, TAPESTRY, TAPESTRY_TOP]
```

### isFreeFallBarrier()
```
isBarrier() AND element NOT in [TAPESTRY, TAPESTRY_TOP]
```

### isBarrierLeft()
```
element in [WALL, MIRROR]
```

### isBarrierRight()
```
element in [GATE, TAPESTRY, TAPESTRY_TOP]
```

### isSeeBarrier()
```
element in [WALL, TAPESTRY, TAPESTRY_TOP]
```

### isDangerousWalkable()
```
element in [LOOSE_BOARD, CHOPPER]
```

### isSafeWalkable()
```
isWalkable() AND NOT isDangerousWalkable()
```

### isExitDoor()
```
element in [TILE_EXIT_LEFT, TILE_EXIT_RIGHT]
```

### getBounds()
```javascript
{ height: 63, width: 4, x: roomX*32 + 40, y: roomY*63 }
```

### getBoundsAbs()
```javascript
new Phaser.Rectangle(x, y, width, 63)
```

### addDebris()

Plays "LooseFloorLands". If already has debris, returns. Sets `debris = true`.

- For LOOSE_BOARD: calls `sweep()`
- For SPIKES: `drop()`, `element = FLOOR`, `revalidate()`
- Sets `debrisElement`:
  - TORCH → `TILE_TORCH_WITH_DEBRIS`
  - FLOOR → `TILE_DEBRIS`
  - other → `TILE_DEBRIS_ONLY`
- Creates `debrisBack` (added to `back`) and `debrisFront` (added to `front`)

### Position properties (defineProperty)
```
x: get/set → back.x and front.x
y: get/set → back.y and front.y
centerX: back.centerX (read-only)
centerY: back.centerY (read-only)
width:   back.width   (read-only)
height:  back.height  (read-only)
```

---

## Tile.Button — `src/tiles/Button.js`

**Constructor:** `PrinceJS.Tile.Button(game, element, modifier, type)`

```
element: STUCK_BUTTON (5), DROP_BUTTON (6), or RAISE_BUTTON (15)
stepMax = element === RAISE_BUTTON ? 3 : 5
step    = 0
active  = false
mute    = false
onPushed = new Phaser.Signal()
frontBevel — extra sprite for depression visual
```

If `element === STUCK_BUTTON`: `debris = true; mute = true`

### update()

- RAISE_BUTTON and active at step 0: `trigger(true)` (re-fires stuck signal)
- If `debris`: always active, `trigger(false)` once, plays "FloorButton" if not muted; calls `reset()` if already pressed
- If active: count up to `stepMax`, then `reset()` and `active = false`

### push()

```javascript
if !active:
    active = true
    offsetY = 1
    frontOriginalY = front.y; front.y += 1
    front.crop(height - 1); frontBevel.crop(height - 1)
    back.frameName += "_down"
    trigger(false)
    if !mute: play "FloorButton"
step = 0
```

### trigger(stuck)
```javascript
onPushed.dispatch(modifier, element, stuck)
// modifier = event index; element = button type for raise/drop logic
```

### reset()
```javascript
front.y = frontOriginalY; delete frontOriginalY
back.frameName = key + "_" + element
offsetY = 0
front.crop(restore); frontBevel.crop(null)
```

---

## Tile.Chopper — `src/tiles/Chopper.js`

**Constructor:** `PrinceJS.Tile.Chopper(game, modifier, type)`

```
tileChildBack   — animated chopper blade (back layer), frame "key_chopper_5"
tileChildFront  — animated chopper blade (front layer), frame "key_chopper_5_fg"
blood           — sprite at (12, 41), "chopper-blood_4", visible=false
step            = 0
active          = false
sound           = false
onChopped       = new Phaser.Signal()
```

### update()

```javascript
if active:
    step++
    if step > 14: step = 0; active = false
    else if step < 6:
        tileChildBack.frameName  = key + "_chopper_" + step
        tileChildFront.frameName = key + "_chopper_" + step + "_fg"
        blood.frameName = "chopper-blood_" + step
        if step === 3:
            onChopped.dispatch(roomX, roomY, room)
            if sound: play "SlicerBladesClash"
```

Animation: frames 0–5 (chopper closes), stays closed frames 6–14, then resets.

### chop(sound)
```javascript
active = true; this.sound = sound
```

### showBlood()
```javascript
blood.visible = true
```

---

## Tile.Clock — `src/tiles/Clock.js`

Cutscene-only. Not a Tile.Base subclass.

**Constructor:** `PrinceJS.Tile.Clock(game, x, y, state)`

```
back      — sprite at (x, y) from "cutscene" atlas
tileChild — sprite at (8, 16) within back, visible=false (sand animation)
sandStep  = 0
clockStep = state   (which clock face frame to start on)
step      = 0
active    = false
```

**Static frame arrays:**
```javascript
sandFrames  = ["clocksand01","clocksand02","clocksand03"]  // 3 frames
clockFrames = ["clock01","clock02",...,"clock07"]           // 7 frames
```

### update()
```javascript
if active:
    sandStep = (sandStep + 1) % 3
    tileChild.frameName = sandFrames[sandStep]
    step++
    if step === 40:   // every 40 ticks, advance clock face
        clockStep = (clockStep + 1) % 7
        back.frameName = clockFrames[clockStep]
        step = 0
```

### activate()
```javascript
active = true; tileChild.visible = true
```

---

## Tile.ExitDoor — `src/tiles/ExitDoor.js`

**Constructor:** `PrinceJS.Tile.ExitDoor(game, modifier, type, open=false)`

Tile type is always `TILE_EXIT_RIGHT` internally (exit left is handled by the adjacent tile sharing this).

```
tileChildBack  — door sprite at (10, 12) [palace: x-=3]
tileChildFront — door_fg sprite, visible=false (shows when player climbs stairs)
step     = 0
heightOpen  = 8 + type   (dungeon=8, palace=9)
heightClose = tileChildBack.height
heightCrop  = heightClose - heightOpen
```

**States:**
```
STATE_OPEN     = 0
STATE_RAISING  = 1
STATE_DROPPING = 2
STATE_CLOSED   = 3
```

**`open` property** (defineProperty):
```javascript
get: return state === STATE_OPEN
set(value): state = value ? STATE_OPEN : STATE_CLOSED; step = 0
```

### update()

- STATE_RAISING: crop door from top, 1px per tick; when full height === heightOpen → open = true
- STATE_DROPPING: un-crop door, 15px per tick; when full → open = false

### raise()
```javascript
if state === STATE_CLOSED: state = STATE_RAISING; play "ExitDoorOpening"
```

### drop()
```javascript
if state !== STATE_CLOSED: state = STATE_DROPPING; play "EntranceDoorCloses"
```

### mask()
```javascript
tileChildFront.visible = true   // shows as player walks through
```

### toggleMask()
No-op (overridden to prevent base masking logic).

---

## Tile.Gate — `src/tiles/Gate.js`

See [Gate.md](Gate.md). Covered in detail above. Summary:

```
posY        — current gate position offset (negative = raised)
state       — CLOSED/OPEN/RAISING/DROPPING/FAST_DROPPING/WAITING
step        — counter within state
closedFast  — true after a fast drop (prevents re-raise via stuck button)
soundActive — whether gate sounds play (muted when not visible)
canMute     — whether sound can be muted
onFastDrop  = new Phaser.Signal()
```

States: gate starts at `modifier * -46`. Raises 1px/tick to -47, waits 50 ticks, drops 1px/4ticks. Fast drop: 10px/tick.

---

## Tile.Loose — `src/tiles/Loose.js`

**Constructor:** `PrinceJS.Tile.Loose(game, modifier, type)`

```
onStartFalling = new Phaser.Signal()
onStopFalling  = new Phaser.Signal()
step   = 0
state  = STATE_INACTIVE
vacc   = 0    // accumulated vertical travel
yTo    = 0    // target travel distance (set by Level.floorStartFall)
```

**States:**
```
STATE_INACTIVE = 0
STATE_SHAKING  = 1
STATE_FALLING  = 2
```

**Constants:**
```
FALL_VELOCITY = 3
frames = ["_loose_1","_loose_2",...,"_loose_8"]   // 8 shake frames
```

### update()

**SHAKING:**
- Cycles through 8 shake frames
- At frame 3: if `!fall` (non-fatal shake), restore and return to INACTIVE
- At frame 8 (all frames done): → FALLING, `onStartFalling.dispatch(this)`
- Plays shake sounds at steps 0, 3, 7 (random from 3 sound variants)

**FALLING:**
```javascript
value = FALL_VELOCITY * step   // accelerating fall
y    += value
step++
vacc += value
if vacc > yTo: state = STATE_INACTIVE; onStopFalling.dispatch(this)
```

### shake(fall)
```javascript
if state === STATE_INACTIVE:
    state = STATE_SHAKING; step = 0
    front.visible = crop   // maintain mask state
this.fall = fall   // true = will fall, false = just shake
```

### sweep()
```javascript
state = STATE_FALLING; step = 0
back.frameName = key + "_falling"
onStartFalling.dispatch(this)
play "LooseFloorShakes1"
```

### fallStarted()
```javascript
return state === STATE_SHAKING && step === frames.length   // 8 = about to fall
```

### toggleMask(actor) (override)
```javascript
PrinceJS.Tile.Base.prototype.toggleMask.call(this, ...arguments)
front.visible = crop   // sync front visibility with crop state
```

---

## Tile.Mirror — `src/tiles/Mirror.js`

**Constructor:** `PrinceJS.Tile.Mirror(game, modifier, type)`

Initialises as `TILE_FLOOR` (invisible mirror). Palace type only:
```
mirrorBack      — mirror overlay sprite at (3,-3), visible=false
mirrorFront     — mirror fg overlay, visible=false
reflectionGroup — group with scale.x=-1 (mirror flip), visible=false
reflection      — kid sprite at (0,0) within group, anchor(0,1), visible=false
reflectionCover — cover sprite at (-103,-5), visible=false
```

### addObject()

Activates mirror (called by Game level 4 logic):
```javascript
element = TILE_MIRROR
mirrorBack.visible = mirrorFront.visible = true
reflectionGroup.visible = reflectionCover.visible = true
```

### syncFrame(actor)

Called by Kid as `delegate.syncFrame` each tick:
```javascript
reflection.frameName = actor.frameName
reflection.x = actor.x - this.x - 55
reflection.x = Math.max(x, faceL() ? -25 : -10)
reflection.y = actor.y - this.y
reflection.visible = (x > -40 && x < 20)
```

### syncFace(actor)
```javascript
reflection.charFace = actor.charFace
reflection.scale.x  = actor.scale.x
```

### hideReflection()
```javascript
reflection.visible = false; reflection = null
```

### toggleMask(actor) (override)
```javascript
if element === TILE_FLOOR:   // only mask when acting as floor
    PrinceJS.Tile.Base.prototype.toggleMask.call(this, ...arguments)
```

---

## Tile.Potion — `src/tiles/Potion.js`

**Constructor:** `PrinceJS.Tile.Potion(game, modifier, type)`

```
specialModifier = modifier (original, before clamping)
isSpecial       = (modifier >= POTION_SPECIAL)   // modifier >= 6
modifier        = Math.max(1, Math.min(5, modifier))   // clamped for display
onDrank         = new Phaser.Signal()
tileChild       — bubble sprite at (25, yy) where yy=53 (or 49 for modifiers 2-4)
step            — random start frame (0–6)
color           — from bubbleColors[modifier-1]
```

**Static data:**
```javascript
frames       = ["bubble_1","bubble_2",...,"bubble_7"]   // 7 bubble frames
bubbleColors = ["red","red","green","green","blue"]
// modifier: 1=red(recover), 2=red(add), 3=green(buffer), 4=green(flip), 5=blue(damage)
```

### update()
```javascript
tileChild.frameName = frames[step] + "_" + color
step = (step + 1) % 7
```

### removeObject()

Called when kid drinks potion:
```javascript
tileChild.destroy()
element = TILE_FLOOR; modifier = 0
front.frameName = key + "_" + element + "_fg"
back.frameName  = key + "_" + element
back.addChild(make.sprite(0,0, key, key + "_" + element + "_0"))
if decoration: decoration.destroy(); decoration = undefined
if isSpecial:
    delayed(() => onDrank.dispatch(specialModifier, TILE_RAISE_BUTTON), 1000)
```

---

## Tile.Skeleton — `src/tiles/Skeleton.js`

**Constructor:** `PrinceJS.Tile.Skeleton(game, modifier, type)`

A purely decorative floor tile (bones lying on ground) that gets replaced when the skeleton enemy activates.

### update() — no-op

### removeObject()
```javascript
element  = TILE_FLOOR; modifier = 0
front.frameName = key + "_" + element + "_fg"
back.frameName  = key + "_" + element
back.addChild(make.sprite(0,0, key, key + "_" + element + "_0"))
```

---

## Tile.Spikes — `src/tiles/Spikes.js`

**Constructor:** `PrinceJS.Tile.Spikes(game, modifier, type)`

```
mortal = (modifier < 5)   // fully extended spikes kill; partial may not
step   = 0
state  = STATE_INACTIVE
```

Modifier remapping for sprite frames:
```
modifier 3,4,5 → 5
modifier 6     → 4
modifier > 6   → 9 - modifier
```

```
tileChildBack  — spike frame sprite
tileChildFront — spike fg sprite
```

**States:**
```
STATE_INACTIVE = 0
STATE_RAISING  = 1
STATE_FULL_OUT = 2
STATE_DROPPING = 3
```

### update()

**RAISING:** step 1→5 (frame per tick); at step 5 → STATE_FULL_OUT, step=0  
**FULL_OUT:** counts 15 ticks, then calls `drop()`  
**DROPPING:** step 5→0 (skip step 3: jumps from 4→2); at step 0 → STATE_INACTIVE  

### raise()
```javascript
if state === STATE_INACTIVE:
    state = STATE_RAISING; play "ImpaledBySpikes"
else if state === STATE_FULL_OUT:
    step = 0   // reset full-out timer
```

### drop()
```javascript
state = STATE_DROPPING; step = 5
```

### maskWidth(actor) (override)
```javascript
return actor.action === "climbdown" ? 21 : 22
```

---

## Tile.Star — `src/tiles/Star.js`

Cutscene-only. Not a Tile.Base subclass.

**Constructor:** `PrinceJS.Tile.Star(game, x, y)`

```
back  — sprite at (x, y) from "cutscene" atlas
state = 1   (0=dark, 1=dim, 2=bright)
```

Calls `update()` immediately on construction.

### update()

```javascript
step = rnd.between(1, 10)
if step === 1: if state > 0: state--
if step === 2: if state < 2: state++
back.frameName = "star" + state
```

Each tick: 10% chance to dim, 10% chance to brighten, 80% unchanged.

---

## Tile.Sword — `src/tiles/Sword.js`

**Constructor:** `PrinceJS.Tile.Sword(game, modifier, type)`

```
tick = rnd.between(40, 167)   // random delay before first gleam
step = 0
```

### update()

```javascript
if step === -1:
    back.frameName = key + "_" + element   // restore normal frame
    tick = rnd.between(40, 167)             // randomize next gleam delay
step++
if step === tick:
    back.frameName += "_bright"   // gleam frame
    step = -1
```

Sword gleams periodically with a random interval (40–167 ticks).

### removeObject()

```javascript
element = TILE_FLOOR; modifier = 0
front.frameName = key + "_" + element + "_fg"
back.frameName  = key + "_" + element
back.addChild(make.sprite(0,0, key, key + "_" + element + "_0"))
```

---

## Tile.Torch — `src/tiles/Torch.js`

**Constructor:** `PrinceJS.Tile.Torch(game, element, modifier, type)`

```
tileChild — fire sprite at (40, 18) from "general" atlas
step      = rnd.between(0, 8)   // random start phase
```

**Static:**
```javascript
frames = ["fire_1","fire_2",...,"fire_9"]   // 9 animation frames
```

### update()
```javascript
tileChild.frameName = frames[step]
step = (step + 1) % 9
```

Torch animates continuously through 9 fire frames, starting at a random offset so multiple torches flicker out of sync.
