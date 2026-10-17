# WZX: move-effect sequences

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
2. the **sequence header** (`fn_801DC5F0`): +0x00 animation slot of the owner (0x0B-0x0F fall back when the model has
   fewer slots), +0x04 entry count + 1, +0x08 flags, +0x0C a byte value (mode 5), +0x10 header mode, +0x14 size of the
   move's **own resource archive**. Mode 1 = 0x10-byte header, mode 2 = 0x14, otherwise the archive (if its size is not
   0) starts at the next 32-byte boundary after a 0x18-byte header and takes its size rounded up to 32 (133 files have
   one: an HSD archive loaded under resource group 0x4E20);
3. the **entries**, each a node header followed by a payload. Every file ends exactly where its last entry ends
   (checked on all 1,098).

**Node header** (`fn_801DC46C`, 0x70 bytes, or 0x6C when +0x68 = 1):

| Offset | Field |
|---|---|
| 0x00 | key (other entries link to it) |
| 0x04 | entry type (below) |
| 0x08 | attachment (*guess:* which model / joint the entry follows) |
| 0x0C | position type (*guess*) |
| 0x10 | link key: start relative to that earlier entry (0 = relative to the owner's time point) |
| 0x14 | own index: which of its own timing values to subtract from the start |
| 0x18 | base index: which timing value of the linked entry, or which owner time point, to start from |
| 0x1C | length index (1-based): the timing value the loader fills with the effect's length when it is 0 |
| 0x20 | 16 timing values (60 Hz frames; the loader rescales them to the video rate) |
| 0x60 | flags |
| 0x64 | part index |
| 0x68 | size mode (1 = header ends at 0x6C; 2 = +0x6C holds a particle-bank link key) |

**Start time** (`wazaSequenceEntryLink`): start = (linked entry's start + its timing[base index]) or (owner's time
point[base index]), minus the entry's own timing[own index]. After loading, all starts are shifted so the earliest is
0. Links only reach entries earlier in the file.

**Entry payloads** (`wazaSequenceLoadData` and its helpers):

| Type | Name | Payload |
|---|---|---|
| 1 | value | s32 kind, s32 / f32 value, 4 bytes (kind 0 = frames, 1 = int, 2 = float, 3 = skip value x 8 bytes) |
| 2 | model | 0x24-byte header (+0x00..+0x18 seven settings, two of them 0 / 1 switches; +0x1C size), the HSD model archive at the next 32-byte boundary |
| 3 | particles | s32, s32, u32 size, s32 format (+4 bytes when format = 3), then the GPT1 bank of that size ([particles.md](particles.md)); with a link key in the node's +0x6C the entry reuses an earlier entry's bank and has no data |
| 4 | screen effect | a descriptor (below) |
| 5 | sound | s32 sound id, s32 kind, and s32 value for kinds other than 1 / 2 |
| 6 | control | s32 value, 4 bytes; at run time: show / hide the Pokémon, remove its null joint and similar (verified at runtime; which value does what is open) |

**Screen-effect descriptor** (`fn_801364A8`, `effect/tracefx.c`): u32 kind, s32 frames, u32, then per kind:

| Kind | Name | Parameters, then embedded data |
|---|---|---|
| 0 | colour fade (*verified at runtime*: colour keys interpolated over time, tinting the battle room's models) | flags, count, mode, (flags word unless mode 1 / 2), then count x {RGBA from, RGBA to, s32 frames, pad} |
| 1 | electron (*verified*, start function name) | 4 vectors, 3 floats, 2 ints, colour, int; GStexture at the next 32-byte boundary after 0x54 |
| 2 | arcs between two joints (*likely*, effects.md) | colour, floats and ints, 0x1C bytes |
| 3 | lightning (*guess*) | 2 colours, int, 2 vectors, 9 floats, int; texture after 0x54 |
| 4 | trail (*verified*, tracefx) | colour, 3 ints; texture after 0x18 |
| 5 | leaf (*verified*) | 10 floats, 3 ints; model archive after 0x3C |
| 6 | sea: water surface (*likely*) | floats, colour, grid size, floats, mode; model archive after 0x40 / 0x44 |
| 7 | surf: wave model (*likely*) | floats, colour, ints, floats; model archive after 0x34 |
| 8 | env-map shine (*likely*) | 2 ints, count, mode; mode 2: count x {f32, f32, s32 frames, pad} |
| 9 | blur / after-images (*likely*) | count, mode (3 adds a value); count x {f32, f32, s32 frames, pad} |
| 10 | aura (*likely*) | colour, five switches, 3 floats, 0x28 bytes |
| 11 | billboard (`patchiru`, *likely*; the Spinda link a *guess*) | model archive size, ints; optional model after 0x18 |
| 12 | distortion (*likely*) | ints, floats, mode; texture after 0x18 / 0x1C |

Kinds 8 and 9 with mode 1 leave the game's cursor unset (an original bug: the next entry would be read from garbage);
no file on the disc uses them. The kind names follow the effect start functions and how the moves use them (effects.md); a header comment in
`effect_visual.c` pairs names and functions shifted by one, so it is not used.

**On the disc (USA):** 1,098 files: 1,594 particle entries (906 with their own bank, the rest reusing one), 1,322
sounds, 1,068 screen effects (955 colour fades, 39 distortion, 30 billboard, 15 env-map shine, 14 leaf, 13 blur,
2 electron), 691 values, 545 models, 100 control steps.

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
   into resource group 0x4E20), and links it into the start-sorted list.
3. While the sequence runs, entries start at their times: particles through `gsParticleCreate`, models through the
   model system, screen effects through the effect modules (`effect_visual.c`, `tracefx.c`), sounds through the sound
   system.

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

- Meanings: node attachment / position / flags / part, the model entry's settings, the particle entry's two settings
  and format, sound kinds, value kinds in practice, and most screen-effect parameters.
- The owner's time points per species (PKX slots) are needed to place a timeline in absolute frames.
