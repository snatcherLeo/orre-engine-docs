# Bringing a GameCube decomp to life in the PC port (the decompiled code running natively): a playbook

This is how a matched (or nearly matched) GameCube decompilation of Pokémon Colosseum is made to run natively on a
PC, one screen at a time, written so another team (an XD decomp, another GameCube port) can follow the same path.
Claims are labelled **verified** (checked against the game's code or seen in a run), **likely** or **guess**.

## 1. The order of work, and why

| Step | What "done" means | Why this order |
|---|---|---|
| 1. **Compile** | every game, HSD and audio source file compiles with the PC compiler (GCC / MinGW) under `TARGET_PC` | cheapest step; it surfaces prototypes, intrinsics and inline assembly before anything runs |
| 2. **Link** | one executable links; whatever is still missing becomes a **logging stub** (generated) | the undefined-symbol list is the first honest measure of what the port lacks |
| 3. **Data in place** | the original program's data (tables, strings, pointer tables) generated as C from the disc's own bytes at build time, with pointers relocated | the game reads thousands of tables; hand-writing them is impossible and copying raw bytes is wrong (byte order, pointer size) |
| 4. **First boot** | `main` runs: OS, heaps, disc, threads, sound, the module loader, the script engine; no crash for the whole time limit | crashes here are almost all data-layout bugs (section 2; *likely*); fix them before chasing visuals |
| 5. **Screen by screen** | logos → title → memory card → top menu → name entry → the opening → the first room → the first battle | each screen exercises a new subsystem; stopping at the first wrong screen keeps the bug list short |

Rules that kept this tractable: the **GameCube build stays byte-identical** through every PC change (PC code lives
under `#ifdef TARGET_PC` or in PC-only headers, and every commit is checked against the original checksums and the
per-function match report); **one heavy build or boot at a time** on the machine (section 4).

## 2. Bug families, how to recognise them, how to fix them

Almost every crash and wrong screen so far belongs to one of these families. Each has a fix pattern and, where
possible, a checker that finds the next case before it crashes.

| Family | How it shows up | Fix pattern | Checker |
|---|---|---|---|
| **Big-endian disc data read natively** | absurd counts and ids (0x01000000 instead of 1), lookups that never match, a room id read byte-swapped so every room lookup fails (verified) | read through big-endian views: a `DISC_STRUCT` (GCC's `scalar_storage_order("big-endian")`) for records, `DISC_U16` / `DISC_U32` for raw reads; counts converted once at load where the code uses them natively | table-width scan (which getters return disc records); review of raw `*(u32*)(p + n)` reads of disc pointers |
| **32-bit pointer slots in GameCube-layout records** | a list walk reads 8 bytes where the record has 4: garbage pointers in window / sprite lists, menu arguments, message tasks (verified) | keep the GameCube layout and make the field a 32-bit slot (`PTR32_FIELD`, read with `P32`, write with `P32_SET`); keep everything the game sees below 4 GB so the values stay exact | a slot-walk checker (native reads at known slot offsets in list walks; part of the gate) |
| **Views of one record that disagree on its layout** | one function reads a GameCube-layout record (32-bit slots) through a private struct with native pointers: a menu model's teardown read the letters of the player's name as a light pointer (verified) | every view uses the same slot types (`PTR32_FIELD`); search for every private struct over the same global | grep for local `typedef struct` views of the same symbol |
| **Two views of one memory block with different sizes** | one file allocates and walks a pool with the GameCube size, another with the PC size: a thread pointer of 0xFB13 when a room unloads (verified) | one size for both views in the PC port, tied together with `_Static_assert`s | the static asserts; grep for second structs over the same allocation |
| **Getters that read a struct at a GameCube offset** | `*(u32*)(ctx + 0x4)` reads the upper half of a native pointer: every room's setup jobs were filed under id 0, never released, and old rooms' setup functions ran again in later rooms (verified) | read the field by name in the PC port | grep for raw offsets into structs that contain pointers |
| **Data addressed past the end of its symbol** | the GameCube compiler reaches neighbouring data as `base + offset`; in the PC port each symbol is its own array, so the read lands elsewhere: a name-entry table at +0x88 from a 12-byte symbol gave a bogus model id and a black screen (verified) | map `base + off` to the symbol that really holds that address, under `TARGET_PC` | a past-the-end checker (offsets beyond a symbol's size, including through local aliases) |
| **Scripts that carry GameCube addresses in their bytes** | battle scripts jump to addresses stored inside the script bytes and run across symbol boundaries (likely; not yet reached at run time) | planned: keep raw script regions at their GameCube addresses in the PC's memory map | none yet |
| **Leftover-register and wrong-arity calls** | the original code passes a value in a register the C prototype doesn't list; in the PC port that argument is garbage | read the real value from the original assembly; never default to 0; flag it for checking against the emulator when the assembly can't settle it | arity audit over all call sites; typed native thunks for the script engine's natives |
| **Generated call glue that trusts the wrong declaration** | a script's flag setter got a huge flag id; a character stood in a T-pose (verified, 2026-10-16): the generator that builds typed call thunks for the script natives took the first declaration it found, often a stray `extern void f(void);`, so the native ran with no arguments and read leftover registers | build glue from the function's own definition, skip bodies the PC doesn't compile (GameCube `asm`), and list every function whose declarations disagree | the generator's own conflict list (72 natives had conflicting declarations) |
| **Functions written `void` whose result is used** | text control codes that return nothing in the PC port: names and numbers vanish (verified) | restore the return value (verified from the assembly) | void-result checker |
| **Signed values turned into pointers** | a pointer kept in a signed integer sign-extends on a 64-bit host | cast through `u32` / `uintptr_t` | signed-to-pointer scan |
| **GameCube-only memory checks** | `if (p <= 0x80000000)` meaning "not in main memory" is false for every PC address | replace with the intent (`p == NULL`) under `TARGET_PC` | grep for 0x80000000 / 0x81800000 comparisons |
| **Multi-byte constants read as bytes** | a 4-byte constant read byte by byte gives the controller port masks in the wrong order: START ignored (verified) | define the constant so its bytes match in the PC port | review of byte reads of constants |
| **Byte pokes into 32-bit colours** | a colour built as `(u8*; &rgba)[3] = alpha` lands in the red channel in the PC port: opaque cyan stripes on every message window (verified) | build the colour by value | grep for `(u8*; colors)[n]` |
| **Textures the CPU rewrites** | the game writes into a texture on the CPU (the text glyph atlas, rebuilt every frame) and relies on the GameCube's cache flush; a PC graphics layer that caches textures keeps serving the first upload: every window showed the first message's letters, jumbled (verified) | the GameCube invalidates its texture cache after a CPU write; the PC renderer needs a new data version or a re-hash at that point: bump the texture's data version on every unlock (one place: the texture unlock function) | none yet; look for CPU writes between a texture lock and unlock |
| **Hand-written externs that disagree with the real prototype** | decompiled files often declare a callee themselves with `u32` stand-ins; on x86-64 the 5th and later arguments go on the stack, so a `u32` 0 passed where the callee expects a pointer or a 64-bit value arrives with garbage in its upper half: the audio library called a "callback" at 0x10000000000 when the opening movie's sound streamed (verified) | under `TARGET_PC`, declare the callee with its real signature (or include its header) | compare every hand-written `extern` against the defining header's prototype (pointer / 64-bit / float parameters, and anything past the 4th argument) |
| **Library tables sized for the GameCube** | a library's static tables are sized in bytes for 4-byte pointers; in the PC port the same byte size holds half the entries, and inserts overflow into the next table: every music note failed to find its sound macro and the title stayed silent (verified) | give the PC its own tables sized by entry count (`[N]` of the native struct), under `TARGET_PC` | grep for tables declared by byte size (`[0x800 / 8]`-style) whose entries contain pointers |
| **Thread-layer state that assumes one caller** | the GameCube runs some engine threads from an interrupt, on whatever stack was interrupted; a PC replacement that keeps one global "scheduler" context breaks when that happens: a thread yielded into another thread's stack, the scheduler's idea of the current thread went stale, and a movie decoder ran past its own suspend and read past the movie (verified) | keep such state per thread (each thread returns to the context that ran it); decide "inside a thread" from the running context, not a global | an assert at every context switch that the running stack belongs to the thread the scheduler thinks is current (cheap, silent until it fires) |
| **Game callbacks on a real host thread** | the audio mixer runs the game's stream callbacks on its own OS thread; a callback that sends a message wakes a game thread, and a scheduler that switches right away does it on the wrong OS thread: from then on stacks are saved under the wrong thread and threads resume in the wrong places, here a movie decoder past its own suspend, about one boot in three (verified with an event trace; it happened in every boot, the crash only when the timing was unlucky) | only the game's OS thread switches; a wake from any other thread leaves a pending reschedule that the game thread takes at its next interrupt point (what the console does when the interrupt ends) | a one-time log line whenever another OS thread wakes or blocks a game thread |
| **Relocations the splitter inferred on data that is really numbers** | data words that happen to look like code addresses (a menu command 0x80030000 inside a 0x8002xxxx function) carry a relocation; harmless on the GameCube (the linker writes back the same value), wrong in the PC port (the startup relocation writes a PC address there): menu cards skipped their start offset and slid into each other, font glyph words became garbage (verified) | restore the original value (target + addend) as plain big-endian bytes for that symbol and drop its relocations; the objects are RELA, so the bytes at the site are zero and simply dropping the relocation is wrong | relocations into the middle of a function (non-zero addend on a function symbol) are almost never real pointers in data |
| **Textures the CPU rewrites, never re-uploaded** | text drawn into a glyph atlas on the CPU: every window kept showing the first message's letters (verified) | the GameCube invalidates its texture cache after a CPU write; a PC renderer that caches textures by object needs a new version or a re-hash at that point (here: re-initialise the texture object's data pointer after each unlock) | look for CPU writes between a texture lock and unlock |
| **Bit-fields over GameCube-order bytes** | the GameCube compiler fills a byte's bit-fields from the top bit, GCC on x86 from the bottom: a room character shown by a script set bit 0 while the loader read bit 7, so a truck stayed invisible though its shadow drew (verified) | make the struct a big-endian view (`DISC_STRUCT`): GCC then lays bit-fields out from the top bit too; or replace the bit-field access with masks | list every native struct with bit-fields and check whether its bytes come from the disc, a save or a mask-reading neighbour |
| **Generated stubs and x86 traps** | a logging stub for an allocator corrupted the free list; integer division by zero traps on x86 where the GameCube returned a value | replace the stub with the real function; guard the division | crash reports name the function |
| **Byte-count copies of pointer groups** | a registration routine copied 0xC bytes (three GameCube pointers) into three native 8-byte pointers: the third (the state-size callback) stayed NULL and the game crashed at the first battle's save of the room state (verified: the next playtest got past it) | copy `n * sizeof(void*)`, and make every caller pass real pointers (callers that copied the pointers out of a 32-bit data table must convert each cell) | search for `memcpy` / `memset` with literal sizes into structs that hold pointers |
| **Pointer arrays filled as 4-byte cells** | the caller declared `void* list[6]`, the callee (still written with 32-bit addresses) stored 4-byte cells into it, so the caller read two cells glued into one pointer and crashed at the first battle's team selection (verified: the next playtest got past it) | declare the caller's array `u32 list[n]` in the PC port and convert each cell with `PC_PTR` | look for pointer arrays passed to functions that index them `base + i * 4` |
| **A 4-byte key read as an 8-byte pointer** | a window's work area held a 4-byte message key followed by two settings bytes; reading the key as `void*` glued the settings onto it (0x3_0000766D), caught by the 32-bit pointer check at the first battle message (likely) | read such slots with the size they were written with (`*(u32*)`), and check every read and clear of the same slot, including in sibling files | a 32-bit pointer check on every slot read (it caught this one) |
| **Private struct copies with native pointers** | one file kept its own copy of a record whose shared header uses 4-byte link fields; the copy used native pointers, so every link after the first pointer was read at the wrong offset (the move-animation cleanup at battle start; likely) | give the file's copy the shared layout (`PTR32_FIELD` with `P32` / `P32_SET`, no-ops on the GameCube) or use the shared header | look for files that declare their own version of a shared record |

## 3. Tools and what each is for

- **Safe runner:** every heavy build and boot goes through it: a machine-wide lock (one heavy job at a
  time), a time limit that kills the whole process tree, a memory cap, low priority. Added after several builds and
  boots at once froze the machine (verified).
- **The gate** (one script): full GameCube build, all checksums, the per-function match report with "no score went
  down", the PC compile probe with 0 errors and the slot-walk checker. Nothing is committed without it.
- **Data generator:** turns the original program's data and the room modules' data into
  C at build time, from the player's own disc image; converts typed fields to native order, relocates pointers, lists
  every symbol it is unsure about. The output never enters version control.
- **Stub generator** (part of the first-link script): every function the port lacks becomes a stub that logs its first
  call and caller, so the game keeps running and the log says what to write next.
- **One definition per symbol:** the matching build compiles some files several times; the PC link
  keeps the copy owned by the unit whose address range holds the symbol.
- **PC compile probe:** compiles every unit in the PC configuration and counts errors; part of the gate.
- **Diagnostics in the PC build:** scripted controller input (`ORRE_PAD_SCRIPT=frame:BUTTON,...`), frame dumps to PNG
  at chosen frames (`ORRE_DUMP_FRAMES`), a switch that skips movies so a run can get past a broken one
  (`ORRE_SKIP_MOVIES`), an audio level meter, a watchdog, a frame log, a crash handler that prints a
  decodable call stack, the game's own debug log echoed to the console, and a few PC-only log lines at room changes
  and battle start / end.
- **The platform layer underneath (what replaces the console):**
  - **Memory map:** the game keeps addresses in 32-bit slots, so everything it can see lives below 4 GB: main memory
    reserved at its GameCube address (0x80000000) before the graphics driver starts, the executable linked at a fixed
    low address without address randomisation, and a separate low region for buffers the port hands to the game.
    A start-up check (`--check`) verifies each piece.
  - **Threads on fibres:** the SDK's threads become cooperative fibres on one host thread, scheduled by the original
    rules (highest priority runs, a woken higher-priority thread takes over at once). Real host threads (audio, disc,
    graphics workers) never run game code; they post "interrupts" that the game thread takes at points where
    interrupts are enabled. The engine's own thread layer (which switched register files in assembly) runs on fibres
    too.
  - **Graphics and windowing:** an existing GX-on-WebGPU layer (Aurora) with SDL3 underneath; the port adds what it
    lacks for this game (fixed-address main memory, disc and memory-card callbacks delivered to the game thread, a
    virtual controller for scripted input, movie drawing) and keeps those changes as separate, documented commits.
- **Finding timing-dependent bugs:** printing changes the timing, and a crash seen one run in three can vanish under
  probes. What worked: a silent in-memory ring of the last few dozen scheduler switches plus a cheap consistency check
  that dumps the ring only when it fails; short repeated runs (30 seconds) with the same settings as the run that
  crashed; and judging a fix by the crash rate over several runs, labelled "likely" until it has held for a while.
- **A log file for every run, with a build stamp:** testers launch the port by hand and the console vanishes on a
  crash, so every run writes its own log file (a copy of everything printed, flushed as it goes, the newest 20 kept),
  and the first line, the window title and every crash report name the exact build (commit, uncommitted changes, build
  time). The crash report also gives the last room and frame. Asking "which build, where were you" stops being a guess.
- **Prototype scanner:** compares every hand-written `extern` in the decompiled code with the real prototype of the
  platform library it calls (pointer / 64-bit / float parameters, anything past the 4th argument). It found the two
  stack-argument crashes of this kind and one more waiting to happen; simple text matching, so it has false positives
  inside console-only `#if` blocks. It now compares against the decompiled code's own
  definitions too, skips console-only blocks, and ends with a short list of declarations that pass fewer arguments
  than the function takes and are actually called (the callee then reads leftover registers): 6 of 170 findings.
- **Audio dumps and comparison:** the port writes its mixed output to a WAV file on request, and a small tool lines it
  up with the emulator's audio dump by loudness envelope, then reports the offset, the level difference and clipping
  on both sides, and the moments where they differ most (where to listen). A sound that plays on one side only shows
  up as a large local difference.
- **Repeatable routes:** scripted button sequences that must reach a given screen (title, memory card, top menu, new
  game) with pass / fail from the log, re-run on every new build as regression checks.
- **Dolphin comparison:** the same moment in the emulator and the port, side by side (scripted Dolphin with memory
  watches for the differential tests), and a list of "eyes needed" checks for a human.

## 4. Working in parallel

- **Parallel workers with strict file ownership:** the work splits into areas (the game code on the boot path, the
  model / rendering library, the platform layer, documentation, data formats and testing); each worker edits only the
  files of its area, and anything else goes to the owner as a request.
- **One integrator:** branches are merged by one integrator only, without rewriting history, and every merge runs the
  gate.
- **One control document** holds only the current state: each area's status, the next tasks, what needs a human
  decision. History moves to an archive and decisions to a log, so the document stays short.
- **One heavy job at a time** across all workers (the safe runner's lock), with a time limit and a memory cap.
- **What worked:** small verified commits with the gate; a crash → log → fix → rerun loop of minutes; giving a bug to
  the worker that found it, with temporary access to the files it needs; documentation read ahead of the boot (it
  found the name-entry route and the yes / no default before the boot got there); turning each boot sequence into a
  regression route.
- **What didn't:** several heavy jobs at once (the machine froze); trusting a passing build as proof of matching (the
  build always matches because unmatched functions fall back to the original assembly; only the per-function report
  counts); quoting a tool's numbers without checking its flags.

## 5. Pitfalls and lessons

- Keep the original layout and change only how it is read: converting a record to a native struct means finding every
  private view of it, and there are often a dozen.
- When a value looks "uninitialised" in matched code, it is a register: trace it to the caller's assembly and give the
  PC the same value.
- A checker's false positives cost trust: exclude pointer variables from "past the end" checks.
- Shell here-documents can mangle escapes (`\n`) in generated C strings; write such edits with a real editor or a
  script file.
- Log lines at room changes and battle boundaries make every later boot run easy to read.
- A MinGW-built executable that links the C++ / thread runtime dynamically picks up whichever runtime DLLs come first
  in PATH; a different toolchain's copy fails at start (0xC0000139). Ship the matching `libstdc++`, `libwinpthread`
  and `libgcc_s` DLLs next to the executable, or link them statically.
- A game's own error message is not always a port bug: "can't open" lines for sound archives turned out to be second
  opens of archives that were already loaded (the game logs the same on the console, likely). Probe the function's
  early returns before fixing anything.
- Run the port from a scratch folder, not the repository, so frame dumps and logs never get committed; the folder
  then needs the executable's libraries on the search path.
- Debug shortcuts skip setup the game does along the normal path. A save-state style "checkpoint" loaded right after
  boot skipped the title screen, which loads sound groups the battles need, and the battle lost its cries. Let
  shortcuts wait until the normal boot has passed the screens that set things up, and compare a failing run's path
  with a working one's before suspecting the code.
- A game's warning can be routine: "Forced termination SE" in this game's sound status check also appears when an
  effect simply ended. Add your own log at the point of failure (here: the sound library's start function saying
  why an effect did not start) instead of reading meaning into the game's messages.
- Changing the memory layout changes what is left over, not only where things are: moving to the console's own arena
  bounds left the system heap with the console's free space (2.6 MB instead of 8.4 MB). Count heap peaks and failed
  allocations right after such a change; a structure that grew in the PC port fails there first.
- Mark test windows: start automated runs minimized and give test builds a distinct title, so nobody plays the
  wrong window.

## 6. What carries over to XD

XD runs on the same engine layers (the Dolphin SDK, the HSD model library, the MusyX audio library and the developer's
own framework), so the families above, the data generator, the gate and the diagnostics should carry over unchanged
(likely). Expect XD-specific work where its code differs: a rewritten script engine (its values carry their own types,
so the bridge for its native functions is different), its own tables and save layout, and its battle AI extensions.
