# Memory: the GSmem handle heap

Status: **partial** (2026-10-14). Code: `src/game/gs_mem.c`, header `include/game/gs_mem.h`.
All 11 functions in `gs_mem` byte-match the game, so **everything under Data structures and Flow is
*verified*** (read from matching C). Remarks about *why* (design intent) are marked *likely* or *guess* inline.

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

**Globals:** `lbl_8047AB28` = heap mode/id passed to `GSmemInit` (1 = debug guards on);
`lbl_8047AB2C` = fit policy (first / best / worst, set by `fn_800E3560`);
`lbl_8047AB34` / `lbl_8047AB38` = top and current bottom of the entry table;
`lbl_8047AB3C` = last error code (1 bad handle, 2 out of memory, 3 no entry, 4 lock overflow,
5 unlock of unlocked, 6 checksum, 7 guard); `lbl_8047AB48`..`AB60` = statistics counters
(locks, allocations, frees, table growths).

## Flow

- `GSmemInit(heapId, start, end)`: aligns both ends to 32 bytes, puts the first table entry at
  the top, and makes the rest one big free block.
- Allocate: `_toolentryAlloc__FUl(size)` (32-byte aligned) -> `fn_800E2C04(size, alignment)`
  (first fit, or best/worst fit by policy). `fn_800E2B00` allocates from the *high* end instead
  (last block that fits, top-aligned), which helps keep long-lived and short-lived data apart (*likely* the intent).
  Both call the internal `fn_800E2DB0(address, size)`, which carves the block, takes a free table
  entry (growing the table by up to 0x4000 bytes from an adjacent free block if needed),
  optionally zero-fills and flushes the data cache, and returns the handle.
- Get pointer / lock: `fn_800E27B0(handle)` checks the handle, verifies guards and checksum in
  debug mode, increments `refCount`, returns the data pointer (+4 in debug mode, past the guard).
- Unlock: `fn_800E24B0(handle)` decrements `refCount`; when it reaches 0 in debug mode, it records
  a checksum so later writes to unlocked memory can be detected.
- Free: `fn_800E209C(handle)` warns if still locked, checks guards and checksum, inserts the block
  in the address-sorted free list, and merges it with free neighbors on both sides.
- Reverse lookup: `fn_800E202C(ptr)` scans the table for the handle that owns a data pointer.

Names used in the decomp's header: GSmemAlloc = `fn_800E2C04`, GSmemAllocRaw =
`_toolentryAlloc__FUl`, GSmemGetPtr = `fn_800E27B0`, GSmemLock = `fn_800E24B0` (it actually
*decrements*, so "unlock" fits better), GSmemFree = `fn_800E209C`.

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
  `fn_800E2C04(size, alignment)` in that order. Either its parameters are misnamed or one
  prototype is wrong (open).
- What sets `lbl_80478AF0` (zero-fill on allocate)?
- How many separate GSmem heaps exist (is `heapId` a mode or really an id)?
