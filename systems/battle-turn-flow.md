# Battle engine: turn order, end of turn, weather, AI scoring

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). Companion to [battle-engine.md](battle-engine.md) (battle setup, battle
kinds, move sequences, critical hits, damage formula). This page covers **who acts first**, **what happens at the end
of a turn**, **how weather is stored and counted down**, and the **AI's scoring core**. Claims are labelled as in the README. None of this has been watched running in Dolphin yet.

Japanese terms used in the code: *subayasa* / *nimbleness* = Speed, *pri* = priority, *tenkou* = weather, *joutai* =
status / condition, *kaisuu* = count (turns), *irekae* = switch, *nigeru* = run, *tokusei* = ability, *soubi* = held
item, *banme* = index ("the n-th").

## What it does

Each turn every trainer picks an action. The engine then sorts the Pokémon on the field into one list, runs their
actions in that order, and finishes the turn with a fixed sequence of timers and damage-over-time effects. The rules
are Generation 3's (Ruby / Sapphire; **verified** for the parts this page shows, **likely** overall): the same tie-breaks, the same Quick Claw roll, the same speed modifiers.
Colosseum adds one action kind (**Call**) and slots it into the ordering.

## Data structures

### Action kinds (`lbl_80375BB8`, 20 rows of 12 bytes; **verified** from the executable's data)
Each row: a signed priority byte, padding, a *flow* function (does the work) and a *disp* function (draws it). The
count (20) is in `lbl_80478D48`. Rows with priority -1 (0xFF) are phases of the battle, not choices a trainer makes;
the five rows a trainer can choose carry real priorities:

| Kind | Action | Priority (lower acts first) |
|---|---|---|
| 8 | Run (`FightNigeru`) | 0 |
| 18 | Use an item (`FightTrainerUseItem`) | 1 |
| 10 | **Call** (`FightTrainerCall`, Colosseum's Hyper Mode cure) | 2 |
| 9 | Switch (`FightOutPokemonIrekae`) | 3 |
| 19 | Use a move (`FightOutPokemonOutWaza`) | 4 |

The other kinds are the phases named in battle-engine.md (0 null, 1 `Kaijou`, 2 `KaisiPre`, 3 / 4 trainers and
Pokémon enter, 5 `KaisiPost`, 6 `TenkouInit`, 7 select actions, 11 run all actions, 12 Focus Punch's pre-turn
message, 13 `OneTurnPost`, 14-16 the end of the battle, 17 `Heijou`).

So the Gen 3 rule "running, items and switching happen before moves" is data here, and Call sits between items and
switching: a Call always happens before the opponent's switch and before every move that turn (**verified** from the
sort below plus this table).

### The battle's state: every `gFightFloor` field (`fightFloorGetStatus(floor, 0, id, arg)`; **verified** from the getter switch)
The battle-floor object (`gFightFloor`, `lbl_8046DD90`, 0xA4E8 bytes) is read and written through one status id per
field. Ids 0x01-0x07 read the arena record; 0x0E-0x34 are derived through the encounter, battle type and battle kind;
the rest are the battle's own state.

| Id | Field | Id | Field |
|---|---|---|---|
| 0x01-0x07 | arena: name, floor (room) id, side data, terrain, intro WZX, music, ambient sound | 0x35 | side pointer [0 / 1] |
| 0x09 | the field's condition list | 0x36 | **acting Pokémon** (attacker) |
| 0x0C | turn counter | 0x37-0x41 | the acting Pokémon's per-move fields (its status 0xDA-0xE4) |
| 0x0D | encounter (battle) id | 0x42 | **receiving Pokémon** (target) |
| 0x0E / 0x0F / 0x10 | arena id / its terrain / its intro WZX (via the encounter) | 0x44 / 0x45 / 0x46 | Pokémon running / switching out / fainting |
| 0x11 / 0x12 | music / ambient sound (encounter's choice) | 0x47 / 0x48 / 0x49 | Pokémon a **secondary effect** applies to / whose **ability** acts / whose **item** acts |
| 0x14 | battle type id | 0x4A | encounter's arena id |
| 0x15-0x19 | battle type: name, trainers per side, Pokémon entered per trainer, Pokémon out per trainer, maximum out per side | 0x4B-0x4F | **appointed** (message-subject) Pokémon, trainer, move, side, item |
| 0x1A / 0x1B | battle kind id / its name | 0x50-0x53 | message ids: appointed, attack, critical, move-effect |
| 0x1C-0x34 | the battle kind's rule flags (battle-engine.md, "The full picture": names shifted from 0x27 on) | 0x54-0x57 | appointed species, move, item, ability ids |
| | | 0x58 | Pokémon entered count |
| | | 0x59 / 0x5A | the 8 field slots: one / the whole array |
| | | 0x5B | turn random |
| | | 0x5C | battle result |
| | | 0x5D | find a field Pokémon |

The "appointed" fields (*shitei*, the subject of the next message) are what message placeholders read; the target
ids 0x11-0x1D (battle-targets.md) are these role fields.

### Per-Pokémon status ids the battle code uses (`pokemonGetStatus(p, 0, id, index)`)
The full selector list is in pokemon-data.md (not yet published); these are the meanings learned from the battle code
(**verified** where the use pins it down, **likely** otherwise):

| Id | Meaning | Label |
|---|---|---|
| 0x63 | Pokédex "caught" flag (Repeat Ball) | verified |
| 0x6E | species | verified |
| 0x6F | personality value (`Rnd`) | verified |
| 0x70 | data check (`Attest`, the validity checksum) | verified |
| 0x71 / 0x72 | the room it was caught in / its level then | verified |
| 0x73 | the ball it was caught in (item id; 11 = Luxury Ball, friendship.md) | verified |
| 0x74 | original trainer's sex | verified |
| 0x75 / 0x76 | original trainer's id (`CatchTrainerRnd`) / name | verified |
| 0x77 / 0x78 | nickname / the original nickname | verified |
| 0x79 / 0x7A | EXP total / level | verified |
| 0x7B | fainted (current HP is 0; computed, not stored) | verified |
| 0x7F / 0x80 | move id / PP, per move slot | verified |
| 0x83 / 0x87 | current HP / maximum HP (catch formula) | verified |
| 0x8C | Speed stat (effective Speed) | verified |
| 0x93-0x98 | IVs: HP, Attack, Defense, Sp. Atk, Sp. Def, Speed (Hidden Power's bit order, battle-types.md) | verified |
| 0x99 | friendship (0-255, friendship.md) | verified |
| 0xBF | nature index | verified |
| 0xC2 / 0xC3 | is a Shadow Pokémon / its Shadow id | verified |
| 0xC4 / 0xC5 | starting heart gauge / current gauge (hundredths) | verified |
| 0xC6 / 0xC7 | stored EXP / Shadow friendship | verified |
| 0xCE | party slot (switch choices, the AI) | likely |
| 0xCF | snagged in this battle (`CatchEntryFlag`) | verified |
| 0xD0 | gained a level this battle (`LevelUpFlag`; post-battle evolution checks only these) | verified |
| 0xD1 | the gauge already dropped this battle (entry drop, once) | verified |
| 0xD5 / 0xD6 | links between the field Pokémon and its party record | likely |
| 0xD9 | the move currently being used (its battle record: damage at status 0x2D, effect chance, ...) | verified |
| 0xDA-0xE4 | per-move working fields read through floor ids 0x37-0x41 | verified (that they exist) |
| 0xE5 | the item record being used (Bag, balls) | verified |
| 0xEA / 0xEB / 0xEC | stat stages / accuracy stage / evasion stage | verified |
| 0xEE | status-effect animation handle (cleared on waking or thawing) | likely |
| 0xF6 | stored target (Bide-style) | likely |
| 0xF9 | Truant's "loafing this turn" flag | likely |
| 0xFE | the Pokémon's chosen action | verified |
| 0x100 | ability | verified |
| 0x106 | fully paralysed this turn | likely |
| 0x108 | the last move had no effect | likely |
| 0x10B | blocked by Imprison | likely |
| 0x112 | already acted this turn (turn loop) | verified |
| 0x114 | pulled a move with Lightning Rod (message flag) | verified |
| 0x116 / 0x117 | Intimidate / Trace already done this entry | verified |
| 0x118 | in the later turns of a multi-turn move (no PP use) | guess |
| 0x11A | Focus Band triggered this hit | verified |
| 0x120 | fainting pending (end-of-turn faint handling) | verified |
| 0x121 | party slot to switch to | verified |

### Condition ids (*joutai*)
Pokémon, sides and the field share one condition-id space (`fightOutPokemon...JoutaiDataId`,
`fightSide...JoutaiDataId`, `fightFloor...JoutaiDataId`); timed conditions carry a length and a current count.
"Verified" below means the code's behaviour pins the id down; "likely" means the context and Gen 3's identical code do.

| Id | Condition | Evidence |
|---|---|---|
| 0x03 / 0x04 | poison / bad poison (Toxic) | **verified**: their definitions' name message is "poison"; 1/8 and 1/16 x counter per turn; bad poison's counter stacks to 15 |
| 0x05 | paralysis | **verified**: Speed / 4, and 1 in 4 "fully paralysed" in the move checks |
| 0x06 | burn | **verified**: name message "burn"; 1/8 per turn |
| 0x07 | freeze | **verified**: thaws 1 in 5 before a move; move effect 125 (fire moves that thaw) clears it |
| 0x08 | sleep | **verified**: counter before each move, Early Bird (ability 48) counts two at a time, Uproar wakes |
| 0x09 | confusion | **verified**: name message "confusion"; lasts 2-5 turns |
| 0x0A | infatuation (Attract) | **likely**: its place in the move checks |
| 0x0C | Bide | **verified**: its place in the move checks and the move it forces (117, Bide) |
| 0x14 | Substitute | **likely**: the damage code skips the survive-at-1-HP rules behind it |
| 0x0E | bound (Wrap, Bind, ...) | **likely**: blocks switching; end-of-turn damage step |
| 0x16 | can't escape (Mean Look, Block) | **likely**: blocks switching |
| 0x11 / 0x12 | flinch / must recharge (Hyper Beam) | **likely**: their place in the move checks |
| 0x17 | Nightmare | **likely**: 1/4 per turn, cleared on waking |
| 0x18 | Curse (Ghost) | **likely**: 1/4 per turn |
| 0x1C | Leech Seed | **likely**: 1/8 drained per turn |
| 0x1D | Lock-On / Mind Reader (stores the locked target) | **verified**: the one-hit-KO and accuracy checks read the stored target |
| 0x0B | Uproar | **likely**: its place in the end-of-turn countdowns |
| 0x0D | Thrash-style rampage (Thrash, Outrage, Petal Dance) | **likely**: countdown that ends in confusion |
| 0x24 | Charge | **verified**: doubles the next Electric move in the damage step; counted down at end of turn |
| 0x26 | Yawn | **likely**: countdown that ends in sleep |
| 0x1E | Perish Song | **likely**: count shown, faints at the end |
| 0x25 | Ingrain | **likely**: heals 1/16 per turn |
| 0x2C | Endure | **verified**: survives at 1 HP (battle-engine.md, "Damage variance") |
| 0x29 / 0x2A | Disable / Encore | **likely**: the move checks / the forced-move replacement |
| 0x30 | Taunt | **likely**: its place in the move checks |
| 0x13 | Rage | **likely**: step 0 of the end-of-move steps raises Attack when hit |
| 0x36 | Choice Band lock (the locked move) | **verified**: set by held-item effect 29 at the end of a move |
| 0x34 | Future Sight / Doom Desire | **likely**: stored move, target and countdown |
| 0x3E | **Hyper Mode** (Colosseum) | **likely**: cured by Call, entered by Shadow Rush |
| 0x48 / 0x49 / 0x4B / 0x4C | side: Reflect / Light Screen / Safeguard / Mist | **verified** for Reflect, Light Screen and Mist by their "wore off" message naming the move |
| 0x4E-0x55 | field: weather | **verified** (section 5) |

The move checks before every move (`fn_8022F2F8`) run in Gen 3's order: sleep, freeze, Truant (ability 54),
recharge, flinch, Disable, Taunt, Imprison, confusion, paralysis, attraction, Bide, thaw, then Colosseum's Hyper Mode
check (**verified** order from the code; ids as in the table).

### Condition definitions and lengths (**verified** from code and the executable's data)
Every condition id has a 20-byte **definition** (`lbl_8035BBA8`, 86 entries, count `lbl_80478B48`; accessors
`fn_80119D90`-`fn_80119F10`), and setting a condition (`fn_8011B2C0`) reads it:

| Offset | Meaning |
|---|---|
| +0x00 | object kind it belongs to, in the status system's numbering (5 Pokémon, 8 side, 7 battle floor) |
| +0x01 | slot within that object (e.g. the side conditions Reflect, Light Screen, Safeguard, Mist use slots 0, 1, 3, 4) |
| +0x02 | kind: 4 = stacking (each set adds 1 up to the limit) |
| +0x03 | stack limit (bad poison's counter: 15) |
| +0x04 | **length id** into the count table below (0 = no countdown) |
| +0x08 | u16 owner kind: 0x7C the major statuses (**likely** the ones kept on the Pokémon after battle), 0xD8 battle-only Pokémon conditions, 0xC8 Hyper Mode, 6 side, 9 field |
| +0x0A | u16 a linked condition (permanent sun 0x4F links to timed sun 0x53) |
| +0x12 | u16 name message (sleep, poison, burn, paralysis, ice, confusion for ids 8, 3 / 4, 6, 5, 7, 9) |

**Count table** (`kaisuuGetKaisuu`, `lbl_80315690`, 10 entries of {min, max}; the length is random in that range):
0 no limit, 1 = 0, 2 = 1, 3 = 2, 4 = 3, 5 = 5, 6 = 2-3, 7 = 2-5, 8 = 3-5, 9 = 3-6 turns. So, from the
definitions: **sleep 2-5, confusion 2-5, Disable 3-5, Encore 3-6, Taunt 2, Bide 2, Perish Song 3, Reflect / Light
Screen / Safeguard / Mist 5, timed weather 5**; the major statuses and Hyper Mode have no countdown. (Gen 3's Disable
lasts 2-5 turns (**likely**, not checked against Gen 3's code); if so, Colosseum's 3-5 is a difference.) Changing a length is a one-byte data edit (the definition's length id) or an edit of the count table.

## Flow

### 0. The whole battle, phase by phase (`_fightInitialize`, the battle thread `fn_801EFA08`; **verified**)
Each phase is one action of the kind shown (action-kind table above), run through the action FIFO:
1. **On the scene thread** (the floor's enter callback `_fightInitialize`): the debug menu is closed and disabled for
   the battle, the action FIFO is set up, then `Kaijou` (kind 1, the venue) and **the trainers enter**, side 1 (the
   opponents) first, then side 0 (kind 3 with target ids 5, then 4). Then the **battle thread** is created (first
   argument 0x14, likely its priority; 16 KB stack).
2. **On the battle thread:** the floor's sound starts if it has one (floor status 0x12), resources load
   (`fightActionFlowKaisiPreSubLoad`), the screen fades in, `KaisiPre` (kind 2); **the Pokémon are sent out**, again
   opponents first (kind 4 with 5, then 4); `KaisiPost` (kind 5: on-entry checks, weather init); the total battle
   timer starts if the battle uses one.
3. **The turn loop** (until a result is set): the time-out check, the command timer, choosing actions (kind 7, the
   info menu open), running them (kind 11), the end of turn (kind 13).
4. **The end:** `SyuuryouPre` (14), `Syuuryou` (15, given the result), the floor sound stops, the fade-out,
   `SyuuryouPost` (16), `Heijou` (17, back to normal), and control returns to the field (`floorSetFadeScript`).

### 1. Turn order (`fightFloorSortFightOutPokemonPtrArySub`, `fightFloorCmpfightOutPokemonNimbleness`; **verified**)
`fightActionFlowAllFightOutPokemonDoFightAction` builds the list of the (up to 8) Pokémon on the field and sorts it.
The sort is a simple exchange sort (fine for 8 entries): empty slots go to the end, then:
1. **Action priority** (table above): lower number first.
2. Same action priority: compare the two Pokémon (`Cmp...Nimbleness`):
   - **move priority** (the move record's signed byte at +4, `wazaGetStatus(0, move, 4, 0)`): higher first. Only
     compared when at least one of the two moves has a non-zero priority;
   - **effective Speed** (below): higher first;
   - **tie: a coin flip** (`GSrand & 1`).

The sort can also run with priorities off (`usePri = 0`): then only Speed counts. The battle start (`KaisiPost`, for
the on-entry ability and item checks) and the end of turn use that form, so **entry effects and end-of-turn effects
happen in Speed order** (**verified**: both callers pass 0).

### 1b. Choosing actions (`fightActionFlowAllFightTrainerSelectFightAction`; **verified** from code)
Every trainer on the field chooses in turn (`fightTrainerSelectFightAction`):
- a trainer whose record names an AI record (trainer field 2, +0x06, non-zero: section 7) is **computer-controlled**
  and goes to the AI (`fn_802342CC`);
- otherwise the trainer's hero kind picks the controls: 0 = **GameCube controller** (`fightMenuFightTrainerGcHeroOpenMenu`),
  1 = **Game Boy Advance** used as a controller (`fightMenuFightTrainerAgbHeroOpenMenu`; *agb* = GBA), as in battle
  mode's link play.

**The player's command menu** (`_fightMenuFightTrainerGcHeroOpenMenuSubMain`), one choice per Pokémon:

| Choice | What happens |
|---|---|
| 0 Fight | the move menu; with no usable move left the game says the Pokémon "has no moves left" (message 0x75FC) and uses Struggle |
| 1 Bag | only if the battle kind allows items (status 0x20); otherwise message 0x75F5 ("this item can't be used now") |
| 2 Pokémon | pick a party member, then a switch action (kind 9) is created |
| 3 | **the same button is Run or Call**: Run (action kind 8) if the battle kind allows running (status 0x22), else Call (kind 0x0A) if it allows calling (status 0x21), else nothing |

Choosing can be undone: going back from the second Pokémon returns to the first one's menu (the loop steps back a
trainer or side). A debug switch (`fn_80008174`, off at boot) makes **every** trainer, AI ones included, use the
player's menu: the developers could play both sides.

### 2. Effective Speed (`fightOutPokemonGetNowNimbleness`; **verified** from code, ids are Gen 3's)
Starting from the Pokémon's Speed stat:
1. **Swift Swim** (ability 33) doubles it in rain; **Chlorophyll** (34) doubles it in sun. Weather here is read
   *with* the Cloud Nine / Air Lock check, so either ability on the field cancels the doubling.
2. Stat stages (`fightAbicntDoKakeWaru`, the usual stage multiplier table).
3. **Badge boost**: x1.1 (`* 110 / 100`) if the battle's badge rule is on and the trainer owns the Speed badge (hero
   flag 0x11). (**likely**: which badge it is is from Gen 3's layout.)
4. **Macho Brace** (held-item effect 24): halved.
5. **Paralysis** (status 5): quartered.
6. **Quick Claw** (held-item effect 26): if the turn random is below `percent x 0xFFFF / 100` (the item's own
   percentage, 20 in Gen 3), Speed becomes the maximum value, so the holder goes first within its action priority.

Because the **turn random is one number for the whole turn** (rolled at the end of every turn in `fn_80213270` and
once at battle start in `KaisiPost`), two Quick Claw holders in the same turn either both trigger or both don't
(**verified** from code; Ruby / Sapphire do the same with their per-turn random).

### 3. Running the actions (`_fightActionFlowAllFightOutPokemonDoFightActionOneLoop`; **verified**)
Two passes over the sorted list. Pass 0 runs only **Run** actions; then the move-sequence module is re-initialised
(`fn_80211A00`: per-Pokémon sequence hooks registered, sequence flag bytes cleared); pass 1 runs every other action
in sorted order. A Pokémon whose action is missing or invalid, or who has fainted, is marked "done" (status 0x112)
and skipped. After each action of pass 1 the targets are reset, and **the turn stops as soon as a battle result is
set** (someone won, ran, or time ran out).

Each action is copied before it runs (`copy = *action; fightActionFlowFifo(&copy)`), so an action that changes the
queue mid-turn can't corrupt the one being run.

### 3b. Switching and running (**verified** from code unless marked)
- **Switch** (`fightActionFlowFightOutPokemonIrekae` -> `fn_80213158`): stores the outgoing Pokémon (floor status
  0x45) and the party slot to bring in (Pokémon status 0x121, from the action's buffer id), clears three of the
  outgoing Pokémon's conditions (0x2E, 0x15, 0x28; which ones: **guess**, not identified yet), then runs the switch
  **script** `lbl_8037889D` (a `WS_*` byte stream, 35 bytes) that does the withdraw and send-out. So switching is
  itself a battle script, like in Gen 3. Decoded (battle-engine.md, "The command set is Ruby / Sapphire's"), it starts
  exactly like Gen 3's switch script: the "come back" message, then **Pursuit**: if an opponent chose Pursuit, the
  attacker and target are swapped, Destiny Bond is checked and the Pursuit damage script is called before the
  Pokémon leaves (**verified** from the bytes).
- **Can it switch?** (`fightFloorCheckFightActionFightOutPokemonIrekaeSelect`; **verified**): no,
  if it is under condition 0x16 (Mean Look / Block: **likely**), 0x0E (a binding move: Wrap, Bind, ...: **likely**) or
  0x25 (Ingrain). No, with the holder named in the message, if an opponent has **Shadow Tag** (23); if an opponent
  has **Arena Trap** (71) and it is neither Flying nor has Levitate; if an opponent has **Magnet Pull** (42) and it is
  a Steel type. The menu and the AI both ask this function.
- **Run** (`fightActionFlowFightNigeru`): resets the runner's action counters and sets the battle result directly: 4
  if the runner is on the host side, else 5. **There is no escape roll in this function** (Gen 3's Speed-based escape
  formula is absent here); whether running is allowed is decided earlier, by the battle kind's "can run" rule and the
  menu (**verified**, the menu's fourth button). On the disc only the battle-mode kinds allow
  running (battle-engine.md, "The 19 battle kinds"): Colosseum has no wild battles, so "run" is forfeiting a
  battle-mode match, and it always works.
- **Battle result ids** (`fightFloorSetFightResultId`): 0 none yet, **1 link error**
  (`fightFloorSetTuusinErrorFightResult`, *tuusin* = communication), **2 / 3: the host side / the other side wins**,
  **4 / 5: the host side / the other side ran**, 6 and 7 combined results. Ids 2-5 are read from the run action
  (host 4, other 5) and the time-out judge below (**verified** that those callers use them so; "wins" for 2 / 3 is
  **likely**). The battle's result rule (floor status 0x23, a battle-kind flag) decides ties: with it on, results 2
  and 3 together become **7 (draw)** and 4 and 5 together 6; with it off, 3 / 5 can't be overwritten and 7 / 6 are
  turned into 3 / 5 (**verified** logic; the rule's name, likely the `Draw` flag, is not confirmed).
- **Time-out** (`fightFloorSetTimeOutAllFightResult`, when Colosseum's total battle timer runs out; **verified**): the
  side with **more Pokémon left** wins; if equal, the side with the **higher remaining HP percentage** (all its
  Pokémon's HP over their max HP); an exact tie sets both results, which the rule above turns into a draw or a loss
  for the host side. Scripts test for 2 after a story battle (new-game-flow.md, step 8).
- **Timers** (`battle_main.c`; **verified**): a per-turn command timer runs while actions are chosen when the battle
  uses one (floor status 0x34), and a total timer is checked at the top of every turn.

### 3c. Call and Hyper Mode (`fightActionFlowFightTrainerCall` -> `fn_80212D6C`)
**Verified** from code unless marked. Condition 0x3E is **Hyper Mode** (**likely**: it is what a successful Call
clears, and it's the condition behind Shadow Rush's 90% critical rule in battle-engine.md); condition 8 is **sleep**
(**verified**: the status table above; waking also clears it together with 0x17, Nightmare).
1. Like a switch, Call first clears the Pokémon's conditions 0x2E, 0x15 and 0x28.
2. **Shadow Pokémon in Hyper Mode:** success if `random % 100 < chance`, where the chance comes from the Pokémon's
   **nature** and its **heart-gauge level** (`fn_8011F634`: nature record +0x1F, seven bytes, one per gauge level;
   level 7 or above gives 0). On success: out of Hyper Mode (message 0x7707) and, if asleep, woken too. On failure
   (message 0x7708) a sleeping Pokémon still wakes with half the chance.
   **In the US disc's data every nature has 100 at every level** (all 25 natures checked), so **a Call always ends
   Hyper Mode in Colosseum**; the chance machinery is there but unused (**verified** from code and data; not yet
   watched in game).
3. **Shadow Pokémon not in Hyper Mode, or a normal Pokémon:** a sleeping Pokémon wakes up (message 0x7706, "responded
   to the call"); otherwise nothing happens (0x771D: the Shadow Pokémon didn't hear; 0x7709: it isn't listening).
   The failure message for Hyper Mode (0x7708, "ignoring orders") can't appear with the US data, since the chance is
   always 100. No accuracy boost or other effect was found in
   this function.
4. When the player's own Pokémon leaves Hyper Mode, a heart-gauge record is updated
   (`pokemonAddDpFormPokemonDpFilterId(.., 0, 2)`; meaning **guess**: counts towards purification or the gauge's
   display).
5. The trainer's call animation (`fn_801FB974`, steps 0-3) and the messages "<trainer> called out ..." (0x7704,
   0x7705) frame the result; a transformed Pokémon (Transform, "Hensin") has its copied status refreshed.

Because Call has action priority 2, it always resolves **before** the Pokémon's own move in that turn would have
(the called Pokémon gives up its own action that turn: **likely**, the action is stored on the Pokémon) and before any
switch.

### 3d. Entering and leaving Hyper Mode on its own (`fn_8022EDEC`; **verified** from code and data)
The "can this Pokémon act" checks that run before every move (`fn_8022F2F8`, a loop over cases 0-14 that stops at 15)
ask `fn_8022EDEC` about Hyper Mode. Gen 3's "attack canceller" has cases 0-13 and ends with the thaw check; Colosseum's
case 13 is that thaw check too (condition 7, cleared by move effect 125: **condition 7 = freeze**, **verified** by the
status table above), and case 14 is Colosseum's addition. All of it needs: a valid Pokémon, the battle
kind's Hyper Mode rule on (floor status 0x31, the `DarkpokemonHypermode` flag), and a Shadow Pokémon that is still
shadowed (`fn_8011FC74`).
- **Entering (mode 0):** only if not already in Hyper Mode and not asleep, and **only when the move being used is
  Shadow Rush** (move 356). Chance: `random % 100 <` the nature record's byte at +0x18 for the Pokémon's heart-gauge
  level (seven bytes, one per level, `fn_8011F6D8`). Then the Hyper Mode condition is set with message 0x771B and the
  move goes on. In the US data these chances run from 5 to 70 and natures fall into six patterns (values not listed
  here: disc data).
- **Leaving by itself (mode 2):** while in Hyper Mode, each check has a **1 in 256** chance (`GSrand % 256 == 0`)
  to calm down on its own: Hyper Mode, sleep and Nightmare are cleared, message 0x771E.
- So the only reliable way out is **Call**, which always works in Colosseum (above).
- **Mode 1** is the last step of every Pokémon's end-of-turn pass (`fn_80230568`): the "is in Hyper Mode" reminder
  (**verified**).
- **Misbehaving in Hyper Mode** (`fn_8022A6C8`): when a Shadow Pokémon in Hyper Mode (Hyper Mode
  rule on, still shadowed) is about to use a move other than Shadow Rush, the game rolls `random % 100` against the
  nature record's bytes **+0x0B to +0x11**, one chance per misbehaviour, each with its own message (0x770C-0x7714). The checks run in this order (**verified** from the code; the own-move branch picks a random usable move
  other than 355, an entry treated like "no move", and 357, the locked-move placeholder), +0x0B (own move, 0x770C), +0x0C (attack the opposing trainer, 0x770D), +0x0D (its own trainer, 0x770E),
  +0x0E (hurt itself, 0x770F), +0x0F (too worked up, 0x7710), +0x10 (held item, 0x7711 / 0x7712), +0x11 (back to its
  ball, 0x7713 / 0x7714). *Mechanics (added 2026-10-16, **verified** from the code and the scripts' bytes unless marked):*
  attacking either trainer and "too worked up" only play their animation and message and then run a 3-command script
  (end-of-move steps, end): **the turn is simply lost**, no damage; "hurt itself" jumps into the confusion self-hit
  script (0x1D bytes into it, past its opening; **likely** the same self-damage as confusion); the held-item one makes
  the Pokémon use its held item if that is item 133-147 (Gen 3's battle berries: **likely**) or has held-item effect 44
  (the held-item code runs with a forced trigger), otherwise
  it fails with one of two messages; using another move picks a random usable move and continues as if chosen.
  Confusion itself (for comparison): 50% chance to hit itself, Gen 3's.
- Other Hyper Mode rules seen: a Pokémon in Hyper Mode won't accept an item (message 0x770B); a snagged Pokémon leaves Hyper
  Mode (`fn_80213A78`, the "give caught Pokémon" command 0xF0).
- How the heart gauge drops (in and out of battle): battle-shadow.md.

### 4. End of turn (`fightActionFlowOneTurnPost` -> `fightSeqPost` -> `fn_80213270`; **verified** order)
1. Close status menus and messages; clear the eight sequence flag bytes.
2. Per-Pokémon turn checks (`_fightSeqTurnCheckSubFightOutPokemon`), reset the per-turn move conditions.
3. **Side timers and weather** (`fn_802317E4`, Pokémon re-sorted by Speed first). For each side, in this order, the
   turn count goes up and the condition ends when it reaches its length, with its "wore off" message:
   - side condition 0x48 = **Reflect** (message names move 115), 0x49 = **Light Screen** (move 113),
     0x4C = **Mist** (move 54), 0x4B = **Safeguard** (**likely**: its own message, not a move name);
   - then **weather** (next section): rain, sandstorm, sun, hail, each with its "continues" / "stopped" message;
     sandstorm and hail set the animation argument (0x0C / 0x0D) for the weather-damage animation that follows
     (it is Gen 3's `animArg1`).
4. **Per-Pokémon damage and healing** (`fn_80230568`, in Speed order). The conditions it checks, in order, with the
   fraction of max HP used (`fn_80203B5C(pokemon, n)` = max HP / n); the condition names are **likely** (the
   fractions and order match Gen 3 exactly):

   | Condition id | Effect | Fraction |
   |---|---|---|
   | 0x25 | **Ingrain**: heals | 1/16 |
   | 0x1C | **Leech Seed**: drains to the seeder | 1/8 |
   | 0x03 | **Poison** | 1/8 |
   | 0x04 | **Bad poison** (Toxic), growing each turn | 1/16 x counter |
   | 0x06 | **Burn** | 1/8 |
   | 0x17 | **Nightmare** | 1/4 |
   | 0x18 | **Curse** (Ghost) | 1/4 |
   | 0x0E | **binding** (Wrap, Bind, ...) | 1/16 |
   | 0x0B, 0x0D, 0x29, 0x2A, 0x1D, 0x24, 0x30, 0x26 | then the countdowns, in this order: Uproar, a Thrash-style rampage (confused when it ends), Disable, Encore, Lock-On, Charge, Taunt, Yawn (puts the Pokémon to sleep when it runs out) (the order is **verified** from `fn_80230568`, and it is exactly Gen 3's end-of-turn order; the names are **likely**) | |

5. The later passes, in order (**verified** from code; condition names **likely**, from the
   behaviour and Gen 3's identical order):
   - `fn_8022FE80`: **faint handling**: each Pokémon flagged as fainting (status 0x120) gets the faint script
     (`lbl_80378721`), then the shared script `lbl_80378798` runs, then four per-Pokémon passes for the Pokémon now on the field
     (**verified**, `fight_range_exact_8022FE20.c` / `..._8022DCB8.c`): **Intimidate** (ability 22,
     when its "pending" condition 0x3B is set: the condition is cleared and the Intimidate script runs), **Trace**
     (ability 36, condition 0x3C), the **held-item check** (`fn_8022BE2C(pokemon, 1)`: berries, herbs; battle-items.md)
     and **Forecast** (`fn_8022EC40`: Castform's form for the weather). These are Gen 3's switch-in effects, in Gen 3's
     order, so a replacement sent in after a faint triggers them;
   - `fn_80230318`: **Future Sight / Doom Desire** (condition 0x34): a counter, a stored move and a stored target;
     when the counter reaches its length the stored move hits the stored target (if it is still there);
   - `fn_802301A8`: **Perish Song** (condition 0x1E): the remaining count is put in the message, and when the counter
     reaches its length the damage is set to the Pokémon's current HP (it faints), then faint handling runs again.
   That is Gen 3's end-of-turn order exactly: field effects, per-Pokémon effects, faints, Future Sight, Perish Song.
6. If no result yet: turn counter +1, **new turn random** rolled.

The action FIFO then returns to the select phase (`AllFightTrainerSelectFightAction`).

### 5. Weather (`fightFloorGetNowTenkouDataId`, the end-of-turn code above; **verified** from code)
Weather is stored as **battle-floor conditions**, one per kind and duration:

| Condition | Weather id | Kind |
|---|---|---|
| 0x4E | 0 | checked first; forces "no weather". A permanent field condition (its definition: floor, no countdown), but nothing in the game code sets it (searched): **likely unused** |
| 0x4F | 1 = sun | permanent (set by Drought) |
| 0x50 | 2 = rain | permanent (set by Drizzle) |
| 0x51 | 3 = sandstorm | permanent (set by Sand Stream) |
| 0x52 | 4 = hail | timed (Hail; there is no permanent hail ability in Gen 3) |
| 0x53 | 1 = sun | timed (Sunny Day) |
| 0x54 | 2 = rain | timed (Rain Dance) |
| 0x55 | 3 = sandstorm | timed (Sandstorm) |

The ids are fixed by the code that uses them: Swift Swim checks id 2 and Chlorophyll id 1, sandstorm and hail set the
weather animations 0x0C and 0x0D (the animation argument, corrected from "damage kinds"). The permanent rows are set by the three weather abilities on switch-in (**verified**,
`fn_8022E410`; battle-abilities.md) and never count down. Each condition carries its own length and current count
(`...GetKaisuuJoutaiDataId`, `...GetNowKaisuuJoutaiDataId`), so the 5-turn length is data, not code.

**Weather damage** (`fn_8021A984`, Gen 3's "weather damage" command 0x96; **verified**): 1/16 of
max HP per turn. Sandstorm spares Rock, Steel and Ground types (5, 8, 4) and Sand Veil (ability 8); hail spares Ice
types (15); both spare a Pokémon in condition 0x20 or 0x21 (**likely** underground and underwater: Dig and Dive's
semi-invulnerable turn). Weather here is read with Cloud Nine / Air Lock applied, so they stop the damage too.

**Cloud Nine (ability 13) and Air Lock (77)**: the weather getter with its check on returns 0 if any Pokémon on the
field has either ability. Speed, the accuracy check (`fightCheckAccuracy`) and the move-effect code ask with the check on; the end-of-turn
countdown (`fn_802317E4`) asks with it off, so **weather keeps counting down while it is suppressed** (**verified**
from the call sites; Gen 3 behaves the same).

### 6. The AI's scoring core (**verified** for the two helpers; the overall procedure is **likely**, from a long
partially matched function in `fight_range_80211A00.c`)
- Every option gets a **score** clamped to **-200 .. +200** (`fightTrainerAiAddValue`).
- For each of its Pokémon and each move, the AI asks the move's own **value function** (the first column of the
  per-move handler table in battle-engine.md). There are about 130 of them, named after the move or the effect they judge
  (`fightTrainerAiWazaValueAmagoi` = Rain Dance, `...Mamoru` = Protect, `...Jibaku` = Self-Destruct,
  `...TuujouIryoku` = plain damage, `...TuikaMahi` = "may paralyse", ...). Damage estimates come from the real damage
  code (battle-engine.md, "Damage prediction for the AI").
- (**verified** from code): **the scores themselves are data.** A value function doesn't return a
  number of its own; it checks conditions and, for each one that holds, adds a **reason**'s score
  (`fn_80239984(score, trainer, reasonId)`, clamped to +-200). The reason's score is the `Value` of row `reasonId` in
  the **AI value add / sub table** (553 rows of 20 bytes, `fightTrainerAiValueAddsubDataBiosGetPtr`); each row also
  carries three message ids (`KoudouName` = behaviour name, `PrefixName`, `Name`) that the AI's debug log prints next
  to the score (`fn_80239EE8`, `fn_8023A118`). Example: the plain-damage function (`...WazaValueTuujouIryoku`) adds
  reason 0x4D ("partner chose Helping Hand", see below) when its condition holds. So retuning the AI's taste (how much it likes status moves, switching,
  weather setters) is a table edit; the code only decides which reasons apply.
- **The reasons table, read** (**verified**): the add / sub table sits in `common_rel` (at file
  offset 0x11A6AC of the US `common_rel.rel`); each 20-byte row is {s32 score, u32 behaviour-name message, u32
  sub-category message, u32 reason message, u32 a fourth id}. **Reason id N reads row N - 1** (verified by seven reasons
  whose developer descriptions match the code that adds them). The descriptions are Japanese developer text naming each behaviour and reason. So the whole AI is readable: 553 named reasons with their scores (values not
  listed here: disc data).
- *Worked example* (`fightTrainerAiWazaValueAmagoi`, Rain Dance; **verified** from code): reason 0x14E if it is not
  raining yet; then for each of the trainer's Pokémon, reason 0x14F if it is Water-type or has Swift Swim or Rain
  Dish, and reason 0x150 if it is Castform (species 0x181, Gen 3's internal number 385, with Forecast); reason 0x151 if
  it is already raining. The scores come from the reasons table. So the AI "likes" Rain Dance in
  proportion to how many of its own Pokémon benefit.
- A random jitter can be added to each score. It is gated by a global switch, not a
  trainer flag: `fn_8000815C` reads byte `lbl_8047882E`, which is **1 in the executable's data** (so on by default,
  **verified**) and can be flipped from the debug menu's battle page (`fn_80007B30`, among the `dbgMenuFight*` functions: **likely**).
- The best score wins (`fightTrainerAiGetValueAryMaxBanme`): every option tied for the maximum is collected (up to 16)
  and **one is picked at random** when the caller asks for it, else the first. If nothing scores above -200, the AI
  has no choice (-1).
- **Switching** (`fightTrainerAiSelectFightActionIrekae`; **verified** structure): the AI first
  scores **withdrawing** the current Pokémon (`...IrekaeModosuValue`, *modosu* = bring back): it lists its own usable
  moves (move 357, the Shadow-locked placeholder, is skipped), checks against every opponent whether any move can hit
  and whether the opponent is faster, then adds a numbered reason's score for each condition that holds (reason ids
  1-6 and more: no move can hit, outsped, bad status, ...; the scores come from the AI value add / sub table,
  **likely**). Only if the total is **above 0** does it pick **who comes in** (`...SelectIrekaeDasuFightPokemon`,
  *dasu* = send out): every bench Pokémon is scored, an optional random jitter is added, and the best score wins
  with random tie-breaks; the switch action (kind 9) is then created like the player's.
- **The decision order** (`fn_802342CC`, **verified**): first a roll against the AI record's
  `IrekaeValue` (+0x07, a percentage) decides whether this turn it **considers switching** at all (then the
  switch logic above runs per Pokémon); then, if the battle allows items, a roll against `ItemValue` (+0x08) decides
  whether it **considers an item** (`fightTrainerAiSelectFightActionItem`, below); (**verified**): a **third roll, against
  `ComboValue`** (+0x06), sends each Pokémon still without an action to a combo step, `fn_8024E534`, which is an
  **empty stub** in Colosseum (it returns 0 and does nothing; XD reportedly has a full combo system there, per TuxSH, xd-insights.md (not yet published));
  then the Pokémon still without an action are sorted (**likely** by turn order) and each gets the **move
  scoring**. A debug switch (`fn_80008164`) shows each roll's result as a message.
- **Items** (`fightTrainerAiSelectFightActionItem`; **verified** structure): the trainer's
  bag (up to 20 ordinary items) is scored item by item. An item is skipped if its battle-use kind
  (`fightSeqGetItemType`) is 7 (not usable in battle) or a dry run of using it (`itemUse2PokemonSimulation`) shows it
  would do nothing. Otherwise reasons are added by kind: HP items when `fn_8023753C` holds (reason 0x2E; **likely** "HP is low"), status cures when
  `fn_80237310` doesn't (0x2F; **likely** "has a status"), and three more kinds (0x30-0x32; conditions not identified). **HP and status items are never
  scored for a Shadow Pokémon** (both checks exclude it), so computer trainers don't heal their Shadow Pokémon with
  items. The best score wins, as for moves (*checked 2026-10-16*, **verified** from the C: it first requires some
  item to score above 0, then takes the maximum with random tie-breaks). XD reportedly takes the first usable item
  instead (xd-insights.md (not yet published), TuxSH's XD write-ups).
- The AI writes a detailed debug log of its scores (message ids 0x34-0x3B, 0x225-0x228): the developers had a score
  viewer (**likely**; the messages go through the battle's debug message path).

### 7. The trainer-AI record (layout **verified** from the accessors in `fight_trainer_db_range_801FBD10.c`)
261 records of 0x28 bytes (`fightTrainerAiDataBiosGetPtr`, `lbl_80478F34`, count in `lbl_80478F30`): the personality
of a computer trainer. Offsets are verified; the meanings are read from the field names (**likely** where the name is
plain, **guess** where marked), not yet from the code that uses each one.

| Offset | Name in the code | Meaning |
|---|---|---|
| +0x00 | `PokemonSelectRandomFlag` | pick the next Pokémon to send in at random |
| +0x01 | `PokemonSelectWeakPointFlag` | send in one that hits the opponent's weak point |
| +0x02 | `PokemonDataOrderOutFlag` | send Pokémon out in party order |
| +0x03 | `PokemonDataOrderAceBossFlag` | keep the ace (boss) Pokémon for last |
| +0x04 | `KeyPlayerFlag` | protect / prioritise a key Pokémon (**guess**) |
| +0x05 | `PokemonJoutaiFlag` | take status conditions into account |
| +0x06 | `ComboValue` | chance (%) per turn of the combo step, which is an empty stub in Colosseum (**verified**, `fn_802342CC`, `fn_8024E534`) |
| +0x07 | `IrekaeValue` | chance (%) per turn that the AI considers switching (**verified**, `fn_802342CC`) |
| +0x08 | `ItemValue` | chance (%) per turn that it considers using an item (**verified**) |
| +0x09 / +0x0A | `ParamExpectFlag` / `ParamStoreFlag` | predict / remember the opponent's stats or moves (**guess**) |
| +0x0B / +0x0C | `ZokuseiCheckFlag` / `TokuseiCheckFlag` | consider types / abilities |
| +0x0D | `NokoriHpValue` | remaining-HP threshold (for healing or switching: **guess**) |
| +0x0E | `WazaDamageFlag` | score moves by predicted damage (the real-formula prediction in battle-engine.md) |
| +0x0F / +0x10 | `AbicntMaxValue` / `AbicntMinValue` | stat-stage limits for stat moves (**guess**) |
| +0x11 | `DefensePokemonRndSelectFlag` | pick the target at random |
| +0x12 | `WazaRndSelectFlag` | break ties between equal move scores at random (the `useRandom` of the best-score pick, **likely**) |
| +0x13 | `WazaInitValueFlag` | start scores from a base value (**guess**) |
| +0x14 | `PartFlag` | (**guess**: uses the per-Pokémon "part" data) |
| +0x15 | `WazaHitFlag` | consider accuracy (the per-move AI hit function) |
| +0x16 | `WazaAvgValue` | accuracy threshold (**guess**) |
| +0x17 | `WazaRiskFlag` | allow risky moves (recoil, self-KO: **guess**) |
| +0x18 | `WazaNokoriPpValue` | remaining-PP consideration |
| +0x19 | `LastValueRevise` | final random adjustment of the scores (the jitter in "AI scoring core", **guess**) |
| +0x20 | 2 x `{type, value}` (`ZokuseiRevise...`) | score bonus or malus for moves of a type (two entries) |
| +0x24 | 2 x `{type, value}` (`WazaTypeRevise...`) | the same per move kind (**guess**: physical / special or move category) |

So trainer difficulty is data: two trainers with the same Pokémon play differently through their AI record, and a
mod can make any trainer smarter by pointing it at another record. **the trainer record's u16 at +0x06
(trainer field 2) is its AI record id** (**verified**: the AI reads its switch and item chances through it, and on
the disc the 820 trainers use values 0-260, 185 distinct, against 261 AI records; 0 means no AI, i.e. a player). A second table, the **AI value add / sub** table (553 x 20 bytes,
`fightTrainerAiValueAddsubDataBiosGetPtr`), holds the score adjustments the switch logic uses (bad status, the
incoming Pokémon's ability: `fightTrainerAiGet...AddsubDataId`).

## Disc data

- The action-kind table and the five priorities are in the executable (`main.dol` data, `lbl_80375BB8`).
- Move priority is the move record's byte at +4 (move table in `common_rel`; see pokemon-data.md (not yet published) / game-data-tables.md (not yet published)).
- Held-item effect ids and their percentages (Quick Claw) come from the item table.
- Condition lengths come from the condition definitions and the count table (above), both in the executable.
- Trainer AI: the AI records (261 x 0x28) and the AI value add / sub table (553 x 20, the score of each reason) are in
  `common_rel`; a trainer names its AI record at +0x06.
- Hyper Mode and Call chances, and the purification rates: the nature records (+0x0B-0x11, +0x18, +0x1F; battle-shadow.md).

## Changing it

- **Turn order rules** are all in two functions (`...SortFightOutPokemonPtrArySub`, `...CmpfightOutPokemonNimbleness`)
  plus the effective-Speed function. A port option for later-generation rules (Trick Room, per-Pokémon Quick Claw
  rolls) hooks there.
- **Action priorities are data**: swapping two bytes in the action-kind table would, for example, let switching happen
  before Call. Keep the table as is for faithfulness.
- New weather would need new floor conditions, a new id in the getter, and a branch in the end-of-turn function;
  existing checks compare ids directly, so a new id is ignored safely by old code.
- AI: each move's value function is one table entry; a mod can give a move a different AI opinion without touching
  the shared scoring.
- **Data-only tuning** (no code): condition lengths (the count table, or a definition's length id); a trainer's AI
  personality (its AI record: switch / item chances, tie-breaking, which checks it makes); the AI's taste (the
  add / sub table's scores); Hyper Mode frequency and misbehaviours (nature records); which rules a battle has (the
  battle kind).
- **Code changes needed** for: new conditions with new behaviour, new action kinds, new AI reasons (each value
  function decides which reasons apply).

## Efficiency

Everything works on fixed-size arrays: 8 field slots, a 16-entry tie list, scores as plain integers. The sort is
quadratic but over at most 8 entries. One random number per turn serves every Quick Claw check. Each action is copied
onto the stack rather than allocated.

## Region notes

None: no text, encoding or layout code here.

## Open questions

- How story trainers' AI records differ from Colosseum and battle-mode trainers'.
- Runtime check in Dolphin (to turn **likely** into **verified**): hold Quick Claw on both sides in doubles and confirm
  they trigger together; confirm weather keeps counting down under Cloud Nine.
