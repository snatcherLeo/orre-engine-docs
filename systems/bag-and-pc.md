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

### How the bag screen is put together (**verified** from the menu tables and callbacks)

The bag screen is one menu (id 0x59) opened inside the shared menu room for the bag, party and shops. Like every
menu, it is data: a chain of item records, each placing one piece of a single 360 x 322 texture (the ring, bar, frame
and arrows) on the 640 x 480 screen, plus a few C callbacks. The pieces:
- the **list panel** (a see-through dark panel), the **title bar** with the panel's top (two mirrored halves), the
  **side edges** and the **bottom strip**;
- the **pocket tabs** above the list; they are the menu's only selectable items, so "which item" is really the
  cursor's row inside a list the callbacks draw, and "which page" is the tab;
- the **highlight bar** (a body and two mirrored end caps) with dots chasing round its outline, and the **ring
  cursor**, which turns as you scroll;
- the **rows**, drawn by a callback: each name in the standard font, the count right-aligned with a "×" message and a
  number message; rows are **31 px apart**, eight in view (nine while a scroll animates); "CANCEL" closes the list;
- the **up and down arrows** (blinking), the **title text**, the **description box** on the left, and the shop's
  money box;
- a clip rectangle with no picture, which the row callback reads to clip the list.

**Scrolling** keeps two numbers per pocket, the first row in view and the cursor's row within it, in the shared
cursor table, so reopening a pocket puts the cursor back. A scroll eases the list by 31 px back to rest.

**Transparency:** a piece's opacity is its item's alpha times its sprite's alpha. The list panel's sprite has alpha
0xCC (about 80 %), which is why the field shows faintly through the bag; the rest is opaque. The callbacks change
colours every frame: the highlight bar is fainter while browsing and solid once an item is chosen, the arrows blink,
and the frame's edges take their colour from the pocket's page.

**Why it is built this way (likely):** one texture page and a table of rectangles let the shops, the party screen and
every pocket share the same art, with only the callbacks and the page colours changing; the menu tables are global
(any menu can open in any room), while textures, message banks and fonts belong to a room, so showing the bag's art
elsewhere needs only its texture loaded there.

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
- **The turning Poké Ball behind the PC's menu** (**verified** from a log of every draw): a 3D
  menu model, not a flat picture, drawn in HSD's third drawing pass by `menuModelRender`.
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
(**verified** from `heroItemGetItemKindToItemAryPtr` and `fn_80141308`): What happens to the rest
depends on a per-pocket "may split" flag. In the **Poké Balls, Items and Berries** pockets it is set: the add keeps
filling any row of that item below the maximum, then takes **free rows**, so one item can fill **several rows** (99 Rare
Candies plus 5 more make a second Rare Candy row, in normal play). In the TM, key-item and scent pockets it is clear:
only the existing row is topped up and the rest doesn't fit. When a row is used down to 0 the pocket is re-sorted, so
the rows below it move up.

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
   (verified from the code) The **Ein File reader** (`itemFieldUseReadEinFile`) looks the item up in a 5-row
   table of {item id, text id} (`lbl_80266C54`: items 538-541 and 547) and shows that text. It has no "not found"
   branch (an unknown item would read past the table), which vanilla never reaches.
   **TMs on the party screen:** in item-use mode with a TM, each row's label (`menuPokemonDrawItem`) shows ABLE or
   NOT ABLE: NOT ABLE for a Shadow Pokémon or a species that can't learn the machine. A Pokémon that already knows the
   move still shows ABLE; the "already knows" message comes only after picking it.
2. For items used **on a Pokémon** (potions, Rare Candy, vitamins), the bag menu (menu 0x59) is **closed** and the party
   screen opens in **"use item" mode** (`menuPokemonOpenItemUse`, mode 3 in the field, 4 in battle).
3. When the use ends, the use function's result (0, 1 or 2) becomes the bag's result code (0, 3 or 4), the item count
   drops (`heroItemDecItemDataId`), and the bag **reopens**. So using a second potion means choosing it again from the
   bag; the party screen does not stay open between uses.
   
   The use function returns **0 = used** (the bag removes the amount the function reports, from the row the cursor was
   on, and reopens), **1 = nothing used** (B, "it won't have any effect", a refusal: the bag reopens, nothing removed)
   and **2 = used, then leave** (the bag removes it and closes together with the pause menu, so a field event can run:
   the scents and the Time Flute). The bag's codes are 0, 3 and 4 for these.
   **Where the player lands** (**verified**): after any item used on a Pokémon from the bag, the
   bag comes back, whether that was the last of the item or not, and also after an **evolution** started by the item
   (a stone or Rare Candy): the party screen is closed before the evolution scene and is not opened again. The only
   items that end in the field instead are the ones whose effect is a field event (the scents, the Time Flute).
   **The use function opens the party screen and closes it itself, on every path** (`itemUseOpenPartyMenu`, which
   first closes the bag window, and `itemUseClosePartyMenu`, which also remembers the picked slot or -1 after B), and
   the party screen always opens with its cursor on the first Pokémon. The party screen runs inside the bag's menu
   room (0x38F), with no room change. Giving an item to hold is a separate action (`bagItemGive`, party screen mode 5).
   **Scents** (**verified**): the Cologne Case has its own branch; the scent's use function only
   picks the Pokémon and queues a field event (common script 0x596, function 16), then returns 2. Once the menus have
   closed, that event calls `std163_useCologne`: the scent's message, rumble, the Pokémon's cry (waited for), Hyper
   Mode ended if it was on, and the heart-gauge drop with its message. So a scent's effect plays over the field, after
   the menus are gone.
   The Cologne Case first lists only the scents the player owns. The scent is removed as soon as a Pokémon is picked,
   and **any** Pokémon can be picked: on a non-Shadow Pokémon, or one whose heart is already open, the scent is still
   used up and only its message and the cry play (the gauge drop needs a gauge above 0). After every scent use, the
   last one or not, the player is back in the field.
   **PP items** (**verified**): Ether, Max Ether, the Leppa Berry, PP Up and PP Max open the
   chosen Pokémon's move list; B there goes back to the party list (the use function's own loop), B on the party list
   leaves. That loop opens the party list again **while it is still open** (**verified** from the
   code and on screen in the PC build): `itemUseOpenPartyMenu` on an open list only refreshes its six records and
   resumes the cursor (the menu system reuses the open window, marking it "reopened"), with no closing or opening
   slide, so the screen does not move. Elixir and Max Elixir restore every move without asking. A Pokémon in Hyper Mode is refused (0x424D), and a
   move (or every move) already full gives "no effect" with nothing used.
   **What the menus remember between openings** (**verified** unless marked): each bag pocket's
   row (a small cursor table, one entry per pocket, kept in memory from start-up; likely not saved). The Cologne Case's
   scent list and the party list do **not** remember: both start on their first entry every time. Which pocket the bag
   opens on, and where the pause menu's cursor starts, weren't traced.
   (**verified** on screen in a native run): After a scent was used (the menus closed, its effect
   played over the field), opening the pause menu again found its cursor still on ITEMS, and the bag opened on the
   **Key Items pocket** with its cursor on the Cologne Case. So the pause menu's cursor and the bag's last pocket are
   remembered too, for the session (where they are kept wasn't traced). Both lists take a starting row when they are
   opened (the menu system's open call accepts one), which the game itself never passes for these two.
   **Which items do what outside battle** (**verified** from each item's field-use function): every
   potion, drink, herb, status cure, revive, Full Restore, Max Elixir, the vitamins, the curing berries and the Blue
   Flute share the medicine function (the Blue Flute is never used up); Ether, Max Ether, Elixir and the Leppa Berry the
   PP function; PP Up and PP Max their own; Rare Candy, Sacred Ash (no choice: it revives every fainted Pokémon), the
   stones, the TMs, the scents and the Time Flute one each. The rest only print a message and use nothing: "can't be
   used now" (held items, valuables, the X items, the Yellow and Red Flutes, the confusion and pinch berries), "can't be
   used" (repels, Escape Rope, shards, scarves, the Black and White Flutes, the Pokéblock berries) and, for the Cologne
   Case itself, "no effect"; key items work only where a room script allows them. Poké Balls have no field use.
   **Giving and taking held items** (**verified** unless marked): there are two ways to give: from
   the bag (an item, GIVE, then a Pokémon), which always ends back in the bag, and from the party screen (a Pokémon,
   ITEM, GIVE, then the bag to pick the item), which stays on the party screen; taking back exists only on the party
   screen (ITEM, TAKE). The only item that can't be held is an HM; a Potion or a Rare Candy can be, and does nothing
   while held (their held-effect number is 0; **likely** nothing in battle). Giving to a Pokémon that already holds
   something asks to swap; YES sends the old item to the bag, unless the bag has no room, in which case nothing changes
   and "the bag is full" shows. Taking back with a full bag also shows that message and the item stays held. Using an
   item from the bag on a Pokémon never touches what it holds.
   (**verified** from the code; the give and the swap also seen in native runs): The give list from
   the bag (party screen mode 5) returns the
   picked slot for every pick, whether the item was given, swapped, refused at the swap question (NO) or not given
   because the bag was full; only B returns -1. The give itself (and its message) happens inside the list, before it
   closes. Swapping an item for another of the same kind is allowed ("The PECHA BERRY was taken and replaced with the
   PECHA BERRY.") and leaves the bag's count unchanged.
   **Key items** (**verified**): besides the Time Flute, the Cologne Case and the Ein Files (which
   show their text and use nothing), a key item works only where a room script has put it on the "usable now" list;
   there the menus close and the room's event runs (the item isn't removed by the bag); anywhere else "can't be used
   now".
   **The USE / GIVE / TOSS menu** (**verified**, `fn_80017CB8` and the bag's page table): each
   pocket has one option list for the field and one for battle, and each option has flags (never shown; shown only if
   the item has a battle use; greyed if it hasn't; hidden for the Time Flute). In the field the options depend only on
   the pocket: Items USE / GIVE / TOSS, Poké Balls GIVE / TOSS (**no USE outside battle**), Berries USE / GIVE / TOSS,
   TMs USE / GIVE (no TOSS), Key items USE; CANCEL always last. In battle: Items USE only for items with a battle use,
   Poké Balls and Berries USE always, TMs and Key items nothing but CANCEL. So many items show a USE that only prints
   "can't be used now" or "can't be used" (held items, valuables, the X items outside battle, repels, shards, scarves,
   several berries), and four key items (Small Tablet, Data ROM, Steel Teeth, Powerup Part) are never made usable by
   any room. The greyed style exists in the code but no option in the game's data uses it.
   (**verified** in native runs): Each option is a small record (its label's message key, the action
   to run, flags), and the actions are shared by all pockets: the toss action works on the TM pocket as on any other
   (it reads the pocket from the page it is given), with the game's own "Toss how many?" and "Is it okay to toss out
   ...?" (which names a TM by its number, "TM06(s)"). The option box sizes itself to the options it gets: two
   (a renamed option and CANCEL) and three (GIVE, TOSS, CANCEL, as the Poké Balls pocket already has) draw correctly.
   An item's description (the text in the bag's help box) is written for that narrow box, in three short lines with no
   "wait for A" at the end; the Ein Files' text, by contrast, is shown in a four-line message window, page by page.
   (**verified** in a native run): An item can fill **more than one row** of the Items pocket in a
   normal save (a late-game save held Rare Candy x90 in one row and x27 in another); the decrement takes from the row
   the cursor was on.
   **The Cologne Case's order** (**verified**, `fn_80019754`): the scent list walks the item table
   in id order and takes the scents the player owns, so it always reads Joy, Excite, Vivid (weakest to strongest).
   **Evolution stones** (**verified**, `menuUseItemEvolution`): a Shadow Pokémon is refused
   (0x424C) and a Pokémon the stone doesn't evolve gets "no effect" (0x4261), both with nothing used. Otherwise the
   party screen closes, the evolution scene runs **without** the option to cancel with B (Rare Candy's evolutions allow
   it), one stone is used, and the bag comes back.

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
  (**verified**, `menuUseItemListLVUp`): The new level's moves are learnt **while the party screen
  is still open** (a free slot at once; with four moves "forget one?", then the forget screen, or "stop learning",
  which leaves the move unlearnt); **then the party screen closes**, and only then does an evolution run (in its own
  menu room, 0x386), with **B allowed to cancel** it (evolution stones don't allow that). A cancelled evolution still
  uses the candy. Shadow Pokémon are refused by Rare Candy, stones and TMs (0x424C); Potions, cures, vitamins and PP
  items work on them, refusing only a Pokémon in Hyper Mode (0x424D).
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
- (**verified**): Before the party screen the game shows "Booted up a TM" (an HM has its own line)
  and asks whether to teach it; NO goes back to the bag. If the player stops learning at the "forget a move?" step,
  the move isn't learnt and **the TM is kept**. In every case (taught, refused, kept) the party screen closes and the
  bag comes back.
- (**verified** in a native run): The ABLE / NOT ABLE labels are drawn every frame from the Pokémon's
  current data, so a list opened again right after a teach already reflects the new move. The teaching path needs
  nothing before the party screen but the item's machine number and its HM number: the "Booted up a TM" text and the
  question are only a prelude, and skipping them leaves the rest unchanged.

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
