# NPCs (people) in the field

Status: **partial** (2026-10-17). The runtime side of the characters standing in a room: how the game
picks who the player talks to, what happens during a conversation, and how characters block the player. Where they
stand and which scripts they have is room data (game-data-tables.md (not yet published), "Character (NPC placement) record"). Code:
`src/game/people/people.c` (`peopleFindTalkTarget`, `fn_8018CB5C`), `hero_move.c` (`updateChat`). **Verified** from
the decomp's C unless marked.

## What it does

Every character in a room, the player and the partner included, is a **person** in one pool of slots, identified by
a **group** and an **index** (the room's characters are one group; the player is group 0, index 100, and the
partner 101, as the scripts use them). Each person has a model, a position, a turn / walk state and a pointer to
its NPC info (body radius, collision type). Scripts address people by (group, index): `std054_characterTalk(group,
index, message)`, `std061_characterMoveToPosition(...)`, `peopleSetVisible(...)` and about 37 other natives in
Outskirt Stand alone (new-game-flow.md step 7).

## Choosing who to talk to

When the player presses A in the field (player-movement.md), `peopleFindTalkTarget(player, 1.2, 40)` looks at
every active person and keeps the best candidate:
- **skipped**: the player themself; people without a model; people with flag bit 1 set (**likely** hidden or
  busy); people whose placement record has **talk start type 3** (**likely** "can't be talked to with A");
  people with no NPC info;
- **reach**: the distance from the player to the other person's **head** must be within **12 units** (10 x the
  first argument) **plus both body radii** (the NPC info's radius; 4 when the player has none);
- **angle**: the person must be within **40 degrees** of where the player faces (the second argument, in
  degrees);
- **walls**: unless the placement record allows **talking through walls** (shop counters), a collision sweep from
  the player to the person must be clear;
- **item balls** (treasure records of type 1; **likely** placed as people without a placement record, since the test looks up the treasure by the person's index) can only be picked up from within **60
  degrees in front of the ball** (their own facing), and never need a clear wall test;
- **the partner** standing between the player and the person blocks the talk (`peopleIsBetween`);
- **ranking**: score = distance² x |angle| / 2 (the area of the sector between them, as the debug log calls it:
  "area"); the **smallest** wins, ties going to the larger angle, then to the positive side. So a person straight
  ahead beats a closer one off to the side.

If nobody qualifies, the examine planes of the room's collision are checked instead (signs, doors; collision.md).

## A conversation (`updateChat`)
1. The pad's pressed / repeat state is cleared so the A press doesn't carry into the text box, and the field is put
   into event mode (`heroMoveInitEventChat`).
2. The player turns to face the person's talk position.
3. An item ball runs the treasure handler (`floorEventCtrlTresure`: give the item, set its collected flag).
4. Anyone else: the person's own movement is paused, the person's **talk script** (placement record +0x14) runs
   synchronously with the person's group and index as its arguments (`cmsRunSync(script, 4, group, index, 0, 0)`;
   that is why talk scripts address "L1, L2": the speaker), then the speech bubble is closed and the person resumes.
5. Event mode ends and the pad state is cleared again.

## Setting people up on room entry (**verified** from `floor.c`)
For each placement record of the room, a person is opened (or, when the room is **restored** after a pushed room
pops, the existing one is reused and none of the following runs: menu-rooms.md, "Fresh load or restored?"). Then:
position and rotation from the record, the collision cylinder (below), visibility, the facing on load, and the
record's types:
- **movement type** (bits 0-2 of byte 0): 0 and 1 stand still (no idle behaviour set here); **2 = wander**: pick a
  random point **15 units** from its home spot, walk there, pause **5 to 8 seconds** (5 + 3 x a random fraction; counted in seconds from the frame rate, **likely**),
  pick again (`fn_80183B44`, `fn_80184D80`; a person that bumps into something turns around); **3** = only the 5-8
  second idle timer with no wander radius (**likely** turning or looking around in place, `fn_801839A0`);
- **talk start type** (byte 1 bits 6-7): 1 and 2 set people flags 0x10 and 0x20 (**likely** "turns to face the
  player" variants); 3 = can't be talked to (the talk-target test skips it, above);
- **talk end type** (byte 1 bits 4-5): 1 sets people flag 0x40 (**likely** "turns back to its original facing
  after talking"), 0 clears it;
- byte 0 bit 5: the person **looks at the player** (its head follows them, `std065_characterLookAtPlayer`).
The movement script (+0x10) runs separately (scripted walkers); its trigger is not traced yet.

## Collision with people
A person whose NPC info asks for it (collision type 1) gets a **cylinder** in the human-collision system when it is
created: the info's radius, a fixed **height of 17** (`fn_8018CB5C`). The player's movement tests against these
(player-movement.md), so NPCs block the way like walls.

## Changing it
- **Talk range and angle** are the two arguments in `updateChat` (1.2 and 40); a port "easier talking" option
  would widen them.
- **A non-talkable NPC**: talk start type 3 in its placement record (data).
- **A counter clerk** needs the talk-through-walls bit (data), or the counter's collision stops the talk.

## Efficiency
- One linear pass over the people pool per A press, with the cheap tests (flags, distance, angle) before the
  wall sweep.
- People share one fixed pool (`fn_8018E920(maxPeople)` sets its size at field start), so no allocation happens
  while walking around a room.
- The pool is one flat array (**verified** from code; `lbl_8047B200`, count `lbl_8047B1F8`) of
  **0xDC-byte** person entries (`PeopleEntry`, `include/game/people/people.h`), reached by index (`peopleGetEntry`:
  base + index × 0xDC) and walked with the same fixed stride. Each entry holds the person's model, script thread, flags,
  position and walk state. The stride is a constant in the code, not `sizeof`, so anything that changes the entry's size
  must change every walker with it (a PC port with 8-byte pointers hits exactly this: its entries grow to 256 bytes).

## Open questions
- Exactly what people flags 0x10 / 0x20 / 0x40 do (talk start / end behaviour), and people flag 1.
- Movement type 3's exact behaviour and when movement scripts (+0x10) start.
- The size of the people pool.
