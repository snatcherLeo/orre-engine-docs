# Battle Mode

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely different**: XD's Battle Mode has its own menus and courses)

Status: **partial** (2026-11-11). How the title menu's Battle Mode works: the menu screens, the record it keeps in the
save, how the Colosseum courses, Mt. Battle and Battle Now pick their opponents, and what a run's wins, continues and
coupons do. **Verified** from the game's code unless marked. The game's own names for the main fields come from
assert strings left in the retail build (`m_eBattleMode`, `m_eBattleType`, `m_eColosseum`, `m_nBattleCount`,
`COLOSSEUMMATCH_MAX`, `BATTLEMODE_BATTLEYAMA100`); "yama" is Japanese for mountain (Mt. Battle).

## What it does

Battle Mode is the part of the game reached from the title menu instead of Story Mode. It offers:
- **Colosseum Battle** (Solo Battle): a run of **8 battles** in one of six courses (Phenac, Pyrite, The Under, Orre
  Colosseum Lv50, Realgam Tower, Orre Colosseum Lv100), with a team **registered** from a Story Mode save or a GBA game;
- **Mt. Battle**: the 100-trainer challenge, played in the story's Mt. Battle rooms with the registered team;
- **Battle Now**: one battle with two random teams, no save needed;
- **Gang Battle**: two to four people (players 2-4 on GBAs), single, double or multi battles;
- the **rules** (three fixed, three editable) and the **PokéCoupon exchange**.

A run (Colosseum or Mt. Battle) can be **suspended** after a win and continued later from the menu.

## Data structures

### The Battle Mode record (save section 14, 0xCC2C bytes)

All Battle Mode state lives in one save section, so it survives going into a battle room and back, and a suspended run
survives switching the console off. Offsets are from the start of the section.

| Offset | Field |
|---|---|
| +0x00 | **mode** (`m_eBattleMode`): 0 Colosseum Battle, 1 Mt. Battle, 2 Gang Battle, 3 Battle Now |
| +0x04 | **battle type** (`m_eBattleType`): 0 single, 1 double, 2 multi |
| +0x08 | **rule** 0-5 (0-2 fixed, 3-5 the player's own) |
| +0x0C | **course** (`m_eColosseum`): 0 Phenac, 1 Pyrite, 2 The Under, 3 Orre Colosseum Lv50, 4 Realgam Tower, 5 Orre Colosseum Lv100, 6 Mt. Battle |
| +0x10 | Battle Now's computer rank 0-3 (Ultimate, Hard, Normal, Easy); 4 in a Gang setup |
| +0x14 | **wins so far** (`m_nBattleCount`) |
| +0x18 | **continues**: 5 at the start of a run |
| +0x1C | a byte that is 1 only **during one battle**: it tells the battle to take both teams from this record (below); a save never stores it set |
| +0x20 | the last battle's result |
| +0x24 | **four battle entries**, 0x1660 bytes each: the teams the battle actually uses |
| +0x59A4 | **coupons earned this run** |
| +0x59A8 | **four player entries**, 0x1660 bytes each: who plays, with which team and controller; each battle's entries are built from these |
| +0xB328 | **the registered team** (one entry), its "present" byte at +0xC988 |
| +0xC98B | the **suspended run** byte |
| +0xC98C | a copy of the seven run words (+0x00 to +0x18) for the suspended run |
| +0xC9A8 | the **player memo** for Mt. Battle: a "valid" byte, two room ids, position and rotation, and the four story variables Mt. Battle uses (details: save-data.md). Battle Mode's alone, not the story's climb (save-data.md "Who owns section 14") |
| +0xC9DC | **six rules**, 0x54 bytes each |
| +0xCBD4 | seven "course open" bytes; new saves open 0, 1, 2 and 4 |
| +0xCBDB | seven pairs of "course cleared" bytes, one for single and one for double |

The rest of the section (+0xCBE9 to the end) is never read.

### An entry (0x1660 bytes)

| Offset | Field |
|---|---|
| +0x00 | u16 trainer id (into the battle trainer table) |
| +0x02 | u16 player number |
| +0x04 | u32 **where the team comes from**: 0 the GameCube save, 1 or 2 one of two GBA game families |
| +0x08 | six u32 slots: the order the player picked their Pokémon in, then the count at +0x20 |
| +0x24 | u32 **controller**: 0 the computer, 1-4 a controller port |
| +0x28 | u32 the player's position in a multi battle |
| +0x2C | **the battle team**: a whole player record (0xB18 bytes, the same layout as the story's player record) |
| +0xB44 | **the master team**: a second whole player record |

Before every battle the battle team is refreshed from the master team, so each battle of a run starts with the team
as registered (**likely**: full health every round, as the copy includes HP).

The trainer id for a player's own team depends on where it came from: the GameCube save uses trainer 1 (Wes); the
two GBA families use trainers 2/3 and 0x309/0x308, chosen by the player's sex.

### A rule (0x54 bytes)

Rules 0-2 always come from a fixed table in the program, whatever the save holds; rules 3-5 are the save's slots,
edited from the rules screen and saved to the card.
- **Rule 0**, free: one Pokémon minimum, repeated species and items allowed.
- **Rule 1**: level 50 cap, level total 300, six Pokémon, species clause and item clause.
- **Rule 2**: level 100 cap, level total 600, six Pokémon, species and item clauses. **Registering a team always
  checks it against rule 2**, and Mt. Battle uses rule 2.
- The three fixed rules store a negative time limit (**likely** "no limit").
- One item is refused under every rule (item 0xAF, **likely** the Enigma Berry); a ban list of up to 60 items applies
  per rule.
- (**verified** from the code, `menuRuleCheckPokemon` check 4) The "legal moves" test only checks
  that each of the four moves is a real entry in the move table (not the empty one). It does **not** look at what the
  species can learn, so any existing move passes registration. The stat test (check 0) recomputes the stats from the
  record and refuses any stat **above** the result, plus effort values over 255 each or 510 in all.

**How many Pokémon enter a battle:** single 3, double 4, multi 2; a Gang battle with rule 0 enters all 6.

## Flow

### The menu

The Battle Mode menu is room 0x395 (`colosseumbattle_menu`). Its main loop is a **state machine over a stack of menu
ids** (eight deep); the stack is kept outside the room, so after a battle in room 0x397 the menu comes back exactly
where it was. The title menu's two Battle Mode choices start it in Colosseum Battle (mode 0) or Battle Now (mode 3).

| Menu id | Screen | What it does |
|---|---|---|
| 0xA8 | Solo or Gang | sets the mode and goes on to 0xAC (Solo) or 0xB3 (Gang) |
| 0xAC | the main list | Colosseum Battle, Mt. Battle, Continue, Register, PokéCoupon exchange. Colosseum Battle and Mt. Battle refuse when no team is registered |
| 0xAE | the course list | six courses plus "Check rules"; each course sets its rule (courses 0-3 rule 1, courses 4-5 rule 2) |
| 0xAF | Single or Double | for Battle Now, the computer's rank comes next |
| 0xB1 | start a run | checks the registered team against the rule; **if a suspended run exists, shows it and asks again** (message 0x44C8); clears the suspended run, saves, writes the run words (wins 0, continues 5) and sets story variable 0x8AE |
| 0xAD | Continue | shows the suspended team with its record, then resumes: a Colosseum run goes straight to the next battle, a Mt. Battle run puts the player back in the saved room |
| 0xB0, 0xEE, 0xED, 0xB2 | Register | from the Story Mode save (it must exist on the card) or from a GBA; checked against rule 2, then saved. If the save fails, the section is put back as it was |
| 0xCC | Battle Now confirm | shows the team you were given; No goes back and choosing again draws new teams (**likely**) |
| 0xB3, 0xBF, 0xC2, 0xE4, 0xB6, 0xB5 | Gang | the player setup (four fixed layouts), the rule, the GBA connection check (errors name the GBA's port) and, for multi, one of three team pairings |
| 0xC0, 0xC1, 0xB4 | rules | view and edit; an edited rule is saved to the card after a confirm |
| 0xB9 | PokéCoupon exchange | opens one of two coupon shops (from a Story Mode save or from a GBA) |
| 0xD1 | to battle | changes to room 0x397 |
| 0x105 | end of a run | pays the run's coupons, back to the course list or the main list |

**What the first screens look like** (**verified** from screenshots of the USA version):
1. **On the title menu**, the Battle Mode panel is an orange card headed "BATTLE MODE" with a picture of two
   Pokémon, and two choices under it: **Colosseum Battle** and **Battle Now**.
2. **Colosseum Battle** opens menu 0xA8, titled "COLOSSEUM BATTLE", which asks **"How many will battle?"** with two
   picture cards, **Solo Battle** (the hero alone) and **Gang Battle** (a group of trainers), and a **Quit** button.
3. **Solo Battle** opens the main list (0xAC). The title bar reads "COLOSSEUM BATTLE" on the left and "SOLO BATTLE"
   on the right, and five picture cards sit in two columns: **Colosseum Battle**, **Register Pokémon**, **Mt. Battle
   VS 100**, **PokéCoupon Exchange** and **Continue**, plus a **Quit** button. A help box at the bottom describes the
   card under the cursor ("Enter a COLOSSEUM battle." for the first).
4. **Gang Battle** opens the player setup (0xB3), titled "COLOSSEUM BATTLE" with "Gang Battle" on the right. It
   shows the **four fixed layouts** as rows of controller pictures, each with a red "VS", grouped under two labels:
   - **Battle with 2:** the GameCube controller against one Game Boy Advance; or two Game Boy Advance systems against
     each other;
   - **Battle with 4:** the GameCube controller and one Game Boy Advance against two Game Boy Advance systems; or two
     against two, all on Game Boy Advance systems;
   plus a **Quit** button and the help line "Link POKéMON COLOSSEUM with a Game Boy Advance system and battle!". So
   every Gang battle needs at least one Game Boy Advance; at most one player uses the GameCube controller.

**Story Mode data wants a save made at a Pokémon Center PC** (**likely**, seen in play; the check itself is not
traced in the code): to use the Story Mode save from Battle Mode, the story game must have been saved at a Pokémon
Center's PC. Which choices ask for it is an open question below.

Every screen shares the same look: orange cards with a Poké Ball corner and a picture strip, a pointing-glove
cursor, and the blue swirl background of the title menus.

### Telling Battle Mode from the story

(**verified** from the code unless marked). Battle Mode runs on the story save's image: the title's
memory card read loads the story save, Battle Mode keeps its record in that image (section 14), and its own saves write
the whole image back to the card. Its Mt. Battle course is even walked in the story's own field rooms with the player
in control. So "a field room, free to move" doesn't say which mode is running. Two of the game's own values do:
- **The top menu's choice** (a word at 0x8047A360, `titleMenuMain`): 1 Continue, 2 and 3 the two Battle Mode entries
  (Colosseum Battle and Battle Now: `menuColosseumBattle(0)` / `(3)`, then room 0x395), 4 Options. Nothing writes it
  outside the top menu, so it keeps 2 or 3 for the whole of Battle Mode, Battle Now and Gang battles included, until
  the top menu room is set up again (`titleIntroStart` puts its default, Continue or New Game, back), which is where
  Battle Mode ends (`floorLink(0x3A1, 0)`). After Continue it holds 1, after New Game 0. While the top menu itself is up
  it follows the cursor (the menu's draw callback writes it).
- **The game mode variable 0x8AE** (`FLAG_system_gamemode`; pause-menu.md): 1 or 2 (single / double) for the length of
  a Colosseum or Mt. Battle run. Starting or resuming a run sets it; the Battle Mode menu's setup and exit clear it. The
  game itself goes by it: the run's short pause menu (0x109: POKéMON and quit, no P★DA, no bag), Mt. Battle trainers
  battling through Battle Mode (`fieldStartEncount`), no party heal (script command 216). Battle Now and Gang battles
  never set it. Battle Mode's saves during a run clear it in the image they write (`gbaConvRunLinkMode`, the Ho-Oh
  gift's save), and its menu saves run with it already clear, so a save on the card never holds it set.

A Mt. Battle course always has both. A tool or port feature that must never treat Battle Mode as the story (an
autosave, a save-anywhere menu) can read either; reading both covers each other.

### The battle room

Room 0x397 (`toolbattle_menu`) runs one battle at a time: the versus screen, choosing which Pokémon enter (with a
selection timer), the battle, then the result screen. The battle itself starts from three **template battles** in the
battle table (0x20A single, 0x20B double, 0x20C multi): Battle Mode overwrites their music, arena, battle kind,
introduction effect, trainers and controllers just before each battle. For Colosseum and Mt. Battle runs, the
player's trainer record is copied into a scratch trainer (0x333) used as the player's side.

While the byte at +0x1C is set (only for that one battle), the battle code takes **each side's team from the record's
battle entries** instead of from the trainer table, and writes the teams back afterwards.

### Opponents

**Colosseum courses and Mt. Battle use a fixed order, not random draws.** One table in the program holds 296 trainer
ids:
- courses 0-5: entry `course × 16 + type × 8 + round` (8 single and 8 double opponents per course; 96 in all);
- Mt. Battle: entry `96 + type × 100 + battle number` (100 single and 100 double).

The trainers' Pokémon come from the battle trainer records on the disc (the same records story battles use). In
**Mt. Battle**, any opponent Pokémon below the level of the player's highest-level Pokémon (up to 100) is **raised**
to it; levels are never lowered. The Colosseum courses never change the opponents' levels.

**Battle kinds:** rounds 1-7 of a course use kind 0xD, round 8 kind 0xE; Mt. Battle uses 0xF, and 0x12 for the 100th
battle; Battle Now 0xC; Gang 0x10 (battle-types and the battle table give each kind's rules).

**Arenas:** courses 0-5 use battle arenas 43, 44, 45, 48, 47 and 48 (Orre Colosseum serves both its courses);
Mt. Battle changes arena with the wins so far: 40 below 30 wins, 41 from 30 to 59, 42 from 60 to 98, and 46 at 99
wins, which is the 100th battle.

**Music:** course rounds 1-7 play track 0x1F, the final round 0x20 (Phenac, Pyrite, The Under, Realgam Tower) or
0x21 (Orre Colosseum); Mt. Battle uses 0x1A, 0x17, 0x18 and 0x3D5 at the same thresholds as the arenas. Battle Now and
Gang rotate through 0x1F, 0x20 and 0x21 with a counter that is not saved.

### Battle Now

- A table holds, for each battle type and each of the four ranks, **eight Mt. Battle trainer numbers** (rank Ultimate
  from the 90s, rank Easy from the first fifteen).
- The opponent is one of the eight, drawn with the game's shared random number generator (`GSrand() & 7`).
- **The player's team is another trainer from the same eight**, drawn until it differs, renamed with one of three
  names (also drawn at random), and played as trainer 1.
- **Every held item is removed** from both teams. Levels stay as stored.
- The arena is course 0's (arena 43). Story variable 0x8AE is not set.
- After the battle, a menu offers a rematch with the same two teams.

Because the picks use the shared generator, which teams appear depends on how many random numbers the game has drawn
since it started (rng.md).

### A run: wins, continues, suspend

- **A win** gives **one extra continue when none of the Pokémon that entered fainted**. Then the player picks:
  - next battle: wins + 1, the next round is set up from the registered team;
  - quit: wins + 1, and the run is **suspended**: the run words are copied to +0xC98C, the suspended byte set, and the
    game saved.
- **A loss** with continues left offers a continue (one is used and the same round is fought again). With no
  continues left, the run ends **without a save**, so it is lost.
- **Clearing a course** (the 8th win) marks it cleared for that battle type. **Orre Colosseum Lv50 opens** once
  Phenac, Pyrite and The Under are each cleared in both single and double; **Orre Colosseum Lv100 opens** once
  Realgam Tower is cleared in both.
- **Mt. Battle from the field:** the run sets variable 0x8AE (1 single, 2 double). While it is set, a trainer battle
  started by the field (`fieldStartEncount`) goes through Battle Mode instead: the player memo is written (room,
  position, the Mt. Battle variables), the battle runs in room 0x397, and a win returns to the field. Saving clears
  0x8AE in the copy that is written, so a save never resumes inside a run by accident; the memo is what resumes it.
- **resuming writes the save at once** with the suspended byte cleared (**verified** on the PC
  port: the card changed the moment a run was resumed, and its suspended byte read 0 afterwards), so a suspended run
  can be continued only once; quitting again after a win suspends it anew.
- (**verified** from the area scripts, e.g. `D2_cloud_4`) Each area has, per trainer, a trigger
  line in front of the trainer and two "Suspend your challenge?" lines. On entering an area in Battle Mode the script
  sets the next trainer slot from the wins (`slot = wins - 90` in cloud 4) and the "behind" checkpoint one slot back;
  that checkpoint's prompt does not advance when answered No, so it asks again for as long as the player stands on its
  line (seen on the PC port: answering No brought the question back at once). Whether a normal resume on the console
  can land on that line is open; a save edit that resumes a run should place the player clear of those lines.

### Coupons

The coupons a run earns start from a factor of 1.0, multiplied by 0.85 for each party member found in a species list
(so strong species lower the reward: two such members give 1.0 × 0.85 × 0.85), times a factor per course or, for Mt. Battle, per ten wins (11 factors). A course pays once, on the 8th win;
Mt. Battle adds to the run's total every 10th win. A GameCube team's coupons are added to the Story Mode save after
a confirm; a GBA team's are sent back to the GBA (after checking its trainer id and name), capped at 9,999,999.
Winning the 100th Mt. Battle battle for the first time leads to the Ho-Oh gift check (new-game-flow.md, menu-rooms.md).

## Disc data

- **On the disc:** the battle trainer records and their Pokémon (the common archive), the three template battles in
  the battle table, the arenas, the coupon species list, the menu pictures (`toolbattle_menu.fsys`), the message text.
- **In the program itself:** the 296-entry opponent order, the course list and its arenas, the music choices, the
  Battle Now trainer table and names, the three fixed rules, the item ban list, the coupon factors, the round count,
  the starting continues.

## Changing it

- **New opponents for a course or Mt. Battle:** change the trainer ids in the opponent table, or the trainer records
  they point to. The order is fixed by formula, so a course always has exactly 8 single and 8 double opponents.
- **A new course** needs more than data: the course count (7 including Mt. Battle) sizes several tables (arenas,
  open and cleared bytes, coupon factors, battle-kind choice), and the opponent formula has no room after Mt. Battle.
- **Round count:** 8 is asserted, and the introduction effects and fanfares are 8-entry tables read without bounds
  checks; the final-round battle kind is chosen by round number.
- **Mt. Battle length:** 100 battles, the 30 / 60 / 99 thresholds, a 10-entry fanfare table and the 11-entry coupon
  table are all fixed.
- **Battle Now:** 2 types × 4 ranks × 8 trainers, three names, three music tracks.
- **Rules:** editing rules 3-5 is a save edit; rules 0-2 are in the program.
- **Continues:** 5 at the start, fixed.

**Mod surface:**
- *Data records read:* the opponent table (296 trainer ids), the battle trainer records, the template battles
  0x20A-0x20C, the arenas, the three fixed rules and the ban list, the Battle Now table, the coupon list and factors.
- *Hooks a mod system would need:* "choose this round's opponent" (replacing the formula), "build this battle's
  record" (music, arena, kind), "rules for this course", and "end of run" (rewards), so new courses can be data.
- *Fixed sizes:* 4 entries, 6 Pokémon per team, 8 rounds, 7 courses, 100 Mt. Battle battles, 6 rules of 0x54 bytes, a
  60-item ban list, 5 starting continues, an 8-deep menu stack.
- *Must stay vanilla in GameCube-accurate mode:* the opponent order and levels, Battle Now's draws from the shared
  generator, the rules and clauses, the continue and coupon arithmetic, and the save layout of section 14 (a save
  must load on a real GameCube).
- *Telling it from the story:* the top menu's choice (2 / 3) and variable 0x8AE ("Telling Battle Mode from the story"
  above); a feature that saves outside the game's own save points must check them, because Battle Mode runs on the
  story save's image.

## Efficiency

Battle Mode keeps almost nothing in memory of its own: the menu borrows one 0x10F60-byte room heap (a spare entry,
the GBA teams being read, the rule being edited and a backup of section 14 for a failed save), and everything that
must last lives in the save section, which is already in memory. Battles reuse three template records and one scratch
trainer instead of reserving a battle per course and round. Storing two full player records per entry (battle and
master) costs about 11 KB per entry but makes "start every round fresh" a single copy.

## Region notes

None found: the course list, opponent table and rules are the same code paths in all regions as far as read; the
names come from the message table.

## Open questions

- The text of messages 0x44C8 (the suspended-run prompt), 0x44C2, 0x3C10 (the extra continue), 0x3C13 (offer a
  continue) and menus 0xEC, 0xD0 and 0x106.
- Which GBA games the two GBA families are.
- What the selection timer's value means in seconds, and what the negative time limits in the fixed rules do.
- What the coupon species list contains.
- Whether Mt. Battle's "cleared" pair is ever set.
- Whether the GBA connection mode switch (which decides between two Gang team-loading paths) is ever on in the retail
  game.
- Menu id 0xB8 has a case in the menu loop but no way in was found (**likely** left over).
- Where the "saved at a Pokémon Center PC" requirement is checked (the save header's return point, or a flag set by
  the PC's save), which Battle Mode choices ask for it, and what message the game shows when it is missing.
