# Game.js — Spec

**File:** `src/Game.js` (859 lines)  
**Purpose:** Main game state. Owns the game loop (80ms fixed tick), level/character setup, camera, opponent assignment, level-specific scripted events.

---

## Instance Variables

```
kid              — PrinceJS.Kid instance
level            — PrinceJS.Level instance
ui               — PrinceJS.Interface instance
currentRoom      — room number kid is currently in
enemies          = []
shadow           — reference to shadow Enemy (or null)
mouse            — reference to Mouse actor (or null)
currentCameraRoom
visitedRooms     = {}    // keyed by room number, true when visited
blockCamera      = false // prevents camera updates (used level 6 exit)
continueTimer    = -1
pressButtonToContinueTimer = -1
pressButtonToNext = false
firstUpdate      = true
specialEvents    — from json.prince.specialEvents (default true)
playDanger       — from json.prince.danger (default true)
```

---

## preload()

Loads level-specific music not in Preloader:
```
"TheShadow"   assets/music/15_The_Shadow.mp3
"Float"       assets/music/16_Float.mp3
"Jaffar2"     assets/music/19_Jaffar_2.mp3
"JaffarDead"  assets/music/20_Jaffar_Dead.mp3
"HeroicDeath" assets/music/13_Heroic_Death.mp3
```

Level JSON path:
```javascript
levelBasePath = currentLevel < 90 ? "assets/maps/" : "assets/maps/custom/"
load.json("level", levelBasePath + "level" + currentLevel + ".json")
```

---

## create()

```javascript
game.sound.stopAll()
continueTimer = pressButtonToContinueTimer = -1
pressButtonToNext = false

// Start timer if not already running (i.e. new game, not resume)
if !PrinceJS.startTime:
    date = new Date()
    date.setMinutes(date.getMinutes() - (60 - PrinceJS.minutes))
    PrinceJS.startTime = date

json = cache.getJSON("level")
if !json: restartGame(); return

level = new LevelBuilder(game, this).buildFromJSON(json)
specialEvents = json.prince.specialEvents !== false
playDanger    = json.prince.danger !== false
```

**Enemy creation (from json.guards array):**
```javascript
for each data in json.guards:
    enemy = new PrinceJS.Enemy(game, level,
        data.location + (data.bias || 0),
        data.direction * (data.reverse || 1),
        data.room, data.skill, data.colors, data.type, i+1)
    if data.visible === false: enemy.setInvisible()
    if data.active === false:  enemy.setInactive()
    if data.sneak === false:   enemy.setSneakUp(false)
    enemy.onInitLife → ui.setOpponentLive
    enemies.push(enemy)
    if enemy.charName === "shadow": shadow = enemy
```

**Kid creation:**
```javascript
turn      = json.prince.turn !== false
direction = json.prince.direction * (json.prince.reverse || 1)
if turn: direction = -direction

kid = new PrinceJS.Kid(game, level,
    json.prince.location + (json.prince.bias || 0),
    direction, json.prince.room)

if typeof json.prince.sword === "boolean": kid.hasSword = json.prince.sword
if turn: kid.charX += 7; delayed(() => kid.action = "turn", 100)
kid.charX += json.prince.offset || 0
```

**Signal wiring:**
```
game.onPause   → onPause
game.onResume  → onResume
kid.onChangeRoom  → changeRoom
kid.onDead        → handleDead
kid.onFlipped     → handleFlipped
kid.onNextLevel   → nextLevel
kid.onLevelFinished → levelFinished
```

**Setup:**
```javascript
PrinceJS.Tile.Gate.reset()
visitedRooms = {}
currentRoom  = json.prince.room
blockCamera  = false

world.sort("z"); world.alpha = 1

ui = new PrinceJS.Interface(game, this)
ui.setPlayerLive(kid)
setupCamera(json.prince.room, json.prince.cameraRoom)

// 80ms fixed game loop tick:
game.time.events.loop(80, updateWorld, this)

// Keyboard shortcuts (require Ctrl or Shift held):
R → restartGameEvent    // Ctrl+R or Shift+R
A → restartLevelEvent   // Ctrl+A or Shift+A
L → nextLevelEvent      // Ctrl+L or Shift+L (levels 1-3 and 90+ only)
SPACEBAR → showRemainingMinutes

firstUpdate = true
if PrinceJS.danger === null:
    PrinceJS.danger = (level.number === 1 && playDanger)
PrinceJS.Utils.resetFlipScreen()
PrinceJS.Utils.updateQuery()
```

---

## update() — Phaser render tick

Handles UI interaction (touch bar at top/bottom edge, gamepad navigation):

```javascript
if continueGame(game):
    buttonPressed()
    // Bottom edge touch (or top edge when flipped):
    if pointer in UI strip:
        if isRemainingMinutesShown() || isLevelShown():
            left 20%  → previousLevel(currentLevel, true)
            right 20% → nextLevel(currentLevel, true, true)
            center     → restartLevel(true)
        anywhere → showRemainingMinutes()
    // Gamepad:
    info button: if minutes/level shown → restartLevel, else showRemainingMinutes
    previous:    previousLevel
    next:        nextLevel(currentLevel, true, true)
```

---

## updateWorld() — 80ms game tick

Called by Phaser timer loop:
```javascript
level.update()
kid.updateActor()
for each enemy: enemy.updateActor()
if mouse: mouse.updateActor()
checkLevelLogic()
ui.updateUI()
checkTimers()
firstUpdate = false
```

---

## checkLevelLogic()

Per-level scripted events. Only runs if `specialEvents === true`.

### Level 1
- `firstUpdate`: fire event 8 (DROP_BUTTON — closes starting gate)
- if `danger`: play "Danger" after 800ms

### Level 2
- `firstUpdate`: adjust specific enemy charX by -12 (room 24, blockX=0, blockY=1)

### Level 3 (skeleton)
- When exit open, kid in skeleton's room, distance reachable: activate skeleton from tile
- Special skeleton room-3 repositioning (delayed 100ms):
  - Face right, teleport to room=3, charX=convertBlockXtoX(4), charY=convertBlockYtoY(1), action="stand"
  - Kid sheathes sword
- If skeleton walks off ledge in room 3 (charX<=45): push to charX=55, startFall
- If skeleton reaches room 8: play "Victory", kid sheathes

### Level 4 (mirror / shadow appearance)
- When exit open and kid in room 11, row 0: attach mirror delegate to kid
- Mirror detected: play "Danger" after 100ms
- Kid runs left into mirror (runjump, faceL, on floor): shadow `appearOutOfMirror`; kid `stealLife`; play "Mirror"
- Kid moves past block 3 in room 4: hide mirror reflection
- Shadow visible above ground: stand + setInvisible

### Level 5 (shadow potion cutscene)
- When gate at (1,0,24) raises and shadow not visible, facing right: run shadow through potion-drinking scripted program

### Level 6 (shadow chase)
- `firstUpdate`: shadow.charX += 8
- Camera room 1: play "Danger"; track kid.charBlockX === 6 → shadow step11
- Kid at bottom of level, right room: blockCamera, nextLevel after 100ms

### Level 8 (mouse)
- When exit open, camera room 16, kid row 0: start 12.5s timer
- After timer: spawn Mouse at room=16, blockX=9, facing=-1; run scripted scurry/raise/turn/scurry program

### Level 12 (shadow merge / leap of faith)
- Remove sword from room 15 when kid is in room 20, row 1
- Shadow combat setup when kid reaches room 15, blocks 5–6
- Shadow merge: when kid gets close enough after shadow defeated
  - `level.leapOfFaith = true`; kid.addLife(); mergeShadowPosition(); showShadowOverlay; flashWhiteShadowMerge
- Leap of faith: replace SPACE tiles in room 2 with hidden FLOOR tiles
- Camera room 23: nextLevel

### Level 13 (Jaffar)
- `firstUpdate`: kid.action = "startrun"
- Shake random loose boards in visited rooms 16 and 23
- Jaffar dead: freeze clock (`endTime = now`), show time, trigger exit door button after 7s

### Level 14
- Camera room 5: nextLevel

---

## performProgram(program, actor)

Scripted actor animation via a Promise chain:
```javascript
program.reduce((promise, op) => {
    promise.then(() => {
        object = op.o || actor
        switch op.i:
            "ACTION":     object.action = op.p2
            "WAIT":       no-op
            "TURN":       object.turn()
            "SOUND":      game.sound.play(op.p2)
            "REM_OBJECT": level.removeObject(charBlockX, charBlockY, room)
            "REM_ACTOR":  object.visible = false; object.kill()
            default:      fn = op.i  (function directly)
        return PrinceJS.Utils.perform(fn, op.p1)
    })
}, Promise.resolve())
```

Each step: execute function immediately, then wait `op.p1` ms before advancing.

---

## checkTimers()

```javascript
if continueTimer > -1:
    continueTimer--
    if continueTimer === 0:
        continueTimer = -1
        ui.showPressButtonToContinue()
        pressButtonToContinueTimer = 260

if pressButtonToContinueTimer > -1:
    pressButtonToContinueTimer--
    if pressButtonToContinueTimer === 0:
        pressButtonToContinueTimer = -1
        restartGame()
```

---

## handleDead()

```javascript
continueTimer = 10    // 10 ticks × 80ms = 800ms before "press button" prompt appears
```

---

## buttonPressed()

```javascript
if pressButtonToContinueTimer > -1: reset(true)    // suppress cutscene, stay on Game
if pressButtonToNext: nextLevel(currentLevel)
```

---

## nextLevel(triggerLevel, skipped=false, keepRemainingTime=false)

```javascript
if triggerLevel !== undefined && triggerLevel !== currentLevel: return

PrinceJS.danger = null
PrinceJS.currentLevel++
PrinceJS.currentHealth = currentLevel === 13 ? kid.health : null
PrinceJS.maxHealth = kid.maxHealth

if currentLevel > 15 && currentLevel < 90: restartGame(); return

if skipped && !keepRemainingTime: setRemainingMinutesTo15()
if currentLevel >= 100 && level.number === 14: resetRemainingMinutesTo60()

reset()
PrinceJS.skipShowLevel = [13, 14].includes(currentLevel) && !skipped
```

---

## previousLevel(triggerLevel, skipped=false)

```javascript
if triggerLevel !== undefined && triggerLevel !== currentLevel: return
PrinceJS.danger = null
if currentLevel > 1: currentLevel--
reset()
PrinceJS.skipShowLevel = [13, 14].includes(currentLevel) && !skipped
```

---

## reset(suppressCutscene)

```javascript
game.sound.stopAll()
continueTimer = pressButtonToContinueTimer = -1
pressButtonToNext = false
enemies = []
if !suppressCutscene && currentLevel in [2,4,6,8,9,12,15]:
    state.start("Cutscene")
else:
    state.start("Game")
PrinceJS.skipShowLevel = [13, 14].includes(currentLevel)
```

---

## changeRoom(room, cameraRoom)

```javascript
setupCamera(room, cameraRoom)
if currentRoom === room: return
currentRoom = room
kid.flee = false
```

---

## setupCamera(room, cameraRoom)

```javascript
if blockCamera: return
if currentRoom > 0 && room <= 0: outOfRoom(); return
if cameraRoom === 0: return
room = cameraRoom || room
if rooms[room]:
    camera.x = rooms[room].x * SCREEN_WIDTH * SCALE_FACTOR
    camera.y = rooms[room].y * ROOM_HEIGHT  * SCALE_FACTOR
    checkForOpponent(room)
    level.checkGates(room, currentCameraRoom)
    currentCameraRoom = room
    visitedRooms[room] = true
```

---

## checkForOpponent(room)

Priority order for assigning `kid.opponent`:

1. **Same room + same blockY** — first matching alive enemy
2. **Near room + same blockY** — enemy in adjacent room, same row
3. **Same room** — any alive enemy in room
4. **Near room** — any alive enemy in adjacent room

On first Jaffar encounter: play "Jaffar2".

Sets `kid.opponent` and `enemy.opponent = kid`. Updates UI hp bars.

---

## checkGateFastDropped(gate)

When a gate fast-drops on enemies in its room: turns them away from the gate.

---

## timeUp()

```javascript
delayed(() => {
    PrinceJS.currentLevel = 16
    state.start("Cutscene")
}, 1000)
```

---

## outOfRoom()

```javascript
kid.die()
handleDead()
```

---

## floorStopFall(tile)

```javascript
level.floorStopFall(tile)
kid.checkLooseFloor(tile)
for each enemy: enemy.checkLooseFloor(tile)
```
