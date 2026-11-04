# Memory: the GSmem handle heap

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-14). Code: `src/game/gs_mem.c`, header `include/game/gs_mem.h`.
All 11 functions in `gs_mem` byte-match the game, so **everything under Data structures and Flow is
*verified*** (read from matching C). Remarks about *why* (design intent) are marked *likely* inline.

## What it does

GSmem is Genius Sonority's own heap, layered on top of the Dolphin OS arena. Game code asks
for memory and gets back a 16-bit **handle**, not a pointer. To use the memory it asks for the
pointer (which also "locks" it), and it unlocks when done. Freeing returns the block to a free
list that merges neighbors. A debug mode adds guard words and checksums to catch overruns and
writes to memory that isn't locked.

## Data structures

**`GSmemEntry`** (16 bytes; one per allocation, stored in a table at the *top* of the heap,
growing downward):

| Offset | Field | Meaning |
|---|---|---|
| 0x00 | `u16 handle` | Its own handle (1-based index from the table top); 0 = free entry |
| 0x02 | `u16 refCount` | Lock count: incremented by get-pointer, decremented by unlock |
| 0x04 | `void* data` | Start of the allocation |
| 0x08 | `u32 size` | Allocated size in bytes (rounded, including debug guards) |
| 0x0C | `u16 align` | Requested alignment (0xFFFF for the internal path) |
| 0x0E | `u16 pad` | In debug mode: checksum of the contents while unlocked |

Handle to entry is plain arithmetic (*verified*): `entry = tableTop - (handle - 1)` (`GSmemEntryFromHandle`).

**`GSmemBlock`**: free-list node written into the free memory itself: `prev`, `next`, `size`.
The free list (`lbl_8047AB30`) is kept sorted by address.

**Globals:** `lbl_8047AB28` = the guard mode passed to `GSmemInit` (1 = debug guards on; the game passes 0);
`lbl_8047AB2C` = fit policy (first / best / worst, set by `fn_800E3560`);
`lbl_8047AB34` / `lbl_8047AB38` = top and current bottom of the entry table;
`lbl_8047AB3C` = last error code (1 bad handle, 2 out of memory, 3 no entry, 4 lock overflow,
5 unlock of unlocked, 6 checksum, 7 guard); `lbl_8047AB48`..`AB60` = statistics counters
(locks, allocations, frees, table growths).

## Flow

- `GSmemInit(heapId, start, end)`: aligns both ends to 32 bytes, puts the first table entry at
  the top, and makes the rest one big free block.
- Allocate: `_toolentryAlloc__FUl(size)` (32-byte aligned) -> `GSmemAlloc(size, alignment)` (0x800E2C04)
  (first fit, or best/worst fit by policy). `fn_800E2B00` allocates from the *high* end instead
  (last block that fits, top-aligned), which helps keep long-lived and short-lived data apart (*likely* the intent).
  Both call the internal `GSmemShrink(address, size)` (0x800E2DB0), which carves the block, takes a free table
  entry (growing the table by up to 0x4000 bytes from an adjacent free block if needed),
  optionally zero-fills and flushes the data cache, and returns the handle.
- Get pointer / lock: `GSmemLock(handle)` (0x800E27B0) checks the handle, verifies guards and checksum in
  debug mode, increments `refCount`, returns the data pointer (+4 in debug mode, past the guard).
- Unlock: `fn_800E24B0(handle)` decrements `refCount`; when it reaches 0 in debug mode, it records
  a checksum so later writes to unlocked memory can be detected.
- Free: `GSmemFree(handle)` (0x800E209C) warns if still locked, checks guards and checksum, inserts the block
  in the address-sorted free list, and merges it with free neighbors on both sides.
- Reverse lookup: `GSmemGetHandleByPtr(ptr)` (0x800E202C) scans the table for the handle that owns a data pointer.

### Boot memory split
At boot (`main.c`) the
game divides the console's free memory (the arena) in two, with **fixed numbers in code** (**verified** by reading
the code and the arithmetic):
- **GSmem gets a fixed 14.5 MB** from the bottom of the arena: `GSmemInit(0, arenaLo, arenaLo + 0xE80000)`. This is the
  only GSmem heap, and the first argument (0) is the guard mode, debug guards off.
- **The SDK heap gets the rest** (`OSInitAlloc` over `[arenaLo + 0xE80000, arenaHi)`), and **HSD allocates from it**
  (hsd.md).
- On the console the arena is `0x8048E740`-`0x817F0560` (**verified** in Dolphin: its bottom is the stack top rounded,
  its top just below the disc's file table), so the SDK heap is **3,628,544 bytes**. The boot log line "OS avail
  memory: %d" prints that heap's free bytes (3,624,320 in Dolphin).
  The top is 0x81800000 minus the US disc's file-table size (0xFA9C), rounded, so it **depends
  on the disc's file table** (other regions, or a modded file system, move it); the game's own log confirms the game
  heap at "8048E740h -> 8130E740h" (**verified**).
- Consequence: more GSmem for a mod means less SDK heap (and so less room for HSD), because the split is a constant,
  not a calculation.
- **The first GSmem blocks, in boot order** (**verified**: each address follows from the previous
  block's size rounded to 32 bytes, and the graphics buffer's address is printed by the game's own log): the log's
  text buffer (64 KB at 0x8048E740), the log's line-length table (1 KB at 0x8049E740), the thread pool (20 records of
  0x24 bytes, 720 bytes rounded to 736, at 0x8049EB40), then the graphics request buffer (450,000 bytes at
  0x8049EE20). Release mode puts no guard bytes between blocks, so a pool written past its end silently damages the
  next block's contents.

The function at 0x800E24B0 (`fn_800E24B0`) *decrements* the lock count, so "unlock" fits it; the raw allocator is
`_toolentryAlloc__FUl`.

### The anti-cheat wipe of low memory 

The game **erases the memory that cheat devices use**, again and again during start-up (**verified** from the code;
the cheat-device side is **likely**, from how Gecko-style code handlers are known to work):
- **What is erased:** the block **0x80001800-0x80003000** (unused by the game, and where a code handler such as Gecko's
  is usually installed), and the **unused tail of every CPU exception vector** (0x80000198-0x80000200,
  0x80000298-0x80000300, ... up to 0x80001798-0x80001800, the spare space after each vector's handler, where cheat
  devices place their hooks). The vector ranges come from a 16-pair table (`lbl_802663A0`, stored as offsets slightly
  off from the real bounds and corrected in code).
- **When:** six times: at the very start of `main`; at the end of the game's start-up (`mainGameThread` (0x80005AAC)); in the first
  run of the reset-countdown task (`mainClearExceptionVectors` (0x80005CE4), which also erases the vector tails); when the movie player starts
  (`fn_800366A8`); and at the start of the two logo screens (`movieTpcStart`, `movieGsLogoStart`). Each erase starts at
  a slightly different address (0x80001800, 0x80001801, 0x80001803, 0x80001805), **likely** for the same reason as
  the shifted table.
- **Effect on a console or emulator:** any code handler or hook installed before boot is erased, so cheat codes
  crash or do nothing unless a **"master code"** first disables these six erases (the public one does exactly that:
  it replaces each of the six calls with a no-op and shrinks the table's first range).
- **For a port:** nothing in the game reads these areas, so in the PC port (the decompiled code running natively) the erase is harmless and can stay as it is.

### Fixed pools and limits (for mod budgets)

What a room or a mod can't exceed on the original
hardware, because the engine sizes these once:

| Pool or limit | Size | Set where | Status |
|---|---|---|---|
| GSmem (the game's own heap: loaded archives, models, textures, scripts) | 14.5 MB, fixed | `GSmemInit` at boot (above) | **verified** |
| SDK heap (HSD's allocations) | 3,628,544 bytes on the console | the rest of the arena (above) | **verified** |
| People in a room (NPCs, the player, partner and other field characters) | **48** entries of 0xDC bytes | `peopleSystemInit(0x30)` (0x8018E920) in `main` | **verified** (npcs.md) |
| Threads | 20 records | the thread pool, at boot | **verified** |
| Room resource requests | three pools, sizes held in variables | `lbl_8047ACB4` / `ACB8` / `ACBC` (floor-resources.md (not yet published)); a full pool fails **silently** | **verified** (sizes not yet read) |
| Lights in a room's scene | 2-7 in the original rooms | the room's scene data | **verified** from the data (the engine's own maximum: to find) |
| Hardware lights per draw | 8 | the GameCube's graphics chip | **likely** (public hardware knowledge) |
| Particle generators | a pool created by `psInitGenerator(count)` | to find (the field call passes 0, so the real count is set elsewhere) | open |
| Open windows (menus, messages) | a pool of 0xB4-byte records | menus.md (not yet published) | count to find |

**What this means for mods:** a room that needs more than one of these pools holds would fail or silently drop
content on the original hardware. A PC port can enlarge most of them, but the original sizes stay the reference for
"runs like the original".

## Disc data

None directly. Everything loaded from disc (archives, models, textures, scripts) lands in GSmem
blocks; see [resources.md](resources.md) for how loaded data gets named.

## Changing it

- Hook point for a port (*likely*: follows from the verified layout): replace the allocator behind these five functions with a host allocator,
  but keep handles. Game code stores handles (16-bit) inside structs (for example
  `GSFloorResource.textureHandle`, the resource table's first field), so handles must stay small
  integers.
- What would break: anything that assumes 32-byte rounding or alignment (GX textures and
  display lists need 32-byte alignment), or the +4 debug offset if the debug mode is turned on.
- Leak and corruption hunting: the debug mode (`heapId == 1`) already has guards, checksums and
  counters. A port could turn it on and log the error codes.

### Mod surface
- **Data it reads:** none from the disc; sizes are set at start-up and per system.
- **Hooks it would need:** a host allocator behind the five functions (keeping 16-bit handles), and pool sizes read
  from a settings or limits file instead of constants.
- **Fixed-size limits:** the main arena (about 14.5 MB), handle values that fit 16 bits, and the fixed pools in the
  table above (for example 48 people per room).
- **Must stay vanilla in GameCube-accurate mode:** allocation behaviour that affects the game (what fails when a pool is
  full), so the original's limits show the same way.

## Efficiency

- *Verified* from the code: no per-allocation header inside the data in release mode: metadata lives in the separate entry
  table, so a 32-byte-aligned request gets 32-byte-aligned memory with no padding.
- Sizes round up to 32 bytes (*verified*), the GameCube cache-line size, so DMA and `DCFlushRange` work on
  whole lines (*likely* the reason).
- Freeing merges neighbors right away (*verified*), so fragmentation stays low without a compaction pass (*likely*).
- High-end allocation (`fn_800E2B00`) lets the game keep two "ends" of the heap, a common console
  trick to stop temporary loads from fragmenting long-lived data (*likely* the intent).
- Linear scans (free list, entry table; *verified*, there's no tree) are fine because counts are small (*likely*).

## Open questions

- Handles plus lock counts usually mean memory can be *moved* (compaction). No compactor has been
  found. Is the lock count only for the debug checksum, or does a defragmenter exist?
- `GSresAllocResourceAlign(align, size, ...)` passes its first two parameters to
  `GSmemAlloc(size, alignment)` (0x800E2C04) in that order. Either its parameters are misnamed or one
  prototype is wrong (open).
- What sets `lbl_80478AF0` (zero-fill on allocate)?
