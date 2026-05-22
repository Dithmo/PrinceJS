# Utils.js — Spec

**File:** `src/Utils.js`  
**Purpose:** Coordinate conversion, async utilities, flash presets, gamepad/pointer input helpers, timer math, URL param parsing.

---

## Coordinate Conversions

```javascript
convertX(x):            Math.floor((x * 320) / 140)
convertXtoBlockX(x):    Math.floor((x - 7) / 14)
convertYtoBlockY(y):    Math.floor(y / 63)
convertBlockXtoX(block): block * 14 + 7
convertBlockYtoY(block): (block + 1) * 63 - 10
    // block=0 → 53, block=1 → 116, block=2 → 179
```

---

## Async Utilities

**`delayed(fn, ms)`** — returns Promise; executes `fn` after `ms` milliseconds.

**`delayedCancelable(fn, ms)`** — returns `{ cancel, promise }`; calling `cancel()` prevents `fn` from running.

**`perform(fn, ms)`** — executes `fn` immediately then resolves after `ms` ms.

---

## Flash Presets

Each preset is an array of durations (in ticks × 4ms). `flashPattern` triggers `Interface.flash()` for each entry, waiting `4 * time` after each.

```
flashRedDamage:         [25]
flashRedPotion:         [50, 25, 25]
flashGreenPotion:       [50, 25, 25]
flashYellowSword:       [50, 25, 25, 50, 25, 25, 25]
flashWhiteShadowMerge:  [50, 25, 25, 50, 25, 25, 25, 50, 25, 25, 50, 25, 25, 25]
flashWhiteVizierVictory:[25, 25, 100, 100, 50, 50, 25, 25, 50]
```

---

## Gamepad Input

**Up:** buttons A(0), R(5), ZR(7), DPadUp(12) OR axes LeftY/RightY < -0.75  
**Down:** DPadDown(13) OR axes LeftY/RightY > 0.75  
**Left:** DPadLeft(14) OR axes LeftX/RightX < -0.75  
**Right:** DPadRight(15) OR axes LeftX/RightX > 0.75  
**Action (Shift):** B(1), Y(3), L(4), ZL(6)  
**Info:** X(2) — debounced 500ms  
**Previous level:** Minus(8) — debounced  
**Next level:** Plus(9) — debounced  

---

## Pointer Helpers

**`pointerPressed(game)`** — edge-detect: true only on the tick where pointer goes from down to up. Uses internal `_pointerPressed` flag.

**`pointerDown(game)`** — true while any pointer is held.

**`effectivePointer(game)`** — returns pointer position adjusted for screen flip.

**`effectiveScreenSize(game)`** — returns `{ width, height }` of canvas.

---

## Timer

**`getDeltaTime()`** — uses `Date.now()` wall clock; when `PrinceJS.endTime` is set, clock is frozen (returns fixed remaining time).

**`getRemainingMinutes()`** — `Math.max(0, Math.min(60, 60 - elapsed_minutes))`

**`getRemainingSeconds()`** — seconds component of remaining time.

**`restoreQuery()`** — adjusts `startTime` to compensate for time elapsed while Phaser was paused (called on game resume).

---

## `applyStrength(value)`

```javascript
if (PrinceJS.strength < 100):
    return Math.ceil((value * PrinceJS.strength) / 100)
else:
    return value
```

---

## URL Parameters

Parsed by `applyQuery()`, cleared by `clearQuery()`:

| Param | Aliases | Range | Effect |
|-------|---------|-------|--------|
| `level` | `l` | 1–14 or ≥90 | `PrinceJS.currentLevel` |
| `health` | `h` | 3–10 | `PrinceJS.maxHealth` |
| `time` | `t` | 1–60 | `PrinceJS.minutes` |
| `strength` | `s` | 0–100 | `PrinceJS.strength` |
| `width` | `w` | >0 | `PrinceJS.screenWidth` |
| `shortcut` | `_` | "true" | `PrinceJS.shortcut = true` |

---

## Screen Flip

**`toggleFlipScreen()`** — toggles a flipped-screen mode.  
**`isScreenFlipped()`** — returns current flip state.

---

## `continueGame(game)`

Returns true if the gamepad "next" button or info button is pressed, triggering skip of intro/cutscene screens.
