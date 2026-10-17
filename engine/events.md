# Story events: what starts them

Status: **partial** (2026-10-17). A map of every way the game starts something in the field (a cutscene,
a battle, a warp, a message, a mail), with links to the pages that hold the details. Story progress itself is kept
in story flags and variables (flags.md (not yet published)). **Verified** from the decomp's C and the decoded US scripts unless marked.

## The trigger kinds

| Trigger | How it fires | Runs | Details |
|---|---|---|---|
| **Entering a room** | the room loads | the room's C pre / main / post functions and its five script slots: +0x28 in its own thread (music), +0x2C synchronously (who is shown), +0x30 / +0x34 / +0x38 on entry / every frame / on leaving | new-game-flow.md steps 6-7 |
| **The room's main loop** | every pass of its +0x34 script, which also calls `heroMoveMain` | position boxes and flag checks in the script start scenes and battles (Outskirt Stand: the arrival scene, then Willie) | new-game-flow.md step 7; player-movement.md |
| **Walking through a plane** | collision walk-through plane -> event id -> hero-event table (kind 1) | usually the common script's **exit** (function 4: target room, entry point, sound) | collision.md, common-script.md |
| **Door planes** | hero-event kind 2 | the common script's **door** function (5): opens the door object (flags, collision, animation); the exit plane behind it warps | common-script.md |
| **Examining** (A on a plane) | collision examine plane -> hero-event kind 3 | e.g. a **sign** (common function 11) or a room function | npcs.md ("Choosing who to talk to": planes are the fallback) |
| **Talking** (A near a person) | the talk-target test | the person's talk script (group, index as arguments); item balls give their item | npcs.md |
| **After a warp** | an auto-event queued before the room change (`heroMoveAddAutoEvent`) | runs at the first `heroMoveMain` pass in the new room (arrival animations of lifts and rides) | player-movement.md, common-script.md (functions 6 / 7, 12) |
| **Room-change fades** | the fade script set with the room change (`floorSetFadeScript`) | common function 8 (fade in) etc. | common-script.md |
| **Battles** | a script calls `fieldStartEncount(battle, ...)` and waits | the battle room is pushed; the script continues with the result (2 = won) | battle-engine.md, "The battle entry" |
| **Footsteps** | every 7 units walked | the step callbacks (poison, friendship, the heart gauge, the day care, P★DA mail) | field-steps.md |
| **Mail** | room scripts and the common script's mail scheduler (function 21, run on entering many rooms) check story flags | `mailMainSendByScrpt(n)`: mail n arrives once | common-script.md, pda.md |
| **Menu rooms** | a script command pushes a full-screen menu (shop, PC, trade, name entry, ...) | the menu's room; the script resumes when it pops | menu-rooms.md |
| **The pause menu** | X / Y / START in the field | the field menu (bag, party, P★DA, save) | pause-menu.md |

## Room variants
- **`_bf` rooms**: the battle copy of a map room, pushed for battles fought there (battle-arenas.md).
- **`_ev_N` rooms**: separate rooms in the room table (ids 0x8A-0x8D) used as **travel scenes**: the four are the
  **subway ride** between The Under (`M4_subway_B3_ev_1/2`) and the Shadow Pokémon Lab's garage
  (`D1_garage_B2_2_ev_1/2`). The train's script (`M4_train_1F`) checks the Subway Key flag (1104) to decide the
  direction, shows "inserted the Subway Key" and links to the departure scene, whose script links on to the other
  end's arrival scene (**verified** from the four scripts). They are not swapped in by code: scripts link to them
  like any other room.

## Changing it
- Most triggers are **data** (collision planes plus hero-event records, placement records, room scripts): adding a
  door, sign or exit needs no code. Position-based events live in room scripts.
- The common script's functions are shared by every room, so changing one changes all exits, doors or fades.

## Open questions
- Which rooms use the common script's minimal main loop (function 3) instead of their own.
- The movement scripts of walking NPCs (placement record +0x10): when they start.
