# The Shadow Pokémon system

Status: **partial** (2026-10-17). The whole Shadow Pokémon loop in one place: what makes a Pokémon
Shadow, the heart gauge and everything that lowers it, Hyper Mode, how the hidden moves unlock, and every way to
purify. Battle details are on battle-shadow.md; this page links them up and adds the field side. **Verified** from
the decomp's C, the decoded US scripts and the US data tables unless marked.

## The loop
1. **Snag** a Shadow Pokémon from a trainer (battle-snag.md). The game records it with the Shadow record's
   "snagged" story flag (+0x12, flags 2126 + n).
2. **Open its heart**: lower its heart gauge through battles, Calls, walking, the day care and scents (below). As the
   gauge falls, its **hidden moves appear** and other limits lift.
3. **Purify** it at the Relic Stone once the gauge is empty (or with Celebi's help, below). The record's "purified"
   flag (+0x14, flags 1262 + n) is set; banked EXP, friendship and evolution are applied.
4. All 48 purified (and snagged) changes the ending's Ho-Oh flight on later endings and enables the Ho-Oh gift
   (unused-content.md, menu-rooms.md "Gift Pokémon").

## Data
- **Shadow table** (`common_rel`, `BattleRangeDef`, 0x38 bytes, ids 1-0x60): per Shadow Pokémon its species, the
  trainers who use it, its **catch rate**, its **starting gauge** and its story flags (snagged, purified, and others).
  The story uses **48 of the 96 slots** (pokemon-data.md (not yet published)); the rest are placeholders (unused-content.md).
- **Per Pokémon** (status ids, battle-shadow.md "Data structures"): Shadow flag (0xC2), Shadow id (0xC3), starting
  gauge (0xC4), current gauge (0xC5, in hundredths), banked EXP (0xC6), banked friendship (0xC7); the save keeps one
  0x18-byte row per Shadow id (save-data.md section 15).
- **Gauge level** (0-7): the gauge as 20% bands of the start value; 6 = empty (ready), 7 = not Shadow. The five bars
  on screen are those bands (**likely**).

## Everything that lowers the gauge (`pokemonAddDpFormPokemonDpFilterId`; battle-shadow.md "Flow")
Each event has a base amount in a **drop table** (data), scaled by the Pokémon's **nature** (a multiply / divide pair
per event: some natures open faster in battle, others while walking):

| Event | When | Notes |
|---|---|---|
| Sent into battle | once per battle, the player's own Shadow Pokémon | at battle start (`fightCheckRuiShadowSense`) |
| Walking | every **256 steps** with it in the party | field-steps.md (the step callbacks) |
| A successful Call | when Call snaps it out of Hyper Mode | battle-turn-flow.md 3c |
| Day care | counted like walking while it is left at the day care (the day care also gives 1 EXP per step) | field-steps.md |
| **Scents** | using **Joy / Excite / Vivid Scent** on it (kept in the **Cologne Case**; key items 543-545, sold for 600 / 800 / 1200); the amount is multiplied by the scent's own strength value | this page; the drop table's "stroking" entry |

There is no gauge drop for gaining levels or for items other than scents (**likely**, from the event list).

## What the gauge level unlocks

| Gauge level | Effect |
|---|---|
| any | slot 1 always shows **Shadow Rush** (its real move stays hidden until purification) |
| below 80% | **move 2** appears |
| below 60% (level 3) | **EXP is banked** instead of lost; **friendship** starts counting (separately); its **nature** is shown |
| below 40% | **move 3** appears |
| below 20% | **move 4** appears |
| empty | ready for the Relic Stone |

The moves were never locked in the data: the move reader just shows Shadow Rush and "?" until the gauge is low
enough (battle-shadow.md "Locked moves").

## Hyper Mode (battle-shadow.md, battle-turn-flow.md 3d)
- Entered only after using **Shadow Rush**, by a chance from the nature and the gauge level.
- While in it: Shadow Rush crits 90% of the time; the Pokémon may misbehave (seven kinds, from the nature: ignoring
  orders, attacking its trainer, ...); items are refused; it may calm down on its own (1 in 256 per turn).
- **Call** ends it (always, in the US data) and lowers the gauge. A battle kind can switch Hyper Mode off entirely.

## Purification
- **The Relic Stone** (Agate Village, `M3_shrine_1F`, examining the stone, function 9): once the story allows it (flag
  1067) and **at least one party Pokémon has an empty gauge** (`scriptGetDarkPointZeroPokemonNum`), the ceremony
  starts (`preReliveSetParameter(0, 0)`); otherwise the stone only shows a short message. In `preReliveMain`
  (`pokemon_relive.c`) the stone path opens a party menu (kind 7) for the player to pick the Pokémon, then warps to
  menu room 0x385 (`reliveCeremonyAll`). The ceremony marks the record purified, applies the banked EXP (levels and
  moves), sets friendship to current + banked + 70 and runs an evolution check (battle-shadow.md "Purification").
- **Flag 1067** ("the Relic Stone works") is set in **Eagun's house** (`M3_houseC_1F`) when the player receives the
  **SMALL TABLET** (key item 502, whose description mentions the Relic Stone's inscriptions) and Eagun says the Relic must affect Shadow
  Pokémon.
- **The Time Flute (Celebi)**: key item 537, which summons Celebi. **Flag 1074** is set when Vander, met in
  `D2_valley_1`, has heard what happened and shows the flute; in the Pokémon Center (`D2_pc_1F`, function 20, while
  1074 is set and **1075** clear) he gives it (`scriptGetItem(537, 1)`) and sets 1075. The Relic Stone registers the
  flute as usable there; using it on a party Pokémon calls `preReliveSetParameter(1, slot)` and sets **flag 3588**
  ("Celebi was summoned": Eagun later mentions it). In `preReliveMain` the flute path
  (selection mode 1) goes **straight to the ceremony for that slot**, in its Celebi version (target mode 1), with
  **no gauge check** (**verified** for this function). Whether the item menu only offers Shadow Pokémon, and whether
  the flute can be used more than once, wasn't traced (**likely** once, from flag 3588 and public knowledge).
- **The mode table** (`preReliveMain`, **verified**): it counts party Shadow Pokémon not yet purified and those with an
  empty gauge, and whether the flute is owned. Examining the stone only does something when an empty-gauge Pokémon is
  in the party (with or without the flute); the flute path ignores the counts.
- **The Japanese extra disc's Celebi** (bonus-discs.md (not yet published)) is a separate gift, not a purification.

## For modders
- **Data-only**: the starting gauge and catch rate per Shadow Pokémon; the drop table (how much each event opens the
  heart); the nature rates; scent strengths and prices; the 48 story Shadow Pokémon and their trainers.
- **Small code hooks**: one function lowers the gauge for every event (a "faster purification" option scales it
  there); the move getter decides when moves show (thresholds 80 / 40 / 20%); the Relic Stone condition is one script
  check.
- **New Shadow Pokémon**: 48 free slots exist in the Shadow table, each with its own story flags; a new one needs a
  record, a trainer-Pokémon entry pointing at it and a battle.

## Open questions
- The Time Flute's item-menu filter and single use.
- The drop table's and the scents' actual numbers (data; read from `common_rel` when a mod tool needs them).
