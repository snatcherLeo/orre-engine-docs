# How Rui follows Wes (the partner follower)

Status: **partial** (2026-10-17). The follower logic of the field: how the partner (Rui) trails the player,
keeps her distance, gets unstuck, enters rooms and behaves in cutscenes. Code: `src/game/hero_move.c`
(`partnerMove`, `partnerGetTarget`, `partnerApplyMove`, `circleLineIntersect`, `initFloor`, the member functions).
**Verified** from the decomp's C unless marked; distances are field units (Wes walks at up to 2 units per tick,
player-movement.md).

## What it does
The field has **two party members**: member 0 (the player, character group 0 index 100) and member 1 (the partner,
index 101). One of them is the **leader**, steered by the controller; the other follows. Following is not path-finding:
the follower **replays the leader's recent path** and, when that's not needed, walks straight at the leader, keeping a
fixed gap.

## Data structures (the move work `lbl_80426BD0`, proposed `gHeroMove`)
- per member: a "present" flag, a **spacing** (0 for the leader, **12** for the first follower, 24 for a second),
  a neck (head-tracking) mode, a stuck timer, footstep state;
- `leader` (0 or 1);
- the **leader's trail**: a ring of **20 positions**; a new point is stored whenever the leader is more than **12
  units** from the last stored one (so the trail covers about 240 units), newest first.

## Flow, every `heroMoveMain` pass (player-movement.md)
1. **Pick the target** (`partnerGetTarget`):
   - if there is no trail yet, or the leader is no farther than the newest trail point, or already within the
     spacing (+0.01): **walk straight at the leader** until 12 units away; if closer than **9 units**, **step back**
     away from him (so she never crowds him); the step is capped at one tick's worth of movement;
   - otherwise **walk toward the newest trail point** (one tick's step at most); if that step would bring her inside
     the 12-unit circle around the leader (`circleLineIntersect`), stop at the circle's edge. Following the trail is
     what makes her go **around corners the way Wes did** instead of cutting through walls.
2. **Move** (`partnerApplyMove`): the step goes through the normal character movement (with collision), at up to
   walking speed; she turns toward where she's heading and her walk animation follows the distance moved.
3. **Standing still**: with head tracking on, she **looks at Wes with her head**; only when he moves beyond her
   model's head-turn limits (`charModelGetHeadYawMin/Max`) does she turn her body.
4. **Getting unstuck**: if she ends a pass **more than 100 units** from the leader and stays that far for **300
   ticks**, and she isn't standing somewhere free, she is **teleported** to the first trail point where she doesn't
   collide (newest first). Within 100 units the timer resets.

## Rooms and doors
- **Joining and leaving**: `fn_8012F1FC(1)` adds her (head tracking on, spacing recomputed, model shown);
  `heroMoveDismissMember(1)` removes her. At room setup the game syncs this with the player record's status 0x18
  ("partner is with you", set and cleared by the story: two sets, one clear in the C), and **never during a Mt.
  Battle challenge run** (variable 0x8AE) (`heroMoveSyncWithHero`).
- **Entering a room** (`initFloor`): the trail is **wiped**, and she is **placed 9 units straight behind the leader**
  (opposite his facing; a second follower would be 18), facing his way, with her height snapped to the floor (any
  floor within a 10-unit step). There is **no wall check** at this point; a bad spot is left to the unstuck rule.
  When a room is **restored** after a menu room pops, she keeps her position (menu-rooms.md).
- Exits and doors don't treat her specially (**likely**: no partner case was found in the exit / door script
  functions or `floorLinkWithSE`): Wes's exit changes the room, and room entry places her behind him again.

## Cutscenes and talking
- During an event (`heroMoveInitEvent` ... `heroMoveTermEvent`) `heroMoveMain` doesn't run, so **she stops
  following**; scenes move and animate her explicitly (she's the character scripts animate most: cutscenes.md) and
  switch her head tracking off and on (`heroMoveSetNeckMode(1, 0 / 1)`, 21 uses).
- **How scenes fake following** (**verified**, the Small Tablet scene in Eagun's house, `M3_houseC_1F`): the script
  walks Wes and Rui through the **same waypoints in turn**, each `characterMoveToPosition` for one waiting on the
  other's move (`peopleMoveCheck`), so she arrives at each corner just after him; on the last leg she stops short of
  his spot, then both turn to the same angle. Scripted following is authored by hand like this, not by the follower
  code.
- She can **block talking**: if she stands between Wes and the person he faces, the talk is refused (npcs.md).
- She isn't part of battles (no battle model use; her battle slot table only holds her Shadow-detection looks).
- **Swapping the leader** is supported (`fn_8012F40C`: makes a member the leader, resets the trail; caller not
  traced; the debug menu also adds and removes members).

## For modders (follower Pokémon)
- The machinery is generic: any field character in member slot 1 follows. A **follower Pokémon** could reuse it by
  giving member 1 a Pokémon model (the model is chosen at room setup: 0x00F30400 for Rui), or by extending the two
  member slots to three (the spacing code already gives each extra follower 12 more units, and room entry already
  places each one 9 units further back).
- Things a mod must handle: the 2-slot arrays; her talk-blocking; scripts that address index 101 directly as Rui
  (cutscenes would move the Pokémon); the head-look limits come from the model's character info.
- Tuning knobs (code constants): spacing 12, minimum gap 9, trail step 12 and 20 points, unstuck 100 units / 300
  ticks, room-entry gap 9.

## Efficiency
- No path-finding: a 20-point breadcrumb trail and one circle test per tick; the stuck check costs nothing until she
  is far away for a long time.

## Open questions
- Who calls the leader swap (`fn_8012F40C`) in the retail game.
- The exact 300-tick duration in seconds (the tick is the text-speed-scaled frame step).
