# Where and when trainers are met

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). How to find, for any trainer, the room and story point where the
player fights them, and how far the US data answers that. A full trainer-by-room list would be generated data, so
this page records the method and the counts. **Verified** counts from the decoded
US scripts and the battle table.

## What it does

A trainer has no location of its own. Where and when the player meets one follows from a chain: the trainer is named
in a battle record, the battle's arena is the room's battle copy, and a room script starts the battle at a story
point. This page follows that chain and counts how much of the game it settles.

## Data structures

No structure of its own: the trainer record (trainers.md (not yet published)), the battle record (battle-engine.md "The battle entry"),
the arena record (battle-arenas.md) and the room script's call to start the battle (script-vm.md) are the links.

## Flow: the chain
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
  table-driven ones above, plus records the game never starts.
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
  five-room pattern of 482-520 fits a trainer who moves between rooms), or they are unused leftovers; neither is
  settled. A runtime log of every battle the game starts during a full playthrough would settle it.

### The 72, by family 

**What they are not** (**verified** from the battle table): every one of the 72 is an **ordinary story-trainer battle
(kind 2) on a field arena** (a town, cave or building). The Colosseum challenge rounds use kinds 3, 5, 6 and 7 on the
four Colosseum arenas, the Deep Colosseum (The Under's lower Colosseum) included; Mt. Battle uses kinds 8 and 9;
Battle Mode uses kinds 12 and up (colosseum-challenges.md; battle-engine.md). So none of them is a challenge round, a
Deep Colosseum battle, a Mt. Battle opponent or a Battle Mode opponent. They split into three families:

1. **Placeholders (64 battles, 445-520).** Every trainer in this family has the **same stand-in team, two level-24
   Zigzagoon** (**verified**), and most of their names were **never translated** (still Japanese in the US data).
   Among them: six Riders named as numbers in six languages (*eins*, *dos*, *trois*, *quatro*, *five*, *six*), each
   placed in five arenas in turn (Agate, Mt. Battle's foot, Outskirt Stand, Pyrite's cave, The Under), and one Chaser
   whose name means "phantom", placed in eight arenas across the whole region. This is the **unfinished postgame**
   that The Cutting Room Floor describes (unused-content.md): trainers meant to turn up around the region after the
   story, never given teams and never started (**likely**: no script names them, and their teams are placeholders).
   So the family does match "postgame rematches", but as **unfinished data**, not as working rematches of trainers met
   in the story: no finished trainer shares their names.
2. **"Shadow already snagged" versions of the Lab's first meetings (6 battles, 550-555).** The six Shadow Pokémon Lab
   ambushers (see "Ambushes" below) each have a first-meeting battle, two later versions (Shadow Pokémon still held, or
   replaced once snagged), and **one more: the first-meeting team without its Shadow Pokémon** (**verified** from the
   teams). That version is for a player who snagged the Shadow Pokémon and then lost: Pyrite's grunt in the first
   building has exactly this branch, picking the team without the Shadow Pokémon when its "snagged" flag is set
   (**verified** from that room's script). The Lab's ambush scripts **never make that check** for the first meeting, so
   these six are never started.
3. **A mid-story pair for that Pyrite grunt (2 battles, 102 and 103).** The same grunt has a first meeting, its
   "snagged" version, a late pair used by the Snagem hideout ambush, and **a middle pair** (Shadow Pokémon held / replaced,
   with teams in between) set in the same Pyrite building; no script starts the middle pair (**likely**
   unused).

So the "runtime log" check above now matters mainly for confirming the obvious: family 1 is placeholder data, and
families 2 and 3 are versions the scripts never select.

## Ambushes: trainers who jump out 

The Cipher (and Snagem) grunts who drop in front of the player before a fight are **ordinary room-script events**,
not a separate system. **Verified** from the decoded US room scripts unless marked.

**How one is staged** (the same sequence everywhere):
1. **Hidden at room load:** the room's setup function makes the ambushers invisible.
2. **Trigger zone:** the room's main loop checks every frame whether the player is inside a rectangle on the floor
   (a bounds check on the player's x / z position) **and** the ambusher's own "done" flag is clear. If so, it calls
   that ambusher's event function. One room can hold several zones, one per ambusher.
3. **The event:** player control is frozen; the ambusher is made visible; it plays its **entrance animation** and the
   script waits for it to finish (animation **53** for the drop-ins; the motion is **likely** the jump-down players see,
   not checked frame by frame); the script waits for its movement to settle and **sets the "done" flag**, so the
   ambush fires only once; the ambusher speaks; the battle starts with the field battle call (battle-engine.md,
   "Battle start").
4. **After a win** (battle result 2) the ambusher is hidden again and control returns. What follows a loss is the game's
   general loss handling (**likely**; not traced here).
- **Camera:** none of the drop-in events moves the camera; they play in the room's normal camera (no camera call in
  any of them). Cutscene-style fights (Agate's shrine scene, a scene on Realgam Tower's first floor) are different events with
  scripted camera moves.
- **Music:** the event starts no song itself; the battle music comes from the battle's own music lookup (music.md (not yet published)).
- **Which battle:** usually chosen by story state. In the Shadow Pokémon Lab each ambusher has a first-visit battle
  and two later versions, picked by a story flag and a second per-ambusher flag. In Realgam Tower and the Snagem
  hideout one flag picks between two versions.

**Where they are:**

| Area | Rooms | Ambushes | How triggered |
|---|---|---|---|
| Shadow Pokémon Lab | B1 (3), B2 (1), B3 (2) | 6 drop-ins (animation 53) | a trigger zone each |
| Realgam Tower | one dome room | 1 drop-in (animation 53) | a trigger zone |
| Team Snagem hideout (revisit) | the first-floor hall | 6 drop-ins (animation 53); one routine picks which of the six appears from the story state (the choice rule is not read yet) | one shared routine |
| Agate Village, the Relic Cave | first cave room | 1 ambusher who jumps out with a different entrance (animation **18**) and walks off after the fight | a trigger zone |

**Entering the room inside a zone** (**likely**): the zones are checked from the room's main loop,
so a player who comes into a hideout room already standing inside a zone (not walking into it) meets the next
ambusher at once, before taking a step. On a GameCube that can't really happen (rooms are entered at their doors);
a port or mod that lets the player save anywhere can do it: a save made right where an ambush was fought, continued
later, starts another of the hideout's drop-ins straight away (seen four times in a replay of a recorded play session,
each a battle the original session never had there). 

**They don't start any of the untraced battles.** Every ambush names its battle numbers directly in the script; those
numbers are all already traced, and none is among the 72 above. The Lab ambushers do own six of them (family 2 above),
as versions their scripts never pick.

## Changing it
- Ambushes are data-like already: the zone rectangle, the entrance animation, the battle number and the flags are
  plain arguments in the room script, so moving, re-arming or retargeting one is a script edit (no code). The camera
  and music would need new script calls, since the event uses the defaults.
- To move a trainer to another room: change the battle number in that room's script call, or give the battle
  record a different arena (battle-arenas.md); the trainer record itself has no location.

### Mod surface
- **Data it reads:** battle records (trainers, arena), room scripts that start battles (the battle number, trigger
  zones, flags), the ambush events.
- **Hooks it would need:** none for moving or re-arming encounters (script edits); a battle-start hook for new lead-ins
  and intros.
- **Fixed-size limits:** the battle table's count; the trigger checks per frame in a room.
- **Must stay vanilla in GameCube-accurate mode:** every encounter's place, trigger and story timing.

## Disc data

The battle and trainer tables (`common_rel`), the arena table, and each room's script (its `fsys` archive). The chain
above is read from those; nothing extra is stored.

## Efficiency

Not applicable: nothing here runs per frame beyond the room scripts' own zone checks (player-movement.md).

## Open questions
- The table-driven battles per room (the Colosseum cups' `var[...]` tables are still unread), and what starts the
  73 battles no script names (search the game code for the battle-start call with a computed number).
- The story flag that gates each battle (needed for "when"): readable from the flag checks around each call.
- What animations 53 and 18 look like on the grunts' models (**likely** the drop-in and the jump-out players see; a
  visual check); how the Snagem hideout routine picks which of its six ambushers appears.
