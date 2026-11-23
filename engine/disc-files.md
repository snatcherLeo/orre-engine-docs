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
`pokemon-data.md` / `trainers.md` (not yet published)) come from the type-14 module `common_rel` in `common.fsys`, whose prolog fills the table pointers
(*verified* from its relocations; [fsys-catalog.md](../colosseum/fsys-catalog.md), game-data-tables.md (not yet published)).

**Content inside the main program (`main.dol`) itself** (from the code that uses each table;
**verified** unless marked). Besides code and engine tables, the main program carries a little game content directly,
outside the archives:
- an embedded **message bank**, 684 messages (about 73 KB, ids 0x1 to 0x44EE), opened by `fn_800057B0` like the
  archive banks. *Decoded 2026-11-01 (**verified**, with the message reader on the bank's bytes):* a mix of menu and
  interface labels, the memory-card and save messages, the move descriptions, Battle Mode's menu text (with a few
  untranslated Japanese lines left in the USA game), and some story and P★DA mail lines;
- three embedded **textures** in the type-9 format ([textures.md](textures.md)) that the menu code loads and registers
  as shared resources (about 65, 65 and 150 KB). *Decoded 2026-11-01 (**verified** by decoding them):* the 640x480
  one (CMPR) is the **Nintendo logo screen** of the boot sequence, the red logo on black; the 156x103 one (RGBA8) is
  a sheet of menu parts (window corner pieces, a Poké Ball, cursor arrows); the 186x87 one (RGBA8) a rounded window
  frame with a small cursor icon;
- the save file's **icon and banner** images (about 8.5 and 6 KB), written into the memory-card file with each save:
  the banner is the "Pokémon Colosseum" logo on black (the picture an emulator's memory-card manager shows for the
  save), the icon an animation of Espeon and Umbreon (**verified** by decoding; formats in save-data.md);
- the battle **move scripts** and their tables (about 150 KB), the bytecode the battle engine interprets.
The species, move, item and trainer data are *not* in the main program: they come from `common_rel` (above).

## FSYS archive format

Source of truth: the game's reader (`src/game/fsys/fsys_file.c`, `gs_range_8017A5FC.c`) checked
against the disc's archive headers. Community documentation (StarsMmd's
tutorials, GoD-Tool's Documentation folder) is a cross-reference only; no GPL code is used.
Big-endian throughout. Full archive list: [fsys-catalog.md](../colosseum/fsys-catalog.md). **How sure:** each field's
evidence is given in its row (code = read from the decompiled reader, disc = holds on every archive on the disc);
both count as *verified*; fields left blank are not identified yet.

**Archive header (0x60 bytes on common.fsys):**

| Offset | Example (common.fsys) | Field |
|---|---|---|
| 0x00 | "FSYS" | magic |
| 0x04 | 0x104 | the same value as in GSfsys.toc |
| 0x08 | 0x0A | **archive id**, the number code uses to request it (GSfsys.toc maps id -> name) |
| 0x0C | 0x20 | **entry count** (copied to the load slot's `numEntries`) |
| 0x10 | | |
| 0x14 | | |
| 0x18 | | offset of the **offset block** (read by `fsysLoadEntrySync` (0x8017E30C)) |
| 0x1C | | data start |
| 0x20 | | archive size |

**Offset block** (at +0x18's value): +0x00 offset of the entry-offset table (one u32 per entry),
+0x04 offset of the name strings, +0x08 data start.

**Entry (0x50 bytes):**

| Offset | Field | Evidence |
|---|---|---|
| 0x00 | entry id: u16 id, then a byte = type x 2, then 0; the whole u32 is compared with the request id | `fsysLoadEntrySync` (0x8017E30C); disc |
| 0x04 | data offset in the archive | disc (first entry = data start) |
| 0x08 | unpacked size | equals the LZSS header's size |
| 0x0C | flags: 0x80000000 = LZSS compressed | `fsysLoadEntrySync` (0x8017E30C) tests `flags & 0x80000000` |
| 0x14 | stored (compressed) size | equals the LZSS header's stored size; the reader allocates its read buffer from this field |
| 0x20 | resource type (index into `lbl_8036C2A0`) | contents of every type agree with the callbacks |
| 0x24 | name offset (into the name strings) | disc |
| 0x28.. | runtime area (the load state the reader keeps per entry); 0x11111111 filler on disc. The reader's view of it is `{u32 state, u32 buffer address, u32 ready}` (`FSYSSubEntry`, the slot's `currentSub` points at it), so the game writes run-time state into the loaded archive's own bytes | disc, code |

Note: the decomp's header names 0x08 `compressedSize`, 0x14 `decompressedSize` and 0x20 `groupID`; the
disc shows 0x08 is the unpacked size, 0x14 the stored size and 0x20 the type (**verified**). The loader sizes its
buffers from these two fields of each entry, so an archive whose entries have other sizes loads with no code change
(**verified** by loading the Japanese disc's larger Rui models in the US game).

**Compressed data:** a 16-byte header ("LZSS", unpacked size, stored size, 0), then LZSS with a
4,078-byte (0xFEE) window: flag bytes, bit = 1 literal, bit = 0 back-reference.

**The load slot** (`FSYSSlot`, 0x140 bytes, `fsys.h`): the archive header is
first read from the DVD straight into the slot's first 0x40 bytes (so `slot->numEntries` is the header's entry
count), then copied to the start of the TOC buffer before the rest of the TOC is read behind it. The LZSS header of
the entry being decoded is copied to a 16-byte global (`lbl_80453FDC`) and the decoder reads the stored size from it.

**GSfsys.toc:** "GLLA", 0x104, count (1,835: it lists 1,835 of the 1,852 archives, fsys-catalog.md) at +0x08, offset of the `{id, name offset}` pair table
at +0x10 (0x7C20), names from +0x20. Loaded whole at FSYS init.

## How the game reads an FSYS entry (`src/game/fsys/fsys_file.c`, `fsysLoadEntrySync` (0x8017E30C))

1. The archive's table of contents is searched for the requested entry (by **request id**, or by a
   **name hash** via `fsysIsEntryLoaded(fileHandle, nameHash)` (0x8017B07C); `_fsysGetFilename` recovers names).
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
A request for an entry of a resident archive (`fn_8017AF6C` → `fsysLoadEntrySync` (0x8017E30C)) is then answered **synchronously**: the
entry is found by its id hash, looked up in the cache (`fn_8017F794`, keyed by file handle, group and id), and copied
into the buffer the request's allocation callback returns, before the call returns. The game relies on this: it starts
a song right after requesting its data. (verified at runtime.)
The allocator treats any node address at or below 0x80000000 as the end of the list, a GameCube "is this MEM1?" check;
a port whose data lives elsewhere must end the walk at NULL instead (done in the PC port (the decompiled code running natively)). (verified)

## Archives kept in ARAM, and the load states (from the code and from slot logs on a GameCube emulator and a native build)

A loaded archive moves through numbered states in its slot (`FSYSSlot` +0x48): 1 (asked for), 2 / 101 (header read),
150 / 152 (table of contents and data reads), 2000 (unpacking and registering the entries), 1000 (loaded). When the
game drops an archive it can keep a copy in **ARAM** (the GameCube's 16 MB auxiliary memory); a later request for it
then skips the disc. The loader copies it back by ARAM DMA, and the copy's completion interrupt
(`fsysARQCallback`) moves the slot to **state 162**; the loader acts on that at its next step, a game frame later
(**verified** in Dolphin: every reload in Willie's first battle sits in state 162 for one game frame, which is two
screen refreshes at the game's 30 frames per second, before 2000). So a reload from ARAM costs about one frame, against a few frames to tens of frames from the disc.
- (from the code and slot logs; likely) the states in more detail. 1 → 2 when the archive
  header's read completes (`fsysSlotStatusFinishRead` maps each "read in flight" state to its "read done" state:
  1 → 2, 101 → 150, 200 → 201, ...). The loader's next step starts reading one compressed entry
  (`fsysLoadEntryCompressed`: status **101**), and that read's completion gives **150**. The step after puts the entry
  into the ARAM cache (`fsysSlotStatusStartEntries`): **151** while the copy to ARAM runs, **152** when it is done (or
  straight to 152 when the entry is already cached or no cache block is free). Then 2000 and 1000 as above.
- **Timing (verified in Dolphin, the battle against Folly on one recorded route):** the loader steps once per pass of the game loop, but a read or
  ARAM copy that completes during a pass moves the slot on at once (its completion interrupt). So on the console a
  small archive can go header-done → 101 → 150 inside one frame: the move Return's attack archive went from 1 to 150
  between two samples and was loaded (1000) three game frames after its header read ended. A port that hands disc
  completions over a frame late adds a step per read, and the move's sequence then starts a frame late.
- **Why it matters:** the battle's move effects (`wzx_*`) and Pokémon models are reloaded this way between moves, so
  the frame each reload takes sets when the next move's camera and effects start, and with them which random numbers
  they draw. 
- **Engine lesson:** a second, slower memory used as an archive cache is a good way to stretch a small main memory,
  but its latency becomes part of the game's timing; a modern engine would keep such caches out of anything
  frame-exact (replays, RNG).
- **How a cached entry is found, and how one is dropped** (from the code; verified in the PC port): each
  entry is cached under three keys, the archive's id (its header's +0x08), the entry's group and the entry's name hash
  (`fn_8017F928` stores, `fn_8017F794` finds). When an archive is loaded again, every entry found there is copied back
  from ARAM (the slot shows state **161**, then 162) instead of being read from the disc; only entries not found are
  read. The game drops a whole archive's copies only to make room (`fsysCacheMakeRoom`: the oldest archive in its
  list of cached archives, `fn_8017F800` by archive id, then that list entry removed). So **the cache never notices a
  file on the disc changing**: on the console it can't, but a port or a mod that swaps a file's bytes while the game
  runs (a port's per-language files, swapped while the game runs) must
  drop that archive's copies the same way, or the old bytes come back.

## One archive load at a time, and the pending queue (from the code; verified in the PC port)
The file system works on **one archive at a time**: the manager keeps one **active slot**, and a load asked for while
another slot is active is **parked** (slot status 500, "pending") instead of starting. When the active load finishes,
the completion routine (`fn_8017D960`) closes the disc file, clears the active slot and calls `fsysStartPendingLoad`,
which starts the first parked slot, then runs the finished slot's callback.
- **Single-entry reads (load mode 3) are never restarted from the queue** (verified): `fsysStartPendingLoad` handles
  load modes 0, 1, 2 and 7 only. A parked mode-3 request (used for one entry of an already-known archive, e.g. a
  Pokémon portrait from the portrait archive, id 0x5C0) stays pending for good. Worse, the scan doesn't step past it,
  so it also hides any parked slot after it. 
- **Why the game never notices** (likely): the code that asks for single entries only does so when nothing else is
  loading. The battle's send-out asks for each lead's portrait after the battle's own archives are in, and the party
  menu asks while the field is idle. A portrait request made earlier, during the battle's own loads, is lost and the
  send-out waits forever. A port that loads portraits ahead of time (for a smoother battle start) has to wait until
  the active slot is clear before asking.
- **Engine lesson:** a single-slot loader with an incomplete pending queue only works if every caller already
  serializes its requests. A modern loader would queue every request kind, or reject what it can't queue.

## How reads behave on the console (measured in Dolphin's emulated drive)
- **One read at a time** (**verified**): the SDK's drive queue was never seen holding a second request; the game
  issues a read, waits for it, then issues the next. About **580 reads** happen from power-on to the first battle's
  command menu.
- **Large reads run at about 69 KB per frame** (~4 MB/s); **small reads are seek-bound**: a 64-byte header after a
  long seek takes **3-6 frames** (**verified**, one route). So many small reads cost far more than their size suggests.
- **A timing model** (fitted to 579 reads in Dolphin's emulated drive, not yet checked on a real console): a read takes
  about **ceil(0.8 · log2(1 + seek / 1 MB) + length / 82 KB) frames**, exact for 510 of the reads and within one frame
  for 558. The misses fit a **constant-angular-speed drive** (**likely**): large reads move 59-124 KB per frame
  depending on where they are on the disc, any short seek still costs a few frames (0.1 MB hops between sample banks:
  3-4 frames), and reads after a long idle are a little faster (an interpretation). Up to the first battle's menu the
  drive is busy for **878 of about 22,900 frames**.
- **Example:** each battle portrait is a 64-byte header read plus one 9-12 KB data read (battle-turn-flow.md, the
  entrance).
- **For a port:** a PC reads these instantly, so anything the game waits on (portraits, room loads) finishes in fewer
  frames than on the console unless the drive's timing is simulated (**likely**).

## Which reads hold up a frame (from the code, a native build's logs and Dolphin's drive)

The archive loader runs as a task in the game's main task loop: the boot thread ends in an endless loop that runs every
registered task in turn (`GSgappUpdate`), and the loader's task (`fsysUpdateTask`) steps each load slot through its
state table until the state's handler says it is done for this pass. Most handlers stop after one step, so a slot
waiting for a read just checks again on the next pass. One kind does not: right after an archive's 64-byte header
arrives, the loader reads the **table of contents** (160 or 256 bytes, just after the header on the disc), and the
handler for that state keeps polling the drive in a tight loop until the read is in (**verified**: the native build's
stack at that moment is the loader polling the drive's status from inside its task).
- **On the console** that busy wait is short: the table of contents sits right after the header, so the drive needs no
  seek, and Dolphin's drive capture shows these reads starting and ending inside the same screen refresh (**verified**,
  one recorded route from the start to Pyrite: of its small reads, about 2,800 took no refresh at all, about 200 took
  one). The frame carries on and stays one refresh long.
- **If such a read is made to end only at the next refresh** (a model that rounds every read up to a whole refresh), the
  busy wait holds the whole frame across that refresh, the frame becomes two refreshes long, and from then on the game
  logic runs a refresh behind the screen. In Willie's battle that happened at every move effect's table of contents
  (24 times in one battle), and the camera's random picks parted from the console's at the 15th shot (**verified** on a
  native build: with the reads ending inside the refresh, the battle's random state matches Dolphin's through the end
  of the battle).
- **Unpacking runs in the same loop** (**likely**): the LZSS unpacking of a large archive (`fsysLZSSDecodeJob`) is a
  task of the same loop, so a big unpack delays the next frame. When the bag menu opens, the console's frames
  right after the 744 KB menu archive arrives last 4 and 3 refreshes. A native build does the unpacking in no
  measurable time, so it needs the console's frame lengths from a recording to stay in step.
- **The loader looks once per frame** (**verified**): the main task loop makes one pass per frame (the game's own frame
  task waits for the screen refresh inside that pass), so a read that ends after the loader's turn in the pass is
  noticed in the next frame. A chain of small reads therefore costs about a frame per read (two refreshes in a 30 fps
  scene), whatever the drive's speed; only the busy-waited table of contents is noticed at once.
- **The drive reads ahead** (**verified** against Dolphin's emulated drive on two recorded routes): after a read the
  drive keeps reading the following data into its buffer while the game works, so the next read in sequence often
  takes no time at all. In one room load the console read 72, 41, 121 and 55 KB pieces one frame-pair apart, each
  within a single refresh, although the drive's raw speed is only 2-3.5 MB/s. So an archive split into many sequential
  pieces loads at the loader's pace (a frame per piece), not the drive's.
- **Engine lesson:** a loader that busy-waits inside the frame for a "small, fast" read ties the frame rate to the
  drive's latency, so a slower medium (or a different seek) silently drops frames. A modern loader would never block the
  frame on I/O: issue the read and check back next frame, or load on a worker thread.

## In later Genius Sonority games 

**Pokémon Battle Revolution** (Wii, 2007) uses the same archive system: its public decompilation (pbr-ret/pbr-decomp,
`src/gs/fsys/`, `include/gs/GSfsys.hpp`) has `GSfsys` with "FSYS" archives, a `GSfsys.toc` index, and an entry record
laid out like Colosseum's above (file id at 0x00, data offset 0x04, unpacked size 0x08, flags 0x0C with a "compressed"
bit, stored size 0x14, file type 0x20, name offset 0x24, then the runtime area), plus the same "LZSS" header and a
4,096-byte ring buffer (**verified** by reading their headers against this page; their names, this page's offsets). Two
details they add: the archive header's offsets to the entry table, names and data sit at 0x40 / 0x44 / 0x48 (the block
this page reaches through +0x18), and the LZSS header's last word holds a **CRC-32** of the data in PBR. So the file
format carried across three games; a reader written for one is a good start for the others (**likely** for XD).

## Open questions

- Which archive holds which tables and which floors.
