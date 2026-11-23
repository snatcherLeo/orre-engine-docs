# Colosseum challenges (story mode)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). How the story's Colosseum tournaments are put together: which rooms run
them, how a run of battles is chained, and how the battle kinds give them their rules. **Verified** from the decoded
US room scripts and the battle table unless marked. Battle Mode's own Colosseums are separate (new-game-flow.md,
the top menu's Battle Mode choice).

## What it does

A Colosseum challenge is a **run of four battles in a row** against different trainers, the last one a boss, with
the party healed between rounds. Losing any round ends the run. There is no tournament engine in the code: each
Colosseum's **entrance room script** keeps the battle ids of its cups in script globals and runs the rounds itself
with `fieldStartEncount`; the battle kinds supply the rules (money pooled to the end, a boss round).

## The Colosseums found

| Colosseum (arena room) | Arena id | Run by the script of | Battles in the battle table |
|---|---|---|---|
| Phenac City's stadium (`M1_water_colo`) | 33 | `M1_stadium_1F` | kinds 6 (opponents) and 7 (the boss): 12 + 4 |
| Pyrite Town's Colosseum (`M2_earth_colo`) | 34 | `M2_enter_1F` | kinds 3 and 5 |
| The Under's Colosseum (`M4_cylinder_colo`) | 35 | `M4_enter_1F` | kinds 3 and 5 |
| The Under's lower Colosseum (`M4_bottom_colo`, **likely** the Deep Colosseum) | 36 | `M4_enter_B1` | kinds 3 and 5 |

In all, 85 battles carry the story-Colosseum kinds (48 kind 3, 21 kind 5, 12 kind 6, 4 kind 7). The story's other
Colosseum-style venues (Mt. Battle's, the Realgam Tower ones) use other scripts and are not covered here yet.

## How a run works (`M4_enter_B1`, the clearest case)
1. On entering, the room's script fills **five sets of four battle ids** into globals (slots 2-5, 6-9, 10-13,
   14-17, 18-21). In each set, the first id is a **boss battle (kind 5)** and the other three are ordinary
   Colosseum battles (kind 3), all on the same arena.
2. When the player enters the arena (a scripted walk through two doors, opened and closed with the common door
   function), the main routine picks the set by **story flags** (which cup the story has reached) and starts the
   loop with a counter of 4.
3. Each round: heal the party (`std216_partyStatusUpdate_8ae(0)`, skipped during a Battle Mode challenge run, variable 0x8AE), start the
   battle `fieldStartEncount(set[counter - 1], 1, 1)` (**verified**: the counter starts at 4 and drops by 1 after each
   win, so the boss in slot 0 comes last), and **stop the run on a loss** (result 3: the
   function returns 3). So the three ordinary opponents come first and the **boss last**.
4. After the fourth win, story flags are set (the cup is cleared, the next becomes available) and the script returns
   2 (won).

The other entrance scripts follow the same pattern with their own tables (Phenac: four calls indexed by one
variable; Pyrite: five; The Under's upper Colosseum: four), so each Colosseum can offer several cups chosen by the
story. **No random pick** (**verified** from the decoded scripts): none of the four entrance scripts (`M1_stadium_1F`,
`M2_enter_1F`, `M4_enter_1F`, `M4_enter_B1`) makes a random call (the decoder prints random calls where scripts make
them, so their absence counts), so each set, its order and which set is used are all fixed by story progress.

## Mt. Battle (**verified** from the decoded `D2_*` area scripts unless marked)
Mt. Battle is a different kind of challenge: a **streak of trainers stationed along the mountain**, not a run in
one arena. Ten area rooms (`D2_valley_1-3`, `D2_magma_1-3`, `D2_cloud_1-4`) each carry the same machinery, with rest
areas (`D2_rest_1-9`) between them and the Pokémon Center (`D2_pc_1F`, room 75) at the base:
- each area's script holds **tables of positions and camera framings per trainer**; as the player advances, the
  main loop checks the next trainer's spot (`std170_cameraFocusOnCharacter` with that trainer's table entry) and
  starts that trainer's battle through one shared helper (function 46 in each area);
- **a win** adds 1 to the streak counter, writes it to story **variable 2673** and **heals the party**
  (`std216_partyStatusUpdate_8ae`, skipped during a Battle Mode challenge run, variable 0x8AE);
- **a loss** resets variable 2673 to 0, sets two story flags and **sends the player back to the Pokémon Center at
  the base** (room 75), with the shared fade-in: the streak starts over;
- each area offers a **quit** prompt (`std223_askYesNoThenWarp395`): during a Battle Mode run it leaves to the
  Battle Mode menu (room 0x395; pause-menu.md, variable 0x8AE);
- the battle kinds are **8** (opponents 1-99) and **9** (the 100th), with EXP but no money (battle-engine.md).
- (**verified** from `D2_pc_1F`'s script and a recorded play session): The **registration** at the
  base is a three-line menu (the script's custom-menu native): line 0 starts or resumes a challenge (the clerk then
  asks "new or resume?"), line 1 explains the rules (several pages: ten areas of ten zones, one trainer per zone, no
  losses, coupons at each area's break room), line 2 or B leaves. The first visit after the story reopens Mt. Battle
  shows a "training will resume" line once (a story flag) instead of the usual greeting.
- (**verified** from `D2_valley_1`'s script and the same session): After a win the beaten trainer
  says one line (it waits for A), then, when the player walks past the next zone's line (the same position check as
  a trainer's spot, with the zone table's exit entries), the area asks **"Advance to the next ZONE?"**. Yes moves the
  zone counter on; no offers a teleport back to the registration counter, which warns that the area's wins won't be
  recorded before it warps.
- (**verified**, the crater Colosseum's script): The story climb's 100th win gives **1,200 Poké
  Coupons** and, when two story flags say the party stayed the same, the **Earth Ribbon** for the party; no Pokémon.
- (**verified**, the Battle Mode result menu `menuCB_range_80062948.c` and `menuSaveLoadReceiveGift`):
  In **Battle Mode's** Mt. Battle, a loss ends the run only when the continue counter is at zero; otherwise the game
  offers a continue (some wins add one). Winning the 100th battle gives **Ho-Oh** when the game has been finished, all
  48 Shadow Pokémon are purified, Ho-Oh hasn't been given before, the team came from the Colosseum save (**likely**, a
  team-source check) and the memory card is available (**likely**); continues used don't matter. The run's coupons
  are added at that point.
The same rooms serve the story's climb and Battle Mode's Mt. Battle (`BATTLEMODE_BATTLEYAMA100`, new-game-flow.md),
told apart by variable 0x8AE (the game mode: 0 = story, 1 / 2 = a Battle Mode run, **likely** single / double battles; pause-menu.md). Which trainers stand in which area,
and how the 100 are distributed, is in the area tables.

**The story climb's progress values** (**verified** from the `D2_*` scripts unless marked; all are
story flags in the save's story section):
- **2629: areas cleared in this climb** (0-12): 0 for a new challenge; leaving cleared area *k* sets *k* (valley 1 → 1
  ... cloud 4 → 10); resuming from area *k* + 1 sets *k*; the crater sets 11 on arrival and 12 after its battle.
- **2673: wins inside the current area** (not the total): + 1 per win; 0 on a loss, on the teleport back, on leaving a
  cleared area, and at the base when a challenge is still marked running. An area's next trainer is simply trainer
  number 2673 of that area: there are no per-trainer "beaten" flags, and beaten trainers stay standing.
- **Trainers per area:** 10 in each of the first nine areas, **9 in cloud 4**, and the **100th is Somek** in the crater
  (trainer 317, battle 319). So trainer 95 is cloud 4's fifth (Crisom, trainer 312, battle 314), reached with 2629 =
  9 and 2673 = 4.
- **Area *k* cleared** (flags 2612, 2614 ... 2628, kept after a loss): the base's "resume from AREA" menu offers each
  cleared area's next one. **A coupon reward waiting** at rest stop *k* (the odd flags 2613 ... 2627, and 2720): 100 ×
  *k* coupons, paid by the stop's attendant.
- **3564: a challenge in progress** (set at the start and on resume, cleared at the base); **2611**: the climb began as
  a new challenge; **2876**: the party changed at a rest stop (**likely**); both feed the crater's Earth Ribbon check.
  **3552**: the climb has been cleared.
- **Saving during the climb** is possible at the base and at every rest stop (each has a PC); the story climb has no
  Ho-Oh (that is Battle Mode's).

**Battle Mode's progress variables** (**verified** from the area scripts and a recorded run; the
story climb leaves all of them at 0). The areas are valley 1-3, magma 1-3 and cloud 1-4 (rooms 0x4C, 0x4E ... 0x5E),
with a rest stop between each two (rest 1-9, 0x4D ... 0x5D) and the crater colosseum (0x5F) at the top; the battles
happen in each kind's battlefield room (valley 0xD9, magma 0xDC, cloud 0xDA, crater 0xE4), set up through the
`toolbattle_menu` room (0x397).
- **0xB59: total wins**, passed to each battle; each win sets it to the area's base (0, 10 ... 90) plus the wins in
  the area, and clearing an area sets 10, 20 ... 90 (cloud 4 and the crater: **99**) (verified from the area
  scripts).
- **0xAFC: areas cleared**, set by each area's script on leaving (valley 1 → 1 ... cloud 4 → 10, the crater → 11).
- **0xB11: rest stops reached**: rest stop *k* sets it to *k* only when it is *k* - 1, so each stop's event runs once.
- An area works out its next trainer from the wins (cloud 4: 90 and the wins, so 90 wins = its first trainer).
How a suspended run stores these is in save-data.md ("A run in progress and a suspended run").

## Battle Mode's colosseums (**verified** from the menu labels and the rule table)
Battle Mode's Colosseum Battle offers six colosseums, each tied to a level rule (`menuCBBattleKinds`, the rule type set
for each choice): **Phenac Stadium, Pyrite Colosseum, The Under Colosseum and Orre Colosseum at Lv 50**, **Tower
Colosseum and Orre Colosseum at Lv 100**. The rules are caps on the team (the built-in rule data holds a maximum level
of 50 or 100 and a level total of 300 or 600), not a change of level (**likely**, from the rule data and the check
code). Single or double battles are a separate setting.

**The course list screen** (**verified** from the menu code and a replay frame compared with an
emulator frame, unless marked). One control routine (`menuBattleModeCourseSelectCtrl`) runs every frame for the list:
- **The rows.** Each of the six rows gets its name sprite when the course is unlocked, otherwise the "?????" sprite.
  The level label at the end of a row is printed by the game itself with `sprintf` ("Lv" and the number, or "Lv???"
  when the course is locked), converted from Shift-JIS to game text and inserted into a message. Courses 3 and 5
  (the two Orre Colosseum rows) are the ones that can be locked; the others always show their level (50, 50, 50, 100).
- **The preview picture.** The panel on the right shows the selected course's picture: one of six 238 x 180 cells of
  a single 716 x 360 texture in the menu's archive (six cells: five arenas, among them a ring of stone pillars in a
  desert, and one empty cell), picked through a six-entry sprite table in which courses 3 and 5, the two Orre
  Colosseum rows, share one picture. Which cell is which course beyond Phenac (the blue dome, seen on screen) is
  **likely** from the pictures, not checked one by one. A locked course shows a "NO DATA" sprite instead.
- **"Champ!".** Each of the Single Battle and Double Battle rows has a "Champ!" label. It is always drawn, but tinted
  to 38% (0x60 of 0xFF on every channel) until that course has been cleared in that mode; once cleared it is drawn at
  full colour, the mode's name turns from grey to white, and two small sparkle sprites appear beside it, turning one
  degree per frame in opposite directions. So a dim gold "Champ!" means "not won yet" (the player's report: it
  "glows / sparkles" once completed, **verified** in play).

## Realgam Tower's Colosseum (first pass)
`D4_casino_colo` (the story's last Colosseum) runs its rounds differently (**verified** structure): its main loop
steps through story flags 1139-1143 to pick the next scene, and each round's battle goes through a helper (function
16, called from **seven** places) that starts **one of two battle ids, chosen by a story flag**: two versions of the
same opponent. Another function counts how many of **40 consecutive story flags (1262-1301)** are set (**likely**
the purified flags of Shadow Pokémon 0-39, shadow-system.md). Not read further yet.

**The Colosseum battle room (room 0x395)** (**verified** from the room table and a replay of a
recorded Realgam Tower play session unless marked). The registration counter is its own room, whose table row names three callbacks:
an **init** run on entering (`menuColosseumBattleInit`), a **main** run every frame, and an **exit** run on leaving.
Init asserts that the room's work area is not already allocated, allocates about 68 KB (0x10F60 bytes) from the game
heap, locks it and keeps the pointer in one global; exit frees it and clears the pointer, so the room can't be entered
twice without leaving. Before the first battle the main callback opens the **party check**: every party member goes
through the rules check (six per-Pokémon checks: legal stats and effort values, not a Shadow Pokémon, not an egg, a
valid held item, legal moves, a real species; then the whole-party rules), and each slot's icon shows empty, error or
OK. The window first collects the party's Pokémon into a small seven-slot list (the last slot doubles as the read
position of the routine that hands the list to the portrait loader) and waits until each portrait is loaded.

## Battle Mode's versus screen (`menu/menuCB_BattleStart.c`; **verified** code, use **likely**)
Battle Mode (not the story) shows its teams on a **versus / team-entry screen** built on Battle Mode's versus setup
(`toolentryTaisen*`: *taisen* = versus): up to **4 players x 6 Pokémon**, each with its portrait (loaded per slot,
`menuCBPokemonEntryLoadTex`), the opposing trainer's name and class prefix, and the trainer image
(`gMenuBattleStartTrainer`). It runs in two modes (`menuCBBattleStartInit(menu, mode)`):
- **mode 0, before a battle**: plays a **fanfare chosen by a number from the versus setup** (its +0x14 field, **likely** the round number): for setup kind 0 from
  an 8-entry table (rounds 1-3 `fanfare00`, 4-6 `fanfare01`, 7 `fanfare02`, 8 `fanfare03`), for kind 1 from a 10-entry
  table by round modulo 10 (**likely** Mt. Battle's areas of ten trainers), otherwise `fanfare00`;
- **mode 1, the team display after entry**: plays `tool_music_song` and shows each Pokémon's HP **counting down from
  its maximum to its current value** (`battleStartDrainHp`), then waits for every slide-in to settle. Setup kind 3
  (Battle Now) opens menu 0x106 instead: Battle Now's after-battle **rematch** menu. See [battle-mode.md](battle-mode.md).
For a mod: kind 0's table has 8 entries and is indexed without a bounds check, so a longer custom cup would read
past it; the table must grow with the cup.

## Rules come from the battle kinds
- **Kinds 3 / 5** (story Colosseum) and **6 / 7** (Phenac's): EXP is given, **money is pooled** and paid out at the
  end of the run, Hyper Mode and the partner's Shadow detection work as in normal story battles
  (battle-engine.md, the battle-kind table; battle-rewards.md). The boss kinds (5 / 7) mark the last round.
- The arena record gives the stage, the crowd and its music (battle-arenas.md).
- **The rule record** (**verified** field reads from the rules code, the level values from a
  replay of that Realgam Tower session): a 0x54-byte record. The game holds **three fixed presets** in a constant
  table and the **active rule in the save**; picking a preset copies it byte for byte, and the game tells "custom
  rules" from a preset by comparing the two records whole. Fields read so far (16-bit numbers unless said):
  **+0** lowest level allowed, **+2** highest level allowed (preset 2: 1 to 100), **+4** the highest total of the
  party's levels, **+6** the fewest Pokémon the party needs, **+0xC** (byte) duplicate species allowed when non-zero,
  **+0xD** (byte) duplicate held items allowed when non-zero, **+0xE to +0x13** six single-byte switches read by
  their own accessors, **+0x14** the battle's time limit in minutes (the game multiplies it by 60), **+0x16** the
  per-turn command time limit.
  The six switches now have their original names (the same accessors, same order and same
  bytes as in XD), and the battle code shows what four of them guard. They are battle clauses, each read as "is this
  allowed":
  **+0x13** Sonic Boom and Dragon Rage (`menuCBRule_EnableSkill_SonicBoom`; **verified**: checked for moves 0x31 and
  0x52); **+0x12** Destiny Bond and Perish Song when the user's side has only one Pokémon left
  (`...EnableSkill_Michidure`; **verified**: moves 0xC2 / 0xC3 with a Pokémon count of 1); **+0x11** a ban on
  "suicide" moves (`menuCBRule_ProhibitionSuicide`; **likely** Self-Destruct / Explosion, from the name, not traced);
  **+0x10** Skill Swap (`...EnableSkillSwap`; **verified**: move 0x11D); **+0x0F** putting a second Pokémon to frozen
  status (`...EnableSbToIceed`, a freeze clause; **likely**, from the name); **+0x0E** putting a second Pokémon to
  sleep (`...EnableSbToSleep`, the sleep clause; **verified**: checked when the target's side already has a sleeping
  Pokémon). The helpers that copy and compare whole records are `menuCBRule_Copy` and `menuCBRule_IsEquals`. A party passes when every Pokémon passes six checks (legal
  stats and effort values, not a Shadow Pokémon, not an egg, an allowed held item, legal moves, a real species), each
  Pokémon is inside the level range with an allowed item, and the whole party meets the totals and clauses.
  (**verified** from the code) `menuRuleGetPartyErrorMsg` runs the checks in that order (the six
  per-Pokémon checks, then minimum level, maximum level and item for each slot through
  `menuRuleCheckPokemonForMode`, then the four party-wide checks through `menuRuleCheckPartyMode`: level total,
  species clause, item clause, party size). It returns the message for the first failure, so the player sees one
  reason at a time.
  (**verified** from the code) The held-item rule uses two more fields. **+0x8** (32-bit) is the
  item mode: 0 any item, 1 no held items, 2 a per-item list. **+0x18** holds 60 one-byte ban flags, one for each entry
  of a fixed 60-item regulation list in the game's data. The rule editor's item page (menu 0xB4) shows 11 rows of
  that list at a time, marks banned items in red, and toggles a flag when the player presses a button on it (key bit 0x10; which button is not checked yet).
  The records keep the disc's byte order wherever they are, which a port has to respect (**verified**: read as
  little-endian numbers, a level floor of 1 became 256 and no Pokémon could enter Realgam Tower's Colosseum).

## Changing it
- **New cups or new opponents**: script data only. Change the battle ids the entrance script writes into its
  globals, or add a set and a story-flag condition that picks it.
- **Longer or shorter runs**: the counter (4) and the loop in the entrance script.
- **Rules** (pooled money, boss handling): the battle kind of each battle, in the battle table.
- **Entry rules** (level range, level total, party size, clauses, time limits): the rule record (above); a preset
  edit changes every challenge that uses it, the save's copy changes only the current one.

**Mod surface** :
- *Data records:* the entrance scripts and their battle-id globals; the battle table's kinds; the three 0x54-byte
  rule presets and the save's active rule record.
- *Hooks it would need:* a rule-check hook (to add a rule beyond the six per-Pokémon checks and the party totals);
  a preset-table hook for more than three presets.
- *Fixed sizes:* three presets; a 0x54-byte rule record; a party of six; four rounds per run.
- *Must stay vanilla in GameCube-accurate mode:* the presets, the checks and their order, the room's portrait
  waits (they take real time on the console).

## Open questions
- Which story flags pick each cup in each Colosseum, and what clearing a cup unlocks.
- Pyrite's fifth call (its first round is started with a different fade option).
- How the pooled money is paid out at the end of a run (battle-rewards.md).
- Realgam Tower's Colosseum: what picks each round's version, and what the 40 counted flags are.
- Mt. Battle's per-area trainer tables (who stands where).
