# Addresses at a glance (Colosseum USA)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely different**: the same kinds of tables and accessors, at XD's own addresses)

Status: **partial** (2026-11-05). A lookup page, not an explanation: each row says **where** something is and links
to the page that explains **what** it is and its field layout. Only layouts and addresses appear here, never the
data itself (no stats, no per-entry values). Each row is labelled **verified** (backed by the code, the executable's
data or a recorded run), or **likely**.

## How to read it

- **Addresses are USA (GC6E01) RAM addresses** as the game runs on the console or in Dolphin. Japanese and PAL builds
  put things elsewhere (see "Region notes").
- **Most data tables have no fixed address.** They live in one relocatable module, `common_rel` (entry of
  `common.fsys`), which the game copies into a heap block when it loads. Its startup code then writes a **pointer**
  and a **count pointer** for each table into fixed globals in the executable. So the fixed address to remember is
  the **pointer global**; the table itself is wherever that pointer says (**verified**: the module's prolog is 47
  pairs of "global = &table, countGlobal = &count", game-data-tables.md (not yet published)). The count global points at a 32-bit count.
- **The save block moves every boot.** It sits at a random 32-byte offset inside a fixed area, so a save field is
  always "current-save pointer + section offset + field offset", never a fixed address (save-data.md).
- **Names** are the decomp's: real function names where the game's symbols gave them, `lbl_<address>` / `fn_<address>`
  where not; a few proposed names (`gFightFloor`) are given for readability (global-state.md).

## 1. Static data tables in `common_rel`

Source for all rows: `common.fsys`, entry `common_rel` (a REL module, id 0x7D; the tables are its section 4). Each
pointer global is 4 bytes in the executable's small-data area; the count global sits 4 bytes before it. **Size** is
the record size; ✓ means the accessor code confirms it, otherwise it is the gap to the next table divided by the count
(an upper bound, **likely** exact). Counts are the USA module's.

| Table | Pointer / count global | Record | Count | Accessor | Layout | Label |
|---|---|---|---|---|---|---|
| Species | `lbl_80478F94` / `F90` | 0x11C ✓ | 413 | `pokemonDataBiosGetPtr` (0x8011E778) | [pokemon-data.md](../systems/pokemon-data.md) | verified |
| Moves | `waza_data` / `_number` | 0x38 ✓ | 358 | `wazaDataBiosGetPtr` (0x8011CA34) | [pokemon-data.md](../systems/pokemon-data.md) | verified |
| Natures | `pokemon_seikaku_data` / `_number` | 0x28 | 25 | `pokemonSeikakuDataBiosGetPtr` (0x8011CE18) | [battle-turn-flow.md](../systems/battle-turn-flow.md) | verified |
| Nature stat rates | `pokemon_seikaku_rate_data` / `_number` | 2 | 8 | `pokemonSeikakuRateDataBiosGetPtr` | game-data-tables.md (not yet published) | likely |
| Shadow Pokémon definitions | `lbl_80478F6C` / `F68` | 0x38 ✓ | 97 | `fn_801EE10C` and neighbours | [shadow-system.md](../systems/shadow-system.md) | verified |
| Shadow flag list | `lbl_80478F64` / `F60` | 8 | 505 | `darkPokemonGetFlagListValue` (0x801EE544) | [shadow-system.md](../systems/shadow-system.md) | likely |
| Trainers | `lbl_80478F24` / `F20` | 0x34 ✓ | 820 | `fightTrainerDataBiosGetPtr` (0x801FCCC4) | trainers.md (not yet published) | verified |
| Trainer Pokémon | `lbl_80478F14` / `F10` | 0x50 ✓ | 5,510 | `fightTrainerPokemonDataBiosGetPtr` (0x801FCA2C) | trainers.md (not yet published) | verified |
| Trainer classes | `fight_trainer_kind_data` / `_number` | 12 | 41 | `fightTrainerKindDataBiosGetPtr` | trainers.md (not yet published) | likely |
| Trainer AI | `lbl_80478F34` / `F30` | 0x28 | 261 | `fightTrainerAiDataBiosGetPtr` | [battle-turn-flow.md](../systems/battle-turn-flow.md) | verified |
| Trainer AI value adjustments | `fight_trainer_ai_value_addsub_data` / `_number` | 20 | 553 | `fightTrainerAiValueAddsubDataBiosGetPtr` | [battle-turn-flow.md](../systems/battle-turn-flow.md) | likely |
| Battles (encounters) | `fight_encount_data` / `_number` | 0x38 | 566 | `fightEncountDataBiosGetPtr` (0x8020E0F8) | [battle-engine.md](../systems/battle-engine.md), [trainer-encounters.md](../colosseum/trainer-encounters.md) | verified |
| Battle kinds | `fight_kind_data` / `_number` | 0x20 | 19 | `fightKindDataBiosGetPtr` (0x8020E488) | [battle-engine.md](../systems/battle-engine.md) | likely |
| Battle types | `fight_type_data` / `_number` | 8 | 7 | `fightTypeDataBiosGetPtr` | [battle-engine.md](../systems/battle-engine.md) | likely |
| Battle sides | `fight_side_data` / `_number` | 20 | 5 | `fightSideDataBiosGetPtr` | [battle-engine.md](../systems/battle-engine.md) | likely |
| Battle arenas | `fight_floor_data` / `_number` | 24 | 50 | `fightFloorDataBiosGetPtr` | [battle-arenas.md](../systems/battle-arenas.md) | likely |
| Rooms (floors) | `lbl_80478FBC` / `FB8` | 0x4C | 196 | `floorDataBiosGetPtr` (0x80115C48, searches by room id) | game-data-tables.md (not yet published), rooms.md (not yet published) | verified |
| Items placed in rooms | `floor_tresure_list` / `_number` | 28 | 74 | `floorEventGetTresure` | [treasure.md](../colosseum/treasure.md) | verified |
| NPC info | `peopleInfoData` / `_number` | 0x2C | 150 | `peopleInfoBiosGetPtr` (0x8018F6F4) | [npcs.md](../systems/npcs.md) | verified |
| Character names | `char_name` / `_number` | 8 | 607 | `charNameBiosGetNameID` (0x802613D4) | [name-entry.md](../systems/name-entry.md) | verified |
| Menu portraits | `lbl_80478E74` / `E70` | 8 | 830 | `_menuFaceBiosGetPtr` | menus.md (not yet published) | likely |
| Story flag definitions | `lbl_80478F9C` / `F98` | 8 ✓ | 3,593 | `std082_flagGetValue` (0x801906A0) | flags.md (not yet published) | verified |
| Flag storage descriptors | `lbl_80478EEC` / `EE8` | 8 | 4 | `std082_flagGetValue`, `GSflagClear` | flags.md (not yet published) | verified |
| Sound effects | `snd_data` / `_number` | 12 | 1,236 | `GSsndGetStatus` | music.md (not yet published) | likely |
| Reverb presets | `lbl_80478FA4` / `FA0` | 24 | 12 | `_sndSetReverbParm` | music.md (not yet published) | likely |
| Menu sound groups | `menu_se` / `_number` | 10 | 8 | `menuSeBiosGetPtr` | menus.md (not yet published) | verified |
| Text colours | `lbl_80478E8C` / `E88` | 4 | 6 | `msgctrlPalette` | [messages.md](../systems/messages.md) | likely |

The whole block of pointer globals runs 0x80478DF8-0x80478FBC (**verified**). Smaller tables (the world-map
destinations, cursor links, sound waves, resource lists and a few still unexplained) are listed in
game-data-tables.md (not yet published).

**Per-room tables.** Each room's NPC placements (0x24 bytes each), spawn points, camera list and lighting are in the
room's own small module (the type-14 entry of the room's `.fsys`). Its startup code fills four three-word slots in a
block of the executable at 0x80478FC0-0x8047A260; the room's record (+0x10 to +0x1C) points at them (**verified**,
game-data-tables.md (not yet published) "Floor (room) entries").

## 2. Static tables in the executable

These are compiled into `main.dol`, so they have fixed addresses.

| Table | Address | Size | Record | Count | Accessor | Layout | Label |
|---|---|---|---|---|---|---|---|
| Item records | `lbl_80363CE8` | 0x3E08 | 0x28 | 397 records (`lbl_80478BB0`) | `itemDataBiosGetPtr` (0x801440A0) | game-data-tables.md (not yet published) | verified |
| Item id → record map | `lbl_803681E8` | 0x448 | u16 | 548 item ids (`lbl_80478BD8`) | same | game-data-tables.md (not yet published) | verified |
| Poké Ball presentation | `item_ball_data` | 0x270 | 0x30 | 13 (`item_ball_data_number`) | `itemBallDataBiosGetPtr` (0x80143DA0) | game-data-tables.md (not yet published) | verified |
| TM / HM moves | `item_waza_machine_no` | 0x1D0 | 8 (`{u32, u32 move}`) | 58 by size (count `item_waza_machine_no_number`) | `itemDataBiosGetWazaMachineNo` | [pokemon-data.md](../systems/pokemon-data.md) | likely |
| Menus (window definitions) | `_menu_data` | 0x1DC0 | 0x1C | 272 by size | `menuDataBiosGetPtr` (0x8005DA18) | menus.md (not yet published) | verified |
| Menu items | `lbl_802EF0A8` | 0x20EF8 | 0x1C | 4,818 by size | `menuItemBiosGetPtr` | menus.md (not yet published) | likely |
| Menu sprites | `_menu_sprite` | | 0x18 | | `menuSpriteBiosGetPtr` | menus.md (not yet published) | verified |
| Window open / close sequences | `_menu_seq` | | 0x0C | | `menuSeqBiosGetPtr` | menus.md (not yet published) | verified |
| Battle action kinds | `fight_action_kind_data` | | 12 | 20 | the action queue | [battle-turn-flow.md](../systems/battle-turn-flow.md) | verified |
| Move-script command table | `lbl_8027A00C` | | 4 (a handler) | 255 | the move-script reader | [battle-engine.md](../systems/battle-engine.md) | verified |
| Weather records | `tenkou_data` | | 24 | | end-of-turn weather | [battle-turn-flow.md](../systems/battle-turn-flow.md) | verified |
| Model variant overrides | `lbl_80374F20` | | | 35 species | `wazaSequencePokemonCreate` | [pokemon-data.md](../systems/pokemon-data.md) | verified |

"By size" counts are the symbol's size divided by the record size (**likely**: the symbol may hold padding).

## 3. Text

Text has no global table: each room archive and each menu archive carries its own **message bank** (a type-5
entry), and the shared banks (`fight`, `tool_fight`, `system_tool`, `pocket_menu`, ...) load with their archives.
Loaded banks form a linked list; a message key is `bank id << 20 | entry id`, looked up by `GSmsgGetGSchar`
(0x800FA280). The message system record is `lbl_804024E8` (0x2C), reached through `_pMw` (**verified**,
messages.md).

## 4. Runtime structures

### The save (while the game runs)

| What | Address / where | Size | Layout | Label |
|---|---|---|---|---|
| Save area | `pokecolo_savedata` (0x80408400) | 0x1E7D0 | the save block sits at a random 32-byte offset inside it | verified |
| Current-save pointer | `now_savedata_ptr` (0x8047ADB8) | 4 | read by `savedataBiosGetNowSavedataPtr` (0x80128E24), set by `savedataBiosSetNowSavedataPtr` | verified |
| Section getter | `savedataGetStatus(save, section)` (0x80129280) | | 17 sections; NULL save = the current one | verified |
| Player (section 2) | save + 0x70 | 0xB18 | name, party, bag, money, badges ([save-data.md](../systems/save-data.md)) | verified |
| **Party** | save + 0x70 + 0x30 | 6 × 0x138 | the 0x138-byte Pokémon record ([pokemon-data.md](../systems/pokemon-data.md)) | verified |
| **PC** (section 3) | save + 0xB88 | 0x7198 | 3 boxes × (0x14 header + 30 × 0x138), then 235 item slots × 4 ([save-data.md](../systems/save-data.md)) | verified |
| Mail (10) | save + 0x7D20 | 0x448 | [save-data.md](../systems/save-data.md) | verified |
| Daycare (11) | save + 0x8168 | 0x140 | 8-byte header, one 0x138 record | verified |
| Strategy Memo (12) | save + 0x82A8 | 0x1774 | [pda.md](../systems/pda.md) | verified |
| Card-e (13) | save + 0x9A1C | 0x49CC | [card-e.md](../colosseum/card-e.md) | verified |
| Battle Mode state (14) | save + 0xE3E8 | 0xCC2C | [save-data.md](../systems/save-data.md) | verified |
| Shadow Pokémon state (15) | save + 0x1B014 | 0xDA8 | [shadow-system.md](../systems/shadow-system.md) | verified |
| Story-flag buffers (4, 5, 6) | save + 0x1BDBC / 0x1BDDC / 0x1BE5C | 0x20 / 0x80 / 0x600 | flags.md (not yet published) | verified |
| Ribbons (16) | save + 0x1C45C | | the last section | verified |

The section offsets are the same on the memory card once the block is decrypted (save-data.md has the card layout
and the encryption).

### Battle

| What | Address / where | Size | Layout | Label |
|---|---|---|---|---|
| The battle object (`gFightFloor`) | `lbl_8046DD90` | 0xA4E8 | statuses through `fightFloorGetStatus` (0x801F54A4) / `SetStatus` (0x801F4C14) ([battle-turn-flow.md](../systems/battle-turn-flow.md)) | verified |
| Side *s* (0 player, 1 opponent) | floor + 0x14 + *s* × 0x5230 | 0x5230 | `fightFloorBiosGetFightSidePtr` | verified |
| Trainer *t* of a side | side + 0x64 + *t* × 0x28E4 | 0x28E4 | `fightSideBiosGetFightTrainerPtr` | verified |
| Field Pokémon *i* of a trainer | trainer + 0x1314 + *i* × 0x6E0 | 0x6E0 | stat stages, conditions ([battle-turn-flow.md](../systems/battle-turn-flow.md)) | verified |
| Field-slot pointer array | floor + 0xA4C4 | 8 × 4 | re-sorted during the battle | verified |
| Attacker / target of the current move | floor + 0xA474 / + 0xA478 | 4 each | | verified |
| Current battle id / result | `lbl_8047B5D6` / `lbl_8047B5D8` | 2 each | | verified |
| Move-script cursor | `lbl_8047B610` | 4 | [battle-engine.md](../systems/battle-engine.md) | verified |

### Random numbers

| What | Address | Size | Label |
|---|---|---|---|
| The one shared random state | `lbl_80478C90` | 4 | verified |
| Pointer to the state (HSD's seed pointer) | `lbl_80478C94` | 4 | verified |
| The generator | `HSD_Rand` (0x801ADCD8) | | verified |
| Front ends | `GSrand` (0x800E0C54), `GSrandf` (0x800E0BE4), `HSD_Randi` (0x801ADC3C), `HSD_Randf` (0x801ADC7C) | | verified |
| Seeding (once, at graphics start-up, from the clock) | `fn_800E0C78` | | verified |
| Shiny test / PID creation | `pokemonCheckRare` (0x80125390), `pokemonCreateRndFit` (0x80124410) | | verified |

The rule and every consumer are in [rng.md](rng.md).

### Field, menus, memory and engine

| What | Address | Size | Layout | Label |
|---|---|---|---|---|
| Current room (a stack) | `lbl_8047ACC8` | 4 (pointer) | [menu-rooms.md](../systems/menu-rooms.md) | verified |
| Room-change state | `_floorChangeInfo` | 0x54 | [room-changes.md](../systems/room-changes.md) | verified |
| Player movement work | `HeroMove` | 0x420 | [player-movement.md](../systems/player-movement.md) | verified |
| NPC array | `lbl_8047B200` | 4 (pointer to a heap pool) | [npcs.md](../systems/npcs.md) | verified |
| Window system (the open-window list, head at +0x0C) | `winSysWork` | 0x9C | menus.md (not yet published), [message-windows.md](../systems/message-windows.md) | verified (size likely) |
| Each open window (in that list) | heap records, next at +0x10 | | +0x04 window id, +0x94 / +0x95 cursor, +0x9C message id of the battle box (menus.md (not yet published)) | likely |
| Menu stack | `lbl_803B6D88` | 8 entries × 8 | menus.md (not yet published) | verified |
| Menu cursor memory | `lbl_80404A98` | 12 × 2 | menus.md (not yet published) | verified |
| Memory-card task (placed at a random offset) | `_memcardStatusPtr` | 4 (pointer) | [memory-card.md](../systems/memory-card.md) | verified |
| Script VM pool | `lbl_80478B00` | 4 (pointer to `lbl_80401BF8`) | [script-vm.md](../systems/script-vm.md) | verified |
| Collision system | `GScolsys2` | 0x3710 | [collision.md](collision.md) | verified |
| Graphics state | `gsgfxState` | 4 (pointer to 0x5A0) | [gfx.md](gfx.md) | verified |
| Heap arena (`GSmem`) | 0x8048E740-0x817F0560 | about 19.4 MB | [memory.md](memory.md) | verified (Dolphin) |
| Heap free list / entry table top / bottom | `lbl_8047AB30` / `lbl_8047AB34` / `lbl_8047AB38` | 4 each | [memory.md](memory.md) | verified |

More globals, grouped by system, are in [global-state.md](global-state.md).

## 5. Main accessor functions

The game almost never touches a record's bytes directly: it asks an accessor with a **status id** (a selector). Using
these, not raw offsets, is the safe way to read or change anything.

| Function | Address | What it does | Label |
|---|---|---|---|
| `pokemonGetStatus(pokemon, 0, id, 0)` | 0x8012640C | reads any field of a Pokémon record by selector | verified |
| `pokemonSetStatus` | 0x801254B4 | writes one, same selectors | verified |
| `pokemonDataBiosGetPtr(species)` | 0x8011E778 | species record | verified |
| `wazaDataBiosGetPtr(move)` | 0x8011CA34 | move record (an out-of-range id gives entry 0) | verified |
| `itemDataBiosGetPtr(item)` | 0x801440A0 | item record (NULL for ids without one) | verified |
| `fightTrainerDataBiosGetPtr` / `fightTrainerPokemonDataBiosGetPtr` | 0x801FCCC4 / 0x801FCA2C | trainer / trainer Pokémon record | verified |
| `fightEncountDataBiosGetPtr` | 0x8020E0F8 | battle record | verified |
| `floorDataBiosGetPtr(roomId)` | 0x80115C48 | room record, searched by room id | verified |
| `peopleInfoBiosGetPtr` | 0x8018F6F4 | NPC info record | verified |
| `heroGetStatus` / `heroSetStatus` | 0x8012A5B0 / 0x8012A450 | the player's save fields (party slot, money, coupons, ...) | verified |
| `heroBiosGetPokemonPtr` | 0x8012AC08 | a party Pokémon | verified |
| `savedataGetStatus(save, section)` | 0x80129280 | a save section | verified |
| `fightFloorGetStatus` / `fightFloorSetStatus` | 0x801F54A4 / 0x801F4C14 | battle state by status id | verified |
| `fightFloorBiosGetFightFloorPtr` | 0x801F6B48 | the battle object | verified |
| `std082_flagGetValue` / `std081_flagCheckTrue` | 0x801906A0 / 0x801902E0 | story flags | verified |
| `GSmsgGetGSchar(key)` | 0x800FA280 | a message's text | verified |
| `GSmemAlloc` / `GSmemLock` / `GSmemFree` | 0x800E2C04 / 0x800E27B0 / 0x800E209C | the heap | verified |
| `menuDataBiosGetPtr` | 0x8005DA18 | a menu definition | verified |
| `fn_8017F484` | 0x8017F484 | loads a type-14 module (`common_rel`, room modules) and runs its prolog | verified |

The full Pokémon selector list is in [pokemon-data.md](../systems/pokemon-data.md); battle status ids in
[battle-turn-flow.md](../systems/battle-turn-flow.md).

## Changing it

- **Read through the pointer globals and accessors, not fixed addresses.** Tables in `common_rel` move with the
  heap, and the save block moves every boot.
- A cheat code or memory watch for a save field must follow `now_savedata_ptr` (0x8047ADB8, the current-save pointer) first.
- A port keeps the same tables but may lay them out anywhere; tools written against this page should go through the
  names, not the numbers.

### Mod surface

- **Data records read:** every table in sections 1 and 2 (record sizes and counts above).
- **Hooks needed:** none for reading; a mod that adds records (more species, moves, trainers) needs the count globals
  raised and the tables moved to bigger blocks, which a port can do when it loads `common_rel`.
- **Fixed sizes:** the counts above, the 0x138-byte Pokémon record, 6 party slots, 3 PC boxes of 30, the 0xA4E8-byte
  battle object (2 sides × 2 trainers × 2 field Pokémon).
- **Stays vanilla in GameCube-accurate mode:** every layout on this page (saves must stay readable by the original
  game).

## Efficiency

The data tables are one module loaded once and kept, so every lookup is a pointer plus an index, with no searching
except for rooms (found by key). Battle state is one preallocated object, so a battle allocates almost nothing while
it runs.

## Region notes

All addresses are for **USA (GC6E01)**. The Japanese and PAL builds have different code and data placement, so
every address here is expected to differ there (**likely**; not checked). The **record layouts and the save section
offsets are likely the same** across regions except where text length matters (names, message banks); PAL adds
languages, which changes the text banks. Counts in `common_rel` may differ by region (not checked). See
[regions.md](../colosseum/regions.md).

## Open questions

- The window system record at `winSysWork` is **likely** just its 0x9C bytes. The fields read
  further out (+0x94, +0x95, +0x9C) are on each window record in its list, not on the system record. The decompiled
  code reads the system record only at +0x01, +0x0C and +0x10; the functions still in assembly are not checked.
- Exact counts for the "by size" rows (TM table, menu items).
- The XD addresses, once XD is studied.
