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
   After START: state 0x1F (a short wait), state 0xC8 **loads the memory card** (`memcardExecTask(1, 2, 0)`; -1 = back to
   waiting; any result but 3 creates an empty save, so 3 is **likely** "a save was loaded"), then state 0x29 **`floorLink(0x3A1, 0)`: the `topmenu` room**,
   where the top menu runs.

   *The memory-card call* (`memcardExecTask(kind, work size, channel)`, `memcard.c`, **verified**; memory-card.md): it starts memory-card
   task `kind` as a per-frame task (`GSgappCreate(.., memcardTaskMain)` (0x801CDB04)), allocates a 0x1E600-byte work buffer and a
   0xA000-byte card work area, and **yields until the task reports a non-zero result**. Kind 1 is this title load;
   kind 4 (ten callers, **likely** the normal save) and kind 7 (the options save) are others. Results seen in the task code: **4** after a write
   completes (the save count is incremented there), **-1** from the abort state, and 1, 2, 3 and 5 from other check
   paths (not decoded). *For the port*: this is
   the next place the title can stall after the song (**likely**): the memory-card task needs a backend that
   finishes with a result (3 for a loaded save, or a value meaning "no save" so an empty one is created).

   **The title song drives the attract demo:** a per-frame function (`fn_8001EF78`) checks, in every state from
   0x1E on except 0xC8, whether the title song is still playing (`GSsndGetStatus(0x46A)`, 0x801666BC); when it is not, it sets
   the state to 0x28, and the title's main loop (`titleMain`) then links **room 0x39C, `auto_demo`** (the attract
   demo). So on the GameCube the demo starts when the title song ends. *For the port* (**likely**, from the code, not
   run): if the title song never starts, `GSsndGetStatus(0x46A)` reads 0 from the first frame of state 0x1E, so the
   title leaves for the attract demo right after the logos, or overwrites the state set by START, and the top menu is
   never reached; the title song (or a stand-in that reports "playing") is on the critical path.
4. **Top-menu room** (0x3A1, `topmenu`): its setup (**likely** `titleIntroStart` (0x80021644), which nothing in the C calls; it runs
   only when `fn_800FF548()` is 0) starts **`worldmap_song`** (sound 0x446; the
   song's name from the disc's song list), loads the top menu's background model and plays camera animation
   0x632 (the Orre map fly-over behind the menu, **likely**). The top menu itself is `titleMenuMain` (0x800210F0), reached through
   the wrapper `titleTopMenuMain` (**verified**: it is the `topmenu` room's main callback in the room table; nothing in `main.dol` calls it).
5. **Top menu** (`titleMenuMain` (0x800210F0), menu 0xAA, choice in `lbl_8047A360`):
   - **New Game** (taken when the menu call itself returns 0, **likely** "the first item chosen") -> a confirmation: with no save, a plain "start a new story?" (message 0x3C3A); when a save exists, a
     warning that it will be deleted (0x3C35, or the same warning in another wording, 0x44C9, when
     `fn_8006A718(0)` is 1: not traced); on "yes": close the menus, start background music 1, **name entry** (name-entry.md; `menuNameEntryOpen(0, 0)`, menu room
     `name_entry_menu`; the call takes a mode: 0 / 1 need no index, 2 names a party Pokémon and checks the slot,
     3 takes an index 0-2; mode 0 is the player's name here, **likely**), **create the save** with the entered name (`savedataCreate`), reset play time, sync the
     player's movement work, and `floorLink(0x3A0, 0)`: the **`opening_demo`** room.
   - **What the new save contains** (`savedataCreate`, `hero.c`; **verified**; item names from
     the US item table): the player record with the entered name (a default name from the executable if none:
     **"レオ" (Leo, the hero's Japanese name)** in Shift-JIS, `lbl_8047D028`; **verified** from code),
     the two starters (`heroPokemonGetBlacky`, `heroPokemonGetEifie`, byte-matched; move names from the US move
     table): **Umbreon, level 26**, with Bite, Secret Power, Taunt and Snatch, and **Espeon, level 25**, with
     Confusion, Return, Reflect and Helping Hand. Both get **friendship 220**, a personality value rerolled until it
     fits (`pokemonCreateRndFit(pokemon, sex 0, any nature, shiny 0, player id)`: sex 0 is **male** and **never
     shiny** for the player's id, **verified** from the reroll loop; "male" is now verified too:
     sex 1 is the branch taken when the species' ratio is above the value's low byte, the Gen 3 female rule, so sex 0
     is male. How the value is drawn and when it is redrawn: rng.md, "How a personality value is drawn"; the shiny
     test itself: rng.md, "Shiny checks"), a fixed **EXP total** (status 0x79: Umbreon 19,055, Espeon 16,372, both part of the way into their level), and are added to the
     party through `heroCatchPokemon` (ball argument 0xFE). Both are recorded in the Strategy Memo
     (pda.md).
   - **Random numbers used** (**verified** by a per-call RNG trace and Dolphin's RNG state at the
     top menu): creating the save takes **1,016 draws** in the usual case, all in one frame: the Strategy Memo's reset
     gives each of its 500 slots a random 32-bit id (1,000 draws), the player's trainer ID takes 2, and each starter
     takes 5 in `pokemonCreate` (a first personality value of 2 draws, two IV words, the ability roll) plus 2 per
     attempt in `pokemonCreateRndFit`, whose value replaces the first one (more attempts when a redraw is needed). The same save creation also runs before the logo and when the title screen leaves for the top menu, so
     the RNG passes through this burst three times before the opening movie. Then **10,000 money**, and the bag: Potion x5, Super Potion x2, Antidote x2, Burn Heal x2,
     Ice Heal x2, Awakening x2, Parlyz Heal x2, Full Heal x2. The save header's **return point** starts at room 2
     (`S1_shop_1F`, the Outskirt Stand shop), entry 1, with the second room id 1: so a blackout before the first save
     goes to the Outskirt Stand shop (**likely**, if nothing writes the return point earlier).
   - **Continue** -> only if a save exists (save-header field 4, the save count, is not 0; otherwise a message,
     0x44EA or 0x444B, likely the memory-card states): show the player's summary (model, name, play time, ...) with
     message 0x3C22 and a yes / no; on "yes", `floorReturnToSavedRoom` (0x80113FE8) warps to the **return point in the save header**, the same
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
     **Where the teams come from** (**verified** by walking the menus in an emulator, with the
     menu code `menuColosseumBattle.c`):
     - The top menu shows two panels, **Story Mode** (Continue, New Game) and **Battle Mode** (Colosseum Battle,
       Battle Now), plus Options and Quit; with no story save the cursor starts on New Game.
     - **Colosseum Battle** first asks "How many will battle?": **Solo Battle** or **Gang Battle**. Solo Battle's menu
       has Colosseum Battle, Mt. Battle vs 100, Continue (a saved run), **Register Pokémon** and PokéCoupon Exchange.
       Colosseum Battle and Mt. Battle refuse until a team is registered ("You may not proceed because you have not
       registered your Pokémon": message 0x3BFE, shown when no battle-mode player record is saved).
     - **Register Pokémon** offers two sources: **Colosseum Pokémon**, the party of the Story Mode save on the memory
       card in Slot A (`heroBiosCopy` from the save's player section; refused with "no Story Mode save data on the
       Memory Card in Slot A", 0x444D, when there is none), and **GBA Pokémon**, Pokémon from a linked Game Boy
       Advance game (a four-trainer party picker, `fn_800849B4`).
     - **Battle Now** needs no save: Single Battle or Double Battle, then the opponent's difficulty (Ultimate, Hard,
       Normal, Easy), then it offers a **ready-made team** under a trainer name (in the run checked: six level 50
       Pokémon) with "Would you like to enter with these Pokémon?". How the teams are chosen (**verified**
       from the code, battle-mode.md "Battle Now"): both teams are Mt. Battle trainers' teams, drawn from eight per type and rank with the shared random number
       generator (the player's is a different one of the eight, renamed with one of three names), with every held
       item removed; "No" goes back, and choosing again draws new teams (**likely**).
     - So a battle with a chosen team needs a Story Mode save (its party) or a GBA game; Battle Now gives fixed or
       random ready-made teams without one.
     - **Registration checks the party against the rules right away** (**verified** on screen
       with two save copies): "One or more POKéMON don't meet the rules", then the reason, then "Please restart the
       Story Mode and change the party POKéMON to those that are allowed under the rules." Reasons seen: **"There are
       not six POKéMON in your party!"** (a five-Pokémon party) and **"At least two POKéMON are holding the same kind
       of item!"** (an item clause). The offending Pokémon (or the empty slot) are marked in red. Two level-100
       Pokémon in the five-Pokémon party were not marked, so level 100 likely passes registration (**likely**: only the
       first failing rule may be shown). A six-Pokémon party with three level-100 Pokémon,
       all holding nothing, **registers** ("Is it okay to register POKéMON?", then "Data has been saved to the Memory
       Card in Slot A!", "Registration completed!"; **verified**). So level 100 passes, six empty hands don't break
       the item clause, and moves from a GBA move tutor (Fire Punch, Ice Punch) trip no rule.
     - **Mt. Battle vs 100 in Battle Mode** (**verified** on screen, one run): Single or Double
       Battle, then "Would you like to enter with these POKéMON?", then the field: the player starts on Mt. Battle's
       bridge (room 0x4C) and walks up to the first trainer, who challenges him. The battle runs through room 0x397
       (`toolbattle_menu`). A vs screen ("MT. BATTLE-VS 100 / Trainer No.: 1", here GLASSES MAN ROOD) comes before a
       "Please select 3 POKéMON." screen (singles).
       - **The player's Pokémon keep their own levels** (levels 27, 25, 100, 100, 100 and 1 shown unchanged on the
         selection screen).
       - **The opponent's Pokémon were all level 100**, even weak species (Sunkern, Magikarp, Wurmple, Azurill, Ralts,
         Silcoon). So Mt. Battle likely sets its opponents' level from the player's team (its highest level, or the
         rule's cap): **likely**, from one trainer.
       - No continue count was shown before the first battle.
     - **Colosseum Battle's list** (**verified** on screen, one run): Phenac Stadium, Pyrite
       Colosseum and The Under Colosseum at Lv50, Tower Colosseum at Lv100, two entries shown as "?????" / "Lv???",
       Exit and Check rules; the selected colosseum shows its trainer level ("Easy" for Phenac Stadium) and a Single /
       Double Battle record. Then Single or Double Battle, "Would you like to enter with these POKéMON?", a save, and
       a vs screen ("PHENAC STADIUM / Battle 1") in room 0x397. A registered team of six level-50 Pokémon with six
       different held items was accepted at Phenac Stadium.
       - Every record shows a **"Champ!"** label from the start (Phenac Stadium's Single and Double read "Champ!"
         even on a save from just after Phenac's first visit); the label **glows and sparkles** once that challenge
         has been completed (**verified** in play). So the early "Champ!" is the normal look, not a save problem.
         Where the completion records live is not traced.
   - **Options** -> choice 4: `titleNewContinueSelect` (0x80020C9C), then back to the menu. The screen has three rows:
     Sound (Stereo / Mono), Rumble (On / Off) and Done, with a help box at the bottom (**verified**, a screenshot of
     the screen). It edits the **sound output mode**
     (`GSsndGetOutputMode`; the screen offers only mono and stereo) and save-header option 9, **rumble off**
     (`gamedatasaveBios...OptionNoVibration`). Cancelling restores both. On leaving, if the rumble setting changed
     and `memcardGetFlag41()` (0x801D04E8) holds (the memory-card task state's `serial_check_enabled` flag; what it means is not
     traced), it asks whether to save (message 0x3D82) and
     saves the options with `memcardExecTask(7, 2, 0)`. (Only option 9 is compared there; how the sound mode is stored, as
     save-header option 10 or not at all, is not traced.)

     **How the Options screen is built** (**verified** from the menu and item tables and the screen's code, 2026-11-13;
     positions in the game's 640 x 480 screen). It is six menus opened together, all data except a few callbacks:
     - **0x80, the rows:** three row panels (items 0x50E-0x510, sprite 0x197, 627 x 53, at y 116 / 192 / 267, so 76
       apart); a highlight bar over each (items 0x511-0x513, sprite 0x195), whose callback sets a flag on it when the screen's
       selection equals that row's number (0 Sound, 1 Rumble, 2 Done; **likely** the "shown" flag) and scales it
       between 1.0 and 1.2 on a sine (a gentle pulse); a second overlay per row (items 0x514-0x516, sprite 0x196); the three labels as **pictures** (items
       0x517-0x519: sprite 0x191 "Sound", 0x190 "Rumble", 0x18F "Done", each about 35 high, 13 below its panel's top);
       and the controller picture (item 0x51A, sprite 0x193), whose callback copies the state record's swing value into
       it (**likely** its rotation: the controller wiggles when rumble is turned on).
     - **0x7B, the choices:** the four words as message text (STEREO and MONO on the Sound row, ON and OFF on the
       Rumble row: message keys 0x1900-0x1903, 14 below their panel's top), and four circle positions (items
       0x522-0x525, 127 x 41). Its draw callback (`menuOptionDrawSoundMode`) draws the circle sprite 0x192 at the
       position for each row's current value. Its cursor callback (`fn_80020618`) moves the selection with up / down
       (0 to 2) and applies left / right to the row under it: Sound sets the output mode, Rumble sets save option 9
       (and gives a short rumble when turned on).
     - **0x7D and 0x7E:** the speaker pictures (sprite 0x194), shown as two or one by the current sound mode.
     - **0x7C:** the "OPTIONS" title bar (sprite 0x263 across the top, y 40, 56 high).
     - **0x7F:** the help box (three sprites, y 349) and its text, drawn from the selection (`titleDrawOptionHelp`,
       the message table `lbl_802E4EF0`, one key per row).
     The selection lives in a small state record (`lbl_803A1FC8`: the selection first, then the rumble-shake and pulse
     timers). The screen's loop (`titleNewContinueSelect`) reopens menu 0x7B until A is pressed with the selection on
     Done (2). **Changing it:** a new row needs a panel, a highlight, a label picture and its value items in the tables,
     the cursor callback's row count, and a help line. The tables are live data, so a port can move and add items at
     start-up; the row numbers are fixed in the callbacks, so new rows need a hook there.
   - Anything else -> back to the title room (0x384).

   **How the top menu's entries are defined** (**verified** from the menu tables in `main.dol` and `titleMenuMain` (0x800210F0)
   unless marked). Part data, part code:
   - **Layout is data:** menu 0xAA is a record in the menu table (menus.md (not yet published)), whose item chain holds four
     non-selectable panels (items 0x90E-0x911, each drawing a sprite: the background frames) and **six selectable
     entries** (items 0x912-0x917: screen position, size and the shared draw callback `titleDrawContinueInfo` (0x80020F54)). An entry's
     choice value is its place in that chain (0-5). The items carry no message keys, so the entry labels are not
     text from the message banks (**likely** drawn as part of the sprites; not traced).
   - **Help line is data:** while the cursor rests on an entry, `titleDrawOptionHelp` (0x80020EA4) shows a one-line description from a
     table of message ids indexed by the choice (`lbl_802E4ED8`; the Options screen has its own table,
     `lbl_802E4EF0`).
   - **What each entry launches is code:** a fixed `switch` on the choice in `titleMenuMain` (0x800210F0): New Game (the menu call
     returns 0) -> the confirmation, name entry, save creation and room 0x3A0; 1 Continue -> the save's return point;
     2 / 3 -> `menuColosseumBattle(0)` / `(3)` and room 0x395; 4 -> the Options screen (`titleNewContinueSelect` (0x80020C9C)); any other value
     (the sixth entry, or cancel) -> back to the title room 0x384. (**verified** from a picture of
     the menu in the port): The sixth entry is labelled **"Quit"**, a small button right of Options; the screen shows
     the Story Mode panel (Continue, New Game), the Battle Mode panel (Colosseum Battle, Battle Now), Options and Quit,
     with a help line under them ("Continue the Story Mode from where it was last saved.").
   - So a mod can move or restyle entries and change their help text by data, but a new entry, or a new target for an
     existing one, needs a new case in that `switch`.
   - (**verified**, from the tables in `main.dol` and the menu code): **the pieces and the glove.**
     The Story and Battle panels are 269 x 249 at (43, 43) and (330, 43); OPTIONS is one picture 209 x 65 at
     (182, 288), its word and controller drawn in; Quit is one picture 143 x 50 at (448, 296), **its word part of the
     pellet**. The six selectable entries are invisible positions for the **glove**: the selected entry's cursor
     sprite (the glove, 70 x 46, with its own small animation) is drawn at that entry's position, so the glove's
     places are New Game (40, 233), Continue (42, 198), Colosseum Battle (288, 198), Battle Now (319, 234), Options
     (125, 297) and Quit (395, 297). **Moving the cursor** is the menu system's general rule, not code of this
     screen: a direction picks the nearest selectable entry that way (left / right look first within 12 px of the
     same height, up / down within 16 px of the same column, widening until one is found), so an entry added to the
     chain is reachable with no new code, and its choice number is its place among the selectable entries. **The
     help table has exactly six slots**, one per entry; the Options screen's table follows it in memory. The menu
     returns the choice; the help box is a separate menu (0x7F) drawn while it waits.

6. **Opening movie** (room 0x3A0): `movieOpeningStart` fades, opens `movie/openingdemo.thp` and starts its song
   (0x495); `movieOpeningPlay` plays it with five frame-keyed sound cues, then writes story flag 0x8D0 and
   **`floorLink(1, 0)`**: room 1, **`S1_out`, Outskirt Stand**, entry point 0. It also sets the room change's fade
   script to `0x05960008` (`floorSetFadeScript`; **verified**), the shared fade-in that the
   battle code also uses when returning to the field (`fight.c`), so it is a generic fade, not a call to the
   room's own function 8. The movie handlers are not referenced
   from `main.dol` (the link keeps them with FORCEACTIVE), so the `opening_demo` room reaches them through its own data.
   (**verified** from the room table's relocations in `common_rel`): They are the room
   record's **C pre / main / post functions** (+0x3C / +0x40 / +0x44): room 0x3A0 = `movieOpeningStart` /
   `movieOpeningPlay` / `movieOpeningExit`. The same holds for every front-end room:

   | Room | Pre (on entry) | Main (every frame) | Post (on leaving) |
   |---|---|---|---|
   | 0x399 `pokemon_logo` | `movieTpcStart` | `movieTpcWait` | `movieTpcExit` |
   | 0x39A `genius_logo` | `movieGsLogoStart` | `movieGsLogoWait` | `movieGsLogoExit` |
   | 0x384 `title` | `titleInit` | `titleMain` (the attract-demo check) | `titleExit` |
   | 0x39C `auto_demo` | `movieStartAttract` | `moviePlayAttract` | `movieExitAttract` |
   | 0x3A0 `opening_demo` | `movieOpeningStart` | `movieOpeningPlay` | `movieOpeningExit` |
   | 0x3A1 `topmenu` | `titleIntroStart` (0x80021644) | `titleTopMenuMain` | `titleTopMenuExit` |

   So the **tpc movie is played by room 0x399 `pokemon_logo`**, and the top menu's
   setup is `titleIntroStart` (0x80021644). A room's script functions run in the same three phases (step 7).
7. **Outskirt Stand** (room 1, `S1_out`; **verified** from the decoded room script on the US
   disc). On entry the script checks story flag **986**: while it is clear (the first arrival) it starts no music
   (`null_bgm_song`); later visits play `stand_song` once flag 2480 is set. The room setup (function 1) only shows or
   hides characters by flags 986 and 990. The **first-arrival cutscene** is the script's function 8, started by the room's **main loop
   (function 11**, which calls `heroMoveMain()` every pass; **verified** from the script's bytecode). The loop calls
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
     loaded (`floorStartSoundScript` (0x80112700) -> `_vmThreadCreate(priority 0xF, ...)` (0x800F7318); **verified**).
   - +0x2C (function 1, characters shown or hidden by flags): run **synchronously** during room setup
     (`cmsRunSync`, **verified**).
   - +0x30, +0x34, +0x38 (functions 2, 11, 3): filed in the room task's **three queues** next to the room's C
     functions: queue 1 with the room's pre-function, queue 2 with its main function and the floor update, queue 3
     with its post-function (`fn_800FEF8C` / `fn_800FEE68` / `fn_800FED3C`, **verified** pairing). So +0x30 is
     **likely** the "on entry" script, +0x34 the room's **main loop** (S1_out's function 11 loops on
     `heroMoveMain()` forever, which fits) and +0x38 the "on leaving" script (function 3 stops sounds, which fits).
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

For comparison: after the staff roll, `movieStaffrollMain` (when the previous room was 0x76 and flag 0x476 is set) sets
flag 0x478, puts the save header's return point at room 2 entry 1, sets the partner-following flag, saves, and links
to room 2 (`S1_shop_1F`, entry 1), the Outskirt Stand shop: the post-game starts there.

## The save and load screens, as the player sees them 

Captured screen by screen in the PC port (the decompiled code running natively) build, which runs the game's own code. **Verified** from pictures of each step;
message keys are from the US disc's banks, and each message is described here, not quoted.

- **Starting with no save on the card** (the title, then START):
  1. the game says it found no Colosseum save file on the card in Slot A (key 0x3BFA, first part);
  2. it asks whether to create one (the same message's second part), with a YES / NO box, the cursor on YES;
  3. it reports the file was created in Slot A (0x3BFD; creating it is the burst of about 1,000 random draws,
     random-numbers page);
  4. the main menu opens with the cursor on New Game; the help line says the story starts from the beginning.
- **Starting with a save on the card:** the game says the card in Slot A was read, then opens the main menu with the
  cursor on Continue (help: continue from where you last saved).
- **New Game from the main menu:**
  - with a story save on the card, it warns that a new game deletes the current save file on the card in Slot A and
    asks if that is okay (YES / NO, the cursor on NO);
  - with only the empty file (just created), it simply asks whether to start a new story (the cursor on NO);
  - then the name screen: a list (New Name and three ready-made names), and New Name opens the letter grid (seven
    letters; capitals, digits and symbols, X for lower case, R to switch case, B back, START done).
- **Saving in the game** (at a PC; **verified** from the code, `pcMenuSave`; pictures not captured yet):
  1. a save window opens (menu 0x28) with the save prompt (key 0x3C01) and the play time filled in (the Time
     control code), and a YES / NO box, the cursor on YES;
  2. NO (or B) closes it; YES reads the card's save first (card task kind 3) to compare it with the game in memory;
  3. if the card holds a save of the same player (the same hero id) or the game has never been saved, it asks
     whether to overwrite the current save file (key 0x1B60), the cursor on NO; NO goes back to step 1. The time
     spent on this question is taken out of the play time;
  4. it sets the save's return point to the current room and writes (task kind 4), which shows the "saving, don't
     touch the card" and "saved" messages (keys 0x3C04, 0x3C06) from the card task itself.
- **Every flow that touches the save** (from the card tasks' callers, memory-card.md "Task
  kinds"; **verified** for the callers, the screens of D-H not yet captured):
  - A, start-up: read the save (kind 1); with none, offer to create it (the first-save create);
  - B, the main menu: Continue (asks whether to continue from the last save) or New Game (the two warnings above),
    then the name screen;
  - C, saving in the game: ask, read the card's save for the overwrite check (kind 3), ask to overwrite, write
    (kind 4), report;
  - D, after the staff roll: the same save (kinds 3 and 4), with the return point set to the post-game start;
  - E, the Options screen: if rumble changed, ask whether to save it (kind 7); captured: leaving with Done after a
    rumble change asks whether to save the settings to the card in Slot A (key 0x3D82), the cursor on NO;
  - F, Battle Mode: saves from a temporary save image (kinds 5 and 6). **Verified** from `menuColosseumBattle.c`:
    after a Colosseum Battle run that earned Poké Coupons, it asks whether to save (key 0x3C03, the cursor on YES);
    NO asks again with a warning (0x3D54, the cursor on NO), and declining there takes the coupons just won back
    off; registering Pokémon for Battle Mode is saved too (kind 5);
  - G, trading: the card and the GBA Game Pak saved together (kind 8);
  - H, the error screens any of them can show: no card, a corrupted card or one to format, a device that isn't a
    card, too little free space, a failed save, then "continue without saving?".
  
- **The card wording** (verified, a search of the US banks): about 35 system messages in the shared bank
  (`common_rel` bank 5, keys 0x1771-0x17A6, 0x1B5D-0x1B60 and 0x3BFA-0x3C06) name the memory card, Slot A / B,
  formatting, free blocks and "don't touch the POWER Button". The e-Card tool's bank and the trading menu have their
  own. Each is a message key, so a PC wording can replace them through the message hooks
  without touching the game's code.

## Story flags on the way (meanings from where each is set, so **likely** unless noted)

| Flag | Set by | Meaning |
|---|---|---|
| 0x8D0 (2256) | the opening movie's end (`movieOpeningPlay`) | the opening movie has been shown (**verified** as its setter) |
| 986 | `S1_out` function 8 | the Outskirt Stand arrival cutscene has played (gates it; **verified** from the script) |
| 988 | `S1_shop_1F` function 3 (leaving the shop) and a shop talk | the player has been in the shop; makes the main loop start the Willie battle (**verified**) |
| 989 | `S1_out` function 9, before the battle | the first battle (Willie) has started |
| 0x9B0 (2480) | `fightStartEncount`, at every battle start | a battle has been fought; `S1_out` plays `stand_song` once it is set. It is a **per-room** flag (class 1, flags.md (not yet published)), so it means "on this visit": cleared at the next room change (**verified**: flag table and a Dolphin flag read) |
| 990 | `S1_out` function 9, after a won battle | Willie has been beaten |

## How long each step takes on the console (**verified** in Dolphin from one recorded route)

Frames between room changes (60 per second): logo → title **437**; title → top menu **3,506**; top menu → name entry
**1,224**; name entry → the opening **1,071**; the opening → Outskirt Stand **3,882**. Several of these include the
time the route spent before pressing a button, so they are the route's times, not minimums. The PC's field room
changes land within 2 frames of these.

**Leaving Outskirt Stand for the world map:** the town-boundary exit (pass event 0) at the south edge, at about
z = 96 (**verified** from a PC log of the exit triggers).

## Data structures
- The top menu's choice: `lbl_8047A360` (1 Continue, 2 / 3 Battle Mode, 4 Options, -1 cancel; New Game is the menu
  call's own return value 0), its state machine in `titleMenuMain` (0x800210F0).
- Room ids: 0x399, 0x39A, 0x384, 0x3A1, 0x3A0, 1, and 0x39C for the attract demo (room table; names from the
  archive table).

## Changing it
- Port: the front end is all room changes, so a port can boot straight into any stage for testing (for example
  `floorLink(1, 0)` with a created save to reach Outskirt Stand, or `floorLink(0x3A0, 0)` for the opening).
- A "skip intro" option: the movie handlers are the hook (movies.md "Hook point for the port").
- New Game's steps (name entry, save creation) must run before room 1, or the save and the player's name are empty.

## Open questions
- The condition `fn_8006A718(0)` behind the second warning wording.
