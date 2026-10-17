# Colosseum challenges (story mode)

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
   battle `fieldStartEncount(set[counter - 1], 1, 1)` (the bytecode computes the index from the counter; counter - 1 is
   **likely**), and **stop the run on a loss** (result 3: the
   function returns 3). So the three ordinary opponents come first and the **boss last**.
4. After the fourth win, story flags are set (the cup is cleared, the next becomes available) and the script returns
   2 (won).

The other entrance scripts follow the same pattern with their own tables (Phenac: four calls indexed by one
variable; Pyrite: five; The Under's upper Colosseum: four), so each Colosseum can offer several cups chosen by the
story.

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
The same rooms serve the story's climb and Battle Mode's Mt. Battle (`BATTLEMODE_BATTLEYAMA100`, new-game-flow.md),
told apart by variable 0x8AE (the game mode: 0 = story, 1 / 2 = a Battle Mode run, **likely** single / double battles; pause-menu.md). Which trainers stand in which area,
and how the 100 are distributed, is in the area tables (not read yet; **likely** ten per area).

## Realgam Tower's Colosseum (first pass)
`D4_casino_colo` (the story's last Colosseum) runs its rounds differently (**verified** structure): its main loop
steps through story flags 1139-1143 to pick the next scene, and each round's battle goes through a helper (function
16, called from **seven** places) that starts **one of two battle ids, chosen by a story flag**: two versions of the
same opponent. Another function counts how many of **40 consecutive story flags (1262-1301)** are set (**likely**
the purified flags of Shadow Pokémon 0-39, shadow-system.md; **guess**: a battle's version could depend on whether
its Shadow Pokémon was already purified). Not read further yet.

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
  opens a further confirmation menu (0x106) instead.
For a mod: kind 0's table has 8 entries and is indexed without a bounds check, so a longer custom cup would read
past it; the table must grow with the cup.

## Rules come from the battle kinds
- **Kinds 3 / 5** (story Colosseum) and **6 / 7** (Phenac's): EXP is given, **money is pooled** and paid out at the
  end of the run, Hyper Mode and the partner's Shadow detection work as in normal story battles
  (battle-engine.md, the battle-kind table; battle-rewards.md). The boss kinds (5 / 7) mark the last round.
- The arena record gives the stage, the crowd and its music (battle-arenas.md).

## Changing it
- **New cups or new opponents**: script data only. Change the battle ids the entrance script writes into its
  globals, or add a set and a story-flag condition that picks it.
- **Longer or shorter runs**: the counter (4) and the loop in the entrance script.
- **Rules** (pooled money, boss handling): the battle kind of each battle, in the battle table.

## Open questions
- Which story flags pick each cup in each Colosseum, and what clearing a cup unlocks.
- Pyrite's fifth call (its first round is started with a different fade option).
- How the pooled money is paid out at the end of a run (battle-rewards.md).
- Realgam Tower's Colosseum: what picks each round's version, and what the 40 counted flags are.
- Mt. Battle's per-area trainer tables (who stands where).
