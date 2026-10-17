# Message windows (`winMsg*`)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). The text boxes the game shows a message in: in battle, on the field, in
menus, the level-up stat windows and the error window. Code: `win_msg_exact_80105A3C.c` and `win_msg.c`, all
byte-matched, so the mechanics below are **verified** unless marked; what each window *looks like* is not checked.
For the general window system (window records, menu tables, the frame loop) see menus.md (not yet published); for the text itself
(message ids, control codes) messages.md.

## What it does

A message window is an ordinary menu window (menus.md (not yet published)) opened with a message id as its argument. The game uses
ten **message window types**, each mapped to one menu id; **only one message window is open at a time**: opening a
different type closes the current one first (with its close animation), except while the error window is up.

## Data structures

### The message window types (`winMsgResolveMenuId`)

| Type | Menu id | Opened by | Use (from the callers) |
|---|---|---|---|
| 0 | 0x40 | `winMsgOpen(0, ...)` | (no direct caller found) |
| 1 | 0x0F | `winMsgOpen(1, ...)` | title screen, battle menus, movies (**likely**: a system / caption box) |
| 2 | 0x10 | `winMsgOpen(2, ...)`, 189 calls | the general box: evolution, summary, GBA link, event scripts, the bag and TM screens |
| 3 | 0x40 | `winMsgOpenField`, `winMsgOpenFieldWithSE` | **field** messages (field poison, ...) |
| 4 | 0x50 | `winMsgOpenFight`, `winMsgOpenFightNoWait` | **battle** messages; this window sizes itself to the message (`GSmsgGetRect`: width and height) |
| 5 | 0x51 | `winMsgOpenLevelUpStatus` | the battle level-up stats window |
| 6 | 0x10C | `winMsgOpenError` | the **error window** (below) |
| 7 | 0xE6 | `winMsgOpen(7, ...)` | the Colosseum battle-mode menus only |
| 8 | 0xE8 | `winMsgOpen(8, ...)` | NPC events (`gs_npc_event.c`) |
| 9 | 0x107 | `winMsgOpenLevelUpFiledStatus` | the field level-up stats window: before and after values, opened from the bag code in `gs_title.c` (**likely** Rare Candy) |

Types 0 and 3 share menu 0x40, which opens with a different animation (window sequence 0x26 instead of 0x22).

### State
- `gWinMsgCurrentType` (`lbl_80478B30`, s8): the open type, -1 when none.
- `gWinMsgErrorOpen` (`lbl_8047AD10`, u8): the error window is up.
- Per window (its free work area): the message handle, two parameters, the keys pressed (`winMsgButton`), and for
  type 4 the message's width and height.

## Flow

1. **Open** (`winMsgOpen(type, message, wait, pause)` and its wrappers): close a different open type, remember the
   type, open the menu with the active window as parent, passing the message, flags (bit 0 = **pause**: wait for the
   player, **likely**; bit 1 = **no wait**) and an optional sound.
2. **Window control** (`winMsgCtrl`, run by the window system each step of the window's life):
   - opening: load the message (`GSmsg`), start the opening animation;
   - showing: run the message (`GSmsgExec`, which prints it at the text speed); the window's "accepted" byte
     (+0x98) turns on when the message has finished;
   - closing: start the close animation (window sequence 0x2A); closed: free the message.
3. **Checking** (`winMsgCheck`, `winMsgCheckField`, `winMsgCloseCheckFight`): -1 = no message window, 0 = finished
   (accepted or cancelled), 1 = still showing. Scripts and battle code wait on this.
4. **Close** (`winMsgClose`, `winMsgCloseField`, `winMsgCloseFight`, `winMsgCloseLevelUpStatus`): closes whichever
   type is open (not the error window) and sets the type to -1.

**The error window** (type 6): opened with no parent window, never waits, and sets `gWinMsgErrorOpen`; while it is
set, the other types do not close the current window when they open, and the common close functions leave it alone.
Only `winMsgCloseError` closes it. Used by the disc-reading code (`gs_dvd.c`: the disc error screen, whose state follows the drive status: cover open,
no disc, wrong disc, retrying, fatal error; **verified**) and a few system screens (`cursor_bios.c`, `gs_range_8000BE74.c`, `gs_range_80101910.c`).

### The yes / no box (**verified** from `menuSub.c` and its callers)
`menuSubOpenYesNo(type, x, y, startOnNo)` opens a small choice window over the active window: type 0 = window
0x11, 1 = 0x12, anything else (0x7F) = 0x44. A non-zero last argument starts the cursor on the **second** item.
It places the window at (x, y) when both are not negative, then waits in `windowCheckCursor` (the standard window
input loop, the same one every menu uses) and returns the window's value as a signed byte: **0 = Yes** (the first
item), **1 = No** (**likely -1** when cancelled with B, from the window value's usual meaning; not shown in this
function). Callers treat only 0 as yes and start on "No" for anything destructive: New Game's "start a new story / delete the
save?" (`menuSubOpenYesNo(0, 0x3C, 0xA6, 1)`), saving the options, overwriting a save; so **A on the starting cursor of
a destructive question answers No**. The scripts' yes / no box (`std011_dialogDisplayYesNoMenu`) also starts on "No"
(**verified** in Dolphin), even for harmless questions such as Rui's "stay with me", so a scripted run must press Up
before A. For a PC port: the box has no input code of its own, so if A does nothing there the
fault is in the shared window input (`windowCheckCursor`, the window's key port), not in the box (**likely**).

### Window input keys (**verified** from `menu.c`, `_menuUpdateKeyInfo`)
Every window and menu reads one **16-bit key word per controller port**, built each frame by
`_menuUpdateKeyInfo`. For each of the four ports it first tries a GameCube pad (`_menuGetGcKeyInfo`) and, if there is
none, a **linked GBA** (`_menuGetAgbKeyInfo`), so a GBA plugged into a port drives menus like a pad (that is how
the trade partner picks, trading.md). The game's own bit layout (pad bits **likely** the SDK's standard `PAD_BUTTON_*`
values; GBA bits the GBA's key register):

| Game key bit | GameCube pad | GBA |
|---|---|---|
| 0x0001 / 0x0002 / 0x0004 / 0x0008 | up / down / left / right (d-pad, and the control stick past 0x20 from centre; a diagonal sets two directions) | up / down / left / right |
| 0x0010 | A | A |
| 0x0020 | B | B |
| 0x0040 / 0x0080 | X / Y | (none) |
| 0x0100 | Z | Select |
| 0x0200 / 0x0400 | L / R | L / R |
| 0x0800 | START | Start |

The key info keeps the current keys, the previous frame's, **pressed** (newly down this frame), **repeat** (pressed,
then again every few frames while held: first after 15, then every 5 ticks, counted down by the text speed's
characters-per-frame) and held. A new direction press restarts all four direction timers. Which port a window
listens to is its port mask (`lbl_8047CDE8`, bytes 1, 2, 4, 8 for ports 1-4; a bug a PC port can hit if it ignores the mask).
**Name entry** (`menuNameEntry.c`) as an example: A picks the letter (when the name is full, the cursor
jumps to a fixed slot, **likely** the OK button), B deletes one, X switches between the two letter pages, **R toggles the last letter's
case** (the code is the Japanese
`exchangeDakuon`, but the US data gives it only an upper / lower case pair list, and the screen labels R "A⇔a";
**verified**, name-entry.md), START finishes (it sets the done flag
directly), the repeat bits move the cursor (all **verified** from the handler; the page names are not checked).

## Text contexts: which code draws which text 

**Every string on screen is a message task** (`gs_msg.c`): something starts a task for a message key at a
position, the task expands the template (control codes, inserted names and numbers) and `msgTaskDraw` draws it.
There are only a few **entry points** (**verified**, `gs_msg.c`):

| Entry point | Arguments | Used by |
|---|---|---|
| `GSmsgExec` (through `winMsgDrawMessage`) | key, mode, type | the **message windows** (types 1-9 above): dialogue, battle text, system boxes |
| `fn_800FB680` | x, y, colour, key | one line inside the current menu item or sprite: by far the most used (about 270 call sites in the C) |
| `fn_800FB8C8` / `fn_800FBB34` | x, y, width, height, colour, key | text fitted into a box (descriptions, aligned numbers); the difference between the two wasn't read |
| `fn_800FAEF8` | x, y, colour, printf-style format | developer text: the move viewer, debug screens, the room-change log (`floor.c`) |

**The font is a property of the string, not of the place**: a task starts in font 0 and switches with the control
code **`<07:NN>`** (font NN, `msgctrlFont` -> `GSmsgSetFontInfo`; fonts.md). So today the same template (for example
0xCB, "number in font 4") looks the same wherever it is used, and a per-context font setting has to come from the
**caller**.

**Which code draws each context** (entry points **verified** from the callers; what each caller shows is **likely**,
from its file, function and message keys):

| Context | Code | How it reaches the text |
|---|---|---|
| Field dialogue (people talking, signs, items) | `std054_characterTalk` (`people.c`), `winMsgOpenField*` | message window type 3 |
| Battle messages | `winMsgOpenFight*` | type 4 (sized to the message) |
| General / system boxes | `winMsgOpen(1 / 2, ...)`, NPC events type 8, level-up windows types 5 and 9 | window types |
| **Battle HUD: names, HP, level** | `fn_80011EA4` (`gs_range_80011EA4.c`), the battle status panel's draw callback, by sprite id: **HP** (0xA1 / 0xAC: the animated current value and the maximum, template 0xCB), **level** (0xA2 / 0xAD, template 0xCB), **name and gender sign** (0xA3 / 0xAE: template 0xE9, then ♂ / ♀ with template 0xD0) | `fn_800FB680` |
| **Move names and the battle menu** | `menuFight.c`: `menuFightDrawWaza` (move name, template 0xE7), `menuFightDrawPP`, `menuFightDrawType`, `menuFightDrawCmdMsg` (FIGHT / POKéMON / ... labels), the timers, and the Shadow / "secret" screens (`menuFightDrawSecret*`) | `fn_800FB680` / `fn_800FB8C8` / `fn_800FBB34` |
| Pokémon names in menus | party menu `menuPokemon.c` (`menuPokemonDrawItem`), summary and bag `gs_pokemon_summary.c` | `fn_800FB680` |
| Menus in general | each menu item's **draw callback**, named in the menu tables (`data_802EF0A8.c`; menus.md (not yet published)): `menu/menuCB*.c`, `menu/menu_middle.c` | `fn_800FB680` / `fn_800FBB34` |
| **P★DA** | `menu/pda*.c`, `menu/menu_pda_mail.c` | the same |
| Shops, name entry | `menuShop.c` (`menuShopDrawListText`, ...), `menuNameEntry.c` (`menuNameEntryDrawName`, `...DrawTitle`) | the same |
| **Title screen** | `gs_title.c` (`fn_800205C8`, `fn_80020EA4`) | `fn_800FB680` |
| GBA link screens, NPC event screens | `gbaCommunication.c`, `gs_npc_event.c` | all three |
| Window sprites with text | `winSpriteDraw` (`win_sprite.c`) | `fn_800FBB34` |

**For the port (recommendation, not built):** keep a small "current text context" value. Set it (1) from the window
type in `winMsgDrawMessage`; (2) from the menu id while a menu's item callbacks run (the menu system knows which
menu it is drawing, so one table maps menu ids to contexts: P★DA, title, shop, bag, ...); and (3) explicitly in the
two battle HUD drawers (`fn_80011EA4`, `menuFight.c`), which share menu machinery with ordinary menus. Then apply
the per-context font override where a task resolves its font id (`GSmsgSetFontInfo`), keyed by context and font id,
so a string's own `<07:NN>` switches still work inside each context.

## Disc data
Message ids index the game text (messages.md). The window frames and animations are the menu tables' (menus.md (not yet published)).

## Changing it
- A new message box style: add a type and menu id to `winMsgResolveMenuId` (code) and a menu table entry (data).
- The one-window rule is in the open functions: a port feature such as "message log" would hook `winMsgOpen*`.
- The text speed applies through `GSmsgExec` (messages.md), not here.

## Efficiency
One message window at a time, its message loaded when it opens and freed when it closes.
- The name-entry screen's pre-simulated background is described in name-entry.md ("Efficiency").
- Key repeat is computed once per frame for all four ports in one pass (`_menuUpdateKeyInfo`), so every window
  reads ready-made pressed / repeat bits instead of polling the pad itself.

## Region notes
- Name entry keeps the Japanese **dakuten** key code (R, `exchangeDakuon`) in the US build, with new data: an
  upper / lower case pair list, so R toggles case (name-entry.md, Region notes).

## Open questions
- What type 0 and types 1, 7 and 8 look like on screen (visual check).
- Whether the pause flag means "wait for A" (likely) or "pause the scene".
