# Particles: GPT1 banks and the particle byte-code

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16). The file format is decoded and every bank on the disc reads cleanly;
what each flag bit means is only partly known. **How sure:** the layouts come from the loader code and the byte-code
table from the interpreter (`psInterpretParticle0`), both checked against every bank on the disc (*verified*); remarks
on purpose are marked *likely* inline.

## What it does

Every particle effect in the game (room smoke and sparkles, the sleep and freeze loops, move effects, the Pokémon
ambient effects such as Charizard's tail flame) comes from a **GPT1 bank**: HAL's HSD particle system, the same family
as Melee's. A bank holds **emitter programs** (command lists) and **textures**. When something starts an emitter
(a room's script, a model joint, an animation key, a move effect), a **generator** spawns particles; each particle runs
the emitter's **byte-code**, a tiny program of waits and changes (colour, size, rotation, velocity, texture frame,
spawning more particles or generators).

## Where the banks are (data, all USA disc)

| Where | Resource type | Count | Notes |
|---|---|---|---|
| room archives (`*_bg_effect`, `*_ptl`, `*_pati`, named after the room) | 10 (`floorReadParticle*`) | 48 | one or two per room that has effects |
| `ex_shrine` (`relive_gpt`), `pokemonchange_menu` (`poke_exchange_gpt`), `waza_viewer` (`M1_water_colo_pati`), `world_map` (`bike_effect_poke`) | 10 | 4 | relive ceremony, trade, move viewer, world-map bike (names) |
| `waza_sequence_global` (`nemuri_loop`, `koori_loop`) | 18 (`floorReadNotLinkedParticle*`) | 2 | sleep and freeze status loops |
| inside PKX battle models | (PKX header +0x04) | 15 | ambient effects (models-animation.md (not yet published)) |
| inside WZX move effects | (embedded) | 906 banks in 865 files | the moves' particles |

All 975 are version 0x43 and parse with the layout below; their 40,100 emitter programs all disassemble to an end
(or a jump back to a mark) with no unknown opcode and no overrun (*verified, data, 2026-10-16*).

## Data structures (big-endian)

**GPT1 header** (`gsParticleBankLoad`, `src/game/field_range_80117E58.c`):

| Offset | Field |
|---|---|
| 0x00 | magic `GPT1` (`GPT0` is rejected) |
| 0x04 | offset of the command block (0x20 in every bank) |
| 0x08 | offset of the texture block |
| 0x0C | texture block size |
| 0x10 | offset of the id table (directly after the texture block) |

**Command block** (`psInitDataBank`, `src/game/particle.c`): u16 version (0x40-0x43; version 0 has no first id), u16,
u32 first emitter id, u32 count, then one u32 offset per emitter (from the block start; 0 = none). Emitter ids run
from the first id upward; every bank on this disc starts at 0 except some PKX banks (Charizard's at 0x190).

**Emitter** (`HSD_PSCmdList`, `include/sysdolphin/baselib/psstructs.h`), 0x3C bytes then its byte-code:

| Offset | Field |
|---|---|
| 0x00 | u16 type (generator shape and flags) |
| 0x02 | u16 texture group |
| 0x04 | u16 generator life (frames the generator keeps spawning) |
| 0x06 | u16 particle life (frames) |
| 0x08 | u32 kind (particle flags; the loader forces bits 25-27 to 4) |
| 0x0C | gravity, friction, velocity x / y / z, radius, angle, random, size, param 1-3 (12 floats) |
| 0x3C | byte-code (until the next emitter or the texture block) |

**Texture block:** u32 number of groups, then one offset per group (from the block start). A group
(`HSD_PSTexGroup`): u32 image count, u32 GX format, u32 palette format, u32 width, u32 height, u16 palette count, u16
palette flags, then the image offsets and, for paletted formats (C4 / C8 / C14X2), the palette offsets (one shared
palette when palette flag bit 0 is set, else `palette count` of them, else one per image). Formats seen: I8 (3,400
groups), I4 (415), RGBA8 (266), IA8 (203), IA4 (43), RGB565 (7); no paletted groups on this disc.

**Id table:** one u32 per emitter. Some commands look up the emitter they start through this table (marked "id table"
below); the others use the id directly.

## The byte-code (`psInterpretParticle0`, `src/game/psinterpret.c`; *verified from code*)

A particle runs commands until one waits; each frame the wait counts down. Times are 1 byte, or 2 bytes when bit 7
is set (15 bits). Floats are 4 bytes. Multi-byte operands are big-endian.

| Byte | Operands | Meaning |
|---|---|---|
| 0x00-0x3F | (1 more byte if bit 5) | wait: 5-bit time, or 13-bit with the next byte |
| 0x40-0x7F | (time byte if bit 5), frame | set the texture frame, then wait |
| 0x80-0x87 | 1-3 floats (x / y / z by bits 0-2) | set position |
| 0x88-0x8F | floats as above | move by |
| 0x90-0x97 | floats as above | set velocity |
| 0x98-0x9F | floats as above | add velocity |
| 0xA0 | time, float | size to value over time |
| 0xA1 | | texture off |
| 0xA2 / 0xA3 | float | gravity / friction |
| 0xA4, 0xB9 | u16 id | spawn a child particle (0xB9 also copies the velocity) |
| 0xF1, 0xF2 | u16 index | the same through the id table |
| 0xA5 | u16 id | start a generator |
| 0xEF, 0xF0 | u16 id, u8 | start a generator (0xF0 through the id table) with 3 kind bits |
| 0xA6 | u16, u16 | life = base + random range |
| 0xA7 | u8 | die with a chance (percent) |
| 0xA8 | 3 floats | random position offset |
| 0xA9 | float | random direction within an angle |
| 0xAA | u16, u16 | spawn a random child from a range (id table) |
| 0xAB / 0xBE | 1 / 3 floats | scale velocity |
| 0xAC | time, 2 floats | size to value + random over time |
| 0xAD, 0xAE, 0xAF, 0xB0, 0xB1, 0xB4, 0xB5, 0xE2, 0xE6, 0xE7, 0xF7 | | set / clear particle kind bits (0xAD = 0x80 always comes just before an environment-colour command: *likely* "use the environment colour"; 0xAE / 0xAF / 0xB0 / 0xB1 pick bits 0x20 / 0x40) |
| 0xB2 | | detach from the parent transform (bake the world position) |
| 0xB3 | time, 3 bytes | alpha-compare ramp (mode, two reference values) |
| 0xB6 | time, float | rotate by over time |
| 0xB7 | u8 | aim at a joint point |
| 0xB8 | u8, 2 floats | pull toward a joint point (force, kill radius) |
| 0xBA / 0xBB / 0xE0 | 4 signed bytes | random primary / environment / both colour offset |
| 0xBC | 2 bytes | texture frame + random |
| 0xBD | 2 floats | speed = value + random |
| 0xBF | u8 | follow a joint point |
| 0xC0-0xCF | time, 1-4 bytes (r / g / b / a by bits 0-3) | primary colour ramp |
| 0xD0-0xDF | as above | environment colour ramp |
| 0xE3 | u8 | palette |
| 0xE4 / 0xE5 | u8 (off, on, toggle, random) | flip S / flip T |
| 0xE8 | float | trail length (negative = off) |
| 0xE9 | flags, steps, 0-4 signed bytes | random colour offset, stepped |
| 0xEA / 0xEB | time, flags, 0-2 bytes | material / ambient colour ramp |
| 0xED | 2 floats, u8 | rotate by value + random (stepped) |
| 0xF3 | u8, 2 floats, time | spin (direction, amount, spread, time) |
| 0xF4 | 4 floats | random direction around a vector |
| 0xF5 / 0xF6 | | generator transform flags 0x2000 / 0x1000 |
| 0xFA | u8 | loop start (count) |
| 0xFB | | loop end |
| 0xFC / 0xFD | | set a mark / jump to the mark (a repeating program) |
| 0xFE, 0xFF | | end (the particle dies next frame) |
| other | | no operands, no effect |

**the position and velocity commands (0x80-0x9F) use all three components even when they name
fewer** (**verified** from the original code). Each command fills only the x / y / z its bits name into a local vector,
then sets or adds all three. The components it doesn't name keep whatever that vector's memory held before. In the
original program each of the four commands has its own place for that vector on the interpreter's stack (set position,
move by, set velocity and add velocity each use a different slot), so a missing component is a leftover: from an
earlier run of the same command at the same stack depth, or from whatever other code used that memory last. That makes
the result depend on what ran before, not on the effect file alone. Pursuit's streaks rely on it (one of its move-by
commands names only y and z). A port or tool that reproduces these effects has to give those components the console's
values, not arbitrary ones. For Pursuit's move-by command (its x not given) the console's leftover is **0.0**
(**verified** by a memory watch on an undisturbed run): Other code had left that memory at zero just before. It is
"whatever the stack holds", so 0.0 is likely, not guaranteed, in other effects; the other three commands are not
measured yet. The particle draws also use the game's one shared random sequence (`rng.md`), so
anything that changes which random numbers particles get (a replay tool feeding in recorded numbers, for example)
changes how sprays and spreads look while the battle itself stays the same.

**the sleep loop's "Z" particles stay where they started** (**likely**, seen on the console): when a
sleeping Pokémon is knocked back by a hit or falls as it faints, its Z's keep floating at the spot where the loop
began instead of following the model. A port or mod that attaches the loop to the model would change how vanilla
looks.

**camera-facing bursts: emitters started by other particles** (**verified** from the code and, for
the counts, from every move-effect file on the US disc). An emitter can carry a placement of its own (kind bit
0x20000; a position, rotation and scale block). On the US disc **no emitter that a move-effect entry starts directly
has one**, while **3,595 emitters started by other particles** (commands 0xA5, 0xEF, 0xF0) do. Nothing in the game
sets that block's rotation or scale, only its position: the spawning particle's position. While the generator's byte
at +0x72 is set (it owns its frames; command 0xF5 clears it), `psDispSubAppSRT` draws the particles' positions as
offsets along the **camera's** axes around that position (the rotation of the inverse view matrix). So a burst a
particle spawns always spreads out facing the camera, from wherever that particle was.

## Flow

1. A room loads its type-10 entries through `floorReadParticle*`; a PKX or WZX bank is loaded by the model or move
   loader. All go through `gsParticleBankLoad`, which hands the blocks to `psInitDataBank` (64 bank slots).
2. Something starts an emitter: a room script, a joint flagged for particles or an animation particle key
   (`gsParticlePtclHook`, models-animation.md (not yet published)), or a move effect's particle entry.
   For move effects, a particle entry creates a **field-particle node** (0x74 bytes: active, selector, bank, generator,
   local position / rotation / scale, attach mode, attached model and part) that positions, scales and attaches the
   generator to a model part (verified at runtime: models-animation.md (not yet published), "Running a move").
   All 15 PKX ambient banks and all 865 WZX files with banks run through the game's own particle code, compiled and
   run natively (*verified*).
3. The generator spawns particles every frame of its life; each particle runs its byte-code (above) and is drawn with
   its texture group, colours and size (`psdisp.c`).
4. **How a particle is drawn** (**verified** from `psdisp.c` and per-step logs of recorded play):
   `psDispMain` walks the particle lists twice a frame (the edge-sorted ones first, then the rest), sets each one's
   blend mode, alpha test, colours and texture, and draws it. Every particle is drawn under the camera's own view
   matrix with its **world positions worked out on the CPU**: the four corners of a square facing the camera, a line
   from its last position to its current one (trails), or a point. A particle attached to a model has its place turned
   into world space first. So a particle has no matrix of its own; its place is only in its vertices.
5. **Particle slots are reused at once** (**verified**): a particle's life counts down by one each frame, and a freed
   particle's memory is taken by the next new one straight away, so the same slot can hold a different particle on the
   very next step (seen: the same address and generator, life 977 and then 1,000). Anything that tracks particles
   across frames (a renderer blending between steps, a debugger) has to check the bank, the generator id and that the
   life didn't go up, not only the address.
6. **More from the code** (**verified** from the code):
   - **One step is one 60 Hz retrace:** a battle running at 30 frames a second steps the particles twice per frame.
   - **Pokémon emitters fire from animation tracks:** all 12 Pokémon effect models start their emitters from animation
     tracks of one type (0x28); none uses joints flagged for particles. Torkoal's animation 4 also has tracks of type
     0x2A, which set the point joint its "pull toward a point" command (0xB8) aims at.
   - **A Pokémon's generator follows the joint's position only,** and emits along the world axes; once emitted, its
     particles stay where they were, they don't follow the Pokémon.
   - **No depth sorting:** a new particle goes to the head of its list, so within a sort group the newest is drawn
     first.

## Disc data and tools

- With the layout and byte-code table above, a reader can find every bank (stand-alone, in PKX models and in WZX
  files), list each emitter program and decode its texture groups to images (*verified*: all 975 banks).

## Changing it (for mods)

(*Likely*: these follow from the verified format; no bank has been rebuilt yet.)

- An emitter is self-contained: its settings and byte-code, plus the texture group it names. New effects can be made by
  writing new emitters and texture groups and rebuilding the offset tables (no relocation table: all offsets are from
  the block starts).
- Ids: commands that go through the id table let an emitter be renumbered by editing the table only.
- Keep the version at 0x43 and the command block at 0x20; the loader rejects `GPT0`.

### Mod surface
- **Data it reads:** particle banks (`GPT1`: emitters, their byte-code, texture groups, the id table), from move
  effect files, the global effect archive and room archives.
- **Hooks it would need:** a bank substitution by file and emitter id (particle packs, the shared asset library), and
  a separate random generator for particles a mod adds.
- **Fixed-size limits** (**verified** from the code): the particle, generator and placement
  pools have **no fixed size**: none is allocated ahead, every allocator takes new memory from the game's heap when its
  free list is empty, and freed objects go back on the lists. The real limits are the heap itself, **64 banks**,
  **32 field banks × 64 slots** (a full field bank kills its oldest node to make room), **128 field nodes** in all
  (past that, a new one isn't created), **8 point joints**, and fewer than 8 links.
- **Must stay vanilla in GameCube-accurate mode:** the original banks, and their random draws on the shared
  generator (particles are the largest single user of it, rng.md).

## Efficiency

Banks are small (a few KB to ~100 KB, mostly textures) and loaded once per room or move (*verified*); textures are
compact intensity formats (I8 / I4) tinted by the colour ramps, so one grey puff serves many colours (*likely* the
intent). The byte-code costs only work on frames where a wait ends (*verified* from the interpreter).

## Open questions

- The exact meaning of the particle kind bits and the emitter `type` bits (generator shapes).
- Which in-game effect each room bank is (names only; a visual check when convenient).
