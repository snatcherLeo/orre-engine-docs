# The common script (script group 0x0596)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**guess**: the shared-script idea, with XD's own scripts)

Status: **partial** (2026-10-17). The one script shared by every room: doors, exits, fades, signs and a
few story helpers that rooms and the hero-event table call by id (`0x0596NNNN` = function NNNN). **Verified** from
its decoded bytecode unless marked; read with a script disassembler.

## Where it is

Not in any archive's script entry: it sits **inside `common_rel`** (section 4, offset 0x89CFC; header id 0x0596,
27 functions), so it is loaded with the game's data tables and stays resident. Room scripts are group 0x0100 (the
current room's script); the hero-event table (594 records) mostly calls this one (flags.md (not yet published)).

## The functions

| Function | What it does | Used by |
|---|---|---|
| 0, 1, 2 | do nothing (return 1) | default slots |
| 3 | a minimal **main loop**: calls `heroMoveMain` forever | rooms without their own loop (**likely**) |
| 4 | **exit**: `floorLinkWithSE` with the record's three arguments (**likely** room, entry point, sound; Phenac Stadium's exit passes room 13) | walk-through planes at room edges (hero-event kind 1) |
| 5 | **door**: `floorEventCtrlDoor(current room, door number, 0)`: if the door's lock flag allows it, sets its open flag, removes its collision and plays its open animation and sound (by door type); it does **not** warp: the exit plane behind the door does (function 4) | door planes (hero-event kind 2, a running door number per door) |
| 6 / 7 | an **animated warp** (elevator, lift): faces both characters forward, plays the door object's animation, fades out, queues function 7 as an auto-event and links to the target room; function 7 runs on arrival and plays the matching "arrive" animation and fade-in | lifts in later areas (**likely**) |
| 8 / 9 | **fade in** / **fade out** over 0.5 s (`fadeSet(2 / 3, 0.5)`) | the room-change fade script: `0x05960008` after the opening movie and after a lost battle's warp |
| 22 / 23 | two other fade kinds (`fadeSet(4 / 5, 0.5)`) | |
| 10 | keeps one person shown or hidden by two flags, forever: while the first (collected) flag is clear, shows it only when the second (appear) flag is set | the treasure setup, for items with an appear flag (treasure.md) |
| 11 | a **sign**: open a field message, close it | examine planes (hero-event kind 3) |
| 12 | a special warp: picks one of three door objects (158-160) by its argument, plays its animation, then links to **room 123 (`D4_out_2`, outside Realgam Tower)** with an auto-event for the arrival (**guess**: a story transition there) | |
| 13 | opens the **PC box** (records the return point, pushes room 0x391 `pcbox_menu`) | PCs |
| 14 | waits for a person's motion, then a people helper | |
| 15 | a short sleep (half a second scaled) | |
| 16 | uses a Cologne (`std163_useCologne`) | |
| 17 | runs **evolution** checks | (callers not traced) |
| 18 | sends mail 10 | |
| 19 | the shared **TV news scene**: per room, a camera position and the news jingle (or a special song under two story flags), `std182_tvNewsEvent(room)`; also updates one Shadow Pokémon's state by story flags | the TV in houses and the Outskirt Stand shop's first visit (**likely**, the shop plays its own copy) |
| 20 / 26 | call two code helpers (`fn_80088428`, `fn_80266250`; not traced) | |
| 21 | the **story mail scheduler**: checks pairs of story flags and sends the matching P★DA mail (mails 6, 8, 12-15, 21-24, 27, 28; **likely** once, the mail system skipping a mail already received). It does not set "sent" flags; after some mails it **clears** 2796 / 2797 (`std079`, swapped name). | Outskirt Stand's on-entry function and other rooms (`std209_threadRunScriptMode4(0x05960015)`) |
| 24, 25 | (not read yet) | |

So the "mail by walking" machinery seen in field-steps.md is idle in Colosseum (all footstep rates are 0): mail is
sent here and by room scripts, when story flags change.

## Changing it
- **New door / exit behaviour** for all rooms: one function edit here instead of per room (a port mod can patch the
  bytecode or replace the native it calls).
- **Fade speed** for every room change: the 0.5 in functions 8 / 9.

## Open questions
- Functions 24 and 25, the helpers behind 20 / 26.
- Which rooms use function 3 as their main loop, and the exact rooms of 6 / 7 and 12.
- The door table (`lbl_80478ECC`: per door an open flag, a lock flag, collision object, model resource, door type and
  animations, from `floorEventCtrlDoor`): its full layout.
