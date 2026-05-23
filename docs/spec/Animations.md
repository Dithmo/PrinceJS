# Animation System Spec

PrinceJS loads character animations from JSON files under `assets/anims/`. Each file (except `sword.json`) has two top-level keys:

- `sequence` — object mapping action name → array of command objects
- `framedef` — array of per-frame physics records, indexed by frame number

`sword.json` is special: it has a single key `swordtab`.

---

## JSON Command Object Format

Each entry in a sequence array is an object:

```json
{ "cmd": <opcode>, "p1": <value>, "p2": <value> }
```

`p1` and `p2` are optional depending on the opcode. `p1` may be a string (action name) when used with `CMD_GOTO` or `CMD_IFWTLESS`.

---

## Opcode Table

Commands 0x01–0xF0 (1–240) are registered as `CMD_NOOP` — a 256-slot dispatch table is initialized to NOOP at startup, then the named opcodes below overwrite specific slots.

| Opcode (hex) | Opcode (dec) | Name | Registered in | p1 | p2 | Effect |
|---|---|---|---|---|---|---|
| 0x00 | 0 | CMD_FRAME | Actor | frame# | — | Set `charFrame=p1`, stop processing loop |
| 0xF1 | 241 | CMD_NEXTLEVEL | Kid | — | — | Save health/maxHealth, play level-end music, dispatch `onLevelFinished` then `onNextLevel` after delay |
| 0xF2 | 242 | CMD_TAP | Actor | sound id | — | In Kid: p1=1→"Footsteps", p1=2→"BumpIntoWallSoft" (no-op if action=softLand). In Actor base: no-op |
| 0xF3 | 243 | CMD_EFFECT | Kid | — | — | No-op in Kid (empty function) |
| 0xF4 | 244 | CMD_JARD | Kid | — | — | `shakeFloor(charBlockY, room)` |
| 0xF5 | 245 | CMD_JARU | Kid | — | — | `shakeFloor(charBlockY-1, room)`; shake loose board at `(charBlockX, charBlockY-1)` and `(charBlockX+1, charBlockY-1)` |
| 0xF6 | 246 | CMD_DIE | Fighter | — | — | Called during death animations (dropdead, impale, etc.) |
| 0xF7 | 247 | CMD_IFWTLESS | Kid | action name | — | If `inFloat`: substitute action. `stepfall`→`stepfloat`, `bumpfall`→`bumpfloat`, `highjump`→`superhighjump` |
| 0xF8 | 248 | CMD_SETFALL | Fighter | xvel | yvel | Set `charXVel=p1`, `charYVel=p2` |
| 0xF9 | 249 | CMD_ACT | Fighter | actionCode | — | Set actionCode; values: 0=standing, 1=running, 3=falling, 4=freefall, 5=dying, 7=turning |
| 0xFA | 250 | CMD_CHY | Actor | delta | — | `charY += p1` |
| 0xFB | 251 | CMD_CHX | Actor | delta | — | `charX += p1 * charFace` |
| 0xFC | 252 | CMD_DOWN | Kid | — | — | If `charBlockY===2` and `charY>189`: `charY-=189`, `baseY+=189`, `charBlockY=0`, `changeRoomDown()` |
| 0xFD | 253 | CMD_UP | Kid | — | — | If `charBlockY===0`: `charY+=189`, `baseY-=189`, `charBlockY=2`, change room up |
| 0xFE | 254 | CMD_ABOUTFACE | Actor | — | — | `charFace *= -1` |
| 0xFF | 255 | CMD_GOTO | Actor | action name | index | Jump to `sequence[p1]`, set `_seqpointer = p2-1` (so next increment makes it p2) |

---

## Sequence Compact Notation

All sequences in this document use the following notation:

```
ACT(n)         = CMD_ACT p1=n
FRAME(n)       = CMD_FRAME p1=n  (stops loop)
CHX(n)         = CMD_CHX p1=n
CHY(n)         = CMD_CHY p1=n
TAP(n)         = CMD_TAP p1=n
ABOUTFACE      = CMD_ABOUTFACE
GOTO(name,n)   = CMD_GOTO p1=name p2=n
SETFALL(x,y)   = CMD_SETFALL p1=x p2=y
IFWTLESS(name) = CMD_IFWTLESS p1=name
JARD           = CMD_JARD
JARU           = CMD_JARU
NEXTLEVEL      = CMD_NEXTLEVEL
DIE            = CMD_DIE
UP             = CMD_UP
DOWN           = CMD_DOWN
EFFECT         = CMD_EFFECT
```

---

## framedef Record Format

Each `framedef` entry is an object:

```json
{ "fdx": <int>, "fdy": <int>, "fcheck": "<hex string>", "fsword": <int> }
```

`fsword` is optional (present only on frames that display a sword sprite). Some entries are placeholder objects with only a `"comment"` key — these frames are unused or aliased.

### fcheck Bitmask

`fcheck` is a hex string parsed to int. Bits:

| Bits | Mask | Field | Meaning |
|---|---|---|---|
| 0–4 | `0x1F` | `charFfoot` | Foot position offset |
| 5 | `0x20` | `charFthin` | Thin/crouching hitbox |
| 6 | `0x40` | `charFcheck` | Collision check enabled this frame |
| 7 | `0x80` | `charFood` | Character is on/touching the floor |

### updateCharPosition() Decode

After `CMD_FRAME` sets `charFrame`, `updateCharPosition()` reads `framedef[charFrame]`:

```
charFdx    = framedef.fdx
charFdy    = framedef.fdy
fcheck     = parseInt(framedef.fcheck, 16)
charFfoot  = fcheck & 0x1F
charFood   = (fcheck & 0x80) === 0x80
charFcheck = (fcheck & 0x40) === 0x40
charFthin  = (fcheck & 0x20) === 0x20

// Display position:
tempx = charX + charFdx * charFace
if ((charFood && faceL()) || (!charFood && faceR())): tempx -= 13
x = baseX + tempx
y = baseY + charY + charFdy
```

---

## kid.json

75 sequences, 241 framedefs. Kid is the player character.

### Sequences

**startrun**
```
ACT(1), FRAME(1), FRAME(2), FRAME(3), FRAME(4), CHX(8), FRAME(5), CHX(3), FRAME(6), CHX(3), GOTO(running,1)
```

**running**
```
ACT(1), FRAME(7), CHX(5), FRAME(8), CHX(1), TAP(1), FRAME(9), CHX(2), FRAME(10), CHX(4), FRAME(11), CHX(5),
FRAME(12), CHX(2), TAP(1), FRAME(13), CHX(3), FRAME(14), CHX(4), GOTO(running,1)
```

**stand**
```
ACT(0), FRAME(15), GOTO(stand,1)
```

**bump**
```
ACT(5), CHX(-2), FRAME(50), FRAME(51), FRAME(52), GOTO(stand,0)
```

**standjump**
```
ACT(1), FRAME(16), FRAME(17), CHX(2), FRAME(18), CHX(2), FRAME(19), CHX(2), FRAME(20), CHX(2), FRAME(21),
CHX(2), FRAME(22), CHX(7), FRAME(23), CHX(9), FRAME(24), CHX(5), CHY(-6), FRAME(25), CHX(1), CHY(6),
FRAME(26), CHX(4), JARD, TAP(1), FRAME(27), CHX(-3), FRAME(28), CHX(5), FRAME(29), TAP(1), FRAME(30),
FRAME(31), FRAME(32), FRAME(33), CHX(1), GOTO(stand,0)
```

**runjump**
```
ACT(1), TAP(1), FRAME(34), CHX(5), FRAME(35), CHX(6), FRAME(36), CHX(3), FRAME(37), CHX(5), TAP(1),
FRAME(38), CHX(7), FRAME(39), CHX(12), CHY(-3), FRAME(40), CHX(8), CHY(-9), FRAME(41), CHX(8), CHY(-2),
FRAME(42), CHX(4), CHY(11), FRAME(43), CHX(4), CHY(3), FRAME(44), CHX(5), JARD, TAP(1), GOTO(running,0)
```

**turn**
```
ACT(7), ABOUTFACE, CHX(6), FRAME(45), CHX(1), FRAME(46), CHX(2), FRAME(47), CHX(-1), FRAME(48), CHX(1),
FRAME(49), CHX(-2), FRAME(50), FRAME(51), FRAME(52), GOTO(stand,0)
```

**runturn**
```
ACT(1), CHX(1), FRAME(53), CHX(1), TAP(1), FRAME(54), CHX(8), FRAME(55), TAP(1), FRAME(56), CHX(7),
FRAME(57), CHX(3), FRAME(58), CHX(1), FRAME(59), FRAME(60), CHX(2), FRAME(61), CHX(-1), FRAME(62),
FRAME(63), FRAME(64), CHX(-1), FRAME(65), CHX(-14), ABOUTFACE, GOTO(running,14)
```

**jumphangmed**
```
ACT(1), FRAME(67), FRAME(68), FRAME(69), FRAME(70), FRAME(71), FRAME(72), FRAME(73), FRAME(74), FRAME(75),
FRAME(76), FRAME(77), ACT(2), FRAME(78), FRAME(79), FRAME(80), GOTO(hang,0)
```

**jumphanglong**
```
ACT(1), FRAME(67), FRAME(68), FRAME(69), FRAME(70), FRAME(71), FRAME(72), FRAME(73), FRAME(74), FRAME(75),
FRAME(76), FRAME(77), ACT(2), CHX(1), FRAME(78), CHX(2), FRAME(79), CHX(1), FRAME(80), GOTO(hang,0)
```

**hang**
```
ACT(2), FRAME(91), FRAME(90), FRAME(89), FRAME(88), FRAME(87), FRAME(87), FRAME(87), FRAME(88), FRAME(89),
FRAME(90), FRAME(91), FRAME(92), FRAME(93), FRAME(94), FRAME(95), FRAME(96), FRAME(97), FRAME(98), FRAME(99),
FRAME(97), FRAME(96), FRAME(95), FRAME(94), FRAME(93), FRAME(92), FRAME(91), FRAME(90), FRAME(89), FRAME(88),
FRAME(87), FRAME(88), FRAME(89), FRAME(90), FRAME(91), FRAME(92), FRAME(93), FRAME(94), FRAME(95), FRAME(96),
FRAME(95), FRAME(94), FRAME(93), FRAME(92), GOTO(hangdrop,0)
```

**hangdrop**
```
ACT(0), FRAME(81), FRAME(82), ACT(5), FRAME(83), ACT(1), JARD, TAP(0), FRAME(84), FRAME(85), CHX(3),
GOTO(stand,0)
```

**hangfall**
```
ACT(3), FRAME(81), CHY(6), FRAME(81), CHY(9), FRAME(81), CHY(12), CHX(2), SETFALL(0,12), GOTO(freefall,0)
```

**runstop**
```
ACT(1), FRAME(53), CHX(2), TAP(1), FRAME(54), CHX(7), FRAME(55), TAP(1), FRAME(56), CHX(2), FRAME(49),
CHX(-2), FRAME(50), FRAME(51), FRAME(52), GOTO(stand,0)
```

**turnrun**
```
ACT(1), CHX(-1), GOTO(startrun,0)
```

**stepfall**
```
ACT(3), CHX(1), CHY(3), IFWTLESS(stepfloat), FRAME(102), CHX(2), CHY(6), FRAME(103), CHX(-1), CHY(9),
FRAME(104), CHY(12), FRAME(105), CHX(-2), SETFALL(1,15), GOTO(freefall,0)
```

**stepfloat**
```
FRAME(102), CHX(2), CHY(3), FRAME(103), CHX(-1), CHY(4), FRAME(104), CHY(5), FRAME(105), CHX(-2),
SETFALL(1,6), GOTO(freefall,0)
```

**bumpfall**
```
ACT(4), CHX(1), CHY(3), IFWTLESS(bumpfloat), FRAME(102), CHX(2), CHY(6), FRAME(103), CHX(-1), CHY(9),
FRAME(104), CHY(12), FRAME(105), CHX(-2), SETFALL(0,15), GOTO(freefall,0)
```

**bumpfloat**
```
FRAME(102), CHX(2), CHY(3), FRAME(103), CHX(-1), CHY(4), FRAME(104), CHY(5), FRAME(105), CHX(-2),
SETFALL(0,6), GOTO(freefall,0)
```

**stepfall2**
```
CHX(1), GOTO(stepfall,0)
```

**freefall**
```
ACT(4), FRAME(106), GOTO(freefall,0)
```

**stoop**
```
ACT(1), CHX(1), FRAME(107), CHX(2), FRAME(108), FRAME(109), GOTO(stoop,5)
```

**rdiveroll**
```
ACT(1), CHX(1), FRAME(107), CHX(2), CHX(2), FRAME(108), CHX(2), FRAME(109), CHX(2), FRAME(109), CHX(2),
FRAME(109), GOTO(stoop,5)
```

**softland**
```
ACT(5), JARD, CHX(1), TAP(1), FRAME(107), CHX(2), FRAME(108), TAP(1), ACT(1), FRAME(109), GOTO(stoop,5)
```

**medland**
```
ACT(5), JARD, CHY(0), CHX(3), FRAME(108), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109),
FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109),
FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109),
FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), FRAME(109), CHX(1), FRAME(110), FRAME(110),
FRAME(110), FRAME(111), CHX(2), FRAME(112), FRAME(113), CHX(1), CHY(0), FRAME(114), CHY(0), FRAME(115),
FRAME(116), CHX(-4), FRAME(117), FRAME(118), FRAME(119), GOTO(stand,0)
```

**hardland**
```
ACT(5), JARD, CHY(-2), CHX(3), FRAME(185), DIE, FRAME(185), GOTO(hardland,6)
```

**crawl**
```
ACT(1), CHX(1), FRAME(110), FRAME(111), CHX(2), FRAME(112), CHX(2), FRAME(108), CHX(2), FRAME(109),
GOTO(stoop,5)
```

**standup**
```
ACT(5), CHX(1), FRAME(110), FRAME(111), CHX(2), FRAME(112), FRAME(113), CHX(1), FRAME(114), FRAME(115),
FRAME(116), CHX(-4), FRAME(117), FRAME(118), FRAME(119), GOTO(stand,0)
```

**testfoot**
```
FRAME(121), CHX(1), FRAME(122), FRAME(123), CHX(2), FRAME(124), CHX(4), FRAME(125), CHX(3), FRAME(126),
CHX(-4), FRAME(86), TAP(1), JARD, CHX(-4), FRAME(116), CHX(-2), FRAME(117), FRAME(118), FRAME(119),
GOTO(stand,0)
```

**step14**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), CHX(3), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132),
GOTO(stand,0)
```

**step13**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), CHX(2), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132),
GOTO(stand,0)
```

**step12**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), CHX(1), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132),
GOTO(stand,0)
```

**step11**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step10**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-2), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step9**
```
ACT(1), FRAME(121), GOTO(step10,3)
```

**step8**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(-1),
FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step7**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(2), FRAME(129),
FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step6**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(2), FRAME(124), CHX(2), FRAME(129),
FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step5**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(2), FRAME(124), CHX(1), FRAME(129),
FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step4**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(2), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step3**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(1), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step2**
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(132), GOTO(stand,0)
```

**step1**
```
ACT(1), FRAME(121), CHX(1), FRAME(132), GOTO(stand,0)
```

**climbup**
```
ACT(1), FRAME(135), FRAME(136), FRAME(137), FRAME(138), FRAME(139), FRAME(140), CHX(5), CHY(-63), UP,
FRAME(141), FRAME(142), FRAME(143), FRAME(144), FRAME(145), FRAME(146), FRAME(147), FRAME(148), ACT(5),
FRAME(149), ACT(1), FRAME(118), FRAME(119), CHX(1), GOTO(stand,0)
```

**climbstairs**
```
ACT(5), CHX(-5), CHY(-1), TAP(1), FRAME(217), FRAME(218), FRAME(219), CHX(1), FRAME(220), CHX(-4), CHY(-3),
TAP(1), FRAME(221), CHX(-4), CHY(-2), FRAME(222), FRAME(222), CHX(-2), CHY(-3), FRAME(223), FRAME(223),
CHX(-3), CHY(-8), TAP(1), FRAME(224), FRAME(224), CHX(-1), CHY(-1), FRAME(225), FRAME(225), CHX(-3), CHY(-4),
FRAME(226), FRAME(226), CHX(-1), CHY(-5), TAP(1), FRAME(227), FRAME(227), CHX(-2), CHY(-1), FRAME(228),
FRAME(228), FRAME(0), TAP(1), FRAME(0), FRAME(0), FRAME(0), FRAME(0), TAP(1), FRAME(0), FRAME(0), FRAME(0),
FRAME(0), TAP(1), FRAME(0), FRAME(0), FRAME(0), FRAME(0), TAP(1), NEXTLEVEL, FRAME(0), GOTO(climbstairs,61)
```

**climbfail**
```
FRAME(135), FRAME(136), FRAME(137), FRAME(137), FRAME(138), FRAME(138), FRAME(138), FRAME(138), FRAME(137),
FRAME(136), FRAME(135), CHX(-7), GOTO(hangdrop,0)
```

**pickupsword**
```
ACT(1), EFFECT, FRAME(229), FRAME(229), FRAME(229), FRAME(229), FRAME(229), FRAME(229), FRAME(230), FRAME(231),
FRAME(232), GOTO(resheathe,0)
```

**resheathe**
```
ACT(1), CHX(-5), FRAME(233), FRAME(234), FRAME(235), FRAME(236), FRAME(237), FRAME(238), FRAME(239),
FRAME(240), FRAME(133), FRAME(133), FRAME(134), FRAME(134), FRAME(134), FRAME(48), CHX(1), FRAME(49),
CHX(-2), ACT(5), FRAME(50), ACT(1), FRAME(51), FRAME(52), GOTO(stand,0)
```

**drinkpotion**
```
ACT(1), CHX(4), FRAME(191), FRAME(192), FRAME(193), FRAME(194), FRAME(195), FRAME(196), FRAME(197),
FRAME(198), FRAME(199), FRAME(200), FRAME(201), FRAME(202), FRAME(203), FRAME(204), FRAME(205), FRAME(205),
FRAME(205), EFFECT, FRAME(205), FRAME(205), FRAME(201), FRAME(198), CHX(-4), GOTO(stand,0)
```

**jumpup**
```
ACT(1), FRAME(67), FRAME(68), FRAME(69), FRAME(70), FRAME(71), FRAME(72), FRAME(73), FRAME(74), FRAME(75),
FRAME(76), FRAME(77), FRAME(78), ACT(0), JARU, FRAME(79), GOTO(hangdrop,0)
```

**jumpbackhang**
```
ACT(1), FRAME(67), FRAME(68), FRAME(69), FRAME(70), FRAME(71), FRAME(72), FRAME(73), FRAME(74), FRAME(75),
FRAME(76), CHX(-1), FRAME(77), ACT(2), CHX(-2), FRAME(78), CHX(-1), FRAME(79), CHX(-1), FRAME(80),
GOTO(hang,0)
```

**highjump**
```
ACT(1), FRAME(67), FRAME(68), FRAME(69), FRAME(70), FRAME(71), FRAME(72), FRAME(73), FRAME(74), FRAME(75),
FRAME(76), FRAME(77), FRAME(78), FRAME(79), CHY(-4), FRAME(79), CHY(-2), FRAME(79), FRAME(79), CHY(2),
FRAME(79), CHY(4), GOTO(hangdrop,0)
```

**superhighjump**
```
FRAME(67), FRAME(68), FRAME(69), FRAME(70), FRAME(71), FRAME(72), FRAME(73), FRAME(74), FRAME(75), FRAME(76),
CHY(-1), FRAME(77), CHY(-3), FRAME(78), CHY(-4), FRAME(79), CHY(-10), FRAME(79), CHY(-9), FRAME(79),
CHY(-8), FRAME(79), CHY(-7), FRAME(79), CHY(-6), FRAME(79), CHY(-5), FRAME(79), CHY(-4), FRAME(79),
CHY(-3), FRAME(79), CHY(-2), FRAME(79), CHY(-2), FRAME(79), CHY(-1), FRAME(79), CHY(-1), FRAME(79),
CHY(-1), FRAME(79), FRAME(79), FRAME(79), FRAME(79), CHY(1), FRAME(79), CHY(1), FRAME(79), CHY(2),
FRAME(79), CHY(2), FRAME(79), CHY(3), FRAME(79), CHY(4), FRAME(79), CHY(5), FRAME(79), CHY(6), FRAME(79),
SETFALL(0,6), GOTO(freefall,0)
```

**rjumpfall**
```
ACT(3), CHX(1), CHY(3), FRAME(102), CHX(3), CHY(6), FRAME(103), CHX(2), CHY(9), FRAME(104), CHX(3), CHY(12),
FRAME(105), SETFALL(3,15), GOTO(freefall,0)
```

**jumpfall**
```
ACT(3), CHX(1), CHY(3), FRAME(102), CHX(2), CHY(6), FRAME(103), CHX(1), CHY(9), FRAME(104), CHX(2), CHY(12),
FRAME(105), SETFALL(2,15), GOTO(freefall,0)
```

**climbdown**
```
ACT(1), FRAME(148), FRAME(145), FRAME(144), FRAME(143), FRAME(142), FRAME(141), CHX(-5), CHY(63), DOWN,
FRAME(140), FRAME(138), FRAME(136), FRAME(91), GOTO(hang,2)
```

**hangstraight**
```
ACT(6), TAP(2), FRAME(92), FRAME(93), FRAME(93), FRAME(92), FRAME(92), FRAME(91), GOTO(hangstraight,7)
```

**engarde**
```
ACT(1), CHX(2), FRAME(207), FRAME(208), CHX(2), FRAME(209), CHX(2), FRAME(210), CHX(3), ACT(1), TAP(0),
FRAME(158), GOTO(engarde,11)
```

**advance**
```
ACT(1), CHX(6), FRAME(164), FRAME(165), GOTO(engarde,9)
```

**fastsheathe**
```
ACT(1), CHX(-5), FRAME(234), FRAME(236), FRAME(238), FRAME(240), FRAME(134), CHX(-1), GOTO(stand,0)
```

**retreat**
```
ACT(1), CHX(-3), FRAME(160), CHX(-2), FRAME(157), GOTO(engarde,9)
```

**strike**
```
ACT(1), FRAME(151), FRAME(152), FRAME(153), FRAME(154), FRAME(155), FRAME(156), FRAME(157), GOTO(engarde,9)
```

**block**
```
FRAME(169), FRAME(150), GOTO(engarde,9)
```

**blocktostrike**
```
FRAME(162), GOTO(strike,2)
```

**stabbed**
```
ACT(5), FRAME(172), CHX(-1), CHY(1), FRAME(173), CHX(-1), FRAME(174), CHX(-1), CHY(2), CHX(-2), CHY(1),
CHX(-5), CHY(-4), GOTO(strike,6)
```

**stabkill**
```
ACT(5), GOTO(dropdead,0)
```

**dropdead**
```
ACT(1), DIE, FRAME(179), FRAME(180), FRAME(181), FRAME(182), CHX(1), FRAME(183), CHX(-4), FRAME(185),
GOTO(dropdead,9)
```

**turnengarde**
```
ACT(5), ABOUTFACE, CHX(5), GOTO(retreat,0)
```

**beginturnengarde**
```
ACT(5), ABOUTFACE, CHX(5), GOTO(engarde,0)
```

**turndraw**
```
ACT(7), ABOUTFACE, CHX(6), FRAME(45), CHX(1), FRAME(46), GOTO(engarde,0)
```

**striketoblock**
```
FRAME(159), FRAME(160), GOTO(block,1)
```

**blockedstrike**
```
ACT(1), FRAME(167), GOTO(strike,5)
```

**impale**
```
ACT(1), JARD, CHX(4), FRAME(177), DIE, FRAME(177), GOTO(impale,5)
```

**halve**
```
ACT(1), FRAME(178), DIE, FRAME(178), GOTO(halve,3)
```

**falldead**
```
ACT(5), FRAME(185), DIE, FRAME(185), GOTO(falldead,3)
```

### Framedefs

Entries marked `(comment)` are placeholders with no physics data. `fsword` column is omitted when absent.

| idx | fdx | fdy | fcheck | fsword | note |
|---|---|---|---|---|---|
| 0 | 0 | 0 | 0x00 | | |
| 1 | 1 | 0 | 0xC4 | | |
| 2 | 1 | 0 | 0x44 | | |
| 3 | 3 | 0 | 0x47 | | |
| 4 | 4 | 0 | 0x48 | | |
| 5 | 0 | 0 | 0xE6 | | |
| 6 | 0 | 0 | 0x49 | | |
| 7 | 0 | 0 | 0x4A | | |
| 8 | 0 | 0 | 0xC5 | | |
| 9 | 0 | 0 | 0x44 | | |
| 10 | 0 | 0 | 0x47 | | |
| 11 | 0 | 0 | 0x4B | | |
| 12 | 0 | 0 | 0x43 | | |
| 13 | 0 | 0 | 0xC3 | | |
| 14 | 0 | 0 | 0x47 | | |
| 15 | 0 | 0 | 0x43 | | |
| 16 | 0 | 0 | 0xC3 | | |
| 17 | 0 | 0 | 0x44 | | |
| 18 | 0 | 0 | 0x46 | | |
| 19 | 0 | 0 | 0x48 | | |
| 20 | 0 | 0 | 0x89 | | |
| 21 | 0 | 0 | 0x0B | | |
| 22 | 0 | 0 | 0x8B | | |
| 23 | 0 | 0 | 0x11 | | |
| 24 | 0 | 0 | 0x07 | | |
| 25 | 0 | 0 | 0x05 | | |
| 26 | 0 | 0 | 0xC1 | | |
| 27 | 0 | 0 | 0xC6 | | |
| 28 | 0 | 0 | 0x43 | | |
| 29 | 0 | 0 | 0x48 | | |
| 30 | 0 | 0 | 0x42 | | |
| 31 | 0 | 0 | 0x42 | | |
| 32 | 0 | 0 | 0xC2 | | |
| 33 | 0 | 0 | 0xC2 | | |
| 34 | 0 | 0 | 0x43 | | |
| 35 | 0 | 0 | 0x48 | | |
| 36 | 0 | 0 | 0xCE | | |
| 37 | 0 | 0 | 0xC1 | | |
| 38 | 0 | 0 | 0x45 | | |
| 39 | 0 | 0 | 0x8E | | |
| 40 | 0 | 0 | 0x0B | | |
| 41 | 0 | 0 | 0x8B | | |
| 42 | 0 | 0 | 0x8A | | |
| 43 | 0 | 0 | 0x01 | | |
| 44 | 0 | 0 | 0xC4 | | |
| 45 | 0 | 0 | 0xC3 | | |
| 46 | 0 | 0 | 0xC3 | | |
| 47 | 0 | 0 | 0xA5 | | |
| 48 | 0 | 0 | 0xA4 | | |
| 49 | 0 | 0 | 0x66 | | |
| 50 | 4 | 0 | 0x67 | | |
| 51 | 3 | 0 | 0x66 | | |
| 52 | 1 | 0 | 0x44 | | |
| 53 | 0 | 0 | 0xC2 | | |
| 54 | 0 | 0 | 0x41 | | |
| 55 | 0 | 0 | 0x42 | | |
| 56 | 0 | 0 | 0x00 | | |
| 57 | 0 | 0 | 0x00 | | |
| 58 | 0 | 0 | 0x80 | | |
| 59 | 0 | 0 | 0x00 | | |
| 60 | 0 | 0 | 0x80 | | |
| 61 | 0 | 0 | 0x00 | | |
| 62 | 0 | 0 | 0x80 | | |
| 63 | 0 | 0 | 0x00 | | |
| 64 | 0 | 0 | 0x00 | | |
| 65 | 0 | 0 | 0x80 | | |
| 66 | 0 | 0 | 0x00 | | |
| 67 | -2 | 0 | 0x41 | | |
| 68 | -2 | 0 | 0x41 | | |
| 69 | -1 | 0 | 0xC2 | | |
| 70 | -2 | 0 | 0x42 | | |
| 71 | -2 | 0 | 0x41 | | |
| 72 | -2 | 0 | 0x41 | | |
| 73 | -2 | 0 | 0x41 | | |
| 74 | -1 | 0 | 0x07 | | |
| 75 | -1 | 0 | 0x05 | | |
| 76 | 2 | 0 | 0x07 | | |
| 77 | 2 | 0 | 0x07 | | |
| 78 | 2 | -3 | 0x00 | | |
| 79 | 2 | -10 | 0x00 | | |
| 80 | 2 | -11 | 0x80 | | |
| 81 | 3 | -2 | 0x43 | | |
| 82 | 3 | 0 | 0xC3 | | |
| 83 | 3 | 0 | 0xC3 | | |
| 84 | 3 | 0 | 0x63 | | |
| 85 | 4 | 0 | 0xE3 | | |
| 86 | 0 | 0 | 0x00 | | |
| 87 | 7 | -14 | 0x80 | | |
| 88 | 7 | -12 | 0x80 | | |
| 89 | 4 | -12 | 0x00 | | |
| 90 | 3 | -10 | 0x80 | | |
| 91 | 2 | -10 | 0x80 | | |
| 92 | 1 | -10 | 0x80 | | |
| 93 | 0 | -11 | 0x00 | | |
| 94 | -1 | -12 | 0x00 | | |
| 95 | -1 | -14 | 0x00 | | |
| 96 | -1 | -14 | 0x00 | | |
| 97 | -1 | -15 | 0x80 | | |
| 98 | -1 | -15 | 0x80 | | |
| 99 | 0 | -15 | 0x00 | | |
| 100 | 0 | 0 | 0x00 | | |
| 101 | 0 | 0 | 0x00 | | |
| 102 | 0 | 0 | 0xC6 | | |
| 103 | 0 | 0 | 0x46 | | |
| 104 | 0 | 0 | 0xC5 | | |
| 105 | 0 | 0 | 0x45 | | |
| 106 | 0 | 0 | 0xC2 | | |
| 107 | 0 | 0 | 0xC4 | | |
| 108 | 0 | 0 | 0xC5 | | |
| 109 | 0 | 0 | 0x46 | | |
| 110 | 0 | 0 | 0x47 | | |
| 111 | 0 | 0 | 0x47 | | |
| 112 | 0 | 0 | 0x49 | | |
| 113 | 0 | 0 | 0xC8 | | |
| 114 | 0 | 0 | 0xC9 | | |
| 115 | 0 | 0 | 0x49 | | |
| 116 | 0 | 0 | 0x45 | | |
| 117 | 2 | 0 | 0x45 | | |
| 118 | 2 | 0 | 0xC5 | | |
| 119 | 0 | 0 | 0xC3 | | |
| 120 | 0 | 0 | 0x00 | | |
| 121 | 0 | 0 | 0x43 | | |
| 122 | 0 | 0 | 0xC4 | | |
| 123 | 0 | 0 | 0xC5 | | |
| 124 | 0 | 0 | 0x48 | | |
| 125 | 0 | 0 | 0x6C | | |
| 126 | 0 | 0 | 0xEF | | |
| 127 | 0 | 0 | 0x63 | | |
| 128 | 0 | 0 | 0xC3 | | |
| 129 | 0 | 0 | 0x43 | | |
| 130 | 0 | 0 | 0x43 | | |
| 131 | 0 | 0 | 0x44 | | |
| 132 | 0 | 0 | 0x44 | | |
| 133 | 0 | 1 | 0xC1 | | |
| 134 | 0 | 1 | 0xC7 | | |
| 135 | 0 | -12 | 0x01 | | |
| 136 | 0 | -21 | 0x00 | | |
| 137 | 1 | -26 | 0x80 | | |
| 138 | 4 | -32 | 0x80 | | |
| 139 | 6 | -36 | 0x81 | | |
| 140 | 7 | -41 | 0x82 | | |
| 141 | 2 | 17 | 0x42 | | |
| 142 | 4 | 9 | 0xC4 | | |
| 143 | 4 | 5 | 0xC9 | | |
| 144 | 4 | 4 | 0xC8 | | |
| 145 | 5 | 0 | 0x69 | | |
| 146 | 5 | 0 | 0xE9 | | |
| 147 | 5 | 0 | 0xE8 | | |
| 148 | 5 | 0 | 0x69 | | |
| 149 | 5 | 0 | 0x69 | | |
| 150 | 0 | 2 | 0x80 | 16 | |
| 151 | 0 | 2 | 0x80 | 26 | |
| 152 | 3 | 2 | 0x00 | 18 | |
| 153 | 7 | 2 | 0xC4 | 22 | |
| 154 | 10 | 2 | 0x00 | 21 | |
| 155 | 7 | 2 | 0x80 | 23 | |
| 156 | 4 | 2 | 0x80 | 25 | |
| 157 | 0 | 2 | 0xCF | 24 | |
| 158 | 0 | 2 | 0xCD | 15 | |
| 159 | 3 | 2 | 0x00 | 20 | |
| 160 | 3 | 2 | 0x00 | 31 | |
| 161 | — | — | — | | comment: "161 equals 150" |
| 162 | 0 | 2 | 0x80 | 17 | |
| 163 | 0 | 2 | 0x00 | 32 | |
| 164 | 0 | 2 | 0x80 | 33 | |
| 165 | 2 | 2 | 0xC3 | 34 | |
| 166 | — | — | — | | comment: "166 equals 15" |
| 167 | 7 | 2 | 0x80 | 19 | |
| 168 | 1 | 2 | 0x80 | 14 | |
| 169 | 0 | 2 | 0x80 | 27 | |
| 170 | — | — | — | | comment: "170 equals 158" |
| 171 | — | — | — | | comment: "171 equals 158" |
| 172 | 0 | 0 | 0xC6 | | |
| 173 | 0 | 0 | 0x46 | | |
| 174 | 0 | 0 | 0xC5 | 45 | |
| 175 | — | — | — | | comment: "175 jumpfall unused" |
| 176 | — | — | — | | comment: "176 jumpfall unused" |
| 177 | 0 | 3 | 0x8A | | |
| 178 | 4 | 3 | 0x87 | | |
| 179 | 0 | 1 | 0x44 | | |
| 180 | 0 | 1 | 0x44 | | |
| 181 | 0 | 1 | 0x44 | | |
| 182 | 0 | 1 | 0x47 | | |
| 183 | 0 | 7 | 0x4B | | |
| 184 | — | — | — | | comment: "unused" |
| 185 | 4 | 7 | 0x49 | | |
| 186 | — | — | — | | comment: "mouse" |
| 187 | — | — | — | | comment: "mouse" |
| 188 | — | — | — | | comment: "mouse" |
| 189 | — | — | — | | comment: "unused" |
| 190 | — | — | — | | comment: "unused" |
| 191 | 0 | 0 | 0x00 | | |
| 192 | 0 | 0 | 0x00 | | |
| 193 | 0 | 0 | 0x80 | | |
| 194 | 0 | 0 | 0x00 | | |
| 195 | -1 | 0 | 0x00 | | |
| 196 | -1 | 0 | 0x00 | | |
| 197 | -1 | 0 | 0x00 | | |
| 198 | -4 | 0 | 0x00 | | |
| 199 | -4 | 0 | 0x80 | | |
| 200 | -4 | 0 | 0x00 | | |
| 201 | -4 | 0 | 0x00 | | |
| 202 | -4 | 0 | 0x00 | | |
| 203 | -4 | 0 | 0x00 | | |
| 204 | -5 | 0 | 0x00 | | |
| 205 | -5 | 0 | 0x00 | | |
| 206 | — | — | — | | comment: "unused" |
| 207 | 0 | 1 | 0x46 | | |
| 208 | 0 | 1 | 0xC6 | | |
| 209 | 0 | 1 | 0xC8 | | |
| 210 | 0 | 1 | 0x4A | | |
| 211 | — | — | — | | comment: "unused" |
| 212 | — | — | — | | comment: "unused" |
| 213 | — | — | — | | comment: "unused" |
| 214 | — | — | — | | comment: "unused" |
| 215 | — | — | — | | comment: "unused" |
| 216 | — | — | — | | comment: "unused" |
| 217 | 0 | 0 | 0x80 | | |
| 218 | 0 | 0 | 0x00 | | |
| 219 | 0 | 0 | 0x00 | | |
| 220 | 0 | 0 | 0x00 | | |
| 221 | 0 | 0 | 0x80 | | |
| 222 | 0 | 0 | 0x00 | | |
| 223 | 0 | 0 | 0x00 | | |
| 224 | 0 | 0 | 0x00 | | |
| 225 | 0 | 0 | 0x80 | | |
| 226 | 0 | 0 | 0x00 | | |
| 227 | 0 | 0 | 0x80 | | |
| 228 | 0 | 0 | 0x00 | | |
| 229 | 1 | 1 | 0xC3 | 35 | |
| 230 | 0 | 1 | 0x49 | 36 | |
| 231 | 0 | 1 | 0xC3 | 37 | |
| 232 | 0 | 1 | 0x49 | 38 | |
| 233 | 0 | 1 | 0xC3 | 39 | |
| 234 | 1 | 1 | 0x49 | 40 | |
| 235 | 1 | 1 | 0x43 | 41 | |
| 236 | 1 | 1 | 0xC9 | 42 | |
| 237 | 4 | 1 | 0xC6 | | |
| 238 | 3 | 1 | 0xCA | | |
| 239 | 1 | 1 | 0x43 | | |
| 240 | 1 | 1 | 0xC8 | | |

---

## fighter.json

23 sequences, 36 framedefs. Fighter is the sword-fighting enemy class. Registers `CMD_DIE`, `CMD_SETFALL`, `CMD_ACT`, and the `_GOTO`/`_ABOUTFACE` base opcodes.

### Sequences

**stand**
```
ACT(1), FRAME(16), GOTO(stand,1)
```

**engarde**
```
ACT(1), TAP(0), FRAME(8), FRAME(20), FRAME(21), GOTO(engarde,4)
```

**resheathe**
```
GOTO(stand,0)
```

**advance**
```
ACT(1), CHX(2), FRAME(13), CHX(4), FRAME(14), FRAME(15), GOTO(engarde,0)
```

**retreat**
```
ACT(1), CHX(-3), FRAME(10), CHX(-2), FRAME(7), GOTO(engarde,0)
```

**strike**
```
ACT(1), FRAME(18), ACT(1), FRAME(1), FRAME(2), FRAME(3), FRAME(4), FRAME(5), FRAME(6), FRAME(7),
GOTO(engarde,0)
```

**block**
```
FRAME(19), FRAME(0), GOTO(engarde,0)
```

**blocktostrike**
```
FRAME(12), GOTO(strike,4)
```

**stabbed**
```
ACT(5), FRAME(22), CHX(-1), CHY(1), FRAME(23), CHX(-1), FRAME(24), CHX(-1), CHY(2), CHX(-2), CHY(1),
CHX(-5), CHY(-4), GOTO(strike,8)
```

**stabkill**
```
ACT(5), GOTO(dropdead,0)
```

**dropdead**
```
ACT(1), DIE, FRAME(29), FRAME(30), FRAME(31), FRAME(32), CHX(1), FRAME(33), CHX(-4), FRAME(35),
GOTO(dropdead,9)
```

**turnengarde**
```
ACT(5), ABOUTFACE, CHX(5), GOTO(retreat,0)
```

**striketoblock**
```
FRAME(9), FRAME(10), GOTO(block,1)
```

**blockedstrike**
```
ACT(1), FRAME(17), GOTO(strike,7)
```

**stepfall**
```
ACT(3), CHX(1), CHY(3), FRAME(23), CHX(2), CHY(6), FRAME(23), CHX(-1), CHX(-1), CHY(9), FRAME(23), CHY(12),
FRAME(23), CHX(-2), SETFALL(1,15), GOTO(freefall,0)
```

**freefall**
```
ACT(4), FRAME(23), GOTO(freefall,0)
```

**impale**
```
ACT(1), JARD, CHX(4), FRAME(27), DIE, FRAME(27), GOTO(impale,5)
```

**halve**
```
ACT(1), FRAME(28), DIE, FRAME(28), GOTO(halve,3)
```

**turn**
```
ACT(5), ABOUTFACE, CHX(5), GOTO(stand,0)
```

**falldead**
```
ACT(5), FRAME(35), DIE, FRAME(35), GOTO(falldead,3)
```

**arise**
```
ACT(5), FRAME(27), CHY(-2), CHX(-5), FRAME(28), CHY(2), GOTO(engarde,1)
```

**laydown**
```
ACT(1), FRAME(27), GOTO(laydown,1)
```

**bump**
```
ACT(5), CHX(-2), FRAME(21), FRAME(21), FRAME(21), GOTO(engarde,0)
```

### Framedefs

| idx | fdx | fdy | fcheck | fsword | note |
|---|---|---|---|---|---|
| 0 | 2 | 1 | 0x00 | 13 | |
| 1 | 3 | 1 | 0x00 | 1 | |
| 2 | 4 | 1 | 0x00 | 2 | |
| 3 | 7 | 1 | 0x44 | 3 | |
| 4 | 10 | 1 | 0x00 | 4 | |
| 5 | 7 | 1 | 0x80 | 5 | |
| 6 | 4 | 1 | 0x80 | 6 | |
| 7 | 0 | 1 | 0x80 | 7 | |
| 8 | 0 | 1 | 0xCD | 8 | |
| 9 | 7 | 1 | 0x80 | 11 | |
| 10 | 3 | 1 | 0x00 | 12 | |
| 11 | 2 | 1 | 0x00 | 13 | |
| 12 | 2 | 1 | 0x00 | | |
| 13 | 0 | 1 | 0x00 | 28 | |
| 14 | 0 | 1 | 0x80 | 29 | |
| 15 | 2 | 1 | 0xC3 | 30 | |
| 16 | -1 | 1 | 0x48 | 9 | |
| 17 | 7 | 1 | 0x80 | 10 | |
| 18 | 3 | 1 | 0x80 | 14 | |
| 19 | 0 | 1 | 0x80 | 8 | |
| 20 | 0 | 1 | 0xCD | 8 | |
| 21 | 0 | 1 | 0xCD | 8 | |
| 22 | 0 | 0 | 0xC6 | 47 | |
| 23 | 0 | 0 | 0x46 | 48 | |
| 24 | 0 | 0 | 0xC5 | 49 | |
| 25 | 0 | 0 | 0xC5 | 49 | |
| 26 | 0 | 0 | 0xC5 | 49 | |
| 27 | 0 | 3 | 0x8A | | |
| 28 | 4 | 4 | 0x87 | | |
| 29 | -2 | 1 | 0x44 | | |
| 30 | -2 | 1 | 0x44 | | |
| 31 | -2 | 1 | 0x44 | | |
| 32 | -2 | 2 | 0x47 | | |
| 33 | -2 | 2 | 0x4A | | |
| 34 | — | — | — | | (empty placeholder) |
| 35 | 3 | 4 | 0xC9 | | |

---

## shadow.json

51 sequences, 241 framedefs. Shadow is the player's mirror-double enemy. Uses the same sprite atlas as Kid so frame numbers correspond to the same sprites, but the framedef physics data differs in the sword-fighting frame range (150–185) and a few death frames.

### Sequences

Shadow sequences that are identical to kid.json are noted. All 51 sequences are listed below.

**startrun** — identical to kid
```
ACT(1), FRAME(1), FRAME(2), FRAME(3), FRAME(4), CHX(8), FRAME(5), CHX(3), FRAME(6), CHX(3), GOTO(running,1)
```

**running** — identical to kid
```
ACT(1), FRAME(7), CHX(5), FRAME(8), CHX(1), TAP(1), FRAME(9), CHX(2), FRAME(10), CHX(4), FRAME(11), CHX(5),
FRAME(12), CHX(2), TAP(1), FRAME(13), CHX(3), FRAME(14), CHX(4), GOTO(running,1)
```

**stand** — identical to kid
```
ACT(0), FRAME(15), GOTO(stand,1)
```

**bump** — identical to kid
```
ACT(5), CHX(-2), FRAME(50), FRAME(51), FRAME(52), GOTO(stand,0)
```

**standjump** — identical to kid
```
ACT(1), FRAME(16), FRAME(17), CHX(2), FRAME(18), CHX(2), FRAME(19), CHX(2), FRAME(20), CHX(2), FRAME(21),
CHX(2), FRAME(22), CHX(7), FRAME(23), CHX(9), FRAME(24), CHX(5), CHY(-6), FRAME(25), CHX(1), CHY(6),
FRAME(26), CHX(4), JARD, TAP(1), FRAME(27), CHX(-3), FRAME(28), CHX(5), FRAME(29), TAP(1), FRAME(30),
FRAME(31), FRAME(32), FRAME(33), CHX(1), GOTO(stand,0)
```

**runjump** — identical to kid
```
ACT(1), TAP(1), FRAME(34), CHX(5), FRAME(35), CHX(6), FRAME(36), CHX(3), FRAME(37), CHX(5), TAP(1),
FRAME(38), CHX(7), FRAME(39), CHX(12), CHY(-3), FRAME(40), CHX(8), CHY(-9), FRAME(41), CHX(8), CHY(-2),
FRAME(42), CHX(4), CHY(11), FRAME(43), CHX(4), CHY(3), FRAME(44), CHX(5), JARD, TAP(1), GOTO(running,0)
```

**turn** — identical to kid
```
ACT(7), ABOUTFACE, CHX(6), FRAME(45), CHX(1), FRAME(46), CHX(2), FRAME(47), CHX(-1), FRAME(48), CHX(1),
FRAME(49), CHX(-2), FRAME(50), FRAME(51), FRAME(52), GOTO(stand,0)
```

**runturn** — identical to kid
```
ACT(1), CHX(1), FRAME(53), CHX(1), TAP(1), FRAME(54), CHX(8), FRAME(55), TAP(1), FRAME(56), CHX(7),
FRAME(57), CHX(3), FRAME(58), CHX(1), FRAME(59), FRAME(60), CHX(2), FRAME(61), CHX(-1), FRAME(62),
FRAME(63), FRAME(64), CHX(-1), FRAME(65), CHX(-14), ABOUTFACE, GOTO(running,14)
```

**runstop** — identical to kid
```
ACT(1), FRAME(53), CHX(2), TAP(1), FRAME(54), CHX(7), FRAME(55), TAP(1), FRAME(56), CHX(2), FRAME(49),
CHX(-2), FRAME(50), FRAME(51), FRAME(52), GOTO(stand,0)
```

**turnrun** — identical to kid
```
ACT(1), CHX(-1), GOTO(startrun,0)
```

**stepfall** — identical to kid
```
ACT(3), CHX(1), CHY(3), IFWTLESS(stepfloat), FRAME(102), CHX(2), CHY(6), FRAME(103), CHX(-1), CHY(9),
FRAME(104), CHY(12), FRAME(105), CHX(-2), SETFALL(1,15), GOTO(freefall,0)
```

**freefall** — identical to kid
```
ACT(4), FRAME(106), GOTO(freefall,0)
```

**stoop** — identical to kid
```
ACT(1), CHX(1), FRAME(107), CHX(2), FRAME(108), FRAME(109), GOTO(stoop,5)
```

**softland** — identical to kid
```
ACT(5), JARD, CHX(1), TAP(1), FRAME(107), CHX(2), FRAME(108), TAP(1), ACT(1), FRAME(109), GOTO(stoop,5)
```

**standup** — identical to kid
```
ACT(5), CHX(1), FRAME(110), FRAME(111), CHX(2), FRAME(112), FRAME(113), CHX(1), FRAME(114), FRAME(115),
FRAME(116), CHX(-4), FRAME(117), FRAME(118), FRAME(119), GOTO(stand,0)
```

**step14** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), CHX(3), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132),
GOTO(stand,0)
```

**step13** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), CHX(2), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132),
GOTO(stand,0)
```

**step12** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), CHX(1), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132),
GOTO(stand,0)
```

**step11** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-1), FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step10** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(3),
FRAME(126), CHX(-2), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step9** — identical to kid
```
ACT(1), FRAME(121), GOTO(step10,3)
```

**step8** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(4), FRAME(125), CHX(-1),
FRAME(127), FRAME(128), FRAME(129), FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step7** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(3), FRAME(124), CHX(2), FRAME(129),
FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step6** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(2), FRAME(124), CHX(2), FRAME(129),
FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step5** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(2), FRAME(124), CHX(1), FRAME(129),
FRAME(130), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step4** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(2), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step3** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(123), CHX(1), FRAME(131), FRAME(132), GOTO(stand,0)
```

**step2** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(122), CHX(1), FRAME(132), GOTO(stand,0)
```

**step1** — identical to kid
```
ACT(1), FRAME(121), CHX(1), FRAME(132), GOTO(stand,0)
```

**resheathe** — identical to kid
```
ACT(1), CHX(-5), FRAME(233), FRAME(234), FRAME(235), FRAME(236), FRAME(237), FRAME(238), FRAME(239),
FRAME(240), FRAME(133), FRAME(133), FRAME(134), FRAME(134), FRAME(134), FRAME(48), CHX(1), FRAME(49),
CHX(-2), ACT(5), FRAME(50), ACT(1), FRAME(51), FRAME(52), GOTO(stand,0)
```

**drinkpotion** — identical to kid
```
ACT(1), CHX(4), FRAME(191), FRAME(192), FRAME(193), FRAME(194), FRAME(195), FRAME(196), FRAME(197),
FRAME(198), FRAME(199), FRAME(200), FRAME(201), FRAME(202), FRAME(203), FRAME(204), FRAME(205), FRAME(205),
FRAME(205), EFFECT, FRAME(205), FRAME(205), FRAME(201), FRAME(198), CHX(-4), GOTO(stand,0)
```

**engarde** — identical to kid
```
ACT(1), CHX(2), FRAME(207), FRAME(208), CHX(2), FRAME(209), CHX(2), FRAME(210), CHX(3), ACT(1), TAP(0),
FRAME(158), GOTO(engarde,11)
```

**advance** — identical to kid
```
ACT(1), CHX(6), FRAME(164), FRAME(165), GOTO(engarde,9)
```

**fastsheathe** — identical to kid
```
ACT(1), CHX(-5), FRAME(234), FRAME(236), FRAME(238), FRAME(240), FRAME(134), CHX(-1), GOTO(stand,0)
```

**retreat** — identical to kid
```
ACT(1), CHX(-3), FRAME(160), CHX(-2), FRAME(157), GOTO(engarde,9)
```

**strike** — identical to kid
```
ACT(1), FRAME(151), FRAME(152), FRAME(153), FRAME(154), FRAME(155), FRAME(156), FRAME(157), GOTO(engarde,9)
```

**block** — identical to kid
```
FRAME(169), FRAME(150), GOTO(engarde,9)
```

**blocktostrike** — identical to kid
```
FRAME(162), GOTO(strike,2)
```

**stabbed** — identical to kid
```
ACT(5), FRAME(172), CHX(-1), CHY(1), FRAME(173), CHX(-1), FRAME(174), CHX(-1), CHY(2), CHX(-2), CHY(1),
CHX(-5), CHY(-4), GOTO(strike,6)
```

**stabkill** — identical to kid
```
ACT(5), GOTO(dropdead,0)
```

**dropdead** — identical to kid
```
ACT(1), DIE, FRAME(179), FRAME(180), FRAME(181), FRAME(182), CHX(1), FRAME(183), CHX(-4), FRAME(185),
GOTO(dropdead,9)
```

**turnengarde** — identical to kid
```
ACT(5), ABOUTFACE, CHX(5), GOTO(retreat,0)
```

**turndraw** — identical to kid
```
ACT(7), ABOUTFACE, CHX(6), FRAME(45), CHX(1), FRAME(46), GOTO(engarde,0)
```

**striketoblock** — identical to kid
```
FRAME(159), FRAME(160), GOTO(block,1)
```

**blockedstrike** — identical to kid
```
ACT(1), FRAME(167), GOTO(strike,5)
```

**impale** — identical to kid
```
ACT(1), JARD, CHX(4), FRAME(177), DIE, FRAME(177), GOTO(impale,5)
```

**halve** — identical to kid
```
ACT(1), FRAME(178), DIE, FRAME(178), GOTO(halve,3)
```

**falldead** — identical to kid
```
ACT(5), FRAME(185), DIE, FRAME(185), GOTO(falldead,3)
```

**runjumpdown** (shadow-only)
```
ACT(1), TAP(1), FRAME(42), CHX(4), CHY(11), FRAME(43), CHX(4), CHY(3), FRAME(44), CHX(5), JARD, TAP(1),
GOTO(running,0)
```

**softlandStandup** (shadow-only)
```
ACT(5), JARD, CHX(1), TAP(1), FRAME(107), CHX(2), FRAME(108), TAP(1), ACT(1), FRAME(109),
GOTO(stoopStandup,5)
```

**stoopStandup** (shadow-only)
```
ACT(1), CHX(1), FRAME(107), CHX(2), FRAME(108), FRAME(109), GOTO(standup,5)
```

### Framedefs

Shadow shares frame indices with Kid (same sprite atlas) but the physics data in the sword-combat range (150–185) and death frames differs. Frames 0–149 and 186–240 are identical to kid.json. The complete table is provided below.

| idx | fdx | fdy | fcheck | fsword | note |
|---|---|---|---|---|---|
| 0 | 0 | 0 | 0x00 | | |
| 1 | 1 | 0 | 0xC4 | | |
| 2 | 1 | 0 | 0x44 | | |
| 3 | 3 | 0 | 0x47 | | |
| 4 | 4 | 0 | 0x48 | | |
| 5 | 0 | 0 | 0xE6 | | |
| 6 | 0 | 0 | 0x49 | | |
| 7 | 0 | 0 | 0x4A | | |
| 8 | 0 | 0 | 0xC5 | | |
| 9 | 0 | 0 | 0x44 | | |
| 10 | 0 | 0 | 0x47 | | |
| 11 | 0 | 0 | 0x4B | | |
| 12 | 0 | 0 | 0x43 | | |
| 13 | 0 | 0 | 0xC3 | | |
| 14 | 0 | 0 | 0x47 | | |
| 15 | 0 | 0 | 0x43 | | |
| 16 | 0 | 0 | 0xC3 | | |
| 17 | 0 | 0 | 0x44 | | |
| 18 | 0 | 0 | 0x46 | | |
| 19 | 0 | 0 | 0x48 | | |
| 20 | 0 | 0 | 0x89 | | |
| 21 | 0 | 0 | 0x0B | | |
| 22 | 0 | 0 | 0x8B | | |
| 23 | 0 | 0 | 0x11 | | |
| 24 | 0 | 0 | 0x07 | | |
| 25 | 0 | 0 | 0x05 | | |
| 26 | 0 | 0 | 0xC1 | | |
| 27 | 0 | 0 | 0xC6 | | |
| 28 | 0 | 0 | 0x43 | | |
| 29 | 0 | 0 | 0x48 | | |
| 30 | 0 | 0 | 0x42 | | |
| 31 | 0 | 0 | 0x42 | | |
| 32 | 0 | 0 | 0xC2 | | |
| 33 | 0 | 0 | 0xC2 | | |
| 34 | 0 | 0 | 0x43 | | |
| 35 | 0 | 0 | 0x48 | | |
| 36 | 0 | 0 | 0xCE | | |
| 37 | 0 | 0 | 0xC1 | | |
| 38 | 0 | 0 | 0x45 | | |
| 39 | 0 | 0 | 0x8E | | |
| 40 | 0 | 0 | 0x0B | | |
| 41 | 0 | 0 | 0x8B | | |
| 42 | 0 | 0 | 0x8A | | |
| 43 | 0 | 0 | 0x01 | | |
| 44 | 0 | 0 | 0xC4 | | |
| 45 | 0 | 0 | 0xC3 | | |
| 46 | 0 | 0 | 0xC3 | | |
| 47 | 0 | 0 | 0xA5 | | |
| 48 | 0 | 0 | 0xA4 | | |
| 49 | 0 | 0 | 0x66 | | |
| 50 | 4 | 0 | 0x67 | | |
| 51 | 3 | 0 | 0x66 | | |
| 52 | 1 | 0 | 0x44 | | |
| 53 | 0 | 0 | 0xC2 | | |
| 54 | 0 | 0 | 0x41 | | |
| 55 | 0 | 0 | 0x42 | | |
| 56 | 0 | 0 | 0x00 | | |
| 57 | 0 | 0 | 0x00 | | |
| 58 | 0 | 0 | 0x80 | | |
| 59 | 0 | 0 | 0x00 | | |
| 60 | 0 | 0 | 0x80 | | |
| 61 | 0 | 0 | 0x00 | | |
| 62 | 0 | 0 | 0x80 | | |
| 63 | 0 | 0 | 0x00 | | |
| 64 | 0 | 0 | 0x00 | | |
| 65 | 0 | 0 | 0x80 | | |
| 66 | 0 | 0 | 0x00 | | |
| 67 | -2 | 0 | 0x41 | | |
| 68 | -2 | 0 | 0x41 | | |
| 69 | -1 | 0 | 0xC2 | | |
| 70 | -2 | 0 | 0x42 | | |
| 71 | -2 | 0 | 0x41 | | |
| 72 | -2 | 0 | 0x41 | | |
| 73 | -2 | 0 | 0x41 | | |
| 74 | -1 | 0 | 0x07 | | |
| 75 | -1 | 0 | 0x05 | | |
| 76 | 2 | 0 | 0x07 | | |
| 77 | 2 | 0 | 0x07 | | |
| 78 | 2 | -3 | 0x00 | | |
| 79 | 2 | -10 | 0x00 | | |
| 80 | 2 | -11 | 0x80 | | |
| 81 | 3 | -2 | 0x43 | | |
| 82 | 3 | 0 | 0xC3 | | |
| 83 | 3 | 0 | 0xC3 | | |
| 84 | 3 | 0 | 0x63 | | |
| 85 | 4 | 0 | 0xE3 | | |
| 86 | 0 | 0 | 0x00 | | |
| 87 | 7 | -14 | 0x80 | | |
| 88 | 7 | -12 | 0x80 | | |
| 89 | 4 | -12 | 0x00 | | |
| 90 | 3 | -10 | 0x80 | | |
| 91 | 2 | -10 | 0x80 | | |
| 92 | 1 | -10 | 0x80 | | |
| 93 | 0 | -11 | 0x00 | | |
| 94 | -1 | -12 | 0x00 | | |
| 95 | -1 | -14 | 0x00 | | |
| 96 | -1 | -14 | 0x00 | | |
| 97 | -1 | -15 | 0x80 | | |
| 98 | -1 | -15 | 0x80 | | |
| 99 | 0 | -15 | 0x00 | | |
| 100 | 0 | 0 | 0x00 | | |
| 101 | 0 | 0 | 0x00 | | |
| 102 | 0 | 0 | 0xC6 | | |
| 103 | 0 | 0 | 0x46 | | |
| 104 | 0 | 0 | 0xC5 | | |
| 105 | 0 | 0 | 0x45 | | |
| 106 | 0 | 0 | 0xC2 | | |
| 107 | 0 | 0 | 0xC4 | | |
| 108 | 0 | 0 | 0xC5 | | |
| 109 | 0 | 0 | 0x46 | | |
| 110 | 0 | 0 | 0x47 | | |
| 111 | 0 | 0 | 0x47 | | |
| 112 | 0 | 0 | 0x49 | | |
| 113 | 0 | 0 | 0xC8 | | |
| 114 | 0 | 0 | 0xC9 | | |
| 115 | 0 | 0 | 0x49 | | |
| 116 | 0 | 0 | 0x45 | | |
| 117 | 2 | 0 | 0x45 | | |
| 118 | 2 | 0 | 0xC5 | | |
| 119 | 0 | 0 | 0xC3 | | |
| 120 | 0 | 0 | 0x00 | | |
| 121 | 0 | 0 | 0x43 | | |
| 122 | 0 | 0 | 0xC4 | | |
| 123 | 0 | 0 | 0xC5 | | |
| 124 | 0 | 0 | 0x48 | | |
| 125 | 0 | 0 | 0x6C | | |
| 126 | 0 | 0 | 0xEF | | |
| 127 | 0 | 0 | 0x63 | | |
| 128 | 0 | 0 | 0xC3 | | |
| 129 | 0 | 0 | 0x43 | | |
| 130 | 0 | 0 | 0x43 | | |
| 131 | 0 | 0 | 0x44 | | |
| 132 | 0 | 0 | 0x44 | | |
| 133 | 0 | 1 | 0xC1 | | |
| 134 | 0 | 1 | 0xC7 | | |
| 135 | 0 | -12 | 0x01 | | |
| 136 | 0 | -21 | 0x00 | | |
| 137 | 1 | -26 | 0x80 | | |
| 138 | 4 | -32 | 0x80 | | |
| 139 | 6 | -36 | 0x81 | | |
| 140 | 7 | -41 | 0x82 | | |
| 141 | 2 | 17 | 0x42 | | |
| 142 | 4 | 9 | 0xC4 | | |
| 143 | 4 | 5 | 0xC9 | | |
| 144 | 4 | 4 | 0xC8 | | |
| 145 | 5 | 0 | 0x69 | | |
| 146 | 5 | 0 | 0xE9 | | |
| 147 | 5 | 0 | 0xE8 | | |
| 148 | 5 | 0 | 0x69 | | |
| 149 | 5 | 0 | 0x69 | | |
| 150 | 2 | 1 | 0x00 | 13 | differs from kid |
| 151 | 3 | 1 | 0x00 | 1 | differs from kid |
| 152 | 4 | 1 | 0x00 | 2 | differs from kid |
| 153 | 7 | 1 | 0x44 | 3 | differs from kid |
| 154 | 10 | 1 | 0x00 | 4 | differs from kid |
| 155 | 7 | 1 | 0x80 | 5 | differs from kid |
| 156 | 4 | 1 | 0x80 | 6 | differs from kid |
| 157 | 0 | 1 | 0x80 | 7 | differs from kid |
| 158 | 0 | 1 | 0xCD | 8 | differs from kid |
| 159 | 7 | 1 | 0x80 | 11 | differs from kid |
| 160 | 3 | 1 | 0x00 | 12 | differs from kid |
| 161 | 2 | 1 | 0x00 | 13 | differs from kid (kid has comment alias) |
| 162 | 2 | 1 | 0x00 | | differs from kid |
| 163 | 0 | 1 | 0x00 | 28 | differs from kid |
| 164 | 0 | 1 | 0x80 | 29 | differs from kid |
| 165 | 2 | 1 | 0xC3 | 30 | differs from kid |
| 166 | -1 | 1 | 0x48 | 9 | differs from kid (kid has comment alias) |
| 167 | 7 | 1 | 0x80 | 10 | differs from kid |
| 168 | 3 | 1 | 0x80 | 14 | differs from kid |
| 169 | 0 | 1 | 0x80 | 8 | differs from kid |
| 170 | 0 | 1 | 0xCD | 8 | differs from kid (kid has comment alias) |
| 171 | 0 | 1 | 0xCD | 8 | differs from kid (kid has comment alias) |
| 172 | 0 | 0 | 0xC6 | 47 | differs from kid (kid has no fsword) |
| 173 | 0 | 0 | 0x46 | 48 | differs from kid (kid has no fsword) |
| 174 | 0 | 0 | 0xC5 | 49 | differs from kid (kid has fsword=45) |
| 175 | 0 | 0 | 0xC5 | 49 | differs from kid (kid has comment) |
| 176 | 0 | 0 | 0xC5 | 49 | differs from kid (kid has comment) |
| 177 | 0 | 3 | 0x8A | | |
| 178 | 4 | 4 | 0x87 | | differs from kid (kid fdy=3) |
| 179 | -2 | 1 | 0x44 | | differs from kid (kid fdx=0) |
| 180 | -2 | 1 | 0x44 | | differs from kid (kid fdx=0) |
| 181 | -2 | 1 | 0x44 | | differs from kid (kid fdx=0) |
| 182 | -2 | 2 | 0x47 | | differs from kid (kid fdx=0, fdy=1) |
| 183 | -2 | 2 | 0x4A | | differs from kid (kid fdx=0, fdy=7, fcheck=0x4B) |
| 184 | — | — | — | | comment: "unused" |
| 185 | 3 | 4 | 0xC9 | | differs from kid (kid fdx=4, fdy=7, fcheck=0x49) |
| 186 | — | — | — | | comment: "mouse" |
| 187 | — | — | — | | comment: "mouse" |
| 188 | — | — | — | | comment: "mouse" |
| 189 | — | — | — | | comment: "unused" |
| 190 | — | — | — | | comment: "unused" |
| 191 | 0 | 0 | 0x00 | | |
| 192 | 0 | 0 | 0x00 | | |
| 193 | 0 | 0 | 0x80 | | |
| 194 | 0 | 0 | 0x00 | | |
| 195 | -1 | 0 | 0x00 | | |
| 196 | -1 | 0 | 0x00 | | |
| 197 | -1 | 0 | 0x00 | | |
| 198 | -4 | 0 | 0x00 | | |
| 199 | -4 | 0 | 0x80 | | |
| 200 | -4 | 0 | 0x00 | | |
| 201 | -4 | 0 | 0x00 | | |
| 202 | -4 | 0 | 0x00 | | |
| 203 | -4 | 0 | 0x00 | | |
| 204 | -5 | 0 | 0x00 | | |
| 205 | -5 | 0 | 0x00 | | |
| 206 | — | — | — | | comment: "unused" |
| 207 | 0 | 1 | 0x46 | | |
| 208 | 0 | 1 | 0xC6 | | |
| 209 | 0 | 1 | 0xC8 | | |
| 210 | 0 | 1 | 0x4A | | |
| 211 | — | — | — | | comment: "unused" |
| 212 | — | — | — | | comment: "unused" |
| 213 | — | — | — | | comment: "unused" |
| 214 | — | — | — | | comment: "unused" |
| 215 | — | — | — | | comment: "unused" |
| 216 | — | — | — | | comment: "unused" |
| 217 | 0 | 0 | 0x80 | | |
| 218 | 0 | 0 | 0x00 | | |
| 219 | 0 | 0 | 0x00 | | |
| 220 | 0 | 0 | 0x00 | | |
| 221 | 0 | 0 | 0x80 | | |
| 222 | 0 | 0 | 0x00 | | |
| 223 | 0 | 0 | 0x00 | | |
| 224 | 0 | 0 | 0x00 | | |
| 225 | 0 | 0 | 0x80 | | |
| 226 | 0 | 0 | 0x00 | | |
| 227 | 0 | 0 | 0x80 | | |
| 228 | 0 | 0 | 0x00 | | |
| 229 | 1 | 1 | 0xC3 | 35 | |
| 230 | 0 | 1 | 0x49 | 36 | |
| 231 | 0 | 1 | 0xC3 | 37 | |
| 232 | 0 | 1 | 0x49 | 38 | |
| 233 | 0 | 1 | 0xC3 | 39 | |
| 234 | 1 | 1 | 0x49 | 40 | |
| 235 | 1 | 1 | 0x43 | 41 | |
| 236 | 1 | 1 | 0xC9 | 42 | |
| 237 | 4 | 1 | 0xC6 | | |
| 238 | 3 | 1 | 0xCA | | |
| 239 | 1 | 1 | 0x43 | | |
| 240 | 1 | 1 | 0xC8 | | |

---

## princess.json

10 sequences, 48 framedefs. The princess is a cutscene-only NPC. Her sequences use no physics opcodes beyond `FRAME`, `GOTO`, `ABOUTFACE`, and `CHX`.

### Sequences

**stand**
```
FRAME(11), GOTO(stand,0)
```

**alert**
```
FRAME(2), FRAME(3), FRAME(4), FRAME(5), FRAME(6), FRAME(7), FRAME(8), FRAME(9), ABOUTFACE, CHX(8), FRAME(11),
GOTO(stand,0)
```

**stepback**
```
ABOUTFACE, CHX(11), FRAME(12), CHX(1), FRAME(13), CHX(1), FRAME(14), CHX(3), FRAME(15), CHX(1), FRAME(16),
FRAME(17), GOTO(stepback,11)
```

**lie**
```
FRAME(19), GOTO(lie,0)
```

**wait**
```
FRAME(20), GOTO(wait,0)
```

**embrace**
```
FRAME(21), CHX(1), FRAME(22), FRAME(23), FRAME(24), CHX(1), FRAME(25), CHX(-3), FRAME(26), CHX(-2),
FRAME(27), CHX(-4), FRAME(28), CHX(-3), FRAME(29), CHX(-2), FRAME(30), CHX(-3), FRAME(31), CHX(-1),
FRAME(32), FRAME(33), GOTO(embrace,21)
```

**slump**
```
FRAME(1), FRAME(18), GOTO(slump,1)
```

**stroke**
```
FRAME(37), GOTO(stroke,0)
```

**rise**
```
FRAME(37), FRAME(38), FRAME(39), FRAME(40), FRAME(41), FRAME(42), FRAME(43), FRAME(44), FRAME(45), FRAME(46),
FRAME(47), ABOUTFACE, CHX(12), FRAME(11), GOTO(rise,13)
```

**crouch**
```
FRAME(11), FRAME(11), ABOUTFACE, CHX(13), FRAME(47), FRAME(46), FRAME(45), FRAME(44), FRAME(43), FRAME(42),
FRAME(41), FRAME(40), FRAME(39), FRAME(38), FRAME(37), FRAME(36), FRAME(36), FRAME(36), FRAME(35), FRAME(35),
FRAME(35), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(35), FRAME(35),
FRAME(36), FRAME(36), FRAME(36), FRAME(35), FRAME(35), FRAME(35), FRAME(34), FRAME(34), FRAME(34), FRAME(34),
FRAME(34), FRAME(34), FRAME(34), FRAME(35), FRAME(35), FRAME(36), FRAME(36), FRAME(36), FRAME(35), FRAME(35),
FRAME(35), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34), FRAME(34),
FRAME(35), FRAME(35), FRAME(35), FRAME(36), GOTO(crouch,63)
```

### Framedefs

| idx | fdx | fdy | fcheck |
|---|---|---|---|
| 0 | 0 | 0 | 0x00 |
| 1 | 0 | 0 | 0x00 |
| 2 | 0 | 0 | 0x80 |
| 3 | 0 | 0 | 0x80 |
| 4 | 0 | 0 | 0x80 |
| 5 | -1 | 0 | 0x00 |
| 6 | 2 | 0 | 0x80 |
| 7 | 2 | 0 | 0x00 |
| 8 | 0 | 0 | 0x80 |
| 9 | 1 | 0 | 0x80 |
| 10 | 2 | 0 | 0x80 |
| 11 | 0 | 0 | 0x80 |
| 12 | 0 | 0 | 0x80 |
| 13 | 0 | 0 | 0x00 |
| 14 | 0 | 0 | 0x80 |
| 15 | 0 | 0 | 0x80 |
| 16 | 0 | 0 | 0x80 |
| 17 | 0 | 0 | 0x00 |
| 18 | 0 | 0 | 0x00 |
| 19 | 0 | 0 | 0x00 |
| 20 | 0 | 0 | 0x00 |
| 21 | 0 | 0 | 0x00 |
| 22 | 0 | 0 | 0x80 |
| 23 | 0 | 0 | 0x00 |
| 24 | 0 | 0 | 0x80 |
| 25 | 0 | 0 | 0x80 |
| 26 | 0 | 0 | 0x00 |
| 27 | 0 | 0 | 0x00 |
| 28 | 0 | 0 | 0x00 |
| 29 | 0 | 0 | 0x00 |
| 30 | 0 | 0 | 0x00 |
| 31 | 0 | 0 | 0x00 |
| 32 | 0 | 0 | 0x00 |
| 33 | 0 | 0 | 0x00 |
| 34 | 0 | 0 | 0x00 |
| 35 | 0 | 0 | 0x00 |
| 36 | 0 | 0 | 0x00 |
| 37 | 0 | 0 | 0x00 |
| 38 | 0 | 0 | 0x80 |
| 39 | 0 | 0 | 0x80 |
| 40 | 1 | 0 | 0x00 |
| 41 | -1 | 0 | 0x00 |
| 42 | 2 | 0 | 0x00 |
| 43 | 1 | 0 | 0x80 |
| 44 | 0 | 0 | 0x80 |
| 45 | 0 | 0 | 0x80 |
| 46 | 0 | 0 | 0x80 |
| 47 | -1 | 0 | 0x00 |

---

## vizier.json

5 sequences, 39 framedefs. Jaffar/vizier — cutscene character for end-of-game sequence.

### Sequences

**stand**
```
FRAME(7), GOTO(stand,0)
```

**stop**
```
FRAME(8), FRAME(9), GOTO(stand,0)
```

**walk**
```
CHX(1), FRAME(1), CHX(2), FRAME(2), CHX(6), FRAME(3), CHX(1), FRAME(4), CHX(-1), FRAME(5), CHX(1), FRAME(6),
CHX(1), GOTO(walk,1)
```

**raise**
```
FRAME(38), FRAME(20), FRAME(20), FRAME(20), FRAME(20), FRAME(20), FRAME(20), FRAME(21), FRAME(22), FRAME(23),
FRAME(24), FRAME(25), FRAME(26), FRAME(27), FRAME(28), FRAME(36), FRAME(37), FRAME(29), GOTO(raise,17)
```

**exit**
```
FRAME(30), FRAME(31), FRAME(32), FRAME(33), FRAME(34), FRAME(35), CHX(1), FRAME(7), FRAME(7), FRAME(7),
FRAME(7), FRAME(7), FRAME(7), FRAME(10), FRAME(11), FRAME(12), FRAME(13), FRAME(14), CHX(2), FRAME(15),
CHX(-1), FRAME(16), CHX(-3), FRAME(17), FRAME(18), CHX(-1), FRAME(19), ABOUTFACE, CHX(16), CHX(3),
GOTO(walk,3)
```

### Framedefs

| idx | fdx | fdy | fcheck |
|---|---|---|---|
| 0 | 0 | 0 | 0x00 |
| 1 | 0 | 0 | 0x80 |
| 2 | 0 | 0 | 0x80 |
| 3 | 0 | 0 | 0x80 |
| 4 | 0 | 0 | 0x00 |
| 5 | 0 | 0 | 0x00 |
| 6 | 0 | 0 | 0x80 |
| 7 | 0 | 0 | 0x80 |
| 8 | 0 | 0 | 0x80 |
| 9 | 0 | 0 | 0x80 |
| 10 | 0 | 0 | 0x80 |
| 11 | 0 | 0 | 0x80 |
| 12 | 0 | 0 | 0x80 |
| 13 | 0 | 0 | 0x80 |
| 14 | 0 | 0 | 0x00 |
| 15 | 0 | 0 | 0x80 |
| 16 | 0 | 0 | 0x00 |
| 17 | 0 | 0 | 0x00 |
| 18 | 0 | 0 | 0x80 |
| 19 | 0 | 0 | 0x00 |
| 20 | 3 | 0 | 0x00 |
| 21 | 3 | 0 | 0x00 |
| 22 | 3 | 0 | 0x00 |
| 23 | 2 | 0 | 0x00 |
| 24 | 3 | 0 | 0x80 |
| 25 | 5 | 0 | 0x00 |
| 26 | 5 | 0 | 0x00 |
| 27 | 1 | 0 | 0x80 |
| 28 | 2 | 0 | 0x80 |
| 29 | 2 | 0 | 0x80 |
| 30 | 1 | 0 | 0x80 |
| 31 | 1 | 0 | 0x00 |
| 32 | 2 | 0 | 0x00 |
| 33 | 3 | 0 | 0x00 |
| 34 | 3 | 0 | 0x00 |
| 35 | 0 | 0 | 0x80 |
| 36 | 2 | 0 | 0x80 |
| 37 | 2 | 0 | 0x80 |
| 38 | 1 | 0 | 0x00 |

---

## mouse.json

5 sequences, 4 framedefs. The mouse NPC that appears at the start of level 1.

### Sequences

**scurry**
```
ACT(1), FRAME(1), CHX(5), FRAME(1), CHX(3), FRAME(2), CHX(4), GOTO(scurry,1)
```

**leave**
```
ACT(0), FRAME(1), FRAME(1), FRAME(1), FRAME(3), FRAME(3), FRAME(3), FRAME(3), FRAME(3), FRAME(3), FRAME(3),
FRAME(3), ABOUTFACE, CHX(8), GOTO(scurry,1)
```

**stop**
```
FRAME(1), GOTO(stop,0)
```

**raise**
```
FRAME(3), GOTO(raise,0)
```

**climb**
```
FRAME(1), GOTO(climb,0)
```

### Framedefs

| idx | fdx | fdy | fcheck |
|---|---|---|---|
| 0 | 0 | 0 | 0x00 |
| 1 | 0 | 0 | 0x44 |
| 2 | 0 | 0 | 0x44 |
| 3 | 0 | 2 | 0x44 |

---

## sword.json

Different format — no `sequence` or `framedef` keys. Contains a single key `swordtab`: an array of 50 entries, each `{id, dx, dy}`.

This is the sword sprite offset table. During rendering, the current frame's `fsword` field (from the active character's framedef) indexes into this table to look up which sword sprite (`id`) to draw and at what pixel offset (`dx`, `dy`) relative to the character's display position.

### swordtab

| index | id | dx | dy |
|---|---|---|---|
| 0 | 1 | 0 | -9 |
| 1 | 6 | -9 | -29 |
| 2 | 2 | 7 | -25 |
| 3 | 3 | 17 | -26 |
| 4 | 7 | 7 | -14 |
| 5 | 8 | 0 | -5 |
| 6 | 4 | 17 | -16 |
| 7 | 5 | 16 | -19 |
| 8 | 31 | 12 | -9 |
| 9 | 9 | 13 | -34 |
| 10 | 10 | 7 | -25 |
| 11 | 11 | 10 | -16 |
| 12 | 12 | 10 | -11 |
| 13 | 13 | 22 | -21 |
| 14 | 14 | 28 | -23 |
| 15 | 15 | 13 | -35 |
| 16 | 16 | 0 | -38 |
| 17 | 17 | 0 | -29 |
| 18 | 18 | 21 | -19 |
| 19 | 19 | 14 | -23 |
| 20 | 20 | 21 | -22 |
| 21 | 20 | 22 | -23 |
| 22 | 18 | 7 | -13 |
| 23 | 18 | 15 | -18 |
| 24 | 8 | 0 | -8 |
| 25 | 2 | 7 | -27 |
| 26 | 29 | 14 | -28 |
| 27 | 9 | 7 | -27 |
| 28 | 5 | 6 | -23 |
| 29 | 5 | 9 | -21 |
| 30 | 11 | 11 | -18 |
| 31 | 14 | 24 | -23 |
| 32 | 14 | 19 | -23 |
| 33 | 14 | 21 | -23 |
| 34 | 21 | 7 | -32 |
| 35 | 22 | 14 | -32 |
| 36 | 23 | 14 | -31 |
| 37 | 24 | 14 | -29 |
| 38 | 25 | 28 | -28 |
| 39 | 26 | 28 | -28 |
| 40 | 27 | 21 | -25 |
| 41 | 28 | 14 | -22 |
| 42 | 0 | 14 | -25 |
| 43 | 0 | 21 | -25 |
| 44 | 30 | 0 | -16 |
| 45 | 9 | 8 | -37 |
| 46 | 32 | 14 | -24 |
| 47 | 33 | 14 | -24 |
| 48 | 34 | 7 | -14 |
| 49 | 9 | 8 | -37 |
