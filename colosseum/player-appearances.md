# Player appearances (where the player is shown)

**Applies to:** Colosseum (**verified** / **likely** per row, USA GC6E01) · XD (**no**: to be extended when XD is in)

Status: **partial** (2026-10-17, first pass). Every place Colosseum shows the player (Wes), with **what it loads**
(3D model, sprite, texture, pre-rendered video) and what a replacement would have to provide. This is the checklist
for player avatar packages: a replacement character is complete only when every row is covered or has a fallback.
Labels: **verified** (code, data, or checked in play or in a model viewer), **likely**, **guess**; rows marked **to
trace** are known gaps.

## The player's assets

| Asset | What it is | Where it lives |
|---|---|---|
| `ken_b1` | the overworld model (walks, runs, talks), 38 joints | `people_archive`; NPC info entry 1, model id `0xF70400` (**verified**) |
| `ken_a1` | the battle model, trainer slot layout, with face-texture swaps (5 faces) | the trainer archive; trainer record 1 (the GameCube hero) names its model (**verified**, trainers.md (not yet published)) |
| `ken_c1` | the high-detail cutscene model (85 joints), used only by the opening scene's rooms | `S2_out_3` / `S2_snatchroom_1` room objects (**verified**, trainers.md (not yet published), movies.md) |
| `bike_pokemon`, `bike_hizuki` | the world-map bike, each a whole model with the riders built in | the world-map menu's map objects (**verified**, menus.md (not yet published) "Bike travel scene") |
| title sprites | Wes and Rui as 2D sprites on the title screen | title menu 0x13, items 0xEFB-0xF0D (**verified**, menus.md (not yet published)) |

## Where the player appears

| # | Place | What is shown and loaded | What a replacement needs | Label |
|---|---|---|---|---|
| 1 | **Title screen** | Wes and Rui as **menu sprites** (four pages that rotate between visits), not 3D models | sprite art for each page, or the title keeps the original pair | verified (menus.md (not yet published)) |
| 2 | **Continue** (top menu) | the player's summary: model, name, play time, before loading the save | the overworld model (read by model id `0xF70400` in the title code) | verified (new-game-flow.md, animation-catalog.md (not yet published)) |
| 3 | **Name entry** | a character model per mode; mode 0 shows the player | the overworld model, idle | verified that a model is shown per mode (name-entry.md); which animation: to trace |
| 4 | **Opening movie** (new game) | **pre-rendered video** (`openingdemo.thp`): Wes escapes the Snagem hideout | can't follow a replacement; an in-engine remake of the scene would use a `ken_c1` replacement | verified (movies.md) |
| 5 | **Attract video and staff roll** | pre-rendered videos (`autodemo01`, `staffroll`) | can't follow a replacement | verified that they are videos (movies.md); which shots show Wes: likely (seen in the videos) |
| 6 | **Field** (every room) | `ken_b1`: stand, walk and run from the motion table, turning, the neck look-at toward points (21 script uses), footsteps from the foot parts, script slots 0 and 1 and the surprise reaction | an overworld model with those animations, the head joint index (18 for Wes) and walk / run cycle distances | verified (player-movement.md, animation-catalog.md (not yet published) "Avatar packages") |
| 7 | **In-engine cutscenes** | the same `ken_b1`, moved and turned by room scripts (character group 0, id 100), framed by the rooms' camera shots | as row 6; the camera shots assume Wes's height (**guess**: a much taller or shorter model may be framed badly) | likely |
| 8 | **World-map travel** | a whole **bike model with the riders built in**, picked by story flag 997 (with Rui, or with the Pokémon) | a bike-with-rider model per variant, or a separate rider attached to the bike | verified (menus.md (not yet published); checked in a model viewer) |
| 9 | **Battle** | `ken_a1` beside the partner: idle (slot 0), ball throw (1), win / entry (5), the Shadow-related reactions (7, 8), being hit with its follow-up (9), recall / lose (10); face expressions by swapping the face texture | a battle model with those slots filled (2-4 and 6 may repeat the idle) and, optionally, face images | verified (animation-catalog.md (not yet published), trainers.md (not yet published)) |
| 10 | **Snagging** | Wes-only battle slots 6 and 7 (the Snag Ball throw and its follow-up); the Snag Machine is part of the model | the two snag animations; the Snag Machine, if it should be separate, cut out as its own part | verified for the slots; the machine being part of the mesh: likely |
| 11 | **Colosseum entrances and battle intros** | the battle model's entry / win slot (5) | as row 9 | verified (slot use, animation-catalog.md (not yet published)) |
| 12 | **P★DA** | the player model is read by the P★DA code (model id `0xF70400`) | the overworld model | verified that it is loaded (animation-catalog.md (not yet published)); which screen shows it: to trace |
| 13 | **Party / summary screens** | not traced: whether a trainer model or picture appears | to trace | to trace |
| 14 | **Battle Mode screens** | the trainer screens before a battle (`gMenuTaisenWork`, the battle-start trainer screen): whether the player's model or a picture appears | to trace | to trace |
| 15 | **TV news in The Under** | Venus's show puts Wes's face on the screen as a wanted criminal | a face picture (a texture) | likely (story-route.md (not yet published) step 85) |
| 16 | **The ending** | the scenes after the final battle (the Realgam Tower Colosseum, the Ho-Oh flight) | to trace: in-engine models or video | to trace |
| 17 | **Battle portraits** | the portrait slots drawn in battle (the portrait queue at battle start): whether the player has one | to trace | to trace |
| 18 | **Text** | the player's **name** (in messages, the save, GBA link records), never a picture | nothing visual | verified |

## Fallbacks

The sensible rule for a replacement that lacks a piece is to fall back to the original asset for that row: the videos
(rows 4-5) always stay original; the bike (row 8) stays original unless a bike-with-rider model (or a rider attached to
the bike) is provided; the title sprites (row 1) stay original unless the package brings sprite art.

## For XD (later)

Extend this list when XD is documented: XD's player (Michael) has more places (the P★DA's own screens, the Poké
Spots, his own vehicle and cutscenes in engine). The rows above become the shared columns; XD adds its own rows.

## Open questions

- The rows marked **to trace** (12-14, 16, 17).
- Which name-entry animation the player model plays (row 3).
