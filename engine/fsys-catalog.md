# FSYS archive catalog (USA disc, GC6E01)

Status: **partial** (2026-10-14). Every `.fsys` archive on the disc: its name, what it contains,
what it seems to be for, and how we know. Names and descriptions only, no game data. The per-archive listings
that used to close this page are now a summary (counts and naming patterns, "Archive counts" at the end); a full
listing can be regenerated from any copy of the disc by reading the archive headers.

Format of the archives themselves: [disc-files.md](disc-files.md). Resource types:
floor-resources.md (not yet published).

**Evidence labels:** *name* = from the archive or entry names; *contents* = from the entry types;
*code* = from the decomp; *guess* = a reasonable reading that nothing confirms yet. A blank cell
means unknown.

## How the game finds an archive

- The disc has **1,852** `.fsys` files. `GSfsys.toc` (loaded whole at FSYS init,
  `gs_range_8017A5FC.c`, path string `lbl_80273F70` = "gsfsys.toc") lists **1,835** of them as
  `{archive id, name offset}` pairs (header: "GLLA" magic, 0x104, count 0x72B at +0x08, pair table
  offset at +0x10). *code*
- Code always asks for an archive by **numeric id**. `_fsysGetFilename` (`fsys_file.c`) scans the
  toc for the id and builds the path with `"%s.fsys"`. The same id is stored at +0x08 of the
  archive's own header (common = 0x0A, field_common = 0x0B, fight_common = 0x0C, ...). *code + disc*
- So the **17 archives not in the toc can never be loaded** (all `wzx_`; 10 of them are empty, with
  misspelled names such as `wzx_awa_attck` next to the real `wzx_awa_attack`): leftovers from
  development. *disc*
- Inside an archive, an entry is found by its **entry id** (entry +0x00, compared with the
  request id in `fn_8017E30C`). Entry ids are global, not per archive: the same model
  (`ken_b1`, id 0x00F7) has the same id in every archive that carries a copy. *disc*

## Entry types

The type code at entry +0x20 is the index into the floor resource-type table `lbl_8036C2A0`
(callbacks per type). The contents agree with the callback names for every type that has them.
*code + contents*

| Code | Kind (callbacks) | Entries | Examples | Evidence |
|---|---|---|---|---|
| 0 | generic data (`floorReadNormalPreFunc`) | 47 | MusyX `snd_*_proj/pool/sdir`, `card01_bin`..., `key` | contents |
| 1 | map model (`floorReadMap*`; HSD, geometry: [hsd-models.md](hsd-models.md)) | 178 | one per room, named after the room | name + size |
| 2 | object / NPC model (`floorReadObj*`; HSD, [hsd-models.md](hsd-models.md)) | 1,100 | `ippan_m_b1`, `t_box_00`, `ball_master` | name |
| 3 | collision (`floorReadCol*`) | 124 | one per walkable room | name |
| 4 | BGM sequence (`floorReadBGM*`; MusyX song, music.md (not yet published) "Song file format") | 78 | `battle2_song`, all in `bgm_archive` | name |
| 5 | message bank (`floorReadMsg*`) | 168 | room name, `fight`, `system_tool`, `pocket_menu` | name; [messages.md](messages.md) |
| 6 | font (`floorReadFont*`) | 162 | paired with each message bank; `jis_1` (648 KB) | name; JIS = Japanese kanji font (guess) |
| 7 | script (`floorReadScript*`) | 175 | one per room or menu | name; [script-vm.md](script-vm.md) |
| 9 | texture (`floorReadTex*`) | 1,487 | `menu_*`, `body001`, `face001`, logos | name |
| 10 | particles (`floorReadParticle*`; GPT1 banks, [particles.md](particles.md)) | 52 | `*_bg_effect`, `*_ptl`, `*_pati` | name |
| 12 | camera (`floorReadCamera*`) | 160 | `cam_*`, `*_01` | name |
| 14 | REL module, linked and its prolog run (`fn_8017F3F8` / `fn_8017F484`, priority 0x64) | 169 | `common_rel` (1.3 MB), `waza_viewer_rel`, one ~1 KB module per room | code: `OSLink` + prolog; game-data-tables.md (not yet published) |
| 15 | PKX: Pokémon / trainer battle model (`floorReadPKXPreFunc`) | 527 | one per `pkx_` archive | name |
| 16 | WZX: move effect (`floorReadWZXPreFunc`; format [wzx.md](wzx.md)) | 1,098 | one per `wzx_` archive | name |
| 17 | GFL = lens flare (`floorReadGFL*` -> `flareInit`; [lens-flare.md](lens-flare.md)) | 1 | `T1_ancient_colo` | |
| 18 | particles, not linked (`floorReadNotLinkedParticle*`; GPT1 banks, [particles.md](particles.md)) | 2 | `koori_loop`, `nemuri_loop` (freeze / sleep loops) | name |
| 19 | REL code module (`floorReadRelModule*`) | 1 | `debugMenuData` | name |

Types 8, 11 and 13 have no callbacks and appear in no archive.

## Shared archives

### common (id 0x0A, 32 entries)

Loaded for the whole game (guess: at boot; to confirm from the caller that requests id 0x0A).

| Entry | Type | What it is |
|---|---|---|
| `common_rel` | 14 | **The game's data tables** (species, moves, trainers, trainer Pokémon, Shadow definitions, battles, floors, treasure, NPC info, story flags, sounds: 47 tables). Its prolog fills the table pointers in the executable. Confirmed from its relocations: game-data-tables.md (not yet published). |
| `mail` | 14 | small data module, presumably mail designs or text (name) |
| `snd_music_*`, `snd_se_*`, `snd_se_battle_*`, `snd_se_motion_*`, `snd_se_nakigoe*_*`, `snd_music_atmos_*` (21) | 0 | MusyX project / pool / sample-directory files for each sound group; the samples are the separate `sound/*.samp` files. *nakigoe* = Pokémon cries. |
| `menu003`, `menu004`, `menu005`, `menu_009`, `menu_012`, `MENU_044`, `gsvtr`, `wipe_ball00/01` | 9 | menu textures and the Poké Ball screen wipes (name) |

### field_common (id 0x0B, 7 entries)

Overworld (field) assets used in every map: `ken_b1` and `hizuki_b1` (object models; the player and
Rui, guess from *hizuki* and the player model being in every field archive), `bbook`, `bikkuri00`
("!" surprise icon; *bikkuri* = surprise), `kirari` (sparkle), `t_box_00` (treasure box), and texture
`menu_040`.

### fight_common (id 0x0C, 7 entries)

Battle assets: message banks and fonts `fight` and `tool_fight`, camera `cam_party_a_1`, textures
`menu_048` and `menu001`. The battle UI text lives here (name).

### Other shared or global archives

| Archive | Id | Contents | Purpose (evidence) |
|---|---|---|---|
| people_archive | 0x009a | 157 object models | every NPC / trainer overworld model (name: *people* = NPCs; names match the models in room archives) |
| bgm_archive | 0x0099 | 78 BGM sequences | all music (name) |
| poke_body | 0x048a | 826 textures | `bodyNNN` and `bodyNNN_c`: Pokémon full-body pictures, probably normal and alternate (shiny?) per species (guess) |
| poke_face | 0x05c0 | 418 textures | `faceNNN`: Pokémon face portraits (`pokemon-data.md` (not yet published), portraits) |
| chara_big / chara_small | 0x05c3 / 0x05c4 | 88 / 10 textures | character portraits (name) |
| camera_shake_data | 0x0634 | 9 type-2 entries | shake presets short/medium/long × weak/medium/strong (name) |
| waza_sequence_global | 0x06f7 | 2 not-linked particles | sleep and freeze loops used by any move sequence (name) |
| menu_common, fight_demo, prog_memcard, save_menu | 0x0097, 0x0010, 0x001c, 0x001a | **empty** | placeholders in the toc with no entries |

## Menus and screens

| Archive | Contents | Purpose (evidence) |
|---|---|---|
| title | script, msg, font, map, model, 2 cams, 20 tex (17 of them `t_vs_*`, *likely*) | title screen (name); the `t_vs_*` textures are the title screen's character pictures (below) |
| topmenu | msg, font, 2 models, cam, 5 tex | main menu after the title (name) |
| name_entry_menu, pcbox_name_entry_menu | script, msg, font, models, tex | name entry (name); the pcbox copy is stored uncompressed |
| pocket_menu, pcbox_pocket_menu | rel?, script, msg, font, tex | bag (*pocket*) (name) |
| pcbox_menu, pcbox_poke_box | script, msgs, fonts, model `savor`, tex | PC box (name) |
| pda_menu | rel?, script, msg, font, map, models, cams, tex | the P★DA (in-game device) menu (name) |
| waza_menu | script, msg, font, tex | move menu (name: *waza* = move) |
| world_map | rel?, script, msg, font, map, models, 15 cams `cam_move_demo_*`, tex | world map and the bike travel scenes (name; `bike_*` models) |
| evolution | map | evolution scene (name) |
| colosseumbattle_menu, colosseumbattleconnection_menu, toolbattle_menu | menus, msgs, fonts, tex | menus for Colosseum-mode battles, link battles, and quick battles (name; guess for *toolbattle*) |
| pokemonchange_menu | `poke_exchange_*`, `p_exchange` | trading (name) |
| pokecoupon_menu, ex_coupon | | Poké Coupon exchange (name); `ex_coupon` is Bonus Disc content (verified from the bonus-disc images) |
| carde_menu, M1_carde_1 | `card01_bin`...`card21_bin`, `card_e`, `jis_1` | e-Reader battle cards (*Card e*; a Japan-only feature, *likely*: the US game has no way into the Card e room, regions.md) |
| ex_shrine, ex_title, ex_top | `kakutyou1/2` (*kakuchō* = expansion), `cerebi_dat`, `relive_gpt` | **Bonus Disc content** (all `ex_` archives; verified from the bonus-disc images), including a Celebi event at the shrine (Celebi: name) |
| DNA_key | map `dnakey`, 2 cams | |
| debug_menu | REL module `debugMenuData` | debug menu (name) |
| waza_viewer | rel `waza_viewer_rel`, `M1_water_colo` map | developer move-animation viewer (name; [regions.md](regions.md) build kind 2) |
| genius_logo, nintendo_logo | one texture each | boot logos (name) |
| pokemon_logo, opening_demo, auto_demo, staffroll | one generic entry `dummy` | placeholders for screens that play `.thp` movies instead ([movies.md](movies.md); guess). (verified) The `dummy` entry in all four is the same 97,704-byte file, a Unix `ar` static library of ELF objects: the developers' own movie library (`GSAPI\GSmovie`: GSmovie, THPRead, THPDraw, THPPlayer, THPAudioDecode, THPVideoDecode, THPDec, THPStats, THPAudio; 66 exported names), probably used as a filler file when the archives were built. A slightly different build than the one in main.dol; its names give the movie player's original function names |
| keydisc | 32-byte `key` | |
| mail000..002 | one texture each | mail stationery (name) |

### The title screen's pictures (*verified*)

The 17 `t_vs_*` textures are pre-rendered 2D character pictures, placed as menu items (the common data table near
0x802E6428: screen position, texture id 0x100C1200-0x101C1200, item number 0x330-0x33E / 0x9F). The title shows one
set at a time: the page counter `lbl_80478878` (0-3) goes up by one each time the title screen starts
(`fn_8002049C`), and `fn_8001F304` shows only that page's items (menus.md (not yet published), "The title screen's art").
The counter starts at 3, so the first title after power-on shows page 0 (main.dol's initial value, and a fresh boot
in Dolphin). Not random, not save-dependent.

| Page | Set | Pictures |
|---|---|---|
| 0 (first after power-on) | c | Espeon, Umbreon, Wes, Rui |
| 1 | b | the final starters: Typhlosion, Meganium, Feraligatr, Swampert, Blaziken, Sceptile |
| 2 | a | the legendaries: Raikou, Entei, Suicune, Kyogre, Groudon |
| 3 | d | the GBA protagonists Brendan and May (the page has 4 menu items; the other two are not identified) |

## Maps (rooms)

Room archives are named `<area><n>_<place>[_<floor>][_<variant>]`. A full room archive carries,
in order: a type-14 module, its script, message bank and font, the map model, its NPC and object
models, collision, and optional particles and cameras.

**Area names** match the community sources, The Cutting Room Floor's Pokémon Colosseum
page and the gc-forever "Pokemon Colosseum (GCN/WIIRD/PAL)" thread. The game's floor table agrees
with the grouping: each room's entry has an **area byte** (`floorDataBiosGetArea`, entry +0x01 in
the floor table, game-data-tables.md (not yet published)), and every prefix maps to one area (**verified**).

| Prefix | Area byte | Area | Place words in the names | Status |
|---|---|---|---|---|
| S1 | 1 | Outskirt Stand | out, shop | verified |
| M1 | 2 | Phenac City | gym, stadium, water_colo, carde, pc, shop, house | verified |
| M2 | 3 | Pyrite Town (includes Pyrite Cave) | building, cave, earth_colo, guild, police, uranai (fortune-teller), windmill, hotel | verified |
| M3 | 4 | Agate Village | shrine, cave, house, pc, shop | verified |
| M4 | 5 | The Under (includes the Under subway) | bottom_colo, cylinder_colo, subway, junk, kanpo, labo, train, hotel | verified |
| D1 | 6 | Shadow Pokémon Lab | garage, labo | verified |
| D2 | 7 | Mt. Battle | cloud, crater_colo, magma, valley, rest_1..9, present_room, pc | verified |
| D3 | | (none on the disc) | | **skipped**: probably a dungeon cut during development (*guess*). In XD the D3 prefix is reused for the S.S. Libra (xd-insights.md (not yet published)). |
| D4 | 8 | Realgam Tower | tower, dome_1..11, casino_colo | verified |
| S2 | 10 | Team Snagem Hideout | bossroom, building, hallway, snatchroom, out | verified |
| T1 | 9 | a stand-alone Colosseum battle arena (Orre Colosseum most likely) | ancient_colo | decided from the code, see below |

**T1 (`T1_ancient_colo`, floor 124).** The two sources disagree (Orre Colosseum vs. the title
screen), so this follows the game's data:
- Its floor entry's class byte (+0x00) is **0x40**, the same as the other Colosseum battle rooms
  (`M1_water_colo`, `M2_earth_colo`, `M4_bottom_colo`, `M4_cylinder_colo`); normal field rooms are
  0x20, menus 0xA0, and the title screen 0xC0.
- Its area byte is 9, the area shared by menus and system screens, because it belongs to no town.
- Its archive holds only a battle script (`fight_common_scd`), the map, collision, and a GFL entry:
  no NPCs, no messages, nothing a title screen needs.
- The title screen is a separate floor (114) with its own archive, `title`, which has the menu
  script, logo demo and `t_vs_*` textures.
So T1 is a **Colosseum battle arena**, not the title screen. Which Colosseum is still to be
confirmed from the battle tables (Orre Colosseum fits "ancient" and the area-free placement).

Floor class byte values seen (+0x00 of a floor entry): 0x20 field room, 0x40 Colosseum arena or
`_bf` variant, 0x80 the move viewer, 0xA0 menu / special screen (and some S2 rooms), 0xC0 title,
top menu and Colosseum-battle menu. Meanings are inferred from which rooms share a value.

Variant suffixes: **`_bf`** = the **battle-field** version of a room: a cut-down copy (module,
script, map, collision) loaded as the stage when a battle starts there. Confirmed by the
battle-floor table, which points every overworld battle at a `_bf` room ([battle-arenas.md](battle-arenas.md)). **`_ev_N`** = event versions (script + map). **`_rl`** = (one room,
`M3_shrine_1F_rl`). Colosseum rooms: `*_colo` (water, earth, bottom, cylinder, crater, casino,
ancient), matching the seven `wzx_colo_name_*` effects that are not in the toc.

## Battle models and move effects

- **`pkx_<name>` (527 archives):** one PKX entry each. Names are romanized Japanese species names
  (`achamo` = Torchic, `absol`) and trainer / character battle models (`agent_m_a1`, `hizuki_a1`).
  Species pick their model through `pkxDataId` (`pokemon-data.md` (not yet published)).
- **`wzx_<move>_<part>` (1,108 archives, 1,091 loadable):** move animations split into parts
  (`_attack`, `_damage`, `_special`, `_sp1`, `_status`, ...), named after the romanized Japanese
  move names (`10manvolt` = Thunderbolt).

## Open questions

- What each room's small type-14 module registers.
- Which code requests each shared archive id, and when (boot, battle start, floor change).
- Which Colosseum T1 is (no story battle uses it: [battle-arenas.md](battle-arenas.md)).
- The 17 unloadable `wzx_` archives: unused effects or a different lookup path.

## Archive counts (USA disc)

A summary in place of the generated per-archive listings (*verified*: counted from the archive headers).

- **Room archives by area prefix:** D1 13, D2 29, D4 26, M1 17, M2 26, M3 15, M4 27, S1 3, S2 10, T1 1. Most rooms
  have a full archive (module, script, message bank and font, map, collision, NPC and object models, optional
  particles and cameras); `_bf` and `_ev_N` variants carry only a module and / or script and a map. The busiest room
  archive, `D2_cloud_1`, carries 24 NPC models.
- **PKX archives: 527**, one type-15 entry each. 449 have plain names (species, romanized Japanese, including the 35
  `rare_*` shiny models, the Unown letter forms `unknown_*` and the Castform forms `powalen_*`); 78 have a numbered or
  gender suffix (`_a1`, `_f_a1`, ...): trainer and character battle models (*likely*, from the names).
- **WZX archives: 1,108**, one type-16 entry each. Suffix counts: `_attack` 306, `_damage` 281, `_special` 79,
  `_sp1` 74, `_status` 23, `_damage2` 14, `_out` 13, `_downin` 12, `_in` 12, `_open` 12, `_pikachu` 10, `_damage3` 10.
- **Empty archives: 14.** Four are in the toc (`fight_demo`, `menu_common`, `prog_memcard`, `save_menu`:
  placeholders); ten are `wzx_` archives outside the toc, several with misspelled names (`wzx_awa_attck`,
  `wzx_haradaiko_attck`).
- **Not in the toc: 17**, so the game can never request them: the ten empty `wzx_` archives above and the seven
  `wzx_colo_name_*` effects (one per Colosseum: ancient, bottom, casino, crater, cylinder, earth, water).
