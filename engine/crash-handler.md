# Crash handler and fatal errors

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16). What the retail game does when the CPU faults or the code hits a fatal
check. Code: `main_tail.c` (`fn_80006250`, `fn_80006378`, `fn_800064C4`, `fn_800060F0`). **Verified** from the code and
the executable's strings unless marked; never triggered on purpose.

## What it does

The game installs **its own handler for every CPU exception** and has a printf-style **fatal error** function used by
its asserts. Both write a plain-text crash report (error name, addresses, a stack trace) into a 4 KB buffer, which a
separate thread then shows (**likely** on screen; the display path was not read).

## Data structures

| Global (proposed) | Address | What |
|---|---|---|
| `gCrashReportText` | 0x803A0700 | the report text (4 KB) |
| `gCrashContext` | 0x803A1700 | a copy of the faulting thread's CPU context |
| `gCrashError` | 0x8047A260 | the exception number (indexes the error-name table `lbl_802E2888`) |
| `gCrashArg0` / `gCrashArg1` | 0x8047A264 / 0x8047A268 | the two extra values the OS passes with the error |
| (max handler index) | 0x80478990 | 16 |

The message formats live in one string table (`lbl_80266448`): the OS error names (`OS_ERROR_DSI`, `..._ISI`,
`..._ALIGNMENT`, `..._FLOATING_POINT`, ...), the assert suffix ` in "<file>" on line <n>.` and a stack-trace header and
line format (address, back chain, saved link register).

## Flow
1. **At boot** (`fn_80006250`): `OSSetErrorHandler` for system reset, machine check, DSI, ISI, alignment, floating
   point, performance monitor, breakpoint, system interrupt and thermal interrupt; and, **only when no debugger is
   attached** (`fn_800A2C58 == 0`), also for program, system call and trace exceptions (so a developer's debugger
   keeps those).
2. **On an exception** (`fn_80006378`): save the error number and arguments, copy the context, install a recovery
   handler (`fn_800064A0` reloads a context), re-enable the scheduler and start a thread for the report.
3. **The report** (`fn_800064C4`): error name, two pairs of context words (likely the faulting address and the
   instruction address, `SRR0` / `DAR`-style; **guess** which is which), the two OS arguments, then a stack trace walked
   through the back-chain pointers.
4. **Fatal checks** (`fn_800060F0(file, line, format, ...)`): format the message, append ` in "file" on line n.`, then a
   stack trace of up to 32 frames from the current stack pointer. Callers pass their source file names, so the
   original file names survive in these calls (for example `pkjb_uploader.c`, `menuCB_Common.c`, `menuGBAC.c`).

## The original source file names (**verified**: strings in the US `main.dol`)
68 source names survive, mostly as the `file` argument of fatal checks. The game's own (Genius Sonority) ones:
`GSdvd.c`, `GCN_Mem_Alloc.c`, `gbaCommunication.c`, `pokeconv.c` (GBA <-> GameCube Pokémon conversion, **likely**),
`pkjb_uploader.c` (gba-upload.md), `menuGBAC.c` (the GBA-as-controller battle code, **likely**), `menuCB_Battle.c`,
`menuCB_Bios.c`, `menuCB_Common.c`, `menuCB_Hook.c`, `menuCB_Rule.c`, `menuCB_SaveLoad.c` (the Battle Mode menus;
`menuCB_Rule.c` holds the party-rule checks), `menuColosseumBattle.c`, `menuToolBattle.c` (`toolbattle_menu`),
`menuPokeCoupon.c` (the coupon exchange), `menuCardE.c`, `menuCardE_Matrix.c`, `cardedecode.c`, `cardesavedata.c`
(Card e), and `menuExDiscCoupon.c` / `menuExDiscShrine.c` (the `ex_coupon` / `ex_shrine` menu rooms: the Japanese
extra disc's rooms, regions.md). The rest are HSD (`aobj.c`, `cobj.c`, `jobj.c`,
`mobj.c`, `tobj.c`, ..., hsd.md), the particle system (`generator.c`, `particle.c`, `psdisp.c`, `psinterpret.c`,
`pslist.c`) and SDK files (`dvd.c`, `vi.c`, ...).

## Names recovered from the assert text (**verified**: strings in the US `main.dol`)

A fatal check prints its **condition as written in the source**, and some messages name the function they come from.
So the executable keeps a set of original **function, variable, field and constant names** that were otherwise lost
with the symbols. Only the names are listed here, not the full messages.

**Function names.** Two kinds:
- **Names that fill gaps** (the original name of a function that had none): `menuCBBios_SaveDataAvailable` and
  `menuCBBios_GetRegisteredTrainer` (the Battle Mode menus: is there save data, which trainer is registered),
  `menuCB_GetColosseumBattleTrainerID` (picks the opponent for a Colosseum battle; its message reports an
  out-of-range index), `GSfloorGetStackPointer` (the room stack depth: Battle Mode asserts it is 0, that is, no
  pushed room), `GSflagGet` (the story-flag read), `loadParticles` (loads a particle data file), `loadArchivePtr` (an
  archive load), `unloadBGM` (frees a music resource, reporting a bad sound id) and `_winGetNewWork` (allocates a
  window's work block). Which address each one belongs to is being traced function by function
  (**verified** for the names; the address ties are not all made yet).
- **Names that confirm existing ones** (the decompiled code already used the name, and the message shows it is the
  original): `_menuPush` / `_menuPop` (the menu stack, depth `m_nMenuStackDepth` in a work block `_CBC`, which asserts on overflow past 8 and underflow, menus.md (not yet published)),
  `heroBiosGetSexDataId` and `heroCheckValid` (the player record), `winSpriteAdd` (window sprites),
  `floorReadTexPostFunc` (room texture loading), `cameraWaitSyncAnime` (the camera), `loadParticle`,
  `GSthreadCreate` and `_msgGetSize` (the message code).
- The library layers add their own: HSD's `HSD_SetInitParameter` and `hsdIsDescendantOf` (the same names as in
  Melee, hsd.md), the debugger kernel's `TargetDoStep`, and the SDK's `DVDRead`, `DVDReadAsync`, `DVDSeek`,
  `DVDChangeDisk` and `DMAErrorHandler`.

**Variable, field and constant names** (from the conditions):
- **Story state:** `FLAG_system_gamemode`, the flag-variable name of the game mode (variable 0x8AE, pause-menu.md),
  and `SYSTEM_GAMEMODE_STORY`, its story value.
- **Battle Mode** (`menuColosseumBattle.c`, save section 14, save-data.md): the work block `_CB` and its fields
  `m_eBattleMode` (values `BATTLEMODE_COLOSSEUM`, `BATTLEMODE_BATTLEYAMA100` = Mt. Battle), `m_eBattleType` (value
  `BATTLETYPE_MULTI`), `m_eColosseum` (value `COLOSSEUM_BATTLEYAMA`), `m_aRule`, `m_aTrainerOrg`, and the rule fields `m_nBattleTimeLimit` / `m_nCommandTimeLimit`; the tables
  `staColosseum` and `staCPU`; the trainer-count constant `CBTRAINER_number_`.
- **Data ids:** `POKEMON_DATA_null`, `FIGHT_ENCOUNT_DATA_null`, `MSG_ID_dummy` (the "none" values of the Pokémon,
  battle and message ids) and `SEX_DATA_man` (the player's gender id: a check that the player is male).
- **Card e** (card-e.md): `CARDE_CARDTYPE_TRAINER`, the trainer states `CARDE_EX_TRAINER_STATE_APPEARING` and
  `..._ALREADY_BATTLED_WITH`, and a card series' `series_number` and `level_max`.
- **Other:** `POKECOUPONREFER_INVALID` (the coupon exchange), `pc_items_num` (a check that a PC item storage holds
  **50 or 30** slots; **likely** the GBA side, since Ruby / Sapphire / Emerald's PC holds 50 items and FireRed /
  LeafGreen's 30, while Colosseum's own count was not compared), `PS_NUM_LINK` (the particle system's link count), and HSD's frame-buffer states
  `HSD_VI_XFB_DRAWING` / `HSD_VI_XFB_WAITDONE`.

**Why it matters:** these names are the developers' own vocabulary, so tools and documentation can use them instead
of invented ones, and they confirm how the code is organised (the `menuCB` Battle Mode family, the `GS` system layer,
the `_CB` work block).

## Changing it
- For the port: route both paths to a host crash log (and a message box) instead of the TV; the report format can stay.
- The source file names in the fatal calls are a free map from code to original files (useful to decompilers).

## Open questions
- How the report is drawn (the display thread's drawing calls), and whether retail players can ever see it.
- Which context words are printed in step 3.
