# Card e: the Virtual Capsule room (Japan-only content left in the US build)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). The
e-Reader "Colosseum-e card" feature: a battle facility room in Phenac City, a card menu and a few battle kinds,
all present on the US disc but **not reachable** in US play. Labels: **verified** / **likely** / **guess** per claim.

## The room seen in the attract video
- **Where it is shown:** the title screen's **attract video** (`movie/autodemo01.thp`, the demo that plays when the
  title is left idle), one shot at about **0:09.3-0:10.5**, between two Phenac City shots: Wes walks past a round
  floor pad under a column of light, with green-lit machine cabinets and a red-topped console behind it
  (**verified**, frames read from the video). It is **not** in the opening video, which shows the Snagem hideout,
  including the cage room `S2_snatchroom_1` where Wes takes the Snag Machine (movies.md).
- **Which room:** **`M1_carde_1`** (room id 0x80, Phenac City). Its main map texture contains the **exact floor
  pad** of the shot (concentric rings, three groups of dash marks, the red arc), and its second texture the green
  monitor panels and the red console top (**verified**, matched by eye against the room's own textures; the room
  has not been rendered to compare the whole view).
- **What it is:** not a lab where the Snag Machine was made (that idea came from the shot's look; the game text that
  mentions a "Snag Machine lab" belongs to the revisited Snagem hideout, `S2_building_1F_2`, which is reachable).
  The room's dialogue (already translated into English on the US disc) describes a **Colosseum-e card battle
  hall**: a receptionist asks the player to register, explains battling a roster of **VR trainers** with an
  e-Reader and Colosseum-e cards, an **Endless Battle** challenge unlocked by beating the top-ranked VR trainers,
  and shows the player's battle record; the light pad is the **Virtual Capsule** (**verified** from the decoded
  script's messages; paraphrased).

### Was it repurposed from an earlier room? (open: a **guess** either way)
The idea: the room was first something else (such as a lab where the Snag Machine was made) and later reused as
the Card e hall. What the US data says (checked 2026-10-17):
- **Nothing left over from another purpose** (**verified** for what was checked): the room's archive holds only the
  map, collision, script, room module, the 17 Card e trainers and staff, and two fonts for card text (`card_e`,
  `jis_1`, the latter **likely** a Japanese character set); the map model has no meaningful internal names, and the
  script and text are all about e-card battles.
- **Signs it was planned as a Phenac City room** (**likely**): its archive number (0x2A) sits inside Phenac City's
  block of room archives (0x1F-0x2C, between the gym's 0x28-0x29 and the stadium's 0x2B; **verified** from the
  archive table), and its main texture is Phenac's beige stone with carved circuit lines, not the Snagem
  hideout's rusty metal.
- The glowing pad fits the "Virtual Capsule" the text describes.
So nothing on the US disc supports or rules out an earlier purpose; the visible evidence points to a room designed
for Phenac City. Where it could still show up: the Japanese release's data, pre-release material, or developer
interviews (outside the disc).

## Everything on the disc that belongs to it

| Piece | What | Label |
|---|---|---|
| Room `M1_carde_1` (archive `M1_carde_1.fsys`) | map, collision, script, room module and 17 character models (trainers and staff) | verified |
| Menu room 0x393 `carde_menu` | the Card e menu (card reading, records); pushed by script command 202 from the room | verified (menu-rooms.md) |
| Code | the Card e "Matrix" unit (`menu_carde_matrix.c`, `gs_range_80033278.c`: a command processor that waits for data transfers, **likely** from the e-Reader over the GBA link) and `cardesavedata.c` (source names from the executable's strings) | verified (names), likely (role) |
| Battles | battle kinds 10 / 17 ("Matrix" battle, ordinary / special) and 11 (consecutive battles), one template record each | verified (battle-engine.md) |
| Text | the room's dialogue and the menu's messages, in English | verified |
| Trainer | trainer record 9, a placeholder named "Card e trainer" (in Japanese) with no party, used by the P★DA's Shadow list for Card e Shadow Pokémon | verified (pda.md) |
| e-Reader program | `bg0thumbcode.bin` / `bg1thumbcode.bin` / `bg2thumbcode.bin` on the disc root: the program sent to the e-Reader for the card battles (one of the three picked at random), with the Card-e Reader+ game code PSAJ / PSAE (gba-upload.md) | verified |
| Video | the attract video's shot of the room | verified |
| Music | `virtual_song`: after a Card e battle the code returns to it when the Card e menu was entered fresh (otherwise to the world-map song), so it plays only on the Card e path (from the code) | verified |

## How a card battle would start
The room places **16 trainer models** (ordinary trainers, riders, a comedian and others) on one spot, all hidden by
default, and all of them share one talk function that only runs script command 215
(`std215_battleStartTrainerWithPrompt`) with the arguments it is given; no fixed battle id appears in the script
(**verified** from the room's NPC placements and its decoded script). So the room is built to show **one opponent at a
time** and start its battle on talking; the battle id and the trainer **likely** come from the card read by the Card e
code. The three Card e battle records fit the kind names (**likely**): battle **8** for an ordinary "Matrix" trainer,
**9** for a special one and **521** for the **Endless Battle** challenge. All three are story-mode kinds that give
EXP, give no prize money and write the party back to the save (battle-kind table, **verified**).

## Why it can't be reached in the US game (**likely**)
Nothing leads into `M1_carde_1`: no exit record, no room script, not the common script; only the Card e menu's own
code links back to it, and that menu is opened only from inside the room (battle-engine.md, Card e notes). The Colosseum-e cards were released only in Japan (public knowledge, not from code;
the e-Reader itself was also sold elsewhere), so the US release appears to have cut the entrance while keeping the room, the code and even the
translated text. Trying to enter it in an emulator would confirm.

## For the port and mods
- The room is complete enough to visit with a debug room jump (room 0x80); a "restore Card e hall" mod would need an
  entrance (an exit record or door into it) and a replacement for the e-Reader card input.

## Region notes
- Japan had the Colosseum-e cards; the US build keeps the content but has no way in. How the Japanese build connects
  the room (an exit from Phenac City or the stadium) is open until the Japanese data is read.

## Open questions
- The Japanese build's entrance; how the VR trainer data arrives (the card format), and how the Card e code picks
  which hidden model to show and which battle record to use.
- Rendering the room to confirm the whole view of the shot, not only its textures.
