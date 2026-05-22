# Preloader.js — Spec

**File:** `src/Preloader.js`  
**Purpose:** Loads all game assets (atlases, animation JSON, music, SFX) then transitions to Title or Game.

---

## Sprite Atlases (atlasJSONHash format)

21 atlases loaded with `game.load.atlasJSONHash(key, png, json)`:

```
"kid"          assets/gfx/kid.png          + kid.json
"princess"     assets/gfx/princess.png     + princess.json
"vizier"       assets/gfx/vizier.png       + vizier.json
"mouse"        assets/gfx/mouse.png        + mouse.json
"guard-1"      assets/gfx/guard-1.png      + guard-1.json
"guard-2"      assets/gfx/guard-2.png      + guard-2.json
"guard-3"      assets/gfx/guard-3.png      + guard-3.json
"guard-4"      assets/gfx/guard-4.png      + guard-4.json
"guard-5"      assets/gfx/guard-5.png      + guard-5.json
"guard-6"      assets/gfx/guard-6.png      + guard-6.json
"guard-7"      assets/gfx/guard-7.png      + guard-7.json
"fatguard"     assets/gfx/fatguard.png     + fatguard.json
"jaffar"       assets/gfx/jaffar.png       + jaffar.json
"skeleton"     assets/gfx/skeleton.png     + skeleton.json
"shadow"       assets/gfx/shadow.png       + shadow.json
"dungeon"      assets/gfx/dungeon.png      + dungeon.json
"palace"       assets/gfx/palace.png       + palace.json
"general"      assets/gfx/general.png      + general.json
"sword"        assets/gfx/sword.png        + sword.json
"title"        assets/gfx/title.png        + title.json
"cutscene"     assets/gfx/cutscene.png     + cutscene.json
```

---

## Animation JSON Files

7 animation definition files loaded with `game.load.json(key, url)`:

```
"kid-anims"       assets/data/kid-anims.json
"sword-anims"     assets/data/sword-anims.json
"fighter-anims"   assets/data/fighter-anims.json
"princess-anims"  assets/data/princess-anims.json
"shadow-anims"    assets/data/shadow-anims.json
"vizier-anims"    assets/data/vizier-anims.json
"mouse-anims"     assets/data/mouse-anims.json
```

---

## Music Tracks (MP3)

8 tracks loaded with `game.load.audio(key, url)`:

```
"PrologueA"   assets/music/01_Prologue_A.mp3
"PrologueB"   assets/music/01_Prologue_B.mp3
"Danger"      assets/music/02_Danger.mp3
"Accident"    assets/music/06_Accident.mp3
"Potion1"     assets/music/07_Potion_1.mp3
"Victory"     assets/music/08_Victory.mp3
"Prince"      assets/music/09_Prince.mp3
"Potion2"     assets/music/10_Potion_2.mp3
```

---

## Sound Effects (33 tracks)

```
"Beep"                  assets/sfx/beep.mp3
"BumpIntoWallHard"      assets/sfx/bump_into_wall_hard.mp3
"BumpIntoWallSoft"      assets/sfx/bump_into_wall_soft.mp3
"DrinkPotionGlugGlug"   assets/sfx/drink_potion_glug_glug.mp3
"EnGarde"               assets/sfx/en_garde.mp3
"FallingFloorLands"     assets/sfx/falling_floor_lands.mp3
"Float"                 assets/sfx/float.mp3
"Footsteps"             assets/sfx/footsteps.mp3
"FreeFallLand"          assets/sfx/free_fall_land.mp3
"GateComingDownSlow"    assets/sfx/gate_coming_down_slow.mp3
"GateReachesBottomClang" assets/sfx/gate_reaches_bottom_clang.mp3
"GateRising"            assets/sfx/gate_rising.mp3
"GateStopsAtTop"        assets/sfx/gate_stops_at_top.mp3
"HeroicDeath"           assets/sfx/heroic_death.mp3
"LooseFloorLands"       assets/sfx/loose_floor_lands.mp3
"LooseFloorShakes"      assets/sfx/loose_floor_shakes.mp3
"MediumLandingOof"      assets/sfx/medium_landing_oof.mp3
"OpponentStabbed"       assets/sfx/opponent_stabbed.mp3
"Potion1"               assets/sfx/potion1.mp3  (SFX, distinct from music)
"Potion2"               assets/sfx/potion2.mp3
"RaiseGate"             assets/sfx/raise_gate.mp3
"SlashMiss"             assets/sfx/slash_miss.mp3
"SoftLanding"           assets/sfx/soft_landing.mp3
"SpikedBySpikes"        assets/sfx/spiked_by_spikes.mp3
"SpikesDrop"            assets/sfx/spikes_drop.mp3
"SpikesRaise"           assets/sfx/spikes_raise.mp3
"StabbedByOpponent"     assets/sfx/stabbed_by_opponent.mp3
"SwordClash"            assets/sfx/sword_clash.mp3
"SwordDrawn"            assets/sfx/sword_drawn.mp3
"TheShadow"             assets/sfx/the_shadow.mp3
"UnsheatheSword"        assets/sfx/unsheathe_sword.mp3
"Victory"               assets/sfx/victory.mp3  (SFX version)
"ChopperChops"          assets/sfx/chopper_chops.mp3
```

---

## create()

```javascript
// Resume AudioContext on first user input (browser autoplay policy)
game.input.onDown.addOnce(() => {
    if (game.sound.context.state === "suspended") {
        game.sound.context.resume()
    }
})

game.input.addPointer()   // called twice → enables multitouch
game.input.addPointer()

game.input.gamepad.start()

// Disable context menu on right-click / long-press
game.canvas.oncontextmenu = (e) => e.preventDefault()

if (PrinceJS.SKIP_TITLE):
    game.state.start("Game")
else:
    game.state.start("Title")
```

---

## SKIP_TITLE Flag

`PrinceJS.SKIP_TITLE = false` by default (defined in Boot.js). When `true`, skips Title and Cutscene states and goes directly to Game.
