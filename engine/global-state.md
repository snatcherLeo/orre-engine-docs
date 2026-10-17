# Where the game keeps its state

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; the addresses are Colosseum's)

Status: **partial** (2026-10-16). A map of the game's global state: for each system, the few globals that
hold what it is doing, what type they are and where to read more. Names here are **proposed names** (the decomp
still calls them `lbl_<address>`); the address is given so they can be found. Labels: **verified** (an accessor or a
byte-matched use proves it), **likely**, **guess**. Covers about 300 globals; the rest are constants or one-use data.

## What it does (the pattern)

Genius Sonority's code keeps state in three ways, and knowing which is used where makes the code much easier to read:

1. **One big object per system, reached through ids.** The battle is one 0xA4E8-byte object (`gFightFloor`), read and
   written only through `fightFloorGetStatus(…, id, …)` / `SetStatus`; Pokémon, trainers and moves work the same way
   (`pokemonGetStatus(pokemon, 0, id, 0)`). The status ids are the real interface (battle-turn-flow.md lists them).
2. **Table pointer + count pairs** for data from `common_rel` (species, moves, trainers, rooms, mail, sounds, ...):
   `gX` and `gXNum`, where the count is a **pointer to** the archive's count word (`u32*`), and every accessor is
   `index >= *count ? NULL : &table[index]` (rooms are searched by key instead).
3. **Small `.sbss` variables** for the current step of a state machine (which window type is open, which mail
   notice is blinking, whether the pause menu is up). These are what the proposals mostly name.

Two GameCube-specific placements: the **save block** sits at a random 32-byte offset inside its area every boot, and
the **memory-card task** at a random 8-byte-aligned offset inside a reserved area (**likely** both against
memory-editing cheats); both are reached through pointers (**likely**, not every access checked), so a port can drop
the randomness.

## Data structures, by system

### Engine core (GS)

| What | Global | Type | Label |
|---|---|---|---|
| Graphics layer state | `gGSgfx` (0x8047AA80) | `GSgfxState*` (0x5A0) | verified |
| GS-to-GX enum tables | 0x80314350-0x80314530 | `u32[]` per enum | likely |
| Matrix stack | `gGSmtxStackTop` / `Base` / `Limit` (0x8047AAC0 / ABC / AC4) | `GSMtxStackEntry*` | likely |
| Frame-buffer captures | `gGSbackFBCapture` (0x80400EE0) | 4 slots | verified |
| Models, materials, lights | `gGSmodel` (0x8047AB74), `gGSmaterial` (0x8047AB1C), `gGSlight` (0x8047AAEC) | arrays | verified / likely |
| Real-time shadows | `gGSshadowSlot` (0x80401490) | 6 slots | verified |
| Scratch memory (the locked cache at 0xE0000000) | `gGSscratchBase`, `gGSscratchUsed`, `gGSscratchAlloc` | blocks of 0x200 | verified |
| Memory manager | heap, handle table (top / bottom), live counts (memory.md) | | verified |
| Threads | `gGSthreadCurrent`, `gGSthreadNext`, the priority list | `GSThread*` | verified |
| Per-frame tasks | `gGSgappList`, `gGSgappCurrent`, the task array and slot counts | `GSgappTask*` | verified / likely |
| Resident resource cache | `gResCache`, its arena and free-list rover | | likely |
| Resources | `gGSresEntry` / `Num` | | likely |
| Archives (fsys) | `gFsysManager`, `gFsysSlot`, `gFsysFileHandle` / `Num` | | verified / likely |
| Message system | `gGSmsg` → `gGSmsgSystem` (double-buffered glyph atlas, tasks, fonts) | `MessageSystem` | verified |
| Script VM | `gGSvmPool` | `GSVMPool*` | verified |
| Collision | `gColSys` (0x80404C68) | `GSColSysState` (0x3710) | verified |
| Particles | generator lists (active / free / cursor), per-bank records, point joints | | verified |
| Input | rumble commands `gInputMotorCmd`; per-port input type (pad / **GBA as controller** / GBA lost) | | verified |
| Disc reading | `gGSdvdWork` / `Num`, the disc-error screen state `gGSdvdErrorState` (cover open, no disc, wrong disc, retry, fatal) | | verified |
| GBA uploads | the PKJB multiboot images and sizes, the random one-of-three payload (gba-upload.md) | | verified |
| Frame pacing | `gFramePaceMode` (minimum retraces per frame) | | verified |
| Crash handling | `gCrashReportText` (4 KB), `gCrashContext`, `gCrashError` and its two arguments (crash-handler.md) | | verified |

### Field (overworld)

| What | Global | Label |
|---|---|---|
| Current room (a stack: full-screen menus are rooms pushed on it, menu-rooms.md) | `gFloorContext` (0x8047ACC8), the room-change state `gFloorChangeState` (from / to room, entry point, start position, return points) | verified |
| Room loader | `gFloorThreadState`, the resource pools | verified / likely |
| Camera | `gCameraState` (script camera, saved with the scene), the field camera points and smoothed position | verified |
| Player movement | `gHeroMoveWork` (party followers, events, the **footstep counter** and 8 step callbacks) | verified |
| NPCs | `gPeopleArray` / `gPeopleMaxCount`; `gFieldMenuIsOpen` freezes them | verified |
| Pause menu, bag | `gFieldMenuChoice`, `gMenuItemHero` / `Pocket`, `gFieldItemEventList` (key items usable here) | verified / likely |
| P★DA mail | `gMailData` / `Num`, the new-mail notice `gMailNotice` | verified |
| Event log (debug) | `gEvlog` ring of 10 | verified |
| Shops | `gShopLocation` / `gShopItemList` (stock), `gShopTravelState`, the coupon-exchange undo backup | likely |
| Trading (GBA) | `gTradeSlotOwn` / `gTradeSlotPartner`, `gTradePartnerHero` (the linked game's trainer) | verified / likely |

### Battle

| What | Global | Label |
|---|---|---|
| The battle object | `gFightFloor` (0x8046DD90, 0xA4E8) | verified |
| Which battle, and its result | `gFightEncountId` (0x8047B5D6), `gFightResult` (0x8047B5D8, script command 103) | verified |
| Action queue | `gFightActionFifo` (32 x 0x30) with write / read indices; the scene and main root actions | verified / likely |
| Battle script interpreter | `gFightSeqAction`, `gFightSeqEndKind`, called move, move-learn state, the 15-step move-check counter | verified / likely |
| Timers | `gFightTimerCommand` (per turn), `gFightTimerAll` (whole battle) | verified |
| Shadow moves shown | `gDarkWazaShadowRush` / `gDarkWazaLocked` (the substituted move records) | verified |
| Presentation | idle camera, battle grid, move-animation camera and sequence work | verified / likely |

### Menus and windows

| What | Global | Label |
|---|---|---|
| Window system | `gWindowSysWork` (one record) | verified |
| Message windows | `gWinMsgCurrentType`, `gWinMsgErrorOpen` (message-windows.md) | verified |
| Party menu, summary screen | `gMenuPokemonWork`, `gMenuPokemonStatus`, `gMenuStatusWork` | likely / verified |
| P★DA | `gPdaSceneWork`, the Strategy Memo list and its temporary Pokémon, the battle list | verified / likely |
| Name entry, title | `gNameEntryResult`, `gTitleState` (`lbl_8047A31C`: the title state machine; 0x20 waits for START, 0x28 = go to the attract demo, 0xC8 = memory-card load; new-game-flow.md), `gTitleTopMenuChoice`, `gTitleOptionState` | likely / verified |
| Battle-mode menus | `gMenuTaisenWork` (0xCE60), the battle-start trainer screen | likely / verified |

### Save

| What | Global | Label |
|---|---|---|
| The save area and current save | `gSaveDataArea` (0x80408400, random offset), the current-save pointer | verified |
| Save header record | `GameDataSaveHeader` (memory-card id, save count, random save id, room, play time, return point, options) | verified |
| Story flags | the flag storage and flag-definition table (flags.md (not yet published)) | verified |
| Memory card | `gMemcardTask` (random placement; the current task's kind, state, result, buffers: memory-card.md), `gMemcardInitialized` | verified |
| Purification | `gReliveTarget`, `gReliveSelection` | verified |

## Flow
Boot creates the core state (memory, threads, tasks, graphics, message system), then the field state per room
(`gFloorContext`, the room-change state), and the battle object only while a battle runs (its own thread and action
queue). Menus allocate their work when opened and keep a few `.sbss` variables for their state machine.

## Changing it
- Port: everything in "Engine core" is the hardware-facing layer (graphics state, GX enum tables, locked-cache scratch,
  rumble, GBA input); the field, battle and menu state is portable data.
- Mods that need new state should follow pattern 1 (a new status id) or 3 (a new small global), not raw offsets.

## Efficiency
State is almost all preallocated and fixed-size (thread and task arrays, 6 shadow slots, 4 capture slots, 32 queued
actions, 8 step callbacks); hot, short-lived data goes to the locked cache. Very little is allocated per frame.

## Open questions
- About 230 more `.sbss` / `.bss` globals with 15 or more uses still have no name (mostly menu and GBA-link state).
- The meaning of several **guess** rows (P★DA mail work, some battle bytes). (The 0xB60 hero record in the NPC event
  code is the trade partner's.)
