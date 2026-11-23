# The Shadow Pokémon system

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: XD keeps the Shadow loop and adds to it; not checked)

Status: **partial** (2026-10-17). The whole Shadow Pokémon loop in one place: what makes a Pokémon
Shadow, the heart gauge and everything that lowers it, Hyper Mode, how the hidden moves unlock, and every way to
purify. Battle details are on battle-shadow.md; this page links them up and adds the field side. **Verified** from
the decomp's C, the decoded US scripts and the US data tables unless marked.

## What it does, and its flow (the loop)
1. **Snag** a Shadow Pokémon from a trainer (battle-snag.md). The game records it with the Shadow record's
   "snagged" story flag (+0x12, flags 2126 + n).
2. **Open its heart**: lower its heart gauge through battles, Calls, walking, the day care and scents (below). As the
   gauge falls, its **hidden moves appear** and other limits lift.
3. **Purify** it at the Relic Stone once the gauge is empty (or with Celebi's help, below). The record's "purified"
   flag (+0x14, flags 1262 + n) is set; banked EXP, friendship and evolution are applied.
4. All 48 purified (and snagged) changes the ending's Ho-Oh flight on later endings and enables the Ho-Oh gift
   (unused-content.md, menu-rooms.md "Gift Pokémon").

## Data structures
- **Shadow table** (`common_rel`, `BattleRangeDef`, 0x38 bytes, ids 1-0x60): per Shadow Pokémon its species, the
  trainers who use it, its **catch rate**, its **starting gauge** and its story flags (snagged, purified, and others).
  The story uses **48 of the 96 slots** (pokemon-data.md); the rest are placeholders (unused-content.md).
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
enough (battle-shadow.md "Locked moves"). An **enemy** trainer's Shadow Pokémon gets only the slot-1 swap: it fights
with Shadow Rush plus its real moves 2-4, at any gauge level (**verified** from the byte-matched
move getter; it matches play experience, for example Cipher Admin Dakim's Shadow Entei using Shadow Rush and hurting
itself with the recoil, which makes it easy to knock out by accident before the snag; not yet checked in a battle
recording).

## Hyper Mode (battle-shadow.md, battle-turn-flow.md 3d)
- Entered only after using **Shadow Rush**, by a chance from the nature and the gauge level.
- While in it: Shadow Rush crits 90% of the time; the Pokémon may misbehave (seven kinds, from the nature: ignoring
  orders, attacking its trainer, ...); items are refused; it may calm down on its own (1 in 256 per turn).
- **Call** ends it (always, in the US data) and lowers the gauge. A battle kind can switch Hyper Mode off entirely.
- **It outlasts the battle** (**verified** from the code): Hyper Mode (condition 0x3E) is one of
  the conditions kept in the Pokémon's own record, in a 16-byte persistent-condition slot at record +0xE8 (the condition
  id in its first two bytes; condition kinds 0x7C and 0xC8 address it, `pokemonBiosGetDarkJoutaiPtr`). The battle
  copies such a condition between the battle Pokémon and its record (`fightOutPokemonCopyPokemonJoutai`), so a Pokémon
  still in Hyper Mode when the battle ends stays in it, in the party, in the PC box and in the save. Outside battle
  three things end it: **using a scent on it** (`std163_useCologne`: after the scent's message the game clears
  condition 0x3E and shows message 0x4277, then applies the scent's gauge drop as usual); **leaving it at the day care**
  (the deposit clears 0x3E before copying the record into the day-care slot); and a **blackout** (field-steps.md). The portrait
  cache picks a different Shadow face variant while it is set (pokemon-data.md, "Models and portraits"), and the
  battle start reads it from the record (likely: a Pokémon still in Hyper Mode starts the next battle in it).

## Purification
- **The Relic Stone** (Agate Village, `M3_shrine_1F`, examining the stone, function 9): once the story allows it (flag
  1067) and **at least one party Pokémon has an empty gauge** (`scriptGetDarkPointZeroPokemonNum`), the ceremony
  starts (`preReliveSetParameter(0, 0)`); otherwise the stone only shows a short message. In `preReliveMain`
  (`pokemon_relive.c`) the stone path opens a party menu (kind 7) for the player to pick the Pokémon, then warps to
  menu room 0x385 (`reliveCeremonyAll`). The ceremony marks the record purified, applies the banked EXP (levels and
  moves), sets friendship to current + banked + 70 and runs an evolution check (battle-shadow.md "Purification").
  (**verified** from the code and a PC trace): In room 0x385, `reliveMain` first loads the
  ceremony's effect sequence for the chosen Pokémon (`loadSequence`: the Pokémon's own battle model plus two
  move-effect entries from a small table of {group, index} pairs) and plays it; only if that load
  succeeds does it call `reliveCeremonyAll`. A failed load skips the ceremony silently: the room fades out and the
  Pokémon stays a Shadow Pokémon.
  (**verified** from the code): Every step of the ceremony that leaves the scene
  (the forget-a-move screen of a level-up move, menu room 0x39D; the evolution scene, menu room 0x386; the nickname
  screen) is followed by the same small replay: the Relic Stone room comes back, the Pokémon's **"look" sequence**
  (one more {group, index} pair, a short pose on its battle model) is loaded and played with a fade-in, and the
  ceremony waits for it before its next message. On a recorded purification that replay takes about 120 frames
  after the forget-a-move screen closes (**verified** from the console's fade and window logs). If the load fails
  the ceremony goes on without the pose, so the purification still completes, only faster.
- **Flag 1067** ("the Relic Stone works") is set in **Eagun's house** (`M3_houseC_1F`) when the player receives the
  **SMALL TABLET** (key item 502, whose description mentions the Relic Stone's inscriptions) and Eagun says the Relic must affect Shadow
  Pokémon.
- **The Time Flute (Celebi)**: key item 537, which summons Celebi. **Flag 1074** is set when Vander, met in
  `D2_valley_1`, has heard what happened and shows the flute; in the Pokémon Center (`D2_pc_1F`, function 20, while
  1074 is set and **1075** clear) he gives it (`scriptGetItem(537, 1)`) and sets 1075. The Relic Stone registers the
  flute as usable there; using it on a party Pokémon calls `preReliveSetParameter(1, slot)` and sets **flag 3588**
  ("Celebi was summoned": Eagun later mentions it). In `preReliveMain` the flute path
  (selection mode 1) goes **straight to the ceremony for that slot**, in its Celebi version (target mode 1), with
  **no gauge check** (**verified** for this function).
  (**verified** from the code and the decoded room script): **how the flute is "usable there".**
  Its use function (`itemFieldUsePlayTimeFlute`) works only while item 537 is on a short "usable now" list, otherwise
  "This item can't be used now" (0x426A). Only one script command fills that list; in the forest it is the room
  script's function 11, a **kind-4 event on the stone's examine plane**. Kind-4 events run only **when the pause menu
  opens**: the game looks a short way ahead of the hero's facing and fires kind 4 for each examine plane it finds. So
  the flute works when **the hero faces the stone, close to it, as the menu opens**: not a room, flag or area check.
  The list is cleared again at each lookup. The flute only accepts a Shadow Pokémon whose heart is **not yet open**
  (gauge above 0; otherwise 0x4261), so Celebi is for a heart that isn't open yet; the ceremony room then sets the
  gauge to 0 itself (`reliveMain`, mode 1). **It is single-use:** the bag removes it as soon as a Pokémon is picked,
  before the ceremony runs (a ceremony whose effect fails to load still costs the flute). The pause menu then queues
  the room script's function 10, which calls `preReliveSetParameter(1, slot)` and sets flag 3588.
- **The stone's event can be started from elsewhere** (**verified** in a PC build and from the
  code): a script event word is the script's id in the high half and the function number in the low half (the VM
  splits it so when it starts a script), so the room script's function 9 (the stone's examine event) is **0x01000009**,
  as the flute's function 10 is 0x0100000A. Queued from the pause menu as the hero step's next event (the way an item
  queues its event), function 9 runs exactly as at the stone: its flag check, its open-heart count, then the
  ceremony's start, which picks the Pokémon and warps to room 0x385. Wes need not stand near the stone; the ceremony
  stages itself and the forest comes back with him where he stood. The flute's "usable now" list is cleared at the end
  of **every** pause menu (the lookup that clears it runs as the menu closes, whatever was chosen), so an entry added
  while the menu opens lasts exactly that one menu. And an entry added there makes the flute work anywhere it is added:
  used at the forest's entrance on a closed heart, it ran the Celebi ceremony and purified the Pokémon (its purified flag set: 1300
  for Shadow id 39, so the flag is 1261 + the Shadow id).
- **The rooms** (**verified** from the room list): the Relic Forest is **one room, 0x27**
  (`M3_shrine_1F`, named "RELIC FOREST" in the game's room list): the forest and the stone's clearing together, reached
  through the caves 0x25 ("CAVE TO RELIC") and 0x26 ("SECRET CAVE"). The ceremony runs in its own menu room, **0x385**
  (`M3_shrine_1F_rl`), pushed over the forest, so the hero is back where he stood afterwards. The stone's sparkle (the
  room's main loop) shows while flag 1067 is set, an open heart is in the party and the hero is inside a box around the
  stone.
- **The heart-open test** (**verified**): "Shadow, and gauge exactly 0", the same in three places
  (the stone's count `scriptGetDarkPointZeroPokemonNum`, the stone's party picker, kind 7, which answers a non-Shadow
  Pokémon with 0x44DD and a heart not yet open with 0x44DE, and, reversed, the flute's filter).
- **What the ceremony does, in order** (**verified**, `reliveCeremonyAll`): the purified flag; Hyper
  Mode cleared; the gauge set to -1 (no longer Shadow); the real first move's message; the banked EXP (levels and new
  moves); friendship = current + banked + 70; an evolution check and scene; the **National Ribbon**; the nickname
  question. The nature is untouched.
- **The mode table** (`preReliveMain`, **verified**): it counts party Shadow Pokémon not yet purified and those with an
  empty gauge, and whether the flute is owned. Examining the stone only does something when an empty-gauge Pokémon is
  in the party (with or without the flute); the flute path ignores the counts.
- **The Japanese extra disc's Celebi** (bonus-discs.md (not yet published)) is a separate gift, not a purification.

## Missed Shadow Pokémon and second chances (**verified** from the code and room scripts)
The game keeps no "missed" state. Each Shadow record has two flags that matter here:
- **Met** (record +0x16): set when an enemy sends the Shadow Pokémon out (`fightOutPokemonSetOnDarkPokemonFlag(…, 0)`,
  at battle start and on a switch-in). From then on every later copy of it is built with the **same nature,
  personality and IVs** as the first one met (saved in the Shadow save data at that first battle).
- **Snagged** (record +0x12, flags 2126 + n): set only when it is caught (`pokemonSetOnDarkPokemonFlag(…, 1)`).

Knocking a Shadow Pokémon out sets nothing beyond "met". A second chance is a **room script** that checks the snagged
flag and starts a battle variant with or without the Shadow Pokémon (the Mystery Troop's starters are the clearest
example: story-route.md (not yet published), part 1, "Where the three Mystery Troop starters come back"). So a fainted Shadow Pokémon
and one never fought differ only in "met", and come back wherever a script offers a variant that still carries it.

## The Shadow "state", and mails that set it 
Each Shadow record has two more story flags (**verified** from the record layout and the code):
- **+0x1C, the state**: a **6-bit** story flag per Shadow Pokémon (flags 2356 + n in the US table, 48 consecutive
  fields), read by `darkPokemonGetFlag1C` and written by the script command `std243_shadowPokemonSetState(id, value)`.
  Six of the 48 state fields start exactly at bit 0 of a storage word (one every 32 flags); the
  flag reader handles those correctly only because the GameCube's processor turns a shift by 32 into 0 (flags.md (not yet published),
  "Data structures"), so a reimplementation must special-case it (**verified**: a recorded mail raised such a state
  from 4 to 5).
  The same command also sets the record's **+0x0C** flag and, the first time, gives the Shadow Pokémon a running
  number in the Shadow save data (the order in which the player learned about it; **likely**).
- **+0x0E**: set by the mail code below.

**Mails can carry a Shadow group** (**verified**, code): a mail record names a group in a Shadow-group table (its
start at record +0x02, entries of `{u8 value, …, u16 Shadow id}` until a zero id). When the player opens the mail
in the P★DA (`fn_801D228C`), the game sets each listed Shadow's +0x0E flag and raises its state to the entry's value
(it never lowers it). After the ending, a P★DA visit in which a mail was read set the state of every Shadow Pokémon
not snagged yet to 1 at once (**verified** in a Dolphin recording: 34 states set during that visit); that the mail
did it is **likely** (the only code path found that sets many states together). 

## Disc data
- The **Shadow table** and the gauge-drop and nature-rate tables are in `common_rel` (pokemon-data.md,
  game-data-tables.md (not yet published)); the Relic Stone, the Agate day care and the scents' shops are room scripts and shop tables.
- The per-Pokémon Shadow state lives in the **save** (section 15), not on the disc.

## Changing it (for modders)
- **Data-only**: the starting gauge and catch rate per Shadow Pokémon; the drop table (how much each event opens the
  heart); the nature rates; scent strengths and prices; the 48 story Shadow Pokémon and their trainers.
- **Small code hooks**: one function lowers the gauge for every event (a "faster purification" option scales it
  there); the move getter decides when moves show (thresholds 80 / 40 / 20%); the Relic Stone condition is one script
  check.
- **New Shadow Pokémon**: 48 free slots exist in the Shadow table, each with its own story flags; a new one needs a
  record, a trainer-Pokémon entry pointing at it and a battle.

### Mod surface
- **Data it reads:** the Shadow table (per Shadow Pokémon: species, owners, catch rate, starting gauge, snagged /
  purified flags), the gauge-drop table, nature rates, scent strengths, the Ho-Oh gift's list of the 48 story Shadow
  Pokémon.
- **Hooks it would need:** the gauge-lowering function (one place for every event: a "faster purification" scale), the
  move getter's unlock thresholds (80 / 40 / 20%), the Relic Stone condition, the "all purified" check.
- **Fixed-size limits:** 96 Shadow slots (48 used); the gift's fixed list of 48; the save holds 96 Shadow records, so a
  port that adds more must store the extra ones outside the original save format.
- **Must stay vanilla in GameCube-accurate mode:** gauge drops, unlock thresholds, Hyper Mode, Rui's detection at
  battle start, the shiny-lock reroll.

## Efficiency

Not applicable in any special way: the gauge is a few integers per Pokémon, updated on events (a battle turn, a
step, a scent), never every frame.

## Open questions
- The drop table's and the scents' actual numbers (data; read from `common_rel` when a mod tool needs them).
