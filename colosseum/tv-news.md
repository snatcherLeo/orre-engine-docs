# The TV news

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-16). Televisions show news that follows the story; it can be watched at the Outskirt Stand
shop. Code: script commands 182 (`std182_tvNewsEvent`) and 211 (`std211_tvSetChannelByStory`) in
`gs_range_801C766C.c`; on the US disc command 182 (watching) is called by the Outskirt Stand shop's script
(`S1_shop_1F`) and by the shared TV scene of the common script (function 19, common-script.md; **likely** the one the
house TVs use), while command 211 (the TV-screen picture) also runs in seven houses: four in Phenac City, one
in Pyrite Town and two in Agate Village. **Verified** from the code and the decoded room script unless marked.

## What it does

Watching the TV plays one news broadcast chosen by story progress, then marks it as seen. The TV screens
(a texture on each room's model, in the shop and in the town houses) show whether there is unseen news.

## Data structures
Only story flags: for each broadcast a "story reached" flag and a **"seen" flag**. The code calls `std078_flagSetFalse`, whose
name is swapped (it **sets** the flag), so watching **sets** a "seen" flag, and the screen shows
the "new" picture while it is clear (**verified**; no room script sets these flags, only command 182 does).

| Story flag (reached) | "Seen" flag set on watching | Broadcast |
|---|---|---|
| (none of the below) | 0xA05 | news 0 (message 0x426E) |
| 0xACF | 0xA06 | news 1 (0x426F) |
| 0xAD0 | 0xA07 | news 2 (0x4270) |
| 0xAD3 | 0xA08 | news 3 (0x4271) |
| 0x42C | 0x42D | news 4 (0x4272) |
| 0xADB | 0x483 | a special broadcast in two parts (0x439D, 0x439E) with TV-screen texture changes 0x0C -> 0x0E |
| 0xADA | 0xA0F | a special broadcast in two parts (0x439F, 0x43A0), textures 0x0C -> 0x0F |

The checks run in a fixed order (Shadow record 0x43 first, then 0xADA, 0xADB, 0x42C, 0xAD3, 0xAD0, 0xACF), so the
first one found set wins; that order is **likely** the story order, newest first. A certain state of Shadow record 0x43
(`fn_801EEAD0(0x43)`) also selects news 4 (**likely**: a news story about that Shadow Pokémon).

## Flow
1. `std182_tvNewsEvent`: pick the broadcast as above, set its "seen" flag, update the screen
   (`std211_tvSetChannelByStory`), wait a moment (timed by the text speed and frame rate), show the message(s) with a
   sound, and end the event.
2. `std211_tvSetChannelByStory`: sets the TV screen's texture mode from the same flags: modes 3 / 4 for the five
   news (seen flag clear / set) and 11 / 12 for the two special broadcasts (**likely** "new" and "watched" pictures).

## Disc data
The messages are in the game text; the TV screen pictures are texture-change entries of the shop's room model.

## Changing it
Data and script: new broadcasts need a story flag pair, a message and a branch in command 182 (code).

## Open questions
- What the TV screen looks like in each mode (visual check).
- The exact condition behind Shadow record 0x43.
