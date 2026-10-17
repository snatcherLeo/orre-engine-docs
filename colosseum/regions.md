# Region and build differences

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). Everything below is from the USA build (GC6E01); the other builds are not
covered here.

## What it does

The game asks two tiny functions which build it is, and branches on the answers. Both return a
constant in this build, so the region logic for other builds is still present in the US executable.

| Function | Returns (USA) | Meaning (evidence) |
|---|---|---|
| `fn_800057A0` (0x800057A0, `src/game/main.c`) | **1** | **Region.** 0 = Japan, 1 = USA, 2 = **PAL** (confirmed by the 50 Hz particle compensation below). Drives the disc and memory-card game codes, the save stamp, and a few content differences. |
| `fn_800057A8` (0x800057A8) | **6** | **Build kind.** 2 = a developer build: move-animation sequence errors are reported (`fn_801D744C`, wazaSequenceEntry.c); 4 = a build that rewrites the mounted disc id to "GC6J" before card access (a Japanese-disc test build? guess); 6 = this retail build. |

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
  (`lbl_802792E8`, Latin-1 `é`).
- On load, the game scans up to 127 card files and accepts the one whose name matches and whose
  game code matches the region: "GC6J" for region 0, "GC6E" for regions 1 and 2; company code "01".
  So **a Japanese save is rejected by a US game** and vice versa: a cross-region save editor or port
  needs to rewrite the game code (and convert text, see below).

## Everything that branches on the region (USA executable)

| Where | Region 0 (Japan) | Region 1 (USA) | Region 2 (PAL) |
|---|---|---|---|
| Disc id at DVD init (`gs_dvd.c`; skipped when build kind = 4) | "GC6J" | "GC6E" | "GC6E" (as compiled here) |
| Memory-card file game code (`field_exact_801CFD08.c`) | "GC6J" | "GC6E" | "GC6E" |
| New save's attestation stamp (`hero.c`, `gamedataCreate(save, ver, gen, area, lang)`) | 0xB, 3, **1, 1** | 0xB, 3, **2, 2** | 0xB, 3, **3, 8** |
| Particles (`field_range_80117E58.c`) | | | one extra particle update every 5th frame: **50 Hz compensation** |
| Map models (`floor.c`) | | in 15 rooms (ids 3, 0xB-0xE, 0x15, 0x24-0x26, 0x72, 0x75, 0x7B, 0xCA, 0xD0, 0xDE) texture animations are linked to the model animation and given a rate: a **US-only fix** (guess) | |

So the attestation stamp is {version 0xB = Colosseum, generation 3, area, language}; a Japanese
save says area 1 / language 1, a US save 2 / 2, a PAL save 3 / 8 (8 = multi-language? guess).

## Text and fonts (expected differences; to verify)

Message banks and fonts are disc data loaded per floor (`floorReadMsgPreFunc`,
`floorReadFontPreFunc`, `messages.md`), so translations mostly swap disc files rather
than code. Names stored in the save (player, Pokémon nicknames, OT) are fixed-size GS-encoded
strings (`save-data.md`): their encoding and length limits are the main region issue
for saves.

(likely) The **Card e room** (`M1_carde_1`) and its menu are a closed loop in the US
build: no exit record leads in (the room's own exit leads to `M1_stadium_1F`). Card e was a Japan-only feature
(**likely**).

(likely) The US main font draws **U+005C as a yen sign** (as in JIS X 0201): the fonts
are keyed by Unicode but were built from a Japanese source font.

## Changing it

- For a multi-region port, these two functions are the switch: return the region of the player's
  disc (identified by its hash) instead of a constant.
- Region 2's particle counter is 50 Hz compensation (PAL); a 50 Hz build also needs other
  timing changes that are presumably compiled differently in the real PAL build.

## Open questions

- What build kind 5 changes in `floor.c` (a resource size 0x3E4 vs 0x320 for kind 4).
- Whether the Japanese build's message/font format differs (byte-wide vs 16-bit text).
- What build kinds 0, 1, 3, 5 are.
