# Save data (in memory)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-17). The in-memory save block, its sections and the memory card file format (below)
are described; several sections' inner layouts are still open.

## What it does

All persistent game state lives in **one contiguous block**. Code never hard-codes offsets into it:
it asks `savedataGetStatus(save, section)` (0x80129280) for a pointer to one of 17 sections. Passing
`save = NULL` means "the current save", fetched from the global `lbl_8047ADB8` (`savedataGetWorkPtr` (0x80128E24)).
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
| 14 | `savedataBiosGetE3E8Ptr` (0x80128CF8) | +0xE3E8 | **Battle Mode state**, **not** the flag storage (that is sections 4-6); holds Battle Mode state (see below); the rest to re-check | medium |
| 15 | `fn_80128CDC` | +0x1B014 | **Shadow Pokémon state**: it opens with 12-byte entries, one per Shadow save slot (the slot number is the Shadow table's +0x0A), whose +0x06 is a 16-bit **footstep count** (**verified** from the code: the field's step callback adds 1 per step for every unpurified Shadow Pokémon in the party and one left at the day care, and at 0x100 applies the walking or day-care gauge drop and resets it; `fn_801EE470` reads it, `darkPokemonSaveSetDp` writes it, despite that name; field-steps.md); then 0x18-byte rows per Shadow id starting at +0x49C (row +0x0A: the heart-gauge size used when the Shadow Pokémon is created, `darkPokemonSaveGet4A6` (0x801EEEB8) / setter `darkPokemonSaveSet4A6` (0x801EEE6C)); reached through `darkPokemonSaveGetPtr` (0x801EF1E4) | medium |
| 16 | `fn_80128CC0` | +0x1C45C | **ribbons** (`exribbon*`: National / Earth ribbons) | high |

Section order in memory is 0/1, 2, 3, 10, 11, 12, 13, 14, 15, 4, 5, 6, 16, so the block is at least
0x1C45C bytes plus the ribbon section (~113 KB).

### Section sizes (from consecutive offsets) and cross-checks

| Section | Start | Size | Check |
|---|---|---|---|
| 2 player | +0x70 | 0xB18 | = `sizeof(HeroSaveData)` summed field by field |
| 3 PC | +0xB88 | 0x7198 | = 3 boxes x 0x24A4 + 235 items x 4 (below) |
| 10 mail | +0x7D20 | 0x448 | |
| 11 daycare | +0x8168 | 0x140 | an 8-byte header, then one 0x138-byte Pokémon record (**verified** from the code: +0 state, 0 = empty; +1 the Pokémon's level when it was left; +4 its heart value when it was left, signed 32-bit; the record from +8, `sodateyaGetPokemonPtr`; `field_exact_801ED218.c`) |
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
| +0x02C | rnd | u32 |
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
- (**verified** on a real save) The box layout reads correctly from a decrypted save block: the
  box names decode as UTF-16 ("MY BOX", "BOX2", "BOX3"), and the boxed Pokémon are full 0x138-byte records in the
  party format (species, level, IVs, EVs, nature and stored stats agree with the Gen 3 stat formula); an empty slot
  has species 0, and no box count field was needed to read them.
- (**verified** by comparing two end-of-session saves of the same play, one from the console and
  one from a PC replay) The 0x138-byte record has **padding bytes the game never names**: after the species id
  (+0x02-0x03), after the catcher's sex (+0x11-0x13), the last two bytes of the 0x18-byte original-nickname buffer
  (+0x5A-0x5B; a name with its end mark fills at most 0x16), and gaps further on (+0xD1, +0xDA-0xDB, +0xE6-0xE7, among
  others). On the console these hold whatever was in memory when the record was built (different values per
  Pokémon, sometimes `FF` fill), so two saves of the same game can differ there while every named field agrees. The
  game is not known to read them (**likely**: no getter covers them). Two named fields that do drift between such
  saves: **+0xB0 friendship** (u16) and **+0xE4 pooled friendship** (u16, getter `pokemonBiosGetPoolFriend`; next to
  the pooled experience at +0xE0, so **likely** the friendship a Shadow Pokémon gathers while still Shadow, handed over
  at purification); both change with walking, so a slightly different walk leaves them one or two apart (**likely**).

### The Strategy Memo's records (section 12; **verified** from the code)

A 16-bit count, then **500 records of 12 bytes**: the species (16-bit; the top bit marks a record the player hasn't
viewed yet, cleared when the memo is opened), two bytes unused, the **catching trainer's id** (32-bit) and the
Pokémon's **personality value** (32-bit). A new game clears every species and fills every personality slot with a
random number (`memoInit`); seeing a new species in battle appends one record with that Pokémon's personality and
trainer id (`memoDataSet`), so the memo can show the individual first seen (its gender, shininess). The lookups walk
the records by species (`memoDataGetPokemonTrainerRndFromID`, `memoDataGetPokemonRndFromID`).

### Comparing two saves 

Two saves of the same play are not equal byte for byte even when the play was identical; these bytes can't match
(**verified** on two saves of the same recorded play session, console and PC port):
- the **memory-card serial** in the header (the card the save was made on);
- **padding** and bytes after a name's terminator (whatever was in memory before);
- each party and box Pokémon slot's **last 0x3C bytes**, past the 0xFC-byte Pokémon record: at save time they hold
  run-time addresses (the console's are 0x80… RAM addresses), so they are working state saved along with the
  record, not part of it (**likely**; pokemon-data.md, "Size discrepancy").
Everything else (story flags, party, bag, memo, mail, Shadow state) should match exactly; the play time matches
when the save is made on the same frame.

The block's first four bytes are the **attestation stamp** (version, generation, area, language; below), single
bytes, so a tool must never swap them as one 32-bit number.

## Battle Mode settings in section 14 (2026-10-16, verified from the menu headers)

+0x00 is the Battle Mode kind (0 Colosseum Battles, 1 Mt. Battle 100: `BATTLEMODE_BATTLEYAMA100` in a retail assert,
2 Gang Battle, 3 Battle Now); +0x08 is the rule (0 any, 1 Lv50, 2 Lv100, 3-5 edited rules).

## Memory card file and the save header

- **Card file:** name `pokemon_colosseum`, created with `CARDCreateAsync(..., 0x60000, ...)`: **0x60000
  bytes (384 KB, 48 card blocks)**, several times the ~113 KB block, so it likely holds more than
  one copy. Region check on load: `regions.md`.
- **Attestation stamp** (`gamedataAttest*`, 4 bytes): byte 0 version id, byte 1 generation id,
  byte 2 area id, byte 3 language-area id. Checked on load (`gamedataAttestCheckValid`). Values written
  for a new save (`hero.c`): version 0xB (Colosseum), generation 3, area/language 1/1 (Japan),
  2/2 (USA), 3/8 (PAL) (`regions.md`). **Every
  Pokémon record carries the same kind of stamp** (`attest[4]` at +0x08 of `PokemonBios`): the
  game, generation, region and language it came from. A Pokémon storage tool must preserve it.
- **Save header** (`gamedatasaveBios*`, section 1): audio option, vibration off, current floor id,
  previous floor id, position index on the floor, play time, save count, memory-card id, and a
  random save id (`Savernd`).

### Card file layout, checksum and encryption (2026-10-14; `field_range_801CB180.c`)

| File offset | Size | Contents |
|---|---|---|
| 0x0000 | 0x6000 | card header area: 0x40 bytes of comment/status, then the banner and icon images copied from `sMemcardBanner` (0x802758E8) (0x1800 + 0x2200 bytes); written as 0x4000 or 0x6000 bytes depending on a flag. (**verified** from the card code's status settings and the sizes) The banner is one 96x32 image in RGB5A3 (0x1800 bytes); the icon is 8 animation frames of 32x32 in 8-bit palette format (C8, 0x400 bytes each) sharing one 0x200-byte palette (0x2200 in all), looping at the middle speed |
| 0x6000 + n x 0x1E000 | 0x1E000 each | **3 save slots** (n = 0, 1, 2), written in 0x2000-byte chunks |

A header chunk is checked with a plain **32-bit word sum**: the sum of words 0..0xE8F must equal
word 0xE90; a mismatch only sets a flag (`raw[0x40]`).

**Save slot (0x1E000 bytes):**

| Slot offset | Field |
|---|---|
| +0x04 | u32 save counter: on load the slot with the highest counter wins (main + backups) |
| +0x08 | the game save block, 0x1DFD0 bytes (copied out to the live save) |
| +0x0C | zeroed before checking (part of the slot header inside the block) |
| +0x18 .. +0x1DFD8 | **encrypted** with a SHA-1 key stream (below) |
| +0x1DFEC | 20-byte **SHA-1 digest** of the plain slot (bytes 0 .. 0x1DFD8) |

**Encryption** (`fn_801CBE44` when saving, `savedataDecryptCheck` (0x801CBCDC) when loading; SHA-1 in `sha1Calc` (0x801CBBAC),
`fn_801CBF64`, `sha1Transform` (0x801CC380)):
- Save: `digest = SHA1(slot[0 .. 0x1DFD8])`; `key = NOT digest` (all five words inverted); then for
  each 20-byte block from +0x18: `block ^= key`, and `key = SHA1(encrypted block)`. The digest is
  stored at +0x1DFEC.
- Load: `key = NOT stored digest`; for each block: `next = SHA1(encrypted block)`, `block ^= key`,
  `key = next`; then the slot is valid only if `SHA1(decrypted slot)` equals the stored digest.
- If no slot verifies, the code repairs the slot header's own small checksum (eight words summing
  to 0, word 3 adjusted) and reports a damaged save.

For a save editor or Pokémon storage tool: decrypt with the rule above, edit the block at +0x08, then
re-encrypt (which recomputes the digest). The first 0x18 bytes of a slot stay unencrypted.

Two more checks every card read makes, both verified by loading converted saves:
- **Every slot's first eight words must sum to zero**, empty slots included (bytes 0 and 1 must be 1; word 1 is the
  save counter). One bad slot header makes the game refuse the whole card ("The game data can't be saved.
  Continue without saving?"), even when another slot holds a good save.
- **The icon file's word sum** (the 0x40-byte title and comment, the 0x1800-byte banner and the 0x2200-byte icon,
  0xE90 words, the checksum word right after): a mismatch makes the game want to format the card.

### Moving a save between a console / emulator and a PC port (2026-10-18)

The save block is one raw byte array copied by value; the game's code reads each field with its own width. A port
that keeps the game's data in its own (little-endian) byte order therefore stores the same save differently, and two
things must be converted (verified: a save moved from an emulator to a PC port, played and saved there, and moved
back, loads in the emulator with the same team, items and story position):
1. **Every 2- and 4-byte field is byte-swapped**, except text. Names (the player name, each Pokémon's trainer name,
   nickname and original nickname, the box names) are byte arrays the code reads as big-endian 16-bit characters, so
   they stay as they are. A table of field widths can be measured rather than typed by hand: compile the game code
   with an instrumented build that reports the address and width of every load and store, play, and keep the save
   block's **reads**. Writes mislead: a new game clears the name fields two bytes at a time.
2. **The seal is redone in the target's word order.** The encryption is byte-wise XOR with SHA-1 digests, so only
   SHA-1's block-word loads, the slot header sums and the icon file sum depend on byte order.

Fields a run never reads are left unconverted and listed, so a converter can say where it is not yet sure.

A port can also do this at the card boundary, so its card file is the console's format and no converter is needed:
swap the fields right after a slot is unsealed and before a slot is sealed, read and write the slot counter and both
sums in big-endian order, and run SHA-1 on big-endian words. Verified 2026-10-19: an emulator save placed unchanged in
the port's card loaded, was played and saved there, and the port's file loaded unchanged in the emulator.

### The "same card" check 

- **The save records the card it was written to** (**verified**, code): at every save the game asks the card for its
  serial number (`CARDGetSerialNo`: an XOR of the card's ID bytes, a property of the physical card) and stores it in
  the save, at bookkeeping +0x08 (two 4-byte words, `gamedatasaveBiosSetMemcardID`), both in the save in memory and in
  the copy it writes.
- **It refuses to overwrite a save from another card** (**verified**, code and a recorded play): before overwriting,
  the save task reads the three slot headers, takes the newest slot and compares the card ID stored in it with the ID
  of the save in memory (`fn_801CF9C8`); if they differ it stops with an error and writes nothing. It is an anti-swap
  check: a card exchanged for another between loading and saving is not overwritten.
- **Mt. Battle's link step restores an older save image** (**verified**, code; **likely** the only place, not
  searched): before Battle Mode's link mode the game copies the whole save, runs the link, then copies it back
  (`gbaConvRunLinkMode`), so the ID in memory goes back to the one the save had before.
- **Why it matters:** a save copied to another card keeps the old card's ID; it loads and saves normally, since the
  card's newest slot and memory agree, until something restores an older image, after which saves can fail the
  check. A port or emulator whose card has no serial (an ID of 0) meets the same situation with every console save.
  

## Flow

Load: the memory card file is read into the block, then checked (`gamedataBiosGetGamedataAtttestPtr`,
attestation). Play: every system reads and writes its section through `savedataGetStatus`.
Save: the block is written back to the card (`gamedatasave*`, `memcard*`).

**Where the player can save** : **only at a PC** (the PC's menu offers saving); the pause menu
saves nothing ([pause-menu.md](pause-menu.md), **verified**). So **no save is possible before the first Pokémon Center,
in Phenac City**: a player who stops earlier has an empty memory card (**likely**: observed in play, consistent with
the code; the PC's save entry itself is not traced here). The game also saves itself in a few scripted places
(after the staff roll, with a yes / no: movies.md, "How the ending reaches the staff roll", **verified**).

## Disc data

None: the save is created at new-game time (`pokemonInitAry`, `sodateyaInit`, `memoInit`,
`exribbonInit`...).

## Changing it

- A save editor can work on this block directly, with the section table above as its map; the
  Pokémon record (0x138 bytes) and the item slots are the most valuable layouts to pin down next.
- Adding data for mods: append a new section rather than resizing one, since many systems compute
  addresses from fixed offsets.
- XD's save almost certainly uses the same mechanism with different sections (`xd-insights.md`).

### Mod surface
- **Data it reads:** the card file (banner, icon, sections), each section's fixed offsets, the checksum and seal.
- **Hooks it would need:** extra data for mods stored **outside** the original save (a side file per save), and a
  save-and-load hook so mod systems save their own state with it.
- **Fixed-size limits:** every section's size; the card file's block count.
- **Must stay vanilla in GameCube-accurate mode:** the save layout, checksums and card behaviour, so a save moves
  between the console, Dolphin and the PC unchanged.
- **The card ID in the save** (bookkeeping +0x08): a save editor that moves a save to another card can leave it as it
  is (the "same card" check compares the save in memory with the card's newest slot, which carry the same ID); setting
  it to a different value than the card's newest slot makes that game refuse to save.

## Efficiency

One allocation, no per-system save code: each system owns a fixed slice. Saving is a single block
write.

## Battle Mode's registered team 

Registering a team in Battle Mode (Register Pokémon → Colosseum Pokémon) **copies** the party into the save: the
copies sit outside the party and the PC boxes (**verified** on a real save: a Lv 50 party member registered earlier
appeared twice, at block +0x199DC and +0x1A4F4, after the party member itself had moved on). Battle Mode then uses the
copy, not the party, until the team is registered again. So a change to the party (levelling, a save edit) only
reaches Battle Mode after registering again. 
The layout is now known (next section); the two copies are the two player records inside the
saved-hero entry.

### Section 14's trainer entries (**verified** from the code and an end-of-story save)

Section 14 (0xCC2C bytes) holds, after its 0x24-byte header (the run words above):
- **four trainer entries** of 0x1660 bytes from section +0x24, one per player of a Battle Mode setup
  (`savedataBattleGetEntryPtr`; computer opponents fill the entries no person plays, **likely**: in an end-of-story
  save the three other entries hold other trainers' teams);
- **a copy of all four** from section +0x59A8 (the setup copies each entry 0x5984 bytes further on before a battle,
  `menuColosseumBattleSetup`; **likely** a "before the battle" state to restore);
- **the saved hero**, one more entry at section +0xB328, with its "present" byte at +0xC988
  (`savedataBattleSetSavedHero` / `GetSavedHero`); the registered team;
- the suspended-run words from +0xC98B (above) and **six 0x54-byte rules** from +0xC9DC (`savedataBattleInit`: three
  fixed rules, then three editable ones whose +6 field starts at 6).

**A trainer entry** (`menuCBBiosCreateTrainerHero`): u16 trainer id at +0, a u16 kept across rebuilds at +2 (the
controller slot, written by the setup), u32 trainer kind at +4, a u32 at +0x24 (the setup's player kind), then **two
complete player records** (the 0xB18-byte structure of save section 2: name, party, bag, ...) at **+0x2C** and
**+0xB44**. The second is the source the next rebuild reads, the first the record the battle uses (**likely**, from the
order of the copies). So a registered Battle Mode team is stored as two whole copies of the player's record, which is
why a registered Pokémon appears twice in the save. Every multi-byte field inside them is the same as in section 2.

**A run in progress and a suspended run** (**verified** from a recorded Mt. Battle run, the
resume code `menuCBResumeYama` and the suspend code `gbaConvCaptureMemo`). Section 14's first words describe the run:
+0x0 and +0x4 (the run kind: 1 / 1 for a Mt. Battle run), +0x8, +0xC (the colosseum: 6 = Mt. Battle, which the resume
asserts), +0x14 **the wins so far**, +0x18 a second counter that starts at 5 and rose by one with every win in the
recorded run (its meaning not settled: continues or a trainer index), and the byte +0x1C (1 while a run is in
progress; cleared when the run ends or a link mode starts). Suspending copies these words to **+0xC98C** with the
"suspended" byte at **+0xC98B**, followed by the player memo: +0x20 the room to resume in, +0x24 the room before it,
+0x28 position (3 floats), +0x34 rotation (3 floats), then the story variables 0xAFC, 0xAFD, 0xB11 and 0xDE1 at +0x40,
+0x44, +0x48, +0x4C. Resuming writes the words back, sets variable 0xB59 from the wins, restores the variables and
puts the player back in the saved room. Starting Mt. Battle again as a new challenge resets the words (wins 0, the
counter 5) and clears the suspended copy: a suspended run is replaced without asking.
(**verified** from the code and a card with a suspended run): The run words are seven 4-byte words
(+0x0 to +0x18), copied whole to +0xC98C; the player memo starts at **+0xC9A8** with its "valid" byte, then two 4-byte
room ids (+0xC9AC, +0xC9B0), six 4-byte floats (+0xC9B4 to +0xC9C8) and the four story values as 4-byte words (+0xC9CC
to +0xC9D8). The card stores all of them big-endian, like every multi-byte field in the save. A save editor that lets
a player change a suspended run (the wins, the room) edits these words; the resume code checks that the colosseum is 6
and the kind is 1 and stops otherwise.

## Open questions

- The 0x40-byte card comment area and the slot header words +0x00, +0x10, +0x14; XD's version id
  and whether XD uses the same SHA-1 scheme.
- The other fields of the section-15 Shadow rows, and the rest of the 12-byte entries before +0x49C (their +0x06 is
  the footstep count, above). Two saves of the same play whose walks differ by a few steps differ in that count's low
  byte only (**verified**: a console save against a PC replay of it).
- Full layout of the 0x138-byte Pokémon record (species, PID, IVs/EVs, moves, Shadow data).
- Region differences in the block (name lengths, text encoding of names).
