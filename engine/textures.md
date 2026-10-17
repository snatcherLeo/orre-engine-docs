# Textures, shiny colouring and portraits

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-14). Written for shiny colour editing and portrait (mugshot) work: texture formats and where textures live, how a Pokémon
becomes shiny, and how portraits are stored. Evidence: `GStextureGetGXformat`
(`src/game/gs_texture_getters_exact_800EF3E0.c`), `GStextureHandle` (`include/game/gs_texture.h`),
`fn_801DE190` / `fn_801DCE0C` (`src/game/sequence.c`), the portrait lookup in
`src/game/gs_range_80109C88.c`, and a survey of every texture entry on the USA disc.
**How sure:** the functions named on this page byte-match the game, so the layouts, the format table and the code
paths described are *verified* (read from matching C); the survey counts are *verified* by reading every texture; other
remarks are marked *likely*, *guess* or open inline.

## What it does

There are two kinds of textures:
- **Stand-alone textures** (resource type 9, `GStexture`): menus, logos, portraits, body
  pictures, character art. One file = one 0x80-byte header + pixel data.
- **Model textures** inside HSD `.dat` models (battle PKX models, overworld models, maps): stored
  the HSD way (TObj image descriptors, same as Melee), not as type-9 files.

## Data structures

### Type-9 texture file

The file is a `GStextureHandle` as saved to disc, then the pixels:

| Offset | Field |
|---|---|
| 0x00 | u16 width, u16 height |
| 0x04 | u8 bits per pixel (0x20 = 32, 0x10 = 16, ...) |
| 0x05 | u8 mip level count (1 = base only) |
| 0x08 | s32 GS format code (table below) |
| 0x0C | u32 palette (TLUT) format |
| 0x10-0x20 | wrap S/T, min/mag filter, LOD (runtime GX settings; *likely*) |
| 0x28 | u32 offset of mip level 0 (0x80 on disc; becomes a pointer at load) |
| 0x48 | u32 offset of the palette, for palette formats (becomes a pointer at load; *guess* from `tlutData`) |
| 0x54 | 0x20 bytes reserved for the GameCube GX texture object, 0x74: 0x0C for the palette object (filled at load: `GStextureLoad` builds them in place). |
| 0x80 | pixel data, in GameCube tiled layout (heights round up to whole tiles: a 256x239 menu stores 240 rows) |

**GS format code -> GameCube GX format** (`GStextureGetGXformat`):

| GS code | GX format | | GS code | GX format |
|---|---|---|---|---|
| 0x00 | C4 (4-bit palette) | | 0x43 | IA8 |
| 0x01 | C8 (8-bit palette) | | 0x44 | RGB565 |
| 0x30 | C14X2 | | 0x45 | **RGBA8** |
| 0x40 | I4 | | 0x90 | **RGB5A3** |
| 0x41 | IA4 | | 0xB0 | **CMPR** (S3TC/DXT1) |
| 0x42 | I8 | | 0xA0 | I8 when drawn with alpha, else GX format 0x27 |

### Survey of every type-9 texture (USA)

| Where | Count | Formats | Typical sizes |
|---|---|---|---|
| `poke_face.fsys` (portraits) | 418 | RGBA8 (417), RGB5A3 (1) | **42x84 (370)**, 84x84 (48): sheets of 42x42 cells |
| `poke_body.fsys` (body pictures) | 826 | RGB5A3 (822), RGBA8 (4) | 64x64 |
| `chara_big` / `chara_small` (character art) | 98 | RGBA8 | 200x150, 71x75 |
| menus, logos and other screens | 144 | RGB5A3 (60), RGBA8 (57), CMPR (24), RGB565 (3) | 640x480 full screens, smaller panels |
| room archives | 1 | RGBA8 | |

Room, NPC, trainer and Pokémon model textures are inside their HSD models (their formats and counts: hsd.md,
hsd-models.md).

## Shiny colouring (code)

Two mechanisms, chosen per species by `fn_801DE190` (`src/game/sequence.c`):

1. **Separate shiny model (35 species).** When the model is created as shiny, the species is looked
   up in `lbl_80374F20` (records of {u16 species, archive id, entry id}); if found, the `pkx_rare_*`
   model is loaded **instead** and the recolour below is **skipped**. The 35 are the three Kanto starters' final forms, Gyarados, the legendary birds,
   Mewtwo and Mew, the Johto starters' final forms, Noctowl, the legendary beasts, Lugia, Ho-Oh, Celebi, all nine Hoenn
   starter forms, Kyogre, Groudon, Rayquaza, Latias, Latios, Jirachi and Deoxys; each archive is named `pkx_rare_`
   plus the species' Japanese name in romaji (*verified* from the table).
   Three of them (Charizard, Articuno, Moltres) carry particle banks identical byte for byte to the normal forms'
   banks (*verified*).
2. **Runtime recolour (every other species):** below.

For a shiny editor: species with a separate model need their `rare_*` model (textures) edited,
or the table entry removed so the recolour applies; all others edit the 20-byte colour block.
This agrees with StarsMmd's GameCube model addon documentation.

### Runtime recolour

- Battle models have **no shiny textures**. When a model is created, `fn_801DE190(index, model,
  variant)` loads the PKX and, if `variant` is nonzero, calls `fn_801DCE0C`, which applies the
  PKX's colour block: `GSmodelEnableColorSwap(model, r, g, b, a)` (a **channel swap**: which source
  channel feeds each of R, G, B, A) and/or `GSmodelEnableModulation(model, rgba)` (a **colour
  multiply**, 0x7F = unchanged).
- Battle code passes the shiny check straight in: `variant = pokemonCheckRare(pokemon)`
  (`fight_out_pokemon_suffix_80207C6C.c`).
- **Where the values are stored:** in each species' own PKX file, after the animation slot table:
  4 u32 channel indices (0, 1, 2, 3 = identity) then 4 bytes of RGBA modulation, present when PKX
  header flags bit 31 is set (378 of 527 PKX files) (models-animation.md (not yet published)).
  The swap is used only if the indices differ from 0,1,2,3; the modulation only if a byte differs
  from 0x7F.
- So a shiny editor edits 20 bytes per species; a delta-species clone gets its own PKX (or its own
  copy of those 20 bytes), so editing it never touches the original (*likely*: follows from the layout above).

## Portraits and body pictures (code + data)

- Each species record ends with **two face entries** (`PokemonData.face[2]`, +0x10C, 8 bytes each):
  **[0] = normal, [1] = shiny**. Each holds a colour byte (Pokédex colour; the same in both), a
  **portrait id** (u16, +0x02) and a **body-picture id** (u32, +0x04).
- Lookup: `pokemonGetStatus(NULL, species, 0x5B, isShiny)` returns the portrait id; the portrait
  code in `gs_range_80109C88.c` picks it with `pokemonCheckRare`. Unown (species 0xC9) uses a
  per-form table instead (`lbl_8035B478[form][isShiny]`). Eggs and invalid Pokémon use portrait 0x33D.
- A portrait id indexes the **menu-face table** (`_menuFaceBiosGetPtr`, 830 records of 8 bytes in
  `common_rel`): byte 0 bit 7 = **shiny cell** (`fn_8010C388`), bit 6 = valid (*guess*), +0x04 = the
  texture's FSYS entry id (for example 0x0DED1200 = `face001` in `poke_face.fsys`). Bulbasaur's
  normal (0x0001) and shiny (0x0183) portrait ids point at the **same texture**; only bit 7 differs.
- **A portrait texture is a sheet of 42x42 cells** (the drawing code crops 0x2A x 0x2A,
  `fn_8010B9E8` and its neighbour):
  - top cell (y 0) = normal, bottom cell (y 42) = **shiny**, used when bit 7 is set and the texture
    is taller than 42 (so shiny portraits are their own pixels, not a recolour);
  - in an 84-wide sheet, the right column (x 42) is used when the sprite is drawn with a negative
    width (facing the other way); 42-wide sheets (370 of 418) have no separate other-facing cell,
    so those portraits are presumably just flipped (*guess*; to check in-game).
- **Body pictures** (64x64, `poke_body.fsys`) are separate files: the shiny one has the `_c`
  suffix (Bulbasaur: `body001` = entry 0x0815, `body001_c` = 0x0816, as stored in face[0] / face[1]).
- Portrait textures are cached in a small most-recently-used list (`fn_8010BD6C`; evicting from
  `poke_face.fsys`, archive id 0x5C0).

### The portrait slots and their load queue (from the code and the first battle)

- **Slots** (`fn_8010C224` sets them up, count given by the caller): each slot holds the face record it shows, its
  portrait id, a **state** (0 = waiting to load, 1 = loading, 2 = ready), the index of its own 0x6EC0-byte texture
  buffer, and an optional "next portrait" callback with its argument (**verified** from the code; all of these
  functions are exact matches of the original).
- **Requesting a portrait** (`fn_8010AE2C`, used by the battle with no callback): it works out the portrait id from the
  Pokémon (species, shiny, Unown form), queues it (`fn_8010B16C`) and then **waits, yielding its thread each frame,
  until that slot is ready** (**verified**).
- **The queue** (`fn_8010B16C`): if the portrait is already in a slot, it moves to the most-recently-used end;
  otherwise it takes a free slot, or **evicts the oldest slot** when all are full. Then it starts the load of the
  first slot still waiting (state 0) through the archive streamer (`fn_8017B000`, archive 0x5C0), unless one is
  already loading (**verified**). Menus pass a callback that hands out the next Pokémon of a list, so a whole party's
  portraits stream in one after another (**likely**).
- **When a load finishes** (`fn_8010B5C4`, called by the archive streamer): the loaded texture is made relative
  again, copied into the slot's own buffer and set up there, the archive copy is released, the slot becomes ready,
  and the queue is run again for the next waiting slot (**verified**). This copy step is why a portrait's texture is
  set up twice (once by the archive, once in the slot).
- **Why it matters / efficiency:** a fixed number of fixed-size buffers, one load at a time, and least-recently-used
  eviction keep portraits within a known memory budget and spread disc reads over frames (**likely** the intent).
- **Open (2026-10-17):** in the PC port (the decompiled code running natively), Willie's battle stops at the trainer intro with the queue spinning on a waiting
  slot whose portrait lookup returns nothing (the PC build now prints every slot's id, state and lookup result when
  the queue spins; next run). Not yet known whether a slot's id is wrong or the lookup misses in the PC port.

## Appearance inventory (every place a Pokémon's look shows up)

For a shiny editor, which must change all of them together. Status per row: known (code or
data), or open.

| Where | What is shown | How normal / shiny is stored | Status |
|---|---|---|---|
| Battle | 3D battle model (PKX) | 35 species: a **separate shiny model** (`pkx_rare_*`, table `lbl_80374F20`); all others: **one model**, recoloured at runtime from the PKX colour block (`fn_801DE190(..., pokemonCheckRare)`) | known (code) |
| Menus, party, summary, trade (portraits) | 42x42 portrait | **separate pixels**: bottom cell of the same `poke_face` sheet (menu-face record bit 7) | known (code) |
| Body pictures (64x64, `poke_body`) | 2D body picture | **separate files**: `bodyNNN` / `bodyNNN_c` (species face[0] / face[1] body ids) | known (data); which screens use them: open |
| PC box | icons or portraits? | | open |
| Summary screen | portrait, body picture or 3D model? | | open |
| Strategy Memo (Pokédex-like memo) | picture of each seen Pokémon | | open |
| Evolution scene, Pokémon Relive / purification, snag, trade | 3D model (the same PKX path is called from `evolution.c`, `pokemon_relive.c`) | presumably the battle model path, with the shiny flag passed (*likely*; to confirm per call) | open |
| Overworld (Pokémon shown in the field, e.g. Castform, `powalen_b1`, in M1_out) | overworld model (type 2) | | open |
| GBA link / Pokémon Box transfers | data only (personality decides shininess) | n/a | known |

## Decoding

Type-9 textures use the standard GX formats RGBA8, RGB5A3, RGB565, I4, I8, IA4, IA8, CMPR, and C4/C8 with an
RGB5A3/RGB565/IA8 palette; a decoder for these formats reads every texture on the disc to PNG (*verified*).

## Changing it

(*Likely*: these steps follow from the verified layouts but have not been tried in the game.)

- New portrait: a type-9 RGBA8 sheet of 42x42 cells (normal on top, shiny below; add a right column
  for a separate other-facing view), a menu-face record pointing at it (bit 7 set for the shiny id),
  and the species' face[0] / face[1] portrait ids.
- New shiny colours for battle models: edit the PKX colour block (or add one and set flag bit 31).
- Tools need a GX texture codec (RGBA8, RGB5A3, CMPR, palette formats) in tiled layout.

## Open questions

- Whether 42-wide portraits are flipped when facing the other way (to check in-game); how the faces
  were framed.
- Shiny Pokémon shown outside battle (PC or summary 3D views): presumably the same colour block (*guess*);
  check `fn_801DE190`'s other callers.
