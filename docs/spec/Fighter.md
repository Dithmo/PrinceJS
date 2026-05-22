# Fighter.js — Spec

**File:** `src/Fighter.js` (1,402 lines)  
**Extends:** `PrinceJS.Actor`  
**Purpose:** Base class for all walking/fighting characters (Kid, Enemy, Shadow, Mouse). Handles physics, combat, room transitions, tile interaction.

---

## Constructor: `PrinceJS.Fighter(game, level, location, direction, room, key, animKey)`

- `location`: integer; `charBlockX = location % 10`, `charBlockY = Math.floor(location / 10)`
- `charX = convertBlockXtoX(charBlockX)`, `charY = convertBlockYtoY(charBlockY)`
- `charFace` = direction (1=right, -1=left)

**Physics constants (prototype):**
```
GRAVITY        = 3
GRAVITY_FLOAT  = 1
TOP_SPEED      = 33
TOP_SPEED_FLOAT= 4
```

**Instance variables:**
```
charXVel        = 0
charYVel        = 0
actionCode      = 1
charSword       = true
flee            = false
allowAdvance    = true
allowRetreat    = true
allowBlock      = true
allowStrike     = true
inJumpUp        = false
inFallDown      = false
inFloat         = false
inFloatTimeoutCancel = null
fallingBlocks   = 0
swordFrame      = 0
swordDx         = 0
swordDy         = 0
hasSword        = true
health          = 3
alive           = true
swordDrawn      = false
blocked         = false
sneakUp         = true
```

**Sword sprite:** `new Phaser.Sprite` at (0,0); `sword.z = 21`

**Splash sprite:** child sprite at (-6, -15) relative to character; `visible = false`, `splashTimer = 0`. NOT created for skeleton characters.

**Signals:**
```
onInitLife       — dispatched when health initialized
onDamageLife     — dispatched(amount) on HP loss
onDead           — dispatched on death
onStrikeBlocked  — dispatched when strike is blocked
onEnemyStrike    — dispatched when enemy hits player
onChangeRoom     — dispatched(room) on room transition
```

---

## Additional Command Handlers

**CMD_SETFALL (0xF8):**
```javascript
charXVel = data.p1 * charFace
charYVel = data.p2
```

**CMD_ACT (0xF9):**
```javascript
actionCode = data.p1
if (data.p1 === 1): charXVel = 0, charYVel = 0
```

**CMD_DIE (0xF6):**
```javascript
alive = false
swordDrawn = false
showSplash()
proceedOnDead()
// delayed sound plays after
```

**CMD_FRAME override:** calls `updateSwordFrame()` and `updateBlockXY()` before setting `processing = false`.

---

## updateBase()

```javascript
baseX = rooms[room].x * 320
baseY = rooms[room].y * 189 + 3    // NOTE: +3 pixel offset
```

---

## updateSwordFrame()

```javascript
charSword = (typeof framedef.fsword !== 'undefined')
if (charSword):
    stab = swordAnims.swordtab[fsword - 1]
    swordFrame = stab.id
    swordDx    = stab.dx
    swordDy    = stab.dy
```

---

## updateBlockXY()

```javascript
footX    = charX + charFdx * charFace - charFfoot * charFace
footY    = charY + charFdy
charBlockX = convertXtoBlockX(footX)
charBlockY = Math.min(convertYtoBlockY(footY), 2)
updateFallingBlocks()
```

**Room transition** (SKIPPED for `climbup`/`climbdown`):
- Left exit: `charX += 140`, `baseX -= 320`, `charBlockX = 9`, room = `links.left`
- Right exit: `charX -= 140`, `baseX += 320`, `charBlockX = 0`, room = `links.right`
- `highjump` action: special case prevents room change

---

## updateFallingBlocks()

Only active when `inFallDown === true`:
- Increments `fallingBlocks` when `charBlockY` changes downward
- At `fallingBlocks === 5` (kid only): plays `"FallingFloorLands"`

---

## Fighter.updateActor() — Execution Order

```
1. updateSplash()
2. processCommand()
3. updateAcceleration()
4. updateVelocity()
5. checkFight()
6. checkRoomChange()
7. updateCharPosition()
8. updateSwordPosition()
9. maskAndCrop()
```

---

## updateAcceleration()

Only active when `actionCode === 4` (falling):
```javascript
if (inFloat):
    charYVel += GRAVITY_FLOAT  // 1
    cap at TOP_SPEED_FLOAT     // 4
else:
    charYVel += GRAVITY        // 3
    cap at TOP_SPEED           // 33
```

---

## updateVelocity()

```javascript
charX += charXVel
charY += charYVel
```

---

## checkFight()

**Auto-turn** (enemies, standing): `actionCode === 1`, opponent in room, not facing, `charX > 0`, `opponent.charX > 0`, distance >= 20, not sneaking → `changeFace()`.

**Strike hit detection frames:** 153/154 or 3/4  
**Hit range:**
```
minHurtDistance = swordDrawn ? 12 : 8
maxHurtDistance = 29  (+2 for fatguard)
distance <= 0 also counts as hit
```

**Block detection:** opponent frame 150 or 0.

---

## checkFloor()

Skipped if character not visible. Advance/retreat override `charFcheck = true`. Wall tile: checks forward tile instead.

**actionCode 0, 1, 5 (stand/walk/bump):**
```
checkCharFcheck && tile is SPACE/TOP_BIG_PILLAR/TAPESTRY_TOP:
    startFall() unless dead/bump/strike
tile is LOOSE_BOARD:
    shake(true)
tile is SPIKES:
    raise()
```

**actionCode 4 (falling):**
```
checkFall()
```

---

## checkFall()

```javascript
if (charY + 6 >= convertBlockYtoY(charBlockY)):
    if tile.isWalkable():
        land()
    else if tile.isFreeFallBarrier():
        adjust charX by (isBarrierLeft ? 10 : 5) * charFace
        updateBlockXY()
        recheck
```

---

## checkRoomChange()

```javascript
if (charY > 192):
    charY -= 192
    baseY += 189
    room = links.down
```

---

## startFall()

```javascript
fallingBlocks = 0
inFallDown = true
action = "stepfall"
maskTile()
if (retreat || swordDrawn): adjust charX
```

---

## land()

```javascript
charY = convertBlockYtoY(charBlockY)
charXVel = charYVel = 0

if skeleton or shadow: fallingBlocks = 1

if spikes: dieSpikes()
else if alive:
    fallingBlocks 0–1 → action = "stand"  (shadow: "softlandStandup")
    else             → die("falldead")
    sneakUp reset with 250ms restore
```

---

## distanceToEdge()

```javascript
if faceR: convertBlockXtoX(charBlockX + 1) - 1 - charX - charFdx + charFfoot
if faceL: charX + charFdx + charFfoot - convertBlockXtoX(charBlockX)
```

## distanceToFloor()

```javascript
convertBlockYtoY(charBlockY) - charY - charFdy
```

## distanceToTopFloor()

```javascript
convertBlockYtoY(charBlockY - 1) - charY - charFdy
```

---

## checkSpikes()

```javascript
if distanceToEdge() < 5: trySpikes(charBlockX + charFace, charBlockY)
always:                   trySpikes(charBlockX, charBlockY)
```

---

## checkChoppers()

Kid only activates choppers. Checks current room and adjacent rooms.
```
inChopDistance:   Math.abs(chopDistance) < 6 + (swordDrawn ? 10 : 0)
nearChopDistance: Math.abs <= 16
dodgeChoppers:    distance 13–16 or -16 to -13 → adjust charX
```

---

## Combat Stance Methods (frame-gated)

| Method | Valid frames |
|--------|-------------|
| `engarde()` | 158, 170, 8, 20–21 |
| `retreat()` | 158, 170, 8, 20–21 |
| `advance()` | 158, 171, 8, 20–21 |
| `strike()`  | 157–158, 165, 170–171, 7–8, 20–21, 15 OR 150, 161, 0, blocked |
| `block()`   | 8, 20–21, 18, 15 or 17 |

---

## stabbed()

```javascript
play sound
if health === 0: return
snap charY to floor
if not skeleton:
    if kid and no sword: die()
    else: damageLife()
    if health === 0: action = "stabkill"
    else: action = "stabbed"
showSplash()
```

---

## damageLife()

```javascript
if health > 1:
    health -= 1
    onDamageLife.dispatch(1)
    if shadow and active: opponent.damageLife()
else:
    die()
    if shadow and active: opponent.die()
```

---

## die(action)

```javascript
if skeleton: action = "stand"; return
health → 0
onDamageLife.dispatch(damage)
action = action || "dropdead"
alive = false
swordDrawn = false
hideSplash()
bringAboveOpponent()
```

---

## opponentDistance()

```javascript
if not same level: return 999 * (canWalkOnNextTile ? 1 : -1)
// Adjacent room offsets:
opponentInRoomLeft  → offset = -150 * charFace
opponentInRoomRight → offset = +150 * charFace
maxCharBlockX = (opponent.charX - charX) * charFace + offset
if maxCharBlockX >= 0 && different facing: maxCharBlockX += 13
```

---

## Room Proximity Methods

```
opponentInSameRoom()       — room === opponent.room
opponentInRoomLeft()       — links.left === opponent.room
opponentInRoomRight()      — links.right === opponent.room
opponentNearRoomLeft()     — inRoomLeft AND canSeeRoomLeft()
opponentNearRoomRight()    — inRoomRight AND canSeeRoomRight()
opponentCloseRoomLeft()    — inRoomLeft AND charBlockX >= 9
opponentCloseRoomRight()   — inRoomRight AND opponent.charBlockX >= 9
```

**canSeeRoomRight():** tile at x=9 (current) AND x=0 (right room): neither `isSeeBarrier()`  
**canSeeRoomLeft():** tile at x=0 (current) AND x=9 (left room): neither `isSeeBarrier()`

---

## State Query Methods

**isHanging():**
```
["hang","hangstraight","climbup","climbdown","hangdrop","jumphanglong"]
```

**moveR() / moveL():** `false` for stoop, bump, stand, turn, turnengarde, strike; `false` for engarde (extended mode); otherwise based on facing + action.

**sneaks():**
```
stoop / stand / standup / turn / jumpbackhang / jumphanglong /
hang / hangstraight / hangdrop / climbup / climbdown / testfoot
OR action.startsWith("step")
```

---

## maskAndCrop()

```javascript
if frameID(16) || frameID(21) || frameID(35):
    level.unMaskTile(this)
```

---

## nearBarrier()

Checks forward tile (`tileF`) for: `TILE_WALL`, gate `canCrossGate` height check, tapestry direction rules.

---

## alignToTile(tile)

```javascript
if faceL: charX = convertBlockXtoX(tile.roomX) - 2
if faceR: charX = convertBlockXtoX(tile.roomX + 1) + 1
charY = convertBlockYtoY(tile.roomY)
room = tile.room
updateBase()
```

---

## alignToFloor()

```javascript
charY = convertBlockYtoY(tile.roomY)
inJumpUp = false
```
