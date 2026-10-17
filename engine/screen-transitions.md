# Screen transitions (fades and battle-entry wipes)

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
| `snapshotUse` | u8 | 1 = the wipe works on a snapshot of the last frame (guess from the name) |
| `wipeEffectSndId` | u16 | sound effect played with the wipe (0x48A for most, 0x467 for the Poké Ball wipe) |
| `wipeEffectTime` | f32 | length in seconds (0.5 for plain fades, 2.5 for stylised wipes, 0.1 for the trainer one; guess: seconds) |
| `wipeFunction` | pointer | the style's init hook |

The 11 entries in order: fade-out-in, `yoko` (horizontal), `tate` (vertical), `ball` (Poké Ball), `boss`, `carde`
(e-Reader card battles, guess), `yoko_or_tate`, `yoko_or_tate_or_ball` (random choice, guess), fade-out-in, fade-in,
`trainer`. Which battles use which entry: to trace from the battle records (open).

**Camera work for the "boss" style** (`FadeCameraWork` at `lbl_80467030`): frame counter, value / step / target of
an easing value, two texture handles (`0x0F861200`, `0x0F871200` resources).

## Flow

1. Battle start: the battle code picks a wipe index, reads the record, plays `wipeEffectSndId` and calls
   `wipeFunction`, which installs the style's frame function (`fadeSetFunctionOnly`).
2. Every frame `fadeDaemon` calls the frame function with (finish flag, texture, frame, duration, angle,
   angleDuration).
3. "Boss" style (`fn_801C55D8`, verified 2026-10-14): on its first frame it sets up the ripple simulation
   (`fn_801C5748`: 40 x 30 grid, five "shock" impulses), every frame advances an eased value and the fluid
   (`fn_801C6688`); **only on the frame the transition finishes** (`finish == 1`) it evaluates the fluid and draws the
   distorted screen (`fadeFluidEvaluate`, `fn_801C6934`).

**Boot sequence:** the rooms requested from power-on to the first playable room are listed in new-game-flow.md
(**verified** in Dolphin with a memory watch).

## Disc data

Two textures for the boss wipe by resource id (0x0F861200, 0x0F871200); the wipe table and timings are in the
executable.

## Changing it

- New transition styles: add an init hook + frame function and a table record; the table's count is a separate
  global (`lbl_80478D20`).
- For a PC port: transitions are timed in frames by `fadeDaemon`; frame-rate independence needs the progress
  computed from time.

## Efficiency

The ripple uses a coarse 40 x 30 grid simulated on the CPU and drawn as a distorted textured mesh, cheap enough to
run during the battle load.

## Open questions

- Which battles use which wipe entry (battle records / trainer classes?).
- `snapshotUse` meaning; whether `wipeEffectTime` is seconds or a frame scale.
- The `Doku` (poison) and `UDLR` styles outside the battle table: where they are used.
