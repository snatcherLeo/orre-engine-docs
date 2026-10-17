# Crash handler and fatal errors

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

## Changing it
- For the port: route both paths to a host crash log (and a message box) instead of the TV; the report format can stay.
- The source file names in the fatal calls are a free map from code to original files (useful to decompilers).

## Open questions
- How the report is drawn (the display thread's drawing calls), and whether retail players can ever see it.
- Which context words are printed in step 3.
