# The field pause menu

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). The menu the player opens on the field: party, P★DA, bag. Code:
`fieldMenuMain` (proposed name; today `fn_8000D710`, in `menuFight.c` / `gs_range_8000D290.c`) and its thread
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
   (choice 1: a yes / no question, message 0x3C46, then `fn_800889E4(1)` and the warp `std223_askYesNoThenWarp395` to
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
setting). **Verified** from the code, assert strings and the decoded room scripts unless marked.
- **Values:** the Battle Mode menu asserts it is 0 (story) on entry; starting a Battle Mode run sets it to 1 when the
  battle type (save section 14 +0x04) is 0, otherwise to 2; leaving the Battle Mode menu (`menuColosseumBattleExit`)
  sets it back to 0. Both Battle Mode kinds set it: Colosseum battles and Mt. Battle 100. Battle Mode's Mt. Battle
  challenge is played in the story's own Mt. Battle rooms, so every Mt. Battle area script (`D2_cloud_*`, `D2_magma_*`, `D2_rest_*`, the crater Colosseum)
  branches on it: value 0 updates the story's Mt. Battle flags (2624, 2625, 2629, 2673), values 1 and 2 set variables
  2812 and 2905 instead.
- **Also cleared** by the GBA conversion code and by the Ho-Oh gift, which ends a run (menu-rooms.md, "Gift
  Pokémon"). No room script writes it.
- **What changes during a run:** the pause menu offers only the party and quitting (above); field battles start
  through the `toolbattle_menu` room (`fn_8006A65C`, with story variable 0xB59 from the area scripts) instead of the
  normal encounter path; when the run ends (0x8AE cleared, **likely** on a loss or on quitting) the game returns to the
  Battle Mode menu (room 0x395, `colosseumbattle_menu`); script command 216 skips its party heal (field-steps.md);
  every rest area's menu (command 199) has a choice leading to the same room change (command 229); and the GBA code
  uses a different player record for the player's name (`fn_8006AEEC`; **guess**: a Battle Mode trainer).
- The rest areas' coupon commands 236 / 239 are either shared with the story run or count something else (not
  traced).

## Disc data
The dialogs (menu ids 0x41 and 0x109) are entries in the menu tables (menus.md (not yet published)).

## Changing it
- New pause-menu entries: the dialog's items are menu-table data, but the choice is handled by the fixed `if`
  chain in step 3 (code). A port mod adding an entry (options, a quick save) would extend that chain.
- The open buttons are in the hero movement code (mask 0x1C00).
- Because NPCs freeze through `gFieldMenuIsOpen`, any new field overlay that should stop the world can set it.

## Efficiency
The field is drawn once into an off-screen image and shown behind the menu, so the 3D scene does not have to be
rendered while the menu is open (**likely**; the image is released on close).
(**verified**, `menu_offscreen.c`): The image is drawn as one full-screen quad and **darkened**, not
blurred: a fade takes its brightness down by a set percentage over a set time, at a speed tied to the text-speed
setting.

## Open questions
- What `fn_800889E4` checks before the quit warp.
