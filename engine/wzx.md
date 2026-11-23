# WZX: move-effect sequences

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16). The file layout is decoded for every WZX file on the disc; the meaning of
several entry settings and of most screen-effect parameters is still open. Related notes:
models-animation.md (not yet published), "WZX file" (which file a move uses, how entries line up with the
Pokémon's animation).

## What it does

A WZX file is one **part of a move's animation** (`_attack`, `_damage`, ...; ball throws, weather and status effects
too): a list of timed **entries**, each a particle bank, a model, a screen effect, a sound, a value or a marker. The
battle loads it into a sequence (`wazaSequenceLoadData`), places every entry on a timeline relative to the attacking
or defending Pokémon's own animation (`wazaSequenceEntryLink`), and plays them as the animation runs. Because start
times hang off the species' **animation time points** (PKX slot records), the same WZX fits every species' timing.

## Data structures (big-endian; *verified from code* and from all 1,098 files)

**File layout:**
1. a 0x70-byte node header with no content of its own;
2. the **sequence header** (`wazaSequenceLoadHeader`): +0x00 animation slot of the owner (0x0B-0x0F fall back when the model has
   fewer slots), +0x04 entry count + 1, +0x08 flags, +0x0C a byte value (mode 5), +0x10 header mode, +0x14 size of the
   move's **own resource archive**. Flags bit 1 (**verified**): the owner draws in the overlay
   layer while the sequence runs (effects.md, "Overlay effects draw after a depth reset"). Mode 1 = 0x10-byte header, mode 2 = 0x14, otherwise the archive (if its size is not
   0) starts at the next 32-byte boundary after a 0x18-byte header and takes its size rounded up to 32 (133 files have
   one: an HSD archive loaded under resource group 0x4E20);
3. the **entries**, each a node header followed by a payload. Every file ends exactly where its last entry ends
   (checked on all 1,098).

**Node header** (`wazaSequenceLoadEntryParams` (0x801DC46C), 0x70 bytes, or 0x6C when +0x68 = 1):

| Offset | Field |
|---|---|
| 0x00 | key (other entries link to it) |
| 0x04 | entry type (below) |
| 0x08 | **attach point** (0-15, or 16 for the model's bounding-box centre): which of the owner's 16 points the entry follows (**verified**; models-animation.md (not yet published), "Attach points") |
| 0x0C | **position type**, or follow mode (0-7): which of the joint's position (P), rotation (R) and scale (S) the entry follows; the rest are set once as offsets from the owner (**verified**): 0 none, 1 P, 2 R, 3 S, 4 P+R, 5 R+S, 6 P+S, 7 all. With 0 a particle generator sits at the owner's own position (its feet), not at a joint. Point 16 at +0x08 is the owner's centre null joint when it has one (its last joint), else a fixed joint from its model record |
| 0x10 | link key: start relative to that earlier entry (0 = relative to the owner's time point) |
| 0x14 | own index: which of its own timing values to subtract from the start |
| 0x18 | base index: which timing value of the linked entry, or which owner time point, to start from |
| 0x1C | length index (1-based): the timing value the loader fills with the entry's own length when it is 0 (**verified** from `_wazaSequenceEffectEntryLoad` and `_wazaSequenceModelEntryLoad`): a screen effect gets its length (a fade's, blur's or env-map shine's keys summed, where a held fade key counts as -1 because the unsigned sum wraps; the billboard 0; every other kind its frames), a model its animation's length (`GSmodelGetFrameCount`). Links that read that value ("start when that one ends") then land where the game puts them: Howl's stage darkening starts about 4 frames in, not 109, and Roar's rings and darkening move a row earlier (both match the game side by side once this is done) |
| 0x20 | 16 timing values (60 Hz frames; the loader rescales them to the video rate) |
| 0x60 | flags (**verified**, 2026-11-19): 1 attach to a joint of a linked model entry (the part at +0x64), 2 place once instead of following, 4 placed in the world, 8 use the battle scale, 0x10 (with 8) keep the height scale at 1 . Flag 1's linked entry must start no later than this one, or the game logs an error. Flag 1's linked entry is the one whose key equals this entry's link key (+0x10, the same key the start rule uses), and only if it is a model entry (type 2) with its model loaded; a particle entry then takes that model's position and rotation instead of the owner's. If the linked entry is not a model, no part is found and the effect stays where it was created |
| 0x64 | part index |
| 0x68 | size mode (1 = header ends at 0x6C; 2 = +0x6C holds a particle-bank link key) |

**Start time** (`wazaSequenceEntryLink`): start = (linked entry's start + its timing[base index]) or (owner's time
point[base index]), minus the entry's own timing[own index]. After loading, all starts are shifted so the earliest is
0. Links only reach entries earlier in the file.

**Entry payloads** (`wazaSequenceLoadData` and its helpers):

| Type | Name | Payload |
|---|---|---|
| 1 | value | s32 kind, s32 / f32 value, 4 bytes (kind 0 = frames, 1 = int, 2 = float, 3 = skip value x 8 bytes) |
| 2 | model | 0x24-byte header, then the HSD model archive at the next 32-byte boundary after +0x24 (model 0 of its scene). The seven settings (**verified**): +0x00 the joint animation (-1 none), played at half rate; +0x04 its kind (0 once, 1 loop); +0x08 the texture animation (-1 none; ignored if equal to +0x00); +0x0C its kind; +0x10 **carry** (the owner rides this model, below); +0x14 the carry joint; +0x18 model flags: 1 overlay layer, 2 hold the last frame until the sequence stops, 4 "60 fps animation" (*clarified 2026-11-20*, **verified** from `GSmodelSet60fpsAnimFlag` and `GSmodelAdvanceAll`: it only moves the model's advance into the per-retrace pass with the particle steps, interleaving them; the overall rate is unchanged, so it is not a speed change), 8 (carry) the owner keeps its displacement when the entry stops, 0x10 (carry) pivot on the owner's centre, 0x20 cast shadows on the battle's shadow surfaces; +0x1C the archive's size. With size 0 the model can't load and the entry does nothing |
| 3 | particles | s32 **emitter id** (which generator in the bank to start), u32 **mode word** (below), u32 size, s32 format (+4 bytes when format = 3), then the GPT1 bank of that size ([particles.md](particles.md)); with a link key in the node's +0x6C the entry reuses an earlier entry's bank and has no data |

**The particle entry's mode word** (bit numbers, **verified** from code, 2026-11-19): bit 0 draw link; 1 the children
follow the joint too; 2 and 4 apply the offset's rotation in local space; 3 the velocity's rotation in local space; 5 scale
the generator's shape; 7 scale the textures; 8 and 10 tornado motion; 9 scale gravity and friction; 11 scale the node;
12 scale the random velocity. A move's particle entries all set bit 0 , so they draw in the overlay
layer.

| Type | Name | Payload |
|---|---|---|
| 4 | screen effect | a descriptor (below) |
| 5 | sound | s32 sound id, s32 kind, and s32 value for kinds other than 1 / 2. At run time a set bit 0 in the node's runtime flags plays the sound on its own small thread, otherwise directly (verified); that this word is the entry's kind is likely; what the value does is open |
| 6 | control | s32 value, 4 bytes (**verified**, `wazaSequenceEntryStart`): 0 the owner's aura on, 1 aura off, 2 hide the owner, 3 show it, 4 remove its null joint, 5 set its null, 6 clear its null, 7 stop it (hold its motion), 8 release the owner if this sequence is its current one, 9 let it move again; any other value fails the entry |

**Carry** (model entries with +0x10 set; **verified** from `_wazaSequenceModelEntryStart` and a scan
of all three parts of the 355 moves; the non-move sequences, such as Poké Balls and status loops, were not scanned):
the model is placed at the owner (or at the linked model, flags bit 0), turned like it, at the
effect scale, and the **owner is attached to the model's carry joint** with the entry's follow mode, so the Pokémon
rides the model. When the entry stops the owner is put back in place and its scale reset. The carry entries with a
model:
- follow mode 1 (position only): **Dig** and **Dive** (the attack half, held), **Waterfall** (both halves),
  **Earthquake** (the damage half, held), and Dive's third part;
- follow mode 4 (position and turn): **Dig's third part** (its second turn);
- follow mode 6 (position and size): **Withdraw's third part** (a model of four meshes).
The carry entries in Tail Whip, Splash, Encore and Hidden Power, and in the third parts of Fly, Growth, Agility,
Double Team, Minimize, Slack Off and Bounce, use small models that **move along a spline** (**verified** from the code and a PC-port run): their joint field holds 0, but the relocation table lists it, so it
points at a real root joint at data offset 0; a path-animation track moves a joint along a spline
([hsd-models.md](hsd-models.md), "Path animation"). 15 moves have such models: Trick, Skill Swap, Splash, Tail Whip,
Encore, Hidden Power, and the third parts of Fly, Mist, Growth, Agility, Double Team, Minimize, Haze, Slack Off and
Bounce. Splash's carrier hops and the Pokémon riding it (follow mode 6) hops with it; in Trick a joint flies along
a spline carrying the linked sparkles from the user to the target. Both match the game side by side in a rebuild. **How Withdraw hides the Pokémon** (**verified** from the data): the shell's carry joint (joint 4)
shrinks from full size to 0.001 over the model's frames 7 to 11, and with follow mode 6 the Pokémon shrinks with it,
disappearing into the shell.

**A move model's animation runs 2 battle frames behind its entry's start** (likely, from the
game's pictures): in a Swords Dance close-up the game's swords are one picture row (2 frames) behind a rebuild's, while
the sparkles linked to them keep time; with this rule Swords Dance and Withdraw match every row. In Withdraw the shell
itself is late, and the Pokémon simply follows the shell's current pose. Where in the update order the delay comes
from is not read yet.
**A carried Pokémon follows its carrier's pose from one frame earlier than the carrier's own drawn pose** (**verified** side by side on Double Team): Double Team's carrier flips the Pokémon to the other side every
frame, and only with this extra frame does the rebuild's Pokémon stand where the game's after-image is in every row
of a close-up (Splash still matches with it). Why, likely: the Pokémon is updated before the effect model that carries
it, so it reads the pose the carrier had on the previous frame.
Rebuilt this way, the attack and damage halves of Dig, Dive, Waterfall and Earthquake, and Withdraw's third part
(apart from that lag), match the game side by side (Dig's second turn not compared yet) (verified by
eye against the PC port's move sweep).

**Held model entries** (model flags 2; **verified** from `_wazaSequenceModelEntryUpdate`, seen in the
game's pictures): a finished held entry waits on its last frame (state 3) and keeps reporting "running"; nothing stops
it. When the half's ending value entry fires while any entry is held, the sequence ends **held** (sequence +0x15 = -1,
otherwise 1). That is how Dig's hole and Earthquake's cracked ground stay on screen, with the carried Pokémon still in
place (Dig's user stays underground until turn 2). Likely, from the pictures: held effects of the attack half end
when the damage half starts (2 frames after the camera cut), not at the cut itself.

**Surf's water is a morphing mesh** (**verified** side by side with the game): Surf's attack half
has a model entry (one mesh, 3,276 vertices) that uses HSD shape animation ([hsd-models.md](hsd-models.md), "Shape
animation"). Drawn without the morph it is a flat sheet above the Pokémon's head; with it, the waves rise and fall
(about 6 to 10 units high over the move), and the Pokémon's head and ears show above the water as in the game. The
water's faceted look and the clouds in its texture are open. The other morphing effect models: Muddy Water, the
damage half of Sky Uppercut, and the third parts of Sharpen (its sphere-to-cube, matching the game side by side),
Milk Drink and Defense Curl (Sharpen and Defense Curl share the model) (the complete list).

**When a model entry is done** (**verified**): only when both its joint animation (none counts as done at once) and its
own texture animation (+0x08, when it differs from +0x00) have ended. Waterfall's damage half is a model with only a
texture animation: the water column.

**Screen-effect descriptor** (`fn_801364A8`, `effect/tracefx.c`): u32 kind, s32 frames, u32, then per kind. **Offsets
in the table:** where a row says "from +0x0C", its offsets count from the end of those three header words (the
descriptor's start + 0x0C); the other rows list their parameters in order from that same point.

| Kind | Name | Parameters, then embedded data |
|---|---|---|
| 0 | colour fade (*verified at runtime*: colour keys interpolated over time, tinting the battle room's models) | flags, count, mode, (flags word unless mode 1 / 2), then count x {RGBA from, RGBA to, s32 frames, pad} |
| 1 | bolt: Thunder's lightning (*verified*: the loader's switch calls `boltEffectCreate`; effects.md, "The bolt") | from +0x0C (*verified*, 2026-11-19): vectors jitter +0x00, spread +0x0C, scale +0x18, step +0x24; floats height +0x30, width +0x34, branch chance +0x38; u16 count +0x3C and depth +0x40; colour +0x44 (red in the word's low byte, as in the fades); owner's part +0x48; texture size +0x4C; the GStexture at the next 32-byte boundary after 0x54 |
| 2 | arcs between two joints (*likely*, effects.md) | colour, floats and ints, 0x1C bytes |
| 3 | lightning (*verified* name: `lightningEffectCreate`; what it draws is not checked, and no move uses it) | 2 colours, int, 2 vectors, 9 floats, int; texture after 0x54 |
| 4 | trail (*verified*, tracefx) | colour, 3 ints; texture after 0x18 |
| 5 | leaf (*verified*; effects.md, "The leaves") | from +0x0C (*verified*, 2026-11-19): ten floats +0x00-+0x24 (size base / range, end-point distance base / range, control-point distance base / range, lift, speed per tick, first end-point factor base / range); u16 count +0x28 (capped at 32); fade start +0x2C (ticks); attach point +0x30; leaf model archive size +0x34; the archive at the next 32-byte boundary after +0x3C |
| 6 | sea: water surface (*likely*) | floats, colour, grid size, floats, mode; model archive after 0x40 / 0x44 |
| 7 | surf: wave model (*likely*) | floats, colour, ints, floats; model archive after 0x34 |
| 8 | env-map shine (*verified*, 2026-11-20; effects.md, "The env-map shine") | from +0x0C: +0x00 mapping mode (0 reflection, 1 gradient, other backlight), +0x04 type (0 multiply, else blend), +0x08 count, +0x0C key mode. Key mode 2: count keys {f32 from, f32 to, s32 frames, pad} from +0x10; key mode 1: the +0x08 word read as one float value, over the entry's frames in thirds (rise, hold, fall). Frames scaled by the video rate |
| 9 | blur / after-images (*verified*; effects.md, type 9) | count, mode (3 adds a value); count x {f32, f32, s32 frames, pad} |
| 10 | aura (*likely*) | colour, five switches, 3 floats, 0x28 bytes |
| 11 | billboard (`patchiru`; *verified*, effects.md, "The billboard in detail"; Spinda's spots are painted by other code, models-animation.md (not yet published), so the shared name is likely all this kind has to do with Spinda) | model archive size, ints; optional model after 0x18 |
| 12 | screen warp (distortion; *verified*, 2026-11-20; effects.md, "The screen warp") | from +0x0C: +0x00 draw particles into the capture too; +0x04 follow the owner (place re-read every update); +0x08 size; +0x0C strength; +0x10 the warp picture's size; +0x14 mode: 1 = peak at 1.0, the picture after +0x18; 2 = peak at the float at +0x18, the picture after +0x1C |

**Every kind's name is verified** : the loader `_wazaSequenceEffectEntryLoad` calls one create
function per kind, in the table's order (effects.md, "Which function each type starts"); the "likely" labels above
are about what each kind's parameters mean. The loader also keeps each effect's attach point (kind 1 its +0x58 word,
kind 3 its +0x16, kind 4 its +0x30 with +0x32 the animation mode, kind 5 its +0x52), and an effect's run length is
its frames scaled by the video rate.

Kinds 8 and 9 with mode 1 leave the game's cursor unset (an original bug: the next entry would be read from garbage);
no file on the disc uses them. The kind names follow the effect start functions and how the moves use them (effects.md); a header comment in
`effect_visual.c` pairs names and functions shifted by one, so it is not used.

**On the disc (USA):** 1,098 files: 1,594 particle entries (906 with their own bank, the rest reusing one), 1,322
sounds, 1,068 screen effects (955 colour fades, 39 distortion, 30 billboard, 15 env-map shine, 14 leaf, 13 blur,
2 bolt), 691 values, 545 models, 100 control steps.

## Flow

*Runtime side (verified by running the game's decompiled loader natively on all 1,098 files; details and the runtime record table in
models-animation.md (not yet published), "Running a move"):* a battle keeps a pool of Pokémon records; attaching a
WZX parses it into a move sequence (one 0xB4-byte record per entry) on the Pokémon's list, found again by two keys:
the move and the **side** (1 = attacker, 2 = target, likely); starting it stops the same move on the other records.
Screen-effect parameters become a 0xD8-byte record (type, duration, the kind's own block) run by the effect system
(`gs_effect.c`, 32 slots). In that run every entry except sound cues (audio off) started. One detail the file layout
settles: particle banks inside a WZX are **not** re-aligned by the loader (only models, effect textures and archives
are), which is why every file ends exactly at its last entry.


1. The battle picks the archive and entry for a move and part (models-animation.md (not yet published), "Which WZX file a move uses").
2. `wazaSequenceLoadData` reads the header, loads the own archive, then each entry (models, banks, effect textures go
   into resource group 0x4E20), and links it into the start-sorted list. It also shifts every entry's start so the
   earliest is 0, and keeps the shift in the sequence (+0x10). The owner's animation then starts that much early (the
   half's **lead-in**, **verified**; models-animation.md (not yet published), "A half's lead-in").
3. While the sequence runs, entries start at their times: particles through `gsParticleCreate`, models through the
   model system, screen effects through the effect modules (`effect_visual.c`, `tracefx.c`), sounds through the sound
   system.
4. A half keeps playing until its last particle, screen effect, sound and model animation have ended, which can be
   well after it counts as done (**verified**; models-animation.md (not yet published), "When a half finishes playing").
5. **Colour fades on the same target replace each other** (likely; seen on Thunderbolt): the user
   half darkens the stage and holds it, then the target half's stage fade brings it back. After that fade ends, the
   stage stays at its normal colour in the game's pictures, so the newer fade replaced the held one rather than both
   running. Also likely (seen on Thunder): **a held fade ends when its half's sequence ends**. The user half's
   darkening holds, yet the stage is back to its normal colour for the target half's first 15 ticks, before that
   half's own darkening starts.

## Poké Ball sequences: how the balls and their effects are built

(**verified** from the files unless marked; Poké Ball checked in full, the other balls by their
matching sizes and structure):
- **Each ball type has its own set of sequences:** `<ball>_out` (send-out), `_in` (return), `_open`, `_downin`
  (drop-in), and for snags `snatch_attack_<ball>` (the Snag Ball throw, played on the trainer's animation slot 7),
  `snatch_shake_<ball>`, `snatch_miss_<ball>`, `snatch_ball_land_<ball>`, plus shared `snatch_poke`, `snatch_shake`
  and `snatch_snatch` (the success). The ball table points each ball at its sequences
  (game-data-tables.md (not yet published)).
- **The ball itself is one rigid model with one joint:** for the Poké Ball, 120 triangles about 0.9 units across, one
  64x128 texture. **It has no separate lid or button parts**, so the original never opens a lid; the opening is shown
  by the light effects around it (**likely** from the parts list; to confirm by watching).
- **It is the same file everywhere:** the ball embedded in `monsterball_open`, `snatch_attack_monster`,
  `snatch_shake_monster` and the others is **byte-identical** to the field model `ball_monster` that the Pokémon
  Centers place on the healing machine. Each ball type has its own copy (the twelve field models: pokemon-data.md,
  "Items as 3D models"; **likely** one-to-one with the sequences).
- **The movement comes from separate animation-only parts:** small embedded models with no geometry and one animation
  (15 to 110 frames: the throw arc, the shakes, the landing), which the ball and effects hang from through the entries'
  attach settings (**likely** from the structure; the attach semantics are partly open, above).
- **The flash and light around the ball are their own models:** the send-out's largest part (about 120 KB for the Poké
  Ball, 18 joints, 17 meshes) is all **semi-transparent effect geometry** (flat rings and light shapes, six 128x128
  textures), not the ball. The Master, Great and Net Balls' versions carry a few more parts (19-21 joints). The snag
  throw has its own effect models (about 60 KB and 250 KB), and each send-out has a **particle bank** (about 200 KB for
  the Poké Ball) and a **sound**.
- **For replacement:** a new ball model must be one joint and one mesh at the original's size and origin, so every
  sequence that embeds it lines up. A ball with moving parts (a lid that opens) would be a **new behaviour**, not a swap:
  the sequences have no slot that moves a lid.

## Disc data and tools

- With the layout above a reader can split every WZX file into its timeline (frame, start rule, type,
  settings) and its embedded parts: particle banks, models and archives (HSD `.dat`, loadable in Blender) and effect
  textures (*verified*: all 1,098 files on the disc read with nothing left over).

## Changing it (for mods)

- Retiming: edit timing values (+0x20) or the start rule (link key, base and own indices); everything else follows.
- Swapping particles, models or textures: replace the embedded data and its size field, keeping 32-byte alignment where
  the loader aligns (models, effect textures and archives); particle banks are not re-aligned by the loader.
- New entries: append (the count is the sequence header +0x04 minus 1) and keep keys unique.

## Open questions

- Meanings: the particle entry's
  format (its emitter id and mode word are now known, above), sound kinds, value kinds in practice, and most
  screen-effect parameters. (the node's attach point, position type, the
  follow-once flag (bit 1) and the flag to attach to a joint of a linked model entry (bit 0, with the part at +0x64)
  are now known:
  models-animation.md (not yet published), "Attach points".)
- The owner's time points per species (PKX slots) are needed to place a timeline in absolute frames.
