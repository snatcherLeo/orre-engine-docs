# Save data (in memory)

Status: **partial** (2026-10-17). The in-memory save block, its sections and the memory card file format (below)
are described; several sections' inner layouts are still open.

## What it does

All persistent game state lives in **one contiguous block**. Code never hard-codes offsets into it:
it asks `savedataGetStatus(save, section)` (0x80129280) for a pointer to one of 17 sections. Passing
`save = NULL` means "the current save", fetched from the global `lbl_8047ADB8` (`fn_80128E24`).
Each section getter is a one-line `base + offset` (`fn_80128CC0`..`fn_80128E04`,
`src/game/pokemon_range_8011F5FC.c`).

## Data structures: the sections

| # | Getter | Offset | What it holds (evidence: callers) | Confidence |
|---|---|---|---|---|
| 0 | - | +0x0 | the whole block (save-integrity checks: `gamedataBiosGetGamedataAtttestPtr`) | high |
| 1 | `fn_80128E04` | +0x0 | save bookkeeping: memory-card id, save id (`gamedatasaveBiosSetMemcardID`, `memcardSetSaveId`) | medium |
| 2 | `fn_80128DEC` | +0x70 | **the player**: name, party, bag, money, badges (`HeroSaveData`, below; `heroBiosGetPokemonPtr`) | high |
| 3 | `fn_80128DD4` | +0xB88 | **PC box** storage (`pcbox*`, `pokemonInitAry`) | high |
| 4 | `fn_80128DB8` | +0x1BDBC | **story-flag buffer, class 1** (0x20 bytes = 8 words) | **verified** (main.c passes sections 4 / 7 to `GSflagInit`, `fn_801909A8`) |
| 5 | `fn_80128D9C` | +0x1BDDC | **story-flag buffer, class 2** (0x80 bytes = 0x20 words) | **verified** (sections 5 / 8) |
| 6 | `fn_80128D80` | +0x1BE5C | **story-flag buffer, class 3** (0x600 bytes = 0x180 words) | **verified** (sections 6 / 9) |
| 7 / 8 / 9 | constants | - | the **word counts** of the flag buffers in sections 4 / 5 / 6 (8, 0x20, 0x180: exactly their sizes) | **verified** (2026-10-16) |
| 10 | `fn_80128D68` | +0x7D20 | **mail**: mailbox, flags, sort mode, received count (`mail*`) | high |
| 11 | `fn_80128D4C` | +0x8168 | **daycare** (`sodateya*`: the Pokémon left there) | high |
| 12 | `fn_80128D30` | +0x82A8 | **Strategy Memo** (`memo*`: seen/caught Pokémon notes) | high |
| 13 | `fn_80128D14` | +0x9A1C | **Card-e** data (`CardEGrid*`, `cardesavedata`) | high |
| 14 | `fn_80128CF8` | +0xE3E8 | **Battle Mode state**, **not** the flag storage (that is sections 4-6); holds Battle Mode state (see below); the rest to re-check | medium |
| 15 | `fn_80128CDC` | +0x1B014 | **Shadow Pokémon state**: 0x18-byte rows per Shadow id starting at +0x49C (row +0x0A: the heart-gauge size used when the Shadow Pokémon is created, `fn_801EEEB8` / setter `fn_801EEE6C`); reached through `fn_801EF1E4` | medium |
| 16 | `fn_80128CC0` | +0x1C45C | **ribbons** (`exribbon*`: National / Earth ribbons) | high |

Section order in memory is 0/1, 2, 3, 10, 11, 12, 13, 14, 15, 4, 5, 6, 16, so the block is at least
0x1C45C bytes plus the ribbon section (~113 KB).

### Section sizes (from consecutive offsets) and cross-checks

| Section | Start | Size | Check |
|---|---|---|---|
| 2 player | +0x70 | 0xB18 | = `sizeof(HeroSaveData)` summed field by field |
| 3 PC | +0xB88 | 0x7198 | = 3 boxes x 0x24A4 + 235 items x 4 (below) |
| 10 mail | +0x7D20 | 0x448 | |
| 11 daycare | +0x8168 | 0x140 | one 0x138-byte Pokémon + 8 bytes |
| 12 Strategy Memo | +0x82A8 | 0x1774 | |
| 13 Card-e | +0x9A1C | 0x49CC | = the size `fn_80083CBC` clears with `memset` |
| 14 Battle Mode state, ... | +0xE3E8 | 0xCC2C | |
| 15 Shadow | +0x1B014 | 0xDA8 | 0x49C header + 96 (0x60) rows x 0x18 = matches the 0x60 Shadow ids |
| 4 | +0x1BDBC | 0x20 | |
| 5 | +0x1BDDC | 0x80 | |
| 6 | +0x1BE5C | 0x600 | |
| 16 ribbons | +0x1C45C | ? | last section |

Four independent size checks agree with the section map, so the offsets can be trusted for a save
editor.

### Section 2: the player (`HeroSaveData`, `src/game/hero_savedata_exact_80129280.c`)

| Offset | Field | Notes |
|---|---|---|
| +0x000 | name (0x2C bytes) | GS-encoded text |
| +0x02C | rnd | u32 (trainer id / random seed? guess) |
| +0x030 | party: 6 x 0x138-byte Pokémon | the in-save Pokémon record is **0x138 bytes** |
| | normal items (0x14), extra items (0x2B), Poké Balls (0x10), TMs/skills (0x40), berries/seeds (0x2E), colognes (3) | each slot `{u16 id, u16 count}` |
| | sex, home place | u8 each |
| | Poké Dollars, Poké Coupons, total coupons earned | s32 each |
| | badge flags x 8, "hizuki" flag | u8 each |
| | "hizuki" items (0xA slots), "hizuki" name (0x56) | "hizuki" is **Rui's internal name**; the 10 slots are her cut item pocket (bag-and-pc.md), not a date |

Access is through `heroGetStatus(save, selector, index)` / `heroSetStatus`: selector 3 = party slot
`index` (a Pokémon pointer), 0xD / 0xE = Poké Coupons / total coupons (`heroAddPokecoupon`).

### Section 3: the PC (`src/game/pcbox.c`)

- **3 boxes** (`pcboxGetNbPokemonBox`), each `{0x14-byte header incl. the box name, 30 Pokémon x
  0x138 bytes}` = 0x24A4 bytes; boxes end at +0x6DEC.
- **PC item storage:** 235 slots (`pcboxGetNbItemSlot`) of 4 bytes `{u16 item, u16 count}` at
  +0x6DEC.
- Section 3 is therefore 0x7198 bytes and ends at +0xB88 + 0x7198 = **+0x7D20, exactly the start of
  section 10 (mail)**: an independent check of the section map and of the 0x138-byte Pokémon slot.

## Battle Mode settings in section 14 (2026-10-16, verified from the menu headers)

+0x00 is the Battle Mode kind (0 Colosseum Battles, 1 Mt. Battle 100: `BATTLEMODE_BATTLEYAMA100` in a retail assert,
2 Gang Battle, 3 Battle Now); +0x08 is the rule (0 any, 1 Lv50, 2 Lv100, 3-5 edited rules).

## Memory card file and the save header

- **Card file:** name `pokemon_colosseum`, created with `CARDCreateAsync(..., 0x60000, ...)`: **0x60000
  bytes (384 KB, 48 card blocks)**, several times the ~113 KB block, so it likely holds more than
  one copy (main + backup slots: guess). Region check on load: `regions.md`.
- **Attestation stamp** (`gamedataAttest*`, 4 bytes): byte 0 version id, byte 1 generation id,
  byte 2 area id, byte 3 language-area id. Checked on load (`gamedataAttestCheckValid`). Values written
  for a new save (`hero.c`): version 0xB (Colosseum), generation 3, area/language 1/1 (Japan),
  2/2 (USA), 3/8 (PAL) (`regions.md`). **Every
  Pokémon record carries the same kind of stamp** (`attest[4]` at +0x08 of `PokemonBios`): the
  game, generation, region and language it came from. A Pokébank must preserve it.
- **Save header** (`gamedatasaveBios*`, section 1): audio option, vibration off, current floor id,
  previous floor id, position index on the floor, play time, save count, memory-card id, and a
  random save id (`Savernd`).

### Card file layout, checksum and encryption (2026-10-14; `field_range_801CB180.c`)

| File offset | Size | Contents |
|---|---|---|
| 0x0000 | 0x6000 | card header area: 0x40 bytes of comment/status, then the banner and icon images copied from `lbl_802758E8` (0x1800 + 0x2200 bytes); written as 0x4000 or 0x6000 bytes depending on a flag |
| 0x6000 + n x 0x1E000 | 0x1E000 each | **3 save slots** (n = 0, 1, 2), written in 0x2000-byte chunks |

A header chunk is checked with a plain **32-bit word sum**: the sum of words 0..0xE8F must equal
word 0xE90; a mismatch only sets a flag (`raw[0x40]`, re-creation of the header, guess).

**Save slot (0x1E000 bytes):**

| Slot offset | Field |
|---|---|
| +0x04 | u32 save counter: on load the slot with the highest counter wins (main + backups) |
| +0x08 | the game save block, 0x1DFD0 bytes (copied out to the live save) |
| +0x0C | zeroed before checking (part of the slot header inside the block) |
| +0x18 .. +0x1DFD8 | **encrypted** with a SHA-1 key stream (below) |
| +0x1DFEC | 20-byte **SHA-1 digest** of the plain slot (bytes 0 .. 0x1DFD8) |

**Encryption** (`fn_801CBE44` when saving, `fn_801CBCDC` when loading; SHA-1 in `fn_801CBBAC`,
`fn_801CBF64`, `fn_801CC380`):
- Save: `digest = SHA1(slot[0 .. 0x1DFD8])`; `key = NOT digest` (all five words inverted); then for
  each 20-byte block from +0x18: `block ^= key`, and `key = SHA1(encrypted block)`. The digest is
  stored at +0x1DFEC.
- Load: `key = NOT stored digest`; for each block: `next = SHA1(encrypted block)`, `block ^= key`,
  `key = next`; then the slot is valid only if `SHA1(decrypted slot)` equals the stored digest.
- If no slot verifies, the code repairs the slot header's own small checksum (eight words summing
  to 0, word 3 adjusted) and reports a damaged save.

For a save editor or Pokébank: decrypt with the rule above, edit the block at +0x08, then
re-encrypt (which recomputes the digest). The first 0x18 bytes of a slot stay unencrypted.

## Flow

Load: the memory card file is read into the block, then checked (`gamedataBiosGetGamedataAtttestPtr`,
attestation). Play: every system reads and writes its section through `savedataGetStatus`.
Save: the block is written back to the card (`gamedatasave*`, `memcard*`).

## Disc data

None: the save is created at new-game time (`pokemonInitAry`, `sodateyaInit`, `memoInit`,
`exribbonInit`...).

## Changing it

- A save editor can work on this block directly, with the section table above as its map; the
  Pokémon record (0x138 bytes) and the item slots are the most valuable layouts to pin down next.
- Adding data for mods: append a new section rather than resizing one, since many systems compute
  addresses from fixed offsets.
- XD's save almost certainly uses the same mechanism with different sections (`xd-insights.md` (not yet published)).

## Efficiency

One allocation, no per-system save code: each system owns a fixed slice. Saving is a single block
write.

## Open questions

- The 0x40-byte card comment area and the slot header words +0x00, +0x10, +0x14; XD's version id
  and whether XD uses the same SHA-1 scheme.
- The other fields of the section-15 Shadow rows and what precedes +0x49C.
- Full layout of the 0x138-byte Pokémon record (species, PID, IVs/EVs, moves, Shadow data).
- Region differences in the block (name lengths, text encoding of names).
