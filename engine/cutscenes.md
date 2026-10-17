# Cutscenes (in-engine scenes)

Status: **partial** (2026-10-17). How the game stages its in-engine scenes: there is no cutscene file
format. Every scene is a **room script** that takes the camera, moves and animates characters and shows dialogue,
then hands control back. Pre-rendered videos (the opening, the logos) are a separate system (movies.md). **Verified**
from the decoded US scripts (counts over all 151 room scripts) and the camera code.

## What it does

A scene is a script function (started by a main-loop check, a talk, a trigger: events.md) that:
1. **freezes the player**: `heroMoveInitEvent` (307 uses), and switches off the automatic field camera
   (`std136_fieldClearFlag_8047ad71`, field-camera.md);
2. **takes the camera**: `GSscene_SetMode(3)` (127 uses in 38 scripts) puts the scene in scripted-camera mode;
   cameras are then **placed by absolute position and rotation** (`std040_cameraSetPosition` +
   `cameraSetRotation`, 128 uses each: a cut), **moved** over time (`cameraMovePositionXYZ`, 81), **aimed at a
   character** (`std170_cameraFocusOnCharacter`, 120; `cameraSetTarget`, 46), or driven by a **camera animation**
   from the room's archive (`cameraPlayAnime` + `cameraWaitSyncAnime`, 18; the `_cam` entries such as the TV news
   cameras, animation-tracks.md);
3. **acts it out**: characters walk (`std061_characterMoveToPosition`, 267, then `peopleMoveCheck` to wait),
   play **numbered animation slots** (`std051_characterSetAnimation(group, index, slot, ...)`, 770, with
   `peopleWaitSyncMotion` to wait for the end, 322), turn, appear and disappear (`peopleSetVisible`), and **talk**
   (`std054_characterTalk` / `std055_characterEndSpeech`: 2,295 lines of dialogue in all);
4. runs things **in parallel** when needed: other functions of the same script as tasks (`scriptExecTask`, 8 uses,
   joined with `std006_threadWaitForCompletion`): the Outskirt Stand arrival scene moves two groups at once this way;
5. **fades** between shots (`fadeSet`, 260 uses);
6. **gives control back**: sets the story flag that stops it replaying, `heroMoveTermEvent`, and
   `cameraReturn(wait, duration)` (57), which glides back to the field camera (field-camera.md).

The Outskirt Stand arrival scene (new-game-flow.md step 7) is a complete small example: two camera cuts, two
parallel walk tasks, two lines of dialogue with talk animations, a song change, three characters hidden, flag 986
set, collision barrier removed, camera returned.

## What a scene addresses (for avatars and model swaps)
- Characters by **(group, index)**: the player is group 0 index 100, the partner 101, room characters their
  placement index (npcs.md).
- **Animation slots by number**, per character model: a swapped model must keep the slot numbers its scenes use
  (player-movement.md, "What a player avatar must provide": the player is rarely animated by scripts, the partner
  often).
- **Camera positions in room coordinates**: a scene's framing assumes the original character heights; a much
  taller or shorter avatar would sit differently in fixed shots (**likely**; not tried).
- The **pre-rendered movies** can't follow a swapped model at all (movies.md).

## Changing it
- A scene is ordinary script bytecode in the room's archive: new or edited scenes need a script editor (a
  disassembler decodes them; writing them back needs an assembler). Camera placements are plain numbers in the
  script; camera animations are data in the room archive.
- A port option to **skip scenes** would run the function's flag and state changes without waiting on animations,
  moves and text: the waits (`peopleMoveCheck`, `peopleWaitSyncMotion`, the talk calls) are the natural hooks.

## Efficiency
- No video and no keyframe file per scene for most cuts: a cut is two calls with six numbers, so scenes cost almost
  nothing in disc space and load with the room's script.

## The scene modes (**verified** from every `GSscene_SetMode` caller; meanings **likely**)
One byte in the camera state (`GSscene_SetMode` / `GSscene_GetMode`):

| Mode | Set by | Meaning |
|---|---|---|
| 0 | scripts (41), `cameraReturn` | the normal field camera (the blended follow camera runs only in this mode) |
| 2, 7, 8 | the move viewer and the move-animation camera (`wazaViewer.c`, `wazaSequenceCamera.c`) | battle move cameras |
| 3 | scripts (127) | scripted scene camera (cutscenes) |
| 4 | the title, top menu, trade and Battle Mode menus | menu-room scenes with their own camera |
| 5, 6 | the debug menu's camera type (`dbgMenuCameraSetType`: 1 -> 5, 2 -> 6) | debug cameras; in mode 6 the player can't move (player-movement.md), **likely** a free-fly debug camera |

## Open questions
- Which scenes use camera animations, and what the `cam_move_demo_*` entries (world map) show.
