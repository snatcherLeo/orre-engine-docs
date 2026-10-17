# Messages and text (`GSmsg*`, `msgctrl*`)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-17). The lookup path, the placeholders, the bank file and the control codes are decoded;
fonts have their own page ([fonts.md](../engine/fonts.md)).

## What it does

All in-game text is looked up by a 32-bit **message key**. `GSmsgGetGSchar(key)` turns a key into
a pointer to the encoded string inside a loaded **message bank**. Text that has blanks to fill
(names, move names, numbers) uses numbered **placeholder slots**: the caller looks up each piece
and stores it with `msgctrlSetValue(slot, value)` before the message box prints the template.

## Data structures

- **Message key:** `key >> 20` is the bank (group) id, `key & 0xFFFFF` is the entry id inside the
  bank (`GSmsgGetGSchar`, `src/game/gs_msg.c`, 0x800FA280). Key 0 means "no text" and returns NULL.
- **Message manager** `lbl_80478B08` (a pointer; name is a guess): `+0x08` is the head of a linked
  list of loaded banks.
- **Bank** (in memory, as loaded): `+0x00` u16 bank id, `+0x04` u16 entry count, `+0x08` next bank,
  `+0x10` an array of `count` 8-byte entries `{u32 entry id, u32 byte offset}` **sorted by id**.
  The string is at `bank + offset`.
- **Placeholder slots:** small integers passed to `msgctrlSetValue`; seen in battle code:
  `0x0D`/`0x0E` (move or ability name), `0x0F`/`0x1F`/`0x20`/`0x21` (a Pokémon), `0x1A`/`0x1C`
  (ability name), `0x28` (move name), `0x29` (item name), `0x41` (trainer name). `msgctrlSetValue`
  dispatches on the slot through an 81-case jump table (`jumptable_80363630`), so there are up to
  81 slot kinds. Slot meanings are guesses from the call sites, not yet from the printer.
- **What a slot holds (2026-10-15, from `msgctrlSetValue`'s definition and 317 call sites):** each slot id maps to
  one global (`lbl_8047ADC8` .. `lbl_8047AEA4`, about 70). Some store a **number** as a u16 (item / move ids: `0x2D`,
  `0x2E`, `0x39`, `0x4E`, `0x58`, `0x59`, `0x5D`), the rest a full **32-bit value that is often a pointer to a string**
  (nicknames `0x32`, the strings `msgctrlString` / `msgctrlString2` print through `0x4D` / `0x57`, `GSmsgGetGSchar`
  results through `0x37`). One getter per slot reads it back (`msgctrlPokemon` returns `0x32`'s, `msgctrlEvStrBuf0`
  `0x0D`'s). So the placeholder system is "id -> number or string pointer": the same number-or-pointer pattern as the
  battle `*GetStatus` getters. *For a PC port:* those 32-bit slots must hold pointers safely (keep game memory below 4 GB, or give the slots a
  pointer-sized type).
  *For modders / translators:* a template's placeholder prints whatever the code last stored in that slot, so a new
  message can reuse an existing slot id only where the code fills it.

### String encoding

- Strings are sequences of **16-bit code units**, terminated by **0x0000**. `_msgGetLength` computes
  `(byteSize + 1; >> 1) - 1`, i.e. characters = bytes / 2 minus the terminator.
- **0xFFFF is an escape:** the next byte is a **control code**. Control codes index a table at
  message manager +0x28 (8-byte entries: a flags byte, then a handler function at +4). Flag bits 3 and
  4 say whether the code applies when measuring vs drawing; bits 6-7 say what the handler's result
  means. A handler can return a message key whose string is **spliced in**: the walker keeps a small
  stack of string pointers (`work + 0x34`, depth `work + 0x40`) and resumes the outer string at its
  0x0000. That is how placeholders (`msgctrlSetValue` slots) and nested messages expand.
- **Control-code handlers and their result (2026-10-15).** The table (`msgctrlcode`, 95 eight-byte records) holds a
  flags word and a handler `u32 handler(u8* work)`. The top two flag bits are the **mode**: 0 = the result is ignored
  (line breaks, waits), 1 = the result is a **string pointer** to splice in (names, numbers, the time), 2 = a **message
  key** to look up and splice in (item, move, NPC text). Most handlers fetch the value through one last call and return
  it in the result register without touching it, so the C must say `return call(...)`; about 30 handlers were written
  `void`, which is the same machine code on the GameCube but would lose the inserted text in the PC port (the decompiled code running natively). **Verified** from
  the assembly and byte-identical builds; `msgctrlTime` even reached a full match once its `return` was added.
  Examples: `msgctrlPokemonID` returns the species name (`pokemonDataBiosGetName(pokemonDataBiosGetPtr(id))`),
  `msgctrlTribe` the trainer-class prefix, the `msgctrl*Mons` handlers either a "trainer's Pokémon" message string or
  the nickname, `msgctrlDigit*` / `msgctrlMoney` the digits buffer `_msgctrlMakeDigit` filled.
- Open: the decompiled `_msgGetSize` advances `ip + 2` on a `u16*` (4 bytes) per ordinary code unit,
  which contradicts the 2-bytes-per-character length rule above. Probably a decompilation artifact
  in an unmatched function; check the asm before relying on it. Fan tools (GoD-Tool) document the
  same `FFFF xx` scheme.

### Bank file on disc (resource type 5; checked on `fight_common.fsys` -> `fight`, 2026-10-14)

| Offset | Field |
|---|---|
| 0x00 | u16 (0 in this file; the bank id is probably filled in at load) |
| 0x04 | u16 entry count (934 in the battle bank) |
| 0x06-0x0F | padding (0x5553, then zeros) |
| 0x10 | `count` entries of `{u32 message id, u32 byte offset}`, sorted by id |
| after the table | the strings |

**Encoding: the code units are Unicode (UTF-16, big-endian).** English text is plain ASCII code
points (0x0041 = "A"); the US battle bank also still contains untranslated Japanese entries in
ordinary Unicode kana and kanji (for example entry 0x4292). So the "character set" question is
answered for text; fonts still map those code points to glyphs.

**Control codes** are `0xFFFF` followed by one byte (so strings are not always 2-byte aligned). The byte indexes
the game's handler table, `msgctrlcode` (190 words: per code a flags word and a handler), so the **handler names are
the game's own (verified)**; the meanings below follow from those names and are **likely** unless marked.

| Code | Handler (`msgctrl*`) | Meaning |
|---|---|---|
| 0x00, 0x01 | `CR` | line break |
| 0x02 | `KeyEnd` | end of a text box: wait for the button |
| 0x03 | `KeyWait` | wait for the button |
| 0x04, 0x05, 0x06 | `RubyStart`, `RubyTop`, `RubyEnd` | furigana (a reading over kanji) in Japanese text |
| 0x07 | `Font` | font switch (**verified**, [fonts.md](../engine/fonts.md)) |
| 0x08, 0x38 | `Color`, `Palette` | text colour, colour palette |
| 0x09 | `Wait` | a timed pause |
| 0x0D, 0x0E, 0x41 | `EvStrBuf0` / `1` / `2` | event string buffers (text filled in by scripts) |
| 0x0F, 0x10, 0x11, 0x12 | `AttackMons`, `DeffenceMons`, `ClientMos`, `TsuikaMons` | battle names: the attacker, the defender, the Pokémon a message is about, the added-effect target |
| 0x13-0x19 | `MyName`, `MyMons`, `MyMons2`, `EnemyMons`, `EnemyMons2`, `EnemyTmons`, `EnemyTmons2` | the player's name, own and opposing Pokémon names |
| 0x1A-0x1D | `SpeabiNamea` / `d` / `c` / `t` | ability names (attacker, defender, client, target) |
| 0x1E | `Clientnowork` | a battle placeholder (meaning not read) |
| 0x1F-0x21, 0x42-0x44 | `SideAttackName` / `SideDefenceName` + `ha` / `wo` / `no` | side names with the Japanese particles は / を / の |
| 0x22-0x27 | `TrainerType`, `TrainerName`, `TrainerLose`, `TrainerEnename`, `TrainerEnename2`, `TrainerClientno` | trainer class, name, defeat line, opponents' names |
| 0x28, 0x29, 0x2A | `WazaName`, `ItemName`, `PasoName` | move and item names; `PasoName` (*guess*: the PC, *pasokon*, box name) |
| 0x2B, 0x2C | `Hero`, `Hizuki` | the player's and the partner's names, always from the save (**verified**) |
| 0x2D, 0x2E, 0x39 | `Item`, `Item2`, `Waza` | an item or move name by id |
| 0x2F, 0x30, 0x34, 0x35, 0x45-0x4A, 0x4F | `Digit`, `MenuDigit`, `MenuUDigit`, `MenuHex`, `MenuZDigit` (each with a `2` variant), `MenuFullDigit` | numbers: decimal, unsigned, hexadecimal, zero-padded, full-width |
| 0x31, 0x55, 0x56 | `MsgID`, `MenuMsgID`, `MenuMsgID2` | another message inserted by id |
| 0x32, 0x33, 0x36, 0x4E | `Pokemon`, `Pokemon2`, `MenuPokemon`, `PokemonID` | Pokémon names; `PokemonID` is the species name by id (**verified**) |
| 0x37, 0x51 | `MenuMsg`, `MenuMsg2` | menu message strings |
| 0x3D, 0x3E, 0x54 | `SeOff`, `SeOn`, `TalkSE` | the text sound off / on, the talking sound |
| 0x4B, 0x50 | `Money`, `MenuMoney` | money amounts |
| 0x4C | `Time` | a time value |
| 0x4D, 0x57 | `String`, `String2` | string buffers |
| 0x52, 0x53 | `Shadow`, `Align` | text shadow, alignment |
| 0x58 | `Tribe` | the trainer-class prefix (**verified**) |
| 0x59 | `Npc` | an NPC's name from the 607-entry character-name table, shown only once that record's "heard" flag is set (otherwise message 0xFA3, the speech-bubble icon); starting a battle sets the opponent's flag (**verified**, `name-entry.md`, "Names in the text") |
| 0x5A | `IndentOff` | indentation off |
| 0x5B, 0x5C | `LineSpace`, `BaseLineBias` | line spacing, baseline offset |
| 0x5D, 0x5E | `SndPlay`, `SndWait` | play a sound, wait for it |

Codes 0x0A-0x0C, 0x3A-0x3C, 0x3F and 0x40 have no handler. Each code's flags word has five high bits that most likely
say whether it takes argument bytes and how it is drawn (*guess*: not read yet); the frequent codes read cleanly
without any.

### Fonts

Fonts have their own page: [fonts.md](../engine/fonts.md) (the file format, every font on the disc, the special symbols).

## Translations

**How text, fonts and the character map fit together.** Message banks store text as Unicode
(UTF-16). To draw a character the game looks its code up in the registered fonts; the glyph
record's code *is* the Unicode code point, so **the character map is simply Unicode**. A
translation pack is therefore **text-only** when every character it uses already has a glyph:
replace the message banks (same message ids, entries sorted by id), keep the fonts.

**Which languages the fonts support as-is** (main 22 px font; *verified* from its glyph list):
- **Supported:** English, French (with Œ / œ; ÿ is missing but very rare), German, Spanish,
  Italian, Dutch (except rare ÿ), Catalan, and Japanese (kana in the main font plus kanji across the
  kanji fonts and `jis_1`).
- **Not supported without new glyphs:** Portuguese (needs ã õ Ã Õ), Danish / Norwegian / Swedish /
  Finnish (å æ ø Å Æ Ø), Icelandic (ð þ ý), Polish, Czech, Turkish and other Latin Extended
  languages, Greek (only α β exist), Russian / Ukrainian (no Cyrillic), Chinese and Korean (no
  Hangul; Chinese needs many characters beyond JIS level 1).
- Missing Latin-1 letters in the main font: `Ã Å Æ Ð Õ Ø Ý Þ ã å æ ð õ ÷ ø ý þ ÿ`.

**Adding new characters (e.g. Cyrillic, Greek, CJK).** Add glyph records and bitmaps to a font:
- the simplest is a **new font file** (an extra font in a room's font entry, or in `common_rel`'s
  font resources), since fonts are looked up by character across all registered fonts; records must
  stay **sorted by character code** (binary search);
- glyphs are 4-bit, up to the cell size (22 px main / 18 px small); width = advance, so
  proportional fonts work;
- the 512 x 512 glyph texture is a cache refilled as text is drawn, so large character sets (CJK)
  work, as `jis_1` shows;
- line-breaking and placeholders are language-independent; text that measures width
  (`GSmsgGetRect`) uses the glyph widths, so layout adapts.
Open: how many font slots the manager has (`GSmsgInit`), and whether any menu draws text through a
different path (pre-rendered textures with English baked in would need image edits, not text).

The tools a translation needs: a text editor for the message banks, a missing-character checker (characters
with no glyph in any registered font) and a font tool for adding glyphs.

## How text reaches the screen: the glyph atlas (2026-10-16; verified at runtime)

GSmsg does not draw text straight from the font bitmap. Each frame it renders the glyphs it needs into one of **two
atlas textures** (a cursor `atlasX` / `atlasY`, reset to 2 / 1 by `GSmsgDaemon`, which also flips the texture index),
by locking the texture, writing glyphs on the CPU and unlocking it; the text quads then sample that atlas. On the
GameCube the unlock flushes the data cache and invalidates the GPU's texture cache so the new glyphs are seen. *Port
note:* a PC graphics layer that caches textures must notice these CPU rewrites; otherwise every window shows the first
message's glyphs at its own positions (seen in a native build, 2026-10-16).

## Flow

1. Data getters return message keys: `wazaGetStatus(0, move, 1, 0)` (move name key),
   `itemGetStatus(0, item, 1, 0)`, `fightTrainerGetStatus(0, trainer, 0x3F, 0)`,
   `pokemonTokuseiDataBiosGetName(pokemonTokuseiDataBiosGetPtr(ability))`.
2. `GSmsgGetGSchar(key)` walks the bank list to the matching bank id, then binary-searches the
   entry table (`lo/hi/mid`, 8-byte stride).
3. `msgctrlSetValue(slot, string)` stores the string for the template; the message box then
   expands the template ("String encoding" above).

The typical battle line is
`msgctrlSetValue(0x28, GSmsgGetGSchar(wazaGetStatus(0, move, 1, 0)));`. Much of the decompiled C
had split this into separate statements that relied on leftover registers; it was rewritten with
real arguments on 2026-10-14.

## Disc data

Banks are loaded as whole files (layout above). Each room archive carries its own bank (type 5,
named after the room); shared banks are `fight` and `tool_fight` (fight_common), `system_tool`,
`pocket_menu` and the menu archives ([fsys-catalog.md](../colosseum/fsys-catalog.md)). A port
can read the same banks from the player's disc unchanged.

## Changing it

**For modders (*likely*, from the formats above):** dialogue edits and translations are the same thing: a text pack
replaces entries by bank and message id (kept sorted), with placeholders written as named tags; fonts are swapped
separately when a language needs new glyphs ("Translations" above).


- New or replaced text: add or patch a bank; entries must stay **sorted by id** or the binary
  search will miss them.
- A mod that adds a whole bank only needs to link it into the manager's list.
- Placeholder slots are a fixed set (81-way switch); new slot kinds need code.
- **Banks and fonts are linked in place (2026-10-15, verified from the code):** opening a bank (`GSmsgOpen`) or a font
  file (`GSmsgFontOpen`) writes the list links (`next` / `previous`, header words +0x08 / +0x0C) into the loaded file
  itself; no separate list nodes are allocated. A font file is a chain of fonts, each starting with its header ([fonts.md](../engine/fonts.md)); the
  header's first 8 bytes (id, width, height, size = the distance to the next font) are what a slot copies; banks with the same font id become one list, searched in
  order. A font slot keeps the first header's id / width / height and a pointer to the first bank. *In a PC port* that keeps
  the files big-endian with 32-bit link slots (a fixed memory layout), the code reads them through
  big-endian views; the slot copy of the header is done field by field (the GameCube copies the whole 8-byte header
  over the slot and then overwrites its second word). Getting this wrong crashes `GSmsgFontOpen` at boot (seen in a
  native build).

## Efficiency

No hashing and no per-lookup allocation: a short linear walk over loaded banks, then
O(log n) inside the bank. Strings are used in place; nothing is copied.

## Open questions

- Which control codes take argument bytes (the flags word, above); the number of font slots.
- Who loads and unloads banks, and when (per floor? per battle?).
- What each of the 81 placeholder slots formats (name, number, gendered text...).
