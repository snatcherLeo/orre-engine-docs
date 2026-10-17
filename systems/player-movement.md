# Player movement in the field

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-17). How the player (and the partner who follows them) walks around a
room, talks to people and triggers events. Code: `src/game/hero_move.c` (`heroMove*`, `moveLeader`,
`heroReadStick`, `partnerMove`, `updateChat`). **Verified** from the decomp's C unless marked; numbers are game
units (the floor is in the same units as collision, collision.md).

## What it does

There is no separate "player object" loop: every room's **main-loop script** calls `heroMoveMain` once per pass
(room record +0x34; new-game-flow.md step 7). Each call reads the controller, moves the party leader, lets the
partner follow, counts footsteps (field-steps.md), and runs any event the movement triggered. A script that wants
the player to stand still for a cutscene brackets it with `heroMoveInitEvent` / `heroMoveTermEvent`.

## Data structures

**The move work** (`lbl_80426BD0`, one block, proposed `gHeroMove`; fields named in the decomp's struct):
- up to **two members** (the player and the partner), each with a following-distance value and a timer; which one
  leads (`leader`);
- the **leader's trail**: a ring of **20 positions**, a new one recorded whenever the leader has moved more than
  **12 units** from the last one (so it covers about 240 units of path);
- three **event lists** (examine planes, walk-through planes and a third kind, up to 0xD0 bytes each with a count),
  filled by the movement and collision code and emptied each pass;
- an **auto-event** slot (a script id and four arguments, `heroMoveAddAutoEvent`, script native) run at the start
  of the next pass;
- a **lock-frame** counter (`heroMoveSetLockFrame`): while it is above 0 the player can't move;
- step callbacks (field-steps.md).

## Flow (one `heroMoveMain` pass)

1. **Skip** when the debug menu is open, menu 0xCA is open, the scene is in mode 6, the lock counter is running
   (it counts down), or the leader's character has a "busy" flag set.
2. **Pending auto-event**: if one is queued, run it (synchronously) and stop this pass.
3. **Dispatch queued events** to the hero-event table (`heroDispatchEvent`): first examine events (kind 3), then
   kind 2, then walk-through events (kind 1); the table maps (room, kind, id) to a script (flags.md (not yet published)). Clear the lists.
4. **Pause menu**: X, Y or START newly pressed on pad 1 opens the field menu (`fieldMenuMain`, pause-menu.md).
   (These are raw pad bits 0x400 / 0x800 / 0x1000, **likely** the SDK's X / Y / START.)
5. **The partner** moves (`partnerMove`) if present and not leading.
6. **The leader** moves (`moveLeader`):
   - **A pressed**: look for someone to talk to (`updateChat`, below); if that starts something, the leader doesn't
     move this pass.
   - **Reading the stick** (`heroReadStick`): the main stick, or the d-pad when the stick is centred (the d-pad
     counts as a full tilt). The tilt is clamped to ±56 per axis, and the **speed is the tilt's length / 28, up to
     2.0**: a light push walks, a full push goes at double speed (**verified** numbers; the walk / run animation
     split is in the character code, `peopleAmountFromMove`). The direction is taken **relative to the camera's
     yaw** (`cameraGetRotY` + the stick angle), so "up" always means away from the camera. (The camera itself: field-camera.md.)
   - The movement amount (minimum 0.2 while the stick is pushed) drives the walk animation; the leader's position
     is added to the trail every 12 units.
7. **Footsteps**: `procStep` (footstep sounds and effects for the leader), then, unless a Mt. Battle challenge run
   is active (variable 0x8AE), `proc1Step` (the step callbacks: poison, friendship, the heart gauge, mail;
   field-steps.md).

**Talking** (`updateChat`): the game looks for a character in front of the leader (`peopleFindTalkTarget(...,
1.2, 40.0)`: 12 units of reach plus both body radii, within 40 degrees of the player's facing; the full rules are in npcs.md). If there is none, it checks the collision's
examine planes instead (`heroMoveCheckEvent`; a sign, a door) and queues their events. If there is one: input is
paused, the player turns to face them, and either the **treasure** handler runs (an item ball:
`floorEventCtrlTresure`) or the person's **talk script** runs (the script id from their placement record, with
their group and id as arguments), with the person's own movement paused during it.

**The partner** (`partnerMove`): she follows the leader's recorded trail at her following distance, and a stuck
follower is teleported back onto the trail; the full follower logic is on partner-follow.md.

## Disc data
- Speeds and distances (56, 28, 2.0, 12, 100, 300, 40) are constants in the code, not data.
- The talk script, the treasure and the character's model come from the room module's character records
  (game-data-tables.md (not yet published), "per-room character slot"); event planes from the room's collision (collision.md).

## Changing it
- **Running speed / a run button**: the speed cap (2.0) and the tilt scale (28) are the two numbers; a port option
  could map a button to full speed, since the d-pad already counts as a full tilt.
- **Talk range**: the two arguments of `peopleFindTalkTarget` (npcs.md).
- **Partner catch-up**: 100 units / 300 ticks in `partnerMove`.
- Anything that should happen "when the player walks somewhere" belongs in the room's main-loop script (position
  checks) or in a walk-through plane (collision); both feed `heroMoveMain`.

## Efficiency
- No per-frame physics for the follower: she replays the leader's recorded path, and only a stuck follower is
  repositioned, by a cheap collision test against at most 20 stored points.
- Events found during movement are queued and dispatched once per pass, so a trigger plane crossed in the middle
  of the movement can't re-enter the movement code.

## What a player avatar must provide (first pass)
For a mod that swaps the player's model (an "avatar"), these are the places found so far that use the player model,
and what they ask of it. Counts are from the decoded US room scripts (**verified** counts; meanings **likely**).
- **Field walking**: the walk / run animation driven by the movement amount (`heroMoveAnimate`, through
  `peopleAmountFromMove`); turning (`peopleStartTurn`); the neck look-at that can be switched on and off by scripts
  (`heroMoveSetNeckMode`, 21 uses: the head turns toward a point, so the model needs a head / neck the code can
  address).
- **Script animation slots played on the player** (character group 0, id 100): only slots **0** and **1** (4 calls in
  all; slot 1 is **likely** "stand"), plus the **surprise** reaction (`std131_characterSurpriseAnimation`, once).
  Most cutscene acting is done by other characters: the partner (id 101) alone uses slots 1, 2, 3, 7, 14, 21-23,
  27-29 and 42-44. A player avatar is therefore cheap on script animations, while a **partner** avatar would need
  many more slots.
- **Footsteps**: the step code reads the model's foot parts to place footstep sounds and effects
  (`getStep__FP8FOOTSTEP...`, the part indices of the feet): the avatar needs the foot bones at the expected part
  indices (**likely**; field-steps.md).
- **Places the player model is chosen**: the field (the hero-move members' models), the **name-entry screen** (a
  character model per mode, name-entry.md), the **battle** (trainer record 1 is the GameCube hero, with its own
  model index: trainers.md (not yet published)), and the **pre-rendered movies**, which show the original player and can't follow a
  swapped model (movies.md). Not yet covered: the battle's trainer animations and facial expressions, the menu
  models (P★DA, summary), and the in-engine cutscene cameras that frame the player.

## Open questions
- The third event list (kind 2): which collision or code fills it.
- What sets the leader's "busy" flag (0x80000000 in `fn_8018C424`).
- How `peopleAmountFromMove` turns the amount into walk / run animations (the character code, npcs.md).
