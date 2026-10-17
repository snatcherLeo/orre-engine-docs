# Menu rooms: full-screen menus are rooms

Status: **partial** (2026-10-16). How the game opens its full-screen scenes (the bag, party, shops, P★DA,
PC box, name entry, evolution, purification, trading, GBA link). Code: `std098_mapWarpToMenuRoom` (`gs_floor_data.c`)
and the room state machine `fn_800FF970` (`gs_floor.c`), both byte-matched, so the mechanics are **verified**; the
menu each room serves comes from its callers.

## What it does

A full-screen menu (and, **every battle**: the arena is pushed the same way) is not a window
drawn over the field: it is a **room** of its own (a "floor", like a map), with its
own resources and scripts. Opening one **pushes** a new room on a small stack of room contexts; the field room is
frozen and its state saved, but **not thrown away**, so closing the menu pops back to the exact same field state
without reloading the map. *Why (likely):* on a 24 MB machine a menu with its own models (the P★DA, evolution) needs
the memory and the frame time the field would use, so the field is suspended instead of drawn behind it; and reusing
the room system gives menus the room loader, scripts and fades for free.

## Data structures

### The menu rooms (room ids, from their callers)

| Room | Opened by | Menu |
|---|---|---|
| 0x385 | `preReliveMain` | purification (the Relic): archive `M3_shrine_1F_rl` |
| 0x386 | `evolutionOpen` | the evolution scene (`evolution`) |
| 0x387 | `fn_800884BC` (`gba_conv.c`) | `DNA_key`: the Shadow Pokémon Lab's DNA-key screen. Script command 204 (`std204_promptDNAKeyCode(a, b, c)`, used once in the Lab) turns three DNA-sample item ids (pairs 0x200-0x211) into species ids and opens it with them (`fn_800884BC(species1, species2, species3)`; **verified**) |
| 0x38F | `fn_80018F88` (bag), `menuPokemonOpenSub` (party), `menuShopOpen` | `pocket_menu`: the shared menu room for the bag, party and shops |
| 0x390 | `menuNameEntryOpen`, `...NoFade` | `name_entry_menu`: player, partner, nickname and box names ([name-entry.md](name-entry.md)) |
| 0x391 | `fn_8000CED0` (`gs_party_access.c`) | `pcbox_menu` (save-data.md, bag-and-pc.md) |
| 0x392 | `menuPdaOpen` | `pda_menu`, the **P★DA** |
| 0x393 | `std202_warpToMenuRoom393` (script command 202) | **`carde_menu`**, the Card e menu; the only script using command 202 on the US disc is in room `M1_carde_1` (Phenac City), where a person shows the player's battle record through it (**likely**) |
| 0x394 | `std198_menuEnterTradingMenu` (script command 198) | `pokemonchange_menu`, the trading menu; its only use on the US disc is in Phenac City's Pokémon Center basement (`M1_pc_B1`), right after the player saves: **trading with a linked GBA game** (**likely**) |
| 0x397 | `fn_8006A65C` (`menu_middle.c`) | `toolbattle_menu`: runs the battles of a 0x8AE challenge run (`fieldStartEncount` starts field battles through it while variable 0x8AE is set; pause-menu.md). Battle Mode's menu also switches to it (**verified**; `menuColosseumBattle`, menu 0xD1 sets a flag with `fn_8006B8FC`, then room 0x397), so it is **likely the shared battle runner** for Battle Mode and the 0x8AE runs; the battle setup is kept in save section 14 at +0xC9A8 |
| 0x39D | `fn_80097A38`, `fn_80097BBC` (`gbaCommunication.c`) | `waza_menu` (a move menu used by the GBA-link code) |
| 0x3E7 | `dbgMenuWazaDebugStart` | `waza_viewer`, the debug move viewer |
| (the arena's room) | `fightStartEncount` | **every battle**: the arena record's room is pushed the same way, so battles suspend the field like menus (**verified**, new-game-flow.md step 8) |
| 0x8F | `d2presentOpen(mode)` | `D2_present_room`, a Mt. Battle-area room pushed like a menu: the **gift-Pokémon scene**. Its caller first gives gift 4 (`std217_playerReceiveGiftPokemon(4)`, after copying the save) and passes the outcome: 0 received, 1 when the gift call returns -1, 2 when it returns -2 (the save copy is then restored); **verified** (`gba_conv.c` range); gift 4 is **Ho-Oh** (see below) |

Menu rooms are entries in the room table like any map (room 0x392 is room record 127, with its own resource set);
the names in the table are their archive names (`<name>.fsys`) from the disc's archive table (**verified**). Other rooms in the same id range are not targets of any `std098_mapWarpToMenuRoom` call in the C (so
they are entered another way, **likely** a normal room change): 0x384 `title`,
0x38E `world_map`, 0x395 `colosseumbattle_menu` (the Battle Mode menu: the title's Battle Mode choices go there, **verified**), 0x396 `colosseumbattleconnection_menu`, 0x398-0x39A
the Nintendo, Pokémon and Genius logos, 0x39B / 0x39C the fight and auto demos, 0x39E `pokecoupon_menu`, 0x39F
`staffroll`, 0x3A0 `opening_demo`, 0x3A1 `topmenu`, 0x3E4 `keydisc`.

### Gift Pokémon (**verified**, `std217_playerReceiveGiftPokemon(kind)`, script command 217)
Kind 1 Umbreon and 2 Espeon (the starters, also given directly by the New Game save), 3 Plusle, **4 Ho-Oh** (the
`D2_present_room` scene above), 5 Pikachu, 6 Celebi; each through its own `heroPokemonGet...` function (kinds 1-3
pass ball argument 0xFE, kinds 4-6 pass 0xFF). In the decoded room scripts only one gift call appears: **Plusle (kind
3) in Pyrite Town** (`M2_guild_1F_1`, the building of Duking's group, **likely**); **Ho-Oh** comes from `fn_800886D0` (`gba_conv.c` range,
**verified**): it runs only when two other checks pass (`fn_8006A76C`, `fn_8006AE18`, not traced), story flag 0x478
is set (set after the staff roll, when the previous room was 0x76 and flag 0x476 is set: `fn_80035F64`; so
**likely** "the game has been finished"), flag 0xAFD (received) is clear, and **all 48 Shadow Pokémon in the list `lbl_80375240` have their Shadow
record's "purified" flag (+0x14) set**, that is, all 48 purified (`fn_801EE398`; the flag's meaning is
**verified**: purification sets it with `fn_801EECD8(id, 1)`, battle-shadow.md);
it copies the save, gives gift 4, shows the `D2_present_room` scene, adds the Poké Coupons earned on the run to the
player's, sets flag 0xAFD, **clears variable 0x8AE** (the Battle Mode run, pause-menu.md) just before saving, and asks to save; if the
player declines and confirms not saving, the copy is restored, so the gift is undone. (That Ho-Oh is the reward for
clearing Mt. Battle in Battle Mode with all 48 Shadow Pokémon purified is public knowledge of the game; the code
fits it.) The Pikachu and Celebi gifts come from GBA-link menu code (`fn_80077ED4`,
`fn_800788BC`, `fn_800798E8` in `menu_range_8007109C.c` / `menu_exact_800798E8.c`), whose messages ask the player to
turn the GBA off and, for Celebi, say that it leaves the player again: **likely** special event features tied to the
GBA link, not traced further (no room script gives them).

### The first room at boot (**verified**, `floor.c`)
The room system's init picks the first room by the build kind (`fn_800057A8`, regions.md): retail (kind 6, the
default case) starts at 0x399 `pokemon_logo`; kind 2 (the developer build) boots straight into 0x3E7 `waza_viewer`;
kind 5 into 0x3E4 `keydisc`; kind 4 into 0x320 **`ex_title`**, the "extra disc" title (so kind 4 is **likely** the
build for an extra / bonus disc using the same executable, which fits regions.md's note that kind 4 rewrites the
disc id); kind 1 into 0x3E6 (no such room in the US table).

### State
- `gFloorContext` (current context), `gFloorContextDepth` (stack depth) and the context array (`lbl_8047ACA4`), with
  its size `lbl_8047ACA8` (set at init); a push is refused when the stack is full.
- `gFloorThreadState`: 4 = push requested, 5 = pop, 1 = loading, 2 = running.
- The room requested (`lbl_80478B18`).
- Per-system **state handlers** (`lbl_80404918`, count `lbl_8047ACE0`): each registers make / restore / size callbacks
  for one resource type (the camera's are `_cameraMakeStateData`, `_cameraRestoreStateData`, `_cameraGetStateSize`).
- Arguments and results pass through a small global block per menu (evolution: `lbl_804787E0`, six arguments in and a
  result in word 6).

## Flow

1. **Open** (`std098_mapWarpToMenuRoom(room)`): set the fade scripts (0x05960009 / 0x05960008), and if the stack has
   room, request a push (state 4) of `room`. The caller then sets the fade and yields (`floorSetFadeScript(0, 0)`,
   `_threadSwitch`): its thread then waits until the menu room ends (**likely**: the push blocks the field room's
   thread group, which the caller belongs to; not traced thread by thread).
2. **Push** (state 4): save the field room's state through every handler of its resource type into one memory block
   (`floorSaveResourceState`), block its resources and its thread group, unload its room data, remove particles,
   generators and SRT animations, free shadow textures; then move to the next context, find the menu room's data and
   start loading it (state 1, then running).
3. The menu room runs its scripts and menus (its resources load like a map's).
4. **Pop** (state 5, requested when the menu room ends: `fn_800FF660` sets the pending state to 5, which the state
   machine takes over once the room is running): release the menu room's resources and data, clear particles and shadows, move back one context,
   reload the field room's data, **restore the saved state** through the handlers (`floorLoadResourceState`), unblock
   its resources and threads, and mark it running. The caller's thread wakes up and reads the result.

**The DNA samples** (**verified** from `std204_promptDNAKeyCode`): nine sample pairs, item ids
0x200/0x201 to 0x210/0x211, stand for species 153, 156, 159, 185, 200, 287, 243, 244 and 245 (Gen 3 internal ids; for
1-251 these equal the national numbers): Bayleef, Quilava, Croconaw, Sudowoodo, Misdreavus, **Mightyena** (internal
287; the item text reads "MIGHTYENA's DNA sample", **verified**), Raikou, Entei and Suicune. The DNA door uses script
command 204 (`std204_promptDNAKeyCode`); command 208 is the analyzer / key-slot prompt (the bag opened directly,
pause-menu.md).

### Fresh load or restored? (**verified** from `gs_floor.c`)
Each room-stack level has a small context (`gFloorContext`, `GSFloorContext`); its bytes +0x0A and +0x0B record **how
the room was entered**, and `floorLoadData` gets a matching mode:
- **pushed or loaded fresh** (a push, `floorLink`): +0x0A = 1, +0x0B = 0 (mode 0);
- **restored after the room above it popped**: +0x0A = 0, +0x0B = 1 (mode 1): the room's resources come back from
  the saved resource state instead of the disc;
- otherwise mode 2.
`fn_800FF548` returns +0x0B, and room code checks it to **skip one-time setup when restored**: the script and font
loaders return early, the room-entry code skips its per-kind setup, and the top menu's pre-function
(`fn_80021644`) skips starting its song (0x446), loading its model and playing its camera animation, so returning
from Options keeps the music running instead of restarting it. The decomp's struct names the two bytes
`doFadeIn` / `doFadeOut`; from these uses, **"fresh" / "restored"** fit better (**likely**).

## Disc data
Each menu room's resources and scripts are in its room's archive (like a map's: floor-resources.md (not yet published)).

## Changing it
- A new full-screen menu (a port options screen, a mod menu) can follow the same pattern: a room of its own and a warp
  in, so the field is suspended cleanly; or, for something light, a window over the field (menus.md (not yet published)).
- Anything that must survive a menu must be saved by a state handler (or live outside the room's resources).
- The stack depth limits nesting (a menu room opening another menu room).

## Efficiency
The field is not redrawn or simulated while a menu room runs, and nothing of the map is reloaded on return: only the
room data is unloaded and the small per-system state block saved. The cost is the menu room's own load.

## Open questions
- Whether room `M1_carde_1` (and so the Card e menu) can be entered in US play.
- What results -1 / -2 of the gift call mean (likely "no room" and "failed").
- The stack size (`lbl_8047ACA8`, set from the init argument).
- Whether the windowed menus over the field (the pause menu itself, message windows) ever stack with a menu room.
