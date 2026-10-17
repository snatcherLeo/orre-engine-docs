# The field camera

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-17). How the camera follows the player while walking around a room, and how
cutscenes take it over and give it back. Code: `src/game/field_camera.c` (`fn_801171C8`, `fn_80117330`),
`floorUpdateFieldCamera` (`field_camera_candidate_80117514.c`), `camera.c` (`cameraSetFloorDefault`,
`cameraReturn`). This page covers gameplay behaviour; how the camera is rendered (the GS camera object, matrices) is
graphics (models-animation.md (not yet published), animation-tracks.md for camera shots). **Verified** from the decomp's C unless marked.

## What it does

In a room the camera sits behind and above the player, described by **three numbers**: a **height**, a
**distance** and a **yaw** (rotation around the vertical axis). Each room can give the camera a few **anchor
points**; as the player walks, the three numbers are blended from the anchors near the player, so the view turns
and pulls back smoothly as they move through the room, without any camera path being authored.

## Data structures

- **Field-camera points** (room module data, the room record's "field camera list" slot, game-data-tables.md (not yet published)):
  each point is 0x18 bytes: an anchor position (x, y, z), then the **distance**, **height** and **yaw** to use near
  it. Outskirt Stand has two (one at each end of the stand); the shop has none.
- **Per-room defaults** (`CameraFloorEntry`, `camera.c`): a default height, distance and yaw per room, set by the
  room's scripts (`cameraSetFloorDefault(height, distance, yaw)`; the Outskirt Stand shop sets its own on entry).
- **State**: the point list and its count for the current room, an "enabled" byte (`lbl_8047AD71`) and the last
  blended values.

## Flow

1. **Room entry** (`fn_80117164`): the room's point list is loaded and the field camera is **enabled**.
2. **Every frame** (`fn_801171C8`, from the floor update), in normal scene mode only:
   - **one point**: its three values are used directly (a fixed framing for the whole room);
   - **several points**: each point is weighted by **1 / (squared distance from the camera's target to the
     anchor)**, the three values are averaged with those weights, and if the target is practically on an anchor
     (squared distance under 1) that anchor's values are used outright (inverse-square blending, a simple form of
     "inverse distance weighting");
   - the result is **smoothed** over frames: new value = 0.25 x old + 0.75 x target, so the view eases toward
     the blend instead of snapping;
   - **no points**: the field camera leaves the camera alone (the room's defaults or a script's settings stay).
3. **Cutscenes** switch it off: a scene script calls `std136_fieldClearFlag_8047ad71` first (S1_out's arrival
   scene and the Willie battle both do), then places the camera itself (`GSscene_SetMode(3)`,
   `std040_cameraSetPosition`, `cameraSetRotation`, ...).
4. **Giving it back**: `cameraReturn(wait, duration)` returns the scene to normal mode and, if the room has field
   points, re-enables the field camera and **glides** the camera (target, position and rotation) to the blended
   framing over `duration` (`fn_80117330`); otherwise it restores the room's default height, distance and yaw.

**The room's authored camera** (**verified** by running the game's own code natively): when a room loads, the room
code hands the room's own camera (perspective, eye and look-at point) to the camera controller
(`cameraSetGScamera`). Without that hand-off the far clipping plane stays at the start-up value and a battle stage's
sky isn't drawn, so the authored camera also sets **how far the camera sees**.

## Disc data
- The points live in each room's module (one small list per room), read through the room record.
- The 0.25 / 0.75 smoothing and the "under 1" snap distance are constants in the executable.

## Changing it
- **Retuning a room's camera** is data: move an anchor or change its three values in the room module. Adding
  points makes the blend follow the room's shape more closely.
- **A free camera (C-stick) as a port option** would sit after step 2: add the player's yaw / distance offset to
  the blended values, and keep step 3 (cutscenes still take over).
- **Known quirk** (**likely**, from the code): the yaw is averaged as a plain number, so two anchors whose yaws lie
  on either side of the ±180° wrap would blend through the opposite direction. Rooms presumably avoid that by
  their placement.

## Efficiency
- No camera paths, collision tests or look-ahead: a handful of weighted averages per frame (a room has at most a
  few points), plus one smoothing step.

## Open questions
- What the room's defaults are used for when a room has field points (only `cameraReturn` without points uses
  them, from the code read so far).
- `fn_801174C4` / `fn_80117500` (used by `cameraReturn`): **likely** "has field points" and "re-enable".
