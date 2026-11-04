# Fonts

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16): file format, layout rules, registration and every font on the US disc are
decoded; two font ids the text uses are missing from the disc (open questions).
Related: [messages.md](../systems/messages.md) ("Fonts": the first write-up, and how text, fonts and translations fit together).

## What it does

All in-game text is drawn from bitmap fonts: each character is a small 4-bit greyscale picture, looked up by its
Unicode code point and copied into a texture when it is first needed. The game keeps a few **font ids** (0 = the
normal 22-pixel font, 1 = a smaller 18-pixel font, 3 = a third 18-pixel slot); text switches between them with a
control code. Every font id can be made of several font files chained together: a big main font loaded at startup,
plus a small extra font each room or menu brings along with just the extra characters its own text needs.

## Data structures

**Font file** (*verified*, from the glyph lookup `_msgGetCodeInfo`, the copy routine `_msgMakeTexture` (0x800FD69C) and `GSmsgFontOpen`,
src/game/gs_msg.c): a list of fonts, ended by font id 0xFFFF (a font whose size field is 0 runs to the end of the file).

| Offset | Size | Field |
|---|---|---|
| +0x00 | u16 | font id (the slot it joins) |
| +0x02 | u8 / u8 | cell width / cell height in pixels |
| +0x04 | u32 | size of this font in bytes (the next font starts at the next 32-byte boundary) |
| +0x08 | u16 | glyph count |
| +0x0A | u16 | 0 (reserved; 0 in every font on the disc) |
| +0x0C | u32 | bitmap data offset, counted from **header + 8** (a quirk: header + offset is 8 bytes short) |
| +0x10 | 8 bytes | 0 (reserved; 0 everywhere) |
| +0x18 | 8 bytes each | glyph records, sorted by character code |

Glyph record: u16 character (**Unicode**), u8 width (also the advance), u8 height, s8 vertical offset (added to the
line's top when drawing), u24 bitmap offset. Bitmap: 4-bit intensity, rows of ceil(width / 2) bytes, high nibble
first. The lookup is a binary search over the records, so they must stay sorted.

**Layout rules** (*verified* from the code; all distances times the text scale, 1.0 unless a caller changes it):

| Rule | Value | Where |
|---|---|---|
| advance of a glyph | its own width; there is **no extra letter spacing** | `msgSetChar` |
| space (U+0020) | never looked up: advances int(cell width / 2) (11 px in the 22-px font, 9 in the 18-px ones) | `msgSetChar` |
| character no registered font has | draws a white box of the cell size and advances cell width + 2 | `msgSetChar` |
| new line (control code CR) | down by cell height + line gap; back to the line's start x | `msgctrlCR` |
| line gap | font 0: 11, font 1: 6, any other id: int(cell height / 2 + 1) | `GSmsgSetFontInfo` |
| line height | font 0: 22 + 11 = **33 px**; font 1: 18 + 6 = **24 px**; font 3: 18 + 10 = 28 px | |
| font switch | control code 0x07 + one byte font id: sets the task's font, cell size and line gap | `msgctrlFont`, `GSmsgSetFontInfo` |

**Font slots** (`GSmsgFontOpen`): one slot per font id, each holding a chain of font files. A glyph is searched only in
the slot of the **current** font id (all its files, in registration order); there is no fallback to another id.
The debug string "[GSmsgFontOpen] FONT[%d]の登録数をオーバーしました" (too many fonts registered for FONT[%d]) shows a
fixed number of files per slot.

## Flow

1. **Startup** (`main.c`, after `GSmsgInit(2, 5)`): registers the three fonts stored in main.dol (`lbl_8027A500`,
   `lbl_802BD260`, `lbl_802C0CB0`), then common_rel resource 1. The main.dol fonts are byte-identical to common_rel
   resources 0, 2 and 3 (*verified*). Resource 1 is an **empty font file** (only the end marker).
2. **Room or menu load**: every room and menu archive has a resource-type-6 entry (named after the archive, or
   `system_tool`, `pocket_menu`, `name_entry_menu`, `tool_fight`, `fight`, `card_e`, `jis_1`, ...); the room loader's
   type-6 callback `floorReadFontPostFunc` registers it with `GSmsgFontOpen` (*verified*).
3. **Drawing**: each character is looked up, copied into one of two 512 x 512 I4 textures and drawn as a quad, with an
   optional shadow or outline pass ([messages.md](../systems/messages.md)).

## Disc data: the fonts on the USA disc

| Font | Where | Id | Cell | Glyphs | Contents |
|---|---|---|---|---|---|
| main | common_rel resource 0 (copy in main.dol) | 0 | 22 | 408 | ASCII, Latin-1 accents, Œ œ, quotes, arrows, ♀ ♂, hiragana, katakana, full-width digits and letters, a few kanji, game symbols |
| main extra | common_rel resource 2 (copy in main.dol) | 0 | 22 | 59 | 51 kanji, ★, symbols |
| small | common_rel resource 3 (copy in main.dol) | 1 | 18 | 354 | 345 kanji and symbols |
| third slot | common_rel resource 3 | 3 | 18 | 1 | only a space (9 px) |
| empty | common_rel resource 1 | - | - | 0 | nothing (end marker only) |
| `jis_1` | `carde_menu`, `M1_carde_1` | 0 | 21 | 2,920 | kanji only, U+4E00-U+9F8D: the **JIS level-1 kanji set** (2,965 in the standard) |
| room / menu fonts | 160+ archives | 0, 1, 3 | 22 / 18 | 1 to ~100 | the extra characters that room's text uses (often just one glyph) |

Text usage of the font switch (all message banks): font 0: 454 messages, 1: 353, **2: 16, 3: 349, 4: 11**.

**Which level-1 kanji `jis_1` leaves out** (**verified** against the JIS X 0208
level-1 table, rows 16-47): `jis_1` holds 2,920 of the standard's 2,965 kanji and nothing outside it. The 45 it leaves
out are among the most common kanji of all (for example 人, 大, 山, 手, 本, 生, 書, 電, 戦, 闘), scattered over the
rows rather than at the end. 37 of them are in other fonts on the US disc (33 in the main fonts, the rest in room and
P★DA fonts); 8 are in no font on the US disc (扱, 押, 幹, 源, 説, 謎, 閉, 明). So `jis_1` is **likely** a supplement
to a Japanese main font that already carries these common kanji, not a complete set by itself: the US main font is a
reduced one, so on the US disc the Card e room's Japanese text would lack those eight. *Region note:* what Japanese-version
support needs is open until a Japanese disc's fonts are read.

**Digits and punctuation live in one font only** (**verified** from the executable's three fonts
and the extracted character maps): on the US disc only the **main 22-px font 0** (408 glyphs) has ',' ':' and the
digits 0-9. The P★DA room's own font 0 (164 glyphs) has none of them, so the P★DA's money and play time rely on the
main font being in the lookup chain behind it. A font mod that replaces the main font must keep those characters.

### Special symbols (code points reused for the game's own glyphs)

Every glyph of every font on the disc (221 fonts, 3,380 code points: the main
fonts, `jis_1` and all room and menu fonts) was checked in two ways: code points outside the ordinary sets (ASCII,
Latin-1 letters, kana, CJK ideographs, full-width forms) were listed, and every glyph was scored against the same
character in a reference font (MS PGothic for Latin, BIZ UDGothic otherwise), with every low score and every unusual
code point looked at by eye. **Twelve code points draw something other than their standard character** (*verified*,
glyph shapes); everything else draws its own character (kana and full-width forms only in this font's own style).
Text use counts every message bank on the US disc.

Each symbol is described in words.

#### U+0024: Pokémon dollar sign

| | |
|---|---|
| Code | U+0024, standard meaning `$` (dollar sign) |
| Looks like | a capital P whose stem is crossed by two horizontal bars below the bowl (₽) |
| Fonts that contain it | main font (common_rel resource 0), 22 px |
| Example messages | 31 times in 30 messages: money amounts (finding or receiving money), e.g. 0x3CB5, 0x3CB7 |
| Status | **verified** (shape and use) |

#### U+FF04: Pokémon dollar sign, full width

| | |
|---|---|
| Code | U+FF04, standard meaning `＄` (full-width dollar sign) |
| Looks like | the same P with two bars, drawn in a full 22-px cell |
| Fonts that contain it | main extra font (common_rel resource 2), 22 px |
| Example messages | 2 messages: 0x150, 0x151 (money displays) |
| Status | **verified** |

#### U+2031: speech bubble

| | |
|---|---|
| Code | U+2031, standard meaning ‱ (per ten thousand sign) |
| Looks like | a solid black oval speech bubble with a short tail at its lower right |
| Fonts that contain it | main font, 22 px |
| Example messages | 1,526 times in 984 messages: the marker before a speaker's line, e.g. 0x3AFE |
| Status | **verified** |

#### U+03B2: e-Reader "e"

| | |
|---|---|
| Code | U+03B2, standard meaning β (Greek small beta) |
| Looks like | a bold, round lower-case e in the e-Reader logo style |
| Fonts that contain it | main font, 22 px |
| Example messages | 13 messages, always in "β-Reader" (shown as e-Reader), e.g. 0x3B62, 0x3B63, 0x3B66 |
| Status | **verified** (shape and use) |

#### U+03B1: e-Reader "e+"

| | |
|---|---|
| Code | U+03B1, standard meaning α (Greek small alpha) |
| Looks like | the same bold e with a small plus sign at its upper right |
| Fonts that contain it | main font, 22 px |
| Example messages | none in the US text |
| Status | shape **verified**; meaning (the e-Reader+ mark) **likely** |

#### U+337C: e-Reader "e+" (second copy)

| | |
|---|---|
| Code | U+337C, standard meaning ㍼ (square era name Shōwa) |
| Looks like | identical to U+03B1 |
| Fonts that contain it | main font, 22 px |
| Example messages | none |
| Status | shape **verified**; meaning **likely** |

#### U+337D: e-Reader "e" (second copy)

| | |
|---|---|
| Code | U+337D, standard meaning ㍽ (square era name Taishō) |
| Looks like | identical to U+03B2 |
| Fonts that contain it | main font, 22 px |
| Example messages | none |
| Status | shape **verified**; meaning **likely** |

#### U+337E: e with acute accent

| | |
|---|---|
| Code | U+337E, standard meaning ㍾ (square era name Meiji) |
| Looks like | a lower-case e with an acute accent above |
| Fonts that contain it | main font, 22 px |
| Example messages | none (the US text writes é as U+00E9) |
| Status | shape **verified**; purpose open |

#### U+2030: two low dots

| | |
|---|---|
| Code | U+2030, standard meaning ‰ (per mille sign) |
| Looks like | two small square dots side by side on the baseline |
| Fonts that contain it | main font, 22 px |
| Example messages | 2 messages: 0x2F07, 0x2F0B (the name-entry symbol list) |
| Status | shape **verified**; meaning (a two-dot mark offered in name entry) **likely** |

#### U+25A1: boxed S

| | |
|---|---|
| Code | U+25A1, standard meaning □ (white square) |
| Looks like | a capital S inside a square frame |
| Fonts that contain it | M3_out (Agate Village) font, 22 and 18 px; name_entry_menu font (in colosseumbattle_menu), 22 px |
| Example messages | 9 times in 4 messages: the name-entry kana lists 0x2EE1, 0x2EE5 (last key after ！？) and a placeholder for an item name, 0x557E |
| Status | shape **verified**; meaning open |

#### U+FF3E: small ring (handakuten key)

| | |
|---|---|
| Code | U+FF3E, standard meaning ＾ (full-width circumflex) |
| Looks like | a small hollow ring in the upper part of the cell |
| Fonts that contain it | name_entry_menu font, 22 px |
| Example messages | 3 messages: name-entry kana lists 0x2EE2, 0x2EE6 (after the dakuten key ‘), and 0x2EFB |
| Status | shape **verified**; meaning (the handakuten ゜ key) **likely** |

#### U+005C: yen sign

| | |
|---|---|
| Code | U+005C, standard meaning `\` (backslash) |
| Looks like | a yen sign ¥ (the JIS X 0201 convention of Japanese character sets) |
| Fonts that contain it | main font, 22 px |
| Example messages | none in the US text |
| Status | **verified** |

The e / e+ / é glyphs also sit at the era-name squares (㍼ ㍽ ㍾). Only β is used by the US text. For a translation or a modern font, these code points must be mapped back
to their real meaning (₽, speech bubble, e-Reader marks), not drawn as the standard character.

## Hints at the original font

- **No name or metadata survives**: the font headers' reserved fields are zero in every font, no string in main.dol or
  any module names a typeface, tool or file (*verified* by a search of all of them). The only font strings are the
  game's own debug messages.
- **A Japanese source font** (*likely*): the glyph at U+005C (backslash) is drawn as **¥**, the JIS X 0201 convention of
  Japanese character sets; the main font also carries hiragana, katakana and full-width ASCII, and `jis_1` is exactly the
  JIS level-1 kanji set in Unicode order. So the bitmaps were very likely rendered from a Japanese system font
  (JIS-ordered) and then converted to this Unicode-keyed format.
- **Which typeface:**
  - **Latin and Latin-1 letters: MS PGothic at 22 px** (*likely*, close to verified). For 91 of the 92 ASCII
    characters (and 50 of 56 Latin-1 ones) the game's advance width is exactly MS PGothic's width at 22 px plus
    1 pixel (the converter likely added one pixel of spacing), and MS PGothic rendered at 22 px matches the game's
    letter bitmaps with a mean correlation of 0.982 (other gothics: 0.63-0.78). Ricoh sells the same design as HG
    PGothic B.
  - **Kana: a straight-stroke square gothic, not MS Gothic** (*likely*), closest to Heisei Kaku Gothic among the
    specimens compared (which weight is open).
    Compared with published specimens and a public type tester, あかさたなアサタ match its construction and
    proportions, at a stroke weight between W3 and W5; Morisawa's Shin Go, nearly identical at 22 px, was not
    tested. Ruled out by eye: Fontworks Rodin, Hiragino Kaku, Kozuka and every rounded gothic.
  - So the main font was most likely **assembled from two typefaces** (*likely*): Western letters from MS PGothic,
    kana (and probably the kanji) from a Japanese gothic. Japanese game fonts often mix sources this way.

## Changing it

- **Adding characters** (translations): append glyph records (keep them sorted), add the bitmaps, update count, size
  and data offset. A room font can extend a font id for one room only.
- Keep widths honest: the width is also the advance, and there is no letter spacing to fall back on.
- A new font id needs a slot: the line gap rule gives any id other than 0 and 1 a gap of half the cell height + 1.
- The port can replace the whole path with a modern font renderer (outline fonts at any size); the metrics above are
  what it must reproduce for a faithful layout (window sizes and line breaks were tuned to them).
- **drawing an outline font in the original's slots** (tested with 4K mock-ups, 640 x 480 scaled
  4.5x, beside the original bitmaps; **verified** as measurements, the look judged by eye):
  - Each glyph is drawn inside its original advance width on the original baseline, so line breaks stay identical.
  - A font with other proportions (Arimo, for example) leaves uneven gaps when drawn at the slot's left edge (worst
    around `t`). Sharing the slot's spare room out by the glyph's own side bearings removes them; scaling the glyph
    0.9-1.1x horizontally toward the slot first gives a slightly fuller look.
  - Size: Arimo at the main font's em has nearly the original cap height (15.2 against 15 px) but an x-height about
    16% larger (11.6 against 10 px), so a slight horizontal narrowing is a compromise.
  - The original apostrophe is curly, so U+0027 is best drawn from a font's U+2019 (’).
  - The special symbols (above) need their own high-resolution drawings: an upscaled bitmap looks soft at 4K. Nine are
    in the main font; the full-width Pokédollar, the e-acute symbol, the boxed S and the small ring are only in other
    fonts.

## Efficiency

Glyphs are never kept as a full font texture: only the characters actually drawn are copied into two 512 x 512 4-bit
textures, so even the 2,920-kanji `jis_1` costs no more texture memory than the main font. Rooms add only the few
glyphs their own text needs instead of shipping one huge font, and the main fonts stay resident.

## Open questions

- **Font
  ids 2 and 4** (used by 27 messages: number inserts and stat labels such as ATTACK, DEFENSE) are small 13 x 13 fonts
  in main.dol's own data, registered at startup right after fonts 0, 1 and 3, so the game's five font slots start out
  full (0, 1, 3, 2, 4) (**verified**: a log of every font registration, on a replay of a recorded play session). Later font
  sets (rooms, menus) re-register 0, 1, 3 and 4 only.
- Labels the game prints itself (for example the COLOSSEUM BATTLE list's "Lv50" / "Lv100" / "Lv???", made with a
  `sprintf` into a Shift-JIS buffer and converted by `SJIStoGSchar`) take the same lookup as message text: a code no
  registered font has draws as a solid white box, which is how a byte-order slip in that conversion shows on screen
  (**verified** in the PC port (the decompiled code running natively) build against a Dolphin frame).
- **Font 3** is switched to by 349 messages, including words (e.g. message 0x1A5 "POWER"), but no font 3 on the disc has
  letters: the main one holds only a space, the Card e menu's `system_tool` font 3 a space and a quote mark
  (*verified* from the glyph lists). Same check needed.
- Whether room fonts are unregistered when the room unloads.

## Region notes

The fonts hold Unicode code points, so the format itself is region-neutral, but the US main font keeps the full
Japanese kana set and kanji leftovers, and resource 1 is empty. Which fonts the Japanese and PAL versions put in ids
0-4 is not compared yet. PAL languages would need more Latin glyphs (see messages.md, "Which
languages the fonts support").
