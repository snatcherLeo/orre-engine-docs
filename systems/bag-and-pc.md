# The bag, Rui's pocket and the PC

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-17). How the player's items and stored Pokémon are kept: the bag's pockets
and their limits, a cut **Rui's pocket**, and the PC (three Pokémon boxes plus item storage). The save layout itself is
on save-data.md (sections 2 and 3); this page covers the rules. Code: `src/game/hero_savedata_exact_80129280.c`
(`heroItem*`), `src/game/pcbox.c`, `src/game/gs_pokemon_summary.c` (the bag screen), `src/game/gs_event_exec.c`.
**Verified** from the decomp's C and the US text unless marked.

## What it does

The bag holds the player's items in one pocket per item kind, each pocket a list of `{item id, count}` slots with a
size and a count limit; the PC stores Pokémon in three boxes and items in its own storage. Picking up, buying, using
and giving items all go through the same add / remove functions, which enforce the limits.

## Data structures

- **Bag slots:** `{u16 item id, u16 count}`, grouped per pocket; pocket sizes and caps come from one switch
  (`heroItemGetItemKindToItemAryPtr`), below.
- **PC:** three Pokémon boxes of 30 records, and 235 item slots; the save layout is on save-data.md (sections 2, 3).

## Flow

Getting an item (a pickup, a shop, a gift script) calls the add function, which finds the pocket by the item's kind,
merges into an existing slot or takes a free one, and reports what didn't fit ("Adding an item"); using one runs its
field-use function and the party screen ("Using an item from the bag"); the PC moves records between the party, boxes
and storage.

## The bag: one pocket per item kind
Every item's record has a **kind** (item table, pokemon-data.md). The bag picks the pocket from the kind
(`heroItemGetItemKindToItemAryPtr`); every add, remove, "has it" and "room for it" check goes through that one switch.

| Item kind | Pocket | Slots | Max per slot |
|---|---|---|---|
| 1 | Poké Balls | 16 | 99 |
| 2 | Items | 20 | 99 |
| 3 | Berries | 46 | **999** |
| 4 | TMs | 64 | 99 |
| 5 | Key items | 43 | 99 |
| 6 | Colognes (the three scents, opened through the Cologne Case) | 3 | 99 |

Each slot is `{item id, count}`. Two more per-pocket switches go with each pocket (meaning not read). A pocket being
full is the "no room" case (message 0x4263 "There's no room for that item!").

**The bag screen** (`gs_pokemon_summary.c`, page table `lbl_80266918`, 6 pages of 0x4C bytes) shows pages 1-5:
**ITEMS, BALLS, TMs, BERRIES, KEY ITEMS** (L / R turn the page and wrap between 1 and 5). Each page names its pocket
kind, its draw function, title message and colours. The colognes are not a page; they are reached through the Cologne
Case.

## Rui's pocket (cut; **verified** code and text, cut status **likely**)
Page **0** of the bag table has pocket kind -1, which selects a separate list: **10 slots, one of each item**, stored
in the player's save record next to a second name field ("hizuki" in the code is **Rui's internal name**: her models
are `hizuki_a1` / `hizuki_b1`, her battle effects `DoHizukiAi` / `DoHizukiMiyaburi`). The page title is the partner's name. Two event functions move items:
- **give to Rui** (`fn_800138B4`): if her pocket has room, the item goes in and she agrees to use it
  (a Japanese line, 0x4264); if not, she says she can't hold any more and asks the player to take one back (0x425F);
- **take back** (`itemStorageWithdraw` (0x8001374C)): the item goes into the player's bag (or "no room"), is removed from her pocket, and
  a "received from Rui" line follows (0x4268).
All these lines are **still in Japanese** in the US build, and the page turning never reaches page 0, so the feature
is unreachable in the US game (unused-content.md). What "leave it to me" was for is not known.

## The PC (`pcbox.c`)
- **Pokémon**: **3 boxes of 30**, named "BOX 1" to "BOX 3" by default (message 0x32C9 with the box number); a name
  may be up to 8 characters (`pcboxSetPokemonBoxName`). Storing a Pokémon (`pcboxAddPokemon`, or a chosen slot)
  copies its record into the first empty slot of the given box, or of any box, and **fully heals it**
  (`pokemonAllKaihuku`). The party menu's "box is full" checks count empty slots per box.
- **Items**: one list of **235 slots, up to 999 each**, not split by kind. The coupon exchanges deliver their prizes
  here (`shopCouponBuyItem` (0x80029EF4): coupon currency -> PC item storage) (**likely**, from the shop code; shops.md), and so does
  any treasure that doesn't fit in the bag (**verified**, `floorEventGetTresure`, treasure.md).
- **Names swapped** (**likely**): `pcboxAddItem` calls the slot-removal helper (`fn_80140ACC`, the one
  `heroItemDecItemDataId` uses) and refuses when the item isn't stored, while `pcboxDelItem` calls the slot-adding
  helper (`fn_80141308`, used by `heroItemAddItemDataId`) and works on items not yet stored. So the two names are
  the wrong way round. Evidence for which helper adds: the bag returns "the amount that didn't
  fit" from `fn_80141308` ("no room" when it's above 0), and treasure passes exactly that remainder to `pcboxDelItem`
  to store it in the PC.

### The PC's menu (**verified** from the code and a Dolphin replay)
- Using a PC runs script 0x0596000D, which pushes the menu room 0x391 `pcbox_menu`; its driver is
  `std024_dialogDisplayPCBoxMenu`. It opens a set of menu windows and asks for a choice with menu 0x1D (four
  choices: Pokémon Storage, Item Storage, Save, Exit), then runs the box screen (`pcBoxStorageMain`), the item
  storage (`itemStorageRun`) or the save (`pcMenuSave`), and returns to the choice.
- Menu 0x1D uses **cursor mode 4**: the menu table entry names its own cursor function (`fn_80038380`), which moves
  the choice up or down by one and clamps it to the item count. The four choices are drawn staircased to the right,
  so the menu system's standard cursor (move to the nearest item straight above or below) could not reach them.
  Values from the real game during a replay: cursor count 4, mode 4, cursor function 0x80038380, the cursor offset
  0, 1, 2 as the player moves down.
- The setup copies two four-entry layout tables (0x18 bytes each: positions as floats) into a work area and then
  past its end into the next two (`lbl_803A6498` + 0x54 and + 0xB4 are `lbl_803A64EC` and `lbl_803A654C`), and
  reads the icon wheel's start angle from the table after the layout table (`lbl_802E51C8` + 0xE0 =
  `lbl_802E52A8[0]`, the four wheel angles 0, pi/2, pi, 3pi/2). The code relies on these tables sitting next to each
  other in memory.
- **The storage screen's windows** (**likely**, read from the real game's window list during a
  replay of a recorded play session that deposits three Pokémon; the roles come from when each window opens, not from the code):
  the screen opens windows 0x10E, 0x8F-0x93 and 0xA0 together; **0x93 is the box grid** (32 cursor positions: the 30
  slots plus two more, likely the box-name bar's arrows), **0x94 the party list** (7 positions: six slots and one
  more, likely "back"), **0x9C the action list** for the chosen Pokémon (six choices, opened with its frame 0x9B),
  **0xA3** the box choice of a deposit (opened after the action list's third entry; its own cursor function, cursor
  mode 4, moving over three positions, one per box; the deposit happens as it closes), and **0xA7** a two-choice window (opened with 0x9B on leaving the screen, likely "quit storage?"). The box grid and
  the party list keep separate cursors; which one the player is on decides whether an action moves a Pokémon into or
  out of the party.
- **The box grid's cursor** (**verified** from the byte-matched code, `fn_80054C44`, and a replay): a table of 32
  entries (`lbl_80267398`, 12 bytes each: a message id, a kind, a slot number) gives each cursor position a kind:
  0 for the 30 slots, 1 for position 30, 2 for position 31 (the two ends of the box bar, **likely**). The grid uses
  the standard nearest-item movement, with one rule on top: a move from a slot onto position 31 is turned into
  position 30, so leaving the grid upward always lands on the same end of the bar. The window's cursor index sits at
  +0x95 and its previous value at +0x97.
- **The box loop** (**verified** from the byte-matched code, `pcBoxRun`): the loop opens the grid
  (0x93) and acts on the chosen position's kind: a slot (kind 0) opens the action list for that Pokémon; position 30
  (kind 1) asks a yes / no question (message 0x1B87 in window 0x9B, the choice in window 0x9E) and on "yes" fades out,
  rebuilds the screen and fades back in (what the rebuild changes is not read yet); position 31 (kind 2) runs a
  similar change with no question (**likely**). (**likely**, from the
  console's frames of a recorded play session): The question at position 30 is the **box menu** ("Do what with this BOX?",
  Name / Cancel), opened with the hand on the box's name bar. The storage screen also has a **quick-move mode** (the
  player's account: the hand, normally white, turns yellow, and A on a Pokémon then picks it up at once instead of
  opening the action list 0x9C); which press switches it is not identified yet.
- **The storage cursor (the orange brackets and the hand)** (**verified** from the code and a
  replay's log): it isn't the menu system's own cursor sprite (the grid's items name none); the box screen draws it as a
  window sprite with an update callback (`fn_80055EB8`) that shows it while the box screen's frame window (0xA0) is open
  and the grid cursor is on a slot, and places it at the **hand's position**. That position (two floats in the screen's
  state block `lbl_803A9768`, +0x27C / +0x280) glides toward the chosen item's x / y from the menu item table: each move
  sets a start and a target (`fn_80057830`) and a step of 1/6, so the hand always takes six updates to reach a slot,
  however far it is. On slot 0 of box 1 the target is (246, 95). (**likely**, from a recorded play session's window
  list on the real game): A on position 31 opens the **party list** (0x94) about 30 frames later, with the grid left
  open behind it, and A on position 30 opens the question (0x9B with the choice window 0x9E) as above. In that
  session a held Right on the party list moved its cursor (0 to 2) on the press, and its first key repeat closed the
  party list, with the grid cursor back on slot 0: Right leaves the party list for the grid. A hold one frame
  shorter stays in the party list, so the key-repeat timing decides where the next presses land. Likewise
  (**likely**, from frame logs of two recorded box visits): For its first few frames after opening, the grid doesn't
  move its cursor on a key repeat (a repeat one frame after it opened did nothing, one six frames after moved it), so
  a held direction carried into a new grid opening moves it once less or once more depending on a frame. B on the grid asks whether to quit (message 0x1B8B and the two-choice
  window 0xA7), or first shows message 0x1B7E when the screen is in one particular state (**likely** a Pokémon still
  being moved). L and R (held, not just pressed) step through the three boxes, showing one of windows 0x90-0x92 for
  the new box; on position 30, left and right do the same.
- **B on that yes / no question** (**verified** from the assembly and a Dolphin recording): the helper that reads the
  choice (`fn_8005D26C`) checks the result with "below -1 or 2 and up", so B's -1 slips through and the helper returns
  the word just before its two-entry answer table on the stack. That word is the return address the helper's last call
  left there, a code address, and the box loop only acts on 0 ("yes"), so on the console B simply means "no". It works
  by luck, not by design: a different stack layout makes B a "yes".
## Adding an item (**verified** from code and a native run)

`heroItemAddItemDataId(hero, item, amount, index)` (what the script native `scriptAddItem` calls) finds the bag pocket
from the item's **kind** (`itemGetStatus(0, item, 2, 0)`: e.g. 2 for a Potion's pocket, 1 for Poké Balls') through
`heroItemGetItemKindToItemAryPtr` (pocket slot count, the per-item maximum 99, sort flags), then inserts with
`fn_80141308`. **It returns how many did not fit** (0 = everything added), or -1 when there is no pocket or the item id
is invalid (an id is valid when it has a name and is below the item count, 548 in the USA game). So a caller checks
"`>= 0` and less than the amount", not "non-zero". New items go in a free slot and the pocket is re-sorted; an item
already present is topped up to the maximum first.

## Item count limits (**verified** from the code)
- Every bag and PC slot is `{u16 item id, u16 count}`, so the saved count could hold up to 65,535.
- The bag's limits come from the pocket switch above: 99 per slot, except the Berries pocket (999).
- The PC item storage (235 slots) allows **999** per item (`pcbox.c`).
- The shop's quantity selector lets the player pick at most **99** at once (`menuShop.c`, three places); its room
  check counts up to the destination's own limit (the bag's, or 999 for the PC).

## Using an item from the bag (**verified** from the code unless marked)

1. Choosing an item calls **the item's own use function**, a function pointer stored in the item record: the field-use
   one outside battle, the battle-use one in battle (`itemDataBiosGetFieldUseFunc` / `...BattleUseFunc`). An item
   without one shows a "can't use this here" message.
   (verified from the US item table) The 279 named items share **16 use functions**. One shared
   **medicine** function serves every potion, status cure, revive, drink, herb, Full Restore, Max Elixir, the vitamins
   (field only) and the bag use of the curing berries; a small effect number in each record picks what it does. The
   others each serve one kind: Poké Balls (battle only), PP restore (Ether, Max Ether, Elixir, Leppa), PP Up / PP Max,
   Rare Candy, Sacred Ash, the X items with Guard Spec and Dire Hit (battle only), evolution stones, TMs, key items,
   the Time Flute, the Ein Files (reading), the scents, and three short ones that only print a message (**likely**,
   from their size): "not now" (held items, valuables), "can't be used" (repels, Escape Rope, shards, scarves, the
   Pokéblock berries) and a third used only by the Cologne Case. Held effects are a separate number (66 ids) read by the battle code. So for testing, one item
   per function (and per medicine branch) covers the code; the rest differ only in table values.
2. For items used **on a Pokémon** (potions, Rare Candy, vitamins), the bag menu (menu 0x59) is **closed** and the party
   screen opens in **"use item" mode** (`menuPokemonOpenItemUse`, mode 3 in the field, 4 in battle).
3. When the use ends, the use function's result (0, 1 or 2) becomes the bag's result code (0, 3 or 4), the item count
   drops (`heroItemDecItemDataId`), and the bag **reopens**. So using a second potion means choosing it again from the
   bag; the party screen does not stay open between uses (which result code means "used" versus "cancelled": **likely**
   2 = used, from the decrement that follows it).

**Rare Candy in the field** (**verified** from the code; the window ids from a Dolphin recording
of about 30 uses in a row):
- The use function opens the party screen, refuses a Shadow Pokémon (message 0x424C) and a Pokémon the candy does
  nothing for (0x4261), and otherwise records the six stats, applies the candy and redraws the party screen.
- It then plays the level-up jingle (sound 0x4CA) **and waits for it to end**: the field music is paused (with a
  fixed 50 ms wait counted in frames), the jingle is started, and the game polls its status once per game frame until
  it has stopped; then the music resumes. Only after that does "<name> grew to level N!" (message 0x423E) appear.
- Next the stats panel (menu 0x107) opens twice in place: first with the gains, then with the new totals, each
  closed with A; then the panel closes. After that come the moves learnt at the new level (the "forget a move?"
  choice when four are known) and the evolution check, which fades the screen and runs the evolution scene.
- Because the message waits for the jingle, the time from choosing the Pokémon to the message depends on when the
  jingle's end falls relative to the game's frames: one game frame either way between two machines is to be expected
  (**likely**: a sound ending inside a frame).

**Teaching a TM or HM** (**verified**, `itemFieldUseTM` (0x80022834)):
- A TM or HM item (item kind 4) stores its **machine number** in its record (+0x18); machine numbers **50 and up are
  HMs**. The machine table gives each machine's move (pokemon-data.md "TM table").
- The game asks "teach it?", opens the party screen in "use item" mode, then refuses a **Shadow Pokémon** (message
  0x424C), a Pokémon that already knows the move, or one whose species can't learn it (the species' compatibility byte
  for that machine is 0); otherwise it runs the usual "forget a move?" choice and teaches the move (and applies
  friendship event 4, friendship.md).
- **Used up or not:** on success the function reports how many to remove: **1 for a TM, 0 for an HM**, and the bag
  subtracts that. So TMs are single-use and HMs reusable, decided at this one point.

## Using an item in battle, and the infinite ball glitch (**verified** from the byte-matched code)

In battle, choosing an item (a Poké Ball, a potion) records the item **and the bag slot it was taken from**. The item is
removed only when the turn plays out, after its effect: `heroItemDecItemDataId` (0x801299C8) is called with that slot
number and passes it to the pocket's remove routine (`fn_80140ACC`). With a slot number, that routine looks **only at
that slot**, and removes one only if the slot **still holds the same item**. If it doesn't, it removes nothing and
reports nothing. (Outside battle the slot number is -1, and the routine searches the pocket for the item instead.)

That is the classic **infinite Poké Ball glitch**: in a double battle, choose a ball for the first Pokémon's turn,
then on the second Pokémon's turn open the bag and move that ball to another position in the list. When the turn
plays out the ball is thrown, but its old slot now holds a different item, so the count doesn't drop. It works with
any ball, the Master Ball included, and with any item used in battle the same way (likely: same call). XD removed it.
**For a port:** this is console behaviour, so a faithful port keeps it exactly; a fix (remove by item, not by slot)
belongs only behind an optional switch.

## Disc data

The item table (names, kinds, prices, effects) is in `common_rel` (game-data-tables.md (not yet published)); the bag's and PC's contents
are save data (save-data.md).

## Efficiency

Not applicable in any special way: short fixed arrays searched linearly when an item is added or used, never per
frame.

## Changing it (for modders)
- Pocket sizes and caps are constants in one switch; a "bigger bag" option or a new item kind changes it in one
  place (plus the bag page table for a new page).
- More PC boxes: the count (3), the box size (30) and the 0x24A4 stride are constants in `pcbox.c`; the save section
  would grow, so a PC port needs its own save format to add boxes (save-data.md).
- **Restoring Rui's pocket**: add page 0 to the page turning and translate the four lines. The behaviour behind "leave
  it to me" doesn't exist in the code read (nothing found reading her pocket in battle), so it would be new design.

### Mod surface
- **Data it reads:** item data (kind, pocket), the bag page table, pocket sizes, the PC box section of the save, mail.
- **Hooks it would need:** pocket sizes and caps (one switch), the box count and size (`pcbox.c` constants) with extra
  boxes stored outside the original save, a new pocket page.
- **Fixed-size limits:** 3 boxes of 30 Pokémon (stride 0x24A4 per box); each pocket's slot count and item cap.
- **Must stay vanilla in GameCube-accurate mode:** pocket sizes, the 3 boxes, the save layout.

## Open questions
- What the two per-pocket switches control.
- Whether any bag mode opens directly on page 0 (Rui's pocket).
- What the second "hizuki" name field (0x56 bytes) in the player record holds.
