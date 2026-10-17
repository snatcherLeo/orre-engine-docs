# Fixed object pools (materials, lights, cameras, gapp tasks)

Status: **partial** (2026-10-14). A pattern used all over the `gs_*` engine, not a single
subsystem. Code: `src/game/gs_material.c` (GSmaterialCreate, matched), `src/game/gs_light.c`
(GSlightLoad, matched), `src/game/gs_render_util.c` (`fn_800D27FC`, `fn_800D29A0`),
`src/game/gs_gapp.c` (GSgappCreate).
Every create function named here byte-matches the game, so **the pool table, slot sizes, in-use flags and the
fields listed are *verified*** (read from matching C); interpretations are marked *likely* or *guess* inline.

## What it does

Instead of allocating each object when it's needed, each system preallocates an array of
fixed-size slots at startup and hands out the first free one. A slot is free when its first
byte or word is 0. Creating an object means "find a free slot, fill it in", and destroying it
means clearing the in-use flag.

## Data structures

| Pool | Array / count | Slot size | In-use flag | Created by |
|---|---|---|---|---|
| Materials | `lbl_8047AB1C` / `lbl_8047AB20` | 0x40 | byte 0 | GSmaterialCreate |
| Lights | `lbl_8047AAEC` / `lbl_8047AAF0` | 0x74 | byte 0 | GSlightLoad |
| Cameras | `lbl_8047AA6C` / `lbl_8047AA70` | 0x128 | byte 0 | `fn_800D27FC` (from a desc), `fn_800D29A0` (defaults) |
| gapp tasks | `lbl_8047AC7C` / `lbl_8047AC80` + `lbl_8047AC84` | 0x18 | word at +0x08 (state) | GSgappCreate |
| Resource registry | see [resources.md](resources.md) | 0x14 | data pointer at +0x04 | GSresRegister/Alloc |
| Floor requests | see floor-resources.md (not yet published) | 0x24 | word at +0x08 | six request functions |

Known slot fields:

- **Material:** +0x00 in use; +0x02 u16 cleared; +0x10..+0x1C set to 0, 1, 2, 3 (*guess*: per-stage indices); +0x20/+0x24/+0x28 cleared; +0x38 and +0x3C set to `0xFEFEFEFE` (sentinels; *guess*: "unset" markers
  for colors or pointers). Logs a message when the pool is full.
- **Light:** +0x00 in use; +0x08 = the light description passed in; +0x0C = the HSD light object
  from `HSD_LObjLoadDesc`; +0x02 = has animation (when the desc's second word is nonzero);
  then animation rate (+0x64) and mode (+0x5C).
- **Camera:** an HSD camera object loaded with `HSD_CObjLoadDesc`, plus animation state
  (has-animation, rate, mode, ended flag, count of animations, current index, end frame) and
  eye/interest vectors when it isn't animated (field names in `GSRenderCamera`).
- **gapp task:** +0x00/+0x04 prev/next, +0x08 state, +0x0C priority (u8), +0x0D flag,
  +0x10 parameter, +0x14 function. Tasks in state 2 come from a second pool placed after the
  first and are pushed onto a separate list (`lbl_8047AC9C`) with interrupts disabled. Other
  tasks are inserted by priority into the main list (`lbl_8047AC98`).

## Flow

Create = inline "find first free slot" helper (early return) + fill fields. The find helpers
are recovered from the matched code (*verified*). Destroy paths
are not documented yet.

## Disc data

Lights and cameras are created from descriptions inside map archives (*verified*) (see
`floorReadMapPostFunc`: models, lights and camera are registered after an archive is parsed).

## Changing it

- Pool sizes are runtime globals (*verified*), so a port can enlarge pools (*likely*) (more lights per map, more
  materials) by changing where the counts are set. Check that nothing else assumes the old maximum.
- Running out of slots fails softly (NULL plus a log message for materials; *verified*), so mods that add
  objects should check for that.
- A port needs real structs for these slots: today much of the C uses raw offsets.

## Efficiency

- No allocation or fragmentation at runtime: all memory for these objects is reserved up front (*verified*),
  which keeps frame times predictable (*likely* the intent).
- First-free linear scans are cheap at these sizes (tens of slots; *likely*, sizes not yet read).
- Slots are contiguous (*verified*), which is friendly to the small caches (*likely*).

## Open questions

- Pool sizes and where they're set (init functions for each subsystem).
- What the material fields at +0x10..+0x1C are (TEV stage ids?).
- Exact meaning of gapp task states (state 2 = background or deferred?).
