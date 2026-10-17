# Battle arenas (where every battle takes place)

Status: **partial** (2026-10-14). Every Colosseum and battle area, which room it is, and which
battles use it. Read from two tables in `common_rel` (game-data-tables.md (not yet published)):
the **battle-floor table** (`lbl_80478F4C`, 50 entries of 24 bytes, `fightFloorDataBiosGetPtr`)
and the **battle table** (`lbl_80478F54`, 566 entries of 56 bytes, `fightEncountDataBiosGetPtr`).
Layout and links are read from the USA data; field names are guesses unless marked.

## What it does

A battle record names its **arena** (an index into the battle-floor table). The arena names the
**room** the battle is drawn in, by room id, through the floor table. Overworld battles use the
room's **`_bf` archive**, a stripped copy of the room (map, collision, script and module, no NPCs)
loaded as the battle stage; the Colosseums use their own `*_colo` rooms.

This also settles what `_bf` means in archive names: the battle-field version of a room.

## Data structures

**Battle-floor entry (24 bytes):**

| Offset | Field | Evidence |
|---|---|---|
| +0x00 | u8 **terrain (tikei) id** (0-8) | `fightFloorDataBiosGetTikeiDataId` (verified) |
| +0x02 | u16 room id | matches the floor table's room id (floor entry +0x0C) for every entry |
| +0x04 | u32 message id | arena **name** (`fightFloorDataBiosGetName`, verified) |
| +0x08 | u32 **introduction sequence (shōkai) WZX id** | `fightFloorDataBiosGetSyoukaiWzxDataId` (verified). Set only for the Colosseums and arena 6 (0x89-0x8F), i.e. the stages with an intro scene |
| +0x0C | u32 | **music sound id** for battles on this arena (0 = none; e.g. 24 = `battle2_song`, 981 = `battle6_song`); `fightFloorDataBiosGetBgmSndId`, music.md (not yet published) |
| +0x10 | u32 **ambient (environment) sound id** | `fightFloorDataBiosGetEnvSndId` (verified). One per Colosseum (0x3D6-0x3DB = crowd sounds, guess) |
| +0x14 | 2 x u16 **battle-side data ids** (side 0, side 1) | `fightFloorDataBiosGetFightSideDataId(arena, side)` (verified); always 2 / 1, into the 5-entry battle-side table (`lbl_80478F3C`, 20 bytes each; contents open) |

**Battle entry (56 bytes)** (**verified** from the typed record `FightEncountData` in `fight.c` and its accessors): +0x00
u8 **battle kind** (the 19 kinds and their rules are named in battle-engine.md, "The 19 battle kinds"), +0x01 u8 "trainer" flag, +0x02 u8 whole-team-fainted (*zenmetu*) flag, +0x04 u16 arena,
+0x08 u32 battle name message, +0x0C u32 music, +0x10 u32 wipe
(screen transition), +0x14 u32 intro WZX, +0x18 four trainer slots of 8 bytes: u16 trainer id and u32 input device
(player pad, CPU, ...).

## The arenas

There are **50 arena records** (*verified*, the battle-floor table). Each names a room, a terrain id and a name
message; most also name a battle song, and the Colosseum arenas an introduction WZX and an ambient sound. Overworld
battles use the `_bf` battle copy of the room they happen in; the Colosseum rooms appear in several records (see
"Findings"). The per-arena values are disc data and are not listed here.

## Flow

Battle start: the battle record gives the arena; the arena gives the room id; the floor table
gives that room's archive (`floorDataBiosGetPtr`, archive id at floor entry +0x04), which is
loaded as the stage. Colosseum battles add the per-Colosseum values (+0x08, +0x10).

## Findings

- **The Colosseums:** M1_water_colo (Phenac, arenas 33 and 43), M2_earth_colo (Pyrite, 0-4, 6, 34,
  44), M4_cylinder_colo (35, 45), M4_bottom_colo (36), D2_crater_colo (Mt. Battle, 37 and 46),
  D4_casino_colo (Realgam Tower, 38 and 47), T1_ancient_colo (48).
- **Arenas 40-48 have no story battles.** They repeat the Colosseums (and Mt. Battle's three
  stages) with new name messages, plus T1. Guess: the stages offered by the menu battle modes
  (Colosseum battle / link battle), which build battles at run time instead of using the table.
  This fits T1: a stand-alone Colosseum used only as a selectable stage, which is why it sits in no
  town (`fsys-catalog.md`, T1).
- **Mt. Battle** (arenas 24-26: valley, magma, cloud) holds 99 battles of type 8, plus 9 others:
  the 100-trainer challenge (**verified**: battle kind 8 is the Mt. Battle challenge, battle-engine.md).
- Pyrite's Colosseum (arenas 0-4) carries the early tutorial and special battle types.

## What an arena consists of (for custom arenas; 2026-10-14)

- **How the original picks the arena (verified):** a **table**, not per trainer or per location: each battle record
  (`fightEncountDataBiosGetPtr`) holds an arena index (+0x04), and the arena record names the room. The script that
  starts the battle only passes the battle id (`fightStartEncount`), so the same trainer can fight on different arenas
  in different battles.
- **Model and collision:** the room's `_bf` archive (overworld battles) or `*_colo` room (Colosseums): map model,
  collision, script and room module (`fsys-catalog.md`, `floor-resources.md` (not yet published)).
- **Introduction scene:** an optional WZX sequence (+0x08), for the Colosseums.
- **Sound:** battle music (+0x0C, unless the battle or trainer class overrides it: `music.md` (not yet published)) and ambient sound (+0x10).
- **Terrain id** (+0x00): selects terrain-dependent move effects. Traced, see
  "Terrain" below (Secret Power, Nature Power, Camouflage; **verified**).
- **Battle-side records (decoded 2026-10-14 night):** the table (`lbl_80478F3C`, 5 x 20 bytes; common_rel section-4
  offset 0x8F910, count at 0x14A688) holds, per side: +0x00 u16 **Y rotation** (`fightSideDataBiosGetYrot`; 0 or 1
  in the data: guess, which way the side faces, 1 = turned around), +0x04 / +0x08 the two **trainer status-window**
  menu ids (`GetFightTrainerStatusMenuDataId`: 243 / 244 or 241 / 242), +0x0C / +0x10 the two **Pokémon status-window**
  menu ids (`GetFightoutPokemonStatusMenuDataId`: 70 / 69 or 71 / 72). Entry 0 is empty; entries 1 / 3 and 2 / 4 are
  identical pairs. So the side records set **facing and the HP / status windows, not positions**: arenas all use
  side ids 2 / 1, i.e. one side turned around with the right-hand windows, the other with the left-hand ones.
- **Where trainers and Pokémon stand:** the battle grid ("Who stands where, and the idle camera" below). Battle
  lighting: open.
- **For modders:** a custom arena = a battle-floor record (append) + a room with a `_bf` archive (map, collision,
  module) + optional intro WZX and sounds; battles point at it by index.

## Terrain (*tikei*; **verified** from code and the executable's data)

The terrain table (`lbl_80363C00`, 12-byte entries, ids 0-8 plus an end entry; bound `lbl_80478BA0` = 9) gives each
terrain three things, read by `tikeiDataBiosGet*`:
- a **type** (+0x00, `GetZokuseiDataId`): what **Camouflage** turns the user into;
- a **secondary effect** (+0x02, `GetFightKoukaId`): what **Secret Power** may inflict (Gen 3's move-effect ids:
  1 sleep, 2 poison, 5 paralysis, 7 confusion, 8 flinch, 0x16-0x18 Attack / Defense / Speed down, 0x1B accuracy
  down: **likely** names, by Gen 3's numbering);
- a **move** (+0x04, `GetWazaId`): what **Nature Power** becomes;
- and a name message (+0x08; developer names, Japanese).

| Id | Name (developer label) | Type | Secret Power | Nature Power |
|---|---|---|---|---|
| 0 | grass | Grass | poison | Stun Spore |
| 1 | long grass | Grass | sleep | Razor Leaf |
| 2 | sand | Ground | accuracy down | Earthquake |
| 3 | water | Water | Defense down | Hydro Pump |
| 4 | sea | Water | Attack down | Surf |
| 5 | pond | Water | Speed down | BubbleBeam |
| 6 | rock | Rock | confusion | Rock Slide |
| 7 | cave | Rock | flinch | Shadow Ball |
| 8 | room | Normal | paralysis | Swift |

These nine are Gen 3's battle terrains in Gen 3's order (Ruby / Sapphire's tall grass, long grass, sand, underwater,
water, pond, mountain, cave, building: **likely**, by the matching Nature Power moves). The battle floor's +0x00 picks
one (`fightFloorGetStatus(.., 0x0F, ..)` reads it; the AI's Secret Power scoring switches on the same effect id).
For the Dive Ball rule in battle-snag.md: floor terrain 3, 4 and 5 are the three water terrains, so the Dive Ball
works on water arenas (**verified** now: the ball reads this same field).

## Changing it

- A new stage: add a battle-floor entry pointing at a room id that has a `_bf` (or `_colo`)
  archive, then point battles at its index. Append, so existing indices stay valid.
- Moving a battle to another arena is a one-field change (+0x04 in the battle entry).

## Battle rooms (**verified**)
An arena record's room id (+0x02) names the room the battle is fought in, and `fightStartEncount` **pushes that room
on the room stack** (menu-rooms.md). Map battles use a "battle field" copy of the map they happen on, named like the
map with `_bf` (Outskirt Stand: arena 5 -> room 0xC8 `S1_out_bf`; a Phenac City house: arena 8 -> room 0xCC
`M1_houseA_1F_bf`). 34 such battle rooms on the US disc each carry a small script named `fight_common_scd` (its own
copy per room: map-object animations and door / elevator controls).

## Who stands where, and the idle camera (**verified** by running the game's own code natively, with frame dumps)
- **The battle grid** places the participants: `battleGridAddTrainer(record, side, depth)` and
  `battleGridAddPokemon(trainer record, Pokémon record)` add them, `battleGridUpdate` positions them. Each grid member
  is a 0x8C-byte battle sequence record (its facing byte at +0x76; moving the record moves its model). Trainer battle
  models are PKX files like the Pokémon's.
- **A single battle's layout**: trainers at x = ±65 and their Pokémon at ±40, the two sides turned to face each other
  (-90° / +90°). The spacing is scaled by a size factor taken from the **largest Pokémon size class** on the field
  (`battleGridGetMaxPokemonField` reads each record's class word at +0x10), so big Pokémon push everyone apart.
- **The idle camera**: while the battle waits for commands, `battle_main` calls `battleCameraStartRandom` every frame;
  about every 200 frames it picks a random grid member (never the same one twice in a row) and starts a camera move
  on it (`fn_801D2C74`): a close-up of one Pokémon, then the trainer behind it, then another Pokémon with its trainer.
- **Moves**: each move's effect-type function (`fight_waza.c`) loads and plays the move animation's halves in order,
  and the camera cuts to the target for the damage half (wzx.md for the animation files).

## Open questions

- Which code reads the arena for the menu battle modes, to confirm that arenas 40-48 are their
  stages and which Colosseum T1 is.
