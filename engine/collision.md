# Collision (CCD files, GScolsys2)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-14). Format decoded and exported for every room; which query uses
which part is verified from the code; a few attribute meanings are guesses (marked).

## What it does

Every room has one collision file (resource type 3, `.col` on the disc, "CCD" in the code). It holds
the room's invisible geometry: the floor people walk on, the walls that stop them, and the planes
that trigger events when walked through (doors, exits, cutscene triggers) or examined with A (signs,
objects). The map model is only for drawing; movement and events use this file.

## Data structures

**File header:** `{u32 records offset, u32 record count}` (`CCD_FILEHEAD`). All pointers in the
file are offsets from its start; `_offsetCCD__FP12CCD_FILEHEAD` (`src/game/gs_range_8010CBD0.c`)
turns them into pointers in place when the room loads.

**Collision object record (0x40 bytes)**, one per collision object:

| Offset | Field | Evidence |
|---|---|---|
| +0x00 | f32[3] translation, f32[3] rotation, f32[3] scale | data: (0,0,0), (0,0,0), (1,1,1) for static objects; copied per object by `fn_8010CD6C`, matrix built by `fn_8010CA30` / `fn_8010C8D0` (order of rotations: guess) |
| +0x24 | group: **floor** | height / layer queries `fn_8010E138`, `GScolsys2WalkGetLayer`; normals face up |
| +0x28 | group: **walls for the player and partner** | `colsysWallAll(.., useAlternate 0)` from `colsysWallSweep` (`hero_move.c`, partner code `gs_range_801C766C.c`) |
| +0x2C | group: **walk-through event triggers** | `colsysPassGetEventID` (`people.c`), `GScolsys2ThruGetEventList` |
| +0x30 | group: **examine (A button) event planes** | `GScolsys2Check` -> `fn_80111864` |
| +0x34 | group: **walls for NPCs** | `colsysWallAll(.., useAlternate 1)` from `fn_8010F188` (`people.c`) |
| +0x38 | group with 0x30-byte triangles | only the debug drawer reads it so far (open) |
| +0x3C | u16 flags: bit 0 = **moving object** | queries transform by the object's own matrix (`fn_8010CA30`) instead of using the grid; the debug view draws static objects once into a display list and moving ones every frame (`GScolsys2Draw`, `fn_8010D8D4`) |

`GScolsys2GetObjEnable` / `GScolsys2SetObjEnable` switch single objects on and off (e.g. a door that
opens), and every query skips disabled objects.

**Group (0x24 bytes):**

| Offset | Field |
|---|---|
| +0x00 | triangles |
| +0x04 | u32 triangle count |
| +0x08 | grid cells: `{u32 first index, u32 count}` per cell |
| +0x0C | u32 triangle index list (the cells point into it) |
| +0x10 | u16 cells across (x), u16 cells deep (z) |
| +0x14 | f32 cell width, f32 cell depth (40 in every room checked) |
| +0x1C | f32 minimum x, f32 minimum z (exactly the lowest vertex of the group) |

The grid order of +0x14 / +0x1C is from the data (M2_out floor: 12 x 28 cells of 40 starting at its
minimum corner, covering the floor's extent). The assembly agrees: every grid lookup computes
`(position - min) / cellSize`, loading the subtracted value from +0x1C/+0x20 and the divisor from
+0x14/+0x18 (`GScolsys2ThruGetFixedMdlEventList` 0x8011069C, `colsysWallSphere`, `fn_8010E138`).
The loader fixes
up only the triangle pointer for the +0x30 and +0x38 groups (no grid there).

**Triangle (0x34 bytes; 0x30 in the +0x38 group):**

| Offset | Field |
|---|---|
| +0x00 | 3 vertices, f32 x, y, z each |
| +0x24 | f32[3] normal |
| +0x30 | u8 attribute A: on floors 0xFF / 0x0F / 0xF0 seen; the debug drawer tints by its two nibbles (guess: floor layer bits) |
| +0x31 | u8 attribute B: on walls 2, 3, 4, 6, 7 (guess: which edges are outer edges, bits 0-2); on floors 0, 1, 17 (guess: sub-layer / surface kind) |
| +0x32 | u16 **event id** of walk-through triggers (one id per trigger object: 4, 5, 6 ... in Pyrite Town). **Examine planes keep their id in the u16 at +0x30 instead** (`fn_80111864` compares +0x30; Pyrite Town: 1, 2, 3, 10) |

The floor-hit record returned by `fn_8010E138` has height, surface type, surface id, layer and
sub-layer (`ColWalkHit`); `GScolsys2WalkGetLayer` picks the hit closest in height and returns its
layer and sub-layer. Which triangle bytes feed which field is still to read from `fn_8010E138`.

**checks on the whole disc (*verified*):** the layout above holds for all 124 collision files (1,348 objects,
2,025 groups): every block lies inside its file, every grid index is a valid triangle, no triangle array is shared
between group kinds. A port that converts the file to native byte order must swap it field by field (the 16-bit
fields and the attribute bytes are not 32-bit words). **End to end:** the decompiled floor query (`fn_8010E138`,
`GScolsys2WalkGetLayer`), compiled and run natively, answers 200 random floor-height queries in every room with a
floor group (122) exactly like an independent re-implementation reading the original big-endian file: 24,400 queries,
0 mismatches (M2_earth_colo and T1_ancient_colo have no floor group).

**Floor triangles count in one winding only (verified from the code):** the floor query's edge test
(`getCpPolyVec`) rejects a triangle whose winding is reversed, so reversed triangles in a floor group are never stood on.
A reference that accepts both windings disagrees in 34 of 122 rooms (e.g. D2_valley_1: a step face about 6.6 units
below the walkable one is ignored). *Likely* deliberate: risers and undersides can sit in the floor group without
catching the player. *For modders:* floor triangles must face up with the game's winding, or they don't count.

## Flow

1. Room load: the room archive's type-3 entry is read through the resource table; the post-load
   callback `floorReadColPostFunc` calls `fn_8010CFE4`, which fixes the offsets and makes the file
   the current collision (state `lbl_80404C68`, layers of 0xDC0 bytes; `GScolsys2GetCurFloor`).
2. Every frame, movement code asks the collision system: the player (`hero_move.c`) and partner
   test their path against the player walls (`colsysWallSweep`), NPCs (`people.c`) against the NPC
   walls (`fn_8010F188`), and both look up the floor height and layer.
3. Walking through a trigger plane returns its event id (`colsysPassGetEventID`); pressing A in
   front of an examine plane does the same (`GScolsys2Check`). The id selects the event in the
   hero-event table (events.md, common-script.md).
4. `GScolsys2UnloadCCD` drops it when the room unloads.

**Shared header:** the triangle, triangle-list and grid types are defined once in
`include/game/gs_colsys_ccd.h`.

## Disc data

- One type-3 entry per room archive (`<room>.fsys`), named like the room (`S1_out`).
- With the layout above a reader can convert each file to a mesh, one object per collision object and group,
  and list each object's groups and event ids (*verified* on every room).

## Changing it

**For modders (*likely*, from the formats involved):** a room mod changes up to four things, each with its own data: the
visible map model (HSD archive in the room's FSYS), the collision file (this format: triangles in six groups plus
the 40-unit grids, which a tool can rebuild), the room module's tables (NPC placements, cameras, treasure; see
`game-data-tables.md` (not yet published)), and the room's script for events and triggers. Collision and visuals are separate, so
moving geometry needs both edited together; trigger event ids must match the script.


(*Likely*: the steps below follow from the verified format and have not been tried in the game.)

- New walls or floors: add triangles to a group and rebuild its grid (cells of 40, min corner =
  lowest vertex). A port could skip the grid and test all triangles; vanilla mode should keep it so
  results and order stay identical.
- New triggers: add a pass / check plane with a new event id, and the matching event in the room's
  script or module.
- Moving parts (flag bit 0) must stay matched to whatever moves them (the object index is the key).

## Efficiency

Static geometry is bucketed into a 2D grid of 40 x 40 cells over x/z, so a query only tests the
triangles in the cells it touches; queries stop after 4 results (*verified* from the code). Moving objects are few and small
and are tested brute force through their matrix. Separate wall sets for the player and NPCs avoid
filtering at run time (*likely* the intent). The debug view records static collision into one display list once.

## Open questions

- Meaning of attribute bytes A and B on floors and walls (layer, surface type, edge mask: guesses).
- The +0x38 group (0x30-byte triangles): who uses it.
- The rotation order in `fn_8010CA30` (matters only for moving objects with rotation).
