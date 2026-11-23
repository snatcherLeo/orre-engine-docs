# Region and build differences

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** . The code read is the USA build's (GC6E01); the European (GC6P01) and
Japanese (GC6J01) discs and saves are compared with it below, from their files and real saves.

## What it does

The game asks two tiny functions which build it is, and branches on the answers. Both return a
constant in this build, so the region logic for other builds is still present in the US executable.

| Function | Returns (USA) | Meaning (evidence) |
|---|---|---|
| `GSgetVersionMajor` (0x800057A0, `src/game/main.c`) | **1** | **Region.** 0 = Japan, 1 = USA, 2 = **PAL** (confirmed by the 50 Hz particle compensation below). Drives the disc and memory-card game codes, the save stamp, and a few content differences. |
| `GSgetVersionMinor` (0x800057A8) | **6** | **Build kind.** 2 = a developer build: move-animation sequence errors are reported (`wazaViewerSetErrorFlag` (0x801D744C), wazaSequenceEntry.c); 4 = a build that rewrites the mounted disc id to "GC6J" before card access; 6 = this retail build. |

Call sites (35): wazaSequenceEntry (13), wazaViewer (4), wazaSequenceCamera (4), the memory-card
code in field_range_801CB180 / field_exact_801CFD08 (5), dbgMenu (3), gs_dvd (2), floor (2), hero,
field_range_80117E58.

(likely) The first room at boot depends on the build kind; kind 4 boots `ex_title`
(likely an extra-disc build: it also skips the disc-id check above).
(verified from the disc images) Build kind 4 is the **Japanese extra disc** ("Nintendo
SPECIAL DISC", a Colosseum build whose executable reports region 1 and build kind 4, with the `ex_` rooms). The US
bonus disc is a different program (a Jirachi download disc), so the `ex_` rooms and their English text on the US main
disc are leftovers of the Japanese extra disc (bonus-discs.md (not yet published)).

## Memory card file (region-dependent part)

- File name **`pokemon_colosseum`**; the card comment is "POKéMON COLOSSEUM" / "SAVE FILE"
  (0x802792E8, a compiler-generated constant, Latin-1 `é`).
- On load, the game scans up to 127 card files and accepts the one whose name matches and whose
  game code matches the region: "GC6J" for region 0, "GC6E" for regions 1 and 2; company code "01".
  So **a Japanese save is rejected by a US game** and vice versa: a cross-region save editor or port
  needs to rewrite the card file's game code (the text needs no conversion: it is the same format in every region,
  below).
- **The save file's header per region** (**verified** against four real saves, one per region and
  language pair): the 0x6000-byte header holds a 0x1800-byte banner, a 0x2200-byte icon, a title and a comment, with a
  word sum after them. Each region's executable keeps these as data and one function copies them in when a save is
  made. **The icon is the same in all three regions; the banner is the same in the US and Europe** and differs in
  Japan (a katakana logo on a sandstone plate); Japan's title and comment are Shift-JIS. **The European game picks the
  comment by its Options language** from a small table of (language, text) pairs, so the header's word sum differs by
  language. A tool that writes a save for
  another region takes that region's banner, title and comment, and redoes the word sum.

## Everything that branches on the region (USA executable)

| Where | Region 0 (Japan) | Region 1 (USA) | Region 2 (PAL) |
|---|---|---|---|
| Disc id at DVD init (`gs_dvd.c`; skipped when build kind = 4) | "GC6J" | "GC6E" | "GC6E" (as compiled here) |
| Memory-card file game code (`field_exact_801CFD08.c`) | "GC6J" | "GC6E" | "GC6E" |
| New save's attestation stamp (`hero.c`, `gamedataCreate(save, ver, gen, area, lang)`) | 0xB, 3, **1, 1** | 0xB, 3, **2, 2** | 0xB, 3, **3, 8** as compiled in the US executable (real PAL saves record the chosen language, below) |
| Particles (`field_range_80117E58.c`) | | | one extra particle update every 5th frame: **50 Hz compensation** |
| Map models (`floor.c`) | | in 15 rooms (ids 3, 0xB-0xE, 0x15, 0x24-0x26, 0x72, 0x75, 0x7B, 0xCA, 0xD0, 0xDE) texture animations are linked to the model animation and given a rate, in the US build only | |

So the attestation stamp is {version 0xB = Colosseum, generation 3, area, language}; a Japanese
save says area 1 / language 1, a US save 2 / 2, and a real PAL save area 3 with **the language the player chose**:
2 for English, 6 for Spanish (**verified** from real European saves, below). The 3 / 8 in the table is only what the
US executable has compiled in for region 2.

### What real saves from each region look like (**verified**)

Four matched saves, each a new game saved right after the first snag: US, PAL in English, PAL in Spanish, and
Japanese. All four were made in an emulator and decoded with the US game's own save code.
- **The same container everywhere:** the same card file name (`pokemon_colosseum`), the same size (a 0x6000 header
  then three slots of 0x1E000) and the same cipher; all four decrypt and pass the US code's checks. **Only the card
  file's game code differs** (GC6E, GC6P, GC6J), with maker code 01 in every region.
- **The card header area** (the first 0x6000 bytes): US and PAL English are byte for byte the same, with the same
  banner, icon and comment ("POKéMON COLOSSEUM" / "SAVE FILE"). PAL Spanish differs only in the comment's second line
  ("Archivo"). The Japanese save has a Shift-JIS comment ("ポケモンコロシアム" / "セーブファイル") and its own banner;
  its icon is the same as the others' (the bytes compared, as above).
- **The save layout is identical.** Every difference between the four is either random per new game (IDs, personality
  values, the Strategy Memo's keys) or one of these region fields:

| Where in the save block | US | PAL English | PAL Spanish | Japan |
|---|---|---|---|---|
| +0x02 / +0x03: the save's area / language | 2 / 2 | 3 / 2 | 3 / 6 | 1 / 1 |
| +0x2B and +0xB48: one byte each, the language again | 0 | 2 | 6 | 0 |
| each Pokémon's origin stamp | 0B 03 02 02 | 0B 03 03 02 | 0B 03 03 06 | 0B 03 01 01 |

- **A save file's first copy is written before the story starts** (**verified** in the PAL
  Spanish save): when the game creates its card file it writes a first slot with no play time and no trainer yet, whose
  stamp still holds the default language (English, 2), while +0x2B already holds the Options language (Spanish, 6).
  The player's real save goes into the next slot with every language byte Spanish. So a reader must take the **newest
  valid slot** (the highest save counter at slot +0x04); an older slot can look like another language.
- **All text is UTF-16 (big-endian) in every region.** Japanese is plain kana, for example a player named "レオ" and a
  first box called "ボックス1". Box names are written in the save's language when the game starts: "BOX1" in US and
  PAL English, "CAJA 1" in PAL Spanish.
- **A Pokémon's name field holds its species name in the save's language:** the Japanese save has "ブラッキー",
  "エーフィ" and "マクノシタ"; PAL Spanish keeps the English names (Umbreon, Espeon, Makuhita).
- **Rui's name** (block +0xB32, next to her flag and item pocket) is **the name chosen for her on the naming screen**, stored
  as UTF-16 text like the player's own name, so a preset name is stored as its text, not as a number. **Each language's
  naming screen offers its own preset names for Rui** (verified: the four saves used presets, one per disc and
  language: "ANCA" US, "RUI" PAL English, "CARLA" PAL Spanish, "チハル" Japanese).

**What the US code does with these bytes** (**verified** from the code unless marked):
- **No region comparison on stamps:** `gamedataAttestCheckValid` only requires all four stamp bytes to be non-zero,
  and it is used only as part of "is this Pokémon record valid". No caller checks the save block's own stamp (in the
  code read). So European and Japanese stamps pass in the US game (**likely**; the card loading path wasn't traced).
- **New Pokémon take the save's area and language:** every Pokémon created later (caught, received) gets the save's
  area and language bytes in its own origin stamp (`hero_pokemon.c`, through `gamedataGetStatus`). So the save's stamp
  decides what every later catch is marked with.
- **The save's language drives text conversion** to and from the GBA games (language 1 means the Japanese text
  encoding; `pokeconv.c`, `gba_misc.c`) and **the Card-e region check** (`cardesavedata.c`), which turns a card's
  region number into a language: 1 → 1, 2 → 2, 3 → 4, 4 → 5, 5 → 3, 7 → 6.
- **+0x2B and +0xB48** have no named field or reader found so far.

So moving a save between regions needs only **the card file's game code changed**. Keeping the save's area and
language bytes as they are lets later catches, GBA trades and Card-e behave as on that region's console, and **each
Pokémon's origin stamp stays as it is**, since it records where that Pokémon came from.

## The European and Japanese discs: languages, text, fonts, word pictures

Every claim below is **verified** from the three discs' files and the PAL executable's code unless marked.

- **The European (PAL) disc has five languages: English, German, French, Italian, Spanish** (no Dutch). Each message
  bank's header carries a language tag in bytes 6-7 (`UK`, `GE`, `FR`, `IT`, `SP`; the US banks say `US`, the
  Japanese ones carry none). Every room archive holds all five banks (the US disc: one per room); the shared module
  data and the main executable hold five copies of their banks; 52 extra archives (`<name>_fr`, `_ge`, `_it`, `_sp`)
  hold the per-language menus and pictures, and 76 round-title and arena-intro sequences come per language.
- **How the PAL game picks one:** `GSgetVersionMajor` returns 2; at boot the game reads the console's language
  (`OSGetLanguage`) and maps English, German, French, Spanish, Italian to its own numbers (anything else, Dutch too,
  falls back to English); a code switch then picks the language's archives. Its **Options screen has a Language row**
  (the PAL options picture sheet carries a "Language" label and a cycling arrow; the US sheet has neither), and the
  save keeps the language: **verified** from real European saves (the save's language bytes hold the language chosen,
  English 2 and Spanish 6; "What real saves from each region look like", above), and Continue refuses a story whose
  language differs from the Options row (**verified**, "The European program's language setting", below).
- **The Japanese disc is Japanese only** (`GSgetVersionMajor` 0, no per-language files).
- **The text lines up message for message:**

  | Language | Messages | Same message IDs as the US game | Missing | Extra |
  |---|---|---|---|---|
  | English (US disc) | 11,647 | 11,647 | 0 | 0 |
  | English (UK), German, French, Italian, Spanish (PAL, each) | 11,673 | 11,647 | 0 | 26 |
  | Japanese | 9,414 | 9,414 | 2,233 | 0 |

  The PAL extras are its own menus' lines (the 50 / 60 Hz choice, the language choice and the five names, "the
  language as saved", level-label variants, an untranslated debug list). Japanese lacks the developer-label bank
  (2,219 entries) and 14 lines its fonts or features don't have (number templates for fonts 3 and 4, the Latin
  name-entry rows, two GBA messages). So a translation of the US game's text can follow the same message IDs, and the
  European languages map one to one.
- **UK English vs US English** : of the 11,647 shared messages, 755 differ: 458 only in their
  formatting codes, 243 are left empty in the UK banks (unused lines, some of them Japanese leftovers on the US disc),
  and 54 change wording, mostly small (a full-width "＄" made a plain "$", Japanese leftovers given English, and one move
  name that differs: the US banks say "SOLID PLANT", the UK banks "FRENZY PLANT"). So the two are the same English for a player (**verified**
  counts; the reading of the 54 from a sample).
- **Fonts:** the PAL fonts are byte-identical to the US ones for every shared letter (all Latin letters and kana), so
  what is known about the US fonts (an MS PGothic-style face) holds for PAL. PAL adds a few signs (° γ ○ ★ ♪ ， ～;
  γ is drawn as a superscript "er", used in Spanish "1.er") and a baseline German low quote („), and drops the e-Reader
  marks. The Japanese disc has fonts 0-2 only (fonts 3 and 4 are a US / PAL addition, **likely**); its kana are
  pixel-identical to the US ones, but its Latin letters are a different, wider design (no bitmaps in common), so not
  the US face.
- **Pictures with words:** 26 picture names change per PAL language (the battle HUD sheet with HP, Lv and the status
  words, options, title, P★DA, PC and battle menus; French uses "PV", "N." and "SOM / GEL / BRU"), plus the per-language
  round and arena title sequences; 30 differ on the Japanese disc (the same menu sheets and a few more). Pictures with
  English words that are identical on every disc can't be found by comparing files: still open.
- **What those pictures are** (**verified** by comparing every menu picture of the three discs
  and looking at each one that differs): the menus are **the same layouts in every region**: the same frames, panels,
  icons, colours and controller art. What changes is **the words drawn into the pictures**:
  - **the battle panel's sheet:** HP and Lv (French PV and N.), EXP, the status badges (SLP, PSN, PAR, FRZ, BRN, FNT,
    PKRS), "Waiting...", "Time Left" and "Choice Time";
  - **the type badges** (GRASS, FIRE, WATER ... and ???) and the **status and contest badges** of the summary;
  - **the summary's labels** (PROFILE, ABILITY, STATS, RIBBONS, LEARNED MOVES, HEART GAUGE, ITEM, BATTLE EFFECT...),
    "PP" and "MOVE TYPE";
  - **a menu picture with four labels** (Pokémon, Save, Items, Exit): not the pause menu, which draws its labels as
    text (pause-menu.md); which menu it belongs to is open;
  - the P★DA's mail headings (Subject, Sender)
    and "PKMN SEEN";
  - **the main menu** (STORY MODE: CONTINUE, NEW GAME; BATTLE MODE: COLOSSEUM BATTLE, BATTLE NOW; OPTIONS) and **the
    Options screen's row labels** (PAL adds Language);
  - **the title screen's "PRESS START"** and its copyright lines;
  - **the battle-mode menus** (Battle with 2 / 4, the mode names list, "Quit", "NO DATA", "Champ!", 1P-4P, the
    Game Boy Advance connection labels, socket names), **the results** (WIN, LOSE, DRAW and the battle-place names),
    Cancel / Check, ITEM / MOVES, and "LOADING".
  The European disc keeps the UK English pictures the same as the US ones, apart from adding the Language row; the
  French, German, Italian and Spanish copies each redraw the words. A few long words simply don't fit and are
  shortened or set smaller.
- **How the European disc stores its languages' pictures** (**verified** on the three discs):
  each language has **its own copies of 13 menu archives** (one per language, side by side), not one archive with
  every language inside. Across them **26 pictures differ** from the English ones; **6 change size** and some change
  their picture format. Each translated picture has **its own texture number**, and the European program holds a
  **per-language sprite table**: for each on-screen piece, the rectangle and the texture number to use in each
  language. So switching a picture's language means switching the sprite's record (where it's cut from and how big)
  **and** the texture it looks up, not only the picture itself.
- **Where each piece sits: the European and Japanese programs' own item tables** (**verified**
  by measuring against the discs in Dolphin, with a port that reads these tables matching to the pixel): the menus'
  item tables (each on-screen piece's place and flags) are **not** identical across regions. The European program has
  an **alignment setting** in the low 4 bits of each item's flag byte:
  - **0:** the picture is stretched to fill the item's box (the US behaviour);
  - **1:** centred at the picture's own size;
  - **2:** anchored bottom-left;
  - **3:** anchored bottom-right.

  98 items use 1 and 15 use 2. So a translated word picture that changed size sits where its own language's game
  puts it, not stretched into the English box. This covers the results screen's WIN / LOSE words, the double-battle
  headers and Italian's CONFERMA, and the main menu's **cursor glove**, which sits further left so it clears longer
  words like FORTSETZEN or CONTINUER. The Japanese program's tables differ in the same way; about 200 bag and PC items
  use a different Japanese layout.
- **The stadium name on the VS and results screens is drawn in code** (**verified**: a port
  following this layout matches the European disc to the pixel in German and French): not placed through the item
  table. A one-piece name is centred on the 640-pixel screen; a two-piece name is drawn as two pictures 8 pixels
  apart, with the "stadium" word first in German, French, Italian and Spanish (STADION PHENAC, COLOSSEUM DE PYRITE,
  COLOSSEUM PYRITUS), the reverse of English's order.
- **The European program's language setting** (**verified**, from its code): the language is a
  number from 2 to 6 (English, German, French, Italian, Spanish), kept in the save's header as the Options language. A story save is also stamped with the language it was played in, and **Continue
  refuses a story whose stamp differs from the Options language** (verified). So, **likely**, changing the language
  on a European console means the old story can't be continued until it's changed back.
- **Smaller European differences in text** (**verified** by eye against the disc in Dolphin):
  the P★DA's rows sit tighter and further left; the trainer card's money has no thousands separators; and the
  Pokémon summary shows the species name stored in the save as written (seen with an English save: the names stayed
  English), rather than a translated name.
- **Frames that widen around longer words** (**likely**): in places such as the bag's label pills, the European game
  widens the frame around a longer word, so a port that keeps the US frames can let long labels overflow.
- **The Japanese disc's pictures** (**verified**): 30 differ from the US ones; 23 are words, the
  other 7 are the differences above (the Nintendo logo, Rui's outfit, the marbled background, button pictures) and
  ribbon pictures. Somek's VS-screen portrait differs too; it sits in another archive (below). Also seen: the Japanese title screen's copyright line reads **2003**, and its main menu card shows a
  **different picture of Wes**. The European disc's Options sheet already has its own **"Language"** label.
- **Differences that aren't words** (**verified** by eye, Japanese disc only): the boot screen's Nintendo logo is
  **blue** in Japan and red elsewhere; **Rui's outfit differs in the title picture of her standing** (`t_vs_c4`, a
  145 x 231 picture in the title archive, shown on the title and main-menu screens; the Japanese one has a shorter
  top showing her midriff, the US and European ones cover it; verified by comparing the pictures). **All four of her 3D
  models differ too, the same way** (verified by comparing every archive entry named after her, "hizuki", on both
  discs): the body texture's strip above the skirt is her purple top on the US disc and skin on the Japanese one, plus
  some model data. The models and where the game keeps each copy:

| Model | What it is | Copies the game uses |
|---|---|---|
| `hizuki_a1` | battle model | the battle model archive `pkx_hizuki_a1.fsys`, and a copy among the room objects of `M1_out.fsys` |
| `hizuki_b1` | overworld model (following Wes) | `field_common.fsys` (a 128 x 128 body texture) |
| `hizuki_b2` | scene model | `M1_out.fsys` |
| `bike_hizuki` | riding the bike | `world_map.fsys` |

  Each battle-model copy is the same size unpacked as its counterpart, with the same animation slots, colours and
  header; of `hizuki_a1`'s 15 textures exactly one differs, the 256 x 256 body texture.
- **Leftover Japanese copies on the US disc** (verified): the US disc's `people_archive.fsys` still holds the
  **Japanese (midriff) versions** of `hizuki_a1`, `hizuki_b2` and `bike_hizuki`, byte for byte the same as the
  Japanese disc's own `people_archive` copies: unedited leftovers next to the edited copies the game uses. Whether
  the US game ever loads them from `people_archive` hasn't been traced. The overworld model `hizuki_b1` exists only in
  `field_common`, so it has no leftover.
- **The other non-word differences** (verified by eye, Japanese disc only): one menu background (a grey marbled
  pattern) is a different, cloudier picture, and a few button pictures differ slightly. The European disc's pictures differ from the US ones only in words.
- **The seven non-word Japanese pictures, one by one** (**verified** by decoding both discs'
  copies and comparing them pixel by pixel; every one keeps its name and texture id on both discs):

| Picture | Where | What it shows; how the Japanese one differs |
|---|---|---|
| `logo_nintendo` | embedded in the main executable (also `nintendo_logo.fsys`, an identical copy) | the boot logo, 640 x 480: **blue** in Japan, red in the US and Europe |
| `t_vs_c4` | `title.fsys` | Rui from the waist up: in Japan she smiles and her shorter top shows her midriff; the US one has a neutral face and a top down to a white belt |
| `chara_bigl074` | `chara_big.fsys` | **Somek's** VS-screen portrait (the Mt. Battle master; an older blond man in a white fur cape): **a moustache in Japan**, clean-shaven and smiling in the US (*identified 2026-11-20*, below) |
| `menu004` | `common.fsys` | the grey tiling menu background: marbled in the US, cloudier in Japan, a different picture in a different format (grey-only in Japan) |
| `menu_024` | four menu archives (the bag, the PC box, the party swap, the move menus) | the ribbon and medal sheet: **the same artwork**, stored at full colour depth in Japan (the US copy uses 15-bit colour) |
| `menu_025` | the P★DA menu | a mint Poké Ball outline icon: the same artwork, the same colour-depth difference |
| `menu_040` | `field_common.fsys` | a metal bracket frame with three rows and a teal plate: the same artwork, the same colour-depth difference |

  So only **four** look different (the logo, Rui's picture, the trainer portrait and the background); the "button"
  differences are the colour depth of three sheets (no pixel more than 64 of 255 apart, most within 31).
- **How the boot logo is drawn** (**verified**): `movieBootStart` opens menu 0x85, whose first item draws sprite record
  0x1B9 of the main executable's sprite table, a full-screen 640 x 480 rectangle using texture id 0x06F71200, which
  `menuInit` registers from the executable's own embedded picture. The Japanese executable registers its blue logo
  under the same id, and its sprite record is the same rectangle: only the picture changes. The European logo is the
  US one.
- **Rui's models: which are the same size** (**verified**, unpacked sizes compared): `hizuki_a1` is the same size on
  both discs (both copies); `hizuki_b1` is 172 bytes smaller in Japan; `hizuki_b2` (21,856 bytes) and `bike_hizuki`
  (1,256 bytes) are larger in Japan. The leftover Japanese copies on the US disc are byte for byte the Japanese
  disc's, so the US team shrank `hizuki_b2` and `bike_hizuki` when they covered her midriff (likely). (**verified** from runs): The archive loader sizes its buffers from
  each entry's own record inside the archive (unpacked size at +0x08, stored size at +0x14; disc-files.md), so an entry
  of another size loads without any change to the game's code, and all four Japanese models were seen on screen in the
  US game.
- **Where Rui's models load** (**verified** from runs): in Phenac City (room 0x3, `M1_out`), people
  group 31 holds the sack (`bag_b1`, index 0) and `hizuki_b2` (index 1), and group 0, index 101, is `hizuki_b1`. The
  battle model archive `pkx_hizuki_a1` is loaded **only for battles with a Shadow Pokémon** (for her "look" sequence
  that spots it); a Cipher battle with no Shadow Pokémon loaded no Rui archive.
- **Somek, the Mt. Battle master, lost his moustache in the US release** (**verified** by decoding
  both discs and comparing them, unless marked). He is trainer 317 (class MT.BTLMASTER; Japanese name バトラス), the
  100th and last trainer of the story's Mt. Battle climb, met in the crater at the top (room 0x5F; battle 319 in the
  crater's colosseum). Three things show him, and all three have the moustache only on the Japanese disc:

| What | Entry and archive | How the US copy differs |
|---|---|---|
| his VS-screen portrait | `chara_bigl074` in `chara_big.fsys` (200 x 150, RGBA8) | the same picture with the moustache painted out: 394 of its 30,000 pixels differ, all around the mouth |
| his battle model | `niku_m_a2` in `pkx_niku_m_a2.fsys` (trainer-model index 59) | the moustache is extra geometry on the face, which the US model leaves out: 37 triangles fewer (1,462 against 1,499; the face part is split into two meshes in the US model, one in the Japanese); 576 bytes smaller unpacked. The textures are the same pictures |
| his model standing in the crater | `niku_m_b2` in `D2_crater_colo.fsys` | the moustache is a small mesh of its own (33 triangles), left out of the US model (4 meshes and 767 triangles against 5 and 800); 2,196 bytes smaller; the textures the same |

  - **A leftover on the US disc:** `people_archive.fsys`'s copy of `niku_m_b2` is the **Japanese** model with the
    moustache, byte for byte the same as on the Japanese disc (both discs' copies are identical), next to the edited
    copy the crater uses. The same kind of leftover as Rui's. There is no leftover of the portrait or the battle model.
  - **Where the portrait would show** (**likely**, from the data): the `toolbattle_menu` module, the room that runs
    Battle Mode's battles, holds a table of 75 records of 0x14 bytes keyed 1 to 75, one per **trainer-model index**
    (the trainer record's +0x10; the player models 1, 2, 3, 10 and 11 are the ones with a second, right-facing
    portrait, which fits the player's side of the VS screen); key 59, Somek's battle model, gives `chara_bigl074`.
    But **no Battle Mode opponent uses his model** (none of the 296 trainers in the course and Mt. Battle opponent
    order; Battle Mode's 100th Mt. Battle opponent is another MT.BTLMASTER, trainer 774, an old man's model), and the
    story's climb doesn't go through that room (its battle 319 starts from the crater and is fought in room 0xE4;
    verified: neither a recorded play of that battle nor a PC run of it ever enters room 0x397). So in normal play his portrait is **likely never on screen**, in either release.
  - **The two models are seen in play** (verified in a PC run of the story climb's end, *2026-11-20*): the crater's
    model when Wes reaches the top and in the talk after the battle (the battle's field room 0xE4 loads the crater's
    archive again), the battle model in his trainer intro, behind his Pokémon, and in his bow after losing.
  - **Everywhere else he could appear, checked against the Japanese disc** (verified; every archive entry on both discs
    compared by name, and the two videos sampled once a second): his small character picture (`chara_small`) and the
    other `niku_m` models (other bodybuilders) are identical; he has no cutscene model (no other entry of his on either
    disc), no menu picture (**likely**: the Strategy Memo lists Pokémon only, and Mt. Battle's registration is a text
    menu), and he isn't in the credits or the title-screen demo videos (**likely**,
    from the one-a-second samples). So the portrait and the two models are the only places his look differs.
  - **Other look differences the same full comparison turned up** (verified as different): the
    Teacher trainer's models (`sensei_b1` in Mt. Battle's areas and Phenac's gym, `sensei_a1`), the Jynx and Jumpluff
    battle models (`pkx_rougela`, `pkx_watacco`), the `kirari` entry in `field_common`, and the title-screen demo video
    (`autodemo01.thp`). Each is described in "Smaller differences on the
    Japanese disc" below (the demo video turned out to be a separate recording, with several shots framed
    differently, not one changed shot).
- **Loaded archive entries are cached in auxiliary memory** (**verified** from the code): every archive's loaded entries
  are copied to the console's auxiliary memory (`fsysCacheEntry`), and a repeat load is served from there with no disc
  read and no decompression. Anything that changes an entry after loading has to change it before that copy is made,
  or also where the cached copy is served.
- **For a multi-language port** (**likely**): the layouts change only a little (the item tables' alignment, the
  code-drawn stadium header and the widened frames above, which come per language too); the word pictures need a version per language: redrawn, taken from the player's own disc of that region, or replaced by a texture pack.
  Everything drawn as text (dialogue, names, menus' item lists, descriptions) follows the font and the message banks.
- **The Options screen, as an example of the mix** : its title "OPTIONS", the values STEREO, MONO,
  ON, OFF and the help lines are ordinary messages drawn with the font (a font change or a translation changes them);
  the row labels Sound, Rumble and Done (and PAL's Language), the speakers and the controller are one picture sheet,
  words drawn in (a translation needs another sheet, as PAL ships per language).
- **PAL's language names** are written in their own language and in capitals, the same in every language's bank:
  ENGLISH, DEUTSCH, FRANÇAIS, ESPAÑOL, ITALIANO, in that order.

**For a multi-language port:** the game's own text needs no new keys, only a bank per language with the same IDs; the
fonts already hold the European letters; the word pictures need a version per language.

### Smaller differences on the Japanese disc

Besides Rui's clothes and Somek's moustache, five more things look different on the Japanese disc. Everything below is
**verified** from both discs' data unless marked. The **European disc carries the US version of all five** (each entry
and the video byte for byte the same as the US disc's), so these are changes made for the international release. None
of them changes how the game plays.

- **The Teacher's skirt is longer in the US.** The trainer class TEACHER (a woman in a yellow suit with glasses, holding
  a pointer) has a battle model, `sensei_a1` (trainer-model index 40), and a field model, `sensei_b1` (NPC info 27).
  - **Both models:** the skirt's hem sits about 1.2 model units lower in the US (in the field model it goes from 7.80
    to 6.63, on a figure 17.2 units tall), so the skirt is about half again as long: it ends just above the knee, where
    the Japanese one ends at mid-thigh. In the field model the skirt is a small mesh of its own (16 triangles in the
    US, 27 in Japan) and the body mesh around the hips is reshaped to meet it (462 triangles against 456); the battle
    model keeps its triangle count (1,575) and moves the vertices of the hips and legs. The textures are identical.
  - **Smaller changes that don't show:** in both models the two joints at the tops of the thighs lose their skin-bone
    flag in the US; in the field model the joint above them gets rotation keys in all seven animations, all zero (no
    change on screen; **likely** a side effect of exporting the model again); in the battle model the head and hair
    sit 0.06 units higher (too small to see), and a hidden helper joint (the throwing hand's attach point in the ball-throw
    slot) has another resting position, which no animation leaves in place (every animation keys it).
  - **Where she appears:** 21 trainers use the battle model: two in the Phenac Stadium battles (Holen, Glais), six on
    the story's Mt. Battle climb (Nevie, Vona, Loden, Kinson, Leine, Nacie) and 13 in Battle Mode. The field model is
    in the archives of nine Mt. Battle areas and Phenac's Pre Gym (`M1_gym_1F`); 15 people placements use it (14 on
    Mt. Battle, one in the Pre Gym).
  - **A leftover on the US disc:** the copy of `sensei_b1` in `people_archive.fsys` is the Japanese model (shorter
    skirt), byte for byte the Japanese disc's, next to the ten edited copies the rooms carry: the same kind of leftover
    as Rui's and Somek's.
  - **Seen in play** (verified in PC runs with each disc's model, *2026-11-20*): the field model on Mt. Battle (Leine on
    platform 51 of area 6) and the battle model in Nacie's battle (her trainer intro and behind her Pokémon). In the
    trainer intro the whole picture also moves by less than a pixel between the two models (**likely** because that
    camera frames her by her model's size, which the skirt changes); everywhere else only her own pixels differ.
  - **Why (likely):** the same kind of modesty edit as Rui's covered midriff, made for the international release.
- **Jynx's face is a lighter purple in the US** (`rougela`, the battle model in `pkx_rougela.fsys`).
  - **What shows:** the face is coloured by a tiny one-colour texture (4 x 4): a bright violet on the US disc, a dark
    greyish plum on the Japanese one. The other 15 textures and every visible mesh are the same.
  - **What doesn't show:** 44 joints have other resting values (every animation keys all of them, so the resting pose
    is never on screen); the eyes' resting texture offset picks a closed eye in the US and an open one in Japan, but
    all 11 animations animate that offset, identically on both discs.
  - **One motion differs:** animation 1, the one slot 1 plays (the ranged-attack motion: **likely**), moves both arms
    differently from about frame 12 to 65 of its 139 (up to about 50 degrees at frame 42: the Japanese arms are
    stretched out sideways there, the US arms bent). Her other 10 animations are identical.
  - **Seen in play** (verified in a PC run with each disc's model, *2026-11-20*): in the send-out and in her moves the
    US face is a light violet, the Japanese one a dark plum. The small face picture in the battle box is the **same on
    both discs** (dark in both: the full comparison found no picture difference for it), so the US release recoloured
    only the 3D model.
  - **Pictures and effect unchanged** (verified, an entry-by-entry comparison of the two discs, *2026-11-22*): every
    mug shot (`poke_face`, 418 entries) and every PC picture (`poke_body`, 826 entries) is byte-identical between the US
    and Japanese discs, so Jynx's mug shots and PC pictures were never recoloured for the US; her Lovely Kiss effect file
    is identical too. Of everything Jynx uses, only the battle model archive differs.
  - **Where:** no trainer in the game has a Jynx (trainer data); it appears only on a player's own team (**likely**
    brought in by trade from a Game Boy Advance game). The move effects have a Jynx-only version of Lovely Kiss.
  - **Why:** not settled; the face colour likely follows how Jynx was shown
    outside Japan at the time.
- **Jumpluff's cotton puffs lost an inner layer in the US** (`watacco`, the battle model in `pkx_watacco.fsys`).
  - **What shows:** each of its three puffs is a flat picture (64 x 64) facing the camera. The Japanese model adds a
    second, smaller one (32 x 32, a softer cotton picture) inside each puff, each on a joint of its own: 3 joints, 3
    two-triangle quads and 2 textures more (39 joints and 776 triangles, against 36 and 770 in the US). Everything
    else, animations included, is the same once those three joints are left out of the count.
  - **A side effect in the US data:** the attach points (the 16 joints per animation slot that a move's effects
    follow) were renumbered for the removed joints in only some slots. In slots 5, 6, 7 and 9 the tail-tip point
    names joint 37, past the US model's 36 joints (so it has no joint), and several hand and foot points name the
    Japanese numbers, which are neighbouring joints in the US model. The points moves use most (the body centre and
    the mouth) come before the removed joints and are right. The US file also has 17 animation slots, the Japanese
    one 16 (the extra slot is empty). **No vanilla move effect uses a mismatched point in the slot it plays**
    (verified over every move-effect file): In slots 5, 6 and 9 none uses those points (Bounce uses only the root), and in slot 7 only the Snag Ball throws do, which play on Wes's model. Two effects would (Outrage's first
    part, slot 2, the hands and feet; one species version of Grudge, slot 0, a rear foot), but Jumpluff learns
    neither (**likely**), so the slip **likely never shows**.
  - **Seen in play** (a PC run with each disc's model, *2026-11-20*): the Japanese inner layer draws as a flatter,
    greyer disc inside each puff with a crisp white rim, the US puff fluffier all through; over consecutive frames
    (idle, the send-out, mid-Bounce) the layers draw in the same order every frame, with no flicker (verified from
    frame dumps).
  - **Where:** story trainers use Jumpluff on Mt. Battle, in the Snagem hideout, in The Under, in the Deep Colosseum
    and in Realgam Tower, and Battle Mode trainers too; the player can get one by evolving the Shadow Skiploom.
  - **Why:** not known.
- **The ground sparkle is bigger and glows in the US** (`kirari` in `field_common.fsys`, the look of items lying in
  the field: 23 of the US item records use it, treasure.md).
  - The US model adds a flat round glow on the ground under it (a 32-triangle disc with a blue-to-dark picture,
    slightly see-through) that grows over the first 18 frames and stays. Its three star bursts grow larger (to about
    0.6-0.7 of their modelled size at their peak, against 0.45-0.52) and last a few frames longer (gone after frames
    15-19, against 10-14); the loop is 42 frames, against 39. The Japanese model has no texture at all; the file is
    4,643 bytes against 2,503.
  - **Why:** not settled.
- **The title-screen demo video was recorded again** (`movie/autodemo01.thp`).
  - Both are 2,700 frames (1:30) with the same scenes in the same order, and the first 11 seconds are the same. After
    that the Japanese video is a **separate recording** of the same demo: Japanese text boxes, small timing
    differences, and other numbers in the battles (the Pokémon's maximum HP differ: Umbreon has 133 in the US video
    and 129 in the Japanese one), so its battles were played with Pokémon of other stats.
  - **Shots framed differently:** Wes and Rui chasing a woman in a pink gown along a walkway (about 0:24-0:27; **likely**
    Venus fleeing in The Under: a side view in the US, from behind and above, down a narrow bridge, in Japan); the
    stairs up to a domed building (about 0:33; the Japanese camera is farther back and shows the dome); Wes's throw in
    the next battle (about 0:34, framed closer in Japan); and a lift going down (about 0:43-0:45, **likely** into The
    Under: a wide side view in the US, from above in Japan).
  - **Why (likely):** the battles needed English text, so the demo was recorded again in the English game.

- **Where `menu004` shows** (**verified on screen** for the party menu): the game never draws it as
  a background of its own. It blends it as a moving pattern over a **Shadow Pokémon's panel while that Pokémon is in
  Hyper Mode** (the party menu's panel; the battle status box takes the same path in the code, **likely**, not seen
  yet), so the Japanese look's cloudier pattern only appears on a Pokémon in Hyper Mode.


### Everything else that differs on the European disc

**Verified** from the files unless marked.
- **Files:** the US disc has 1,872 files and the European one 2,006. 1,869 are shared, 184 of which differ.
  - **Only on the European disc (137; the groups below add to 136, one file not yet placed in a group):**
    - 52 per-language menu archives (`<menu>_fr/_ge/_it/_sp.fsys`: field / fight common, title, top menu, P★DA, PC,
      bag, moves, Battle Mode menus);
    - 76 Battle Mode title sequences per language: rounds 1-6, semi-final, final, their "scenario" versions and seven
      arena intros (`wzx_<lang>_battle_name_*`, `wzx_<lang>_colo_intro_*`);
    - five `colbtl_<e/f/g/i/s>.bin` (about 92 KB each; **likely** Battle Mode's per-language trainer and team names);
    - the renamed Battle Mode menus (`cb_menu`, `cb_connect_menu`, `pokechg_menu`).
  - **Only on the US disc:** `colosseumbattle_menu`, `colosseumbattleconnection_menu` and `pokemonchange_menu` (the
    same menus under their US names).
- **Inside archives on both discs, by resource type:**

  | Type | What | Difference |
  |---|---|---|
  | 5 | message banks | each European archive holds five language banks (704 more) |
  | 6 | fonts | 68 differ (a few added signs), 5 US-only (the e-Reader marks) |
  | 9 | textures | 5 differ, 9 European-only, 12 US-only (the word pictures, some moved between archives) |
  | 1 | map model | the title screen's `logo_demo` (twice) |
  | 2 | model | `ken_b1` in the US title archive only |
  | 12 | camera | 2 US-only |
  | 7 | script | **two differ:** Realgam Tower's Colosseum (`D4_casino_colo`, 32 bytes shorter) and Realgam Tower 2F (`D4_tower_2F`, same size, different contents) |
  | 14 | REL modules | 167 differ (rebuilt with the European program) |
  | 19 | debug data | 1 |

- **Outside archives:** the credits movie `staffroll.thp`, the disc banner `opening.bnr` and the file table `GSfsys.toc`.
- **main.dol:** 3,779,808 bytes on the US disc, 4,096,576 on the European one (five copies of its message bank, the
  language and 50 / 60 Hz code).
- **What the two script differences are** is not studied yet.
- **For a port that keeps the US program** (**likely**): what changes with the language is all content: text, word
  pictures, Battle Mode's title sequences and data, the credits movie, the title logo demo, and a few font signs. The
  program, the room modules and the two scripts are the European version's own code.

## Text and fonts

Message banks and fonts are disc data loaded per floor (`floorReadMsgPreFunc`,
`floorReadFontPreFunc`, `messages.md`), so translations mostly swap disc files rather
than code. Names stored in the save (player, Pokémon nicknames, OT) are fixed-size UTF-16 strings in
every region (from the real saves above; `save-data.md`): only their length
limits matter between regions, not their encoding.

(likely) The **Card e room** (`M1_carde_1`) and its menu are a closed loop in the US
build: no exit record leads in (the room's own exit leads to `M1_stadium_1F`). Card e was a Japan-only feature
(**likely**).

(likely) The US main font draws **U+005C as a yen sign** (as in JIS X 0201): the fonts
are keyed by Unicode but were built from a Japanese source font.

(**verified** by a read-only count of both discs' fonts and text): The **kanji font** (a font file
named `jis_1`) holds about **2,920-2,950 kanji** on the Japanese disc, close to JIS level 1, Japan's standard set of
everyday kanji (2,965). Not the whole
set: the fonts page's count, made against the standard's table, finds **2,920** of its 2,965, with 45 left out (a
different count of the file's entries gives 2,938 here and 2,948 on the US disc; what makes up the difference isn't
checked yet). The Japanese text
uses **922** different kanji, 80 hiragana, 83 katakana and about 76 full-width letters, digits and signs. 76 of the
kanji it uses are not in that file (**likely** in the main fonts inside the code module, not counted here). The **US
disc carries the same kanji font too**, and leftover Japanese messages using 662 kanji.

## Changing it

- For a multi-region port, these two functions are the switch: return the region of the player's
  disc (identified by its hash) instead of a constant.
- Region 2's particle counter is 50 Hz compensation (PAL); a 50 Hz build also needs other
  timing changes that are presumably compiled differently in the real PAL build.

## Movies and sound across the three discs (**verified**, the files compared byte for byte)

- **Movies:** each disc has the same five videos (`movie/`: the opening, the developer logo, the company logo, the
  title-screen demo, the credits). The opening and both logos are identical on all three discs. The **credits**
  (`staffroll.thp`) differ on every disc (the European one is longest), and there is **one European credits video for
  all five European languages**, not one per language. The **title-screen demo** (`autodemo01.thp`) differs on the
  Japanese disc only.
- **Sound:** the seven sample banks (`sound/*.samp`: music, ambience, sound effects, battle and motion sounds, cries)
  are identical on all three discs. (The game has no voice acting.)
- **Names in the save keep their language:** a Pokémon's name is stored in the save as text when it is caught (the
  party screen kept "FURRET" while the game's text was switched to another language, verified in the PC port (the decompiled code running natively)), so only
  names drawn from the message banks (species, moves, items and abilities in menus and battle text) follow a language
  change; stored names (nicknames, the player's, and un-nicknamed Pokémon's) stay as they were written.

## Open questions

- What build kind 5 changes in `floor.c` (a resource size 0x3E4 vs 0x320 for kind 4).
- What the PAL Options screen's rows look like in play (only the picture sheet was seen).
- Which pictures carry English words on every disc alike (needs a look at the pictures, not a file comparison).
- What build kinds 0, 1, 3, 5 are.
