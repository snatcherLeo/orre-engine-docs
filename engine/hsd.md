# HSD / sysdolphin (scene graph and rendering library)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-15): library role, class system, archives, the description -> object pattern, start-up,
the render frame, render passes and the port consequences are described; per-class field detail is still thin.
Code: `src/hsd/` (all but one of its 524 functions matched; the last is an assembly wrapper). Reference: the Melee decompilation (doldecomp/melee,
`src/sysdolphin/baselib`; Melee ships an older version of the same library). Graphics built on
top of it: [gfx.md](gfx.md). Claims are labelled as in the README.

## What it does

HAL Laboratory's "sysdolphin" library (HSD), also used by Super Smash Bros. Melee. Colosseum uses a **later
revision (1.3.0.0 or newer)**: the initialise code reports two parameters as "obsolete since 1.3.0.0" (verified,
`initialize.c`). It gives the game:

- a **class system** in C: every runtime object (joint, mesh, material, texture, light, camera, animation, ...) is an
  instance of a class with a virtual-function table, allocated from per-size pools;
- a **scene graph** of joints (`JObj`) carrying meshes (`DObj`) made of materials (`MObj`) with textures (`TObj`)
  and shapes (`PObj`), plus lights (`LObj`), cameras (`CObj`), fog, constraints (`RObj`) and splines;
- **keyframe animation** (`AObj` holding `FObj` tracks) for every object kind;
- a **TEV expression compiler** (`TExp`, `texp.c` / `texpdag.c`) that turns a material's colour / alpha equations
  into GX TEV stages;
- **archives** (`.dat`): relocatable binary blobs holding the "description" structs everything is built from;
- start-up of the GameCube hardware (XFBs, GX FIFO, VI, heap) and the render-pass frame bracket.

Genius Sonority's `GS*` engine sits on top: it loads models, cameras, lights and fog from HSD archives, draws through
HSD's joints and materials, and adds its own graphics state and command recorder ([gfx.md](gfx.md)).

**Why it is built this way (for an engine of your own):** artists' tools export *descriptions* (plain data:
`HSD_Joint`, `HSD_MaterialDesc`, ...) into an archive; at run time each description becomes an *object* of a class
named in the description (or a default class). Data stays data, behaviour lives in classes, and a game can subclass
any object kind (Melee's fighters do) without touching the loader. The cost is two parallel struct families
(description vs runtime object) and a pointer-heavy on-disc format.

## Data structures

### Class system (`hsd_class.h`, `class.c`, `hsd_class.c`; verified)

- `HSD_Class` is the base of every object: its first word points to the `HSD_ClassInfo`.
- `HSD_ClassInfo` = a header (`info_init`, library and class name, object and info sizes, parent / next / child
  links in the class tree, live and peak instance counts) plus the virtual functions `alloc`, `init`, `release`,
  `destroy`, `amnesia`. Each class extends the info with its own methods (`HSD_JObjInfo`: `load`, `make_mtx`,
  `make_pmtx`, `disp`, ...).
- `hsdNew(info)` allocates through `alloc` and runs `init`; `hsdDelete(obj)` runs `release` then `destroy`.
  `hsdSearchClassInfo(name)` finds a class by name: the loaders use it to honour the class name stored in a
  description (`jobj.c` `HSD_JObjLoadJoint`: falls back to the default JObj class when the name is NULL or unknown).
- `HSD_Obj` (`hsd_object.h`) adds a reference count; joints and their children are released when it drops to zero.
- Memory: `hsdAllocMemPiece` / `hsdFreeMemPiece` keep free lists per object size (`HSD_MemoryEntry`), and
  `HSD_ObjAllocData` pools (`objalloc.c`; a 12-byte record) hand out fixed-size objects
  with optional count / heap limits and peak statistics.

### Descriptions vs objects

| Description (disc) | Runtime object | What it is |
|---|---|---|
| `HSD_Joint` (0x40) | `HSD_JObj` (0x88) | a node: flags, transform (rotation, scale, position, optional matrix), first child, next sibling, mesh / spline / particle list, constraints |
| `HSD_DObjDesc` | `HSD_DObj` | one mesh entry of a joint: material + shape list, linked list per joint |
| `HSD_MaterialDesc`, `HSD_TObjDesc` | `HSD_MObj`, `HSD_TObj` | material (colours, render mode, TEV via TExp) and its textures |
| `HSD_PObjDesc` | `HSD_PObj` | shape: vertex descriptor, display list, rigid / skinned (envelope) / shape-blend variants |
| `HSD_LightDesc`, `HSD_CameraDesc`, `HSD_FogDesc` | `HSD_LObj`, `HSD_CObj`, `HSD_Fog` | lights, cameras (with position / interest `WObj`s), fog |
| `HSD_AObjDesc` + `HSD_FObjDesc` | `HSD_AObj` + `HSD_FObj` | animation: frame rate, end, loop, and one keyframe track per animated value |

`HSD_JObj` (verified layout, `hsd_jobj.h`): class header and ref count (+0), next sibling (+8), parent (+0xC),
first child (+0x10), flags (+0x14), mesh / spline / particle list (+0x18), rotation as Euler or quaternion
(+0x1C..+0x28, `JOBJ_USE_QUATERNION`), scale (+0x2C), translation (+0x38), the world matrix (+0x44, 3x4), optional
scale and envelope matrices, `AObj` (+0x7C), constraints `RObj` (+0x80), id (+0x84).

Key joint flags (verified names from Melee, used the same way in the decomp's code): `JOBJ_HIDDEN` (bit 4), `JOBJ_MTX_DIRTY`
(bit 6: the world matrix must be rebuilt before use), lighting / texgen / specular, `JOBJ_INSTANCE` (bit 12: shares
another joint's children), billboard kinds (bits 9-13), `JOBJ_SKELETON` / `JOBJ_SKELETON_ROOT` / `JOBJ_ENVELOPE_MODEL`
(skinned models), render-pass bits (opaque / texture-edge / translucent, below) and the root-level `JOBJ_ROOT_*`
summaries of what a subtree contains.

**Which joint flags the game's data uses (verified from data, 2026-10-15: every `scene_data` joint tree in the 1,735
archives on the disc, 87,549 joints).** (Scan scopes on this page, *likely*: 1,735 = every archive with a `scene_data`
root, including the models inside PKX files; 1,277 = the same without PKX; 1,079 = the map archives. The counts differ
for that reason.) This tells a port or tool which joint features matter:
- **Common:** skeleton bits 0-2 (1,380 archives), lighting (bit 7, 1,607), the per-joint render-pass bits 18-20
  (opaque 1,698 archives, translucent 394, texture edge 444; the decomp's header has no names for these, Melee's are
  `JOBJ_OPA` / `JOBJ_XLU` / `JOBJ_TEXEDGE` (likely)) with their root summaries 28-30, joint kind bits 21-22 (896),
  specular (bit 16, 425), texgen (bit 8, 250), hidden (bit 4, 671), classical scale (bit 3, 85).
- **Rare:** instances (bit 12, 62 archives), splines (bit 14, 41), particle joints (bit 5, 25) and billboards (about
  140 joints in about 40 archives, all kinds, the `JOBJ_PBILLBOARD` bit 13 included).
- **Never on the disc:** quaternion rotation (bit 17), IK flip (bit 15), the user / independent matrix bits (23-25)
  and the dirty flag (bit 6, which only the running game sets).

**Which mesh and texture formats the data uses (verified from data, same scan).** This is the list a PC graphics layer
must support for the game's models and maps:
- **Textures:** CMPR (GameCube block compression) is 89% of the 27,402 image descriptions. The rest are CI8 (1,111),
  I8 (620), RGB565 (381), RGBA8 (312), CI4 (246), RGB5A3 (173), I4 and IA8 (a few each); none use IA4 or CI14.
  1,028 are mip-mapped. Palettes are RGB565 (1,192) or RGB5A3 (165).
- **Shapes:** 24,095 rigid and 13,163 envelope-skinned. **No shape-animation (morph) mesh exists on the disc**, so
  HSD's CPU blending path is likely never used by the game's data.
- **Vertex attributes:** every position, normal, binormal / tangent and texture-coordinate array is 32-bit float
  (no quantized vertex data). Arrays are indexed with 8- or 16-bit indices. Colours are indexed arrays. Matrix
  indices (position / normal and texture matrix) are sent directly for skinned meshes. Up to six texture
  coordinate sets appear (TEX0-TEX5).

**Which animation features the data uses (verified from data, same scan; track meanings verified from `jobj.c`).**
- 1,777 model sets have joint animations, 811 have material / texture animations, and none have shape animations.
- **Joint tracks:** almost all are rotation, translation and scale. A few kinds are rare:
  - visibility of one joint (`NODE`, 1,920 tracks) or of its whole subtree (`BRANCH`, 207): shown above 0.5;
  - spline-path position (231);
  - type 40 (705): launches a particle effect through the game's callback (bank in the low 6 bits, effect id
    above);
  - type 42 (4): sets a particle target.
  - Type 41 (play a sound) and the `SETBYTE` / `SETFLOAT` user tracks never appear.
- **Material tracks:** ambient / diffuse / specular colour and alpha. **Texture tracks:** UV move / scale / rotate
  and the image-swap and palette-swap tracks.
- **Value storage:** only about 4% of tracks store 32-bit floats; the rest are fixed-point (u8 37%, s16 29%, u16 24%,
  s8 8%), so a port or tool must support every encoding.
- **Looping:** about 55% of joint animations and most material animations have the loop flag (`AOBJ_LOOP`).

**Which material and light features the data uses (verified from data, same scan; 29,429 materials).**
- **Lighting and colour:** half the materials use a constant colour and half use vertex colours (bits 0-1). 12,490
  are lit (diffuse, bit 2) and 1,008 have specular (bit 3). The alpha comes from the material (22,475) and / or the
  vertices (8,719).
- **Textures per material:** 25,378 have one, 3,093 none, 887 two, and very few have three to six.
- **Never used:** toon shading (bit 12), the shadow bit (26) and "always pass the Z test" (27).
- **Translucency:** 6,861 materials are translucent (bit 30) and 4,849 skip Z writes (bit 29).
- **Pixel-engine descriptions:** 2,636 materials have one. Their blend type is almost always normal blending (2,605);
  19 subtract and 12 have blending off.
- **Lights in `scene_data`:** one ambient light per scene (1,735), 4,648 infinite (directional) lights, 356 point
  lights and only 13 spot lights. Nearly all non-ambient lights light both diffuse and specular.

### Materials and textures (`hsd_mobj.h`, `hsd_tobj.h`)

`HSD_MObj` (verified layout): `rendermode` (+4), the texture list `tobj` (+8), colours `mat` (+0xC: ambient,
diffuse, specular, alpha, shininess), pixel-engine settings `pe` (+0x10: blend, alpha test, Z), animation (+0x14) and
the compiled TEV program (`tevdesc` / `texp`, +0x18 / +0x1C).

`rendermode` bits (values verified from the code that tests them; names from Melee, likely the same meaning):
bits 0-1 diffuse source (constant material colour, vertex colour, both), bit 2 diffuse lighting, bit 3 specular,
bits 4-11 one bit per texture in use, bit 12 toon shading, bits 13-14 alpha source, bit 26 shadow (receives the
projected shadow texture), bit 27 Z test always, bit 29 no Z update, bit 30 translucent (bits 29 + 30 = blending).

`HSD_TObj` (verified layout, 0xAC bytes): GX texture map id and texgen source assigned at setup, a texture-matrix
transform (rotation, scale, translation and the resulting matrix), wrap modes and repeat counts, `flags`, a blend
factor, filters, the image / palette / LOD descriptions (with tables for texture-swap animation) and an optional
per-texture TEV block. `flags` packs:
- the **coordinate kind** (bits 0-3): UV, reflection, highlight, shadow, toon, gradation, back-light. Colosseum's
  `HSD_TObjAssignResources` generates coordinates for kinds 1, 2, 3, 5 and 6 (Melee's for 1-3 only; verified,
  section "Colosseum's HSD vs Melee's");
- light-map use (bits 4-5: diffuse / specular);
- how the texture combines with the material: a **colour map** op (bits 16-19: alpha mask, RGB mask, blend,
  modulate, replace, pass, add, subtract) and an **alpha map** op (bits 20-23, same list), which `TObjMakeTExp` turns
  into TExp operations (names likely, from Melee).

The material's TExp program is built from the material colours plus each texture's colour / alpha map op, then
compiled to TEV stages (`texp.c`, `texpdag.c`); so artists describe "texture 0 modulates the diffuse colour, texture
1 adds a highlight" and the library works out the GX TEV setup.

### Shapes (`HSD_PObj`, `pobj.c`; verified)

A shape is one display list with its vertex-attribute list (`verts`: which attributes, their GX formats and arrays)
and `flags`; the type in bits 12-13 decides how its vertices are transformed:
- **skin / rigid** (type 0): the vertices follow one joint (`u.jobj`, or the owning joint) through one matrix;
- **shape animation** (type 1): `u.shape_set` blends between several vertex / normal sets (average or additive
  blending, `SHAPESET_*`); Colosseum keeps the shape animation's AObj on the PObj itself and drives it through the
  class's update method;
- **envelope** (type 2): `u.envelope_list` gives, per matrix slot, a list of (joint, weight) pairs; the setup code
  builds each blended matrix from the joints' envelope matrices and loads it into a GX matrix slot, and the display
  list's matrix-index attribute picks a slot per vertex (smooth skinning on GX's 10 matrix slots).

Colosseum additions (verified): flag 0x800 hides the shape, the library counts vertices and triangles as it draws
(`fn_801ACDAC`, statistics), and it invalidates the GX vertex cache after loading a shape. Normals get the same
matrix (or its inverse transpose) depending on the joint's lighting and texgen needs (`SETUP_NORMAL`,
`SETUP_REFLECTION`, `SETUP_HIGHLIGHT`).

### Lights, cameras, fog (`hsd_lobj.h`, `hsd_cobj.h`, `fog.c`)

- **`HSD_LObj`** (verified layout): flags (light kind and options), a priority, colour (and the colour last sent to
  GX, so unchanged lights are not re-uploaded), position and interest as `WObj`s (world points that can follow a
  joint or be animated), kind-specific parameters (point: reference brightness / distance and distance function;
  spot: cutoff and spot function; or raw GX attenuation coefficients), shininess, and **two GX light objects**: one
  for diffuse and one for specular, each with its own GX light id. Kind names (ambient, infinite, point, spot)
  follow Melee (likely).
  Light flags (verified from `lobj.c`, 2026-10-15): bits 0-1 the kind (0 ambient, 1 infinite, 2 point, 3 spot);
  bit 2 lights diffuse colour, bit 3 specular, bit 4 alpha (a light with none of the three is skipped); bit 5 hidden
  (the visibility animation track sets and clears it: on at a value of 0.5 or more); bit 6 "raw parameters" (the
  description gave GX attenuation coefficients directly, so animation tracks write those instead of brightness,
  distance and cutoff); bits 7 and 8 "diffuse / specular light object needs re-uploading" (set when colour,
  position or parameters change, cleared after the upload). The description's attenuation flag (1) selects raw
  coefficients over the point / spot parameter blocks. When a light is set up for a frame, the diffuse / alpha
  path is taken first and the specular path only otherwise (likely: one GX light id each; the specular object gets
  its own id elsewhere).
- **`HSD_CObj`** (verified layout): flags, viewport and scissor rectangles, eye and interest `WObj`s, roll or an up
  vector, near / far, and a projection that is perspective (fov, aspect), frustum or orthographic (top, bottom,
  left, right). The function that applies the viewport is replaceable; GSgfx installs its own (`fn_800D3EC4`,
  gfx.md).
  Camera flags (verified from `cobj.c`, 2026-10-15): bit 0 the camera uses an up vector instead of a roll angle
  (the description's up vector, or (0, 1, 0) if it has none); bit 1 the viewing matrix is never rebuilt from eye /
  interest (likely: a caller-supplied matrix); bit 30 the viewing matrix and bit 31 the projection matrix need
  rebuilding (set by any change, cleared on rebuild). Loading a description replaces every flag except those two
  dirty bits. Projection types: 1 perspective, 2 frustum, 3 orthographic.
- **Fog** (`fog.c`): HSD_FogSet takes a fog description (type, start, end, colour) and an optional range-adjustment
  table (GX's `GXFogAdjTable`: GX fog follows depth, and the table raises it towards the screen's sides so it
  follows distance from the eye); GSgfx's fog setter turns HSD fog on
  and off per scene (gfx.md, `fogActive`).

### Archives (`.dat`, `hsd_archive.h`, code in `hsd_wobj.c`; verified)

A 0x20-byte header (`file_size`, `data_size`, relocation / public / extern counts, version) followed by: the data
block, the relocation table (offsets of every pointer inside the data), the public table (offset + name-string
offset), the extern table, and the name strings. `HSD_ArchiveParse` checks the size, points into the loaded buffer
(nothing is copied) and **relocates in place**: for each relocation it adds the data block's address to the 32-bit
word at that offset (`Locate`). `HSD_ArchiveGetPublicAddress(archive, name)` returns a public symbol by name; the
game asks for named roots (a map's public table, a model's joint tree, its animations).

### Description formats (the model format inside archives; layouts verified from the headers our matched code uses)

All pointers below are 32-bit after relocation; numbers are big-endian (GameCube order) unless stated.

| Description | Size | Fields |
|---|---|---|
| `HSD_Joint` | 0x40 | class name, flags, first child, next sibling, mesh list (or spline / particle list), rotation (3 floats), scale (3), position (3), optional inverse-bind matrix, constraint descriptions |
| `HSD_DObjDesc` | 0x10 | class name, next mesh, material, shape list |
| `HSD_MObjDesc` | 0x18 | class name, render mode (bits in "Materials and textures"), first texture, colours (`HSD_Material`: ambient, diffuse, specular as RGBA bytes, alpha, shininess), render description, pixel-engine description |
| `HSD_PEDesc` | 0x0C | flags, alpha-test references, destination alpha, blend type and factors, logic op, Z compare, alpha-test compares and op (one byte each) |
| `HSD_TObjDesc` | 0x5C | class name, next texture, GX map id, texgen source, rotation / scale / translation of the texture matrix, wrap S / T, repeat S / T, blend flags (coordinate kind, colour / alpha map ops), blend factor, mag filter, image, palette, LOD, per-texture TEV |
| `HSD_ImageDesc` | 0x18 | pixel data, width, height, GX texture format, mip-map count, min / max LOD |
| `HSD_TlutDesc` | 0x10 | palette data, GX palette format, palette slot, entry count |
| `HSD_TexLODDesc` | 0x10 | min filter, LOD bias, bias clamp, edge LOD, max anisotropy |
| `HSD_PObjDesc` | 0x18 | class name, next shape, vertex-attribute list, flags (type in bits 12-13), display-list length in 32-byte blocks (verified: `n_display << 5` bytes), display list, then the joint / shape-set / envelope table by type |
| `HSD_VtxDescList` | 0x18 | GX attribute, how it is sent (direct / indexed 8 / 16), component count and type, fraction bits, stride, array pointer; a list ends with the null attribute (likely, GX convention) |
| `HSD_EnvelopeDesc` | 8 | joint, weight (arrays of these per matrix slot, NULL-terminated, likely) |
| `HSD_AObjDesc` | 0x10 | flags (loop etc.), end frame, first track (`HSD_FObjDesc`), object id |
| `HSD_FObjDesc` | 0x14 | next track, stream length, start frame, track type (which value it animates), value encoding, slope encoding, the keyframe stream |

**Instance joints (verified, code and data).** A joint with `JOBJ_INSTANCE` (bit 12) does not own its child pointer:
it refers to another part of the tree. Every loader and walker skips it (`jobj.c`, the `JObjForeachAnim` walker),
and map archives rely on that: a reader that follows an instance joint's child can loop forever.

**`scene_data` (verified from data and code, 2026-10-15).** +0 the model-set list, +4 one camera set (set in all 1,079 map archives;
the ones checked are perspective cameras with a 640 x 480 viewport), +8 the light list, +0xC a list whose first entry
is the fog description (set in 637 map archives). The map loader (`floorReadMapPostFunc`) registers each
part as a resource of the map: models, lights (loaded through `GSlightLoad`), the camera (turned into a game camera by
`fn_800D27FC`) and the fog. Entering the map makes the camera active (`cameraSetGScamera`) and hands the fog
description to `gfxSetFog`, which loads it into HSD and also uses its colour as the screen clear colour (gfx.md).

**Model sets and their animations (verified from data, 2026-10-15).** `scene_data` +0 is a null-terminated list of
model sets. A model set starts with the root joint (+0) and the null-terminated list of joint animations (+4, one
`HSD_AnimJoint` tree per animation); the next two words are material and shape animation lists (likely, from the game
code's own copy of the layout). An animation's length is the largest end frame among its `HSD_AObjDesc`s. Pokemon
models carry between 4 and 16 joint animations (8 is the most common). Which slot is which motion (idle, attack, damage, faint,
...) is chosen by index in the battle and field code; naming the slots is an open question below.

**Keyframe streams (verified, `fobj.c`).** A track is a byte stream, read byte by byte with the **lowest byte first**
(so it is little-endian even inside a big-endian archive). It is a sequence of packs:
1. An opcode byte: bits 0-3 the interpolation (1 constant, 2 linear, 3 spline with zero slope, 4 spline with a slope,
   5 "set the next slope only", 6 key / step), bits 4-6 the number of keys in the pack minus 1, bit 7 "more count
   bytes follow"; each extra byte adds 7 more bits of the count.
2. Per key: the value; for type 4 also the slope; then the frames to wait before the next key, as a variable-length
   number (7 bits per byte, low bits first, bit 7 = another byte follows). Type 5 carries only a slope and no wait.
- Value and slope encodings (`frac_value`, `frac_slope`): bits 5-7 the storage type (32-bit float, s16, u16, s8, u8),
  bits 0-4 the number of fraction bits, so a stored integer n means n / 2^bits. Small encodings make animation data
  compact; a modding tool must pick an encoding whose range fits the values.
- Splines are cubic Hermite curves between two keys with their slopes (`fn_801B2560`, likely Hermite from its
  parameters: 1 / frame count, time, both values, both slopes).

**One root per archive: `scene_data` (verified by a read-only scan of all 1,277 HSD archives on the disc: maps,
NPC models and a few others; Pokémon models use the separate PKX container).** Every archive exposes exactly one
public symbol, `scene_data`, a block of NULL-terminated pointer lists:
- +0x00 model sets (each one a joint tree with its animations; verified, `floorReadParticlePostFunc` and the
  floor resource iterator walk it);
- +0x04 the camera set (verified: the map loader reads it, see "`scene_data`" above; the map-entry function's own
  copy of the layout skips it);
- +0x08 lights (verified, the floor resource iterator registers each as a light resource);
- +0x0C the fog list (verified for maps, see above): present in 797 of these archives, empty in 480.
So a map room's archive carries its whole scene (geometry, animated objects, lights, camera, fog) under one name, and
the game registers each model and light as a numbered resource of the room (`floorReadMakeModelResID` /
`floorReadMakeLightResID`). No archive names a class in its joints, meshes or materials, so every object is built by the
library's default classes (verified: a walk of every model set's joint tree, 56,738 joints with 25,088 meshes and
materials, found no class name; each model set starts with its joint tree).

## Flow

**Start-up (verified, `initialize.c`).** The game sets parameters with `HSD_SetInitParameter` (`fn_8019C3C4`):
GSgfxInit asks for 2 XFBs (double buffering) and its render mode. `HSD_InitComponent` then: DVD init, XFBs and the
GX FIFO from the arena (newer library: `OSAllocFromArenaLo`), heap and memory callbacks (`HSD_OSInit`: either the
game's own allocator callbacks or HSD's default, an OS heap over the arena), `HSD_VIInit` (black screen, clear colour
black, clear z max, all updates on), GX init, the id table, object pools (`HSD_ObjInit`).

**Loading a model (verified for joints).** Archive parse -> `HSD_ArchiveGetPublicAddress` for the root `HSD_Joint`
-> `HSD_JObjLoadJoint`: picks the class from the description's class name, allocates the joint, loads its meshes,
constraints and animation descriptions, and recurses into children and siblings. Every other object kind loads the
same way from its own description.

**Each frame.** Animation (`HSD_AObjInterpretAnim` per AObj: advance by the rate, loop with `fmod` or stop, push
values through the FObj tracks; `HSD_ForeachAnim` applies one operation to every AObj of a tree), then matrices are
rebuilt lazily (only joints with `JOBJ_MTX_DIRTY`, when something reads or draws them), then display:
`HSD_StartRender(pass)` (`fn_8019C708`: sets the pixel / Z format for the render mode, applies pending vertex-cache /
texture invalidations) -> camera setup (`HSD_CObj`, its viewport function is replaceable: GSgfx installs
`fn_800D3EC4`, see gfx.md) -> joint display.

**Render passes (verified, `HSD_JObjDispDObj` = `fn_80197784`).** A joint's mesh is drawn in up to three passes,
opaque, texture-edge (alpha-tested) and translucent, selected by joint flags 0x40000 / 0x100000 / 0x80000
(`trsp_mask << 18`); texture-edge and translucent meshes are queued in z-lists and drawn sorted afterwards
(`fn_801974A8`). Per mesh: `MObj` setup (render mode, colours, textures, the TEV stages its `TExp` compiled to) then
`PObj` display (load the vertex descriptor and matrices: rigid, envelope-skinned or shape-blended, then call the
display list). Colosseum's library skips meshes whose PObj has flag 0x800 (hidden), Melee's doesn't.

### Animation walkers (`HSD_ForeachAnim`, aobj.c; matched 2026-10-14)

`HSD_ForeachAnim(obj, type, mask, func, argType, ...)` walks every animation (`AObj`) hanging off an
object tree and calls `func` on each. It is how the game says "set the speed / restart / stop every
animation of this model" in one call.
- `type` says what `obj` is (JObj, DObj, MObj, PObj, TObj, LObj, CObj, RObj, WObj, Fog).
- `mask` picks which kinds of AObj receive the call (one bit per object kind), so a caller can, say,
  touch only material animations.
- `argType` picks the callback's argument shape: 12 variants of (aobj [, obj [, type]] [, extra])
  where the extra argument is a float, pointer or integer taken from the varargs. **The order is
  Melee's `AObj_Arg_Type`** (A, AF, AV, AU, AO, AOF, AOV, AOU, AOT, AOTF, AOTV, AOTU), verified
  against the jump tables in the DOL.
- Per-kind walkers: `JObjForeachAnim` recurses through children (not into instances), the joint's
  mesh list (`DObjForeachAnim`: mesh, material, textures, PObjs) and its constraint `RObj`s;
  `LObjForeachAnim` / `CObjForeachAnim` also walk the light's / camera's position and interest
  `WObj`s. The small walkers (RObj, WObj, TObj, MObj, PObj, Fog) are inlined into their callers.
- Colosseum difference from Melee: the PObj walker visits every PObj in the list using an AObj
  pointer at PObj+0x18 (Melee only handles one shape-animation PObj through `u.shape_set`).

`HSD_AObjInterpretAnim` advances one AObj by its frame rate, loops it (`fmod` into
[rewind, end)) or stops it at the end, then pushes the new values to the object through the FObj
tracks. Two counters (`lbl_8047B38C` / `lbl_8047B390`) count running vs stopped animations per frame
(guess: debug statistics).

### Colosseum's HSD vs Melee's (found while matching, 2026-10-14)

Colosseum ships a later revision of the library. Differences confirmed against the target (verified):
- **Compiler settings (verified from the build's per-unit flags, which every HSD function matches with):** per file,
  and nearly all of HSD differs from the game's defaults. 35 of the 40 HSD units build at `-O1`. 34 of them also use
  deferred inlining, `-use_lmw_stmw on` and read-only pooled strings; `hsd_wobj.c` uses only `-O1 -use_lmw_stmw on`.
  The other five are three data-only units, where the flag does not matter, and two small code units (`hsd_object.c`
  and one more) that match with the project default (`-O4,p`); whether those two would also match at `-O1` is not
  tested. Compiler versions: GC/1.3.2 for most units; GC/2.5 for the TEV and render-state units (`state.c`, `tev.c`,
  `texp.c`, `texpdag.c`); GC/1.3 for `hsd_wobj.c`, `hsd_object.c` and that one other unit. So HAL's library was
  probably built separately from the game, with its own settings, and partly with a newer compiler (likely).
- **Meshes can be hidden:** `HSD_PObjDisp` returns early when PObj flag 0x800 is set (Melee has no such check).
- **PObj animation walk:** the ForeachAnim PObj walker visits every PObj in the list through an AObj at PObj+0x18.
- **Texture coordinates:** `HSD_TObjAssignResources` gives texture coordinates to coord types 1, 2, 3, 5 and 6
  (Melee 1-3); `TObjMakeTExp` passes `repeat` as a 0 / 1 flag.
- **Different source revision:** assert line numbers differ everywhere (jobj.h 0x25D vs Melee 564; tobj, JObjLoad,
  fobj); each file keeps its own `.sdata2` copies of its assert strings.
- **Start-up:** XFBs and FIFO from `OSAllocFromArenaLo`, `HSD_INIT_HEAP_MAX_NUM` / `AUDIO_HEAP_SIZE` obsolete, three
  new init parameters (heap size, arena range, memory callbacks), `HSD_StartRender` applies pending invalidations.
- Much of it is built at `-O1` (see the open question below); ForeachAnim walkers match at scoped -O1.
- `HSD_RObjGetGlobalPosition` returns its counter on a NULL list (same value, different code).

## Disc data

`.dat`-style archives inside the game's `.fsys` packages (`fsys-catalog.md`): models (PKX models wrap one,
`models-animation.md` (not yet published)), maps (`floorReadMapPostFunc` asks for a map's public table by name), cameras,
lights and fog for scenes (`object-pools.md`). Everything in the data block is big-endian and every
pointer is a 32-bit offset until `Locate` turns it into a 32-bit address.

## Changing it (port)

- **The central port issue is the description structs.** They are read straight out of relocated archive data, so
  they are big-endian with 32-bit pointers, while the same C structs on a 64-bit PC would have 8-byte pointers and
  little-endian fields. The port's data model (the Melee PC port's approach) keeps
  disc data as it is: description structs get the byte-swap marker (GCC `scalar_storage_order`) and 32-bit pointer
  slots that are resolved on use; runtime objects (`HSD_JObj` ...) are ordinary native structs. `Locate`'s in-place
  32-bit add stays valid because game memory is below 4 GB.
  *Done 2026-10-15 for every description HSD loads* (archive records, joints, meshes, materials, textures, palettes,
  shapes, vertex lists, envelopes, shape sets, animations, lights, cameras, fog, world objects, constraints), with
  no change to the GameCube build. How it works:
  - GCC applies the byte order only to the marked struct itself, not to an unmarked struct nested inside it, and it
    refuses to take the address of a swapped field. So embedded vectors, rectangles, the material colours and the
    attenuation block have marked twins, and code that needs a native object (a `Vec*` argument, the runtime
    `HSD_Material`) gets a field-by-field copy.
  - Arrays of disc pointers (envelope lists, shape-set index tables, texture-swap tables) become arrays of pointer
    slots.
  - Not marked, because they never come from the disc: the pixel-engine description (bytes only), the expression
    description (holds a code pointer; likely built by game code) and the TEV descriptions the TExp compiler builds.
  - Keyframe streams need nothing: they are read byte by byte (above).
  - Asserts print their condition as text, which sits in the binary's read-only data, so a rewritten condition must
    keep the original text (found when one did not: it showed up as a data mismatch, not a code one).
  - Verified in the PC port (the decompiled code running natively): a test loads real archives, relocates them and walks every joint / mesh / material / texture
    / shape / vertex-list description through the marked types; an independent reader of the raw bytes agrees on
    432 of 434 sampled archives (models, maps, effects; the other two differ only in float rounding of a checksum).
  - Still open: game code that builds descriptions at run time or keeps its own copy of these layouts, and display
    lists / vertex arrays, which stay big-endian for the graphics layer.
- **Class tables are code-filled data:** every class's `HSD_ClassInfo` (the method table; `lbl_8036C5F0` is the WObj
  class's, 0x48 bytes on the GameCube) lives in the data section and is filled at start-up by the class's
  `info_init` (verified, `WObjInfoInit`). It is mostly function pointers, so in the PC port it is larger and must be
  defined and declared with its C struct type; code must reach the parent class through `head.parent`, not raw
  offsets (fixed in `hsd_wobj.c`, byte-identical).
  *Done 2026-10-15:* the records the code does not define itself (the root class `hsdObj`, the WObj and DObj
  classes, and the game's material MObj class) plus `HSD_StateInvalidate`'s table of {state bit, callback} pairs now
  have typed PC definitions next to their GameCube bytes. Only `info_init` (or the pairs) are set in the original
  data, so the definitions are short. Without them a PC build stops at the first class initialisation: it calls
  `info_init` through big-endian bytes, and `hsdInitClassInfo` would then write past the 4-byte-slot table.
- **Disc data used in place:** some runtime fields point straight into archive data instead of copies. A particle
  joint's list (`jobj->u.ptcl = joint->u.ptcl`, verified, `jobj.c`) keeps the disc list, and its `data` words are
  not pointers but packed particle references (bank, offset, and bit 31 meaning "not launched yet", which the loader
  sets and the display code clears; verified, `jobj.c` / `displayfunc.c`). In the PC port such lists must keep their
  32-bit disc layout and be read as values (done: a marked particle-list type). Joint paths (splines: control points,
  segment lengths and arc-length polynomials) and vertex lists are also used in place, and the runtime shape set keeps
  pointing at the disc index tables.
- **HSD runs in the PC port (2026-10-15).** A standalone port test compiles all of `src/hsd`
  for the PC, initialises HSD through the game's own `HSD_InitComponent`, loads archives with the disc marks and
  draws them through Aurora. A Pokemon model (Pikachu) and a map (Outskirt Stand's room, with its own camera and
  lights) render correctly: verified by screenshot, and they look the same as in the game (checked in-game). What the PC needed:
  - the game's start-up order: `OSInitAlloc` before any allocation; HSD then owns the rest of the arena, so later
    loads go through HSD's heap; more exactly, the game's own heap (GSmem) takes a **fixed 14.5 MB from the bottom of the arena**
    (`GSmemInit(0, arenaLo, arenaLo + 0xE80000)` in `main.c`), the SDK heap (`OSInitAlloc`) gets the rest of the arena,
    and **HSD allocates from that SDK heap** (verified by reading the code; memory.md "Boot memory split");
  - C definitions for data that holds pointers (class-info tables, the state-callback table), not generated bytes;
  - one fix in HSD itself: the base object class gave its sizes as GameCube numbers instead of `sizeof`;
  - vertex arrays: Aurora wants each array's size and byte order. Disc arrays are big-endian, and the size comes from
    the highest index the meshes' display lists use (computed once at load);
  - an Aurora bug: setting the alpha channel's material / ambient colour also overwrote the RGB, which turned every
    lit material black.
  Display lists themselves need nothing: Aurora takes the disc's big-endian GX command lists as they are.
  *Animation too (2026-10-15, verified by screenshot):* a Pokemon's joint and material animations from its model
  set play through the matched code (keyframe streams, texture swaps such as closing the eyes) with no PC-specific
  change.
  *Every model and map runs (2026-10-15, verified):* all 385 Pokemon models and all 167 map room archives load,
  animate and draw in the PC port for several frames without a crash or assert (appearance not checked for each).
- Melee's PC port uses the same library, so its handling of descriptions, display lists (big-endian vertex data fed
  to Aurora's GX) and the class system is the reference.
- Subclassing: the class-name lookup in the loaders is the natural mod hook (a mod can register a class and name it
  in a description).
- `HSD_SaveContext` (crash-dump register snapshot) and other assembly come from the port's platform layer in the PC port.

## Efficiency

- Object pools per class / size (`HSD_ObjAllocData`, `hsdAllocMemPiece` free lists) with peak statistics: no general
  heap traffic per object, and the peaks tell the developers how big to make each pool.
- Archives are relocated in place after loading: no parsing or copying of the data block.
- World matrices are rebuilt lazily (`JOBJ_MTX_DIRTY`), only for joints that changed and are used.
- Only texture-edge and translucent meshes are depth-sorted (z-lists); opaque meshes draw in tree order.
- Material TEV setups are compiled once by the TExp compiler (simplification passes, then a DAG scheduler that packs
  the operations into as few TEV stages as possible, `texpdag.c`) rather than per draw.
- HSD's own GX state cache (`state.c`) skips redundant blend / Z / colour-update changes (GSgfx keeps a second one
  for its own draws, gfx.md).

## Open questions

- Field-by-field meaning of the description structs (`HSD_MaterialDesc`, `HSD_TObjDesc`, `HSD_PObjDesc`, ...).
- Animation slot names: which index the battle and field code asks for as idle, attack, damage, faint and so on
  (to find in the code; leftovers by watching them, with a list of per-model exceptions).
