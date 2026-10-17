# Room changes: links, entry points and arrival

Status: **partial** (2026-10-17). How the game moves the player from one room (floor) to another: the
calls scripts use, the small "link" record that carries the destination and entry point, and what happens on
arrival (temporary flags cleared, player placed). Loading the room's files is on floor-resources.md (not yet published); full-screen
menus that push and pop rooms are on menu-rooms.md. Code: `src/game/floor.c` (`floorLink`, `floorLinkWithSE`,
`floorChangePos`, the arrival setup), `src/game/gs_floor_data.c` (`floorRequestChange`). **Verified** from the
decomp's C unless marked.

## The calls

| Call | Used for | Destination given as |
|---|---|---|
| `floorLink(room, entry)` | most room changes in scripts (doors, story jumps) | room id + **entry-point number** |
| `floorLinkWithSE(room, entry, sound)` | exit planes (common script function 4) | the same, plus the door sound 0x3F9 unless `sound` is 0 |
| `floorChangePos(room, facing, x, y, z)` | a change to an exact spot | room id + position and facing in degrees |
| `floorRequestChange(room)` | the low-level request all of the above end in; also used directly (movies, the title, the save resume) | room id only (the player is placed wherever the link record says) |
| `std098_mapWarpToMenuRoom(room)` | full-screen menus (bag, shop, PC, ...) | a **push**, not a change: the room underneath is kept (menu-rooms.md) |

Every request also installs the standard **fade scripts** (common script functions 9 and 8, the leaving and
arriving fades; which is which is **likely** from the argument order, common-script.md). A request made while the room is running stops its script threads first.

## The link record (`lbl_80408378`, proposed `gFloorLink`)

| Offset | Field |
|---|---|
| +0x00 | the room we come from (`floorGetPrevFloorID`) |
| +0x04 | the room we go to (`floorGetNextFloorID`) |
| +0x08 | "take the position from an entry point" (1 for `floorLink`, 0 for `floorChangePos`) |
| +0x0C | entry-point number (`floorGetNextPosIndex`) |
| +0x10 / +0x1C | the player's start position and rotation |
| +0x48 .. +0x51 | a pending "continue" destination (room, previous room, entry), used instead of the save header's when set |

Scripts read the previous room to act on where the player came from (`floorGetPrevFloorID`: the world map's
"same place" check, the GBA link, a script that sends the player back the way they came).

## Arrival
1. **Temporary flags** (`floorEnterReset`): unless the room is restored after a menu or a battle, flag class 1
   (**per-room** flags) is cleared on every room change, and flag class 2 (**per-area** flags) when the new room is in
   a different area than the old one (area = the room record's area byte; a change out of a battle room keeps them).
   The flag classes are flags.md (not yet published)'s four storage classes.
2. **Placing the player**: if the link asked for an entry point, it is looked up in the new room's **entry-point list**
   (the room module's position list: x, y, z and facing per entry); a number past the end of the list puts the
   player at the origin facing 0. Then `heroMoveInit` places Wes there, and the partner 9 units behind him
   (partner-follow.md).
3. **The room's characters, doors and treasure** are set up (npcs.md, treasure.md; treasure as the third call is
   **likely**), then the field camera
   (field-camera.md), then the room's entry scripts run (new-game-flow.md "the five script slots").
4. A **restored** room (after a menu or a battle) skips steps 1 and 3 and keeps everyone where they were
   (menu-rooms.md "Fresh load or restored?").

**Continuing a save** (`floor.c`, the resume function): the room, previous room and entry come from the save header
(or the pending continue destination); the game requests that room with that entry, then restores the previous-room
field, so the first room behaves as if entered normally.

## For modders
- A new door or warp is a script call `floorLink(room, entry)` plus an entry point in the target room's list. Keep
  entry numbers stable: other rooms' scripts and exit planes refer to them by number.
- A bad entry number doesn't crash: the player appears at the room's origin. A validator should check every
  `floorLink` target against the target room's list.
- Per-room and per-area flags give "until you leave" state for free (puzzle states, NPCs that only appear once per
  visit); mods should use those classes instead of permanent story flags for that.

## Efficiency
- Only one record and a request id: the actual work (unloading the old room's resources, loading the new one) is
  queued and done by the room state machine over several frames (floor-resources.md (not yet published)), behind the fade.

## Open questions
- What room kind 2 is exactly (the "fights keep them" comment in the decomp; **likely** the battle rooms).
- The pending-continue fields (+0x48 .. +0x51): who sets them (`std162_fieldClearPending` clears them).
