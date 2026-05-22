# Enemy.js — Spec

**File:** `src/Enemy.js` (456 lines)  
**Extends:** `PrinceJS.Fighter`  
**Purpose:** AI-controlled guards, skeleton, shadow, Jaffar. Handles probability-based combat decisions, sight/patrol logic, and guard-specific behaviour.

---

## Constructor: `PrinceJS.Enemy(game, level, location, direction, room, skill, color, key, id)`

```javascript
baseCharName = key
if key === "guard": key = "guard-" + color   // select coloured guard atlas
PrinceJS.Fighter.call(this, game, level, location, direction, room, key,
    key === "shadow" ? "shadow" : "fighter")

this.id = id
this.charX += direction * 7   // offset from spawn block
```

**Probability tables** (indexed by `skill` 0–11):
```
strikeProbability      = applyStrength(STRIKE_PROBABILITY[skill])
restrikeProbability    = applyStrength(RESTRIKE_PROBABILITY[skill])
blockProbability       = applyStrength(BLOCK_PROBABILITY[skill])
impairblockProbability = applyStrength(IMPAIRBLOCK_PROBABILITY[skill])
advanceProbability     = applyStrength(ADVANCE_PROBABILITY[skill])
```

**Timers:**
```
refracTimer  = 0    — refractory period after taking damage (prevents instant retaliation)
blockTimer   = 0    — grace period after blocking a strike
strikeTimer  = 0    — cooldown after receiving a strike (prevents advance)
```

**Other:**
```
lookBelow  = false   — whether to detect opponent on level below
startFight = false   — set true after proximity delay expires
charSkill  = skill
charColor  = color
health     = EXTRA_STRENGTH[skill] + STRENGTH[level.number]
```

**Signal subscriptions (auto-wired):**
```
onDamageLife    → resetRefracTimer
onStrikeBlocked → resetBlockTimer
onEnemyStrike   → resetStrikeTimer
```

If `charColor > 0`: `tintSplash(COLOR[charColor - 1])`.

---

## Static Probability/Strength Tables

All arrays indexed by `skill` (0–11):

```javascript
STRIKE_PROBABILITY      = [61, 100,  61,  61,  61,  40, 100, 150,   0,  48,  32,  48]
RESTRIKE_PROBABILITY    = [ 0,   0,   0,   5,   5, 175,  16,   8,   0, 255, 255, 150]
BLOCK_PROBABILITY       = [ 0, 150, 150, 200, 200, 255, 200, 250,   0, 255, 255, 255]
IMPAIRBLOCK_PROBABILITY = [ 0,  61,  61, 100, 100, 145, 100, 250,   0, 145, 255, 175]
ADVANCE_PROBABILITY     = [255, 200, 200, 200, 255, 255, 200,   0,   0, 255, 100, 100]
REFRAC_TIMER            = [16,  16,  16,  16,   8,   8,   8,   8,   0,   8,   0,   0]
EXTRA_STRENGTH          = [ 0,   0,   0,   0,   1,   0,   0,   0,   0,   0,   0,   0]
```

Health by level (indexed by level number 0–15):
```javascript
STRENGTH = [4, 3, 3, 3, 3, 4, 5, 4, 4, 5, 5, 5, 4, 6, 10, 0]
```

Guard tint colours (indexed by `charColor - 1`):
```javascript
COLOR = [0x4890fc, 0xa83000, 0xfc5000, 0x0c9000, 0x5a00fc, 0xc858fc, 0xfcfc00]
```

---

## updateActor() — Enemy Execution Order

```
1. updateSplash()
2. updateBehaviour()
3. processCommand()
4. updateAcceleration()
5. updateVelocity()
6. checkFight()
7. checkSpikes()
8. checkChoppers()
9. checkBarrier()
10. checkButton()
11. checkFloor()
12. checkRoomChange()
13. updateCharPosition()
14. updateSwordPosition()
15. maskAndCrop()
```

---

## CMD_TAP (0xF2 override)

Only plays sounds if `charName === "shadow"` AND `visible`:
```javascript
if action === "softLand": return
if p1 === 1: play "Footsteps"
if p1 === 2: play "BumpIntoWallHard"
```

---

## updateBehaviour()

```javascript
if opponent === null || !alive || !opponent.alive: return

if willStartFight():
    delayed(() => {
        if willStartFight(): startFight = true
    }, baseCharName === "jaffar" ? 2300 : 500)

// Countdown timers each tick:
if refracTimer > 0: refracTimer--
if blockTimer > 0:  blockTimer--
if strikeTimer > 0: strikeTimer--

// Skip AI during these actions:
if action in ["stabbed","stabkill","dropdead","stepfall"]: return

distance = opponentDistance()
if distance === -999: return   // opponent not reachable at all

if swordDrawn:
    distance >= 35:   oppTooFar(distance)
    distance < -20:   turnengarde()
    distance < 12:    oppTooClose(distance)
    else:             oppInRange(distance)
else:
    if canReachOpponent(lookBelow) || canSeeOpponent(lookBelow):
        if !sneakUp || facingOpponent(): engarde()
```

---

## willStartFight()

```javascript
return active
    && !startFight
    && (opponentCloseRoom(opponent, room) || Math.abs(opponentDistance()) < 35)
```

---

## enemyAdvance()

```javascript
if !startFight: return
if !canReachOpponent(lookBelow) && !canSeeOpponent(lookBelow):
    swordDrawn = false; stand(); return

tile = getTileAt(charBlockX, charBlockY, room)
if tile.isSpace() && action NOT in ["advance","retreat","strike"]:
    startFall(); return

tileF = getTileAt(charBlockX + charFace, charBlockY, room)
tileR = getTileAt(charBlockX - charFace, charBlockY, room)
if canWalkSafely(tileF, true): advance()
else if canWalkSafely(tileR): retreat()
```

---

## canWalkSafely(tile, below=false)

```javascript
if tile.isSafeWalkable(): return true
if below && tile.isSpace() && tile.roomY < 2 && opponent.charBlockY === tile.roomY + 1:
    tile = getTileAt(tile.roomX, tile.roomY + 1, tile.room)
    return tile.isSafeWalkable()
return false
```

---

## engarde() (override)

```javascript
if !hasSword || !startFight: return
lookBelow = true
PrinceJS.Fighter.prototype.engarde.call(this)
```

---

## retreat() (override)

```javascript
if !canReachOpponent(lookBelow): return
if nearBarrier(charBlockX, charBlockY, true): return
if !opponent.opponent: opponent.opponent = this

// Auto-turn if facing same direction while in combat stance
if !action.includes("turn") && !opponent.action.includes("turn")
   && !facingOpponent() && charFace === opponent.charFace && opponentOnSameLevel():
    turnengarde(); return

tileR = getTileAt(charBlockX - charFace, charBlockY, room)
if canWalkSafely(tileR): PrinceJS.Fighter.prototype.retreat.call(this)
```

---

## advance() (override)

```javascript
if !canReachOpponent(lookBelow): return
if nearBarrier(charBlockX, charBlockY, true): return
if !opponent.opponent: opponent.opponent = this

tileF  = getTileAt(charBlockX + charFace, charBlockY, room)
tileFF = getTileAt(charBlockX + 2 * charFace, charBlockY, room)
if canWalkSafely(tileF, true) && (!opponent.isHanging() || canWalkSafely(tileFF, true)):
    PrinceJS.Fighter.prototype.advance.call(this)
```

---

## Combat Decision Methods

### oppTooFar(distance) — distance >= 35

```javascript
if refracTimer !== 0: return
if opponent.action === "running" && distance < 40: strike(); return
if opponent.action === "runjump" && distance < 50: strike(); return
enemyAdvance()
```

### oppTooClose() — distance < 12

```javascript
if charFace === opponent.charFace || opponent.action NOT in ["engarde","advance","retreat"]:
    retreat()
else:
    advance()
```

### oppInRange(distance) — 12 <= distance < 35

```javascript
if !opponent.swordDrawn:
    if refracTimer === 0:
        if distance <= 25: strike()
        else: advance()
else:
    oppInRangeArmed(distance)
```

### oppInRangeArmed(distance)

```javascript
if !opponentOnSameLevel(): return
if distance < 10 || distance >= 28:
    tryAdvance()
else:
    tryBlock()
    if refracTimer === 0:
        if distance < 12: tryAdvance()
        else: tryStrike()
```

---

## Probability-Based Action Methods

### tryAdvance()
```javascript
if charSkill === 0 || strikeTimer === 0:
    if advanceProbability > rnd.between(0, 254): advance()
```

### tryBlock()

Triggers when opponent is on wind-up frames (152-153, 162, 2-3, 12):
```javascript
if blockTimer !== 0:
    if impairblockProbability > rnd.between(0, 254): block()
else:
    if blockProbability > rnd.between(0, 254): block()
```

### tryStrike()

Skips if opponent is on recovery frames (169, 151, 19, 1):
```javascript
if frameID(150):   // enemy on engarde-ready frame
    if restrikeProbability > rnd.between(0, 254): strike()
else:
    if strikeProbability > rnd.between(0, 254): strike()
```

---

## Timer Resets (signal callbacks)

```javascript
resetRefracTimer(): refracTimer = REFRAC_TIMER[charSkill]
resetBlockTimer():  blockTimer = 4
resetStrikeTimer(): strikeTimer = 15
```

---

## fastsheathe() (override)

Only applies to shadow:
```javascript
if charName === "shadow":
    setInactive()
    action = "fastsheathe"
    swordDrawn = false
```

---

## Visibility / Activity Control

```javascript
setVisible():   visible = true; sword.visible = true
setInvisible(): visible = false; sword.visible = false

setActive():
    setVisible(); active = true
    if charName === "skeleton": action = "arise"

setInactive():
    active = false; startFight = false
    if charName === "skeleton": action = "laydown"
```

---

## checkBarrier() (override)

Skip if: `!alive`, `charName === "shadow"`, or `action in ["stand","turn","stepfall","freefall"]`.

```javascript
tile = getTileAt(charBlockX, charBlockY, room)
if moveR() && tile.isBarrier():
    if tile.intersects(getCharBounds()): bump(tile)
else:
    blockX   = convertXtoBlockX(charX + charFdx * charFace - 12)
    tileNext = getTileAt(blockX, charBlockY, room)
    if tileNext.isBarrier():
        WALL:                            bump(tileNext)
        GATE/TAPESTRY/TAPESTRY_TOP:
            if tileNext.intersects(getCharBounds()): bump(tileNext)
```

---

## bump(tile) (override)

Enemy-specific (simpler than Kid):
```javascript
if moveR():   charX -= 2
else if moveL(): charX += 10
else:         charX += 5 * (centerX > tile.centerX ? 1 : -1)
updateBlockXY()
action = "bump"
```

---

## appearOutOfMirror(mirror)

Used in levels with mirror encounters:
```javascript
charX    = convertBlockXtoX(mirror.roomX) + 20
charY    = convertBlockYtoY(mirror.roomY) - 14
action   = "runjumpdown"
charFrame = 42
updateBlockXY()
updateCharPosition()
processCommand()
setVisible()
```
