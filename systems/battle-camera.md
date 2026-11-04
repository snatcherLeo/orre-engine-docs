# Battle camera: who controls it, when, and how it uses random numbers

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the code is the same engine layer, and several functions match XD's by name; not checked against XD)

Status: **partial** (2026-10-18). Read from the code and the disc data; the draw counts per shot are worked out from
the code and are **likely** until measured in a run. Code: `src/game/battle/battle_camera.c` (the idle camera),
`src/game/wazaSequenceCamera.c` (every camera shot, idle or move), `src/game/wazaSequence.c` (when a sequence takes the
camera), `src/game/battle/battle_main.c` (switching the idle camera on and off). Related: [battle-arenas.md](battle-arenas.md)
(the battle grid; its section "How a move camera picks its shot" covers the same shot lists from the arena side),
[wzx.md](../engine/wzx.md) (the move-effect files), [rng.md](../engine/rng.md) (the shared generator).

## What it does

Every camera shot in a battle is started by one function, `battleCameraStartWaza(owner, sequence)`, for a participant
on the battle grid (a trainer or a Pokémon). It does one of two things:
- **plays an authored camera animation** stored in the playing effect file (a fixed shot, no random numbers), or
- **builds a procedural shot** around the participant: it picks a motion (dolly, slide, orbit, ...), then a distance,
  height, angle and field of view inside ranges taken from the participant's size, **drawing from the game's shared
  random-number generator** for each choice.

Between actions, while the player chooses commands, an **idle camera** picks a random participant every few seconds and
starts a procedural shot on it. Because both the idle camera and the procedural shots use the shared generator, the
camera changes **which random numbers later battle events get** (damage rolls, critical hits, AI choices), so a
faithful port must reproduce every camera draw exactly.

## Data structures

| What | Where | Meaning |
|---|---|---|
| Idle timer | `lbl_80478CA8` (s32) | counts down from **200**; reset to 200 after each pick or whenever the idle camera is paused |
| Last idle target | `lbl_80478CAC` | the grid member picked last; never picked twice in a row |
| Idle camera off | `lbl_8047B398` (u8) | set by `battleCameraDisable` when a sequence with a camera starts |
| "Simple" mode | `lbl_8047B399` (u8) | 1 = idle camera paused (`battleCameraDoSimple`), 0 = running (`battleCameraDoFull`) |
| Active shot | `lbl_8047B3EC` (owner), `lbl_8047B3F0` (sequence or none) | the participant the camera is on, and the effect sequence driving it (none = an idle shot) |
| Shot parameters | `lbl_804673A0` (0x34 bytes, `WazaSequenceCameraParams`) | motion, size and height of the owner, rotation / height / distance ranges, near / middle / far lengths |
| Field-of-view keys | `lbl_804673D4` (two `CameraFovKey`: start and end value, start and end frame) | the zoom over the shot |
| Last motion | `lbl_80478CD8` | the procedural motion used last; the next shot avoids repeating it |
| Shot patterns | `lbl_80371F60` (8 entries) and `lbl_803721C0` (13 entries), each with a duration weight | timing patterns for the procedural shots, chosen by weighted random pick. Each record is an f32 duration plus two `{s32 mode, s32 duration mode, f32 thresholds[4], u32 initial flags, u32 flags, s32 timing mode}` blocks (**verified** from code and data); the 8-entry table serves move sequences other than kinds 8 / 9 |
| Per-sequence switches | the effect sequence's flags (+0x08) and its camera byte (+0x16) and target part (+0x17) | which motions are allowed, which side to shoot from, and whether this sequence takes the camera at all |

## Flow, phase by phase

The battle's phases are in [battle-turn-flow.md](battle-turn-flow.md) (section 0). The camera in each:

| Phase | What plays | Camera | Random draws |
|---|---|---|---|
| **Venue and trainers' entry** | the arena intro (`colo_intro_*` in Colosseums) and the trainers' entrance (`trainer_enter_a`-`e`, boss intros `intro_*`) | **authored** (each of these files carries its own camera, verified below) | none from the camera |
| **Send-out** | the ball sequences: `<ball>_out` (throw), `<ball>_open` | `_out` has **no** authored camera, `_open` **has one** | `_out`: a procedural shot if the sequence takes the camera (**likely**; draws as below) |
| **Command selection** | nothing playing; the idle camera runs | **procedural, on a random participant** every ~200 screen refreshes | yes (below); the models' idle **blink checks** also draw, 3-5 per game frame at Willie's first menu (rng.md, "Idle blinking draws every frame in battle") |
| **Moves** | the move's effect files, first half on the user, second half on the target | **authored** for a few moves (Surf, Waterfall, Earthquake, Fissure, Magnitude, Spikes, Whirlpool, the four weather moves and a few more: 133 files in all); **procedural** for every other move | procedural shots draw (below) |
| **Damage half** | the move's second file, owned by the target | the camera cuts to the target (the sequence's target part, byte +0x17) | as for moves |
| **Faint, return, switch** | `<ball>_in` (return: **authored**), `<ball>_downin` and the next `_out` | authored for the return; procedural otherwise (**likely**) | procedural parts draw |
| **Shadow Pokémon moments** | `dark_pokemon_look` (Rui senses a Shadow Pokémon) | **authored** | none |
| **Hyper Mode, Shadow moves** | their effect files | procedural unless the file carries a camera (none of the Hyper Mode files do, **likely**) | procedural shots draw |
| **Snag** | `snatch_attack_<ball>` (the Snag Ball throw), `snatch_ball_land_*`, `snatch_shake_*`, `snatch_miss_*`, `snatch_snatch` (success) | **all authored** | none from the camera (the shake rolls draw separately: [battle-snag.md](battle-snag.md)) |
| **Battle end** | the win / lose poses (trainer slot 5 and 10 sequences) | procedural unless the file has a camera (**likely**) | procedural shots draw |

**When the idle camera runs** (`battle_main.c`, **verified**): just before each command selection the battle switches
the camera to "full" (`battleCameraSelect(0)` (0x801EF8F4), which also reloads the room's battle-grid data, `battleCameraStartOffsetAnime` (0x801C2F00)), and right after
the selection it switches back to "simple" (`battleCameraDoSimple`). The entry, the action phases and the end run in
"simple" mode. So **the idle camera only draws while the player is choosing commands.**

**When a sequence takes the camera** (`wazaSequence.c`, **verified**): every effect sequence is loaded with its camera
switch on (+0x16 = 1); when it starts playing and the switch is still on, it calls `battleCameraStartWaza(owner,
sequence)`, which also turns the idle camera off. Some battle code turns the switch off for particular sequences
before they play (`wazaSequenceSys.c` and a battle-floor setting, floor status 0x22): which moments is not traced yet.

## How a move's camera is placed (in plain words)

(**verified**: read from the byte-matched camera code, and **14 of the first 14 shots of Willie's
first battle identical frame by frame** between a GameCube emulator and a native build of the same battle).

1. **The move asks for a camera.** If its effect file carries its own camera (a minority, listed in the next
   section), that authored camera plays. Otherwise the game builds a shot.
2. **It measures the Pokémon being filmed.** The model's bounding box gives a size and a height, so a small Pokémon
   and a large one get shots scaled to them.
3. **It picks a kind of shot at random** (a "motion": orbit, slide, push, dolly, or two fixed views). The move's effect
   file can limit which kinds are allowed, and the game avoids repeating the previous shot's kind.
4. **It turns the kind and the size into ranges**: how far around the Pokémon to swing, how high and how far away.
5. **It picks random values inside those ranges** and places the camera relative to the attacker's and target's
   positions and the way they face. An orbit, for example, swings around the target from one random angle to another
   over the shot's duration; a slide moves sideways past it; a push moves in a straight line.
6. **It picks the zoom** (field of view) at random inside a range that comes from the shot's near, middle and far
   distances.
7. **The far side is mirrored.** When the filmed Pokémon stands on the far side of the field, the angles are mirrored,
   so the shot looks the same from the other side.

Because every pick stays inside ranges fitted to the filmed Pokémon, a correct pick always gives a normal-looking shot,
and any move frames any Pokémon sensibly (a move a Pokémon can't normally learn still looks natural). A camera inside a
Pokémon or behind its tail can only come from a wrong position, facing or size. The random picks draw from the battle's shared generator ("When the shared generator
is drawn from" below).

## Authored camera or procedural shot

**Authored** (`battleCameraStartWaza`, sequence given and its camera resource present): the effect file's **own
resource archive** holds an HSD scene with a camera; the code plays it as an animation offset to the owner (position,
facing, and the owner's size when the file's flags ask for it), mirrored when the owner faces the other way. No random
numbers. **Verified from the disc:** 133 of the 1,098 effect files have an own archive, and **every one of the 133
contains a camera** (the Colosseum intros, boss and trainer entrances, the ball returns and openings, the whole snag
set, the four weather moves, Surf, Waterfall, Earthquake, Fissure, Magnitude, Spikes, Whirlpool, a few others, Rui's
looks, evolution, purification and the present scenes). The other 965 effect files have no camera of their own.

**Procedural** (every other shot, and every idle shot): `battleCameraStartWaza` works in four steps:
1. **Parameters** (`_wazaSequenceCameraCalculateParams`): ranges for rotation, height and distance from the owner's
   size and whether it is a trainer or a Pokémon; no draws.
2. **Motion** (`_wazaSequenceCameraSelectMotion`): one of five motions. The effect file's flags (+0x08) allow some:
   bit 0 forces motion 5; bits 3-6 allow motions 3, 0, 1, 2. With no bits (and for every idle shot) all four are
   allowed. With exactly one allowed it is used without a draw; otherwise **one draw per try, retried until the motion
   differs from the previous shot's**. One species (internal id 0x13A) always gets motion 4.
   The retry is narrower than that. The draw is laid against the allowed motions in the order
   3, 0, 1, 2, each taking an equal share; when the motion it lands on is the previous shot's, the walk goes on to the
   next allowed motion **with the same number**, and only a number in the last share (whose motion is the previous
   one) costs another draw. So the same draw gives a different motion depending on the previous shot (**verified**
   from the code and a trace: 0.551 with four allowed is motion 1, or motion 2 when the previous shot was motion 1).
   **The previous shot can be an idle shot** (`lbl_80478CD8` is shared): an idle shot during command selection changes
   which motion the move's first camera takes, and with it how many numbers that camera draws (motion 1: 3 for the
   position, motion 2: 4), so every later roll of the turn (**verified** on a Pyrite Colosseum cup battle).
3. **Position** (`_wazaSequenceCameraDoPosition`): the motion places and moves the camera:

   | Motion | What it does | Draws |
   |---|---|---|
   | 0 dolly | a move toward or away from the owner, with a random hold and timing | **about 7-11** (two chances, up to two random integers of 2 draws each, two distances, a height, two angles) |
   | 1 slide | a sideways move past the owner (offset 25-35, height 1-10, distance 20-50) | **3** |
   | 2 orbit | a turn around the target from one random angle to another (the smaller first) over the shot's duration, at a random distance and height | **4** |
   | 3 push | a straight move at a random angle, distance and height | **3** |
   | 4 fixed | a fixed wide shot (distance 110) | **0** |
   | 5 close | a fixed-distance close-up at a random height | **1** |

   **Mirroring:** when the owner's side value (+0x76) is negative (the far side) and the effect file doesn't set flag
   0x80, the angles are mirrored (`reverse`), so a shot from the far side is the same shot turned around.
4. **Field of view** (`_wazaSequenceCameraDoFOV`, with `wazaSequenceCameraGetPattern` and
   `_wazaSequenceCameraSelectDuration`): a weighted pick of a timing pattern (one draw, plus two if the pick falls past
   the table) and one or two draws for the zoom range and duration (**likely** 2-4 in total).

**What the parameters come from** (**verified** by logging the computed parameters every frame on
a GameCube emulator and a native build of the same battle): the owner model's **bounding box** gives the shot's scale:
`size` is 0.577 × the length of the box's extent vector (the box diagonal over √3), `height` is the extent's Y, and
the height range starts from the box's minimum and maximum Y (clamped to the motion's limits). The rotation range
comes from the effect file's flags and whether the owner is alone or has a partner; the distance range from the flags
and a per-size scale. So a model with a wrong bounding box gets every procedural shot framed at the wrong height and
distance, while authored and fixed shots are unaffected. In Willie's first battle the first procedural shot has size
14.17, height 20.82 and a height range up to 17.32.

**An example sequence** (**verified**, frames captured in Dolphin from the first battle against Willie): Umbreon's
Bite opens on a **wide shot from behind and to the left of Umbreon**, held while it runs in, then **cuts to a close-up
of the target** with Willie in the background. Bite has no camera of its own, so both shots are procedural; which
motion each one drew is not traced.

## When the shared generator is drawn from

All draws go to the one shared generator (`HSD_Randf` for a fraction, two calls of `HSD_Rand` for a random integer;
[rng.md](../engine/rng.md)). **Verified** from the code unless marked:

**The idle camera, every game frame during command selection** (`battleCameraStartRandom`):
1. If the idle camera is off or "simple", or a shot is still playing: timer back to 200, **no draw**.
2. Otherwise the timer goes down by **the number of screen refreshes this game frame took** (the function the code
   calls, `GSgetFrameStep` at 0x800D3088, earlier named `msgGetCharsPerFrame`, returns the video layer's
   retraces-per-frame count, verified in `gs_gfx.c`). So the timer counts **screen refreshes, not game frames**: 200 refreshes is about 3.3 seconds, whatever
   the scene's frame rate.
3. When it reaches 0: **one draw**; if the fraction is above 0.5 (`lbl_8047DF58`), the pick is skipped and the timer
   reset. So about half of the expiries cost exactly **1 draw** and change nothing on screen.
4. Otherwise a random participant: **2 draws per try**, retried while it equals the last pick. With n participants the
   expected number of tries is n/(n-1) (n = 4 in a single battle, 6 in a double with two trainers: **likely** the count
   used, it is the grid's trainers plus Pokémon).
5. Then a procedural shot on that participant: **motion + position + field of view** as above (about 4-17 draws,
   depending on the motion).

**Likely total per idle shot:** 1 (the 50% roll) + about 2.4-2.7 (the target, on average) + about 1.3 (the motion, on
average) + 0-11 (the position) + 2-4 (the zoom) ≈ **7-20 draws**, one shot about every 3.3 seconds of waiting, plus
1 draw for each skipped roll. *To measure:* a per-caller trace of the generator at the first command menu of Willie's
battle, attributing each draw to these functions.

**A move with no authored camera:** the procedural shot's motion, position and zoom draws (about 4-17), once per
effect sequence that takes the camera (**likely** both halves of a damaging move).

**Never from the camera:** authored shots (the 133 files above), the snag set, and the shake rolls (those are the
throw's own draws).

### What this means for exact replays (for a port and for RNG tools)
- **The idle timer counts refreshes, so frame pacing matters:** the count it subtracts is the video layer's
  retraces-per-frame value. Whether a slow frame raises that value depends on the frame-pacing mode: with the
  wait-for-retrace mode off, the game reports exactly the scene's frame length and a slow frame just runs slower
  ([gfx.md](../engine/gfx.md), "How the game paces itself"); with it on, a frame that took two refreshes subtracts two, so the
  idle camera picks sooner and every later draw moves (**likely** relevant to a stall such as the short wait at the
  first command menu). Which mode battles use is not traced yet. **For a port:** the count
  the game sees stays at the console's 60 per second whatever the monitor's rate, interpolated frames never advance
  it, and battle speed-ups scale it with the rest of the battle; the check is the same replay giving identical random
  numbers at 30, 60, 120 and 144 fps.
- **How long the player waits at the command menu changes the RNG:** each 3.3 seconds of waiting costs one or more
  idle picks. RNG manipulators use this; a replay must press the same buttons on the same frames.
- **Not every random-fraction draw is a particle:** the battle trace's largest caller, the random-fraction wrapper
  `GSrandf` (0x800E0BE4), is used by the camera code as well as by particles, so part of its count in Willie's battle (rng.md)
  is camera draws. A trace that records the wrapper's own caller settles the split.

### Checked against the console 
- **Verified:** with the same random numbers, the procedural shots come out the same number for number: in two
  recorded story battles (Rider Willie, Folly), 31 of the 32 shots compared match the console at every frame (eye,
  target and field of view), covering motions 0-3; the 32nd follows the identical path one frame earlier (a timing
  difference in that move's sequence, not the camera's math). The framing (size and height) and the ranges per motion
  were the same in every shot of three recorded play sessions, also where later random picks differed.
- **What decides "right":** a shot is wrong only if it breaks a rule the code fixes: the framing for that participant,
  the ranges for that motion and box, or the path of that motion. Which motion and which distances are picked is
  random, so many different shots are all correct; a different shot in a replay means the shared generator was drawn a
  different number of times before the camera's turn (particles, blinks and the camera share it).
- **Covering every rule:** motions 0-3 are random picks; motion 5 needs a move whose effect flags set bit 0, motion 4
  needs species 0x13A, and a move that allows a single motion always uses it. A complete check therefore needs battles
  chosen by move and species (each motion, small and large participants, both sides, single and double battles,
  self- and ally-targeted moves), not more battles at random.

## Disc data

- **Authored cameras:** inside the 133 effect files with an own archive (an HSD scene with a camera), [wzx.md](../engine/wzx.md).
- **Procedural shot ranges:** code and two pattern tables in the executable (`lbl_80371F60`, `lbl_803721C0`), not on
  the disc as separate files.
- **Per-room battle grid data:** loaded by room (`fn_80113FB4`, copied by `battleCameraStartOffsetAnime` (0x801C2F00), 0x1E0 bytes into
  `lbl_80466E50`) each time the camera switches to "full"; its layout is not
  decoded.
- **Field cameras** (rooms, cutscenes) are a different system: [animation-tracks.md](../engine/animation-tracks.md) and the
  room camera files.

## Changing it

- **Swap a move's camera for an authored one:** give its effect file an own archive with a camera (the 133 files show
  the format); that shot then draws nothing.
- **Different idle behaviour** (timing, targets, skipping) changes the battle's random numbers, so it can only be an
  optional setting, off in a mode that must match the console; framing changes that keep every draw (zoom, distance offsets applied after the draws) are safe.
- **A camera-off or fixed-camera option** must still make the same draws (or be an optional setting only), since skipping the
  procedural picks would shift every later battle event.

## Efficiency

The camera does a handful of float operations per shot and one timer update per frame; the authored cameras are small
animations in already-loaded effect archives. Nothing loads from the disc for the camera itself.

## Open questions

- Which battle moments turn a sequence's camera switch off (`wazaSequenceSys.c`, floor status 0x22), and therefore
  draw nothing.
- Exact draw counts per shot (measure with the RNG trace, attributing draws to the camera functions).
- The grid member count the idle pick uses in doubles (+0x40 and +0x42 of the grid table).
- The layout of the per-room battle grid data.
- Which species is internal id 0x13A (always the fixed wide shot).
