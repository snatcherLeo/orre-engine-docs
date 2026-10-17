# Memory card tasks

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). How the game talks to the memory card: every load and save is a
numbered **task** run in the background while the caller waits. Code: `memcard.c` (byte-matched for the parts used
here). The save data's layout is in save-data.md; this page covers how it gets to and from the card. **Verified**
from the code unless marked.

## What it does

A caller asks for a task kind and blocks until it finishes: `fn_801D0748(kind, work size, channel)` (proposed
`memcardRunTask`) starts the task and yields every frame until the task reports a non-zero result, which it returns.
The task itself is a per-frame state machine (`fn_801CDB04`) run as a GS per-frame task, so the screen keeps drawing
(dialogs, the memory-card messages) while the card is accessed.

## Data structures
**The task state** (`gMemcardTask`, `lbl_8047B3D4`, `MemcardTaskState`): task kind, card channel, work size, result
(0 while running), the state machine's state, an error code (which message to show), the player's answer to a dialog,
a work buffer (0x1E600 bytes) and a card work area (0xA000 bytes) allocated per task, and the per-frame task's id.
The state block itself sits at a **random 8-byte-aligned offset** chosen once at init (`fn_801D0A30`: `CARDInit`, then
an offset from the game's random function), **likely** to keep memory-editing cheats from finding it.

## Task kinds (from their callers)

| Kind | Called by | Meaning |
|---|---|---|
| 1 | the title, after START (`std021_newGameIntroSequence`, state 0xC8) | load the save at boot (new-game-flow.md, step 3) |
| 2 | Battle Mode menus (two callers: `fn_800798E8`, `fn_80075C94`) | **read the story save for Battle Mode** (**likely**): the caller goes on only if the result is 3 and the loaded save's **save count** is non-zero, otherwise it shows message 0x44DB (no Story Mode save on the card in slot A; **verified** from the text) and closes the menu. (**likely**: Battle Mode borrows the story save's party) |
| 3 | after the staff roll (`fn_80035F64`, `movie.c`), the P★DA's save (`fn_80038A0C`), each followed by kind 4 | **read the card's save for the overwrite check** (**verified** from both callers): the caller backs up the save in memory, reads the card's save over it with kind 3 and, if the card has a save (save count non-zero), compares hero value 2 of the card's save with the current game's, asks whether to **overwrite the existing save** in slot A (message 0x3C02, **verified** from the text) when they are equal or the current game's save count is 0, restores the backup, then writes with kind 4 |
| 4 | the save menu (`std235_menuShowSaveMenu`, script command 235), the shop's coupon exchange, the P★DA, Battle Mode, GBA conversion, the staff roll | **save** (**verified** by the save-menu caller; result 4 = written) |
| 5 | Battle Mode, after registering Pokémon (`fn_800889A4` from `menuColosseumBattle.c`) | **save the Battle Mode registration** (**likely**): section 14 is backed up first and restored if the save fails; the no-card path warns that the registered Pokémon will be lost on reset (message 0x3C60, **verified** from the text) |
| 6 | Battle Mode saves (`fn_80088D84` and the Mt. Battle path, `gba_conv.c`) | **a Battle Mode save written from a temporary save image** (**likely**): the whole save image (0x1DFD0 bytes) is copied aside, slot ids renumbered, section 14 +0x1C and variable 0x8AE cleared, the task run, then the image restored; the Mt. Battle path first records the player's memo, asserting the mode is `BATTLEMODE_BATTLEYAMA100` |
| 7 | the options menu (`fn_80020C9C`) | save the options |
| 0xD | Battle Mode's rule editor (`fn_80088964`) | **save the edited rules** (**verified**: the prompt before it asks whether to save these rules to the card in slot A, message 0x44B1) |
| 2, 0xB, 0xC | `fn_8000C358`, `fn_8000C3A4`, `fn_8000C328` (`gs_range_8000BE74.c`, next to `dbgMenuMsgTest`) | **likely** debug-menu entries that run a task on their own; kinds 0xB and 0xC not decoded |

(**verified** from the task code's kind switches): Kinds **1, 2, 3 and 8** share the
header-read path (`memcardBeginHeaderRead`) and the same ending (`memcardEndAccess`), so they are the **read-side**
tasks (**likely** loads and checks) (kind 8 reads the header first and then writes, see below); kind 4 and the kinds above it take the write paths. Kind 8's caller is the trade menu, which starts it with `fn_801D055C(8, 2, 0)` directly and polls the result itself, rather than
going through `fn_801D0748`. It is **the trade save** (**verified** from the messages around it: the card in slot A
and the GBA Game Pak are saved together, and on failure the trade is reported as not saved). While the task runs,
the trade code sends the GBA its go / stop signal (`fn_801D046C`), and kind 8's write path waits at a resume state
for it (**likely**). The trade counts as done only on result 4; otherwise the save image is restored from the copy
made before the exchange (trading.md, state 18).

**Results** seen in the task code: **4** after a write completes (the save count in the save header, `gamedatasaveGetStatus` field 4, is incremented
there), **-1** from the abort state, 1, 2, 3 and 5 from other check paths (not decoded). The title, Battle Mode and the
overwrite check all treat **3 as "a save was read"** (**verified** from the three callers; the
title creates an empty save for any other result, **likely**); the coupon exchange treats anything
but 4 as "not saved" and undoes the exchange.

*Results in detail* (from `memcard.c`):

| Result | Set when | Label |
|---|---|---|
| 1 | a read kind (1-3) finds **no valid save header** on the card (the newest-header search returns none) | **likely** "no save" |
| 2 | a task other than the first save (kind 9) has just **created the save file** and set its banner and icons | **likely** "new empty file made" |
| 3 | a read kind (1-3) copied the card's save into memory | **verified** "save read" |
| 4 | a write finished (kinds 4, 9, 10, 11 also bump the save count) | **verified** "written" |
| 5 | after card question 8, when an error code is still set | **guess** "gave up after an error" |
| -1 | the abort state (43), e.g. after 6 failed probes | **verified** |

Kinds **10 and 11** copy the current save into the work buffer before writing (**verified**), so they are
write-side kinds too (kind 0xB is the debug entry `fn_8000C3A4`).

## Flow
1. `fn_801D055C` (proposed `memcardStartTask`): wait for any running task to finish, set kind, channel and work size,
   clear result, state, error and dialog answer, allocate the two buffers, and create the per-frame task.
2. *First states* (**verified**, `fn_801CDB04`): state 0 clears the per-task fields and, for
   kind 4 with a save count of 0 (the **first save**), switches the task to **kind 9** (**likely** "create the file");
   state 1 probes the card (`CARDProbeEx`): busy -> probe again, ready -> state 2 (mount, `CARDMountAsync`),
   anything else -> retry up to **6 times**, then record the error and go to state 0x2B, the path that ends with
   result -1 (state 43 sets it). The title answers -1 by going back to the PRESS START wait, so a port's card layer
   must report "ready" (or a "no card" the game handles) for the title to get through (**likely**).
3. The per-frame task steps through its states (card checks, dialogs such as "save?" and error messages, reads or
   writes), sets the result when done.
4. `fn_801D0748` sees the non-zero result, cleans up (`fn_801D039C`) and returns it.

## Changing it
- For the PC port: replace the card access under the task (the `CARD*` calls) with files; keep the task interface
  (kinds and results), since callers branch on the results. Each task must finish with a result, or its caller
  waits forever (the title waits on kind 1 right after START).

**Source file name** (**verified** from a retail assert string): the Battle Mode save wrappers
(kinds 5, 6, 0xD) come from **`menuCB_SaveLoad.c`**, whose assert names the mode enum value
`BATTLEMODE_BATTLEYAMA100` ("battle yama" = Mt. Battle) for Battle Mode kind 1. In the decompiled code
these functions sit in `gba_conv.c`.

## Region notes
The memory-card file's game code follows the region (regions.md).

## Open questions
- Kind 0xC (debug, `fn_8000C328`) and 10 (no caller found); results 1, 2, 5 are labelled from the task code only;
  what exactly kinds 5 and 6 write; why the overwrite check asks when hero value 2 matches (what
  that value is).
- The per-frame task's states and its dialog messages.
