# Kid.js — Spec

**File:** `src/Kid.js` (1,763 lines)  
**Extends:** `PrinceJS.Fighter`  
**Purpose:** Player character — keyboard/pointer/gamepad input, movement state machine, all player-specific logic.

---

## Constructor: `PrinceJS.Kid(game, level, location, direction, room)`

```javascript
PrinceJS.Fighter.call(this, game, level, location, direction, room, "kid")
this.id = 0
```

**Additional Signals:**
```
onLevelFinished  — dispatched(currentLevel) on level exit
onNextLevel      — dispatched(currentLevel) after wait time
onRecoverLive    — dispatched() when HP recovered from potion
onAddLive        — dispatched() when max HP increased
onFlipped        — dispatched() when screen flips
```

**Additional instance variables:**
```
pickupSword         = false
pickupPotion        = false
allowCrawl          = true
charRepeat          = false
recoverCrop         = false
checkFloorStepFall  = false
backwardsFall       = 1          // becomes -1 when swordDrawn during startFall
ledgeSwing          = 0          // counts frame 92 in hang; used by checkLedgeSwing
grabWait            = false      // true for 500ms after grab, prevents immediate climbup
bumpTimer           = 0          // prevents repeat bump sounds; 10-tick cooldown
shadowFlashTimer    = 0          // 30-tick countdown for shadow overlay blink
opponentSync        = false
blockEngarde        = false
hasSword            = (PrinceJS.currentLevel > 1)   // no sword on level 1
```

**Health:**
```
maxHealth = PrinceJS.maxHealth
health    = PrinceJS.currentHealth || maxHealth
```

**Input devices:**
```
cursors  = game.input.keyboard.createCursorKeys()
shiftKey = game.input.keyboard.addKey(Phaser.Keyboard.SHIFT)
```

**Additional commands registered:**
```
0xFD → CMD_UP        room transition upward
0xFC → CMD_DOWN      room transition downward
0xF7 → CMD_IFWTLESS  float-mode action substitution
0xF5 → CMD_JARU      shake floor above (jump up impact)
0xF4 → CMD_JARD      shake floor below (jump down impact)
0xF3 → CMD_EFFECT    no-op
0xF1 → CMD_NEXTLEVEL level exit / next level logic
```

---

## Additional Command Implementations

### CMD_UP (0xFD)

```javascript
if (charBlockY === 0):
    charY += 189
    baseY -= 189
    charBlockY = 2
    room = rooms[room].links.up
    onChangeRoom.dispatch(room)
```

### CMD_DOWN (0xFC)

```javascript
if (charBlockY === 2 && charY > 189):
    charY -= 189
    baseY += 189
    charBlockY = 0
    changeRoomDown()
```

### CMD_IFWTLESS (0xF7)

```javascript
if (inFloat):
    "stepfall"  → action = "stepfloat"
    "bumpfall"  → action = "bumpfloat"
    "highjump"  → action = "superhighjump"
```

### CMD_TAP (0xF2 override)

```javascript
if action === "softLand": return   // no sound during soft land
if p1 === 1: play "Footsteps"
if p1 === 2: play "BumpIntoWallSoft"
```

### CMD_JARU (0xF5)

```javascript
level.shakeFloor(charBlockY - 1, room)
tile  = getTileAt(charBlockX, charBlockY - 1, room)
if tile.element === LOOSE_BOARD: tile.shake(true)
tileR = getTileAt(charBlockX + 1, charBlockY - 1, room)
if tileR.element === LOOSE_BOARD: tileR.shake(true)
```

### CMD_JARD (0xF4)

```javascript
level.shakeFloor(charBlockY, room)
```

### CMD_NEXTLEVEL (0xF1)

```javascript
PrinceJS.maxHealth = maxHealth
if currentLevel === 4:
    play "TheShadow"; waitTime = 9000
else if currentLevel NOT in [13, 14]:
    play "Prince"; waitTime = 13000
else:
    waitTime = 0
onLevelFinished.dispatch(currentLevel)
delayed(() => onNextLevel.dispatch(currentLevel), waitTime)
```

---

## updateActor() — Kid Execution Order

```
1.  updateTimer()
2.  updateSplash()
3.  updateBehaviour()      ← input/state machine, before processCommand
4.  processCommand()
5.  updateAcceleration()
6.  updateVelocity()
7.  checkFight()
8.  checkSpikes()
9.  checkChoppers()
10. checkBarrier()
11. checkButton()
12. checkFloor()
13. checkRoomChange()
14. updateCharPosition()
15. updateSwordPosition()
16. maskAndCrop()
```

---

## updateTimer()

```javascript
if bumpTimer > 0: bumpTimer--
if shadowFlashTimer === 1:
    hideShadowOverlay(); shadowFlashTimer--
if shadowFlashTimer > 1:
    shadowOverlay.visible = (shadowFlashTimer % 2 === 0)
    shadowFlashTimer--
```

---

## updateBehaviour() — Input State Machine

**Entry guard:** `if (x === 0 && y === 0) return` — not yet positioned.

**Key-release permission resets:**
```
!keyL() && faceL()  →  allowCrawl = allowAdvance = true
!keyR() && faceR()  →  allowCrawl = allowAdvance = true
!keyL() && faceR()  →  allowRetreat = true
!keyR() && faceL()  →  allowRetreat = true
!keyU()             →  allowBlock = true
!keyS()             →  allowStrike = true
```

**Switch on `this.action`:**

### "stand"
```
blockEngarde = false; ledgeSwing = 0
if !flee && canReachOpponent() && facingOpponent() && hasSword: tryEngarde()
if flee && keyS() && canReachOpponent() && facingOpponent() && hasSword: tryEngarde()
keyL() && faceR()                  → turn()
keyR() && faceL()                  → turn()
keyL() && keyU() && faceL()        → standjump()
keyR() && keyU() && faceR()        → standjump()
keyL() && keyS() && faceL()        → step()
keyR() && keyS() && faceR()        → step()
keyL() && faceL()                  → startrun()
keyR() && faceR()                  → startrun()
keyU()                             → jump()
keyD()                             → stoop()
keyS()                             → tryPickup()
```

### "startrun"
```
blockEngarde = false; charRepeat = false
frameID(1,3) && keyU()             → standjump()
frameID(4,6) && keyU()             → runjump()
else && keyU()                     → standjump()
```

### "running"
```
charRepeat = false
keyL() && faceR()                  → runturn()
keyR() && faceL()                  → runturn()
!keyL() && faceL()                 → runstop()
!keyR() && faceR()                 → runstop()
keyU()                             → runjump()
keyD()                             → rdiveroll()
```

### "turn"
```
blockEngarde = false; charRepeat = false
keyL() && faceL() && frameID(48)   → turnrun()
keyR() && faceR() && frameID(48)   → turnrun()
```

### "stoop"
```
charRepeat = false
pickupSword && frameID(109)        → gotSword()
pickupPotion && frameID(109)       → drinkPotion()
!keyD() && frameID(109)            → standup()
keyL() && faceL() && allowCrawl    → crawl()
keyR() && faceR() && allowCrawl    → crawl()
```

### "hang"
```
charRepeat = false
tile at (charBlockX,charBlockY) isBarrier() → action = "hangstraight"
tile above is LOOSE_BOARD: shake(true); if fallStarted(): startFall()
keyU() && !grabWait                → climbup()
!keyS()                            → startFall()
frameID(92)                        → ledgeSwing++
```

### "hangstraight"
```
charRepeat = false
keyU() && !grabWait                → climbup()
!keyS()                            → startFall()
```

### "jumpfall","rjumpfall","bumpfall","stepfall","freefall"
```
charRepeat = false
keyS()                             → tryGrabEdge()
```

### "engarde"
```
charRepeat = false
keyL() && faceL() && allowAdvance  → advance()
keyR() && faceR() && allowAdvance  → advance()
keyL() && faceR() && allowRetreat  → retreat()
keyR() && faceL() && allowRetreat  → retreat()
keyU() && allowBlock               → block()
keyS() && allowStrike              → strike()
keyD()                             → fastsheathe()
```

### "advance","blockedstrike"
```
charRepeat = false
keyU() && allowBlock               → block()
```

### "retreat","strike","block"
```
charRepeat = false
keyS() && allowStrike              → strike()
```

### "climbup","climbdown"
```
charRepeat = false
tile is LOOSE_BOARD && fallStarted()   → startFall()
frameID(142) && tile is SPACE          → startFall()
frameID(142) [climbup]                 → level.recheckCurrentRoom()
frameID(140) [climbdown]               → level.recheckCurrentRoom()
```

---

## tryEngarde()

```javascript
if !hasSword: return false
if blockEngarde: return false
dodgeChoppers()
level.recheckCurrentRoom()
engardeDistance = (!opponent.facingOpponent() && opponent.sneakUp) ? 35 : 100
if opponent && opponent.alive && opponentDistance() <= engardeDistance:
    return engarde()
return false
```

---

## inFallDistance(tile)

```javascript
if tile.element in [SPACE, TOP_BIG_PILLAR, TAPESTRY_TOP]: return true
if x === 0 || action NOT in ["runstop","runturn","runjump","standjump"]: return true
offsetX = faceL() ? 10 : -14
return Math.abs(tile.centerX - centerX + offsetX) >= 25
```

---

## tryGrabEdge()

```javascript
updateBlockXY()
if fallingBlocks > 2 && !inFloat: return

tileT  = getTileAt(charBlockX,         charBlockY - 1, room)
tileTF = getTileAt(charBlockX+charFace, charBlockY - 1, room)
tileTR = getTileAt(charBlockX-charFace, charBlockY - 1, room)

isInDistance =
    distanceToEdge() <= 10 + (action === "stepfall" ? 3 : 0)
    && (distanceToTopFloor() >= -50
        || (action in ["jumpfall","freefall"] && distanceToFloor() > -3))

// Primary: grab at forward tile
if tileTF.isWalkable()
   && tileT.element in [SPACE, TOP_BIG_PILLAR, TAPESTRY_TOP]
   && isInDistance
   && inGrabDistance(tileTF)
   && !(faceL() && tileTF.element === TAPESTRY):
    grab(charBlockX)

// Alternate: grab at current tile (backward)
else if tileT.isWalkable()
        && tileTR.element in [SPACE, TOP_BIG_PILLAR, TAPESTRY_TOP]
        && isInDistance
        && inGrabDistance(tileT, 20)
        && !(faceL() && tileT.element === TAPESTRY):
    grab(charBlockX - charFace)
```

---

## grab(x)

```javascript
updateBlockXY()
if faceL(): charX = convertBlockXtoX(x) - 3
if faceR(): charX = convertBlockXtoX(x + 1) + 1
charY = convertBlockYtoY(charBlockY)
charXVel = charYVel = 0
ledgeSwing = 0
stopFall()
updateBlockXY()
action = "hang"
play "BumpIntoWallHard"
processCommand()
// if tile above is LOOSE_BOARD: shake(true)
grabWait = true
delayed(() => grabWait = false, 500)
```

---

## inGrabDistance(tile, distance=30)

```javascript
offsetX = faceL() ? 2 : -5
return Math.abs(tile.centerX - centerX + offsetX) <= distance
```

---

## checkBarrier()

**Skip conditions:**  
- Not alive  
- Action in `["jumpup","highjump","jumphanglong","jumpbackhang","climbup","climbdown","climbfail","stand","turn","fastsheathe"]`  
- Action starts with "step" (except "hangdrop")  
- Action starts with "hang" AND action is not "hangdrop"  

**Freefall + double barrier:**
```javascript
if action === "freefall" && tile.isFreeFallBarrier() && tileT.isFreeFallBarrier():
    if moveL(): charX = convertBlockXtoX(charBlockX + 1) - 1
    else if moveR(): charX = convertBlockXtoX(charBlockX)
    updateBlockXY(); bump(); return
```

**Moving right into barrier (tile or tapestry behind):**
```javascript
if moveR() && (tile.isBarrier() || (centerX <= tileR.centerX && tileR.element in [TAPESTRY,TAPESTRY_TOP])):
    if tile.element === MIRROR: return
    if tile.intersects(charBounds) || (tile.intersectsAbs(charBoundsAbs) && !swordDrawn):
        if !tile.isBarrier(): charBlockX -= charFace
        if swordDrawn: charX = convertBlockXtoX(charBlockX) - 2
        else:          charX = convertBlockXtoX(charBlockX) + 5
        updateBlockXY(); bump()
```

**Next-block forward barrier check:**
```javascript
offsetX  = swordDrawn ? 12 * charFace : 0
blockX   = convertXtoBlockX(charX + charFdx * charFace - offsetX)
tileNext = getTileAt(blockX, charBlockY, room)

if tileNext.isBarrier():
    WALL:
        if !swordDrawn:
            if moveL(): charX = convertBlockXtoX(blockX + 1) - 1
            else if moveR(): charX = convertBlockXtoX(blockX)
            updateBlockXY()
        bump()
    GATE / TAPESTRY / TAPESTRY_TOP:
        if moveL() && intersects(charBounds):
            if action === "stand" && tile.element === GATE:
                charX = convertBlockXtoX(charBlockX) + 3; updateBlockXY()
            else if centerX - 8 > tileNext.centerX:
                charX = convertBlockXtoX(blockX + 1) - 1; updateBlockXY(); bump()

// Behind-tile mirror check:
tileNext = getTileAt(blockX - charFace, charBlockY, room)
if tileNext.element === MIRROR && moveL() && action !== "runjump":
    charX += 5; bump()
```

---

## getCharBoundsAbs()

```javascript
return new Phaser.Rectangle(x, y - height, width, height)
```

---

## bump()

```javascript
tile = getTileAt(charBlockX, charBlockY, room)

if tile.isSpace():
    charX -= 2 * charFace * backwardsFall
    bumpFall()
else:
    y = distanceToFloor()
    if y >= 5:
        bumpFall()
    else if frameID(24,25) || frameID(40,42) || frameID(102,106):
        charX -= 5 * charFace
        land()
    else:
        if !swordDrawn && !tile.isWalkable() && action !== "highjump":
            charX -= 5 * charFace
        if swordDrawn:
            if moveR(): charX -= 2
            else if moveL(): charX += 6
            else: charX += 5 * charFace
            bumpSound()
        else if action NOT in ["softland","medland"]:
            blockEngarde = true
            setBump()
            processCommand()
crop(null)
```

---

## setBump()

```javascript
bumpSound()
action = "bump"
alignToFloor()
```

---

## bumpSound()

```javascript
if bumpTimer === 0:
    play "BumpIntoWallSoft"
    bumpTimer = 10
```

---

## bumpFall()

```javascript
inFallDown = true
if actionCode === 4:   // already in freefall
    charX -= charFace * backwardsFall
    charXVel = 0
    ledgeSwing = 0
else:
    charX -= 2 * charFace * backwardsFall
    bumpSound()
    action = "bumpfall"
    processCommand()
```

---

## fastsheathe()

```javascript
flee = true
action = "fastsheathe"
swordDrawn = false
if opponent !== null:
    opponent.fastsheathe()
    opponent.refracTimer = 9
```

---

## prepareCheckFloor()

Shared setup for `checkButton()` and `checkFloor()`:

```javascript
if charFrame === 141: return { skip: true }   // skip on alternating chx frame

if action in ["hang","hangstraight"]
   || (action === "climbup"   && frameID(135,140))
   || (action === "climbdown" && frameID(91,140)):
    checkCharBlockY = charBlockY - 1

if action in ["hang","hangstraight","climbup","climbdown","runturn"]:
    checkCharFcheck = true

return { tile: getTileAt(checkCharBlockX, checkCharBlockY, room), checkCharFcheck }
```

---

## checkButton()

Active for `actionCode` 0, 1, 2, 5, 6, 7:
```javascript
if checkCharFcheck && tile:
    if tile.element in [RAISE_BUTTON, DROP_BUTTON]:
        tile.push()
```

---

## checkFloor()

Early exits:
- `action in ["climbup","climbdown"]` AND tile is NOT LOOSE_BOARD → return
- `action === "stoop"` AND tile is NOT SPACE → return
- `action === "strike"` → return
- `pickupPotion || pickupSword` → return

**actionCode 0, 1, 5, 7 (stand/run/bump/turn):**
```javascript
inFallDown = false
if checkCharFcheck:
    FLOOR (hidden): unhide and revalidate tile
    SPACE / TOP_BIG_PILLAR / TAPESTRY_TOP:
        if !alive: return
        if inFallDistance(tileR): startFall()
        else if tileR is LOOSE_BOARD: shake(true)
        else if tileR is SPIKES: raise()
        else if tileR is RAISE_BUTTON or DROP_BUTTON: push()
    LOOSE_BOARD: shake(true)
    SPIKES:
        if inSpikeDistance(tile):
            if (state != FULL_OUT && action in ["running","runjump","runturn"])
               || action === "softland"
               || (action === "medland" && frameID(108,109))
               || (action === "standjump" && frameID(26,28)):
                play "SpikedBySpikes"; alignToTile(tile); dieSpikes()
        tile.raise()
```

**actionCode 3 (stepfall):**
```javascript
inFallDown = true
if !checkFloorStepFall: return
checkFloorStepFall = false
checkFall(tile)
checkLedgeSwing()
```

**actionCode 4 (freefall):**
```javascript
inFallDown = true
checkFall(tile)
checkLedgeSwing()
```

---

## checkLedgeSwing()

```javascript
if ledgeSwing >= 4:
    charX += (inFloat ? 2.0 : 1.5) * charFace
```

---

## checkRoomChange()

Skip on frames: `[16,17,27,28,47,48,49,50,51,61,62,76,77,116,117,125,126,127,128,157]`

```javascript
footX      = charX + charFdx * charFace
footBlockX = convertXtoBlockX(footX)

// Right exit:
if footBlockX >= 9 && (moveR(false) || action === "bump"):
    cameraRoom = room
    if footX > 142 || (swordDrawn && (footX > 130 || footX < 0)):
        cameraRoom = links.right
    if action NOT in ["climbup","climbdown","stand","jumpup"]:
        onChangeRoom.dispatch(room, cameraRoom)

// Left exit approach:
if moveL(false) && (footBlockX === 8 || (footBlockX === 9 && footX < 135)):
    if action NOT in ["climbup","climbdown","stand","jumpup"]:
        onChangeRoom.dispatch(room)

// Falling down into next room:
if charY > 189:
    charY -= 189
    baseY += 189
    changeRoomDown()
```

---

## changeRoomDown()

```javascript
if rooms[room]:
    room = links.down
    if room > 0: this.room = room
    else if charBlockX >= 9:
        room = links.right → links.down
        if room > 0: charX -= 140; baseX += 320; charBlockX = 0
        this.room = room
    else if charBlockX <= 0:
        room = links.left → links.down
        if room > 0: charX += 140; baseX -= 320; charBlockX = 9
        this.room = room
    onChangeRoom.dispatch(room)
```

---

## maskAndCrop()

```javascript
// Climbing frames — append "r" suffix to frameName when facing right
if faceR() && charFrame > 134 && charFrame < 145:
    frameName += "r"

// Mask tile above when hanging (faceR)
if faceR() && action.startsWith("hang"):
    level.maskTile(charBlockX, charBlockY - 1, room, this)
else if faceR() && action === "jumphanglong" && frameID(79):
    level.maskTile(charBlockX, charBlockY - 1, room, this)
else if faceR() && action === "jumpbackhang" && frameID(79):
    level.maskTile(charBlockX, charBlockY - 1, room, this)
else if faceR() && action === "climbdown" && frameID(91):
    level.maskTile(charBlockX, charBlockY - 1, room, this)

// Unmask on specific frames
if frameID(15) || frameID(158) || frameID(185) || (faceL() && action === "hang"):
    level.unMaskTile(this)

// Jump-up crop (hides feet appearing through ceiling)
if recoverCrop:
    crop(null); recoverCrop = false

if inJumpUp && frameID(78,79):
    crop(new Phaser.Rectangle(0, 7, -width * charFace, height))

if inJumpUp && frameID(81):
    crop(null)
    crop(new Phaser.Rectangle(0, 3, -width * charFace, height))
    inJumpUp = false
    recoverCrop = true
```

---

## tryPickup()

```javascript
tile  = getTileAt(charBlockX, charBlockY, room)
tileF = getTileAt(charBlockX + charFace, charBlockY, room)

pickupSword   = tile.element === SWORD || tileF.element === SWORD
pickupPotion  = tile.element === POTION || tileF.element === POTION

if pickupPotion || pickupSword:
    if faceR():
        if tileF.element in [POTION, SWORD]: charBlockX++
        charX = convertBlockXtoX(charBlockX) + 1 * pickupPotion
    if faceL():
        if tile.element in [POTION, SWORD]: charBlockX++
        charX = convertBlockXtoX(charBlockX) - 3
    action = "stoop"
    allowCrawl = false
```

---

## Input Methods

### Keyboard + Pointer + Gamepad

```javascript
keyL() = cursors.left.isDown || pointerL() || gamepadLeftPressed(game)
keyR() = cursors.right.isDown || pointerR() || gamepadRightPressed(game)
keyU() = cursors.up.isDown || pointerU() || gamepadUpPressed(game)
keyD() = cursors.down.isDown || pointerD() || gamepadDownPressed(game)
keyS() = shiftKey.isDown || pointerS() || gamepadActionPressed(game)
```

### Touch/Pointer Zones

Screen divided into a 3×3 grid. Positions adjusted for screen-flip mode.

```
pointerL(): x in [0, 1/3 * width],           y in [0–96% height] (or 4–100% flipped)
pointerR(): x in [2/3 * width, width],        y in [0–96% height]
pointerU(): x in [0, width],                  y in [0, 1/3 * height]
pointerD(): x in [0, width],                  y in [2/3 * height, 96% height]
pointerS(): x in [(0.5+bias)/3, (2.5-bias)/3] * width
            y in [(0.5+bias)/3, (2.5-bias)/3] * height
            where bias = swordDrawn ? 0.5 : 0
```

---

## Movement Action Methods

### turn()
```javascript
if !hasSword || !canReachOpponent(false, true): action = "turn"
else if hasSword && canReachOpponent(false,true) && !facingOpponent() && !nearBarrier():
    action = "turndraw"
    flee = false
    if !swordDrawn: play "UnsheatheSword"
    swordDrawn = true
else: action = "turn"
```

### standjump() — `action = "standjump"`, calls `syncShadow()`

### startrun()
```javascript
if nearBarrier(): return step()
action = "startrun"; syncShadow()
```

### runturn() — `action = "runturn"`

### turnrun()
```javascript
if nearBarrier(): step(); charX -= 2 * charFace; return
action = "turnrun"
```

### runjump() — `action = "runjump"`, calls `syncShadow()`

### rdiveroll() — `action = "rdiveroll"`, `allowCrawl = false`

### standup() — `action = "standup"`, `allowCrawl = true`

### crawl() — `action = "crawl"`, `allowCrawl = false`

### runstop()
```javascript
if frameID(7) || frameID(11):
    action = "runstop"; syncShadow()
```

---

## step()

Complex method. Calculates `px` (pixel step distance 1–14) then sets `action = "step" + Math.min(px, 14)`.

**Default `px = 11`**

**Overrides:**
- Chopper ahead: `px = distanceToEdge() - 4 - (faceL() ? 1 : 0)`, min 1
- Mirror ahead: `px = distanceToEdge() - 8`; if `px <= 0`: `bump(); return`
- nearBarrier or tileF is space/potion/loose/button/sword:
  - `px = distanceToEdge()`
  - Gate fast-dropping or tapestry ahead (faceR): `px -= 6`; if `<= 0`: `setBump(); return`
  - tileF is POTION or SWORD and not near barrier and `px === 0`: `px = 11`
  - tileF is a barrier and `px - 2 <= 0`: `setBump(); return`
  - `px === 0` and tileF is loose/button/space:
    - if `charRepeat` or tileF is button: `charRepeat = false; px = 11`
    - else: `charRepeat = true; action = "testfoot"; return`

If `px > 0`: `action = "step" + Math.min(px, 14)`, call `syncShadow()`.

---

## jump()

Decision tree for which jump action to trigger:
```javascript
if tile.isExitDoor():
    if tile.open: return climbstairs()

if faceL() && tile.element === MIRROR && abs(tile.x - x) < 30:
    return bump()

if tileTF.element === MIRROR:
    return jumpup()

if checkJump(tileT) && checkClimbable(tileTF):
    return jumphanglong()

if checkClimbable(tileT) && checkJump(tileTR) && tileR.isWalkable():
    if faceL() && convertBlockXtoX(charBlockX+1) - charX < 11:
        charBlockX++; return jumphanglong()
    if faceR() && charX - convertBlockXtoX(charBlockX) < 9:
        charBlockX--; return jumphanglong()
    return jumpup()

if checkClimbable(tileT) && checkJump(tileTR):
    if faceL() && convertBlockXtoX(charBlockX+1) - charX < 11:
        return jumpbackhang()
    if faceR() && charX - convertBlockXtoX(charBlockX) < 9:
        return jumpbackhang()
    return jumpup()

if tileT.isSpace(): return highjump()
jumpup()
```

**Tile checks used:**
- `tile`  = `getTileAt(charBlockX, charBlockY, room)`
- `tileT` = `getTileAt(charBlockX, charBlockY-1, room)`  (above)
- `tileTF`= `getTileAt(charBlockX+charFace, charBlockY-1, room)`  (above + forward)
- `tileTR`= `getTileAt(charBlockX-charFace, charBlockY-1, room)`  (above + backward)
- `tileR` = `getTileAt(charBlockX-charFace, charBlockY, room)`    (backward)

### checkJump(tile)
```javascript
(faceL() && tile.isSpace()) || (faceR() && tile.isJumpSpace()) || tile.hidden
```

### checkClimbable(tile)
```javascript
tile.isWalkable() && (faceR() || tile.element !== TAPESTRY)
```

---

## jumpup() — `action = "jumpup"`, `inJumpUp = true`

## highjump()
```javascript
tileTR = getTileAt(charBlockX - charFace, charBlockY - 1, room)
action = "highjump"
if faceL() && tileTR.isWalkable():
    level.maskTile(charBlockX + 1, charBlockY - 1, room, this)
```

## jumpbackhang()
```javascript
if faceL(): charX = convertBlockXtoX(charBlockX) + 7
else:        charX = convertBlockXtoX(charBlockX) + 6
action = "jumpbackhang"
if faceR(): level.maskTile(charBlockX, charBlockY - 1, room, this)
```

## jumphanglong()
```javascript
if faceL(): charX = convertBlockXtoX(charBlockX) + 1
else:        charX = convertBlockXtoX(charBlockX) + 12
action = "jumphanglong"
if faceR(): level.maskTile(charBlockX + 1, charBlockY - 1, room, this)
```

---

## stoop()

```javascript
tileR = getTileAt(charBlockX - charFace, charBlockY, room)
if tileR.element in [SPACE, TOP_BIG_PILLAR]
   || (faceL() && tileR.element === TAPESTRY_TOP):
    if faceL() && charX - convertBlockXtoX(charBlockX) > 4:  return climbdown()
    if faceR() && charX - convertBlockXtoX(charBlockX) < 9:  return climbdown()
action = "stoop"
```

---

## climbdown()

```javascript
blockEngarde = false
tile = getTileAt(charBlockX, charBlockY, room)
if faceL() && tile.element === GATE && (state === FAST_DROPPING || !canCross(15)):
    charX = convertBlockXtoX(charBlockX) + 3
else:
    if faceL(): charX = convertBlockXtoX(charBlockX) + 6
    else:        charX = convertBlockXtoX(charBlockX) + 7
    action = "climbdown"
```

---

## climbup()

```javascript
blockEngarde = false
tileT = getTileAt(charBlockX, charBlockY - 1, room)
if faceL() && tileT.element === GATE && (state === FAST_DROPPING || !canCross(15)):
    action = "climbfail"
else:
    action = "climbup"
    if faceR(): level.unMaskTile(this)
if tileT.element === LOOSE_BOARD: tileT.shake(true)
```

---

## climbstairs()

```javascript
tile = getTileAt(charBlockX, charBlockY, room)
if tile.element === EXIT_RIGHT: charBlockX--
else: tile = getTileAt(charBlockX + 1, charBlockY, room)
if faceR(): charFace *= -1; scale.x *= -1   // force face left
charX = convertBlockXtoX(charBlockX) + 3
tile.mask()
action = "climbstairs"
```

---

## land() (override)

```javascript
charY = convertBlockYtoY(charBlockY)
charXVel = charYVel = 0
ledgeSwing = 0
fallingBlocks = inFloat ? 0 : this.fallingBlocks
stopFall()

tile = getTileAt(charBlockX, charBlockY, room)
if tile.element === SPIKES:
    play "SpikedBySpikes"; alignToTile(tile); dieSpikes()
else if alive:
    switch fallingBlocks:
        0,1: action = PrinceJS.danger ? "medland" : "softland"
             play PrinceJS.danger ? "MediumLandingOof" : "SoftLanding"
        2:   action = "medland"
             play "MediumLandingOof"; damageLife(true)
        3+:  play "FreeFallLand"; die("falldead")

alignToFloor()
processCommand()
level.unMaskTile(this)
PrinceJS.danger = false
level.recheckCurrentRoom()
```

---

## startFall() (override)

```javascript
if action in ["turn","turnrun","turnengarde","highjump","hangdrop"]:
    checkFloorStepFall = true

fallingBlocks = Math.min(0, fallingBlocks)
inFallDown = true
backwardsFall = swordDrawn ? -1 : 1

if action.startsWith("hang"):
    // hangdrop from ledge
    blockX = charBlockX
    if action === "hangstraight": blockX -= charFace
    tile = getTileAt(blockX, charBlockY, room)
    if tile.element NOT in [SPACE, TOP_BIG_PILLAR, TAPESTRY_TOP]:
        tile = getTileAt(charBlockX, charBlockY, room)
        if tile.isBarrier(): charX -= 7 * charFace
        action = "hangdrop"
        stopFall()
    else:
        tile = getTileAt(charBlockX, charBlockY, room)
        if tile.isBarrier(): charX -= 7 * charFace
        action = "hangfall"
        level.maskTile(charBlockX - charFace, charBlockY, room, this)
        processCommand()
else:
    act = "stepfall"
    if frameID(44): act = "rjumpfall"
    if frameID(26): act = "jumpfall"
    if frameID(13): act = "stepfall2"

    if distanceToEdge() <= 5 && action in ["running","runstop"] || action.startsWith("step"):
        charX -= 7 * charFace

    tile = getTileAt(charBlockX, charBlockY, room)
    dx = tile.isWalkable() ? 10 : 5
    if action === "retreat" || swordDrawn:
        charX += dx * charFace * (action === "advance" ? 1 : -1)
        level.maskTile(charBlockX + charFace, charBlockY, room, this)
    else:
        level.maskTile(charBlockX + 1, charBlockY, room, this)
    swordDrawn = false
    action = act
    processCommand()
```

---

## HP Methods

### damageLife(crouch=false)
```javascript
if !alive: return
showSplash(); flashRedDamage(game)
if crouch: splash.y = -5
if health > 1: health -= 1; onDamageLife.dispatch(1)
else: die()
```

### stealLife()
```javascript
if health > 1:
    damage = health - 1; health = 1; onDamageLife.dispatch(damage)
```

### recoverLife()
```javascript
if health < maxHealth: health++; onRecoverLive.dispatch()
```

### addLife()
```javascript
if maxHealth < 10: maxHealth++
health = maxHealth
onAddLive.dispatch()
```

---

## drinkPotion()

```javascript
pickupPotion = false
tile = getTileAt(charBlockX + charFace, charBlockY, room)
if tile.element !== POTION: tile = getTileAt(charBlockX, charBlockY, room)
if tile.element !== POTION: allowCrawl = true; return

play "DrinkPotionGlugGlug"
action = "drinkpotion"
potionType = tile.modifier
level.removeObject(tile.roomX, tile.roomY, tile.room)
if tile.isSpecial: return   // special potions (cutscene) handled elsewhere

delayed(() => {
    POTION_RECOVER: flashRedPotion(game); play "Potion1"; recoverLife()
    POTION_ADD:     flashRedPotion(game); play "Potion2"; addLife()
    POTION_BUFFER:  flashGreenPotion(game); floatFall()
    POTION_FLIP:    flipScreen()
    POTION_DAMAGE:  flashRedDamage(game); play "StabbedByOpponent"; damageLife()
    allowCrawl = true
}, 1000)
```

---

## gotSword()

```javascript
pickupSword = false
allowCrawl = true
action = "pickupsword"
flashYellowSword(game)
play "Victory"
level.removeObject(charBlockX + charFace, charBlockY, room)
hasSword = true
```

---

## floatFall()

```javascript
if inFloatTimeoutCancel !== null:
    inFloatTimeoutCancel(); inFloatTimeoutCancel = null
inFloat = true
play "Float"
handle = delayedCancelable(() => {
    inFloat = false; inFloatTimeoutCancel = null; fallingBlocks = 0
}, 18000)
inFloatTimeoutCancel = handle.cancel
```

Float lasts 18 seconds unless cancelled by another potion.

---

## flipScreen()

```javascript
PrinceJS.Utils.toggleFlipScreen()
onFlipped.dispatch()
```

---

## damageStruck()

Called when a loose board falls on Kid:
```javascript
if !alive: return
if action.includes("land"): return
if fallingBlocks < 2: fallingBlocks = 2
if !inFallDown: land()
```

---

## proceedOnDead()

```javascript
delayed(() => {
    if opponent && opponent.baseCharName === "jaffar":
        play "HeroicDeath"
    else:
        play "Accident"
    onDead.dispatch()
}, 1000)
```

---

## Shadow Overlay Methods

### showShadowOverlay()
```javascript
shadowOverlay = make.sprite(0, 0, "shadow", "shadow-15")
shadowOverlay.anchor.setTo(0, 1)
addChild(shadowOverlay)
delegate = {
    syncFrame: (actor) => shadowOverlay.frameName = "shadow-" + actor.frameName.split("-")[1],
    syncFace:  (actor) => shadowOverlay.charFace = actor.charFace
}
```

### hideShadowOverlay()
```javascript
shadowOverlay.visible = false
delegate = null
```

### flashShadowOverlay()
```javascript
shadowFlashTimer = 30
```

---

## syncShadow()

```javascript
if opponentSync && opponent && opponent.charName === "shadow"
   && !opponent.active && opponent.charFace !== charFace:
    opponent.action = this.action
```

---

## mergeShadowPosition()

```javascript
shadow = opponent
opponent = null; shadow.opponent = null
shadow.action = "stand"; shadow.setInvisible()
charX  = shadow.charX;  charY  = shadow.charY
charFdx = shadow.charFdx; charFdy = shadow.charFdy
charFood = shadow.charFood
if charFace !== shadow.charFace: changeFace()
updateCharPosition(); updateBlockXY()
```
