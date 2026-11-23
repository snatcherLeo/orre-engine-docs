# HSD models: reading the geometry (maps, objects, NPCs, battle models)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16): static geometry in the bind pose, materials' base colour and first texture
are decoded for every model on the disc; animation (its key format: animation-tracks.md), lighting and the full material / texture-stage setup are not.
**How sure:** the layouts below come from the decompiled HSD headers and code and were checked by walking every
model on the disc with no errors (*verified*); remarks on purpose are marked *likely*.
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

**A stored 0 can be a real pointer** (**verified** on Trick's and Splash's effect models and in
the running game): whether a pointer field is set is decided by the **relocation table**, not by its value. The game
(`HSD_ArchiveParse`) adds the data block's address to every listed field, so a listed field that holds 0 points at the
**start of the data block**, and an unlisted field is NULL whatever it holds. Small effect models often put their root
joint first in the data block, so their model set's joint field reads 0 yet is listed: Trick's 1,099-byte model and
Splash's 979-byte model both do, and in the game both load a joint tree (the model's root joint is not NULL). A reader
that treats a stored 0 as "no joint" loses those joints, and with them everything attached to them (below and
models-animation.md (not yet published), "Effect models whose joints carry no mesh").

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
joint (rigid with a shared joint) or the envelope list (each envelope = {joint, f32 weight} pairs until a null joint),
or, for shape animation, a shape set (below).

**Shape animation** (polygon-object flags bit 12 = 0x1000; **verified** from `drawShapeAnim`,
`src/hsd/pobj.c`, and a frame capture): a mesh that morphs between whole stored shapes. The polygon object's +0x14 is
a **shape set** (`HSD_ShapeSetDesc`, 0x1C bytes): u16 flags, u16 shape count, s32 position-index count, the position
attribute description, a list of the shapes' position arrays, s32 normal-index count, the normal description, and a
list of the shapes' normal arrays. Every shape is a full set of positions (and normals) over the same indices, which
the display list points into. Two blend modes:
- **average** (flags 1): one blend value *b*; shape ⌊*b*⌋ is mixed toward shape ⌊*b*⌋+1 by the fraction (clamped);
- **additive** (flags 2): shape 0 plus, for each further shape, max(0, weight) × that shape; one weight per animation
  track type, starting at the first weight track type (2).
The weights come from animation tracks. **Where they live:** the model set's +0x0C is a null-terminated list of
shape-animation joint roots (one per animation), walked in step with the joint tree (child +0x00, next +0x04). Each
holds a list of shape-animation display-object records (+0x08; next +0x00, animation +0x04) beside the joint's
display objects, and each of those a list of shape animations (next +0x00, track description +0x04) beside that
display object's polygon objects. The blended positions are rebuilt on the CPU every draw. Two blends of 2 shapes
(flags 5) turn a sphere into a cube; Sharpen and Defense Curl share that model.

**A field is a pointer when the relocation table says so** (**verified** from the code and a run of
the PC port): a field holding 0 can still be a pointer. If the archive's relocation table lists it, after loading it
points at data offset 0 (often the root joint). So whether a field is a pointer is decided by the relocation table,
never by its value; a reader that treats 0 as "none" misses real joints.

**Path animation: a joint flying along a spline** (**verified** from `src/hsd/spline.c` and the
game's pictures): animation track type 4 (`HSD_A_J_PATH`) sets the animated joint's position to the point a given
fraction (clamped 0 to 1) of the way along a spline, **measured by arc length**, so the joint moves at an even speed
along the curve (`splArcLengthPoint`). The spline sits on the joint the track's description names, which carries the
spline flag (1 << 14) and keeps an **HSD spline** where other joints keep their display objects: u8 type (0 linear,
1 Bézier, 2 B-spline, 3 cardinal), s16 control-point count, f32 tension, the control points, f32 total length, each
segment's start, and five arc-length polynomial coefficients per segment (found by Simpson's rule and bisection).
In moves: Trick's carrier joint flies along a spline, and the sparkles linked to it stream from the user to the target;
Splash's carrier joint hops, and the Pokémon riding it hops with it.

**Parts hidden by a zero-scale rest pose** (**verified**): effect models often hide a part by
giving its joint a scale of 0 at rest and letting the animation grow it (Mind Reader's lock-on rings: joints 4-6 rest
at 0 and reach 0.662 by frame 6). The game draws a rigid part with its joint's **current** matrix
(`HSD_JObjSetupMatrix`), never through the rest pose, so this is harmless in the game. A tool that bakes vertices into
the rest pose and later poses them with "current × inverse(rest)" loses these parts, because a zero scale has no
inverse; keep such parts in their joint's own space. Found in 15 parts of 13 moves' effects: Twineedle, Bite,
Bonemerang (its returning bone), Mind Reader (its rings), Faint Attack, Rain Dance, Beat Up, Torment, Memento, Taunt,
Knock Off, Hydro Cannon and Block.

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
- **An example: Lunatone** (**verified** from logs of the running game): every body part is
  envelope-skinned, fully weighted to one skeleton joint (joint 7, which has an inverse-bind matrix), with each part's
  own joint as its node matrix. A vertex lands at joint 7's world matrix × joint 7's envelope matrix × the node matrix
  × the vertex; for a part directly under the skeleton root, the node matrix is the part joint's own world matrix.

**How a joint's matrix is built: HSD's scaling rule** (**verified** from the code,
`HSD_JObjMakeMatrix` in `jobj.c`, which calls `HSD_MtxSRT`). A joint's matrix is **not** simply parent × local:
- Each joint keeps an **accumulated scale**: its own scale times its parent's accumulated scale. A joint flagged
  `JOBJ_CLASSICAL_SCALE` keeps the parent's scale alone (or none, if the parent has none).
- When the parent's accumulated scale is **uneven** (different along x, y and z), the child's matrix is adjusted: in
  row *i*, column *j* is multiplied by the parent's scale on *j* divided by its scale on *i*. That cancels the
  parent's uneven scale wherever it would shear or stretch the child. This is HSD's default, "non-classical" scaling.
- A viewer that multiplies plain parent × local matrices gets such parts wrong. Crawdaunt shows it clearly: its
  pincers come out about twice as long and turned edge-on.

## Texture coordinates and texture stages (verified from `src/hsd/tobj.c` unless marked)

- **Texture coordinate transforms** (`MakeTextureMtx`): uv' = (uv - translate) x repeat / scale, with the GX_MIRROR
  shift on t. 1,951 meshes use a non-identity transform; none rotates its texture (checked on a textured render of
  Phenac City).
- **Scrolling textures** (water, waterfalls, clouds, glows; **verified** from `tobj.c` and per-step logs of recorded
  play): the texture's transform above, animated by its animation object, is loaded into a
  **post-transform texture matrix** (`GX_PTTEXMTX<n>`, the slot `HSD_TObjAssignResources` picks with
  `HSD_TexMapID2PTTexMtx`). `setupTextureCoordGen` sends the plain UV through the identity first and then through that
  post matrix, so the normal texture matrices (`GX_TEXMTX<n>`) stay at identity and **the scroll lives only in the post
  matrices**. In one recorded stretch of play (12,000 steps) about 22,000 of these matrices changed between steps, for
  example the waterfall in room 0x24. A scroll animation runs from 1 back to 0 at the wrap, which shows the same picture
  on a repeating texture. Not every moving texture is a scroll: Agate Village's band of leaf light across the ground
  doesn't go through these matrices.
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

- **Bump maps** (**verified** from the material flags; found comparing a viewer's picture with the
  game's own): Sharpedo's grainy skin is a bump map, on its second texture stage with the bump bit (24) set and a
  colour operation of "blend" at factor 0. So the stage adds **nothing to the colour**: it only changes how light
  hits the surface. A viewer without the game's lighting draws Sharpedo's skin smooth; the grain appears only once
  the lights are right.
- **How HSD draws a bump map: emboss** (**verified** from the code, `tobj.c`:
  `setupTextureCoordGenBump`, `SetupEmbossBumpTev`). A material has at most one bump stage, and it costs **two texture
  coordinates and two TEV stages**:
  - the second coordinate is a **bump coordinate** (`GX_TG_BUMPn`), where *n* is the first diffuse light the
    material uses: the first coordinate **shifted a little toward that light**, worked out by the GameCube's
    hardware from each vertex's normal, binormal and tangent;
  - the first TEV stage adds the height picture × the lit colour to the result so far (no clamp); the second
    subtracts the height picture × the lit colour **at the shifted coordinate** (clamped).

  So where the height picture slopes toward the light the surface gets brighter, and where it slopes away it gets
  darker: the classic **emboss** trick, a carved look with no extra geometry. That is where the grainy, carved look of
  these Pokémon in the game comes from, and why a viewer without bump maps and the game's lights draws them flat.
- **Which Pokémon use it** (**verified**, a survey of every battle model and form on the US disc,
  counting bump stages): **20 of 417 battle models**. Bump stages per model, with how many of its parts carry one:
  Sharpedo 20 (of 32 parts), Onix 17 (of 24), Groudon 14 (of 35), Corphish 13 (of 48), Regirock 12 (of 20), Rayquaza 7
  (of 40), Claydol 7 (of 16), Kyogre 6 (of 25), Shelgon 5 (of 18), Huntail 4 (of 29), Registeel 3 (of 20), Lunatone 3
  (of 10), the Egg 3 (all 3 parts); Seedot, Deoxys and Camerupt 2 each; Torkoal, Solrock, Shedinja and Metagross 1
  each. Mostly rock, metal and scaly skins; Regice has none. Trainer and world models weren't surveyed.

## How Pokémon battle models are drawn: blending, depth and faces 

From a survey of every species' normal and shiny battle model on the US disc (772 model reads, 11,937 visible mesh
parts), read by an independent model reader, and from the pixel-engine setup code (`HSD_SetupPEMode`,
`src/hsd/hsd_texp.c`).

**The default pixel-engine rules** (**verified** from the code): a material with no pixel-engine description of its own
(`HSD_PEDesc`) has **no alpha test** when it is opaque. The material's render mode then decides the rest:
- **translucent** (bit 30, `RENDER_XLU`): blended by its alpha (source alpha / one minus source alpha), and, while depth
  writes are on, an alpha test that keeps only pixels with alpha above 0;
- **no depth update** (bit 29, `RENDER_NO_ZUPDATE`): depth writes off;
- **depth always** (bit 27, `RENDER_ZMODE_ALWAYS`): the depth test always passes; otherwise "less or equal".

**What the Pokémon models actually use** (**verified**, the survey):
- **90% of parts** are one texture stage "uv coordinates, diffuse channel, blend factor 1": the texture is simply the
  diffuse colour before lighting.
- **94% of stages clamp** their texture coordinates (no repeat).
- **Faces:** 93% of parts hide back faces, 7% are double-sided, 10 parts hide front faces. With the display lists'
  triangle order, **clockwise is the front**. A viewer that draws every face shows things it shouldn't: Venusaur seemed
  to have an "extra eye" that was really **the inside of its head** seen through, gone once back faces were hidden
  (**verified** by eye).
- **Only 190 parts have their own pixel-engine description:** 168 are alpha-blended with a near-hard cut-out (an alpha
  test keeping only alpha above 252) and depth writes on, used for Venusaur's and Ivysaur's leaves and petals; 18 are
  **additive** (no depth write) for glows.
- **About 700 reflection stages** (coordinate source 1), mostly on the "ext" lighting channel with an "add" colour
  operation; about 370 stages carry a TEV override, mostly the reflection ones; a few models use specular-channel stages
  (Gyarados, Sharpedo, Haunter).
- Almost every part uses the same render mode (0x2015); 62 parts have vertex colours.
- **How lighting reaches a material** (**verified** from the code): the material's diffuse, alpha and
  specular colours enter the TEV stages as **constants**, so the lit colour channels carry only the light (the
  material colour register is white); the lighting is per vertex, as HSD sets up the GameCube's colour channels.
  Battle lights come from the **arena map's light list**. And there is **no toon shading** in Colosseum: loading a
  material sets the toon flag on every one, but no toon ramp texture is ever assigned, so it does nothing. A viewer
  built this way comes within a few colour steps of the game's own pictures (**likely**, five species compared).
- **Glow cards** (**verified** from the disc): a flat part on a **billboard joint** (one that always
  faces the camera; joint flags 0xE00 or 0x2000) whose material is translucent with no depth update. Five species
  have them: Gastly 2 (its purple gas haze), Chinchou 2 and Lanturn 2 (the light round the lures), Espeon 1 (the
  sparkle on its forehead gem) and Wailmer 1 (not visible on its idle's first frame). They look like particles but are
  part of the model. Thirteen other species have billboard parts that are ordinary parts.
- **No species uses a fixed particle joint** (**verified**): every Pokémon's own effect starts from
  a key in one of its animations (models and animation, "particle keys").

**The unusual ones** (**verified**, the same survey):
- **A fourth texture-coordinate set** (`GX_TG_TEX3`): Spheal, Spinda, Wailord, Rayquaza and Deoxys. Spinda and Deoxys
  also have a **fifth texture stage, and it reads a fifth coordinate set** (`GX_TG_TEX4`, vertex attribute 17;
  **verified** from the disc), no other species:
  - **Spinda's mouth:** the head mesh has five stages; the fifth draws the mouth strip (a 64 × 320 texture holding
    five mouth pictures) with ordinary texture coordinates from TEX4.
  - **Deoxys:** one mesh has five stages; the fifth names TEX4 but is a **reflection** stage (its coordinates are made
    from the surface normal, like its fourth stage), so the stored TEX4 coordinates likely don't change its look.

  So battle models use **up to five** coordinate sets (TEX0 to TEX4), not four. A viewer that stops at TEX3 draws
  Spinda with no mouth and smears the strip across its ears; Deoxys looks the same either way.
- **Legs solved by inverse kinematics (IK):** only Ralts, Sableye and Exploud. Each leg is a chain of a first joint, a
  second joint and an end point (`JOBJ_JOINT1` → `JOINT2` → `EFFECTOR`), with IK hints (bone lengths in the joint's
  local units) and other joints as the target and the pole (`REFTYPE_JOBJ`). A
  viewer has to run HSD's two-joint solver (`resolveIKJoint1` / `resolveIKJoint2`) to place those legs; Exploud's chains
  solve exactly to its standing pose.
  - **How the solver sizes the two bones** (**verified** from the code, `resolveIKJoint1`, and
    from Sableye matching the game joint for joint): the first bone is its hint length × the first joint's
    accumulated scale; the second bone is the second joint's hint length × **its own local scale** × the first joint's
    accumulated scale. Using the second joint's accumulated scale instead counts the first joint's scale twice and
    draws the leg too straight.
  - **In the move viewer** (**verified** from logs of the running game): the solver runs at frame 0 with targets that
    already sit where the animation puts the end points (each end point's position is set to its target's), so a
    correct solve barely moves the leg; Sableye's front leg at idle frame 0 is the animation's own pose. Ralts too: all
    77 of its joints at frame 0, the IK chains included, match the game's own log to the printed precision
    (**verified**).
- **Specular-only materials:** Grumpig's pearls have no texture; their shine is the material's specular colour
  (`HSD_Material` +0x08) and shininess (+0x10), drawn through `RENDER_SPECULAR`.

So a faithful viewer needs: back-face hiding with the right winding, the alpha cut-out for leaf and petal parts,
additive blending for glows, and the reflection stages added on top; the rest is a plain textured, lit mesh.

## On the disc (USA, all checked 2026-10-16)

1,804 models walk with no truncated display list and no unknown primitive: 178 maps, 1,099 objects / NPCs (the disc's 1,100
type-2 entries are 1,099 distinct ones: `people_archive` lists `agent_m_a2` twice, **verified**; 44 have no
geometry: helpers), 527 PKX models; 38,704 polygon objects (13,384 envelope, 0 shape animation in these; the models
inside move effect files were not part of this count, and Surf's water is a shape animation), 3.4 million
triangles. Rendered checks (the decoded geometry converted to OBJ and viewed, *verified* by eye): Phenac City (round plaza, Colosseum dome, sky dome),
a Cipher Peon, Wes fully textured, Jirachi, Pikachu, all as expected. Several Pokémon carry untextured 12-triangle
boxes on hidden joints.

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

- Spline joints are skipped by the extraction tool's geometry reader (their format and use are now understood, "Path
  animation" above). Shape-animation polygon objects are now understood (above); none occur in
  the maps, objects or Pokémon models. In move effects (*the complete list, 2026-11-20*, from a scan of all moves'
  parts): Surf, Muddy Water, the damage half of Sky Uppercut, and the third parts of Sharpen, Milk Drink and Defense
  Curl. Mist, Haze, Waterfall and Crabhammer have none.
