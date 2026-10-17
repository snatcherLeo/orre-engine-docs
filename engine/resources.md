# Resource registry: naming loaded data by (group, id)

Status: **partial** (2026-10-14). Code: `src/game/gs_res.c` (GSresInit, GSresGetResource,
GSresRegisterResource, GSresAllocResource, GSresAllocResourceAlign; all matched) and
`src/game/gs_thread_hi_range_800F8268.c` (`fn_800F9210` release, matched; `fn_800F92D4` lookup by id, matched).
Every function on this page byte-matches the game, so **Data structures, Flow and the key conventions are
*verified*** (read from matching C). Usage remarks ("usually a group id") are marked *likely* inline.

## What it does

A small global table that maps a two-part key, `(key1, key2)`, to a pointer. Systems that load
data (maps, models, lights, cameras, faces) register what they loaded under a key. Other code
fetches it later by the same key instead of passing pointers around. An entry either owns memory
it allocated from [GSmem](memory.md) or just records a pointer that someone else owns.

## Data structures

Table: `lbl_8047AC5C` (pointer to an array), `lbl_8047AC60` (entry count), `lbl_8047AC58`
(the GSmem handle of the array). Each entry is 0x14 bytes:

| Offset | Field | Meaning |
|---|---|---|
| 0x00 | `u16 handle` | GSmem handle if this entry owns its memory, else 0 |
| 0x04 | `void* data` | The resource pointer; 0 means the slot is free |
| 0x08 | `u32 key1` | Usually a group or owner id (*likely*, from the callers) |
| 0x0C | `u32 key2` | Usually the item id within the group (*likely*) |
| 0x10 | `u32 val` | Extra value; for owned entries it is a release callback (see below) |

## Flow

- `GSresInit(count)`: allocates `count * 0x14` bytes from GSmem, locks it, and clears every
  entry's data pointer.
- `GSresGetResource(key1, key2)`: linear scan for a live entry with both keys; returns `data` or 0.
- `fn_800F92D4(key2)`: same scan but matches `key2` only (used as a fallback, for example in
  `fn_80176C04`: try `GSresGetResource(group, id)`, then `fn_800F92D4(id)`).
- `GSresRegisterResource(ptr, key1, key2, val)`: refuses duplicates, takes the first free slot,
  stores `handle = 0` (not owned), the pointer, the keys and `val`.
- `GSresAllocResource(size, key1, key2, val)` / `...Align(...)`: refuses duplicates, takes a free
  slot, allocates and locks GSmem memory, and records the handle so the entry owns it. Returns the
  data pointer.
- `fn_800F9210(key1, key2)` (release): finds the entry. If `val` is set, it calls it as
  `val(data, key1, key2)`, and a zero return cancels the release. Otherwise, if the entry owns a
  handle, it unlocks and frees it. Then it clears `data`.
- Two shared inline helpers (recovered 2026-10-14): `GSresFindByKey` (count down) and
  `GSresFindFree` (count up). The original code clearly used them (*verified*: the matched code needs them inline).

**Known key conventions** (from `floorReadMapPostFunc`, `src/game/field_range_80114AE0.c`): map
models are registered under `(owner, (param & 0x7FFF0000) | 0x1000 | index)` and lights under
`(owner, (param & 0x7FFF0000) | 0x1600 | index)`. Menu face textures use group `0x5C0`
(`fn_8010B5C4`).

## Disc data

Indirect: archive loaders (`floorRead*PostFunc`, HSD archive parsing) parse what they loaded and
register the public pieces (models, lights, camera, extra data) here under keys derived from the
file's resource id. Every floor read callback has the signature `(u32 resId, u32 loadMode,
u32 dataSize)` and starts by fetching the just-loaded file with `GSresGetResource(resId, loadMode)`,
so the file loader registers each loaded file under exactly those two keys (*verified*: four callbacks written with no
arguments matched only because the first two argument registers still held them).

## Changing it

- Mod hook (*likely*): wrapping `GSresGetResource` lets a mod swap any named resource (a model, a light
  set) without touching the loaders.
- Port note: the table stores pointers as `u32` and uses raw offsets. On a 64-bit host it needs a
  real struct with pointer-sized fields; the offsets above give that struct directly.
- What would break (*verified*): duplicate keys are silently refused (register returns, alloc returns NULL),
  so a mod that re-registers a key must release it first.

## Efficiency

- Fixed-size table allocated once; lookup is a linear scan (*verified*). Fine for the small counts the game
  uses, and there is no hashing overhead or allocation per lookup.
- Owned entries tie a GSmem handle's lifetime to a name, so a whole group can be released by key
  without each system tracking its own allocations (*likely* the intent).

## Open questions

- What is the table size (`GSresInit`'s count) and who calls it?
- Full list of key groups (0x1000 models, 0x1600 lights, 0x5C0 faces, ...): worth a table.
- `fn_800F9210` treats `val` as a callback for owned and unowned entries alike. What do the
  registered callbacks do (for example, `GSlightLoad` results)?
