# Where and when trainers are met

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). How to find, for any trainer, the room and story point where the
player fights them, and how far the US data answers that. A full trainer-by-room list would be generated data, so
this page records the method and the counts. **Verified** counts from the decoded
US scripts and the battle table.

## The chain
1. **Trainer -> battle**: a battle record (battle table) names its trainers per side (battle-engine.md, "The battle
   entry"; trainers.md (not yet published)). A trainer can appear in several battles (rematches, Colosseum cups).
2. **Battle -> arena -> room**: the battle record's arena names the stage; for map battles that is the `_bf` copy of
   the room where the fight happens (battle-arenas.md). That says *where*, but not *what starts it*.
3. **Battle -> script**: a room script starts the battle with `fieldStartEncount(battle, ...)` (or
   `fightStartEncount`). The function it sits in says **how** it starts (a talk script, the room's main-loop check,
   a cutscene), and the flags around it say **when** in the story (flags.md (not yet published), events.md).

## How much the scripts settle
- 364 story battles (kinds 1 and 2: bosses and ordinary trainers). **271** are started by a call with the battle
  number written out, from **53** room scripts; only one battle is started from more than one room.
- The Colosseum cups and Mt. Battle start theirs through **tables in script globals** (`fieldStartEncount(var[...])`):
  Phenac's stadium, Pyrite's and The Under's Colosseum entrances (colosseum-challenges.md) and the Mt. Battle areas
  (`D2_*`), plus Realgam Tower's Colosseum (`D4_casino_colo`). Those need the table contents read per room.
- **93** story battles have no literal call. Grouped by arena, the largest groups are Realgam Tower's Colosseum (12),
  Pyrite's entrance and cave (11 each), Outskirt Stand (9), Mt. Battle (15) and The Under (6): **likely** the
  table-driven ones above, plus records the game never starts (leftovers or alternate versions; **guess** until the
  tables are read).

## Changing it
- To move a trainer to another room: change the battle number in that room's script call, or give the battle
  record a different arena (battle-arenas.md); the trainer record itself has no location.

## Open questions
- The table-driven battles per room (Mt. Battle, Realgam Tower, the Colosseum cups), to close the 93.
- The story flag that gates each battle (needed for "when"): readable from the flag checks around each call.
