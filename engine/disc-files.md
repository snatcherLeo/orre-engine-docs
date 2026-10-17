# Disc files (what the player's disc contains)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same archive format; XD's disc contents differ)

Status: **partial** (2026-10-14). Read from the disc image's file table (a walk of the
GameCube file-system table; metadata only). 

## What is on the disc (USA, GC6E01)

| Kind | Count | Notes |
|---|---|---|
| `.fsys` archives | **1,852** | Genius Sonority's archive format; nearly all game data (maps, models, scripts, messages, fonts, textures, battle data). Reader: `src/game/fsys/`. |
| `.samp` | 7 | in `sound/`: MusyX sample banks (music.md (not yet published)) |
| `.thp` | 5 | in `movie/`: the prerendered videos ([movies.md](movies.md)) |
| `.bin` | 6 | root: `colbtl.bin`, `coupon.bin` and `pkjb_test.bin` are GBA programs (a valid GBA ROM header); `bg0/1/2thumbcode.bin` are header-less e-Reader programs for the Card e battles (**verified**; gba-upload.md) |
| `.toc`, `.bnr` | 1 each | `GSfsys.toc`: archive id -> name (below); `opening.bnr`: the disc banner |

Large shared archives: `common.fsys` (743 KB), `field_common.fsys` (211 KB), `fight_common.fsys`
(194 KB), `menu_common.fsys`. The species, move, item and trainer tables (runtime pointers, see
`pokemon-data.md` (not yet published) / `trainers.md` (not yet published)) come from the type-14 module `common_rel` in `common.fsys`, whose prolog fills the table pointers
(*verified* from its relocations; [fsys-catalog.md](../colosseum/fsys-catalog.md), game-data-tables.md (not yet published)).

## FSYS archive format

Source of truth: the game's reader (`src/game/fsys/fsys_file.c`, `gs_range_8017A5FC.c`) checked
against the disc's archive headers. Community documentation (StarsMmd's
tutorials, GoD-Tool's Documentation folder) is a cross-reference only; no GPL code is used.
Big-endian throughout. Full archive list: [fsys-catalog.md](../colosseum/fsys-catalog.md). **How sure:** each field's
evidence is given in its row (code = read from the decompiled reader, disc = holds on every archive on the disc);
both count as *verified*; *guess* marks the rest.

**Archive header (0x60 bytes on common.fsys):**

| Offset | Example (common.fsys) | Field |
|---|---|---|
| 0x00 | "FSYS" | magic |
| 0x04 | 0x104 | version (guess; GSfsys.toc has the same value) |
| 0x08 | 0x0A | **archive id**, the number code uses to request it (GSfsys.toc maps id -> name) |
| 0x0C | 0x20 | **entry count** (copied to the load slot's `numEntries`) |
| 0x10 | | flags (guess) |
| 0x14 | | |
| 0x18 | | offset of the **offset block** (read by `fn_8017E30C`) |
| 0x1C | | data start |
| 0x20 | | archive size |

**Offset block** (at +0x18's value): +0x00 offset of the entry-offset table (one u32 per entry),
+0x04 offset of the name strings, +0x08 data start.

**Entry (0x50 bytes):**

| Offset | Field | Evidence |
|---|---|---|
| 0x00 | entry id: u16 id, then a byte = type x 2, then 0; the whole u32 is compared with the request id | `fn_8017E30C`; disc |
| 0x04 | data offset in the archive | disc (first entry = data start) |
| 0x08 | unpacked size | equals the LZSS header's size |
| 0x0C | flags: 0x80000000 = LZSS compressed | `fn_8017E30C` tests `flags & 0x80000000` |
| 0x14 | stored (compressed) size | equals the LZSS header's stored size; the reader allocates its read buffer from this field |
| 0x20 | resource type (index into `lbl_8036C2A0`) | contents of every type agree with the callbacks |
| 0x24 | name offset (into the name strings) | disc |
| 0x28.. | runtime area (the load state the reader keeps per entry); 0x11111111 filler on disc. The reader's view of it is `{u32 state, u32 buffer address, u32 ready}` (`FSYSSubEntry`, the slot's `currentSub` points at it), so the game writes run-time state into the loaded archive's own bytes | disc, code |

Note: the decomp's header names 0x08 `compressedSize`, 0x14 `decompressedSize` and 0x20 `groupID`; the
disc shows 0x08 is the unpacked size, 0x14 the stored size and 0x20 the type (**verified**).

**Compressed data:** a 16-byte header ("LZSS", unpacked size, stored size, 0), then LZSS with a
4,078-byte (0xFEE) window: flag bytes, bit = 1 literal, bit = 0 back-reference.

**The load slot** (`FSYSSlot`, 0x140 bytes, `fsys.h`): the archive header is
first read from the DVD straight into the slot's first 0x40 bytes (so `slot->numEntries` is the header's entry
count), then copied to the start of the TOC buffer before the rest of the TOC is read behind it. The LZSS header of
the entry being decoded is copied to a 16-byte global (`lbl_80453FDC`) and the decoder reads the stored size from it.

**GSfsys.toc:** "GLLA", 0x104, count (1,835: it lists 1,835 of the 1,852 archives, fsys-catalog.md) at +0x08, offset of the `{id, name offset}` pair table
at +0x10 (0x7C20), names from +0x20. Loaded whole at FSYS init.

## How the game reads an FSYS entry (`src/game/fsys/fsys_file.c`, `fn_8017E30C`)

1. The archive's table of contents is searched for the requested entry (by **request id**, or by a
   **name hash** via `fn_8017B07C(fileHandle, nameHash)`; `_fsysGetFilename` recovers names).
2. The entry flags' top bit (0x80000000) = **compressed**: the compressed bytes are DMA-read into a temporary buffer, the
   **16-byte LZSS header** is copied out, a **4,078-byte sliding window** is cleared, and
   `FSYSDecompressLZSS` expands the data into the final buffer (then D-cache flush).
   Uncompressed entries are DMA-read straight into their buffer.
3. The entry's **resource type** picks its callbacks from `lbl_8036C2A0`, the same table as the
   floor resource types (`floor-resources.md` (not yet published)): `pre` allocated the buffer, `post` now registers or
   relocates the data. So each FSYS entry is a typed resource (map, model, message bank, script...).
4. Memory: archives share pools; `FSYSEvictUntilSpace` frees older entries when a slot runs out of
   room (a cache, not a fixed load list).

For the port: an FSYS reader + LZSS decoder is the whole asset path; everything above it already
works on typed resources.

## The resident cache (2026-10-16, from the code; verified by the PC boot)

Some archives stay resident after they're loaded (the song archive `bgm_archive`, file handle 0x99, is preloaded at
sound setup by `fn_80165F40`). Their entries are added to a cache (`fn_8017F928`) whose blocks come from a small
first-fit free-list allocator (`fn_8017FDB0`, a classic K&R-style allocator with a sentinel node, `lbl_80465070`).
A request for an entry of a resident archive (`fn_8017AF6C` → `fn_8017E30C`) is then answered **synchronously**: the
entry is found by its id hash, looked up in the cache (`fn_8017F794`, keyed by file handle, group and id), and copied
into the buffer the request's allocation callback returns, before the call returns. The game relies on this: it starts
a song right after requesting its data. (verified at runtime.)
The allocator treats any node address at or below 0x80000000 as the end of the list, a GameCube "is this MEM1?" check;
a port whose data lives elsewhere must end the walk at NULL instead (done in the PC port (the decompiled code running natively)). (verified)

## Open questions

- Which archive holds which tables and which floors.
