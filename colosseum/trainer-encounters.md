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
- Two of those routes are now read. **Realgam Tower's Colosseum** passes its
  battle numbers as a helper function's **starting locals** (script-vm.md, "Calls and returns"): each pair of
  numbers (431 / 432 up to 441 / 442) goes with a story flag, and the helper starts the first battle while the flag is
  clear and the second once it is set (**verified** from the bytecode). **Mt. Battle's first valley** picks battles
  204-212 by a branch that stores the number in a script global, which a helper then returns for the battle call
  (**likely**). With these 16 traced, **73** story battles still have no number anywhere in the room scripts:
  the chain of Pyrite Town street battles (445-473) and the run 482-520, spread over five rooms in turn (Outskirt
  Stand, Pyrite's cave, The Under, Agate's shrine, Mt. Battle's foot), plus 550-555 in the Shadow Pokémon Lab and a
  few singles. The game
  code starts only a few battles by a fixed number: 521 and 561 for the Card e room (card-e.md) and Battle Mode's
  quick battles (522-524); battle 561 was one of the 73. The other **72** are named neither by a room script nor by a
  fixed number in the code. So either something computes their numbers from data not read yet (the repeating
  five-room pattern of 482-520 fits a trainer who moves between rooms), or they are unused leftovers; a **guess**
  either way. A runtime log of every battle the game starts during a full playthrough would settle it.

## Changing it
- To move a trainer to another room: change the battle number in that room's script call, or give the battle
  record a different arena (battle-arenas.md); the trainer record itself has no location.

## Open questions
- The table-driven battles per room (the Colosseum cups' `var[...]` tables are still unread), and what starts the
  73 battles no script names (search the game code for the battle-start call with a computed number).
- The story flag that gates each battle (needed for "when"): readable from the flag checks around each call.
