# Battle engine: battle setup, move sequences, critical hits and damage

Status: **partial**, well covered . This page: battle setup data, battle-kind rules,
the move-script machine (Gen 3's command set), how a move runs, critical hits, damage, accuracy, type effectiveness.

**The battle pages:**

| Page | Covers |
|---|---|
| [battle-engine.md](battle-engine.md) (this page) | setup, battle kinds, move scripts, a move step by step, critical hits, damage, accuracy, types |
| [battle-turn-flow.md](battle-turn-flow.md) | the battle's phases, choosing actions, turn order and Speed, switching, running, Call and Hyper Mode, end of turn, condition ids, weather, the AI |
| [battle-abilities.md](battle-abilities.md) | where every ability is checked |
| [battle-items.md](battle-items.md) | held items |
| [battle-shadow.md](battle-shadow.md) | every Shadow rule in battle, Rui's detection, the heart gauge |
| [battle-snag.md](battle-snag.md) | snagging and the catch formula |
| [battle-rewards.md](battle-rewards.md) | EXP and prize money |
| [battle-targets.md](battle-targets.md) | target ids and move target ranges |
| [battle-types.md](battle-types.md) | types, the type chart, the "???" type, adding a type |
| [battle-arenas.md](battle-arenas.md) | the battle stages |

How a battle starts (**verified**): `fightStartEncount(encounter)` prepares the wipe, music and
trainer name, then **pushes the arena's room on the room stack** (menu-rooms.md); the field is suspended and comes
back when the battle room pops. Details: new-game-flow.md, step 8.

**The battle entry, step by step** (**verified** from `fightStartEncount` in `fight.c` and
`fieldStartEncount` in `gs_range_801C766C.c`; flag names in the code are swapped: `std078` sets, `std079` clears):
1. A room script calls `fieldStartEncount(battle, fadeByCaller, noFade)` (our names for the two options, **likely**); Outskirt Stand's Willie battle passes
   (10, 1, 0). It clears story flag 0xE05 and, if either option is set, sets it for the battle's duration, which
   tells the battle not to fade the field back in itself (**likely** the meaning of 0xE05). (During a Mt. Battle
   challenge run, variable 0x8AE set, the battle goes through the `toolbattle_menu` runner instead: menu-rooms.md.)
2. `fightStartEncount(battle)`: clears flag 0x9B0, records the battle id and the room it was started from, looks up
   the arena (its room id), **stops mail delivery** for the battle, and for battle kinds that show the opponent's
   name, marks that trainer's name as heard and stores it for the "<trainer> would like to battle" text.
3. The field music fades out, the **screen wipe** starts from the battle's wipe record (duration, wipe style,
   whether it uses a snapshot of the screen, a sound), then the arena's room is **pushed** (Outskirt Stand:
   `S1_out_bf`, room 0xC8) and flag 0x9B0 is set (**likely** "a battle is running"). The calling script thread
   waits here until the battle room pops.
4. Afterwards: the room it returns from is recorded as the previous room; if the battle is a **whiteout battle**
   (its record's zenmetsu flag) and the player lost, the blackout runs (half the money, warp to the return point:
   battle-rewards.md, "Losing a battle").
5. Flag 0x9B0 is cleared, the field room's **sound script runs again** (`fn_80112700`: function 0 of the room
   script, room record +0x28), so the room's music comes back, pending field events are cleared, and unless 0xE05
   is set the field fades back in. With `fadeByCaller` set (Willie's case), `fieldStartEncount` does that fade
   itself with its own duration; with `noFade` the script handles it. The script then reads the result
   (`std103_battleGetResult`: 2 = the player's side won, **likely**; battle-turn-flow.md, "Battle result ids").

Related pages outside the battle : friendship.md (friendship changes from battles, Return /
Frustration), evolution.md (level-up evolutions after battle), field-steps.md (field poison and the blackout),
global-state.md (the battle's globals among all the game's state).

Code: `src/game/fight.c`, `fight_seq.c`, `fight_range_80211A00.c`
(the large shared battle unit), `fight_*`, `waza*.c`. Names are romanized Japanese: fight = battle, waza = move,
kouka = effect, jouken = condition, irekae = switching, tenkou = weather, tikei = terrain, tokusei = ability,
soubi = held item.

## What it does

A battle is set up from an **encounter** record (who fights, where, which music, which screen transition), runs
under a **battle kind** (a set of rule switches: can you run, catch, gain EXP, ...) and a **battle type**
(singles / doubles, how many trainers), and executes every move as a small script, a **move sequence** of `WS_*`
commands. Damage, critical hits, effects and messages are all steps in those scripts. Mechanically the core is
Pokémon Ruby / Sapphire's (Generation 3) battle system: the same damage formula, the same held-item effects and
abilities, with the same internal ids; Colosseum adds rule switches, doubles as the default, and Shadow Pokémon.

## Data structures

### Encounter record (`fightEncountDataBiosGetPtr(index)`, fight.c)
Per battle: up to N trainers (each with a trainer data id and an input device: player, CPU, ...), the battle floor
(arena), the battle kind, a "trainer" flag, the intro animation (`SyoukaiWzx`), the music (`BgmSnd`, see
`music.md` (not yet published)), a wipe (screen transition: sound, time, snapshot use, function), a "zenmetu" (whole-team-fainted) flag
and the battle's name. Setters exist too: the game builds or edits encounters at run time (`fightEncountDataBiosSet*`).

### Battle kind (`fightKindDataBiosGetPtr(index)`): the rules
One record per kind of battle, a list of yes / no switches the engine checks (names as in the code, meaning in plain
terms; meanings from the names, **likely**):

**Caution:** several of these are the decomp's names, shifted by one from offset 0x0B on; the
verified meanings are in "The full picture" below.

| Flag | Meaning |
|---|---|
| Nigeru | the player can run |
| UseItem | items can be used |
| Call | the **Call** command is available (Colosseum: calls a Shadow Pokémon out of Hyper Mode) |
| DarkpokemonHypermode | Shadow Pokémon can enter Hyper Mode |
| DoHizukiMiyaburi / DoHizukiAi | Rui's Shadow-Pokémon detection; her advice |
| GetExp / Keikentihueru | EXP is given / the EXP-boost rule applies |
| GetOkane / OkanePool / GetNekoniKoban | prize money / money pooling / Pay Day money |
| GetFriend | friendship changes |
| GetInfectPokerus | Pokérus can spread |
| Monohiroi / Dorobou | Pickup works / Thief and Covet keep the item |
| DoCriticalAttack | critical hits happen |
| DoItemSoubiTokukoutokubouup | the Soul Dew's Sp. Atk / Sp. Def boost applies |
| DoBadgeCheck | badge stat boosts apply |
| DoZukanMita / DoZukanTukamaeta | the Pokédex records seen / caught |
| Draw | a draw is possible |
| Boss | boss battle |
| BackSaveData | the party is restored afterwards (battle-mode style) |
| HostEnemyMsg, PokemonStatusMenuSubbar | message and menu presentation |

So "story battle", "Colosseum battle", "battle mode" and so on differ only by data: the same engine with different
switches. The engine reads them through `fightFloorGetStatus(floor, 0, id, 0)`, one big switch over the current
battle's state; ids 0x1C-0x34 are the battle-kind flags, **id = 0x1C + the flag's offset in the record** (0x1A is the
battle kind itself, 0x19 the Pokémon out per side).


**The full picture** (**verified** from the code that uses each status id; status id =
0x1C + offset). The decomp's `FightKindData` field names (`fight.c`) are right up to offset 0x0A and **shifted by one
from 0x0B on**: the real meaning of offset N is the decomp's name for offset N + 1. Evidence:

| Offset | Status id | Read by | Real meaning | Decomp's name at that offset |
|---|---|---|---|---|
| 0x01 | 0x1D | effective Speed, damage (badge boosts) | badge check | `DoBadgeCheck` (correct) |
| 0x04 / 0x05 / 0x06 | 0x20 / 0x21 / 0x22 | the command menu | items / Call / run allowed | `UseItem` / `Call` / `Nigeru` (correct) |
| 0x07 | 0x23 | battle result ids | draws allowed | `Draw` (correct) |
| 0x08 / 0x09 / 0x0A | 0x24 / 0x25 / 0x26 | EXP, prize money | EXP given / money given / money pooled | `GetExp` / `GetOkane` / `OkanePool` (correct) |
| 0x0B | 0x27 | EXP (friendship on level-up) | friendship changes | `GetNekoniKoban` (Pay Day): **wrong** |
| 0x0D | 0x29 | critical check | critical hits | `GetInfectPokerus`: wrong |
| 0x0F | 0x2B | Rui's detection | Rui's detection | `DoHizukiAi`: wrong |
| 0x10 | 0x2C | damage (Soul Dew) | Soul Dew boost | `DoHizukiMiyaburi`: wrong |
| 0x11 | 0x2D | EXP x1.5 | EXP boost | `DoItemSoubiTokukoutokubouup`: wrong |
| 0x15 | 0x31 | Hyper Mode (entry, misbehaviour) | Hyper Mode allowed | `Monohiroi` (Pickup): wrong |
| 0x0C | 0x28 | read once after the battle, right after Pickup, and the value is thrown away (`fightActionFlowSyuuryouPost`) | Pokérus spreading, whose step is gone (**likely**: Gen 3 spreads Pokérus at that point) | `GetFriend`: wrong |
| 0x0E | 0x2A | never read by the game code | Rui's advice (by the shift) | `DoCriticalAttack`: wrong |
| 0x12 | 0x2E | trainer entry: a friendship bonus for the player's Pokémon (with the friendship rule on) | boss battle (Gen 3's "league battle" friendship event) | `Keikentihueru` (EXP boost): wrong |
| 0x13 | 0x2F | secondary effect 31 (Thief / Covet): no stealing for non-player Pokémon when on | Thief rule | `Boss`: wrong |
| 0x14 | 0x30 | after the battle, Pickup (`heroCheckSetMonohiroiAllTemotiPokemon`) | Pickup works | `Dorobou` (Thief): wrong |

**The disc's battle-kind table agrees** (19 kinds): the story-style kinds have 0x0B (friendship), 0x0F (Rui's
detection), 0x11 (EXP boost) and 0x15 (Hyper Mode) on and the battle-mode-style kinds have them off; 0x0D (critical
hits) is on in every real kind; 0x0E, which the shift makes "Rui's advice", is on in only a few story kinds, which fits
advice and would make no sense for the decomp's name there (critical hits).

So one name in the middle of the decomp's list is in the wrong place; the most likely fix is that `GetNekoniKoban`
(Pay Day) is not at 0x0B and every later name moves up one. Pay Day's own script command (0x91, `fn_8021B0B0`) checks
only the general money rule (status 0x25), so there may be no separate Pay Day flag at all (**likely**). The shift covers 0x0B-0x15 (at least). The byte-matched code is unaffected;
only the decomp's field and getter names (`fightKindDataBiosGet...Flag`) are shifted, so the names were most likely
assigned to the getters by position from another source, probably XD's symbols, whose battle-kind record would then
have one more field in that range (**guess**; check against the XD decomp's record).

**The 19 battle kinds** (from each kind's name message, developer names in Japanese,
translated here; **verified** names, meanings of the abbreviations **likely**):

| Kind | Developer name (translated) | Notable rules on the disc |
|---|---|---|
| 0 | (empty) | |
| 1 / 2 | story, on the map: boss / ordinary trainer | EXP, money, Hyper Mode, Rui |
| 3 / 5 | story Colosseum: opponents 1-3 / the boss | as above, **money pooled** |
| 4 | test battle | |
| 6 / 7 | story Phenac Colosseum: opponents 1-3 / the boss | money pooled |
| 8 / 9 | story Mt. Battle "100 wins": opponents 1-99 / the 100th | EXP, no money |
| 10 / 17 | story Card e "Matrix" battle: ordinary / special | |
| 11 | story Card e consecutive battles | |
| 12 | battle mode: "just battle" (quick battle) | no EXP, no money, no Hyper Mode, running allowed |
| 13 / 14 | battle mode, single-player Colosseum: opponents 1-7 / the boss | as 12 |
| 15 / 18 | battle mode, single-player "100 wins": 1-99 / the 100th | as 12 |
| 16 | battle mode: link (communication) battle | as 12 |

(**verified**, counted on the US disc's battle table, 566 records): Kinds 10, 17 and 11 each
have **exactly one** battle record (records 8, 9 and 521), each with two trainer slots filled, while ordinary
trainer battles (kind 2) have 344. So the Card e kinds are present as single template-like records in the US data.
Whether anything in the US game starts them (the Card e menu code, `menu/cardesavedata.c`, is in the executable; the
e-Reader features were sold only in Japan, from public knowledge, not from code) is not traced: a region question (regions.md).
The Card e "Matrix" display is driven by a command processor (`_sysvarsProcessData`, `gs_range_80033278.c`)
that walks a data stream and waits for transfers between steps, which fits card data arriving from an e-Reader
(**likely**). The Card e menu (room 0x393, `carde_menu`) and its room `M1_carde_1` in Phenac City are on the US disc,
but they form a closed loop: nothing in normal US play leads in (**likely**: every known entrance path was checked;
details in card-e.md (not yet published)). On the US disc the display works as a battle-record viewer (**likely**). No room script starts
the Card e battle records (8, 9, 521) with a literal battle id, so if they are used, the Card e code starts them
itself (**likely**).

In story Colosseums the prize money is pooled (x4, `scriptAddPremium`, battle-rewards.md) and paid out together
(**likely**: paid when the challenge ends; the payout is not traced).

### Battle type (`fightTypeDataBiosGetPtr(index)`)
Number of trainers, Pokémon out per side (1 or 2), Pokémon entered per trainer, a name.

### Effect records (`fightKoukaDataBiosGetPtr(index)`)
An effect id, a target id and a **condition** id (`jouken`); `fightKoukaDoFightKoukaJoukenAndKouka(target, index)`
checks the condition and applies the effect. This is how item, ability and field effects are expressed as data.

## Flow

### Battle start
`fightStartEncount(index)` sets up the battle from the encounter record (trainers, floor, music, wipe) and starts
the battle sequence. `fightEncountGetBgmSndDataId` picks the music (music.md (not yet published) has the details).

### The battle thread and its turn loop (battle_main.c, `fn_801EFA08`)
The battle runs on its own thread. Its work is organised as **actions** queued in a FIFO and "flowed" (run) by
`fightActionCreateAndFlowFifo(queue, ..., kind, sub, order table)`; the order tables (`lbl_80375CC8`, `lbl_80375D30`,
`lbl_80375D80`) list which action kinds run in which order for the start, the turn and the end of a battle.
In order (from the code; the phase names come from the `fightActionFlow*` functions, the exact kind numbers per
phase are **likely**):
1. **Start** (`Kaisi`): load (`KaisiPre`), the trainers enter, then the Pokémon (with their appear messages), then
   the on-entry checks: Shadow Pokémon (Rui sees a Shadow Pokémon), held items, abilities (`KaisiPost`); weather is set
   up (`TenkouInit`). The battle music starts unless the floor names a sound; the screen fades in.
2. **Each turn** (until the battle result is set):
   - the info menu opens and **every trainer selects an action** (`AllFightTrainerSelectFightAction`: the player's
     command menu, the AI for the others); an optional per-command timer runs (Colosseum rule);
   - Focus Punch's "tightening its focus" runs before anything else (`WazaKiaipantiPre`);
   - **the actions run** (`AllFightOutPokemonDoFightAction`): use a move (`OutWaza`, which runs the move sequence),
     switch (`Irekae`), use an item (`UseItem`), **Call** (`Call`, Colosseum's Hyper Mode cure), run (`Nigeru`);
   - **end of turn** (`OneTurnPost`): weather and status damage, timed effects;
   - an optional total battle timer can end the battle (time-out result).
3. **End** (`Syuuryou`): result messages, EXP and money, the fade-out, then the battle's own clean-up; control
   returns to the field (`floorSetFadeScript`). `std103_battleGetResult` is how scripts read the outcome.

### Move sequences (`WS_*` commands)
Every move runs as a script: a byte stream read through a cursor (`lbl_8047B610`), one handler per command
(`WS_CRITICAL_CHECK`, `WS_ABARERU` (Thrash), `WS_KUROIKIRI` (Haze), `WS_NEKODAMASHI` (Fake Out), `WS_WEATHER_CHANGE`,
`WS_STATUS_SET8/16/32`, `WS_STATUS_GET8/16/32`, conditional jumps, waits for messages and animations, ...). A
command advances the cursor by its own length; jump commands load a new cursor from their operands (for example a
10-byte conditional jump: target slot at +1, a 4-byte condition kind at +2, the jump target at +6). Eight shared
flag bytes (`lbl_80478D78`) carry state between commands (which branch was taken, what to report).
(verified from code and the executable's data) The command table (`lbl_8027A00C`)
has **255 entries**, one handler per command byte, starting with `WS_HITCHECK` and `fightCheckAccuracy`; the
interpreter calls `table[*cursor]` until a handler ends the script. The scripts themselves are byte streams in the
executable's data (for example `lbl_80375F98`, `lbl_80377AB8`, `lbl_803798D8`), and a jump stores its 32-bit target
address right after the command byte, so addresses sit at odd offsets in the stream (the link step relocates them
there). Operands are big-endian like all GameCube data.

### The command set is Ruby / Sapphire's battle-script commands 
The 255 `WS_*` command numbers line up with the **Generation 3 battle-script commands** (Ruby / Sapphire's
`gBattleScriptingCommandsTable`, known from the public Ruby / Sapphire / Emerald disassemblies), number for number.
Evidence (now verified): the Colosseum column is **verified** (the handler at that table index
has that name or behaviour); the Gen 3 column was checked against the public Emerald disassembly's command list
(pret/pokeemerald `asm/macros/battle_script.inc`, read 2026-10-16: 249 commands, 0x00-0xF8, Emerald's last one,
0xF8, being an Emerald addition), and every row matches by number. **Strongest check**: the cursor step
of each handler was read from the C source for the 200 commands whose handlers advance by a constant: **192 step
exactly Gen 3's length, 8 differ**: 0x0A (`waitanimation`, 2 bytes: one argument), 0x10 (`printstring`, 5: a 4-byte
message id instead of 2), 0x4D (3), 0x4F (7), 0x5D (`getmoneyreward`, 5: a jump address for "no money"), 0x70 (1),
0xAA (`setdestinybond`, 5), 0xF0 (`givecaughtmon`, 5). With those 8 lengths, Gen 3's for the rest, and 20 bytes for
Colosseum's commands 0xF8-0xFD, all **214 move-effect scripts**, each decoded linearly from its start (branch targets
not followed), run cleanly to an end command (return / end / goto / finish) (**verified**, 214 / 214). Colosseum's six extra commands (`WS_STATUS_GET8/16/32`, `WS_STATUS_SET8/16/32`) are 20 bytes:
an id at +1 and pointers or values at +2, +6, +0x0A, +0x0C and +0x10, passed to the game's general status functions
(`statusSetStatus` / `statusGetStatus`, `status.c`). The id is an **object kind**: 1 save data (`gamedata`), 2 PC
box, 3 the player (`hero`), 4 an item, 5 a Pokémon, 6 a move, 7 the battle floor, 8 a side, 9 a trainer; the other
operands are the object, the field id and an index, as for the matching `*GetStatus` / `*SetStatus` function
(**verified**, `status.c`). So a battle script can read or write any of these, including the save data; 104 of them appear on the main paths of the 214 scripts.

| Index | Colosseum handler | Gen 3 command at that index |
|---|---|---|
| 0x01 | `fightCheckAccuracy` | accuracy check |
| 0x04 / 0x06 | `WS_CRITICAL_CHECK` / `WS_TYPE_CHECK` | critical-hit calc / type calc |
| 0x07 / 0x08 | `WS_DAMAGE_LOSS` / `_ONLY` | adjust damage (Endure, False Swipe) |
| 0x16 / 0x17 | `WS_TSUIKA_DIRECT_ACT` / `INDIRECT_ACT` | set primary / secondary effect |
| 0x1C / 0x1F / 0x21 / 0x22 | condition / side-condition / move-condition / type checks | jump if status / side affecting / status3 / type |
| 0x28 | `fn_80222ACC`: cursor = the 4-byte address after the command | goto |
| 0x3C-0x3F | `WS_SEQRET`, `WS_WAZAEND`, `WS_SEQEND`, `WS_SPEABIEND` | return, end, end2, end3 |
| 0x5F | `WS_SWITCH_A_D`: toggles state bit 0x1000 | swap attacker with target (the same bit, 1 << 12) |
| 0x82 / 0x8A / 0x8C | Fake Out / Haze / Thrash handlers | jump if not first turn / normalise buffs / confuse when Thrash ends |
| 0x93 / 0x95 | `fightCheckOneHitKO` / `WS_WEATHER_CHANGE` | try KO / set weather |
| 0xAA, 0xB1, 0xBD, 0xBF | Destiny Bond, Foresight, Psych Up, Defense Curl | the same four moves' commands |
| 0xB7 | `fn_8021799C`: Present (random 40 / 80 / 120 power, or heals a quarter) | Present damage calculation |
| 0xC5-0xC7, 0xCB, 0xCE, 0xD0, 0xD3, 0xD5, 0xD7, 0xDC, 0xE4 | semi-invulnerable on / off, Minimize, Charge, Torment, Taunt, Role Play, Ingrain, Yawn, Grudge, Secret Power | the same commands |
| 0xF6 / 0xF7 | `WS_ITEMEND` / `WS_GETEND` | finish action / finish turn (R/S's last command) |
| 0xF8-0xFD | `WS_STATUS_GET8/16/32`, `WS_STATUS_SET8/16/32` | **none: Colosseum's own additions** |

**A whole script decodes the same way** (**verified** from the executable's bytes, read with Gen 3's argument
layouts): the switch script `lbl_8037889D` (battle-turn-flow.md) is `74 14` (HP thresholds, the switching Pokémon),
`13 <4-byte table address>` (print from a message table), `12 00 40` (wait for the message, 0x40 frames, Gen 3's long
wait), `2E <address> 00` and `2E <address> 08` (set two bytes), `BA <address>` (jump if no Pursuit damage on switch),
`5F` (swap attacker and target), `AB` (try Destiny Bond), `41 <address>` (call the Pursuit damage script), `5F`
(swap back). That is Gen 3's switch-out script with Pursuit, command for command, and every length fits.

So Genius Sonority ported Game Freak's battle scripts and their interpreter, then added generic read / write
commands for its own status system. **Consequence (verified for 192 of the 200 lengths checked, likely for semantics):** the Gen 3 disassemblies' command reference (argument
layouts, what each command does) describes most Colosseum commands, and Colosseum's scripts should read like Ruby /
Sapphire's `data/battle_scripts_1.s`. The state bits in `lbl_8047B618` match Gen 3's "hit marker" bits where checked
(0x1000 swap attacker / target), so that word is Colosseum's `gHitMarker` (**likely**). Argument layouts differ in
two ways: Colosseum stores addresses big-endian (GameCube), Gen 3 little-endian, and the 8 commands listed above
have different lengths.

### A plain attack, step by step (effect 0's script, decoded)
Decoded from the executable with Gen 3's command names and lengths (the decoding is **verified**; that each command
does what its Gen 3 name says is **likely**, except where this page describes the handler):
1. **Colosseum preamble** (not in Gen 3): read battle-floor field 0x37 (the attacker's Pokémon status 0xDA) and, unless
   it is 0x39 or the target has move condition 0x21, set state bit 0x40000 and floor field 0x39 (the attacker's status
   0xDC) to 2. What 0xDA / 0xDC hold is not traced.
2. `attackcanceler` (the move checks: sleep, freeze, Truant, ... battle-turn-flow.md), `accuracycheck` (jumps to the
   miss script if it misses), `attackstring` ("X used Y!"), `ppreduce`, `critcalc`, `damagecalc`, `typecalc`,
   `adjustnormaldamage` (variance, Focus Band, Endure).
3. `attackanimation`, `waitanimation` (with a 1-byte argument in Colosseum), `effectivenesssound`, `hitanimation`,
   `waitstate`, `healthbarupdate`, the critical-hit and effectiveness messages, then `datahpupdate` (the HP data is
   written after the messages; Gen 3 writes it before them) and a second `waitanimation`.
4. `seteffectwithchance` (the move's secondary effect), `tryfaintmon` (faint check for the target), `moveend`
   (end-of-move effects: held items, abilities, ...), `end`.

So the body is Gen 3's "effect hit" script, with Colosseum's preamble, a cleared byte after the damage adjustment,
waits with arguments, and the HP data written after the messages.

### End of a move (`moveend`, command 0x49, `fn_8021FAD4`; **verified** where named)
A small state machine (step counter `lbl_8047B627`, steps 0-17, like Gen 3's 17 steps plus "done") runs after each
move. Identified steps: 0 **Rage** (condition 0x13: the hit Pokémon's Attack rises), 1 a thaw / status step, 2
**Synchronize** for the target (`fn_8022D2CC`), 3 the **contact abilities** (`fn_8022D6BC`), 5 Synchronize for the
attacker (`fn_8022D20C`), 6 the **Choice Band lock** (held-item effect 29 sets condition 0x36, the locked move), 9 **King's
Rock / Shell Bell** (`fn_8022BB84`), 14 the "last move used" bookkeeping (for Mirror Move and friends), 16 **next
target**: for a move that hits several Pokémon, the effect's script (`lbl_80379BFF[effect]`) is run again for the
next target. Steps 0-6 are in Gen 3's order; King's Rock comes at step 9 where Gen 3 has it at 12, so steps 7-15 are
at least partly reordered (not all identified).

### How a move picks its script (**verified** from code and the executable's data)
`fightActionFlowFightOutPokemonOutWaza` (the "use a move" action) re-reads the move from the Pokémon's move slot (in
case it changed since the choice), then `fn_802128D0` starts it:
1. clears the per-move floor fields and flags;
2. **Encore** (**likely**: condition 0x2A, a forced move kept in a slot): replaces the chosen move with the encored one
   and re-picks the target, except for move 165 (Struggle);
3. **move 165 (Struggle) and move 356 set state bit 0x800**, which is Gen 3's "don't deduct PP" bit (**likely**).
   Move 356 is past Gen 3's last move (354): it is **Shadow Rush** (**verified** from the disc: its description is the
   recoil text, 90 power, effect 48 = recoil, the same effect as Struggle), the only Shadow move in Colosseum. It also
   has the special critical-hit rule in "Critical hit" below. Move 357 is the "heart's door is shut" placeholder for a
   Shadow Pokémon's locked moves (disc text);
4. **two-turn moves** (move effects 26, 39, 75, 145, 151, 155: Bide, Razor Wind, Sky Attack, Skull Bash, Solar Beam,
   Fly / Dig; **verified** against the disc's move records: Razor Wind, Sky Attack, Skull Bash, Solar Beam, Fly, Dig
   and Bide carry exactly these effect ids) pick their target only on the turn they hit; Solar Beam in sun
   picks it at once (weather id 1, the sun);
5. if the target has fainted, the target is re-chosen (the partner, or the other opponent);
6. sets the move's name for messages and **runs the script `lbl_80379BFF[effect]`**, where `effect` is the move
   record's field 9 (its move effect id).

`lbl_80379BFF` holds **214 script addresses**, exactly the number of Gen 3 move effects (0 = plain damage ...
213 = Camouflage in Gen 3's numbering), and starts at an odd address like the scripts it points to. So in Colosseum,
as in Ruby / Sapphire, **a move's behaviour is its effect id**: moves that share an effect share a script, and the
move's other fields (power, type, accuracy, effect chance) parameterise it. The effect ids are Gen 3's (**verified** for
the moves checked against the disc's move records: Pound 0, Karate Chop 43 (high critical), Double Slap 29 (multi-hit),
Quick Attack 103, Struggle 48 (recoil), the two-turn moves above; the highest id used is 213). The effect id is the
move record's u16 at +0x1A (`wazaDataBiosGetSeqId`, field 9 of `wazaGetStatus`).

**All 354 Gen 3 moves are unchanged in Colosseum** (**verified**): every move 1-354 in the US disc's
move table was compared with the public Emerald disassembly's move data (pret/pokeemerald `src/data/battle_moves.h`,
read 2026-10-16): **effect, power, type, accuracy, PP, secondary-effect chance and priority are identical for all 354**,
and the effect names map one-to-one onto Colosseum's effect ids (198 effects in use, no conflicts; anchors Pound 0,
Karate Chop 43, Double Slap 29, Camouflage 213 match Emerald's numbers). So Colosseum made no balance changes to the
Gen 3 moves; its only additions are 355-357 (355: an entry that `pokemonWazaCheckValid` treats like "no move", **verified** from the assembly; 356 Shadow Rush; 357 the locked-move placeholder). (Emerald's
values for these fields are taken to equal Ruby / Sapphire's: **likely**.) Fields not compared: target range, the
flag bytes (contact, Protect, ...), names and descriptions.

### Per-move handler table (verified from code)
Besides its script, every move has a row of five handler addresses in the executable's data (`lbl_8035C430`, 0x14
bytes per row, one row per move; the move record's u16 at +0x1C is the row index): +0x00 the trainer AI's **value**
function, +0x04 the AI's **hit** function, +0x08 the AI's **damage** function, +0x0C the WZX effect **type** function,
+0x10 the WZX effect **variation** function (accessors `wazaDataBiosGet/SetFightTrainerAiWaza*FuncPtr`,
`wazaDataBiosGetFightWazaWzx*FuncPtr` in `waza.c`). So the AI's opinion of a move and the choice of its visual effect
are code per move, picked through this table; a mod that adds a move needs a row here too (or a default).

### Cries in battle (verified from code)
- **One cry per species:** the species record's cry id (`pokemonGetStatus(.., 0x61)` = `pokemonDataBiosGetVoice`).
  There are no per-situation variants.
- **Send-out** (`fight_out_pokemon_suffix_80207C6C.c`): the code starts the send-out animation on the Pokémon's
  model sequence, plays the cry at once (`soundStart(cry, 0, 0xFF, 0)`), then waits until the animation ends; a shiny
  then gets an extra effect (`pokemonCheckRare` -> sequence 0x67).
- **From move scripts:** a two-byte script command (`fn_8021EE48`: command, target) plays the target's cry the same
  way, so any move or battle event script can cry (which scripts use it is not traced yet).
- **Timing:** a cry is started by code when an event begins, not by an animation frame. Move-effect sounds are
  different: WZX sequences start sound resources from their timed nodes (`wazaSequenceEntry.c`).
- **No pitch or volume tricks found:** every cry call uses the same arguments (pan 0, volume 0xFF). Whether a faint
  plays a cry, and how (a script command or code), is open.

### Critical hit (WS_CRITICAL_CHECK, fight_seq.c)
The critical stage is the sum of: Focus Energy (+2), a high-critical move (+1; move effects 43, 75, 200, 209), Scope
Lens (+1), Lucky Punch on Chansey (+2), Stick on Farfetch'd (+2); capped at the last row of the critical table. No
critical hit if the defender has Battle Armor or Shell Armor, or if the battle's critical-hit rule (flag offset 0x0D,
see above) is off; otherwise
the chance is 1 / (the table's divisor for that stage). One extra rule: move 356 with the attacker's status 62 set
crits 90% of the time. Move 356 is **Shadow Rush** (verified from the disc's move data) and
status 62 (0x3E) is **likely Hyper Mode** (what Call cures; battle-turn-flow.md), so: Shadow Rush in Hyper Mode crits
90% of the time (**likely**). The result (1 or 2) is stored as the
move's damage multiplier.

### Damage (the core formula, `fn_80232110`)
In order, as the code does it (**verified** from the decompiled code, which is a byte-for-byte match of the game;
every id below is a Gen 3 internal id: ability, held-item effect, species, type):

1. **Stat modifiers:** Huge Power / Pure Power double Attack. Badge boosts (rule 0x1D on, the trainer has the badge)
   x1.1 to Attack, Defense, Sp. Atk, Sp. Def (flag offset 0x01). Type-boosting held items (a 17-row item / type table) raise Attack or
   Sp. Atk by the item's percentage, by the move type's physical / special category. Choice Band x1.5 Attack. Soul
   Dew (flag offset 0x10, see above) x1.5 Sp. Atk / Sp. Def for Latias and Latios. Deep Sea Tooth / Scale x2 Sp. Atk / Sp. Def for
   Clamperl. Light Ball x2 Sp. Atk for Pikachu. Metal Powder x2 Defense for Ditto. Thick Club x2 Attack for Cubone and
   Marowak. Thick Fat halves the attacker's Sp. Atk against Fire and Ice moves. Hustle x1.5 Attack. Plus / Minus x1.5
   Sp. Atk with the partner ability on the field. Guts x1.5 Attack, Marvel Scale x1.5 Defense when statused.
2. **Power modifiers:** Mud Sport halves Electric moves, Water Sport halves Fire moves; Overgrow / Blaze / Torrent /
   Swarm x1.5 for their type at a third of HP or less. Explosion-type moves halve the target's Defense.
3. **Base damage** (physical or special by the move's type, as in Gen 3):
   `attack x power x (2 x level / 5 + 2) / defense / 50`, with stat stages applied; on a **critical hit** the
   attacker's stages count only if they help and the defender's only if they hurt.
4. **Physical only:** burn halves damage (not with Guts); Reflect halves it (two thirds in doubles with two targets).
5. **Special only:** Light Screen the same way; rain: Fire x0.5, Water x1.5; sun: Fire x1.5, Water x0.5; Solar Beam
   halved in any weather but sun; Flash Fire x1.5 for Fire moves.
6. Spread moves hitting two targets in doubles: x0.5. Minimum 1, then **+2**.

The caller then multiplies by the critical multiplier and another per-move multiplier (STAB / type effectiveness:
**verified**, see "Type effectiveness" below), doubles Electric moves after Charge, x1.5 with Helping Hand, and applies the random
variance (`fightApplyDamageVariance`).

### Damage variance and surviving at 1 HP (`fightApplyDamageVariance`; **verified** from code)
- **Variance:** damage x (100 - `random % 16`) / 100, i.e. 85% to 100% in 1% steps, at least 1 (Gen 3's).
- **Focus Band** (held-item effect 39): survives with a chance equal to the item's value (%).
- Unless the target is behind a **Substitute** (condition 0x14): **False Swipe** (effect 101), **Endure** (condition
  0x2C) or a Focus Band that triggered leave it at 1 HP when the damage would knock it out; the Endure and Focus Band
  cases set a result flag for their messages.

### PP use (`fn_8022808C`, Gen 3's `ppreduce` 0x03; **verified**)
No PP is used when state bit 0x800 (Struggle, Shadow Rush) or 0x200 is set, or while the attacker's status 0x118 is set (**guess**: the later turns of a multi-turn move;
not identified). Otherwise 1 PP, plus 1 per **Pressure** (ability 46) holder among the Pokémon the move targets: every other
Pokémon for target range 6, the opponents for ranges 4 and 7, otherwise the target (battle-targets.md, "Move target
ranges"). The PP lives on the party Pokémon (status 0x80, per move slot).

### Secondary-effect chance (`fn_80224820`, Gen 3's "set effect with chance" 0x15; **verified**)
The chance is the move's effect chance (field 0x0C), **doubled by Serene Grace** (ability 32); roll `random % 100`
against it, then apply the effect as a secondary effect (`WS_TSUIKA_INDIRECT_ACT`'s path). A menu switch
(`fn_8000817C`, byte `lbl_8047A271`, 0 at boot, set from a yes / no prompt in the debug menu's battle page, `fn_80007B30`, which sits among the `dbgMenuFight*`
functions: **likely**) forces the chance to 99: a developer option, off by default.

### Secondary-effect ids (`fn_802249B8`, behind `WS_TSUIKA_DIRECT_ACT` / `INDIRECT_ACT`)
The effect to apply is in sequence flag byte 3 (`lbl_80478D78[3]`). Before the main `switch`, a per-effect table
(`lbl_80279EF4[effect]`) names a condition that blocks the effect if the target already has it; the `switch` then
covers effects **7 to 59**. That is exactly the range of Gen 3's `MOVE_EFFECT_*` ids from confusion (7) to "Sp. Atk
-2 on the user" (59), with the major statuses 1-6 (sleep, poison, burn, freeze, paralysis, bad poison) handled on
their own path (**likely**: the range and the terrain table's Secret Power ids, 1 / 2 / 5 / 7 / 8 / 22-24 / 27, all fit
Gen 3's numbering; each case not checked one by one). The byte also carries Gen 3's two flag bits: **0x40 = the effect applies to the user**, **0x80
= certain** (no chance roll). Example (**verified** from the script bytes): the recoil effect's script (effect 48, used
by Struggle and Shadow Rush) is just "set the effect byte to 0xCE, go to the plain-hit script": 0xCE = 14 (Gen 3's
"recoil 1/4") | 0x40 | 0x80, exactly Gen 3's recoil script. So Shadow Rush's recoil is a quarter of the damage dealt
(**likely**, by Gen 3's meaning of effect 14; the recoil code itself not read). The scripts these effects run are a
second table, `lbl_80399F58` (39 entries; which value indexes it is not traced: it has fewer entries than there are effects).

### Accuracy (`fightCheckAccuracy`, Gen 3's command 0x01; **verified** from code)
1. Two special move values skip the roll (Gen 3's "current move" / "check Lock-On only" markers, 0xFFFF / 0xFFFE).
2. Protect-type moves, Magic-Coat-style reflection and the always-hit checks (`fn_80228DAC`: Lock-On, semi-invulnerable
   targets, ...) come first.
3. **Stage** = attacker's accuracy stage + 6 - defender's evasion stage (Foresight, condition 0x19, ignores evasion),
   clamped; the move's accuracy (field 6) is multiplied by the stage table's multiply / divide pair
   (`fightWazaHitKakurituDataBios...`). Thunder (effect 152) drops to 50 in sun.
4. Compound Eyes x1.3; Sand Veil in sandstorm x0.8; Hustle x0.8 for physical-type moves; BrightPowder
   x(100 - its value) / 100.
5. Miss if `random % 100 + 1 > accuracy`. In doubles, a spread move (target kinds 4 and 6) that misses one target
   records a partial miss.

### One-hit-KO moves (`fightCheckOneHitKO`, Gen 3's `tryKO` 0x93; **verified** from code)
- **Sturdy** (5) makes it fail.
- Under **Lock-On / Mind Reader** (condition 0x1D, which stores the locked target) it hits if the user's level is at
  least the target's. Otherwise it hits if `random % 100 + 1 < accuracy + (user's level - target's level)` **and** the
  user's level is at least the target's.
- On a hit the damage is the target's current HP; Endure (condition 0x2C) or a triggered Focus Band leave it at 1 HP
  instead, each with its own message.

### Type effectiveness and STAB (`fn_802274F0`, used by `WS_TYPE_CHECK` 0x06; **verified**)
- **Struggle (165) and Shadow Rush (356) skip it entirely**: no STAB, no weakness, no resistance, no immunity. So
  Shadow Rush always does neutral damage, whatever the target's types.
- **STAB:** x1.5 when the move's type matches either of the attacker's types (applied to the move's damage value,
  move status 0x2D).
- **Levitate** (ability 26) against Ground (type 4): no effect.
- Otherwise the type table (`zokuseiGetWazaJoutai(move type, defender type)`) is applied once per defender type (once
  if both types are the same). Foresight (condition 0x19) lets Normal and Fighting (types 0, 1) hit Ghost (type 7).
- **Wonder Guard** (25): anything not marked super effective is blocked, except a two-turn move on its charging turn
  (and Solar Beam in sun is treated as hitting).
- The type ids are Gen 3's (0 Normal, 1 Fighting, 4 Ground, 6 Bug, 7 Ghost, 9 the "???" type, 11 Water, ...;
  **verified** for the ids used in these checks and in the Net Ball rule).

### Damage prediction for the AI (`fightSeqGetNromalWazaDamage`)
The AI estimates a move's damage by **running the real damage steps** on saved copies: it saves both Pokémon (1,760
bytes each), the move's state, the sequence cursor and flags, runs the critical check (optional), the damage
calculation and the variance (optional) with hook callbacks before and after, reads the result if the move would
hit, then restores everything. So the AI's damage numbers always agree with the real formula. (**verified**,
`fight_trainer_ai_waza_damage.c`): Every AI caller turns **both optional steps off** (no critical-hit check, no
random 85-100% roll), so the AI always predicts the full, non-critical damage: the top of the real range. TuxSH's XD
write-up reports the same for XD (xd-insights.md (not yet published)).

### Colosseum's data against Gen 3's (**verified**)
Compared field by field with the public Emerald disassembly's data (pret/pokeemerald, read 2026-10-16) using the US
disc's tables:
- **Moves 1-354:** effect, power, type, accuracy, PP, secondary-effect chance and priority identical for all 354
  (see "How a move picks its script").
- **Species 1-251 and 277-411** (386 species, in Gen 3's internal order with the same 25 empty slots 252-276):
  base stats, both types, catch rate and base EXP identical for all; abilities and growth rate map one-to-one onto
  Gen 3's ids for 1-251 (e.g. Overgrow = 65, medium-slow growth = 3), so they are the same numbers too.
- **Items 1-225**: every item Colosseum names has the same id, name, price and
  held-item effect as in Emerald's item data (176 items; the only text difference is "GUARD SPEC." without its full
  stop; held-item effects map one-to-one, e.g. Quick Claw = 26). Colosseum leaves the **12 Mail items** (121-132)
  empty, and in the rest of the table has TM01-TM50 (ids after 225 not compared one by one) followed by empty slots where
  Gen 3 has the eight HMs: no Mail
  items and no HMs in Colosseum's item table (**verified** for the empty entries; the reasons are design choices, not
  traced).
- So the Pokémon side of Colosseum's battles uses Ruby / Sapphire's numbers unchanged; what Colosseum adds is Shadow
  data (Shadow table, catch rates, Shadow Rush) and its own rules (battle kinds, Hyper Mode, Call). Practical use for
  the port and for mods: Gen 3 references (move and species data, battle-script documentation) apply directly, and a
  Gen 3 data table can serve as an independent check of a decoded Colosseum table.
- Not compared: move target ranges and flags, species EV yields, held items, egg groups, learnsets, evolutions,
  gender ratio and friendship for 277-411.

## Disc data

Encounter, battle-kind, battle-type, effect, move and critical tables live in `common_rel` (common.fsys) and the
executable; their layouts are partly in `game-data-tables.md` (not yet published) and `pokemon-data.md` (not yet published). Move sequences (the `WS_*`
scripts) are data too (the move table points at them).

## Changing it

- New battle rules are new battle-kind records; most rule changes need no code.
- The damage formula is one function: a port option (for example XD's or later generations' mechanics) would hook it
  there; keep the original as the default for faithfulness.
- Move behaviour lives in move sequences (data); new moves can reuse existing `WS_*` commands.

## Efficiency

No allocation during a move: the sequence machine walks static scripts with a cursor and a few flag bytes; the AI's
damage prediction copies two Pokémon structs on the stack rather than allocating.

## Region notes

None found in this part (no text or layout code).

## Open questions

- The per-move multiplier (field 0x2C of the move state): STAB and type effectiveness (**likely**).
