# HSD models: reading the geometry (maps, objects, NPCs, battle models)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16): static geometry in the bind pose, materials' base colour and first texture
are decoded for every model on the disc; animation (its key format: animation-tracks.md), lighting and the full material / texture-stage setup are not.
**How sure:** the layouts below come from the decompiled HSD headers and code and were checked by walking every
model on the disc with no errors (*verified*); remarks on purpose are marked *likely* or *guess*.
Related: models-animation.md (not yet published) (PKX, animation slots), [hsd.md](hsd.md) (the HSD library),
[textures.md](textures.md).

## What it does

All 3D models in the game are HSD archives (`.dat`, HAL's sysdolphin format, shared with Melee): room maps
(resource type 1), objects and NPCs (type 2), and the model inside each PKX battle model. The game loads them through
the decompiled HSD code; this page is how to read their geometry without the game, for viewers, converters and
editors.

## Data structures (big-endian; offsets from the archive's data block)

**Archive:** 0x20-byte header (file size, data size, relocation count, root count, reference count), the data block,
the relocation table, the root table (offset + name). Every model on the disc has one root, `scene_data`: a list of
model sets, each with its root joint at +0.

**Joint** (`HSD_Joint`, `include/hsd/hsd_jobj.h`, 0x40 bytes): +0x04 flags, +0x08 child, +0x0C next sibling, +0x10
display-object list (or a spline / particle list, by flag), +0x14 rotation (X, Y, Z radians), +0x20 scale,
+0x2C translation, +0x38 inverse bind matrix (3x4, skinned skeletons). World matrix = parent x T x Rz x Ry x Rx x S.
Flags used here: 0x01 skeleton, 0x02 skeleton root, 0x04 envelope model, 0x10 hidden.

**Display object** (`HSD_DObjDesc`): +0x04 next, +0x08 material, +0x0C polygon-object list.
**Texture transform** (`MakeTextureMtx`, `src/hsd/tobj.c`): TObjDesc +0x10 rotation, +0x1C scale, +0x28 translation,
+0x38 wrap T, +0x3C / +0x3D repeat S / T; the texture matrix is scale(repeat / scale) x rotation x translate(-translation,
minus 1 / (repeat_t / scale_t) on t for mirrored wrap).
**Material** (`HSD_MObjDesc`): +0x04 render mode, +0x08 texture list, +0x0C colours (ambient, diffuse, specular RGBA,
f32 alpha, f32 shininess). **Texture** (`HSD_TObjDesc`): +0x04 next, ..., +0x4C image description {pixels, u16 width,
u16 height, u32 GX format, ...}, +0x50 palette description.

**Polygon object** (`HSD_PObjDesc`): +0x04 next, +0x08 vertex-attribute list, +0x0C u16 flags (bits 12-13: 0 = rigid,
1 = shape animation, 2 = envelope), +0x0E u16 display-list length in 32-byte units, +0x10 display list, +0x14 the
joint (rigid with a shared joint) or the envelope list (each envelope = {joint, f32 weight} pairs until a null joint).

**Vertex attributes** (`HSD_VtxDescList`, 0x18 bytes each, until attribute 0xFF): GX attribute, type (1 direct, 2
8-bit index, 3 16-bit index), component count, component type, fraction bits, u16 stride, array pointer. Attributes
0-8 (position / normal matrix index and the eight texture-matrix indices) are always one direct byte.

**Display list:** GX primitives: opcode (0x90 triangles, 0x98 strip, 0xA0 fan, 0x80 quads; low bits = vertex format),
u16 vertex count, then each vertex's attributes in list order; a zero opcode ends it.

**Texture animations** (2026-10-16): each model set's +0x08 is a null-terminated list of material-animation trees
(`HSD_MatAnimJoint` {child, next, matanim} -> `HSD_MatAnim` {next, aobjdesc, texanim, renderanim} -> `HSD_TexAnim`
{next, id, aobjdesc, image table, palette table, u16 image count, u16 palette count}); the image tables hold the frames
(e.g. the TV-news screens in the houses). With
materials, extra stages and these frames, a reader reaches 32,127 of the 32,131 image descriptors anything points at
in all 1,804 models (*verified*); the other 4 lie inside other structures (*likely* not textures).

## Where vertices sit (verified from code: `SetupEnvelopeModelMtx`, `src/hsd/pobj.c`; `_HSD_mkEnvelopeModelNodeMtx`, `src/hsd/displayfunc.c`)

- **Rigid** polygon objects: in their joint's space (world = joint's world matrix x vertex).
- **Envelope** polygon objects: each vertex's matrix index / 3 picks an envelope. If the owning joint is a skeleton
  root, a one-joint envelope uses that joint's world matrix and a blended envelope is already in model space. Otherwise
  every vertex goes through one "node" matrix: the inverse of the owner's inverse-bind matrix when the owner is the
  skeleton itself, or inverse(skeleton root's world) x owner's world when the nearest skeleton is a root. (At the bind
  pose every joint's world x inverse-bind is the identity, which is what makes these simplifications exact.)
- Getting this wrong is very visible: before following the code, Pokémon came out with spiky stretched parts.

## Texture coordinates and texture stages (verified from `src/hsd/tobj.c` unless marked)

- **Texture coordinate transforms** (`MakeTextureMtx`): uv' = (uv - translate) x repeat / scale, with the GX_MIRROR
  shift on t. 1,951 meshes use a non-identity transform; none rotates its texture (checked on a textured render of
  Phenac City).
- **Several texture stages per material:** 135 models use more than one, mostly texgen source 5, reflection-style
  mapping (*likely*).
- **How a stage combines** (the TObjDesc +0x40 flags, `include/hsd/hsd_tobj.h`, the same layout as Melee's HSD): bits
  0-3 the coordinate source (uv, reflection, highlight, shadow, toon, gradation, backlight), bits 4-8 the lighting
  channel the texture maps (diffuse, specular, ambient, ext, shadow), bits 16-19 the colour operation (alpha mask, rgb
  mask, blend by the stage's +0x44 factor, modulate, replace, pass, add, sub), bits 20-23 the alpha operation, bit 24
  bump. On a sample (25 Phenac City models, 10 Pokémon) the first stage is almost always "uv, diffuse, modulate" or
  "blend"; the Pokémon add a "highlight, ext, modulate" stage (a specular-highlight map, *likely*).
- **The TEV stage** (`TObjMakeTExp`): each texture is one TEV stage (out = d + (1 - c) x a + c x b), chained on the
  result so far ("prev") of the lighting channel it maps (bits 4-8). Colour operations, with T = the texture colour,
  Ta = its alpha, k = the stage's blending factor (+0x44): alpha mask: prev x (1 - Ta) + T x Ta; rgb mask:
  prev x (1 - T) + T x T; blend: prev x (1 - k) + T x k; modulate: prev x T; replace: T; none / pass: prev;
  add: prev + T; sub: prev - T (clamped). The alpha operations are the same with alphas, and are skipped when an
  earlier stage already mapped the same lighting channel.
- **TEV overrides** (TObjDesc +0x58, `HSD_TObjTevDesc`: colour / alpha op, bias, scale, clamp, the four inputs a-d of
  each, a constant and two TEV colours): on a sample of 96 models (every 20th map model, every 10th Pokémon), 147
  material stages carry one, for example the Mt. Battle prize room's map (*verified*). When a stage has an active
  override, `MakeColorGenTExp` first builds a stage from the override's own inputs, op, bias, scale and clamp, and its
  output replaces T / Ta in the formulas above.

So a viewer can reproduce a material by applying the stages in order to the lit diffuse colour.

## On the disc (USA, all checked 2026-10-16)

1,804 models walk with no truncated display list and no unknown primitive: 178 maps, 1,099 objects / NPCs (the disc's 1,100
type-2 entries are 1,099 distinct ones: `people_archive` lists `agent_m_a2` twice, **verified**; 44 have no
geometry: helpers), 527 PKX models; 38,704 polygon objects (13,384 envelope, 0 shape animation), 3.4 million
triangles. Rendered checks (the decoded geometry converted to OBJ and viewed, *verified* by eye): Phenac City (round plaza, Colosseum dome, sky dome),
a Cipher Peon, Wes fully textured, Jirachi, Pikachu, all as expected. Several Pokémon carry untextured 12-triangle
boxes on hidden joints (*guess:* helper volumes for shadows or hit tests).

## Disc data and tools

The models are HSD archives (`.dat` inside the room, NPC, trainer and Pokémon archives). A reader that follows the
layout above can turn each one into a viewable mesh (visible meshes in the bind pose with normals, UVs, base colour and
first texture; meshes on hidden joints kept apart) (*verified*: every model on the disc converts).

## Changing it (for mods)

A converted mesh is for viewing and reference; editing models for the game still needs an HSD writer or the Blender importer / exporter route. Facts a writer must keep: 32-byte display-list chunks, attribute
lists ending with 0xFF, envelope matrix indices = envelope number x 3, the skeleton flags that decide the vertex spaces.

## Efficiency

Indexed vertex arrays (8- or 16-bit indices) shared between polygon objects, compact component types (16-bit fixed
point with fraction bits for many positions and UVs), triangle strips, and envelopes limited to 10 matrices per polygon
object (the GX matrix memory) keep models small and fast to draw (*likely* the intent; the limits themselves
are *verified*).

## Open questions

- Shape-animation polygon objects (none on this disc) and spline joints are skipped.
