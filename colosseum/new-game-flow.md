# From power-on to Outskirt Stand: the front-end flow

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-16). The chain of rooms and calls the game goes through from boot to the first
map of a new game, and to the saved room on Continue.
**Verified** from the decompiled C (room changes, menu calls) and the disc's room and archive tables unless marked.
**Also seen in play** (the
PC port running the decompiled code, 2026-10-17): the logos, title, START, memory card, top menu, New Game, name entry,
the opening movie, Outskirt Stand (walking, doors, the TV news, the grunts' scene) and Willie's battle and after-battle
scene ran end to end; Continue, Battle Mode and Options were not part of that run. Room names are the rooms' archive
names (menu-rooms.md).

## What it does

Every stage of the front end is a **room** in the room table: the logos, the title, the opening movie and the first
map. Moving on is a room change (`floorRequestChange(room)` or `floorLink(room, entry)`); full-screen menus on the way
(name entry) are pushed menu rooms (menu-rooms.md).

## Flow

1. **Boot.** The room system's init picks the first room by build kind; retail starts at **0x399 `pokemon_logo`**
   (menu-rooms.md, "The first room at boot").
2. **Logos.** The logo rooms play their movies (`movies.md`): when the `tpc` movie ends, `movieTpcWait` changes to
   **0x39A `genius_logo`**; when the `gs_logo` movie ends, `movieGsLogoWait` changes to **0x384 `title`**. (**verified** from the room
   table: `pokemon_logo` itself shows the `tpc` movie, through its room-record functions `movieTpcStart` /
   `movieTpcWait`; table in step 6. The Nintendo logo room 0x398 `nintendo_logo` has a record whose three functions
   are **empty**, and nothing changes to it: a leftover room, unused in the US build.) The attract demo (`autodemo01` in room 0x39C `auto_demo`, started when the title song ends: step 3) returns to
   0x384 afterwards.
3. **Title** (room 0x384; its script is a stub; **verified** from
   `gs_title_8001EF78.c`): the title is a state machine (`std021_newGameIntroSequence`, state in `gTitleState`,
   `lbl_8047A31C`) that starts the **title song, sound 0x46A** (`std104_soundSetBackgroundMusic(0x46A, 0, 0xFF)`),
   then plays the logo animations (states 0-8), then waits at state 0x20 in the "PRESS START" menu (menu 0x13).
   After START: state 0x1F (a short wait), state 0xC8 **loads the memory card** (`fn_801D0748(1, 2, 0)`; -1 = back to
   waiting; any result but 3 creates an empty save, so 3 is **likely** "a save was loaded"), then state 0x29 **`floorLink(0x3A1, 0)`: the `topmenu` room**,
   where the top menu runs.

   *The memory-card call* (`fn_801D0748(kind, work size, channel)`, `memcard.c`, **verified**; memory-card.md): it starts memory-card
   task `kind` as a per-frame task (`GSgappCreate(.., fn_801CDB04)`), allocates a 0x1E600-byte work buffer and a
   0xA000-byte card work area, and **yields until the task reports a non-zero result**. Kind 1 is this title load;
   kind 4 (ten callers, **likely** the normal save) and kind 7 (the options save) are others. Results seen in the task code: **4** after a write
   completes (the save count is incremented there), **-1** from the abort state, and 1, 2, 3 and 5 from other check
   paths (not decoded). *For the port*: this is
   the next place the title can stall after the song (**likely**): the memory-card task needs a backend that
   finishes with a result (3 for a loaded save, or a value meaning "no save" so an empty one is created).

   **The title song drives the attract demo:** a per-frame function (`fn_8001EF78`) checks, in every state from
   0x1E on except 0xC8, whether the title song is still playing (`soundGetStatus(0x46A)`); when it is not, it sets
   the state to 0x28, and the title's main loop (`fn_800203B4`) then links **room 0x39C, `auto_demo`** (the attract
   demo). So on the GameCube the demo starts when the title song ends. *For the port* (**likely**, from the code, not
   run): if the title song never starts, `soundGetStatus(0x46A)` reads 0 from the first frame of state 0x1E, so the
   title leaves for the attract demo right after the logos, or overwrites the state set by START, and the top menu is
   never reached; the title song (or a stand-in that reports "playing") is on the critical path.
4. **Top-menu room** (0x3A1, `topmenu`): its setup (**likely** `fn_80021644`, which nothing in the C calls; it runs
   only when `fn_800FF548` is 0) starts **`worldmap_song`** (sound 0x446; the
   song's name from the disc's song list), loads the top menu's background model and plays camera animation
   0x632 (the Orre map fly-over behind the menu, **likely**). The top menu itself is `fn_800210F0`, reached through
   the wrapper `fn_80021624` (**likely** from the `topmenu` room's data: nothing in `main.dol` calls it).
5. **Top menu** (`fn_800210F0`, menu 0xAA, choice in `lbl_8047A360`):
   - **New Game** (taken when the menu call itself returns 0, **likely** "the first item chosen") -> a confirmation: with no save, a plain "start a new story?" (message 0x3C3A); when a save exists, a
     warning that it will be deleted (0x3C35, or the same warning in another wording, 0x44C9, when
     `fn_8006A718(0)` is 1: not traced); on "yes": close the menus, start background music 1, **name entry** (name-entry.md; `menuNameEntryOpen(0, 0)`, menu room
     `name_entry_menu`; the call takes a mode: 0 / 1 need no index, 2 names a party Pokémon and checks the slot,
     3 takes an index 0-2; mode 0 is the player's name here, **likely**), **create the save** with the entered name (`savedataCreate`), reset play time, sync the
     player's movement work, and `floorLink(0x3A0, 0)`: the **`opening_demo`** room.
   - **What the new save contains** (`savedataCreate`, `hero.c`; **verified**; item names from
     the US item table): the player record with the entered name (a default name from the executable if none),
     the two starters (`heroPokemonGetBlacky`, `heroPokemonGetEifie`, byte-matched; move names from the US move
     table): **Umbreon, level 26**, with Bite, Secret Power, Taunt and Snatch, and **Espeon, level 25**, with
     Confusion, Return, Reflect and Helping Hand. Both get **friendship 220**, a personality value rerolled until it
     fits (`pokemonCreateRndFit(pokemon, sex 0, any nature, shiny 0, player id)`: sex 0 (**likely** male) and **never
     shiny** for the player's id, **verified** from the reroll loop; the shiny test itself: rng.md, "Shiny checks"), a fixed **EXP total** (status 0x79: Umbreon 19,055, Espeon 16,372, both part of the way into their level), and are added to the
     party through `heroCatchPokemon` (ball argument 0xFE, **guess**: "no ball"). Both are recorded in the Strategy Memo
     (pda.md). Then **10,000 money**, and the bag: Potion x5, Super Potion x2, Antidote x2, Burn Heal x2,
     Ice Heal x2, Awakening x2, Parlyz Heal x2, Full Heal x2. The save header's **return point** starts at room 2
     (`S1_shop_1F`, the Outskirt Stand shop), entry 1, with the second room id 1: so a blackout before the first save
     goes to the Outskirt Stand shop (**likely**, if nothing writes the return point earlier).
   - **Continue** -> only if a save exists (save-header field 4, the save count, is not 0; otherwise a message,
     0x44EA or 0x444B, likely the memory-card states): show the player's summary (model, name, play time, ...) with
     message 0x3C22 and a yes / no; on "yes", `fn_80113FE8` warps to the **return point in the save header**, the same
     warp the field-poison blackout uses (field-steps.md). So Continue resumes in the room the save recorded.
   - **Battle Mode** -> choices 2 and 3: `menuColosseumBattle(0)` / `(3)`, which store that kind in save section
     14 (`ColosseumSaveWork.battleKind`; section 14 holds Battle Mode state) and change to
     room **0x395 `colosseumbattle_menu`**, the Battle Mode menu.
     (**verified** from the Battle Mode menu's header messages per kind, `menu_middle.c`): The
     kind is **0 Colosseum Battles, 1 Mt. Battle 100-Trainer Knockout, 2 Gang Battle, 3 Battle Now** (header text
     paraphrased). The tool-battle setup writes kind 1 before entering `toolbattle_menu` (0x397, menu-rooms.md). The game's own
     name for kind 1 is `BATTLEMODE_BATTLEYAMA100` (a retail assert string in `menuCB_SaveLoad.c`; memory-card.md).
     So the top menu's two Battle Mode entries open the menu on **Colosseum Battles** (0) and **Battle Now** (3).
     Section 14 +0x08 is the rule set: 0 anything goes, 1 level 50 cap, 2 level 100 cap, 3-5 the player's three
     edited rules (**verified** from the headers).
   - **Options** -> choice 4: `fn_80020C9C`, then back to the menu. It edits the **sound output mode**
     (`GSsndGetOutputMode`, likely mono / stereo / surround) and save-header option 9, **rumble off**
     (`gamedatasaveBios...OptionNoVibration`). Cancelling restores both. On leaving, if the rumble setting changed
     and `fn_801D04E8` holds (the memory-card task state's `serial_check_enabled` flag; what it means is not
     traced), it asks whether to save (message 0x3D82) and
     saves the options with `fn_801D0748(7, 2, 0)`. (Only option 9 is compared there; how the sound mode is stored, as
     save-header option 10 or not at all, is not traced.)
   - Anything else -> back to the title room (0x384).

   **How the top menu's entries are defined** (**verified** from the menu tables in `main.dol` and `fn_800210F0`
   unless marked). Part data, part code:
   - **Layout is data:** menu 0xAA is a record in the menu table (menus.md (not yet published)), whose item chain holds four
     non-selectable panels (items 0x90E-0x911, each drawing a sprite: the background frames) and **six selectable
     entries** (items 0x912-0x917: screen position, size and the shared draw callback `fn_80020F54`). An entry's
     choice value is its place in that chain (0-5). The items carry no message keys, so the entry labels are not
     text from the message banks (**likely** drawn as part of the sprites; not traced).
   - **Help line is data:** while the cursor rests on an entry, `fn_80020EA4` shows a one-line description from a
     table of message ids indexed by the choice (`lbl_802E4ED8`; the Options screen has its own table,
     `lbl_802E4EF0`).
   - **What each entry launches is code:** a fixed `switch` on the choice in `fn_800210F0`: New Game (the menu call
     returns 0) -> the confirmation, name entry, save creation and room 0x3A0; 1 Continue -> the save's return point;
     2 / 3 -> `menuColosseumBattle(0)` / `(3)` and room 0x395; 4 -> the Options screen (`fn_80020C9C`); any other value
     (the sixth entry, or cancel) -> back to the title room 0x384 (**likely** the sixth entry is "back"; its label not
     checked).
   - So a mod can move or restyle entries and change their help text by data, but a new entry, or a new target for an
     existing one, needs a new case in that `switch`.

6. **Opening movie** (room 0x3A0): `movieOpeningStart` fades, opens `movie/openingdemo.thp` and starts its song
   (0x495); `fn_80035E04` plays it with five frame-keyed sound cues, then writes story flag 0x8D0 and
   **`floorLink(1, 0)`**: room 1, **`S1_out`, Outskirt Stand**, entry point 0. It also sets the room change's fade
   script to `0x05960008` (`floorSetFadeScript`; **verified**), the shared fade-in that the
   battle code also uses when returning to the field (`fight.c`), so it is a generic fade, not a call to the
   room's own function 8. The movie handlers are not referenced
   from `main.dol` (the link keeps them with FORCEACTIVE), so the `opening_demo` room reaches them through its own data.
   (**verified** from the room table's relocations in `common_rel`): They are the room
   record's **C pre / main / post functions** (+0x3C / +0x40 / +0x44): room 0x3A0 = `movieOpeningStart` /
   `fn_80035E04` / `fn_80035DD4`. The same holds for every front-end room:

   | Room | Pre (on entry) | Main (every frame) | Post (on leaving) |
   |---|---|---|---|
   | 0x399 `pokemon_logo` | `movieTpcStart` | `movieTpcWait` | `fn_800365B0` |
   | 0x39A `genius_logo` | `movieGsLogoStart` | `movieGsLogoWait` | `fn_800363BC` |
   | 0x384 `title` | `fn_8002049C` | `fn_800203B4` (the attract-demo check) | `fn_80020328` |
   | 0x39C `auto_demo` | `movieStartAttract` | `fn_80036240` | `fn_80036210` |
   | 0x3A0 `opening_demo` | `movieOpeningStart` | `fn_80035E04` | `fn_80035DD4` |
   | 0x3A1 `topmenu` | `fn_80021644` | `fn_80021624` | `fn_800215C4` |

   So the **tpc movie is played by room 0x399 `pokemon_logo`**, and the top menu's
   setup is `fn_80021644`. A room's script functions run in the same three phases (step 7).
7. **Outskirt Stand** (room 1, `S1_out`; **verified** from the decoded room script on the US
   disc). On entry the script checks story flag **986**: while it is clear (the first arrival) it starts no music
   (`null_bgm_song`); later visits play `stand_song` once flag 2480 is set. The room setup (function 1) only shows or
   hides characters by flags 986 and 990. The **first-arrival cutscene** is the script's function 8, started by the room's **main loop
   (function 11**, which calls `heroMoveMain` every pass; **verified** from the script's bytecode). The loop calls
   function 8 when flag 986 is clear and the player (character group 0, id 100) is inside a box
   (`std058_characterCheckWithinBounds(0, 100, -45, -45, 0, 40)`: the floor rectangle from corner x -45, z -45 to corner
   x 0, z 40, **verified** from `people.c`; with the game's debug resource flag on, the check also draws the box as
   translucent walls 50 units tall). The player arrives at **entry point 0, (50, 0, 70) facing 180 degrees** (the room
   module's position list), which is **outside** the box: so after the opening movie the player has control and must
   walk about 60 units towards the stand (roughly 8 footsteps of 7 units) before the scene plays (**verified**
   coordinates; the walk itself not tried). The same loop calls **function 9, the Willie battle, when flag 988 is set
   and flags 2650 and 990 are clear**, and function 21 when flag 2575 is set and 2758 clear (a later-story scene: a TV
   reporter at the stand, played once; it sets 2758).

   **Function 8** (**verified** from the bytecode unless marked): camera cuts (`GSscene_SetMode(3)`,
   `std040_cameraSetPosition`, `cameraSetRotation`), other functions of the same script run as **parallel tasks**
   (`scriptExecTask(0x0100000N, ...)`; script ids are "group in the high half, function in the low half", **verified**
   from `cms_ScriptCreateThread`, so `0x0100000N` is function N of the room's own script, **likely**) with
   `std006_threadWaitForCompletion` waiting for them, two lines of dialogue (`std054_characterTalk` /
   `std055_characterEndSpeech`) with talk animations, song 90, three characters hidden (`peopleSetVisible`), then
   **flag 986 set** so it never replays, the characters' event flags restored, the camera handed back (`cameraReturn`)
   and **collision object 10 switched off** (`scriptSetCol(10, 0)`, also done by function 2 on later visits).
   Collision object 10 has **only walls** (player and NPC walls, no trigger planes; collision.md), so it is an
   **invisible barrier** that stays up until the cutscene has played. The room's two walk-through triggers (ids 0 and
   4) are **exits**: their hero-event records run the shared script's function 4 with room 0x38E (`world_map`) and room
   2 (the shop) (common-script.md). See step 8 for the first battle.

   **What a re-implementation needs for this room** (the script's command use, counted): 37 different native commands, most
   often flag checks and sets (`std081` x38, `std078` x16, `std079`), character talk / animation / position / turn /
   move (`std054` x28, `std051` x25, `std055` x17, `std059` x11, `std068` x9, `std061` x7), camera position (`std040`
   x6), background music (`std104` x6), event flags (`std076` / `std077`), task threads (`std006`, `std071`,
   `std209`), sleeps (`std157`), map objects (`std113`, `std114`, `std115`), the battle result (`std103`) and one
   yes / no dialog (`std011`), plus the non-numbered natives above (`scriptExecTask`, `GSscene_SetMode`,
   `cameraSetRotation`, `peopleWaitSyncMotion`, `peopleSetVisible`, `scriptSetCol`, `cameraReturn`).

   **How a room's script functions run** (`floor.c`; the room record's five script slots, room
   table +0x28 to +0x38; for `S1_out` functions 0, 1, 2, 11, 3):
   - +0x28 (function 0, sound setup): started in its **own script thread** as soon as the room's script file has
     loaded (`fn_80112700` -> `fn_800F7318(priority 0xF, ...)`; **verified**).
   - +0x2C (function 1, characters shown or hidden by flags): run **synchronously** during room setup
     (`cmsRunSync`, **verified**).
   - +0x30, +0x34, +0x38 (functions 2, 11, 3): filed in the room task's **three queues** next to the room's C
     functions: queue 1 with the room's pre-function, queue 2 with its main function and the floor update, queue 3
     with its post-function (`fn_800FEF8C` / `fn_800FEE68` / `fn_800FED3C`, **verified** pairing). So +0x30 is
     **likely** the "on entry" script, +0x34 the room's **main loop** (S1_out's function 11 loops on
     `heroMoveMain` forever, which fits) and +0x38 the "on leaving" script (function 3 stops sounds, which fits).
   **The shop** (`S1_shop_1F`, room 2; **verified** from its decoded script and room record):
   - Its room record names script functions 0, 1, 2, 4, 3 (+0x28 .. +0x38), so function 4 is the shop's **main
     loop** and function 3 its **on-leaving** script (the one that sets flag 988, the Willie trigger).
   - On entry: function 0 plays the stand's song when the player came from room 1 (`floorGetPrevFloorID`); function
     1 sets the TV channel by story progress and shows or hides a character by flag 990; function 2 places the
     clerk and the camera and, under flag 1144, sends a mail.
   - **First visit** (flag 987 clear): the main loop first plays a short **TV news scene** (a camera animation, the
     news jingle, `std182_tvNewsEvent(2)`; tv-news.md), then sets 987 so it plays once.
   - **The clerk** (function 8): greets a newcomer, or advises stocking up once Willie is beaten (flag 990), then
     opens the shop (`menuShopOpen`, through function 16, with a story-dependent shop number; shops.md).
   - **A later scene** (function 15, called by the main loop under story flags 997 / 1000 / 2462 / 1161, so not
     part of the opening; **likely** once the partner travels with the player): she asks for Poké Balls, the clerk
     finds a dusty box, gives the player **5 Poké Balls** (`scriptGetItem(4, 5)`) and stocks them from then on
     (it opens shop 1; this supports shops.md's "shops 0-2, stock growing to balls", **likely**).
   - Talking to one customer (function 14, a trainer who guesses the player is one too) also sets flag 988, so
     Willie can be triggered without leaving through the door's on-leaving script first (**likely**; he still only
     runs up once the player is back outside, since the check is in `S1_out`'s main loop).
   - Entering from Outskirt Stand puts the player at the shop's entry point 0 or 1 (two points in the room module;
     which one the door uses is not traced).
8. **The first battle** (**verified** from the decoded `S1_out` script and the US battle and
   trainer tables). *How it starts (added 2026-10-16, **verified** from the bytecode):* leaving the Outskirt Stand
   shop (`S1_shop_1F`, its on-leaving function 3; a talk inside the shop also does it) sets story flag **988**;
   back outside, the room's main loop calls function 9 while 988 is set and 990 / 2650 are clear, so Willie runs
   up as the player comes out. (The battle entry itself, step by step: battle-engine.md, "The battle entry".) Function 9 of the script sets story flag 989, then **`fieldStartEncount(10, 1, 0)`**: battle 10,
   an ordinary trainer battle (kind 2) against **WILLIE** (trainer record 10: two Zigzagoon, level 24). (**verified** from the US trainer, trainer-Pokémon and
   AI tables; field meanings from the AI record's field names, battle-turn-flow.md section 7, **likely**): Both
   Zigzagoon have all IVs and EVs at 0 and no held item, one knows Tackle and Growl, the other Tackle and Tail Whip.
   Willie's AI record (id 81) has a **0% switch chance** (he never switches), a 25% item chance, but **no items**: a trainer record's field 6 (+0x14, up to 8 item ids) is its bag
   (**likely**: the 39 trainers that have entries use only Gen 3's battle items, healing items and the X items), and
   Willie's is empty, so the item step never picks anything, no combo roll, no score jitter, and picks its **target at random**, with
   type, ability, damage and accuracy checks on. So a faithful run shows two Zigzagoon using Tackle and the two
   stat-lowering moves on random targets (his defeat line, "wiped out ... still need seasoning" paraphrased, is the third of the
   trainer record's four battle messages, field 8), never switching, never using items: an easy first check for a re-implementation's battle and AI. After the battle the script reads the result
   (`std103_battleGetResult`): when it is 2 (the first, host side won: battle-turn-flow.md, "Battle result ids") Willie speaks and flag 990 is set. A second
   battle against Willie, battle 13, is started later in the script with `fightStartEncount(13)` under other story
   flags (**likely** a rematch; its trainer record, 11, has a Zigzagoon at 29 and a Linoone at 30). For battle 10 the arena's room is **0xC8, `S1_out_bf`** (arena 5), the battle copy of the
   Outskirt Stand map (battle-arenas.md, "Battle rooms").

For comparison: after the staff roll, `fn_80035F64` (when the previous room was 0x76 and flag 0x476 is set) sets
flag 0x478, puts the save header's return point at room 2 entry 1, sets the partner-following flag, saves, and links
to room 2 (`S1_shop_1F`, entry 1), the Outskirt Stand shop: the post-game starts there.

## Story flags on the way (meanings from where each is set, so **likely** unless noted)

| Flag | Set by | Meaning |
|---|---|---|
| 0x8D0 (2256) | the opening movie's end (`fn_80035E04`) | the opening movie has been shown (**verified** as its setter) |
| 986 | `S1_out` function 8 | the Outskirt Stand arrival cutscene has played (gates it; **verified** from the script) |
| 988 | `S1_shop_1F` function 3 (leaving the shop) and a shop talk | the player has been in the shop; makes the main loop start the Willie battle (**verified**) |
| 989 | `S1_out` function 9, before the battle | the first battle (Willie) has started |
| 0x9B0 (2480) | `fightStartEncount`, at every battle start | a battle has been fought; `S1_out` plays `stand_song` once it is set. It is a **per-room** flag (class 1, flags.md (not yet published)), so it means "on this visit": cleared at the next room change (**verified**: flag table and a Dolphin flag read) |
| 990 | `S1_out` function 9, after a won battle | Willie has been beaten |

## Data structures
- The top menu's choice: `lbl_8047A360` (1 Continue, 2 / 3 Battle Mode, 4 Options, -1 cancel; New Game is the menu
  call's own return value 0), its state machine in `fn_800210F0`.
- Room ids: 0x399, 0x39A, 0x384, 0x3A1, 0x3A0, 1, and 0x39C for the attract demo (room table; names from the
  archive table).

## Changing it
- Port: the front end is all room changes, so a port can boot straight into any stage for testing (for example
  `floorLink(1, 0)` with a created save to reach Outskirt Stand, or `floorLink(0x3A0, 0)` for the opening).
- A "skip intro" option: the movie handlers are the hook (movies.md "Hook point for the port").
- New Game's steps (name entry, save creation) must run before room 1, or the save and the player's name are empty.

## Open questions
- The condition `fn_8006A718(0)` behind the second warning wording.
