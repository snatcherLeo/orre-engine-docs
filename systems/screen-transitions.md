# Screen transitions (fades and battle-entry wipes)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-14 night). Code: `src/game/effect/fade.c` (fade core), `src/game/effect/fade_effect.c`
(per-style init hooks), `src/game/effect/fade_range_801C4CB8.c` (per-style frame functions, the "fluid" ripple
simulation), `src/game/fight.c` (battle-entry wipe table). Data: `fight_encount_wipe_data` (0x803759C4 area,
`src/game/data/data_80375938.c`).

## What it does

Every full-screen transition goes through one small fade system: plain fades to and from black, and the stylised
wipes played when a battle starts (horizontal / vertical bars, a Poké Ball, the "boss" ripple, ...). A transition is
a **style init hook** that installs a per-frame function with `fadeSetFunctionOnly`; `fadeDaemon` then calls that
function every frame with the transition's progress until it reports that it is finished.

## Data structures

**Battle-entry wipe table** `fight_encount_wipe_data` (12-byte records, count in `lbl_80478D20`, read by
`fightEncountWipeDataBiosGetPtr(index)`; field names from the game's code, verified):

| Field | Type | Meaning |
|---|---|---|
| `snapshotUse` | u8 | a switch |
| `wipeEffectSndId` | u16 | sound effect played with the wipe (0x48A for most, 0x467 for the Poké Ball wipe) |
| `wipeEffectTime` | f32 | length (0.5 for plain fades, 2.5 for stylised wipes, 0.1 for the trainer one) |
| `wipeFunction` | pointer | the style's init hook |

The 11 entries in order: fade-out-in, `yoko` (horizontal), `tate` (vertical), `ball` (Poké Ball), `boss`, `carde`, `yoko_or_tate`, `yoko_or_tate_or_ball`, fade-out-in,
`trainer`, fade-in.

**which battles use which entry** (verified from the disc). Each battle's encounter record holds
its wipe index (the 32-bit word at offset 0x10, read by `fightEncountDataBiosGetWipeId`). Across the 566 battle
records the random horizontal-or-vertical wipe is by far the most common, followed by the trainer swirl (entry 9),
the Poké Ball wipe and the short fade-in (entry 10); the boss wipe and the fade-out-in are used by a few dozen battles
each, and the remaining styles by a handful. The first story battle (Willie at Outskirt Stand) uses entry 9, the
trainer swirl.

The executable's table points entry 9 at the init that installs `fn_801C4CB8` (`fadeEffectHookFunction_trainer_Init`)
and entry 10 at the one that installs `fn_801C54FC` (`fadeEffectHookFunction_fadein_Init`) (verified from the
executable's data, and in Dolphin: Willie's battle starts with the trainer swirl).

**What the trainer swirl looks like** (verified in Dolphin, Willie's battle): the field freezes and warps more and
more (a horizontal wave and a radial smear) while the battle loads, then a large ring overlay cross-fades into the
battle field.

**The fade-in style** (entry 10, `fadeEffectHookFunction_fadein_Init` → frame function `fn_801C54FC`, verified from
the code) draws no snapshot. While the fade is still pending it draws an opaque black full-screen rectangle; once the
fade runs, the rectangle's alpha falls with the fade's progress while the fade itself draws its own colour rectangle.
Its wipe time is 0.1 s, so it is a brief blink, not a long black screen.

**Camera work for the "trainer" style** (`FadeCameraWork` at `lbl_80467030`): frame counter, value / step / target
of an easing value, two texture handles (`0x0F861200`, `0x0F871200` resources). The code that sets it up
(`fn_801C53BC`) runs from the frame function `fn_801C4CB8`, which the trainer init installs (the boss init installs
`fn_801C55D8`). The trainer style
also captures every frame into its texture itself and draws it back rotated and zoomed as three offset copies, then
blends it with the two textures (verified from the code).

## Flow

1. Battle start: the battle code picks a wipe index, reads the record, plays `wipeEffectSndId` and calls
   `wipeFunction`, which installs the style's frame function (`fadeSetFunctionOnly`).
2. Every frame `fadeDaemon` calls the frame function with (finish flag, texture, frame, duration, angle,
   angleDuration).
   (**verified** from the code): **when and how the fade advances.** `fadeDaemon` runs inside
   `mainRetraceLogic`, one of the game's per-frame tasks (the task list `GSgappUpdate` walks once per pass of the main
   loop, in priority order: 0x00, 0x01, 0x0A the main logic, 0xFD, 0xFE `mainRetraceLogic`, 0xFF the frame's draw). Each
   pass adds `frameRetraces / fps` to the fade's elapsed time, where `frameRetraces` is how many screen refreshes the
   previous frame spent in the frame-pacing wait (`GSgetFrameStep`). So a fade is measured in refreshes the game
   *waited*, not in wall time: a frame whose extra time passes outside that wait, or a pass in which the task doesn't
   run, stretches the fade on screen without changing its values (**likely**; seen once on the console, a fade-in that
   held one value for an extra refresh, cause being traced). `fadeSet` restarts the elapsed time at 0.
3. "Boss" style (`fn_801C55D8`, verified 2026-10-14): on its first frame it sets up the ripple simulation
   (`fn_801C5748`: 40 x 30 grid, five "shock" impulses), every frame advances an eased value and the fluid
   (`fn_801C6688`); **only on the frame the transition finishes** (`finish == 1`) it evaluates the fluid and draws the
   distorted screen (`fadeFluidEvaluate`, `fn_801C6934`).
   (**verified** from the code): **how the ripple works.** It is the classic height-field wave
   simulation. The grid holds (columns + 1) x (rows + 1) points, each a 3D vector whose z is the water height, in two
   height pages that swap every step. Each step computes a point's new height from its height now, its height one
   step ago, and the sum of its four neighbours' heights, with three weights (`fadeFluidCalcParms`: wave speed,
   damping and grid spacing turned into "keep", "damp" and "spread" factors). The edge points never move. Then each
   inner point gets two slope vectors from its neighbours' height differences, used as the vertex normal when the
   distorted screen is drawn. A "shock" (`fadeFluidSetShock`) pushes one point down, and its eight neighbours by half as much. Before
   using its wave limit, the setup checks it against the largest value that stays stable for the grid spacing and time
   step (`(cellSize / (2 x step)) x sqrt(2 + damping)`). If the limit is out of range, it prints a warning and uses
   just under the maximum.
   **For a 64-bit port** (**verified**, a crash in the PC port (the decompiled code running natively)): the neighbour above is read at index `x - (columns + 1)`
   computed in unsigned 32-bit arithmetic. On the GameCube the wrapped value plus the 32-bit pointer lands back on the
   row above; with 64-bit pointers the read lands about 48 GB past the grid. The index has to be computed signed.
4. **How a fade is timed** (**verified** from the code and in Dolphin). `fadeSet` resets the
   elapsed time to 0. Each call of `fadeDaemon` adds the last frame's length in retraces (`GSgetFrameStep`, the
   graphics state's frame-retrace count) divided by the frame rate. `fadeDaemon` runs once per game-logic pass, from
   the menu daemon (`menuDaemon`, called by `mainRetraceLogic`). So a 0.5 s fade takes 30 steps at 60 fps, 15 at
   30 fps, and a fade "pauses" in any refresh where no logic pass runs. On the console, the fade-in into some rooms
   (seen so far: rooms 0x3A1, 0x390, 0x38E, 0x391, 0x39D and 0x38F) holds its first step or two this way: the frame step stays
   1 and drawing stays on schedule, but the logic pass skips a refresh and later runs twice (**likely** caused by the
   room's setup work delaying the main thread).

**Boot sequence:** the rooms requested from power-on to the first playable room are listed in new-game-flow.md
(**verified** in Dolphin with a memory watch).

## Disc data

Two textures for the trainer wipe by resource id (0x0F861200, 0x0F871200); the wipe table and timings are in the
executable.

## Changing it

- New transition styles: add an init hook + frame function and a table record; the table's count is a separate
  global (`lbl_80478D20`).
- For a PC port: transitions are timed in frames by `fadeDaemon`; frame-rate independence needs the progress
  computed from time.

### Mod surface
- **Data it reads:** the wipe table (duration, style, screen snapshot, sound), each encounter's wipe index, the wipe
  textures.
- **Hooks it would need:** a wipe chosen per battle (custom transitions), new wipe styles, and a "smooth" option where
  loads allow.
- **Fixed-size limits:** the wipe table's entries.
- **Must stay vanilla in GameCube-accurate mode:** every transition's style and its timing in frames.

## Efficiency

The ripple uses a coarse 40 x 30 grid simulated on the CPU and drawn as a distorted textured mesh, cheap enough to
run during the battle load.

## Open questions

- `snapshotUse` meaning; whether `wipeEffectTime` is seconds or a frame scale.
- The `Doku` (poison) and `UDLR` styles outside the battle table: where they are used.
