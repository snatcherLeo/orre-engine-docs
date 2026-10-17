# Script VM (`cms_*`)

Status: **partial** (2026-10-14: full opcode table, file header, native table and a disassembler; checked against the opening scene). Code: `src/game/gs_thread.c` (`cms_ScriptExecute`, the
`cms_opNN_*` handlers, `cms_CallStdTrampoline`); the `std*` functions throughout `src/game` are
the native functions scripts call.

## What it does

The game's event and NPC logic runs as bytecode scripts. A script object holds an instruction
pointer and a value stack. `cms_ScriptExecute` fetches one opcode byte at a time and dispatches
it through a table of 38 handlers, until the script finishes or yields.

## Data structures

Dispatch table: `lbl_803155D0`, 38 (0x26) function pointers indexed by opcode. Names known from
symbols: `cms_op03_pop`, `cms_op05_return`, `cms_op18_negate`, `cms_op19_pushVar2`,
`cms_op28_greaterThanEqual`, `cms_op35_printf`, `cms_op36_yield`, ...

### Context fields (from the handlers; named in `GSVMCtx`, `src/game/gs_thread.c`)

| Offset | Field | Meaning |
|---|---|---|
| +0x00 | base | Pointer to the loaded script file (its header); jump targets and function entry points are offsets from here |
| +0x04 | state | State byte: 0 = invalid (logs a warning), 3 = finished, 4 = reset to 0 on exit |
| +0x08 | id | Printed in the warning |
| +0x10 | callback | Completion callback, called as `cb(obj, returnValue)` |
| +0x14 | `ip` | Instruction pointer |
| +0x18 | `globals` | Global variable array |
| +0x1C | `frame` | Current frame: stack index where the active function's arguments start |
| +0x28 | `stackCount` | Stack depth (max 0x40 = 64 entries; pushing past it logs an error and drops the value) |
| +0x6C | `stack[0x41]` | Operand stack of 32-bit values (the return value is the top entry) |

### Bytecode encoding

- Each instruction is **one opcode byte** at `ip`, followed by its operands; `ip` advances past them.
- `push immediate` (op 08): **a type byte then a 4-byte value** (type 2 = integer, 3 = float; the VM skips the type byte). `jump` (06) / `jump if false` (07): a 4-byte target,
  **relative to the script base** (`ip = base + target`); `jump if false` pops the condition.
- **Variable operands** are a descriptor byte:
  - `0` invalid (logs, uses a default value from `lbl_8047E710`);
  - bit `0x80`: the value is taken from the stack (popped);
  - otherwise an index is popped and bit `0x40` selects a **local** (`stack[frame + index]`) or, when
    clear, a **global** (`globals[index]`);
  - bit `0x20` or `0x100`: pass the variable's **address** instead of its value (references, used by
    `pushVarRef` / `copyStruct`). `setVar` (31) takes a source and a destination descriptor.
  The decoder exists both as the `GS_VM_*` macros and written out per handler.

### Calls and returns

- `call` (04) operands: u32 **function index**, u16 **argument count**, u16 **local count**. It pushes
  the return address, the old `frame` and the local count, sets
  `frame = stackCount - (args + locals + 3)` (so arguments and locals sit just above the frame), and
  jumps to `base + header.functionOffsets[index]`. The script file header has the **function count**
  at +0x04 (u16) and the **offset table** at +0x18 (u32 per function); an index past the count logs
  an error.
- `return` (05) pops the local count, the saved frame and the return address, then drops that many
  locals. A return address of 0 means the script's top level returned: state = 3 (finished).
- `callStandard` (37) calls a numbered native `stdNNN` (the `std*_...` functions across `src/game`).
  Operands: 2 skipped bytes, u16 argument count. It pushes the frame and count like `call`, takes the
  **native index** from the bottom of the argument area, and looks it up in the **native table**
  (pointer at `lbl_80478B00 + 0x10`): 12-byte records `{u32 function pointer, u8 argType[8]}`. Up to 8
  stack values are copied into two arrays by type: type 2 = float (`lbl_80401BB8`, for f1-f8),
  anything else = integer (`lbl_80401BD8`, for r3-r10). `cms_CallStdTrampoline` (0x800F106C) then calls the native
  through **one prototype with 8 integer and 8 float parameters**, filled from those arrays; its
  return value replaces the native index slot. On PowerPC that is exactly how arguments reach r3-r10
  and f1-f8, so every native sees its real arguments. **Port note:** calling through a mismatched
  prototype is undefined behaviour on PC; the port needs a typed dispatcher here (the per-native
  type bytes already say which arguments are floats).
  Comparing every native's C signature with its table type bytes found two whose C argument order differed
  from the table (std170_cameraFocusOnCharacter: int, int, five floats, int; std177_characterMoveToCharacter: the float
  before the last int), harmless on the GameCube (separate integer and float registers) but wrong for a PC calling
  convention (*verified*). Several natives read arguments their table entry doesn't supply (std200_gbaConnectEvent's two
  integers, heroMoveInit, peopleOpen's third, cameraSetAnimeRate's float, mailMainSendByScrpt's two floats; *verified*
  from each function's code reading those registers): they see whatever the VM's argument arrays still hold from the
  previous native call (an original quirk, *likely* harmless). So a faithful PC dispatcher calls each native by its C
  signature, slot by slot per register kind, exactly like the GameCube trampoline.
- `yield` (36) most likely suspends the script until the scheduler runs it again (*guess*, from the name: opcode table).

## Opcode table (all 38; operand sizes verified from each `cms_opNN` handler)

Operand kinds: **imm** = u8 type + u32 value; **u32**; **d** = one variable-descriptor byte (see
"Bytecode encoding"); **u16**. "Meaning" is *verified* where the handler's behaviour was read, *guess*
where it comes from the handler's symbol name only.

| Op | Name (symbol) | Operands | Meaning | Status |
|---|---|---|---|---|
| 00 | nop | - | nothing | verified |
| 01 | exit | - | state = 3 (script finished) | verified |
| 02 | pushMinus1 | - | push -1 | verified |
| 03 | pop | 1 byte (skipped) + u16 n | drop n values | verified |
| 04 | call | u32 function, u16 args, u16 locals | call a function of this script | verified |
| 05 | return | - | return from a function (0 return address = script done) | verified |
| 06 | jump | u32 offset | ip = base + offset | verified |
| 07 | jumpFalse | u32 offset | pop; jump if 0 | verified |
| 08 | pushImmediate | imm | push the value | verified |
| 09-12 | (none) | | empty table slots; executing one would crash | verified |
| 13 / 14 | and / or | d, d | bitwise and / or | guess |
| 15 | pushVarRef | d | push a variable's address | verified |
| 16 | pushMultiVar | d, u16 n | pop a start index, push n consecutive variables | verified |
| 17 / 18 | not / negate | d | logical not / arithmetic negate | guess |
| 19 | pushVar2 | d | push a variable | guess |
| 20-24 | multiply, divide, mod, add, subtract | d, d | arithmetic on two operands | guess |
| 25-30 | lt, le, gt, ge, eq, ne | d, d | comparisons (push 0 / 1) | guess |
| 31 | setVar | d (source), d (destination) | assign | verified |
| 32 | copyStruct | d, d, u16 n | copy n values | guess |
| 33 / 34 | logical and / or | d, d | && / \|\| | guess |
| 35 | printf (symbol) | 2 bytes + u16 n | behaves like a call with n stack values (the "printf" name does not fit; open) | guess |
| 36 | yield (symbol) | 2 bytes + u16 n | same shape as 35; seen as `return op36(30)` at the end of event functions | guess |
| 37 | callStandard | 2 bytes + u16 argc | call native `stack[frame]` with the following argc-1 values | verified |

### Script file (`.scd`, resource type 7)

| Offset | Field | Status |
|---|---|---|
| 0x00 | u32 0x01000101 in every script (version? guess) | data |
| 0x04 | u16 function count | verified (`cms_op04_call` bounds check) |
| 0x06 | u16 relocation count | verified (fn_800F76E4) |
| 0x08 | u16 (0xA5 in S1_out) | unknown |
| 0x0A | u8 "relocated" flag (set to 1 by the loader, in the loaded copy) | verified |
| 0x0C | u32 offset of the relocation table | verified |
| 0x10 | u32 offset of the globals | verified |
| 0x14 | u32 link to the next loaded script (written at run time) | verified |
| 0x18 | u32 function offsets, one per function (from the file start); 0 for an empty function | verified |

The loader code and all 164 room scripts agree on this **layout**: header, function offsets (ending at the relocation table, +0x0C), the
relocation table (+0x06 entries of u32 file offsets), the globals (+0x10), then the bytecode, which starts at the first
non-zero function offset (0x70 in S1_out). Loading links the script into the VM's list and adds the file's base address
to every word the relocation table names; those words can lie in the globals or in data after the code (strings).

*For a PC port:* script files can stay big-endian: read the header through a big-endian view and the bytecode's 2-
and 4-byte operands with big-endian reads, add the base in big-endian when relocating, and swap the globals to
native order once at load (the VM and the natives read them natively).

### Native calls in practice

Every native call is compiled as: push 0 and `pushMultiVar L, 1` (the caller's handle, local 0),
push the **native number**, push the arguments, then `callStandard` with argc = 1 + argument count.
The native table is **`lbl_802E1CF0`** in `main.dol` (247 slots of 12 bytes: function pointer +
8 argument-type bytes; 246 used); its slot number is the `stdNNN` number in the symbol names.
Examples: 81 `std081_flagCheckTrue`, 104 `std104_soundSetBackgroundMusic`, 54
`std054_characterTalk(object, character, message)`, 102 `fightStartEncount(battle)`,
152 `fieldStartEncount(battle, ...)`, 157 `std157_sleep(seconds)`.
**Naming error found:** `std078_flagSetFalse` actually sets the flag (writes 1) and
`std079_flagSetTrue` clears it (they differ only in that constant; *verified*).

### Reading scripts

With the opcode table, the header and the native table above, a disassembler can turn each script into a
listing: function and jump labels, native calls rebuilt from the stack code as `name(args)`, assignments, and
`if !(cond) goto label`, with message ids resolved to text.

**Check against the opening (Outskirt Stand, `S1_out`; *verified*):** function 8 shows the camera on the two Snagem
grunts finishing their meal and talking about their catch, then hides them and sets flag 986; function 9 has Willie
running up, introducing himself, a yes/no battle prompt that loops until accepted, walking Wes to the spot, and
`fieldStartEncount(10, 1, 0)` (battle 10), then his line after he loses. Function 0 starts the area music: silence
(`null_bgm_song`) until flag 986 is set, then `stand_song`. This matches the scene as played, so the opcode table and
the call convention are right.

## Flow

`cms_ScriptExecute(obj)` loops: check the state (0 -> warn and stop; 3 -> mark free and stop),
fetch an opcode, log on an invalid opcode (>= 0x26), otherwise call its handler. On exit it pops
the return value, or warns if the stack is empty, then calls the completion callback.

## Disc data

Scripts are resource type 7 in each room archive and in some menu archives (175 on the USA disc),
loaded by the `floorReadScript*` callbacks. Format: "Script file" above.

## Changing it

The opcode table and the `std*` natives are the natural hooks for mods (new natives, script
debugging, tracing). Not explored yet.

## Efficiency

Scripts run cooperatively: they execute until they yield, so they cost nothing between yields.
There is an odd empty countdown loop over the depth field after each instruction; its purpose is
unknown (see below).

## Region notes

- Scripts refer to text only by message id; the words live in the message banks, so a translated
  version can share scripts (*likely*; to confirm against the Japanese disc).
- Nothing region-specific found in the opcode handlers.

## Open questions

- The real behaviour of ops 35 and 36 (named printf / yield) and of the guessed arithmetic ops.
- The header words at +0x00 and +0x08.
- What the post-instruction countdown loop on +0x28 is for (compiled from `for (d--; d >= 0; d--) {}`
  in the current C; possibly a decompilation artifact).
- Thread scheduling: how yielded scripts resume (`gs_thread*.c`, gapp tasks).

## Original bugs found in script natives (port notes)

- `scriptAddItem(itemId, count)` (matched 2026-10-14): adds `count` items when positive, removes `-count` when
  negative, and with `count == 0` returns an **uninitialised** value: r0 still holds the caller's return address, 0x800F10D4 (the VM's native
  call), verified from the target assembly. A PC port has to pick a value there (whether any caller uses it is not checked yet).
