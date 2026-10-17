# Animation tracks (HSD AObj / FObj) and room cameras

Status: **partial** (2026-10-14 night). The keyframe format is decoded and verified on every room camera; joint,
material and light animations use the same format but have not been decoded yet.
Code: `src/hsd/hsd_aobj_range_801C01C8.c` (AObj, ForeachAnim), `src/hsd/hsd_dobj.c` (FObj load / interpret).
**How sure:** the track format is read from the decompiled HSD code and decoded on every room camera on the disc
(*verified*); remarks on purpose are marked *likely* or *guess* inline.
Reference: Melee `sysdolphin/baselib/aobj.c`, `fobj.c`, `cobj.c`, `wobj.c` (same library).

## What it does

Everything that moves on its own in an HSD scene is animated the same way: a camera flying along a path, a
joint rotating, a material scrolling its texture, a light changing colour. Each animated object has an **AObj**
(animation object: play state, current frame, end frame, loop / rewind) that owns a chain of **FObj** tracks,
one per animated property (eye x, roll, field of view, joint rotation y, ...). Every frame,
`HSD_AObjInterpretAnim` advances the AObj and each FObj evaluates its curve and writes the value into the object
through an update callback.

The tracks are stored compressed: a few bytes per key, with the interpolation type, value precision and timing
chosen per track by the exporter.

## Data structures

**FObj descriptor (on disc, 0x14 bytes):** next track, data length, start frame (f32), track type (u8: which
property; meaning depends on the object kind, e.g. camera 1/2/3 = eye x/y/z, 9 = roll, 10 = fov), value format
(u8), slope format (u8), data pointer.

**Value formats (`frac_value` / `frac_slope`):** high 3 bits = storage (0 float, 0x20 s16, 0x40 u16, 0x60 s8,
0x80 u8), low 5 bits = fixed-point shift (value = stored / 2^shift). Multi-byte values are **little-endian**
inside an otherwise big-endian file (the parser assembles them byte by byte).

**Key stream:** groups of keys sharing one interpolation op. A group starts with a byte whose low 4 bits are the
op and whose bits 4-6 (+1) are the key count, extended by 7 bits per extra byte while bit 7 is set. Each key
then holds its values and a wait (frames to the next key, 7 bits per byte, low bits first):

| Op | Name | Key data | Meaning |
|---|---|---|---|
| 1 | CON | value, wait | constant: jump to the value and hold |
| 2 | LIN | value, wait | linear interpolation to the next key |
| 3 | SPL0 | value, wait | Hermite spline, outgoing slope 0 |
| 4 | SPL | value, slope, wait | Hermite spline with explicit slope |
| 5 | SLP | slope | sets the next key's incoming slope; no wait, the next key follows immediately |
| 6 | KEY | value, wait | key value (held; used for discrete properties) |

A track's clock starts at its start frame: a key reached after waits totalling W plays on animation frame
W - start frame (`HSD_FObjReqAnim`: time = startframe + requested frame).

**AObj:** flags (loop, no-update, first-play, rewound, stopped), current frame, rewind frame, end frame,
frame rate, FObj chain, the object it animates.

**Room camera file (`.cam`):** HSD archive, public symbol `scene_data` -> scene descriptor; +0x04 -> camera
entry {CObjDesc, list of camera animations}. CObjDesc: projection (perspective / frustum / orthographic), eye and
interest position objects (WObj: a position, optionally animated), roll, near, far, fov, aspect. Each camera
animation has an AObj for the camera itself (roll, fov, near, far, eye / interest components) and one each for
the eye and interest WObjs (x, y, z, or `path` = position along a spline, 0..1).

## Flow

1. Load: `floorReadCameraPostFunc` (0x8011487C) parses the archive and builds the camera; `HSD_AObjLoadDesc`
   creates the AObj and `HSD_FObjLoadDesc` the track chain (the latter is Melee's recursive loader).
2. Play: the camera code requests the animation (`HSD_AObjReqAnim`), then each frame
   `HSD_AObjInterpretAnim` advances it; `HSD_ForeachAnim` applies one operation (rate, restart, stop) to every
   animation in a tree.
3. Each FObj keeps a cursor into its key stream and decodes keys lazily as time passes, so the uncompressed
   curve never exists in memory.

## Disc data

- 126 room camera shots in the map archives (type 12; the disc's other 34 type-12 entries are in menu archives such
  as the title, top menu and world map), 514 tracks (*verified* counts): mostly SPL (2,536 keys), then CON (646),
  SPL0 (533), SLP (117), LIN (11); no KEY in cameras. Values are often 16-bit fixed point (positions with 3-8
  fraction bits), slopes 8- or 16-bit.
- 358 tracks end exactly on the clip's last frame; 151 run exactly 10 frames past it (exporter padding, *guess*).

## Changing it

- New or edited camera paths: write tracks with float values (format 0) and SPL keys; the engine needs no change
  (*likely*, from the format; not tried in the game).
- A track decoder works unchanged for joint / material / light animations (same format, *verified* from the code),
  which is the basis for an animation viewer.
- What would break: changing a track's value format without re-encoding its keys; key counts above the pack
  info's range are fine (it is variable-length).

## Efficiency

- Keys are 2-5 bytes typically (fixed-point values, 1-byte waits) instead of 12+ bytes for float triples (*verified*).
- Decoding is incremental: each track reads only the next key when the current interval ends, so a 500-frame
  camera costs a handful of byte reads per frame (*verified* from the code).
- Spline evaluation uses the two stored values and slopes of the current interval only.

## Open questions

- Where the spline used by `path` tracks is stored (a spline object in the archive referenced by the WObj
  descriptor; not decoded yet).
- Which game code picks which camera shot (cutscene scripts vs room logic).
- The +10-frame overrun on 151 tracks: exporter behaviour or deliberate?
