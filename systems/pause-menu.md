# The field pause menu

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). The menu the player opens on the field: party, P★DA, bag. Code:
`fieldMenuMain` (proposed name; today `heroMoveRunEvent` (0x8000D710), in `menuFight.c` / `gs_range_8000D290.c`) and its thread
`fn_8000DAB0`; both byte-matched, so the flow is **verified** unless marked. Window mechanics: menus.md (not yet published).

## What it does

On the field, pressing **Start, X or Y** (key mask 0x1C00, checked by the hero movement code after its events; the
button names assume the SDK's pad bit layout: **likely**)
opens the pause menu. While it is open the field is frozen: model animations pause, NPCs stop updating, the music
is ducked; the menu offers the **party**, the **P★DA** and the **bag**, and returns to the menu after each until the
player closes it, or leaves straight to the field when an item from the bag starts an event.

## Data structures
`gFieldMenuIsOpen`, `gFieldMenuDialogId`, `gFieldMenuThreadGroup`, `gFieldMenuChoice` (`lbl_8047A2A0`-`lbl_8047A2B0`;
proposed names). Nothing is saved.

## Flow (`fieldMenuMain(mode)`)
1. Hand pending hero events to the event system, end any new-mail notice (`mailMainReceiveTerminate`: the blinking
   mail icon stops, field-steps.md), pause every model animation, set `gFieldMenuIsOpen` (the people task checks it
   and **skips NPC updates**), draw the field into an off-screen image (`menuCreateOffScreen`, timed by the frame
   rate) and wait for it, duck the music (`soundDuck`).
2. Pick the dialog: menu 0x41, or **0x109 during a Battle Mode challenge run** (variable 0x8AE, below).
3. **Mode 0** (the buttons): loop: block the field task and the room's threads, open the dialog on its own thread,
   wait, then by the choice:
   - 0 **party** (`menuPokemonOpen(1, 0, 0)`), then back to the menu;
   - 1 **P★DA** (`menuPdaOpen`), then back;
   - 2 **bag** (`fn_80018F88`): if the chosen item starts a field event, queue it as a hero auto-event and leave
     the loop; otherwise back to the menu. Which items start an event comes from a small list (`lbl_803A1B90`: up to
     8 pairs {item id, event}, count at +0x40): `fn_80019070(item)` returns the paired event (or -1) and **empties the
     list** after each lookup; the bag code checks the same list and otherwise shows message 0x426A (that the item can't be used
     now). (**verified**): The list is filled by **script command 17** (`std017_menuQueueAdd(item,
     event)`, up to 8 entries), so **a key item works only where the current script has registered it** before the
     bag is opened (23 registrations in the decoded US room scripts, in places such as The Under, Realgam
     Tower, the Shadow Pokémon Lab and the Pyrite Town police station);
   - anything else (cancel): leave.
   Model animations run again while a sub-menu is open and pause again when it closes.
   The pause menu itself is a window over the frozen field, but the party, the bag and the P★DA are **menu rooms**:
   opening one pushes its room on the room-context stack and suspends the field (menu-rooms.md).
   **During a challenge run** (variable 0x8AE, below) dialog 0x109 offers only the **party** (choice 0) and **quit**
   (choice 1: a yes / no question, message 0x3C46, then `menuSaveLoadSaveGame(1)` and the warp `std223_askYesNoThenWarp395` to
   the Battle Mode menu, room 0x395): no P★DA and no bag. The built copy of this loop is `gs_range_8000D290.c`; an
   older copy in `menuFight.c` lacks this branch.
4. **Mode 1**: open the bag directly, without the menu. Script command 208 (`std208_promptDNASampleSelection`)
   uses this; on the US disc it occurs 15 times, in the **Shadow Pokémon Lab** and once in Realgam Tower (decoded
   room scripts): the doors there ask the player to choose a **DNA sample** item from the bag
   (the sample items, ids 0x200-0x211, are all named DNA SAMPLE in the US item table: **verified**; menu-rooms.md maps them
   to species), and the chosen item comes back as the event.
5. Close: unduck the music, close the dialog, release the off-screen image, end the hero events, resume animations,
   clear `gFieldMenuIsOpen`.

## Variable 0x8AE (2222): the game mode

Variable 0x8AE (`FLAG_system_gamemode`, its original name from an assert string) holds the **game mode**: 0 = story,
1 or 2 during a Battle Mode run (**likely** single or double battles: a helper picks 2 or 4 Pokémon from the same
setting; **verified**, 1 single and 2 double, set only by Colosseum and Mt. Battle runs; Battle
Now and Gang battles never set it, battle-mode.md). **Verified** from the code, assert strings and the decoded room
scripts unless marked.
- **Values:** the Battle Mode menu asserts it is 0 (story) on entry; starting a Battle Mode run sets it to 1 when the
  battle type (save section 14 +0x04) is 0, otherwise to 2; leaving the Battle Mode menu (`menuColosseumBattleExit`)
  sets it back to 0. Both Battle Mode kinds set it: Colosseum battles and Mt. Battle 100. Battle Mode's Mt. Battle
  challenge is played in the story's own Mt. Battle rooms, so every Mt. Battle area script (`D2_cloud_*`, `D2_magma_*`, `D2_rest_*`, the crater Colosseum)
  branches on it: value 0 updates the story's Mt. Battle flags (2624, 2625, 2629, 2673), values 1 and 2 set variables
  2812 and 2905 instead.
- **Also cleared** by the GBA conversion code and by the Ho-Oh gift, which ends a run (menu-rooms.md, "Gift
  Pokémon"). No room script writes it.
- **What changes during a run:** the pause menu offers only the party and quitting (above); field battles start
  through the `toolbattle_menu` room (`menuCBBattleReturnToMenuRoom` (0x8006A65C), with story variable 0xB59 from the area scripts) instead of the
  normal encounter path; when the run ends (0x8AE cleared, **likely** on a loss or on quitting) the game returns to the
  Battle Mode menu (room 0x395, `colosseumbattle_menu`); script command 216 skips its party heal (field-steps.md);
  every rest area's menu (command 199) has a choice leading to the same room change (command 229); and the GBA code
  uses a different player record for the player's name (`fn_8006AEEC`).
- The rest areas' coupon commands 236 / 239 are either shared with the story run or count something else (not
  traced).

## Disc data
The dialogs (menu ids 0x41 and 0x109) are entries in the menu tables (menus.md (not yet published)).

**what the menu looks like on screen.**
- **The entry names are text, not pictures** (**verified** 2026-11-15 from the code and the US, European and Japanese
  text: each of the menu's item records carries its message key, POKéMON 0x136, P★DA 0x138, ITEMS 0x137 and EXIT
  0x13A, and the menu's draw step prints them in font 0; the Italian text bank picks font 3 for them instead). The US game's shared text bank (`common_rel`)
  holds the three labels POKéMON, ITEMS and P★DA (entries 0x136-0x138, so the US menu says POKéMON and ITEMS rather
  than "party" and "bag"), each starting with the font-switch code for font 0, the main 22 px font (fonts.md). A
  fourth label, **EXIT** (0x13A, same font code), comes right after them and closes the menu (**likely**: it matches
  the frame's fourth area below and the code's "any other choice leaves", and players remember an exit entry).
  **B closes the menu** (**verified** 2026-11-15: the shared menu routine's B branch returns "no choice", and the
  pause menu then leaves); Start does not.
- **On screen** (**verified**, a screenshot of the running game in Dolphin): **four separate rounded buttons** stacked
  down a metal rail on the right, in the order **POKéMON, P★DA, ITEMS, EXIT** (the order of the code's choices: 0
  party, 1 P★DA, 2 bag, then leave; the text bank's order is different). The rail has one notch beside each button.
  The **chosen** button slides out to the left and its text turns **bright green**; the others sit back with dimmed
  text.
- **The art** (**verified**, decoded and viewed): texture `menu_040` in `field_common.fsys` holds a rail piece with
  notches and one rounded button panel. **Each button and each rail piece is its own menu item** (**verified**
  2026-11-15, from the menu data): the rail grows by one middle piece, 43 pixels tall, per button, so the buttons sit
  43 pixels apart (the challenge-run dialog 0x109 builds its shorter rail the same way). A fifth button fits a 640 x 480
  screen (at y 95, 138, 181, 224 and 267). The labels are drawn onto the
  buttons by the text system.
- So a translation changes the words without touching any picture, and an added entry likely needs only a new item
  record (with its own button drawn like the others), plus new art only if the rail has to grow (below).

## Changing it
- New pause-menu entries: the dialog's items are menu-table data, but the choice is handled by the fixed `if`
  chain in step 3 (code). A port mod adding an entry (options, a quick save) would extend that chain.
  Each entry is its own button on the rail, so a fifth entry is one more button drawn
  from the same pieces, and its label is one more text line in font 0. Verified (above): the
  rail grows by one 43-pixel piece per button, so **no new art is needed**.
- **Where a save puts you back** (**verified** 2026-11-15): saving at a PC stores the game's **return point**, which is
  where Continue starts **and** where a whiteout sends the player; only the script command run at a PC sets it. So a
  save made away from a PC (a port's "save anywhere") still continues at the last PC used, unless the port keeps the
  exact spot itself. Setting the return point to the current room instead would also send whiteouts there, so a port
  shouldn't.
- **The PC menu** the pause menu could open is a menu room of its own (the PC box menu: Pokémon Storage, Item
  Storage, Save, Exit), likely opened the same way the pause menu opens the P★DA.
- The open buttons are in the hero movement code (mask 0x1C00).
- Because NPCs freeze through `gFieldMenuIsOpen`, any new field overlay that should stop the world can set it.

## Efficiency
The field is drawn once into an off-screen image and shown behind the menu, so the 3D scene does not have to be
rendered while the menu is open (**likely**; the image is released on close).
(**verified**, `menu_offscreen.c`): The image is drawn as one full-screen quad and **darkened**, not
blurred: a fade takes its brightness down by a set percentage over a set time, at a speed tied to the text-speed
setting.

## Open questions
- What `menuSaveLoadSaveGame` (0x800889E4) checks before the quit warp.
