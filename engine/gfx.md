# Rendering front end (`GSgfx*` / `gfx*`)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-15). The global state struct is complete (every field the code touches is named in
the decomp's GSgfx header); the frame flow, the command recorder and the state cache are described below.
The layer Genius Sonority built over HSD (`hsd.md`) and the Dolphin SDK's GX. For the port this is one
of the two natural boundaries for replacing graphics (the other is HSD's GX calls themselves).

## What it does

Owns the global graphics state, the request buffer that records drawing commands during the frame, the
display-list capture, the matrix / projection / light pools, the video mode and frame pacing, and a cache of GX
state (`gfxSet*`) so redundant GX calls are skipped.

The central design idea: **game code never talks to GX directly during the frame.** While `recordMode` is 1 every
GS setter and draw call appends a command to the request buffer (`gfxRecordCommand`, a command id plus its
arguments as 32-bit words). The frame loop (`GSgfxDrawScene` (0x800D3FA4)) switches to `recordMode` 2 and plays the buffer back,
once per render pass (opaque 0x10, then 0x1000, then translucent 0x2000) and per layer. That gives sorting by pass
for free and lets the same recorded frame be drawn twice (two layers). For an engine of your own: this is a
command buffer with replay, the same idea modern renderers use, built in 2003 to fit GX.

## Data structures

### `GSgfxState` (behind `lbl_8047AA80`, 0x5A0 bytes on the GameCube)

Allocated by `GSgfxInit` from the scratch memory (`GSscratchAlloc(3, notify)`) or,
when no scratch is free, the tool heap (`_toolentryAlloc__FUl(sizeof(GSgfxState))`). The 3 is
a **block count**, not a slot number: the scratch allocator hands out 0x200-byte blocks of the 16 KB locked cache, so
the state gets three blocks (0x600 bytes for its 0x5A0), the first fit; when the scratch is taken back, the notify
callback copies the state to a static backup and back again (**verified** from `GSscratchAlloc` and
`_gfxScratchNotify`). GSgfx is the only user of the scratch allocator in the retail code (**likely**: the only call). The GameCube offsets are
checked at compile time (size 0x5A0 and key fields) under `#ifndef TARGET_PC`.

| Offset | Field | Meaning |
|---|---|---|
| 0x000 | `recordMode` | 1 record commands into the request buffer, 2 execute directly (frame playback, init) |
| 0x004 | `drawMask` | pass being played back: 0x10 opaque, 0x1000, 0x2000 translucent, -1 outside passes |
| 0x008 | `drawEnable` | pass the current blend mode draws in (`gfxSetBlendMode`: 0x10 off, 0x2000 on) |
| 0x00C | `frameTarget` | texture the frame renders to (NULL = screen); `GSGFX_NO_FRAME` (0xFEFEFEFE) between frames |
| 0x010 | `flags` | `gfxSetFlags` / `gfxClearFlags`; bits 4 normal, 1 colour, 2 texcoord select the default vertex format |
| 0x014 | `primType` | current primitive: 0 points, 1 lines, 2 line strip, 3 triangles, 4 triangle strip, 5 triangle fan, 6 quads (verified, the GX opcodes in `gs_gfx_dl.c`), 7 a rectangle: each pair of vertices (two opposite corners) is drawn as a quad (likely, from `gfxFlushPendingVertex` (0x800D6B00)) |
| 0x018 | `lineHalf` | primitive 7: the first corner of a pair has been stored and waits for the second (verified, `gfxFlushPendingVertex` (0x800D6B00)) |
| 0x019 | `eraseEnabled` | clear the frame to `eraseColor` (`HSD_SetEraseColor` + `HSD_EraseRect`) |
| 0x01A / 0x01B | `layerCur` / `layerDraw` | a draw happens only when they are equal; `layerDraw` is the render pass index (0 / 1) |
| 0x01C | `eraseColor` | `GXColor`; set by `fn_800D36B4` from floats, all zero turns `eraseEnabled` off |
| 0x020 | `defaultVtxDesc` | vertex descriptor built by `gfxBegin` from `flags` when none is set |
| 0x024 | `vtxDesc` | current vertex descriptor (`gfxSetVtxDesc`) |
| 0x028 | `boundTexture[8]` | texture loaded into each GX texture slot (`gfxLoadTexture` skips a rebind) |
| 0x048 | `retraceCallback` | called by the retrace handler `fn_800D3F5C` with `retraceCount` |
| 0x04C .. 0x05C | frame pacing | `retraceCount` (VI retraces), `lastRetrace`, `frameRetraces` (retraces the last frame took; `GSgetFrameStep` returns it, so text speed follows frame time), `minFrameRetraces` (wait until this many passed: 1 = 60 fps), `waitRetrace` |
| 0x05D | `fogActive` | `fn_800D9E4C`: HSD fog on / off |
| 0x060 | `tev` | the GX channel / texgen / TEV / indirect block (`GSgfxTevState`, see the header) |
| 0x414 | `dirty` | 1 channels, 2 texgens, 4 TEV: flushed to GX by `_gfxFlushTevState` (0x800D923C) |
| 0x418 .. 0x42D | GX pixel state cache | `dstAlpha`, `colorUpdate`, `alphaUpdate`, `zUpdate`, `zCompare`, `zFunc`, `zCompLoc`, `alphaCompare`, `dither`, `cullMode`, `clipMode` and one more field |
| 0x42E | `stageSrc[16]` | per TEV stage: colour / texcoord / texmap source and preset mode |
| 0x46E / 0x476 | `scissor[4]` / `viewport[4]` | inclusive corners x1, y1, x2, y2 (verified: `gfxSetScissor` (0x800D9D68) / `gfxSetViewport` (0x800D9C24) turn them into x, y, width = x2 - x1 + 1, height; `gfxSetViewport` also hands them to the active camera) |
| 0x47E .. 0x488 | display-list capture | `captureActive`, `captureDL`, `captureCursor`, `capturePrimType` |
| 0x48C .. 0x498 | request buffer | `requestBufferHandle`, `requestBuffer` (start), `requestBufferPos` (write position), `requestBufferSize` (GSgfxInit asks for 0x6DDD0 bytes) |
| 0x49C | `overlayColorUpdate` | debug-menu toggle (`fn_801334DC` -> `fn_800D4610`): lets the full-screen quad between the two layers draw colour (a dark 0xC4-alpha overlay) instead of only resetting depth (verified call chain; effect likely); menu label "draw after the Z clear, on / off" (verified) |
| 0x49D | `frameBegun` | the frame target was prepared this frame |
| 0x49F | `vtxPending` | a vertex's attributes are set but it is not yet sent to GX (verified: position setters set it, `gfxFlushPendingVertex` (0x800D6B00) emits and clears it) |
| 0x4A0 .. 0x59F | vertex emitters | `emitMtxIdx` + `mtxIdx`, `emitPos` + `pos`, `emitNrm` + `nrm`, `emitClr[2]` + `clr[2]`, `emitTex[8]` + `tex[8]`: for each attribute the function that writes it to the GX FIFO (0xCC008000) in the current format, and the last value in every format (bytes, shorts, floats) |

### Other blocks

- **Statistics** `lbl_804001F0` (22 words, zeroed at init): per-frame counters (GX state uploads, pass timings in
  ticks, peak request-buffer use at +0x28).
- **State backup** `lbl_80400248` (`GSgfxState`): `_gfxScratchNotify` copies the state there when scratch slot 3 is
  taken back and copies it into the slot again when it is returned; `lbl_8047AA80` follows the copy.
- **Vertex descriptors** (`GSVtxDesc`, 0x190 bytes on the GameCube; pool `lbl_8047AAAC`, count `lbl_8047AAB0`,
  allocated by `gfxVtxDescPoolInit`, 0x800D7B80). A descriptor says, for each of 14 GX vertex attributes (0 matrix index, 1 position,
  2 normal, 4-5 colours, 6-13 texture coordinates), whether it is present (`enabled`), how it is sent (`type`:
  index into none / direct / index8 / index16), its format (`count`, `compType`, `frac`) and, for indexed data,
  the array (`data`, `stride`). `gfxVtxDescAlloc` (0x800D7894) takes a free one and gives it the next of eight GX vertex-format
  slots (`vtxFmt`, round robin over `lbl_803144D0`); `gfxVtxDescSetAttr` (0x800D7868) fills one attribute; `gfxVtxDescClear` (0x800D7650) /
  `gfxVtxDescFree` (0x800D75F4) clear it; `gfxLoadVtxDesc` programs GX from it (GXSetVtxAttrFmt, GXSetVtxDesc, GXSetArray);
  `fn_800D7940` draws indexed vertices with it (8- or 16-bit indices per attribute `type`). Users outside GSgfx:
  fades, effects, collision debug drawing, floor and field objects. Field roles are likely
  (from the GX calls they feed).
- **Matrix stack** (`GSMtxStackEntry`, one 3x4 matrix): `gfxMatrixStackInit(count)` (0x800D83E4) allocates it; `lbl_8047AABC` first
  entry, `lbl_8047AAC4` last, `lbl_8047AAC0` current (the model-view loaded into GX position / normal slot 0 after
  each change). It starts at the last entry; push (`gfxPopMatrix`, 0x800D7E5C) moves up and refuses at the last entry ("matrix
  stack underflow"), the pops move down and refuse at the first. Likely, from the bounds checks and messages.
  The decompilation names this function `gfxPopMatrix`, which fits its "underflow" message, so the
  push / pop labels above are likely swapped: this one is the pop and the others are pushes (**likely**; the bounds
  checks are as described either way).
- **Render modes** (`GXRenderModeObj`s): video mode 1 -> `lbl_80312D30`, 2 or 3 -> `lbl_803130F0` (and flag
  `lbl_8047AA9C = 1`), 4 -> `lbl_80312F4C` (see `GSGFX_VMODE_*` in `gs_gfx.h`: NTSC, PAL50, PAL60, MPAL).

## Flow

`GSgfxInit(requestBufferSize, mtxStackCount, vtxDescCount, dlCaptureCount, videoMode, efbHeight)`:

1. `GSmathInit` (GQR setup for paired-single math), allocate the state and set every field (recordMode 2 during
   init, frameTarget `GSGFX_NO_FRAME`, capturePrimType -1, ...), clear the statistics.
2. Pick the render mode from `videoMode`; hand it to the HSD video layer (`HSD_SetInitParameter` (0x8019C3C4)), then
   `HSD_InitComponent` and `fn_800D37D4(videoMode, ...)` (VI / framebuffer setup).
3. Register three HSD hooks (verified from `src/hsd/video.c` and `cobj.c`): `fn_800D3F5C` as the post-retrace
   callback (`HSD_VISetUserPostRetraceCallback`: counts retraces, calls `retraceCallback`); `fn_800D3F50` as the GX
   draw-done callback (sets the flag `GSgfxRenderEnd` (0x800D3190) waits on, giving up after about 3 seconds of bus-clock ticks);
   `GSgfxSetViewportJitter` (0x800D3EC4) as the camera viewport function (`fn_80196C3C`): a game hook `lbl_8047AA88` if set, otherwise
   `GXSetViewportJitter(..., VIGetNextField)` for jittered (interlaced) cameras or `GXSetViewport`, the same as
   HSD's default.
4. Allocate the request buffer (`gfxAllocRequestBuffer(heapSize)`), the matrix stack (`gfxMatrixStackInit`), the vertex-descriptor
   pool (`gfxVtxDescPoolInit`), the display-list capture table (`fn_800DB890`), and the default vertex descriptor
   (`gfxVtxDescAlloc`). The two functions allocate `count` descriptors (`sizeof(GSVtxDesc)`) and capture slots (`sizeof(GSgfxDLCapture)`).
5. With recordMode 2: scissor and viewport to the full display, default GX state (`gfxSetUpdates(1,1,1)`,
   `gfxSetZMode(1, 2, 1)`, alpha compare, cull mode, the 16 TEV stages), then recordMode 1 and
   `GSgfxBackFBInit`.

**Immediate-mode vertices (verified, `gs_gfx_range_800D3FA4.c`):** `gfxBegin(primitive, count)` then, per vertex,
a position setter followed by the other attributes (colour, normal, texcoords). Each position setter first emits the
previous pending vertex (`gfxFlushPendingVertex` (0x800D6B00)), then stores the new position and marks it pending; `gfxEnd` emits the last
one. So the position comes first, the opposite of OpenGL's `glVertex`-last order. Each attribute keeps its last value
in every format and an emitter function chosen by the setter's format, so a vertex is one indirect call per attribute
straight into the GX FIFO. Primitive 7 (rectangle, likely) emits four vertices per two corners: the first corner is
stored (`lineHalf`), the second produces the two in-between corners by swapping x / y and texcoord components.
Draws outside the pass and layer being played back are filtered twice with the same test (`layerDraw == layerCur`
and `drawMask & drawEnable`): `gfxBegin` skips `GXBegin`, and the vertex emitter drops the vertices (verified,
`GSgfxEmitVertex`); only the vertex attributes the current descriptor enables are emitted. In the PC port (the decompiled code running natively) `gfxEnd` adds
the `GXEnd` Aurora needs under the same test (`GS_PC_GXEND`).

Each frame: game code records commands (recordMode 1). `GSgfxDrawScene` (0x800D3FA4) notes the request-buffer peak, switches to
recordMode 2, plays the buffer back per pass for layer 0, draws a full-screen quad (alpha 0xC4) between the layers,
plays it back for layer 1 (the quad resets depth to a fixed value so layer 1 draws over layer 0: likely, from
`gfxSetZMode(1, 7, 2)` with the colour update off unless the debug toggle `overlayColorUpdate` is set), restores the
mode and resets the write position.

Frame functions (verified from `gs_gfx_range_800D2B90.c`):

- `GSgfxRenderLayer(target, doUpdates)` (0x800D3410) **begins a frame** when none is open: `frameTarget = target` (NULL = the
  screen), prepares a render-to-texture target, stores `doUpdates` (`lbl_8047AA90`). With `doUpdates` it runs the
  per-frame updates with `frameRetraces` (render utilities `fn_800D13C4`, `fn_800DC874`, models `GSmodelApplyAll` (0x800E3928),
  particles `gsParticleUpdate`), each timed into the statistics block. Drawing to the screen, it then renders the
  model shadows (`modelShadowRender`), starts HSD's frame (`HSD_StartRender(0)` (0x8019C708)) and erases the frame to `eraseColor`
  when `eraseEnabled`; for a texture target it calls `HSD_StartRender(3)`.
  (**verified** from a per-copy log of the game's own calls in a replay): Each model shadow is drawn
  into the top-left corner of the frame buffer and copied out **with clear**, one pass per shadow-casting group, every
  frame. The shadow texture's size is set per scene (`HSD_ShadowSetSize` from `lbl_8047AB90` / `lbl_8047AB8C`): 384 x 384
  in Outskirt Stand's opening (two or three shadows a frame), **640 x 480, the whole frame**, in a Colosseum battle (two a
  frame). Because the clear wipes everything the copy covered, the order above matters: shadows first, then the erase
  and the scene; anything drawn to the screen before a shadow pass in the same frame would be lost.
  (**verified** from the code and GPU captures of Pyrite, where the texture is 640 x 480 and its copy
  is GX_CTF_R4, a 4-bit intensity): The pass first fills the whole texture white with a rectangle 1.2 times wider and
  taller than the light camera's view, bounded only by the viewport and the scissor, then draws the casters dark
  (`HSD_ShadowStartRender`; a debug flag adds a thin black frame two pixels in). Receivers multiply by the texture,
  so white means "lit". Every receiver pixel outside the light's view reads the texture's outermost texel, **likely**
  through clamp-to-edge addressing; so that texel must stay white, or the whole receiver beyond the shadow's area
  darkens. The ground of the streets, battles and shop floors are receivers.
- `GSgfxRenderEnd` (0x800D3190) **ends the frame**: GX flush and draw-done wait (about 3 seconds at most), HSD's frame end
  (`fn_8019C6FC`), copies a render-to-texture target out, `frameTarget = GSGFX_NO_FRAME`; then, if the updates are
  on, the end-of-frame updates (`fn_800D1070`, `GSlightUpdate`, model animation `GSmodelAdvanceAll(frameRetraces, 0)`, 0x800E3884)
  and, **once per retrace that passed**, one more model step `GSmodelAdvanceAll(1, 1)` and `gsParticleUpdateAll`. Like the
  particles, this keeps animation in real time when frames are slow.
- `GSgfxWaitRetrace(mode)` **paces the frame**: waits one VI retrace, measures `frameRetraces`; in mode 1 it keeps waiting
  until `minFrameRetraces` have passed (1 = 60 fps, 2 = 30 fps) and, if `waitRetrace` is off, reports exactly
  `minFrameRetraces` so the game advances by a fixed step; then `lastRetrace = retraceCount`.

### Display-list capture (verified, `gs_gfx_dl.c`)

GSgfx can record immediate-mode drawing into a GX display list and replay it later:
- `GSgfxDLBegin(desc, size)` takes a free slot in the capture table (`lbl_8047AAD4`), a 32-byte-aligned buffer of
  `size` bytes from the GS memory system, and turns capture on (`captureActive`, `captureDL`, `captureCursor`); it
  refuses while another capture is open or a vertex is pending.
- While capturing, `gfxBegin` writes the GX primitive header (opcode, vertex format, count; `fn_800DB758`) and each
  emitted vertex writes its attributes in their formats (`fn_800DB098`) into the buffer instead of the GX FIFO; it
  stops writing and marks the capture as overflowed when fewer than 0x68 bytes remain.
- `GSgfxDLEnd` pads the list with zero bytes (GX no-ops) to a 32-byte boundary, shrinks the buffer to the used size,
  flushes it from the CPU cache and returns the capture (or frees it on overflow). `GSgfxDLDraw` replays it in the
  current pass and layer (load the descriptor, set up channels / TEV, `GXCallDisplayList`), or records command 0x2A.
- The same file parses a GX display list back into per-vertex callbacks (begin / vertex attribute / end, `_dlParse*`),
  a reader for HSD's display lists. Its one user is the model-bounds code (`_modelBoundPObj`), which feeds it a
  mesh's display list and vertex list straight from the archive. In the PC port the list's counts and 16-bit indices are
  read big-endian (`GS_DL_U16`, 2026-10-15). The vertex values the callbacks receive are pointers into the disc's
  big-endian vertex arrays, so the callbacks must read them that way too.
- User found so far: the collision debug display builds its collision-mesh overlay as one captured list of 0x80000
  bytes (`gs_colsys_exact_8010D20C.c`).

### Back-framebuffer captures (verified, `gs_gfx_backfb.c`, `GSgfxBackFBDoFrame`)

Up to four textures can receive a copy of each rendered frame (`include/game/gs_gfx_backfb.h`):
`GSgfxBeginBackFBCapture(texture, callback, userData)` accepts only some texture formats (0x40-0x45 and 0x90, the
GameCube's copy formats, likely), sets the texture's wrap and filter, locks its image and takes a free slot. At
frame end (from the frame-begin / frame-end functions, when a frame is being prepared) `GSgfxBackFBDoFrame` converts
the copied frame into every active capture's texture and calls the callback with the texture, the number of frames
captured so far and the user data; a callback that returns 0 ends its capture, and `GSgfxEndBackFBCapture(texture)`
ends one directly (unlocks the image, frees the slot). This is how screen fades and transitions, some move effects
and the title screen get the previous frame as a texture (blurs, wipes, freeze frames: likely, from the callers).
In the PC port the slots hold pointers, so the slot table is larger and must be declared with its struct type.

### A frame with no display copy keeps its picture 

The GameCube draws into its embedded framebuffer (EFB) and shows a frame by **copying** the EFB to the display; that
copy is also what clears the EFB for the next frame. **A frame that ends without a display copy leaves the EFB as it
was**, so the next frame draws on top of the previous picture (**verified**: a PC fix and frame dumps, below).
The game relies on this during loads: while the battle room loads, the battle-start swirl keeps drawing over the
held picture and its captured background, with no new display copy in between. A renderer that clears every frame
shows black frames there and blends the swirl's ring over black (Willie's battle start in the PC port, fixed by keeping the
EFB into the next frame whenever a frame ends at the retrace without a copy).
- **It happens during play too, not only in loads** (**verified** in the PC port: logs plus frame dumps; that the console
  skips the very same frames is **likely**): the battle room's load (the swirl), the first wave frame of Confusion /
  Psychic and once mid-wave (the distortion copies the EFB to a texture and clears it), and now and then while a battle
  is idle at the command menu (three such frames in one run). On the console the TV keeps showing the
  previous picture (XFB) in those frames. With the EFB kept, the Confusion wave in the PC port starts with no black frame
  (**verified**, frame dumps); a frame-by-frame Dolphin capture of that moment would settle the console side.
- **Battles show 30 pictures a second** (**verified**, a PC per-frame log): in battle the game logic
  runs every frame (60 per second) but makes a display copy only every second frame, so each picture is held for two
  retraces.
- **Movies (THP) draw without HSD's display copy at all** (**verified**, PC log): the movie player puts its frames out
  by its own path.

### Menu backdrop: the frozen, dimmed scene behind a menu 

When a full-screen menu opens over the game (shops, the party screen from the field, battle menus that cover the
stage), a still picture of the scene's last frame, dimmed, is drawn behind it (verified from the code,
`menu_offscreen.c`; callers include the shop, `menuPokemon` and the battle menus). Whether the live scene also stops
drawing meanwhile wasn't read. The steps:
1. **Set-up, once** (`menuOffScreenInit`): one texture the size of the screen (asking for size 0 x 0 means "the
   framebuffer's size"), format 0x44 (RGB565: no alpha, half the memory of RGBA8).
2. **Opening** (`menuCreateOffScreen(time)`): register the texture as a back-framebuffer capture, **wait** (yielding
   the thread) until the capture callback reports that one frame has been copied into it, show it behind the menu, and
   start a fade from full brightness to 50 % over `time`. The callers divide a constant by `fn_800D37CC`, and the
   per-frame step divides the retraces that passed by the same value, so the fade likely lasts the same real time at
   30 or 60 fps.
3. **Every frame** (`menuOffScreenDraw` (0x801093C8), called at the start of the menu system's frame, `menuDaemon`): advance the fade by
   the retraces that passed, then draw the texture as one full-screen quad with no blending, its vertex colour set to
   the brightness (255 at 0 %, 127 at 50 %), so the GPU dims the picture for free. A priority flag picks whether the
   quad is drawn in front of or behind other layers.
4. **Closing** (`menuReleaseOffScreen(time)`): fade back to 0 %, wait for the fade to end, release the backdrop.

Why (likely): if the scene stops drawing, a menu over a frozen picture costs one textured quad per frame instead of
the whole 3D scene, so menus stay smooth on the GameCube. The same capture slots serve screen transitions and effects.

### Menu model pictures: a 3D character drawn into a texture 

Menus that show a 3D character next to a window (the name entry's player, the P★DA, the field TV) do not draw the
model into the scene. They render it into a texture and draw that texture as a flat picture (**verified** in the PC port
from logs and a readback of the texture, name entry, 2026-10-17):
1. **Set-up** (`menuModelInit`): a texture the size of the picture (the name entry's is 256 x 480, RGBA8, so the
   picture keeps an alpha channel around the character), a camera and up to three lights. Their start values come from
   a small table of floats (two white lights, a grey ambient of 80, a light position). The model is loaded hidden: it
   never appears in the normal 3D pass.
2. **Every frame** (`menuModelRender`): switch the frame target to the texture, set the viewport and scissor to its
   size, make the menu's camera active, switch on its lights, apply a pixel-engine descriptor to the model's materials
   (alpha blending; an alpha test that keeps only pixels with alpha above 0; depth test and depth writes on), draw the
   model, copy the result into the texture (clearing behind the copy), then restore the camera, viewport, lights and
   materials. HSD's framebuffer format is switched to RGBA6_Z24 for this pass so the copy carries alpha.
3. **Drawing the picture**: the window's draw callback (for the name entry `menuNameEntryBackDrawHumanModel`) draws
   the texture as a two-corner rectangle (primitive 7), blended by its alpha, so only the character covers what lies
   behind.

The name entry shows a mode-2 window (naming a Pokémon) without a model: the callback returns early for it.

**The menu model record** (0x48 bytes, one per picture; from `menuModelInit`, `menuModelRender`,
`menuModelSetMotion`, `menuModelFree` and the loader thread):

| Offset | Holds |
|---|---|
| +0x00 | state: `menuModelRender` draws only at 2; `menuModelCheck` waits while it is 1, so 1 = loading and 2 = ready (**likely**); 0 = free |
| +0x14 | nonzero when the picture is a **Pokémon with its move effects** (a move-sequence owner), zero for a plain model (**verified**: every user branches on it) |
| +0x1C | the requested animation (`menuModelSetMotion`) |
| +0x20 | the Pokémon's key for the Strategy Memo's size / angle values (`memoGetScaleAngle`; **likely**) |
| +0x24 | the model, or the move-sequence owner whose model is drawn (a 32-bit value; **verified**) |
| +0x28 | the loader thread while loading (**likely**) |
| +0x2C / +0x30 | picture width / height |
| +0x34 | the texture the model is rendered into |
| +0x38 | the picture's camera |
| +0x3C, +0x40, +0x44 | up to three lights |

Why (likely): loading runs on its own thread so the menu keeps animating while the model streams in; the record keeps
everything one picture needs, so a menu can own several (the P★DA has two side by side).

**Why the render-to-texture can happen in the middle of a window callback** (verified from the code, 2026-10-17):
the window callback runs while the game is *recording* (recordMode 1). Everything else drawn so far, such as the
striped background, is only in the request buffer, not yet in the framebuffer. `menuModelRender` switches to
recordMode 2, so the model goes straight to the GPU, and the framebuffer is still empty when it does:
- On the GameCube the previous frame left the framebuffer in two steps. It was copied out to the display buffer
  (`fn_801BF8A0`, HSD_VICopyXFBAsync: `GXCopyDisp` with a clear) at the start of this frame, by the first
  `GSgfxRenderLayer` that finds `frameBegun == 0`, together with `GSgfxBackFBDoFrame`.
- When the first frame-begin of the frame is the menu model's, the display copy runs right there. The model is then
  drawn into a cleared framebuffer, copied into its texture (clearing again), and the recorded commands are replayed
  afterwards by `GSgfxDrawScene` (0x800D3FA4), picture quad included.

The design shares one framebuffer between offscreen pictures and the screen at no extra memory cost. In exchange, the
order of the display copy, the immediate renders and the replay must hold exactly. A port or emulator that ends its
output frame anywhere other than at the display copy draws the offscreen picture on top of the previous frame, and the
copy's clear then leaves a black rectangle where the picture should be (**verified** on the PC port's name-entry
screen: moving the frame end to the display copy fixed it).


### Request-buffer commands (groups and formats verified from `gfxPlayCommands` and the recording setters; the
exact meaning of some entries, such as the packed colour forms and the matrix-stack operations, is likely)

Each recorded command is one word with its id, followed by its arguments in the layout its format code gives
(`gfxRecordCommand`: 1-10 words, 11-14 one to four floats, 15 a word and two floats, 16 a 3x4 matrix copied in, 17 a
word and a matrix, 18 a word, 0x18 bytes and a word, 19 three words and a matrix, 20 a word and a colour). The player
re-calls the same setter with recordMode 2.

| Ids | What they set |
|---|---|
| 0x01-0x04 | primitive type, begin (vertex count), end, flush |
| 0x05-0x10 | vertex position: x y z as f32 / u16 / s16 / u8 / s8 (0x05-0x09), x y in the same five formats (0x0A-0x0E), or an array index, 16- or 8-bit (0x0F / 0x10) |
| 0x11-0x15 | normal: f32, s16, s8, index 16 / 8 |
| 0x16-0x1B | colour (per colour slot): RGBA bytes, RGB bytes, packed 32-bit, two packed 16-bit forms, index 8 |
| 0x1C-0x22 | texture coordinate (per slot): s t as f32 / u16 / s16 / u8, index 16 / 8 |
| 0x23 | matrix index of the vertex |
| 0x24 / 0x25 | line width / point size (0-42.5 pixels, sent to GX in sixths; likely by the SDK functions' order) |
| 0x26 | bind a texture to a slot (`gfxLoadTexture`) |
| 0x27 | texture-matrix setup for a slot |
| 0x28 / 0x29 | set / clear `flags` |
| 0x2A | draw a captured display list |
| 0x2B-0x38 | pixel state: blend (two forms), destination alpha, colour / alpha / Z updates, Z mode, alpha test, dither, cull, clip, coplanar, layer, fog on / off, scissor, viewport |
| 0x39 / 0x3A | perspective / orthographic projection (four floats) |
| 0x3B-0x45 | matrix stack: reload the current matrix, rotate x / y / z, translate, scale, pop and load, multiply, pop and multiply, push, load into a GX matrix slot, read a slot back, reload after a stack change |
| 0x46 / 0x47 | set the vertex descriptor; draw a vertex list with a descriptor |
| 0x48 | TEV stage sources |
| 0x49-0x5B | channel and TEV setters (colour channels, TEV ops and inputs, constant colours, swap tables, indirect stages) |

Pointers recorded as one word (0x26, 0x2A, 0x46, 0x47) go through `PC_PTR32` / `PC_PTR` in the PC port ("Changing it").

### Map fog (verified, `gfxSetFog` in `gs_gfx_range_800D2B90.c`; 2026-10-15)

Entering a map hands its fog description (`scene_data` +0xC, hsd.md) to `gfxSetFog`. It releases the previous fog
object, loads the new description into an HSD fog object (`lbl_8047AA8C`), tells the particle renderer to fog its
particles too (`psSetFog`, 0x8016EB30), and sets the frame's clear colour to the fog colour, so the sky behind the
geometry fades into the same colour as distant objects. A null description turns HSD fog and particle fog off and
leaves the clear colour alone.

### The screen shape the game assumes 

The game is built for a **4:3 picture of 640 x 480 logical pixels** (**verified** from the code below unless marked):
- **Three kinds of camera projection.** HSD camera objects (`HSD_CObjSetCurrent`) use one of three: **perspective**
  (a field of view plus an aspect ratio, through the SDK's `C_MTXPerspective`), **frustum** (explicit left / right /
  top / bottom edges, `C_MTXFrustum`; also used by the two half-screen camera set-ups, `setupTopHalfCamera` /
  `setupBottomHalfCamera`) and **orthographic** (flat 2D, `C_MTXOrtho`). The game's own matrix helpers wrap the same
  calls (`GSmtx44Perspective`, `GSmtx44Ortho`). Which projection a camera uses comes from its data (hsd.md, the camera
  description).
- **2D is orthographic in 640 x 480:** menus, windows and text, and the movie player's picture (`THPGXYuv2RgbSetup` (0x801E1FF8), the
  THP draw set-up, builds its own orthographic projection).
- **Culling works in screen space.** Each frame, a model whose cull bit is set (models-animation.md (not yet published), flag 0x00100000)
  has the two corners of its bounding box projected through the current camera onto the screen (`fn_80191118`,
  `gfxWorldToScreen`); it is **out of view only if both corners lie beyond the same edge of a window 100 pixels larger
  than the screen on every side** (x from -100 to 740, y from -100 to 580). An out-of-view model is not drawn and its
  animation is not advanced. So culling follows whatever projection the camera has, but its window is fixed to the
  640 x 480 screen.
- **Menu model pictures** (above) render a 3D character with the menu's own camera into a fixed-size texture, then draw
  that texture as a flat 2D picture.

**Wider screens:** a wider picture needs a wider perspective (more view at the sides), the 2D layer kept in proportion,
and the culling window widened to match; frustum cameras, full-screen 2D effects and the 4:3 movies each need their own
handling.

## Disc data

None directly; textures, display lists and vertex data arrive through the model loaders (`hsd.md`,
`models-animation.md` (not yet published)).

**How a frame draws models (verified in the PC port, 2026-10-15).** The main loop's frame (`fn_801E0FB4`) is: frame begin
(`GSgfxRenderLayer`), playback of the recorded requests (`GSgfxDrawScene(flags, setupCamera, resetQueue)`, the main loop
passes 0x10, 1, 1), frame end (`GSgfxRenderEnd` (0x800D3190)). During playback, for each pass, `gfxSetupLayerCamera` makes the
active GScamera's HSD camera current, sets up the GS lights and calls `GSmodelDrawAll(flags, pass)` and the particle
draw, then the fog. Before such a frame the game makes a camera active (`fn_800D258C`) and calls
`_cameraLoadCameraMatrix`, which builds the GS view matrix from eye / up / interest (or rotation) and copies it into
the HSD camera with the flag that makes HSD use that matrix as is; without that call only the eye position reaches
HSD (`GScameraSetPosition` sets it directly) and the look-at is lost. The debug menu has a "draw a coloured triangle"
item (`fn_801330C8`) that exercises the immediate-mode path with an orthographic projection.
Frame flags: 0x10 selects only the opaque pass of `GSmodelDrawAll`; the main loop passes 0x10 or -1 (every pass)
depending on `lbl_80478820` (main.c `mainRetraceDraw` (0x80005D80)). With 0x10 alone a room loses its translucent / texture-edge
pieces (Outskirt Stand's baked shadow under the ramp disappears; verified in the PC port). A camera built from a room's
description (`GScameraCreateFromDesc` (0x800D27FC)) has its animation attached but stopped and look-at off, so until the animation runs its
view comes from a zero rotation (a level view from the eye); with `GScameraStartAnimation` it shows the authored
view (verified in the PC port).

## Changing it (port)

- **Display-list byte order (verified):** disc display lists stay big-endian and Aurora's
  `GXCallDisplayList` takes them unchanged (it emulates the GameCube's command FIFO). Pikachu and Outskirt Stand
  render correctly that way (hsd.md, "HSD runs in the PC port"). Only CPU-side readers of display lists need big-endian
  reads (GSgfx's parser does, `GS_DL_U16`). Vertex arrays need Aurora's size and byte-order arguments.

- Replacing graphics at the `gfx*` level means implementing the request-buffer player and the GX state cache on a
  modern API; replacing at HSD's GX calls keeps all of this code. The Melee PC port (same HSD) is the reference for
  the second option.
- **Use field names, never offsets.** In the PC port the pointer fields (frame target, descriptors, textures, capture,
  request buffer, emitters) are 8 bytes, so everything after 0x00C moves. Thirteen vertex-cache emitters
  (`_gfxEmitTexCoord2f32` (0x800D7304) .. `_gfxEmitPos3u16` (0x800D75AC)) and two colour setters keep a byte-pointer form for matching; they use
  `GSGFX_OFFSET(field)` (a constant on the GameCube, `offsetof` in the PC port), so they stay correct.
- The state is allocated with `sizeof(GSgfxState)`; the scratch slot 3 and the backup `lbl_80400248` must be that
  size in the PC port too (the backup is linker data on the GameCube: the platform layer must define it).
- **Pointers held in 32-bit words (2026-10-15, done in GSgfx):** descriptors and matrices travel as typed pointers
  (`GSVtxDesc*`, `GSMtxStackEntry*`). The request buffer keeps one 32-bit word per argument, so the four commands
  that record a pointer (0x26 texture, 0x2a display-list capture, 0x46 vertex descriptor, 0x47 descriptor + count)
  store it with `PC_PTR32` and `gfxPlayCommands` turns it back with `PC_PTR`; the descriptor pool base, the
  display-list end alignment and the frame-target sentinel test do the same. This relies on the port keeping game-visible memory below 4 GB
  (debug builds check every conversion). The matrix / colour command formats copy
  their data into the buffer, so they need nothing.
- **Descriptor contents in the PC port:** `GSVtxAttr.data` is a pointer, so a descriptor is larger in the PC port (the 0x1C
  attribute stride grows). GSgfx reaches attributes by field; code elsewhere that builds descriptors through
  `gfxVtxDescSetAttr` is fine, but any code that pokes descriptor bytes directly would not be (none found in GSgfx).
- `GSgfxRenderLayer` passes `frameRetraces` to four per-frame updates. Only `gsParticleUpdate(u32)` uses
  it; `fn_800D13C4`, `fn_800DC874` (empty) and `GSmodelApplyAll` ignore it, but the matched caller still loads it into
  the argument register, so those calls keep passing it (declared `void*`, harmless in the PC port).
- **Built-in records in the data section (2026-10-15):** eight ready-made vertex descriptors (`lbl_80314638` ..
  `lbl_80315128`, 0x190 bytes each, no arrays: the first two hold position and colour 0 only and differ in the
  position component-type index; the later ones add attribute 6, likely texture coordinate 0, and one adds attribute
  2) and the
  table of 30 SDK render modes (NTSC / MPAL / PAL / EuRGB60, 240- and 480-line, anti-aliased variants) sit in the
  data section as GameCube bytes. Read natively in the PC port those bytes give wrong values (big-endian, and the
  descriptor holds a pointer), so the PC defines each one with its C type and the same values. The render modes come
  from the disc, not from a PC graphics library's copy: one such copy differs on the MPAL TV mode and on PAL 528's
  sizes (verified for NTSC 480 interlaced double-strike, field by field).
- **A descriptor fixes the vertex layout, the setters do not (verified 2026-10-15).** The colour setters each write
  a fixed size whatever the descriptor says: `gfxVtxColor(slot, r, g, b)` emits 3 bytes, `gfxVtxColor4u8(slot, r, g, b,
  a)` (0x800D5CB8) 4. The caller has to pick the setter that matches the descriptor's colour format (`lbl_80314C78` declares colour
  0 as RGB8, so the lens flare uses `gfxVtxColor`). On the GameCube a mismatch draws garbage; in the PC port Aurora's
  command reader stops with "Reader overrun". Any new drawing code (a mod, a port feature) has to keep them paired.
- **Display-list callbacks get disc bytes (2026-10-15).** `GSgfxParseDisplayList` hands each vertex callback a pointer into the disc data (the vertex array for indexed attributes, the display list itself for direct ones), so the values are big-endian. Callbacks read them with `GS_DL_U16` / `GS_DL_F32` (gs_gfx_layer.h; plain loads on the GameCube). The model-bound walk is the only user found so far.
- **Display-list capture writes GX bytes (verified in the PC port 2026-10-15).** `GSgfxDLBegin` / `GSgfxDLEnd` build a display list in memory by writing the GX command bytes themselves (primitive opcode, 16-bit vertex count, then each vertex's attributes in the descriptor's formats), and `GSgfxDLDraw` plays it with `GXCallDisplayList`. Display lists are big-endian, so in the PC port every 16- and 32-bit value is written swapped (`GS_DL_BE16` / `GS_DL_BE32`; bytes as they are). Without that Aurora read a 3-vertex triangle as 768 vertices.
- **Back-framebuffer captures in the PC port (verified 2026-10-15).** `GSgfxBeginBackFBCapture(texture, callback, user)` (the fades, title / world-map overlays, the off-screen menu) copies the EFB into the texture at the **start of the next frame** (`GSgfxBackFBDoFrame` from the frame begin), when on the GameCube the EFB still holds the finished frame, then calls the callback with the frame count; returning 0 ends the capture. A port must keep this timing: the copy at the next frame's start, before the display copy empties the EFB (verified: copying at the end of the frame instead left the battle-start swirl's warp black). The main loop's `GSgfxSwapBuffers` (0x800D30F0) runs the same capture when no frame has begun yet.
- **Render to texture (verified in the PC port 2026-10-15).** The game draws into a texture by opening a frame with the
  texture as its target (`GSgfxRenderLayer(texture, 0)`), switching GSgfx from recording to drawing at once
  (`fn_800D4604(2)`), setting the camera, viewport and scissor to the texture's size, drawing, and closing the frame
  (`GSgfxRenderEnd` (0x800D3190)), whose end copies the EFB into the texture (`GStextureConvertFromHW`, `GXCopyTex`). The TV screen
  in the field (`tvRender`) is the cleanest example; the result is put on a model with `GSmodelSetTextureChange`. On
  the PC this works unchanged: Aurora keeps the copy on the GPU, keyed by the `GXCopyTex` destination address, and a
  later `GXInitTexObj` on that address uses it. The CPU-side bytes in the texture's buffer are **not** written on the
  PC; they come back only through Aurora's asynchronous readback (a few frames late). So any game code that reads a
  copied texture's pixels on the CPU would need that readback. None found so far (likely none): the back-framebuffer
  captures ("Back-framebuffer captures" above) hand the texture to callbacks that either draw it as a screen quad or do nothing. The test reads the
  copy back that way: the bytes come out in the GameCube's own layout (RGB565 in 4x4 tiles, big-endian) and decode
  to the expected picture.
- **GX lookup tables** (`lbl_80314350` primitive types, `lbl_80314370` attributes, `lbl_803143A8` / `B4` / `D8`
  attribute type / count / component type, `lbl_80314404` / `lbl_803144F0` TEV order coordinate and map): arrays of
  32-bit GX enum values indexed by GSgfx's small codes. In the PC port they must be converted per 32-bit word.

### Mod surface
- **Data it reads:** the scene descriptions it is handed (models, cameras, lights, fog), the video mode, per-scene frame
  pacing (`minFrameRetraces`).
- **Hooks it would need:** render resolution and aspect (widescreen), a post-processing pass, frame interpolation
  that keeps game logic on screen refreshes, and photo-mode switches (hide the HUD, free camera).
- **Fixed-size limits:** the matrix stack, the vertex-descriptor pool, the light and material pools; 640 x 480
  full-screen captures.
- **Must stay vanilla in GameCube-accurate mode:** frame pacing, frames with no display copy keeping their picture,
  and how many logic steps each drawn frame runs.

## Frame rate

**Frame-rate independence (verified from the code, `gsParticleUpdate` in `field_range_80117E58.c`):** the particle
system runs one simulation step (`psInterpretParticles` + `psExecGenerator`) per VI retrace that passed during the
last frame (`frameRetraces`), so particles keep real-time speed when the game drops to 30 fps. When
`GSgetVersionMajor` (0x800057A0) returns 2 (likely the 50 Hz PAL mode), one extra step runs every
5 frames, which brings 50 Hz up to 60 steps per second, the NTSC rate the effects were tuned for (likely). For the
port: an uncapped or 144 Hz frame rate must keep feeding the particle system retrace-equivalent steps (60 per second),
not one step per rendered frame. (The parameter is named `particleCount` in that file; it is a step count.)

### How the game paces itself, and what frame interpolation would need 

**How the game paces frames (verified from the code, `gs_gfx.c`).** Time is counted in **VI retraces** (60 per second
on NTSC). The retrace interrupt increments `retraceCount`. At the end of each frame `GSgfxWaitRetrace` (0x800D361C) waits for a retrace,
then keeps waiting until at least `minFrameRetraces` have passed since the last frame, and stores the number in
`frameRetraces`. If `waitRetrace` is off it then reports exactly `minFrameRetraces`, so a slow frame makes the game
run slower instead of skipping ahead. Every system then advances by `frameRetraces`:
- particles run one step per retrace (above);
- models get `GSmodelAdvanceAll(frameRetraces, 0)` plus one `GSmodelAdvanceAll(1, 1)` and one `gsParticleUpdateAll(1)` (0x801181B0) per retrace;
- lights (`GSlightUpdate`) and the other end-of-frame updates take the count;
- game code reads it through `GSgetFrameStep` (0x800D3088; it returns the retrace count). Menus run their tick
  callbacks that many times, and the title turns it into seconds with `frameRetraces / GSgfxVideoVsyncRate`.

So the game logic is a **fixed step of one retrace (1/60 s)**, run once or twice per drawn frame.

**The rate is chosen per scene (verified, the callers of `fn_800D3074`).** `fn_800D3074(n)` sets `minFrameRetraces`;
1 = 60 fps, 2 = 30 fps:
- rooms: `fn_801127BC` (floor.c) sets 1 for room kinds 5 and 6 and 2 for kinds 1-4 and the rest; another floor.c
  path resets to 1;
- battle menus: `menuFight.c` switches between 1 and 2;
- the title: 2;
- boot (`main.c`): 1;
- the debug menu: both values.

Which room kinds are 5 and 6 is not decoded yet. **Two different "60 fps"
codes** circulate for the US game, both **verified** from the code they patch:
- **The common community code** replaces the second instruction of `fn_800D3074` (at 0x800D3078, its "return if 0"
  check) with "set the value to 1", so the function **stores 1 whatever a scene asks for**: every scene asks for 60 fps.
- **The code Dolphin bundles** changes **one instruction in the per-frame retrace handler** (`mainRetraceDraw` (0x80005D80)): the load of
  the frame-pacing mode (`lbl_80478DD0`, proposed `gFramePaceMode`) becomes the constant 0, so `GSgfxWaitRetrace` (0x800D361C) is always
  called in **mode 0, no frame cap**; the game never waits for `minFrameRetraces`.
Either way the game logic, which advances one step per retrace counted, runs more often; neither code touches
the logic itself, which is why timing-sensitive scenes break.
- **Why some scenes didn't reach 60 (likely):** where the hardware can't finish a frame in one retrace, the real
  rate stays lower.
- **Why there were timing glitches (likely):** code that counts drawn frames instead of reading `frameRetraces`
  speeds up. Candidates: wait loops that call `_threadSwitch` a fixed number of times, and anything tuned for
  30 fps scenes.

**What this means for the port (likely).**
- **Run the logic at the game's own rate:** one tick = `minFrameRetraces` retraces, with `frameRetraces` reported as
  exactly that.
- **The retrace count the game sees stays at the console's 60 per
  second whatever the monitor's rate.** Interpolated frames (60 / 120 / 144 fps) never advance it, and battle speed-ups
  scale it together with the rest of the battle. This matters beyond smoothness: the idle battle camera's timer counts
  these retraces before it draws from the shared random generator ([battle-camera.md](../systems/battle-camera.md)), so an extra
  counted retrace moves battle RNG. **Check for the interpolation work:** the same battle replay gives **identical
  random numbers** at 30, 60, 120 and 144 fps.
- **Draw extra frames by interpolating between the last two ticks.** Forcing 60 fps logic everywhere (what the
  Dolphin cheat does) changes timing the original was tuned for.
- **The tick period changes per scene** (1/30 s or 1/60 s): the interpolation clock must read `minFrameRetraces`
  each tick, not assume 30 Hz.

What each system would need:

| System | How it updates now | What interpolation needs | Effort (rough estimate) |
|---|---|---|---|
| Model transforms (HSD JObj world matrices) | rebuilt from position / rotation / scale when drawn | record each joint's final matrix by its JObj pointer per tick and blend the last two for in-between frames | medium: one hook where HSD builds and loads the matrices |
| Skeletal animation (HSD AObj, envelope skinning) | AObj frames advance per retrace through the model update | covered by the joint matrices above (skinning uses them); rotations near 180° need angle blending instead of matrix blending (SoH's "paper effect") | low once the matrix hook exists |
| Camera (GScamera / HSD CObj) | eye, target and field of view per tick | blend eye / target / fov; reset on cuts (room change, camera snaps, cutscene shots) | low to medium: cuts must be detected |
| Particles (GPT1 / `ps*`) | one step per retrace; particles born and die in the step | track each particle by pointer, blend position, skip a particle's first frame; generators that respawn every step need pairing (Dusklight hit the same thing) | high |
| UI tweens (window sequences, title column easing, cursor moves) | eased per tick (`fn_8001EF78`, `winSeqMoveMenu`) | blend the sprite or window positions per pointer, or run the easing on real time (Dusklight's approach for its UI) | medium |
| Screen fades (`fadeDaemon`) | alpha per tick | blend the alpha | low |
| Text typing (GSmsg) | characters per retrace | leave at the logic rate (characters appear on ticks) | none |
| Back-framebuffer captures (screen blur, fades, overlays) | copy of the last drawn frame | capture only on logic ticks, or the blur strength changes with the frame rate | medium |
| Lights, fog, material scrolls | per tick | blend colours and texture offsets | low |

**How other decomp ports do it (ideas only; read 2026-10-16).**
- **Ship of Harkinian** (Ocarina of Time, HarbourMasters/Shipwright):
  - Records every final matrix each logic frame in a tree built from the game's display-list open / close points
    and hand-placed child markers, so the matching matrix can be found in the next frame.
  - Draws in-between frames by blending the final matrices with a factor 0..1.
  - Blending inputs (angles, scale) and rebuilding was tried and dropped: some animations looked wrong. Rotations
    near 180° need a special case.
  - Also: a "don't interpolate the camera" call on camera cuts, per-object opt-outs, and separate paths for skinned
    limbs and ribbon trails.
- **Dusklight** (Twilight Princess, TwilitRealm/dusklight, CC0-1.0, so compatible with anything):
  - A game clock keeps the simulation at 30 ticks per second and draws "presentation" frames between ticks.
  - Final matrices are recorded per key and replaced with blended ones while drawing.
  - Separate modules: camera (re-runs the camera's draw step with blended values), particles (pairs particles that
    are recreated every frame), skinned vertices, materials, lines (ropes) and UI.
  - A "presentation epoch" changes whenever history must be thrown away (cuts, teleports), so nothing blends
    across a jump.
  - The UI runs its animations on real time between ticks.
  - Its 2.0 notes list the kinds of fixes that follow: weather and particle effects, map markers, reticles, small
    animals, ropes, grass out of sync with the camera, and lock-on markers blending between unrelated targets.

Colosseum shares Dusklight's renderer approach (Aurora). HSD gives one central matrix path (JObj setup), so the
matrix-recording hook is likely simpler than in either port. The particle system is the hardest part in all three.

## Efficiency

All pools are sized once at boot from the caller's numbers. The state cache skips redundant GX register writes
(dirty bits for channels, texgens and TEV; compare-before-set for blend, Z and pixel modes; `boundTexture` skips
texture rebinds). Recording the frame once and replaying it per pass avoids walking the scene graph twice. The
vertex emitters are function pointers chosen per attribute format when the format changes, so the per-vertex path
is a single indirect call per attribute that writes straight to the GX FIFO.

## Debug menu toggles

The 0x49C toggle (`lbl_8047AED8`) is labelled "Zクリア後描画のON/OFF"
("draw after the Z clear, on / off"), message 0xEAFE. Its neighbour `debugMenuShadowBorderDisp` is "シャドウ外枠表示
ON/OFF" ("show the shadow's outer frame"), message 0xF2CA (verified: the debug menu's entry table is in its own
REL module, `debugMenuData`, packed in an `.fsys` archive; each 12-byte entry holds the label's message ID and the
function, and the labels are in the shared message bank). The same menu page also has collision display, a
collision-disable button, "GSgfx info on / off" and BGM volume.

## Region notes

- Video modes 1-4 (`GSGFX_VMODE_*`: NTSC, PAL50, PAL60, MPAL) pick the render mode at init; USA uses NTSC.
- `gsParticleUpdate` adds one particle step every 5 frames when `GSgetVersionMajor` (0x800057A0) returns 2, to keep 60 steps per
  second at 50 Hz. In the USA build `GSgetVersionMajor` (0x800057A0) returns the constant 1 (verified, `main.c`), so the branch is never
  taken. `main.c` calls it a "major version"; its use here and the shared numbering (1 NTSC, 2 PAL, as
  `GSGFX_VMODE_*`) suggest it is the build's video standard (likely): a PAL build would return 2.
- `lbl_8047AA9C` is set by `GSgfxInit` when the video mode is 2 or 3 (PAL50 / PAL60), and the mode switcher
  `fn_800D37D4` refuses the PAL50 standard unless it is set (verified): a "PAL machine" flag.

## Open questions

- What `GSgetVersionMajor` (0x800057A0) returns in the PAL build (to confirm the video-standard reading).
