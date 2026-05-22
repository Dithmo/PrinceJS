# Game States — Spec

Covers: `Title.js`, `Cutscene.js`, `Credits.js`, `EndTitle.js`, and the Interface.

---

## Title.js — `src/Title.js`

**Purpose:** Opening title sequence with timed animation reveals and music.

### State flow

```
Boot → Preloader → Title → Game  (keyboard pressed / gamepad)
                         → Cutscene  (after tween3 fade-out completes + 3500ms)
```

### create()

```javascript
stopMusic()
tick = 0
world.setBounds(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT)

// Background image, starts invisible:
back = add.image(0, 0, "title", "main_background"), alpha=0
tween1 = tween back.alpha → 1 over 2000ms
// On tween1 complete: play "PrologueA"

presents = image(centerX, centerY+29.5, "title","presents"), visible=false
author   = image(centerX-3, centerY+37, "title","author"),   visible=false
prince   = image(0, 0, "title","prince"),                     visible=false

textBack = image(0, world.height, "title","in_the_absence"), anchor(0,1)
cropRect = Rectangle(0, 0, 0, textBack.height)   // starts with width=0
tween2 = animate cropRect.width → textBack.width over 200ms
textBack.crop(cropRect)

tween3 = tween textBack.alpha → 0 over 2000ms
// On tween3 complete: delayed 3500ms → cutscene()

keyboard.onDownCallback = play
```

### update() — tick schedule

```
tick 0:    tween1.start()                        (fade in background)
tick 250:  presents.visible = true
tick 450:  presents.visible = false
tick 530:  author.visible = true
tick 730:  author.visible = false
tick 1030: prince.visible = true
tick 1600: tween2.start(); play "PrologueB"
tick 2250: back.visible=false; prince.visible=false; tween3.start()
```

Each update also: `tick++`, `textBack.updateCrop()`, and checks `continueGame()` → `play()`.

### play()

```javascript
stopMusic()
keyboard.onDownCallback = null
state.start("Game")
```

### cutscene()

```javascript
stopMusic()
keyboard.onDownCallback = null
state.start("Cutscene")
```

---

## Cutscene.js — `src/Cutscene.js`

**Purpose:** Plays level introduction/ending cutscenes using a bytecode-like program from JSON.

### preload()

Loads `"cover"` image + level-specific music (varies by `PrinceJS.currentLevel`):
```
level 1:         Princess, Jaffar, Heartbeat
level 2,4,6,12:  Heartbeat2
level 8,9:       Timer
level 15:        Embrace
level 16:        TragicEnd
```
Loads cutscene JSON: `assets/cutscenes/scene{currentLevel}.json`

### States

```
STATE_SETUP   = 0
STATE_READY   = 1
STATE_WAITING = 2
STATE_RUNNING = 3
```

### create()

```javascript
reset()
cutscene = cache.getJSON("cutscene")
if !cutscene: next(); return

program = cutscene.program
scene = new PrinceJS.Scene(game)
cover = add.sprite(0, 0, "cover")   // fade cover, added to scene.front
scene.front.addChild(cover)

executeProgram()

keyboard.onDownCallback = null
delayed(() => keyboard.onDownCallback = continue.bind(this), 1000)
game.time.events.loop(120, updateScene, this)
```

### executeProgram()

Processes opcodes from `program[pc]` until state changes:

```javascript
if state === STATE_WAITING:
    waitingTime--
    if waitingTime === 0: state = STATE_READY
    return

while state === STATE_SETUP || state === STATE_RUNNING:
    opcode = program[pc]
    switch opcode.i:
        "START":
            world.sort("z"); state = STATE_READY
            if opcode.p1 === 0: fadeOut(1) else: fadeIn()
        "END":
            endCutscene(opcode.p1 !== 0)
            state = STATE_WAITING; waitingTime = 1000
        "ACTION":
            actors[p1].action = p2
        "ADD_ACTOR":
            actors[p1] = new PrinceJS.Actor(game, p3, p4, p5, p2)
        "REM_ACTOR":
            actors[p1].kill()
        "ADD_OBJECT":
            objects[p1] = new PrinceJS.Tile.Clock(game, p3, p4, p2)
            scene.addObject(objects[p1])
        "START_OBJECT":
            objects[p1].activate()
        "EFFECT":
            scene.effect()
        "WAIT":
            state = STATE_WAITING; waitingTime = p1
        "MUSIC":
            stopMusic(); game.sound.play(p2)
        "SOUND":
            game.sound.play(p2)
        "FADEIN":
            fadeIn(p1 * 120)
        "FADEOUT":
            fadeOut(p1 * 120)
    pc++
```

### updateScene() (called every 120ms)

```javascript
if state === STATE_RUNNING: return
else if state === STATE_READY: state = STATE_RUNNING

executeProgram()
scene.update()
for each actor: actor.updateActor()
```

### endCutscene(fadeOut=true)

```javascript
if fadeOut:
    fadeOut(2000, () => next())
else:
    next()
```

### continue()

```javascript
if currentLevel < 15: play()
else: next()
```

### play()

```javascript
stopMusic()
keyboard.onDownCallback = null
state.start("Game")
```

### next()

```javascript
keyboard.onDownCallback = null
if currentLevel === 1:  state.start("Credits")
else if currentLevel === 15: PrinceJS.Restart(); state.start("EndTitle")
else if currentLevel === 16: PrinceJS.Restart(); state.start("Title")
else: play()
```

### reset()

```javascript
actors = []; objects = []
pc = 0; waitingTime = 0
sceneState = STATE_SETUP
```

### fadeIn(duration=2000, callback)

```javascript
add.tween(cover).to({ alpha: 0 }, 2000, Linear, true)
delayed(() => callback && callback(), duration)
```

### fadeOut(duration=2000, callback)

```javascript
add.tween(cover).to({ alpha: 1 }, 2000, Linear, true)
delayed(() => callback && callback(), duration)
```

---

## Credits.js — `src/Credits.js`

**Purpose:** "Marry Jaffar" ending screen shown after level 1 (demo mode completion). After animation plays `PrinceJS.currentLevel = 1` and starts "Game".

### Tween schedule (tick-based, each tick = 1 Phaser update)

```
tick 0:    tween1.start()   — fade in "marry_jaffar" image
tick 1200: tween2.start()   — reveal "prince" scroll
tick 1400: tween3.start()   — reveal "credits" scroll
tick 1600: hide textBack and back; tween4.start() — fade out credits
```

tween4 completion calls `play()`.

Keyboard enabled after 1000ms delay, any key → `play()`.

### play()

```javascript
PrinceJS.currentLevel = 1
keyboard.onDownCallback = null
state.start("Game")
```

---

## EndTitle.js — `src/EndTitle.js`

**Purpose:** Victory ending screen shown after level 15 (Princess rescued). Plays "Epilogue" music, then returns to Title.

Loads: `"Epilogue"` audio in preload.

### Tween schedule

```
tick 100:  tween1.start()   — fade in "the_tyrant" image (alpha 0→1, 2000ms)
tick 1250: tween2.start()   — reveal "main_background" scroll (200ms)
tick 7250: back.visible=false; tween3.start() — fade out main_background (2000ms)
```

tween3 completion calls `next()`.

Keyboard enabled after 1000ms delay, any key → `next()`.

### next()

```javascript
keyboard.onDownCallback = null
state.start("Title")
```

---

## Interface.js — `src/Interface.js`

**Purpose:** HUD layer. Manages HP bars, text display, flash effects, timer readout.

### Constructor: `PrinceJS.Interface(game, delegate)`

Creates a black bitmap strip at `y = (SCREEN_HEIGHT - UI_HEIGHT) * SCALE_FACTOR`, fixed to camera. Overlay bitmaps for red/green/yellow/white flash effects (all initially hidden). Bitmap text centred in UI strip.

```
layer       — main HUD sprite, fixedToCamera=true
layerRed    — red flash overlay
layerGreen  — green flash overlay
layerYellow — yellow flash overlay
layerWhite  — white flash overlay
flashMap    = { FLASH_RED→layerRed, FLASH_GREEN→layerGreen, ... }
text        — bitmapText, anchor(0.5, 0.5)
player      = null
playerHPs   = []
playerHPActive = 0
opp         = null
oppHPs      = []
oppHPActive = 0
pressButtonToContinueTimer = -1
hideTextTimer = -1
PrinceJS.InterfaceCurrent = this
```

### setPlayerLive(actor)

```javascript
player = actor; playerHPActive = actor.health
for i in 0..playerHPActive-1:  playerHPs[i] = sprite(i*7, 2, "general","kid-live")
for i in playerHPActive..maxHealth-1: playerHPs[i] = sprite(i*7, 2, "general","kid-emptylive")
actor.onDamageLife → damagePlayerLive
actor.onRecoverLive → recoverPlayerLive
actor.onAddLive → addPlayerLive
```

### damagePlayerLive(num)

```javascript
n = Math.min(playerHPActive, num)
for n times: playerHPs[--playerHPActive].frameName = "kid-emptylive"
```

### recoverPlayerLive()

```javascript
playerHPs[0].frameName = "kid-live"
playerHPs[playerHPActive].frameName = "kid-live"
playerHPActive++
```

### addPlayerLive()

```javascript
playerHPActive = playerHPs.length
if playerHPs.length < 10:
    hp = sprite(playerHPActive*7, 2, "general","kid-live")
    playerHPs[playerHPActive] = hp; playerHPActive++
for i in 0..playerHPActive-1: playerHPs[i].frameName = "kid-live"
```

### setOpponentLive(actor)

```javascript
if opp === actor: return
resetOpponentLive()
opp = actor
if !actor || !actor.active || actor.charName === "skeleton": return
oppHPs = []; oppHPActive = actor.health
for i from actor.health to 1:
    oppHPs[i-1] = sprite(SCREEN_WIDTH - i*7 + 1, 2, "general", baseCharName+"-live")
    if charColor > 0: oppHPs[i-1].tint = COLOR[charColor-1]
actor.onDamageLife → damageOpponentLive
actor.onDead → resetOpponentLive
```

### resetOpponentLive()

```javascript
if !opp: return
for each hp: hp.destroy()
opp.onDamageLife.removeAll()
opp.onDead.removeAll()
opp = null; oppHPs = []; oppHPActive = 0
```

### damageOpponentLive(num)

```javascript
if !opp || opp.charName === "skeleton": return
n = Math.min(oppHPActive, num)
for n times: oppHPs[--oppHPActive].visible = false
```

### updateUI()

Called every 80ms tick:
```javascript
showRegularRemainingTime()

// Blink last HP when at 1:
if playerHPActive === 1: toggle playerHPs[0] between "kid-live" / "kid-emptylive"
if oppHPActive === 1:    toggle oppHPs[0].visible

// "Press button" blink countdown:
if pressButtonToContinueTimer > -1:
    pressButtonToContinueTimer--
    if < 70 && % 7 === 0:
        text.visible = !text.visible
        if text.visible: play "Beep"

// Hide text timer:
if hideTextTimer > -1:
    hideTextTimer--
    if hideTextTimer === 0: hideText()
```

### Timer Display Methods

**showRegularRemainingTime():**
```javascript
if endTime: return
if getRemainingMinutes() === 0: showRemainingSeconds(); timeUp(); startTime = null
else if getRemainingMinutes() === 1: showRemainingSeconds()
else if force || (minutes < 60 && minutes % 5 === 0 && seconds === 0):
    showRemainingMinutes(force)
```

**showRemainingMinutes(force):**
```javascript
if showTextType && !force: return
minutes = getRemainingMinutes()
showText(minutes + " MINUTE(S) LEFT", "minutes")
hideTextTimer = 30
```

**showRemainingSeconds():**
```javascript
if showTextType in ["level","continue","paused"]: return
seconds = getRemainingMinutes() > 0 ? getRemainingSeconds() : 0
showText(seconds + " SECOND(S) LEFT", "seconds")
```

**showLevel():**
```javascript
if endTime || skipShowLevel: return
showText("LEVEL " + currentLevel, "level")
hideTextTimer = 25
delayed(() => {
    if !showTextType || showTextType === "level":
        hideText(); showRegularRemainingTime(true)
}, 2000)
```

**showPressButtonToContinue():**
```javascript
delayed(() => {
    showText("Press Button to Continue", "continue")
    pressButtonToContinueTimer = 200
}, 4000)
```

**showGamePaused():** `showText("GAME PAUSED", "paused")`

**showText(text, type):**
```javascript
text.setText(text); showTextType = type; hideTextTimer = -1
```

**hideText():**
```javascript
text.setText(""); showTextType = null; hideTextTimer = -1
```

### flash(flashColor)

```javascript
Object.keys(flashMap).forEach(color => {
    flashMap[color].visible = (color === String(flashColor))
})
```

Shows one colour overlay, hides all others.

### flipped()

```javascript
text.scale.y *= -1
text.y = (UI_HEIGHT - 2) * 0.5
if text.scale.y === -1: text.y += 2
```

---

## Mouse.js — `src/Mouse.js`

**Extends:** `PrinceJS.Actor`  
**Purpose:** Mouse scripted actor (level 8 cutscene). Simpler than Fighter — no physics, no combat.

### Constructor: `PrinceJS.Mouse(game, level, room, location, direction)`

```javascript
charBlockX = location % 10
charBlockY = Math.floor(location / 10)
x = convertBlockXtoX(charBlockX)
y = convertBlockYtoY(charBlockY)
PrinceJS.Actor.call(this, game, x, y, direction, "mouse")
action = "stop"
updateBase()
```

### updateActor()

```javascript
processCommand()
checkButton()
updateCharPosition()
```

No physics, no combat, no room transitions.

### CMD_FRAME (override)

```javascript
charFrame = data.p1
updateCharFrame()
updateBlockXY()
processing = false
```

### updateBlockXY()

```javascript
footX = charX + charFdx*charFace - charFfoot*charFace
footY = charY + charFdy
charBlockX = convertXtoBlockX(footX)
charBlockY = Math.min(convertYtoBlockY(footY), 2)
```

### updateBase()

```javascript
baseX = level.rooms[room].x * ROOM_WIDTH       // NOTE: no +3 offset (unlike Fighter)
baseY = level.rooms[room].y * ROOM_HEIGHT + 3
```

### checkButton()

```javascript
if !visible: return
tile = getTileAt(charBlockX, charBlockY, room)
if tile.element in [RAISE_BUTTON, DROP_BUTTON]: tile.push()
```

### turn()

```javascript
changeFace()
charX += charFace * 5
```
