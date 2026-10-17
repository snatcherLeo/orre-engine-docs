# Lens flare (GFL, resource type 17)

Status: **complete** (2026-10-16): format and trigger. Every flare function named here byte-matches the game,
so the layout and flow are *verified* (read from matching C and checked against the one file on the disc).

## What it does

A camera lens flare over a battle stage: when the sun (a 3D position set by `flareSetSunPos`) projects near the screen,
a row of flare sprites is drawn along the line from the sun through the screen centre, and a full-screen glare fades
in as the sun gets close to the centre. The only GFL on the USA disc is in `T1_ancient_colo` (the T1 arena, likely Orre Colosseum; fsys-catalog.md), so the flare belongs to
that stage (*verified*).

## Data structures (big-endian; verified from code: `flareInit`, `src/game/battle/battle_range_exact_801ED640.c`, and `flareDraw`, `battle_range_candidate_801ED780.c`; and the one file)

| Offset | Field |
|---|---|
| 0x00 | magic 0x7B1EE3F0 |
| 0x04 | u16 texture count |
| 0x06 | u16 element count (also where the texture table starts, in 12-byte units after +0x1C) |
| 0x08 | u32 far distance: the flare starts fading in when the sun is this close to the centre (*likely* screen pixels) |
| 0x0C | u32 mid distance: full flare; the glare starts |
| 0x10 | u32 near distance: full glare |
| 0x14 | f32 x 2 glare strength (the second is used unless `fn_800D2F34` reports 2) |
| 0x1C | elements, 12 bytes: u8 type (1 = drawn on the sun, 0 = along the line), pad, u16 texture id, f32 position along the line (in thirds of the sun-centre distance; negative = past the centre), f32 scale |
| then | textures, 8 bytes: u16 id, pad, u32 offset of a GStexture file (type-9 layout) |

The T1 flare has 7 elements on 4 RGB565 128x128 textures (*verified*).

## Flow

The room loader's GFL callback (`floorReadGFLPostFunc`) calls `flareInit`, which loads the textures into the flare's
texture table; `flareCreateTask` registers `flareDraw` as a draw task; `flareSetEnable` and `flareSetSunPos` switch it
on and place the sun; `flareRelease` turns it off. Entering a room, `floor.c` turns the flare off, then, if the room's
lighting record names a sun (only T1: map model set 3), places the sun at that model's first part and turns the flare
on; so the flare appears on the T1 stage whenever it is loaded (*verified* from the code).

## Disc data and tools

The GFL is entry type 17 in a room archive; with the layout above a reader can list its elements and decode
its textures (type-9 layout, [textures.md](textures.md)) (*verified* on the one file).

## Changing it (for mods)

Any stage can get a flare with data alone: a GFL entry in its archive plus a sun in its room module's lighting record
(a map model set whose first part marks the sun, in the room module's records);
elements and textures can be retuned freely (positions in thirds of the sun-centre distance, scales relative to the
texture size).

## Open questions

- None known.
