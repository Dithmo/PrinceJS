# PrinceJS Rewrite Gotchas

Non-obvious behaviors, tricky invariants, and things that are easy to get wrong when rewriting the PrinceJS codebase. Each item includes the file and line number, the correct behavior, and what a naive implementation would do instead.

---

## 1. CMD_GOTO sets `_seqpointer = p2 - 1`, not `p2`

**File:** `src/Actor.js`, lines 79–82 and 108–111

```js
PrinceJS.Actor.prototype.CMD_GOTO = function (data) {
  this._action = data.p1;
  this._seqpointer = data.p2 - 1;  // NOTE: p2 - 1, not p2
};

// In processCommand():
let data = this.anims.sequence[this._action][this._seqpointer];
this.commands[data.cmd](data);
this._seqpointer++;  // increment AFTER executing
```

**Correct behavior:** `CMD_GOTO` stores `p2 - 1` so that after `processCommand` increments `_seqpointer`, the pointer lands exactly on `p2`. The main loop's post-increment means the target index is always `p2 - 1` at storage time.

**Wrong assumption:** Set `_seqpointer = p2` and the next read will be `p2 + 1`, skipping the intended target frame entirely. For example, `stand` sequence `GOTO stand at index 1` would land at index 2 instead of 1, causing an infinite loop or wrong frame.

**Also note:** Setting `this.action = value` (via the property setter in Actor.js line 165–168) always resets `_seqpointer` to 0 automatically, because the setter writes `this._seqpointer = 0`. Direct assignment to `this._action` (bypassing the setter) does not reset the pointer — `CMD_GOTO` sets `this._action` directly, not through the setter.

---

## 2. `baseX` vs `baseY` offset asymmetry between Fighter and Mouse

**Files:** `src/Fighter.js` lines 141–146, `src/Mouse.js` lines 43–46

```js
// Fighter.updateBase():
this.baseX = this.level.rooms[this.room].x * PrinceJS.ROOM_WIDTH;      // no +3
this.baseY = this.level.rooms[this.room].y * PrinceJS.ROOM_HEIGHT + 3; // +3

// Mouse.updateBase():
this.baseX = this.level.rooms[this.room].x * PrinceJS.ROOM_WIDTH;      // no +3
this.baseY = this.level.rooms[this.room].y * PrinceJS.ROOM_HEIGHT + 3; // +3
```

Both Fighter and Mouse add +3 to `baseY`. Neither adds anything to `baseX`. (The original description of a Mouse difference is incorrect — Mouse.updateBase matches Fighter.updateBase for `baseY`.)

**Correct behavior:** `baseY` is always `room.y * ROOM_HEIGHT + 3` for both characters. `baseX` is always `room.x * ROOM_WIDTH` with no offset.

**Wrong assumption:** Forgetting the +3 on `baseY` shifts every character sprite 3 pixels up on the screen, misaligning them with the tile floor geometry. `ROOM_HEIGHT = BLOCK_HEIGHT * 3 = 63 * 3 = 189`, and the +3 compensates for the tile's own coordinate system.

---

## 3. `charFood` / rounding correction adds +0.5, not 13

**File:** `src/Actor.js`, lines 130–136

```js
let tempx = this.charX + this.charFdx * this.charFace;

if ((this.charFood && this.faceL()) || (!this.charFood && this.faceR())) {
  tempx += 0.5;
}

this.x = this.baseX + PrinceJS.Utils.convertX(tempx);
```

`convertX` is `Math.floor((x * 320) / 140)`. Adding 0.5 to `tempx` before the floor division creates a +1 pixel shift in certain cases, correcting a rounding artifact in the coordinate-space conversion.

**Correct behavior:** Add `+0.5` to `tempx` (in original coordinate space) when `(charFood && faceL) || (!charFood && faceR)`. This is a sub-pixel rounding correction, not a large physical offset.

**Wrong assumption:** Treating this as a 13-pixel sprite anchor correction, or applying the condition only when facing left (it applies to both directions depending on `charFood`). The `charFood` bit (bit 7 of `fcheck`, `0x80`) is frame-data controlled and varies per frame, not per facing direction.

---

## 4. Frame 141 skip in `climbup` / the "161 equals 150" comment

**File:** `assets/anims/kid.json` (framedef and sequence data), `src/Kid.js` lines 826–831

Frame 161 has no `fdx`/`fdy`/`fcheck` data — it has only a comment: `"161 equals 150"`. Frame 166 similarly is `"166 equals 15"`, and 170/171 are `"170/171 equals 158"`. These are alias markers: if code ever sets `charFrame = 161`, it must treat it identically to frame 150.

```js
// Kid.prepareCheckFloor() (lines 826-831):
if (this.charFrame === 141) {
  return { skip: true };
}
```

Frame 141 occurs in the middle of `climbup`, at the moment the character transitions from row Y to row Y-1. At this exact frame, `charBlockY` has just changed (due to CMD_UP in the climbup sequence), so `prepareCheckFloor` skips the floor check entirely to avoid acting on stale block coordinates.

**Correct behavior:** When `charFrame === 141`, skip all floor/button checks for that tick. When `charFrame === 161`, use the same framedef data as frame 150 (engarde idle). The alias frames (161, 166, 170, 171) have no independent framedef entries — accessing `framedef[161]` returns an object with only a `comment` property, no `fdx`, `fdy`, or `fcheck`.

**Wrong assumption:** Looking up `framedef[161]` and expecting numeric animation data, which would crash on `parseInt(undefined, 16)`. Or failing to skip frame 141 in floor checks, causing spurious falls at the top of a ledge.

---

## 5. `backwardsFall` sign controls fall direction on sword retreat

**File:** `src/Kid.js`, lines 20, 1427, 727, 779, 783

```js
// Initialized:
this.backwardsFall = 1;

// Set in startFall() (line 1427):
this.backwardsFall = this.swordDrawn ? -1 : 1;

// Used in bump() (line 727):
this.charX -= 2 * this.charFace * this.backwardsFall;

// Used in bumpFall() (lines 779, 783):
this.charX -= this.charFace * this.backwardsFall;
this.charX -= 2 * this.charFace * this.backwardsFall;
```

**Correct behavior:** `backwardsFall` is normally `1` (fall pushes against movement direction). When sword is drawn at the moment of falling, it is `-1`, which negates the push — the character falls in the forward direction rather than backward. This models the original game's behavior where retreating off an edge with a sword drawn sends you forward (toward the enemy).

**Wrong assumption:** Treating `backwardsFall` as a simple boolean or always using value `1`. The sign flip is intentional and affects how the character bounces off edges during sword fights.

---

## 6. `charFfoot` is stored as unsigned 5-bit (0–31)

**File:** `src/Actor.js` line 66; `src/Fighter.js` line 162

```js
// Actor.updateCharFrame():
this.charFfoot = fcheck & 0x1f;  // unsigned, 0-31

// Fighter.updateBlockXY():
let footX = this.charX + this.charFdx * this.charFace - this.charFfoot * this.charFace;
```

In `kid.json`, frame 23 (`standjump`) has `fcheck = 0x11`, giving `ffoot = 17`. In original Prince of Persia source, the 5-bit value was treated as a signed offset, making 17 equal to -15. In PrinceJS, `charFfoot` is treated as **unsigned** (0–31), so frame 23's foot offset is +17, not -15.

**Correct behavior:** Use `fcheck & 0x1f` directly as an unsigned value. There is exactly one frame in the kid animset (frame 23) where the raw bits have a value >= 16, and PrinceJS deliberately treats it unsigned.

**Wrong assumption:** Applying two's-complement to the 5-bit value (subtracting 32 when >= 16) to get negative foot offsets. This would incorrectly compute block collision for the `standjump` sequence.

---

## 7. `actionCode` (integer) vs `action` (string) — two parallel representations

**Files:** `src/Actor.js` line 26; `src/Fighter.js` lines 17, 115–121; `src/Enemy.js` lines 84–143

```js
// Fighter.CMD_ACT (line 115-121):
PrinceJS.Fighter.prototype.CMD_ACT = function (data) {
  this.actionCode = data.p1;
  if (data.p1 === 1) {
    this.charXVel = 0;
    this.charYVel = 0;
  }
};
```

`actionCode` values from the animation sequences:
- `0` = stand/idle (no momentum)
- `1` = running/moving (zeroes velocity on set)
- `2` = hanging
- `3` = stepfall (floor check differs from freefall)
- `4` = freefall (gravity active, `updateAcceleration` runs)
- `5` = bump
- `6` = hangstraight
- `7` = turn

`action` is a string (`"stand"`, `"freefall"`, `"engarde"`, etc.) that selects the sequence played and drives behavior logic in `updateBehaviour()`.

**Correct behavior:** `actionCode` is the integer set by `CMD_ACT (0xf9)` within sequence data and controls physics (gravity in `updateAcceleration`, `actionCode === 4`), floor-check modes, and enemy AI probability table lookups. `action` is the string set by game logic that selects which sequence to play. They are NOT redundant — changing `action` does not change `actionCode` until the new sequence's `CMD_ACT` executes.

**Wrong assumption:** Using `actionCode` to drive sequence selection, or deriving `actionCode` from `action`. There is a tick of latency between setting `action` and `actionCode` updating, which matters for physics.

---

## 8. Enemy `skill` range is 0–11; skill 0 is weakest, skill 11 is near-perfect

**File:** `src/Enemy.js`, lines 39–46

```js
PrinceJS.Enemy.STRIKE_PROBABILITY =    [61, 100,  61,  61,  61,  40, 100, 150,  0,  48,  32,  48];
PrinceJS.Enemy.RESTRIKE_PROBABILITY =  [ 0,   0,   0,   5,   5, 175,  16,   8,  0, 255, 255, 150];
PrinceJS.Enemy.BLOCK_PROBABILITY =     [ 0, 150, 150, 200, 200, 255, 200, 250,  0, 255, 255, 255];
PrinceJS.Enemy.IMPAIRBLOCK_PROBABILITY=[ 0,  61,  61, 100, 100, 145, 100, 250,  0, 145, 255, 175];
PrinceJS.Enemy.ADVANCE_PROBABILITY =   [255,200, 200, 200, 255, 255, 200,   0,  0, 255, 100, 100];
PrinceJS.Enemy.REFRAC_TIMER =          [16,  16,  16,  16,   8,   8,   8,   8,  0,   8,   0,   0];
PrinceJS.Enemy.EXTRA_STRENGTH =        [ 0,   0,   0,   0,   1,   0,   0,   0,  0,   0,   0,   0];
```

Enemy health is `EXTRA_STRENGTH[skill] + STRENGTH[level.number]`, where `STRENGTH` is indexed by level number (0–15).

Skill 8 has all-zero probabilities and zero refrac — it is used for the skeleton (which cannot be damaged by sword).

**Correct behavior:** Skill ranges 0–11. Probability values are compared against a random value from `game.rnd.between(0, 254)`; the action triggers when `probability > random`. A value of 255 means the action always triggers; 0 means never.

**Wrong assumption:** Treating skill as simply "low/high" with a small range, or expecting skill to directly equal health. Health is entirely level-dependent; only `EXTRA_STRENGTH[4] = 1` gives one extra HP at skill 4.

---

## 9. TROB list — only specific tiles get `update()` called each tick

**File:** `src/LevelBuilder.js`, lines 162–238; `src/Level.js` lines 94–104

The `trobs` array in `Level` is the only list of tiles that receive a `update()` call each game tick. Tiles not added to `trobs` are completely static.

Tiles added to TROB:
- `Button` (STUCK_BUTTON, RAISE_BUTTON, DROP_BUTTON) — line 164
- `Torch` / `TORCH_WITH_DEBRIS` — line 170
- `Potion` — line 182
- `Sword` — line 187
- `ExitDoor` (TILE_EXIT_RIGHT only) — line 193
- `Chopper` — line 204
- `Spikes` (**only when `modifier === 0`**) — lines 209–210
- `Loose` board — line 218
- `Skeleton` — line 223
- `Mirror` — line 228
- `Gate` — line 237

Tiles NOT in TROB (static, no `update()`):
- Wall, Floor, Space, Tapestry, Tapestry_Top, Pillar variants, Balcony variants, Lattice variants, Debris, Null
- `TILE_EXIT_LEFT` (element 16) — falls through to `default` case, becomes a static `Base` tile

**Critical sub-gotcha — Spikes `modifier` check:** A Spike tile with `modifier > 0` is created but never added to TROB. That means it animates from its initial visual state (controlled by the modifier value in `Spikes` constructor) but its `update()` never runs. Pre-extended spikes (modifier 1–9 sets the initial visual frame) are purely cosmetic static decoration. Only `modifier === 0` (fully retracted) spikes are interactive.

**Wrong assumption:** Calling `addTrob()` for all spikes regardless of modifier, which causes static pre-extended spikes to start animating and cycling, changing gameplay.

---

## 10. Exit door is a two-tile structure; only `EXIT_RIGHT` is functional

**File:** `src/LevelBuilder.js` lines 190–199

```js
case PrinceJS.Level.TILE_EXIT_RIGHT:
  open = id === startId && Math.abs(tileNumber - startLocation) <= 1;
  tile = new PrinceJS.Tile.ExitDoor(this.game, t.modifier, this.type, open);
  this.level.addTrob(tile);
  ...
  break;
// TILE_EXIT_LEFT has no explicit case -> falls to default -> Base tile, no trob
```

The exit door occupies two tiles side by side: `TILE_EXIT_LEFT` (16) on the left and `TILE_EXIT_RIGHT` (17) on the right. Only `EXIT_RIGHT` is built as an `ExitDoor` instance and added to TROB. `EXIT_LEFT` becomes a static `Base` tile.

The `open` flag for the initial state is only computed on `EXIT_RIGHT`, based on whether the tile number is within 1 position of the prince's start location in the starting room.

**Correct behavior:** Fire events target `EXIT_RIGHT` (or walk left to it via `Level.fireEvent` line 251–253). `Kid.jump()` checks `tile.isExitDoor()` on both left and right tiles, then resolves to whichever is the `EXIT_RIGHT` instance.

**Wrong assumption:** Creating an `ExitDoor` for `EXIT_LEFT`, or trying to call `.raise()` / `.open` on the left tile.

---

## 11. `PrinceJS.Tile.Gate.reset()` must be called before any level loads

**File:** `src/Game.js` line 113; `src/tiles/Gate.js` lines 39–45

```js
// Gate.js:
let syncSoundGatesRaise = new Set();
let syncSoundGatesDrop = new Set();

PrinceJS.Tile.Gate.reset = function () {
  syncSoundGatesRaise.clear();
  syncSoundGatesDrop.clear();
};

// Game.create():
PrinceJS.Tile.Gate.reset();
```

`syncSoundGatesRaise` and `syncSoundGatesDrop` are **module-level variables** (not instance variables). They survive across level transitions. If `reset()` is not called when a new level loads, gate sound synchronization from the previous level persists — stale gate references remain in the sets, and the new level's gates may never play rising/dropping sounds, or may play them at wrong times.

**Correct behavior:** Call `PrinceJS.Tile.Gate.reset()` in `Game.create()` before building the level, clearing both sound-sync sets.

**Wrong assumption:** Assuming gate sound state is per-instance and resets automatically when gate objects are garbage collected. The sets hold strong references, preventing GC, and the sound logic checks `syncSoundGatesRaise.values().next().value === this` (first item wins), so stale entries corrupt subsequent-level gate audio.

---

## 12. Shadow overlay uses `delegate` pattern via `Kid.showShadowOverlay()`

**File:** `src/Kid.js` lines 129–146; `src/Actor.js` lines 119–122 and 139–141

```js
// Kid.showShadowOverlay():
this.delegate = {
  syncFrame: (actor) => {
    this.shadowOverlay.frameName = "shadow-" + actor.frameName.split("-")[1];
  },
  syncFace: (actor) => {
    this.shadowOverlay.charFace = actor.charFace;
  }
};

// Actor.changeFace() calls delegate.syncFace()
// Actor.updateCharPosition() calls delegate.syncFrame()
```

The `delegate` field on Actor is a general-purpose hook: whenever a character changes its display frame or facing direction, it calls the delegate. For Kid, `showShadowOverlay()` installs a delegate that mirrors the Kid's sprite into the shadow overlay sprite.

The `Mirror` tile (level 4) also implements `syncFrame` and `syncFace` and is installed as the Kid's delegate while the mirror reflection is active (`Game.js` line 271: `this.kid.delegate = tile`).

**Correct behavior:** There is only ever one `delegate` at a time. Installing a new delegate (mirror tile) overwrites the shadow overlay delegate. Calling `hideShadowOverlay()` sets `this.delegate = null`.

**Wrong assumption:** Treating `delegate` as an event emitter that notifies multiple listeners. It is a single-slot callback object. Assigning the mirror tile as delegate silently disconnects the shadow overlay (intentional for level 4).

---

## 13. `world.sort("z")` is called once; sprites added after the sort go to the top

**File:** `src/Game.js` line 118; `src/Actor.js` line 31; `src/Fighter.js` line 57; `src/Level.js` lines 14, 17

```js
// z values assigned at construction:
// Level.back group: z = 10
// Actor sprite:     z = 20
// Fighter.sword:    z = 21
// Level.front group: z = 30

// In Game.create():
this.world.sort("z");
```

`world.sort("z")` sorts all currently-added display objects by their `z` property. It is called once after all level tiles and character sprites are added. Any sprite added after this call is appended at the top of the display list, ignoring the `z` sort. The Mouse is added lazily (level 8 cutscene), so it always renders on top of everything.

**Correct behavior:** Add all persistent sprites before calling `world.sort()`. Accept that dynamically-added sprites (Mouse, shadow overlay child, sword) layer by creation order, which is intentional for those specific cases (sword at z=21 is in the right order at construction time because Fighter adds it before `world.sort()` runs).

**Wrong assumption:** Assuming re-sorting happens automatically or that `z` is respected at any time. In Phaser 2, `sort()` must be called explicitly; modifying `z` after sort does nothing until sort is called again.

---

## 14. Exit door `open` start condition: within 1 tile of prince's start location

**File:** `src/LevelBuilder.js` lines 191–199

```js
case PrinceJS.Level.TILE_EXIT_RIGHT:
  open = id === startId && Math.abs(tileNumber - startLocation) <= 1;
  tile = new PrinceJS.Tile.ExitDoor(this.game, t.modifier, this.type, open);
```

`tileNumber = y * 10 + x`. The exit door starts open (immediately crossable) only when:
1. It is in the same room as the prince's starting room (`id === startId`), AND
2. Its tile number is within 1 of the prince's starting tile number.

The `<= 1` allows for the two-tile door structure: if the prince starts at the `EXIT_LEFT` tile (tileNumber N), the `EXIT_RIGHT` tile is at N+1, which satisfies `abs(N+1 - N) = 1`.

**Correct behavior:** On level 1, the prince starts near an open exit. The `startLocation` is read from `json.prince.location + (json.prince.bias || 0)`.

**Wrong assumption:** Always starting the exit door closed, or checking for exact tile equality (`=== startLocation`) rather than `<= 1`. The off-by-one check is required to handle the two-tile span of the door.

---

## 15. `PrinceJS.Restart()` resets all global state including URL query params

**File:** `src/Boot.js` lines 25–66

```js
PrinceJS.Init = function () {
  PrinceJS.currentLevel = 1;
  PrinceJS.maxHealth = 3;
  PrinceJS.currentHealth = null;
  PrinceJS.minutes = 60;
  PrinceJS.startTime = undefined;  // note: undefined, not null
  PrinceJS.endTime = undefined;
  PrinceJS.strength = 100;
  PrinceJS.screenWidth = 0;
  PrinceJS.shortcut = false;
  PrinceJS.danger = null;
  PrinceJS.skipShowLevel = false;
};

PrinceJS.Restart = function () {
  PrinceJS.Utils.clearQuery();  // clears URL params
  PrinceJS.Init();
  PrinceJS.Utils.applyQuery();  // re-reads URL params, may override level/health/time
};
```

`PrinceJS.startTime` uses `undefined` (not `null`). The check in `Game.create()` is `if (!PrinceJS.startTime)` — both `null` and `undefined` pass this check. However, `Utils.getDeltaTime()` also uses `if (!PrinceJS.startTime)`, so the distinction doesn't matter functionally. Be consistent with `undefined` to avoid introducing subtle differences.

**Correct behavior:** After `Restart()`, `currentLevel` may not be 1 if the URL contains `?level=N`. The URL query re-apply is intentional — it lets the player bookmark a specific level while still resetting runtime state.

**Wrong assumption:** Assuming `Restart()` always resets to level 1. Or setting `startTime = null` (works) vs `undefined` (original) — either passes the falsy check, but mixing them creates confusion.

---

## 16. Loose board chain-fall: landing on another loose board forces it to fall immediately

**Files:** `src/tiles/Base.js` lines 193–194; `src/Level.js` lines 230–238

```js
// Base.addDebris():
if (this.element === PrinceJS.Level.TILE_LOOSE_BOARD) {
  this.sweep();  // forces immediate fall, bypassing STATE_SHAKING
  return;
}

// Tile.Loose.sweep():
PrinceJS.Tile.Loose.prototype.sweep = function () {
  this.state = PrinceJS.Tile.Loose.STATE_FALLING;
  this.step = 0;
  this.back.frameName = this.key + "_falling";
  this.onStartFalling.dispatch(this);  // triggers Level.floorStartFall -> propagation
  this.game.sound.play(Sounds[0]);
};
```

Chain reaction path:
1. Loose board A finishes shaking and falls (`STATE_FALLING`).
2. Board A lands: `Level.floorStopFall(A)` runs.
3. The tile at A's landing position is another loose board B.
4. `B.addDebris()` is called, which sees `B.element === TILE_LOOSE_BOARD`, calls `B.sweep()`.
5. `B.sweep()` sets B to `STATE_FALLING` immediately (no shaking phase) and dispatches `onStartFalling`.
6. `Level.floorStartFall(B)` replaces B with a SPACE tile and computes B's fall distance.

After landing, `Level.floorStopFall` also calls `shakeFloor(row)` which calls `tile.shake(false)` on all loose boards in that row. `shake(false)` starts the shaking animation but allows the board to abort at step 3 if no weight is on it. This is a visual-only effect; it does not force a fall unless `fall=true`.

**Correct behavior:** Forced chain falls happen only when a board lands directly on another loose board (via `addDebris` → `sweep`). Adjacent loose boards in the same row only get a visual shake from `shakeFloor`.

**Wrong assumption:** Implementing chain falls as `shakeFloor` always forcing full falls. Or missing the `addDebris` early-return path for `TILE_LOOSE_BOARD`, which causes the debris animation to also be added on top of the board, breaking the fall-through behavior.

---

## 17. Game timer uses real wall-clock time; pausing does not pause the timer

**Files:** `src/Game.js` lines 33–36; `src/Utils.js` lines 320–341; `src/Utils.js` lines 416–422

```js
// Game.create() (initializes timer once):
if (!PrinceJS.startTime) {
  let date = new Date();
  date.setMinutes(date.getMinutes() - (60 - PrinceJS.minutes));
  PrinceJS.startTime = date;
}

// Utils.getDeltaTime():
let diff = (PrinceJS.endTime || new Date()).getTime() - PrinceJS.startTime.getTime();

// Utils.restoreQuery() (called on game resume):
if (PrinceJS.Utils.getRemainingMinutes() < PrinceJS.minutes) {
  let date = new Date();
  date.setMinutes(date.getMinutes() - (60 - PrinceJS.minutes));
  PrinceJS.startTime = date;
}
```

`PrinceJS.startTime` is a `Date` object. Remaining time is `60 min - (now - startTime)`. While the game is paused, `new Date()` continues to advance, so the countdown does not pause. When the game resumes, `restoreQuery()` checks if the stored `PrinceJS.minutes` (updated in the URL on pause) is still larger than the current remaining time; if so, it resets `startTime` to effectively restore the paused value. This compensates partially for the elapsed pause time.

**Correct behavior:** The timer ticks in real time. On pause, the current remaining minutes are saved to `PrinceJS.minutes` (via `updateQuery`). On resume, if the saved time is still larger, `startTime` is adjusted forward so the displayed time matches the saved value.

**Wrong assumption:** Implementing the timer as a game-tick counter that stops when `updateWorld` stops running. The 80ms `updateWorld` loop does not drive the timer — wall-clock time does.

---

## 18. `update()` runs at 60 fps for input; `updateWorld()` runs at ~12.5 fps (80 ms) for game state

**File:** `src/Game.js` lines 125, 142–176, 178–191

```js
// Fixed-rate game logic loop:
this.game.time.events.loop(80, this.updateWorld, this);

// Phaser render-loop callback (60fps):
update: function () {
  if (PrinceJS.Utils.continueGame(this.game)) {
    this.buttonPressed();
    // touch/gamepad continue-screen checks...
  }
}
```

Phaser's `update()` method is called every render frame (~60fps) and is used only for detecting touch/gamepad press to dismiss the "press button to continue" screen. All game state updates (physics, AI, tile animation, UI) happen in `updateWorld()`, which fires at fixed 80ms intervals (12.5fps).

Keyboard input for the Kid (`Kid.keyL()`, `Kid.keyU()`, etc.) reads `Phaser.Keyboard` key state synchronously inside `updateWorld()`, so it runs at 12.5fps. The keyboard is effectively sampled at 12.5fps even though key events are registered at 60fps.

**Correct behavior:** Game logic (physics, movement, collision) advances at 12.5fps. Input polling happens within that 12.5fps window. Touch/gamepad "any press" detection for the pause/continue screen happens at 60fps via `update()`.

**Wrong assumption:** Putting gameplay logic in Phaser's `update()` (60fps). This would run the game at 60fps, which is historically incorrect and changes the feel/speed of all animations, timing, and physics.

---

## 19. The `action` property setter always resets `_seqpointer` to 0

**File:** `src/Actor.js` lines 160–169

```js
Object.defineProperty(PrinceJS.Actor.prototype, "action", {
  get: function () { return this._action; },
  set: function (value) {
    this._action = value;
    this._seqpointer = 0;  // ALWAYS resets to index 0
  }
});
```

Any assignment `this.action = "newAction"` resets `_seqpointer` to 0. `CMD_GOTO` bypasses this by assigning directly to `this._action` (not the setter) and manually setting `_seqpointer = p2 - 1`.

**Correct behavior:** When game logic calls `this.action = "engarde"` or any action string, the sequence always starts from index 0. This is appropriate for most transitions. `CMD_GOTO` is the only mechanism for mid-sequence jumps.

**Wrong assumption:** Calling `this.action = sameAction` to "stay" in the current sequence restarts it from index 0. For example, `this.action = "freefall"` inside `freefall` would restart the gravity sequence, which is usually wrong. The loop sequences end with `CMD_GOTO` back to mid-sequence to avoid this.

---

## 20. `getTileAt()` returns a dummy wall tile for out-of-bounds or undefined rooms

**File:** `src/Level.js` lines 118–146

```js
getTileAt: function (x, y, room) {
  if (!this.rooms[room]) {
    return this.dummyWall;  // sentinel Wall tile
  }
  // ... resolve room links for out-of-bounds x/y ...
  if (newRoom <= 0) {
    return this.dummyWall;
  }
  return this.rooms[newRoom].tiles[newX + newY * 10];
},
```

`dummyWall` is a `PrinceJS.Tile.Base` with `element = TILE_WALL`. Any query for a tile in a room that doesn't exist, or that navigates to a room with `links.x <= 0` (no neighbor), returns this single shared sentinel object.

**Correct behavior:** `getTileAt` never returns `null` or `undefined`. Callers do not need null checks. The sentinel object is shared — do not modify properties on tiles returned from boundary queries, as that modifies the sentinel.

**Wrong assumption:** Checking `if (tile)` after `getTileAt()` is unnecessary and should not be added. Or modifying a tile returned from `getTileAt(x > 9, ...)` assuming it's a unique object.

---

## 21. Room transition coordinate adjustments use fixed offsets, not computed deltas

**File:** `src/Fighter.js` lines 172–204

```js
// Moving left (charBlockX < 0):
this.charX += 140;
this.baseX -= 320;
this.charBlockX = 9;
this.room = leftRoom;

// Moving right (charBlockX > 9):
this.charX -= 140;
this.baseX += 320;
this.charBlockX = 0;
this.room = rightRoom;
```

`charX` is in the original coordinate space (0–140 per room, 14 units per block). When crossing a room boundary, `charX` is adjusted by 140 (one full room width in original coords). `baseX` is in screen pixels, adjusted by 320 (one full room width in screen pixels = `ROOM_WIDTH = SCREEN_WIDTH = 320`). `BLOCK_HEIGHT = 63`, `ROOM_HEIGHT = 189`.

For vertical transitions (down in `checkRoomChange`):
```js
// Fighter.checkRoomChange():
if (this.charY > 192) {
  this.charY -= 192;  // not 189!
  this.baseY += 189;
  this.room = rooms[this.room].links.down;
}
```

Note `charY > 192` threshold with `charY -= 192` but `baseY += 189`. The 3-unit difference accounts for the `+3` offset in `updateBase()`. Kid's `checkRoomChange` uses `charY > 189` and `charY -= 189` / `baseY += 189`.

**Correct behavior:** Horizontal: ±140 to `charX`, ±320 to `baseX`. Fighter vertical: threshold 192, subtract 192 from charY, add 189 to baseY. Kid vertical: threshold 189, subtract 189 from charY, add 189 to baseY.

**Wrong assumption:** Using 189 for both the threshold and the charY adjustment in Fighter, or using 192 for Kid. These small differences cause characters to teleport to slightly wrong Y positions when crossing room boundaries.

---

## 22. `danger` flag: set once per level, never set to true again after first `land()`

**File:** `src/Game.js` lines 135–137; `src/Kid.js` lines 1297–1317

```js
// Game.create():
if (PrinceJS.danger === null) {
  PrinceJS.danger = this.level.number === 1 && this.playDanger;
}

// Kid.land() (lines 1297-1317):
switch (fallingBlocks) {
  case 0:
  case 1:
    if (PrinceJS.danger) {
      this.action = "medland";  // damage on first landing when danger=true
    } else {
      this.action = "softland";
    }
    break;
  ...
}
...
PrinceJS.danger = false;  // line 1317: cleared after first land
```

`PrinceJS.danger` is `null` at startup. On level 1 only (and only if `json.prince.danger !== false`), it is initialized to `true`. After the prince first lands from any fall, it is set to `false` permanently for the rest of the game session. It is set to `null` again only when advancing to the next level (`Game.nextLevel()`).

**Correct behavior:** The "danger" state causes the prince to take a landing hit on his first fall (at the start of level 1). Once he lands, `danger` is cleared. Between levels, it's reset to `null` so each level start re-evaluates. When restarting a level (`restartLevel`), `danger` is not reset — only `nextLevel` and `previousLevel` reset it.

**Wrong assumption:** Making `danger` a per-level persistent flag. After the first landing on level 1, `danger = false` persists, so restarting level 1 (without going to next level first) means the danger effect does not repeat.

---

## 23. `Level.fireEvent()` follows a linked-list chain via `events[event].next`

**File:** `src/Level.js` lines 241–268

```js
fireEvent: function (event, type, stuck) {
  if (!this.events[event]) { return; }
  // ... fire the tile ...
  if (this.events[event].next) {
    this.fireEvent(event + 1, type);  // chain to next event
  }
}
```

Events are stored in `this.events` (array from `json.events`). Each event has a `next` boolean. When a button is pushed with event index N, if `events[N].next === true`, `fireEvent(N+1, ...)` is called recursively. This allows one button press to trigger multiple gates/doors.

The `stuck` parameter (true when a RAISE_BUTTON triggers with `debris=true`, i.e., a weight stuck on it) propagates only to the first event; recursive calls omit it.

**Correct behavior:** Always build the recursive chain exactly as: `this.fireEvent(event + 1, type)` without `stuck` in recursive calls. Event 0 is the first event, and chaining always increments by 1.

**Wrong assumption:** Treating events as a flat list where a button triggers only its own assigned event. Multiple effects from one button require the linked chain.
