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

### Action data lists (**verified** from code and the executable's data)
Creating an action (`fightActionCreate`) takes a kind, a "buff" value and a small **data list**: 8-byte records of a
u16 kind (+0x00) and a u32 buff (+0x04), ended by a kind-0 record. The action keeps the record whose kind and buff
match its own (`fightActionSetData`); with none it is refused. The move menu passes the list with one move record
(kind 0x13); the battle bag passes the list with one item record (kind 0x12, `fightOutPokemonCreateFightActionUseItem`);
neighbouring lists in the same data block hold the phase kinds (7 to 13, 14 to 17) and the move record's variants.
The list is a cheap permission check: a menu can only create the action kinds its list names.

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
The full selector list is in pokemon-data.md; these are the meanings learned from the battle code
(**verified** where the use pins it down, **likely** otherwise):

| Id | Meaning | Label |
|---|---|---|
| 0x63 | Pokédex "caught" flag (Repeat Ball) | verified |
| 0x6E | species | verified |
| 0x6F | personality value (`Rnd`) | verified |
| 0x70 | the record's origin stamp (`Attest`: game version, generation, area, language; "valid" when all four are nonzero, `gamedataAttestCheckValid`; not a checksum of the contents) | verified |
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
| 0xE6-0xEA / 0xEB / 0xEC | stat stages (Attack, Defense, Sp. Atk, Sp. Def, Speed) / accuracy stage / evasion stage; the record offsets are under "The field Pokémon record" below | verified |
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
| 0x118 | not identified yet | open |
| 0x11A | Focus Band triggered this hit | verified |
| 0x120 | fainting pending (end-of-turn faint handling) | verified |
| 0x121 | party slot to switch to | verified |

### The field Pokémon record, and where a battler's state lives 

Each Pokémon on the field has three records, read through `pokemonGetStatus` selectors (**verified** from the getters
`fightOutPokemonBios*`, `fightPokemonBios*` and their callers):

- **The battle floor** is one fixed object, `lbl_8046DD90` (0xA4E8 bytes; `fightFloorBiosGetFightFloorPtr`). At +0xA4C4
  it holds **8 pointers to the field Pokémon records**, one per field slot (`fightFloorBiosGetFightOutPokemonPtrAry`,
  selector 0x59 / 0x5A). The order changes during a battle (**likely**: re-sorted into turn order each turn), so a
  slot number doesn't name a battler. A replacement takes over the fainted Pokémon's battle record (**verified**: a
  trainer's second Pokémon came in on the first one's record).
- **The field records themselves live inside the floor, in a fixed layout** (**verified** from the
  getters and in a recorded battle): side *s* (0 = the player's, 1 = the opponent's) at floor +0x14 + *s* × 0x5230
  (`fightFloorBiosGetFightSidePtr`); trainer *t* of that side at side +0x64 + *t* × 0x28E4
  (`fightSideBiosGetFightTrainerPtr`); that trainer's field Pokémon *i* (0 or 1) at trainer +0x1314 + *i* × 0x6E0
  (`fightTrainer_GetFightoutPokemonPtr`). So the battle has **two orders** of the same field Pokémon: this fixed
  side → trainer → position order, which `fightFloorLoopValidFightOutPokemon` walks (the battle AI's list of
  opponents comes from it), and the slot array above, which is re-sorted. They need not agree: in one recorded
  double battle the player's first position held the Pokémon sent out first from the party, while the slot array
  listed the other one first from the send-out on.
- **The move's attacker and target** are field Pokémon records at floor +0xA474 and +0xA478 (floor statuses 0x36 /
  0x42, `fightFloorBiosGetAttackPokemonPtr` / `fightFloorBiosGetDefensePokemonPtr`; **verified** from the getters
  and against a recorded battle: each "used" line's target was the Pokémon that then lost HP or fainted).
- **Field Pokémon record** (the *fight-out* Pokémon: what is on the field right now; reset when it leaves):

  | Offset | Field | Selector |
  |---|---|---|
  | +0x000 | pointer: the original battle record (`MotoFightPokemon`) | 0xD5 |
  | +0x004 | pointer: the battle record in use | 0xD6 |
  | +0x008 | the Transform copy (`HensinBuff`) | 0xD7 |
  | +0x15C | **52 condition slots** of 16 bytes (`FightoutJoutai`): confusion, Leech Seed, Substitute, and the other field-only conditions | 0xD8 |
  | +0x49C | the move being used (`FightWaza`) | 0xD9 |
  | +0x5F4 | the item record in use | 0xE5 |
  | +0x604-0x608 | **stat stages**: Attack, Defense, Sp. Atk, Sp. Def, Speed (u8 each) | 0xE6-0xEA |
  | +0x609 / +0x60A | accuracy stage / evasion stage | 0xEB / 0xEC |
  | +0x610 | turns on the field (u16) | 0xED |

- **Battle record** (*fight* Pokémon: stays with the Pokémon for the whole battle, through switches): +0x004 is a
  **copy of the Pokémon's party record** (`PokemonBuff`: species, moves and PP at +0x78, HP at +0x8A, maximum HP at
  +0x8C, as in pokemon-data.md), and +0x13C is **one condition slot** (selector 0xCD).
- **The party record itself** has one condition slot at +0x064 (selector 0x7C): the status that is saved and survives
  the battle (sleep, poison, burn, paralysis, freeze; **likely**, from where it is kept).
- **A condition slot** (16 bytes; one struct for all three places): +0x00 u16 condition id (the ids below), +0x02 u16
  a second id (the source or stored value the condition needs), +0x04 s8 **turns left** (set from the condition's
  length table when written), +0x05 u8, +0x06 u8 **stack count** (conditions that stack, like bad poison's counter,
  capped by the condition's maximum), +0x08 u32 value, +0x0C u16. Which slot a condition uses comes from the
  condition's definition (`joutaiDataBiosGetSlot`), not from its id (**verified** from `statusWriteJoutaiDataId`).
- **Stat stages are stored with 6 as neutral** (range 0-12 for -6..+6; **verified** neutral: every battler reads 6 at
  the start of a battle; the range is **likely**, Gen 3's).
- **The move a Pokémon uses** is read from its move slot when the move starts (the "use a move" action re-reads it,
  battle-engine.md). A move slot is +0x78 + 4 × slot in the Pokémon record: u16 move id, u8 PP, u8 PP-ups. The move
  table gives a move's PP (entry +0x01; 0x38-byte entries, `wazaDataBiosGetPtr`). *Tested 2026-10-19:* overwriting a
  move in the battle record's copy at the start of a battle wrote the bytes, and the battle then went into the
  bag / party menu room (0x38F, `pocket_menu`) and never finished. **Likely** cause: a replayed input recording, not
  the write: the new move (Growl, which hits both foes in a double battle) skips the target choice the recorded move
  needed, so every later recorded press landed one menu off, one of them on the Bag or Pokémon command. A record
  check is ruled out: the record's only validity check is its origin stamp (`Attest`, below), which a move change
  doesn't touch.
- **The battle's windows during a turn** (**likely**: read from the window lists of a recorded
  street battle on the console and the PC, roles from when each opens and how many cursor positions it has): at the
  start of each command phase six windows open together, 0x45-0x48 and 0xF1 / 0xF3 (the Pokémon status boxes and
  their frames; they also open on their own during moves, when a box updates); the player's choice runs through
  **0x4B** (the four-choice command menu), **0x4C** (the move list, four positions) and **0xFF** (the target choice,
  two positions in a double battle). The six boxes close about four frames after the last choice, and **the opponent
  AI chooses in that closing frame** (both sides, verified frame by frame). Before the battle, 0x59 (six positions),
  0x5A and 0x10 are the team-choice screens.
- **Battle messages:** the battle's own message box is the message window of type 4 (menu id 0x50,
  message-windows.md), and its first open parameter, copied into its work area, is the plain message id
  (`winMsgOpenFight(0x75FC, ...)` in the battle menus; **verified**: the intro messages of a battle read back as their
  ids). **The battle scripts print through the same box:** command 0x10 (`printstring`, a 4-byte message id) calls
  `fightMenuOpenMsg`, which calls `winMsgOpenFight(id, 1, 1)` (**verified** from the code), and in a recorded battle
  the box opened 28 times, once before each move's text (**verified** from the window log), and its message id
  changed with each line: one id before every move (likely the "used" line) and others for the effects, such as a
  Speed drop's line once per Pokémon it lowered (**verified**: the ids, logged beside the battlers' stat stages).
  Looked up in the common message table, the ids seen most often are 0x768D ("<attacker> used
  <move>!", so the "used" reading above is **verified**), 0x753E (the attack missed), 0x7631 (a critical hit),
  0x7545 (a Pokémon fainted), 0x7670 (a trainer sends out a Pokémon), 0x756E (became confused) and 0x7602 (an
  ability raised a stat, Speed Boost's end-of-turn line). Most of these lines open together with one of the four
  status boxes (0x45-0x48), most likely the box of the Pokémon the line is about (**likely**: the "used" line opened
  with the attacker's box in a recorded double battle). 

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

The move checks before every move (`_wsHitCheckJoutaiSub` (0x8022F2F8)) run in Gen 3's order: sleep, freeze, Truant (ability 54),
recharge, flinch, Disable, Taunt, Imprison, confusion, paralysis, attraction, Bide, thaw, then Colosseum's Hyper Mode
check (**verified** order from the code; ids as in the table).

### Condition definitions and lengths (**verified** from code and the executable's data)
Every condition id has a 20-byte **definition** (`lbl_8035BBA8`, 86 entries, count `lbl_80478B48`; accessors
`joutaiDataBiosGetKaisuuId` (0x80119D90)-`joutaiDataBiosGetObjectKind` (0x80119F10)), and setting a condition (`statusWriteJoutaiDataId` (0x8011B2C0)) reads it:

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

### 0. The whole battle, phase by phase (`_fightInitialize`, the battle thread `fightMainThread` (0x801EFA08); **verified**)
Each phase is one action of the kind shown (action-kind table above), run through the action FIFO:
1. **On the scene thread** (the floor's enter callback `_fightInitialize`): the debug menu is closed and disabled for
   the battle, the action FIFO is set up, then `Kaijou` (kind 1, the venue) and **the trainers enter**, side 1 (the
   opponents) first, then side 0 (kind 3 with target ids 5, then 4). Then the **battle thread** is created (first
   argument 0x14, likely its priority; 16 KB stack).
   *Where the battle thread runs within a frame* (**verified** in a recorded double battle by
   watching which task and game thread made each random draw): in a frame where the AI decides, the per-frame logic
   task (priority 0x0A) first runs the battlers' idle-blink update on its own thread, then switches to the battle
   thread, which makes the AI's opening draws still inside that task. The battle thread then keeps running after the
   frame's task list is done (the per-Pokémon move and target draws, and the next frame's accuracy roll, happen
   outside any task). So the random draws the AI makes depend on that order: if the AI ran before the blink update,
   every choice would take a draw meant for the blinks.
2. **On the battle thread:** the floor's sound starts if it has one (floor status 0x12), resources load
   (`fightActionFlowKaisiPreSubLoad`), the screen fades in, `KaisiPre` (kind 2); **the Pokémon are sent out**, again
   opponents first (kind 4 with 5, then 4); `KaisiPost` (kind 5: on-entry checks, weather init); the total battle
   timer starts if the battle uses one.
   (**verified** from code): The trainers' entrance (`fightActionFlowKaisiNyuujouTrainer`) creates
   each side's trainers and their battle Pokémon. (**verified** by recording every random draw with
   its place in a PC run): This happens in step 1, **on the scene thread, before the battle thread exists**, all in one
   frame: the trainer's own draws (`heroCreate`), then each Pokémon's personality and IVs (`pokemonCreateRndFit`, which
   draws again and again until the personality gives the nature, gender and shininess the trainer's data asks for). One
   recorded battle took **956** draws for one trainer's team, another 116, so how much of the shared random sequence a
   battle start uses depends on how many retries the team's fixed natures need. The battle thread's own first draw came
   about 1,700 frames later. (**verified** in a console-emulator run that watched every draw's
   caller): 18 frames after the creation burst, still on the scene thread, the entrance **picks and orders the
   opponent's team** (`fightTrainerSortFightTrainerDataIdToHeroTemotiPokemon` → `fightTrainerSortTemoti`), with 6 more
   draws in that battle. So which Pokémon a computer trainer leads with can depend on the shared random sequence, not
   only on its data (**likely**; how the sort uses each draw is not traced yet). The Pokémon entrance (`fightActionFlowKaisiNyuujouPokemon`) takes,
   per trainer, the first two battle Pokémon that can fight. For each one it **loads the portrait and waits until it
   is ready**, then creates the model sequence and the
   "appears" message. Each action kind in the table has a flow function and a display function.
   (**verified**, per-frame log on a PC build): Each **new** portrait costs about **8-10 frames**
   (~0.15 s) of waiting, because the portrait streamer takes one step per tick (header, data, setup) and the entrance
   checks once per tick; a species whose portrait is already loaded waits 0 frames. The game keeps running and
   drawing during the wait (no frame is dropped); only the battle sequence pauses. On a console the disc read time
   adds to this: (**verified**, Dolphin), Willie's battle waits **14 frames** (Zigzagoon, one
   portrait shared by both), **12** (Umbreon) and **8** (Espeon). The portrait slots are a table of 24 records of
   16 bytes: a data word, a 16-bit key (the species' **National Dex number**, **likely**: it holds for all three cases
   seen, Zigzagoon 263, Umbreon 197, Espeon 196) and a state byte (1 loading, 2 ready); the entrance waits, one thread switch per frame, until its slot reaches 2.
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
  and goes to the AI (`fightTrainerAiSelectFightAction` (0x802342CC));
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
trainer or side). A debug switch (`dbgMenuFightGetAllManual` (0x80008174), off at boot) makes **every** trainer, AI ones included, use the
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

Because the **turn random is one number for the whole turn** (rolled at the end of every turn in `fightSeqTurnEnd` (0x80213270) and
once at battle start in `KaisiPost`), two Quick Claw holders in the same turn either both trigger or both don't
(**verified** from code; Ruby / Sapphire do the same with their per-turn random).

### 3. Running the actions (`_fightActionFlowAllFightOutPokemonDoFightActionOneLoop`; **verified**)
Two passes over the sorted list. Pass 0 runs only **Run** actions; then the move-sequence module is re-initialised
(`fightActionFlowAllFightOutPokemonDoFightActionPost` (0x80211A00): per-Pokémon sequence hooks registered, sequence flag bytes cleared); pass 1 runs every other action
in sorted order. A Pokémon whose action is missing or invalid, or who has fainted, is marked "done" (status 0x112)
and skipped. After each action of pass 1 the targets are reset, and **the turn stops as soon as a battle result is
set** (someone won, ran, or time ran out).

Each action is copied before it runs (`copy = *action; fightActionFlowFifo(&copy)`), so an action that changes the
queue mid-turn can't corrupt the one being run.

### 3b. Switching and running (**verified** from code unless marked)
- **Switch** (`fightActionFlowFightOutPokemonIrekae` -> `fightActionFlowIrekaeSub` (0x80213158)): stores the outgoing Pokémon (floor status
  0x45) and the party slot to bring in (Pokémon status 0x121, from the action's buffer id), clears three of the
  outgoing Pokémon's conditions (0x2E, 0x15, 0x28; which ones is not identified yet), then runs the switch
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

### 3c. Call and Hyper Mode (`fightActionFlowFightTrainerCall` -> `fightActionCallMain` (0x80212D6C))
**Verified** from code unless marked. Condition 0x3E is **Hyper Mode** (**likely**: it is what a successful Call
clears, and it's the condition behind Shadow Rush's 90% critical rule in battle-engine.md); condition 8 is **sleep**
(**verified**: the status table above; waking also clears it together with 0x17, Nightmare).
1. Like a switch, Call first clears the Pokémon's conditions 0x2E, 0x15 and 0x28.
2. **Shadow Pokémon in Hyper Mode:** success if `random % 100 < chance`, where the chance comes from the Pokémon's
   **nature** and its **heart-gauge level** (`pokemonGetCallRate` (0x8011F634): nature record +0x1F, seven bytes, one per gauge level;
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
   (`pokemonAddDpFormPokemonDpFilterId(.., 0, 2)`).
5. The trainer's call animation (`fightTrainerWazaSeq57` (0x801FB974), steps 0-3) and the messages "<trainer> called out ..." (0x7704,
   0x7705) frame the result; a transformed Pokémon (Transform, "Hensin") has its copied status refreshed.

Because Call has action priority 2, it always resolves **before** the Pokémon's own move in that turn would have
(the called Pokémon gives up its own action that turn: **likely**, the action is stored on the Pokémon) and before any
switch.

### 3d. Entering and leaving Hyper Mode on its own (`_fightCheckDarkPokemonAppear` (0x8022EDEC); **verified** from code and data)
The "can this Pokémon act" checks that run before every move (`_wsHitCheckJoutaiSub` (0x8022F2F8), a loop over cases 0-14 that stops at 15)
ask `_fightCheckDarkPokemonAppear` (0x8022EDEC) about Hyper Mode. Gen 3's "attack canceller" has cases 0-13 and ends with the thaw check; Colosseum's
case 13 is that thaw check too (condition 7, cleared by move effect 125: **condition 7 = freeze**, **verified** by the
status table above), and case 14 is Colosseum's addition. All of it needs: a valid Pokémon, the battle
kind's Hyper Mode rule on (floor status 0x31, the `DarkpokemonHypermode` flag), and a Shadow Pokémon that is still
shadowed (`fn_8011FC74`).
- **Entering (mode 0):** only if not already in Hyper Mode and not asleep, and **only when the move being used is
  Shadow Rush** (move 356). Chance: `random % 100 <` the nature record's byte at +0x18 for the Pokémon's heart-gauge
  level (seven bytes, one per level, `pokemonGetHyperModeRate` (0x8011F6D8)). Then the Hyper Mode condition is set with message 0x771B and the
  move goes on. In the US data these chances run from 5 to 70 and natures fall into six patterns (values not listed
  here: disc data).
- **Leaving by itself (mode 2):** while in Hyper Mode, each check has a **1 in 256** chance (`GSrand % 256 == 0`)
  to calm down on its own: Hyper Mode, sleep and Nightmare are cleared, message 0x771E.
- So the only reliable way out is **Call**, which always works in Colosseum (above).
- **Mode 1** is the last step of every Pokémon's end-of-turn pass (`_fightSeqTurnEndJoutaiSub` (0x80230568)): the "is in Hyper Mode" reminder
  (**verified**).
- **Misbehaving in Hyper Mode** (`_wsHitCheckSub` (0x8022A6C8)): when a Shadow Pokémon in Hyper Mode (Hyper Mode
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
  Mode (`wsCmdF0` (0x80213A78), the "give caught Pokémon" command 0xF0).
- How the heart gauge drops (in and out of battle): battle-shadow.md.

### 4. End of turn (`fightActionFlowOneTurnPost` -> `fightSeqPost` -> `fightSeqTurnEnd` (0x80213270); **verified** order)
1. Close status menus and messages; clear the eight sequence flag bytes.
2. Per-Pokémon turn checks (`_fightSeqTurnCheckSubFightOutPokemon`), reset the per-turn move conditions.
3. **Side timers and weather** (`fightSeqTurnEndAll` (0x802317E4), Pokémon re-sorted by Speed first). For each side, in this order, the
   turn count goes up and the condition ends when it reaches its length, with its "wore off" message:
   - side condition 0x48 = **Reflect** (message names move 115), 0x49 = **Light Screen** (move 113),
     0x4C = **Mist** (move 54), 0x4B = **Safeguard** (**likely**: its own message, not a move name);
   - then **weather** (next section): rain, sandstorm, sun, hail, each with its "continues" / "stopped" message;
     sandstorm and hail set the animation argument (0x0C / 0x0D) for the weather-damage animation that follows
     (it is Gen 3's `animArg1`).
     (**verified** from the data and a recorded battle): Sandstorm's and hail's lines come from a
     six-entry message table that the shared move script prints with a selector set by the turn-end code: "the
     sandstorm rages" / "hail continues to fall", "is buffeted by the sandstorm" / "is stricken by hail" (the damage
     lines, with the Pokémon's name), "the sandstorm subsided" / "the hail stopped". A second table, one record per
     weather (none, sun, rain, sandstorm, hail), holds each weather's start / continues / ends messages, plus the
     message shown when a battle starts in that weather (rain and sandstorm have one).
4. **Per-Pokémon damage and healing** (`_fightSeqTurnEndJoutaiSub` (0x80230568), in Speed order). The conditions it checks, in order, with the
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
   | 0x0B, 0x0D, 0x29, 0x2A, 0x1D, 0x24, 0x30, 0x26 | then the countdowns, in this order: Uproar, a Thrash-style rampage (confused when it ends), Disable, Encore, Lock-On, Charge, Taunt, Yawn (puts the Pokémon to sleep when it runs out) (the order is **verified** from `_fightSeqTurnEndJoutaiSub` (0x80230568), and it is exactly Gen 3's end-of-turn order; the names are **likely**) | |

5. The later passes, in order (**verified** from code; condition names **likely**, from the
   behaviour and Gen 3's identical order):
   - `wsRunEndOfTurnScript` (0x8022FE80): **faint handling**: each Pokémon flagged as fainting (status 0x120) gets the faint script
     (`lbl_80378721`), then the shared script `lbl_80378798` runs, then four per-Pokémon passes for the Pokémon now on the field
     (**verified**, `fight_range_exact_8022FE20.c` / `..._8022DCB8.c`): **Intimidate** (ability 22,
     when its "pending" condition 0x3B is set: the condition is cleared and the Intimidate script runs), **Trace**
     (ability 36, condition 0x3C), the **held-item check** (`fightOutPokemonDoSoubiItemCheck(pokemon, 1)` (0x8022BE2C): berries, herbs; battle-items.md)
     and **Forecast** (`fightGetForecastForm` (0x8022EC40): Castform's form for the weather). These are Gen 3's switch-in effects, in Gen 3's
     order, so a replacement sent in after a faint triggers them;
   - `fn_80230318`: **Future Sight / Doom Desire** (condition 0x34): a counter, a stored move and a stored target;
     when the counter reaches its length the stored move hits the stored target (if it is still there);
   - `_fightJoutai1ETurnEndSub` (0x802301A8): **Perish Song** (condition 0x1E): the remaining count is put in the message, and when the counter
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
`fightOutPokemonTokuseiCheckAppear` (0x8022E410); battle-abilities.md) and never count down. Each condition carries its own length and current count
(`...GetKaisuuJoutaiDataId`, `...GetNowKaisuuJoutaiDataId`), so the 5-turn length is data, not code.

**Weather damage** (`wsCmd96` (0x8021A984), Gen 3's "weather damage" command 0x96; **verified**): 1/16 of
max HP per turn. Sandstorm spares Rock, Steel and Ground types (5, 8, 4) and Sand Veil (ability 8); hail spares Ice
types (15); both spare a Pokémon in condition 0x20 or 0x21 (**likely** underground and underwater: Dig and Dive's
semi-invulnerable turn). Weather here is read with Cloud Nine / Air Lock applied, so they stop the damage too.

**Cloud Nine (ability 13) and Air Lock (77)**: the weather getter with its check on returns 0 if any Pokémon on the
field has either ability. Speed, the accuracy check (`fightCheckAccuracy`) and the move-effect code ask with the check on; the end-of-turn
countdown (`fightSeqTurnEndAll` (0x802317E4)) asks with it off, so **weather keeps counting down while it is suppressed** (**verified**
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
  to the score (`fn_80239EE8`, `fightTrainerAiLogText` (0x8023A118)). Example: the plain-damage function (`...WazaValueTuujouIryoku`) adds
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
- (**verified** from code, the executable's data and a replay): **each move's three AI functions.**
  The move record's +0x1C indexes a table of 20-byte rows (`lbl_8035C430`); a row names the move's **value function**,
  its **hit check** ("would this move work right now?", `fightTrainerAiWazaHitNNN`, numbered by the move's effect id)
  and its **damage function**, plus two move-animation callbacks. Mud Sport's row: the generic value function
  `...ValueToriaezutukae` ("use it for now"), `fightTrainerAiWazaHit201`, `fightTrainerAiWazaDamage201`. Before the
  value terms, `fightTrainerAiScoreWazaToTarget` asks the hit check and adds reason 0x3E (generic value function) or
  0x3C (others) when the move would work, reason 0x3D when it would not; the hit checks for "already active" effects
  ask whether the user already has the status (Mud Sport 0x38, Water Sport 0x39) or whether the weather is already the
  one the move sets, so the AI doesn't repeat them (in a replayed battle such moves scored −190). The jitter's
  half-width is the trainer AI record's field 0x38 (`randomBase`; the jitter is `rand % (2 x base + 1) − base`); with
  base 0 a random number is still drawn per move but adds nothing.
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
- **Who comes in** (`fightTrainerAiSelectIrekaeDasuFightPokemon`; **verified** from the code and
  seen on the console with a per-call random-number log): the same choice serves a voluntary switch and the
  replacement after a faint. The candidates are the party Pokémon that can switch in
  (`fightTrainerGetIrekaeFightPokemonAry`). Then, in order:
  1. if `PokemonSelectRandomFlag` is set (record +0x00), **one random pick** among the candidates, and that's the
     choice;
  2. if the outgoing Pokémon used **Baton Pass** (move 0xE2), a table of 7 candidate "kinds" is tried in order, and a
     random candidate of the first kind that has any is sent in;
  3. if `PokemonDataOrderOutFlag` is set, **the first lowest-level candidate in party order** (send out by data
     order, with the level test);
  4. otherwise every candidate is **scored**, and the highest score wins. The score adds: a bonus from the candidate's
     per-Pokémon "part" data kind (1, 2 or 3); bonuses for the AI's two favoured types (the record's two
     `ZokuseiRevise` type / value entries at +0x20, scaled);
     a penalty when none of its moves can hit any opponent now on the field; bonuses for being the best candidate on
     each of two matchup measures; a bonus for each super-effective move it has against each opponent (type
     multiplier 0x41, x2) and an adjustment for each opponent move that is super effective against it; weather
     bonuses by type (rain, sun, sandstorm and hail); an ability-based adjustment; and, when a debug-style switch
     (`fn_8000815C`) returns 1, random noise of plus or minus the record's `LastValueRevise`, **one random draw per
     candidate scored**.

  The bonus amounts come from the AI data tables (`fn_80239984`, value ids 0x1A-0x2C), not decoded yet. The flags are
  read through the trainer's status getter (`fightTrainerGetStatus` fields 0x1F `PokemonSelectRandomFlag`, 0x21
  `PokemonDataOrderOutFlag`, 0x38 `LastValueRevise`, 0x39 / 0x3A the `ZokuseiRevise` type and value), which reads the
  trainer's AI record (section 7). The names and the +0x00 and +0x20 offsets are **verified** from the getter; the other
  offsets are the section 7 table's. This also settles `LastValueRevise`'s meaning: it is the range of this random
  noise (**verified** for the replacement choice). Because the noise takes one draw per
  candidate, the number of random draws in a replacement choice equals the number of candidates scored: a wrong
  candidate list shifts every later random number in that battle by one.
- **The decision order** (`fightTrainerAiSelectFightAction` (0x802342CC), **verified**): first a roll against the AI record's
  `IrekaeValue` (+0x07, a percentage) decides whether this turn it **considers switching** at all (then the
  switch logic above runs per Pokémon); then, if the battle allows items, a roll against `ItemValue` (+0x08) decides
  whether it **considers an item** (`fightTrainerAiSelectFightActionItem`, below); (**verified**): a **third roll, against
  `ComboValue`** (+0x06), sends each Pokémon still without an action to a combo step, `fn_8024E534`, which is an
  **empty stub** in Colosseum (it returns 0 and does nothing; XD reportedly has a full combo system there, per TuxSH, xd-insights.md);
  then the Pokémon still without an action are sorted (**likely** by turn order) and each gets the **move
  scoring**. (**verified** from the code, `fightOutPokemonCheckFightActionWazaSelect`): A Pokémon
  whose move is **forced** never reaches the scoring; the same check that builds the list queues its action
  directly: with no usable move it gets Struggle (move 0xA5), and under **Encore** (condition 0x2A, whose status
  holds the encored move's slot) it gets the encored move (Encore ends first if that slot no longer holds the
  encored move). The forced move's target comes from `fightSeqSelectTarget` with no AI choice (a random opponent
  for a single-target move). So in a double battle a forced Pokémon also changes which random numbers its partner's
  choice takes. (**verified** from the code): Before any of this, the AI clears each of its field
  Pokémon's queued action unless the Pokémon is in the middle of a multi-turn move (conditions 0x12 and 0x22, the
  second turn of a charging or flying move), whose stored move it queues again with its stored target
  (`fightOutPokemonCheckFightActionSelect`); the list of Pokémon still to choose then leaves out any whose action is
  still valid (`fightTrainerGetNoActionFightOutPokemonPtr`: kind non-zero and an action data pointer set). The
  Speed the sort compares is `fightOutPokemonGetNowNimbleness`: the Speed stat, doubled by Swift Swim in rain or
  Chlorophyll in sun, then the stat stage, x1.1 for the player's side with the badge rule, halved by Macho Brace,
  quartered under condition 5 (paralysis, **likely**), and the maximum when a Quick Claw (held-item effect 0x1A) passes its check against the
  turn's first-attack random number (`random < chance x 0xFFFF / 100`). (**verified** with a per-call log on the PC port): The sort is the field's Speed order
  without move priority (`fightFloorSortFightOutPokemonPtrArySub` with `usePri` 0; Speed ties by a coin flip), so the
  faster Pokémon chooses first and takes the earlier random numbers. A trainer whose record has both random flags
  (+0x11, +0x12) spends exactly two draws per Pokémon: target, then move. Example: Chaser Emok's first battle in Pyrite
  (AI record 81: switch 0 %, item 25 %, combo 0 %, both random flags on) makes 3 + 2 x 2 = 7 draws per turn. A debug switch (`fn_80008164`) shows each roll's result as a message.
- **Items** (`fightTrainerAiSelectFightActionItem`; **verified** structure): the trainer's
  bag (up to 20 ordinary items) is scored item by item. An item is skipped if its battle-use kind
  (`fightSeqGetItemType`) is 7 (not usable in battle) or a dry run of using it (`itemUse2PokemonSimulation`) shows it
  would do nothing. Otherwise reasons are added by kind: HP items when `fightTrainerAiGetHpRatio` (0x8023753C) holds (reason 0x2E; **likely** "HP is low"), status cures when
  `fightTrainerAiIsJoutaiNormal` (0x80237310) doesn't (0x2F; **likely** "has a status"), and three more kinds (0x30-0x32; conditions not identified). **HP and status items are never
  scored for a Shadow Pokémon** (both checks exclude it), so computer trainers don't heal their Shadow Pokémon with
  items. The best score wins, as for moves (*checked 2026-10-16*, **verified** from the C: it first requires some
  item to score above 0, then takes the maximum with random tie-breaks). XD reportedly takes the first usable item
  instead (xd-insights.md, TuxSH's XD write-ups).
- The AI writes a detailed debug log of its scores (message ids 0x34-0x3B, 0x225-0x228): the developers had a score
  viewer (**likely**; the messages go through the battle's debug message path).

### 7. The trainer-AI record (layout **verified** from the accessors in `fight_trainer_db_range_801FBD10.c`)
261 records of 0x28 bytes (`fightTrainerAiDataBiosGetPtr`, `lbl_80478F34`, count in `lbl_80478F30`): the personality
of a computer trainer. Offsets are verified; the meanings are read from the field names (**likely**), not yet from
the code that uses each one. Fields whose meaning the name doesn't make clear are marked "not identified yet".

| Offset | Name in the code | Meaning |
|---|---|---|
| +0x00 | `PokemonSelectRandomFlag` | pick the next Pokémon to send in at random |
| +0x01 | `PokemonSelectWeakPointFlag` | send in one that hits the opponent's weak point |
| +0x02 | `PokemonDataOrderOutFlag` | send Pokémon out in party order |
| +0x03 | `PokemonDataOrderAceBossFlag` | keep the ace (boss) Pokémon for last |
| +0x04 | `KeyPlayerFlag` | not identified yet |
| +0x05 | `PokemonJoutaiFlag` | take status conditions into account |
| +0x06 | `ComboValue` | chance (%) per turn of the combo step, which is an empty stub in Colosseum (**verified**, `fightTrainerAiSelectFightAction` (0x802342CC), `fn_8024E534`) |
| +0x07 | `IrekaeValue` | chance (%) per turn that the AI considers switching (**verified**, `fightTrainerAiSelectFightAction` (0x802342CC)) |
| +0x08 | `ItemValue` | chance (%) per turn that it considers using an item (**verified**) |
| +0x09 / +0x0A | `ParamExpectFlag` / `ParamStoreFlag` | not identified yet |
| +0x0B / +0x0C | `ZokuseiCheckFlag` / `TokuseiCheckFlag` | consider types / abilities |
| +0x0D | `NokoriHpValue` | remaining-HP threshold |
| +0x0E | `WazaDamageFlag` | score moves by predicted damage (the real-formula prediction in battle-engine.md) |
| +0x0F / +0x10 | `AbicntMaxValue` / `AbicntMinValue` | not identified yet |
| +0x11 | `DefensePokemonRndSelectFlag` | pick the target at random: one draw, `rand % (number of opposing Pokémon on the field)`, over the field's list of them (**verified**, `fightTrainerAiSelectWaza`). (**verified** from the code): The list walks the opposing trainer's two field slots in order (slot 0, then slot 1: `fightFloorLoopValidFightOutPokemon`'s side → trainer → slot walk), so the same draw hits a different Pokémon when the two were sent into the other slots; such an AI can pick a target its move can't affect (a Normal move into a Ghost: "It doesn't affect …") |
| +0x12 | `WazaRndSelectFlag` | pick the move at random: one draw, `rand % (number of usable moves)`, over the Pokémon's moves in slot order with the unusable ones left out (encore, taunt, imprison, choice lock, no PP), **no scoring at all** (**verified**, `fightTrainerAiSelectWaza`) |
| +0x13 | `WazaInitValueFlag` | not identified yet |
| +0x14 | `PartFlag` | not identified yet |
| +0x15 | `WazaHitFlag` | consider accuracy (the per-move AI hit function) |
| +0x16 | `WazaAvgValue` | an accuracy value |
| +0x17 | `WazaRiskFlag` | allow "risky" moves |
| +0x18 | `WazaNokoriPpValue` | remaining-PP consideration |
| +0x19 | `LastValueRevise` | a final adjustment of the scores: the range of the random noise added to each candidate's score when choosing who comes in (**verified** from the code, section 6 "Who comes in") |
| +0x20 | 2 x `{type, value}` (`ZokuseiRevise...`) | score bonus or malus for moves of a type (two entries) |
| +0x24 | 2 x `{type, value}` (`WazaTypeRevise...`) | the same per move kind |

So trainer difficulty is data: two trainers with the same Pokémon play differently through their AI record, and a
mod can make any trainer smarter by pointing it at another record. **the trainer record's u16 at +0x06
(trainer field 2) is its AI record id** (**verified**: the AI reads its switch and item chances through it, and on
the disc the 820 trainers use values 0-260, 185 distinct, against 261 AI records; 0 means no AI, i.e. a player). A second table, the **AI value add / sub** table (553 x 20 bytes,
`fightTrainerAiValueAddsubDataBiosGetPtr`), holds the score adjustments the switch logic uses (bad status, the
incoming Pokémon's ability: `fightTrainerAiGet...AddsubDataId`).

### 8. One turn from move choice to the next turn, in order (a map for the first battle)
The sections above in the order one ordinary turn runs them, with the function at each step, so a stall or crash can
be placed quickly. Example: the first battle, against Willie (battle 10, an ordinary trainer battle, kind 2: Espeon and
Umbreon against his two level-24 Zigzagoon, two Pokémon per side). Each step is **verified** from code unless marked.
**Also verified by play** (the PC port running the
decompiled code, 2026-10-17): Willie's battle ran start to finish: both sides sent out two Pokémon, moves, the AI's
turns and fainting worked, Willie's defeat line and the prize money showed, and the game returned to the field.

1. **Choosing** (`fightActionFlowAllFightTrainerSelectFightAction`, section 1b). For each trainer: Willie's record names
   an AI record, so the AI chooses (`fightTrainerAiSelectFightAction` (0x802342CC), section 6: it predicts damage by running the real damage steps on
   saved copies, `fightSeqGetNromalWazaDamage`, battle-engine.md); the player gets the command menu, FIGHT, then the
   move list, then a target. Each choice becomes an action record in the action FIFO (`gFightActionFifo`, 32 slots of
   0x30), the target stored as a target id (battle-targets.md).
2. **Ordering** (`fightActionFlowAllFightOutPokemonDoFightAction`, section 1): the field's Pokémon sorted by action
   priority, then move priority, then effective Speed (section 2), ties by a coin flip from the RNG.
3. **Running** (`_fightActionFlowAllFightOutPokemonDoFightActionOneLoop`, section 3): pass 0 (running away only), the
   move-sequence module re-initialised (`fightActionFlowAllFightOutPokemonDoFightActionPost` (0x80211A00)), then pass 1 in sorted order. Each action is copied and run
   through `fightActionFlowFifo`; a Pokémon that has fainted is skipped; the loop stops once a result is set.
4. **Using a move** (`fightActionFlowFightOutPokemonOutWaza` → `fightSeqRunWaza` (0x802128D0), battle-engine.md "How a move picks its
   script"): the move is re-read from its slot; Encore, two-turn moves and a fainted target are handled (the target
   is re-chosen); the move's effect id picks its script (`lbl_80379BFF[effect]`), and the script interpreter runs it:
   the cursor `lbl_8047B610`, one handler per command byte from the 255-entry table `lbl_8027A00C`. **Script bytes
   hold big-endian 32-bit addresses for jumps and data** (battle-engine.md "Move sequences"), which is where a port
   must resolve GameCube addresses.
5. **The attack script** (effect 0, the plain hit; battle-engine.md "A plain attack, step by step"): the move checks
   (`attackcanceler`), accuracy (0x01, `fightCheckAccuracy`), "X used Y!", PP (0x03), critical hit (0x04), damage
   (0x05, the formula `fightSeqCalcDamage` (0x80232110)), type effectiveness and STAB (0x06, `fightSeqWazaTypeCalc` (0x802274F0)), damage variance (0x07,
   `fightApplyDamageVariance`), then the animations and waits (the move's WZX sequence: wzx.md), the HP bar, the
   critical / effectiveness messages, the HP written (`datahpupdate`), the secondary effect (0x15, `wsCmd15` (0x80224820)),
   the faint check for the target (`tryfaintmon`), the end-of-move steps (0x49, `wsCmd49` (0x8021FAD4)), end.
6. **A faint inside the move** (command 0x19, `wsCmd19` (0x80224158), Gen 3's `tryfaintmon`: the faint itself, with the
   friendship change below; then, run separately, the two-command scripts `lbl_80378721` = **getexp (0x23) only** and
   `lbl_80378798` = **checkteamslost (0x24) only**, both **verified** from their bytes, battle-engine.md "The first
   battle's scripts"; where the faint message and animation are played inside 0x19 is **likely**, not traced): **EXP** for the winning side (command 0x23, `wsCmd23` (0x802232F4):
   battle-rewards.md; levels, new moves and the "level up" flag that triggers evolution after the battle);
   **friendship** for the fainted player Pokémon (events 6 / 8: friendship.md); then **command 0x24, `wsCmd24` (0x80222C44)**
   (Gen 3's `checkteamslost`): if side 1 (the opponent) has no usable Pokémon left, the result becomes **2, the host
   side wins**; if side 0 has none, **3**; with neither, it schedules the send-in check `_fightCheckFaintedFightOutPokemonSub` (0x80222EF0).
7. **The next action** runs (back to step 4 for the next Pokémon in order), unless a result is set (step 3).
8. **End of turn** (`fightActionFlowOneTurnPost` → `fightSeqPost` → `fightSeqTurnEnd` (0x80213270), section 4): messages closed, the
   per-Pokémon turn checks, side timers and weather, damage and healing by condition, **faint handling**
   (`wsRunEndOfTurnScript` (0x8022FE80), then the switch-in effects for any replacement: Intimidate, Trace, held items, Forecast), Future
   Sight, Perish Song, then the turn counter and a new turn random.
9. **Replacing a fainted Pokémon.** *Choosing* (script command 0x50, `wsCmd50` (0x8021F24C), Gen 3's `openpartyscreen`; 6 bytes:
   a target id and a jump address): nothing happens if a replacement is already chosen (status 0x119); if the trainer
   has nothing left to send (`fightTrainerGetIrekaeFightPokemon` (0x801F8A18)), the script jumps to the given address; otherwise
   `fightTrainerSelectIrekaeFightPokemon` asks the same way as choosing actions (step 1): the AI
   (`fightTrainerAiSelectIrekaeDasuFightPokemon`) for a trainer with an AI record, the player's party menu
   (`fightMenuFightTrainerGcHeroSelectIrekaeFightPokemon`, or the GBA version) otherwise. The choice is stored as the
   fainted Pokémon's **switch target** (status 0x121, `IrekaeTargetEntryId`, the same field a normal switch uses), with
   0x119 set and floor status 0x45 pointing at it; a link error (-2) ends the battle with result 1. *Sending in*
   (`_fightCheckFaintedFightOutPokemonSub` (0x80222EF0)): for each field slot whose Pokémon is no longer out: if a switch target is recorded, the slot's
   roles are set (floor status 0x45 switching out, 0x46 fainting) and
   the send-in script `lbl_8037879E` runs; with none, the slot is cleared and the battle grid updated. Willie has no
   third Pokémon, so his slots empty as they faint (**likely**, from his two-Pokémon party).
10. **Back to choosing** (step 1), or, once a result is set, **the end of the battle** (section 0, step 4): the
    "Syuuryou" actions, prize money (command 0x5D, `wsCmd5D` (0x8021E754): battle-rewards.md), the after-battle steps
    (`fightActionFlowSyuuryouPost`: evolution checks, Pickup, the party written back to the save for story kinds),
    the fade-out, and the return to the field, where the room script reads the result (`std103_battleGetResult`,
    script command 103) and continues: for Willie, story flag 990 (new-game-flow.md, story-route.md (not yet published) step 11).

Where each kind of work happens, for a crash report: **choices** in the menu and AI code (step 1); **numbers**
(damage, accuracy, Speed) in plain C functions called from script commands (steps 2, 5); **text and animation** in
script commands that wait on the message and WZX systems (step 5); **state changes** through the `*SetStatus`
functions (floor, side, trainer, Pokémon, by status id: section "Per-Pokémon status ids" and battle-targets.md).

## Disc data

- The action-kind table and the five priorities are in the executable (`main.dol` data, `lbl_80375BB8`).
- Move priority is the move record's byte at +4 (move table in `common_rel`; see pokemon-data.md / game-data-tables.md (not yet published)).
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

### Mod surface
- **Data it reads:** the battle record and battle kind (its rules), the action-kind priority table, the condition
  definitions and count table, each trainer's AI record, the AI add / sub score table, nature records (Hyper Mode),
  move and species data.
- **Hooks it would need:** turn order (the two sort functions and effective Speed), a new condition's end-of-turn
  step, a new weather id, a move's AI value function, and before / after each action (for scripted events).
- **Fixed-size limits:** the condition and action-kind tables, the AI value-function table (one entry per move effect),
  the per-Pokémon condition slots in the battle record.
- **Must stay vanilla in GameCube-accurate mode:** turn order and its tie-break draws, every random draw and its order,
  condition timings, the AI's scores and choices.

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
