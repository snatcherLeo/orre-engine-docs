# Dolphin SDK (OS, DVD, CARD, ...)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-14). Nintendo's standard GameCube SDK (2.3-era, *likely*). Most of it matches the
public SDK headers and the Melee decomp (*verified* for the matched units); this page records Colosseum-specific placement and what
was learned while matching.

## What it does

The hardware layer: OS (threads, interrupts, reset), DVD (disc reads), CARD (memory card saves),
PAD (controllers), GX (graphics), AI/AR/DSP (audio), THP (video). Everything above it goes through
these calls, so for a PC port **this is the layer to replace** (*likely*).

## Data structures

- **`__CARDBlock`** (`lbl_803FC620`, 0x220 bytes = two 0x110-byte `CARDControl` blocks, one per
  memory card slot). Owned by the CARDBios unit (*verified*) (`src/dolphin/sdk_range_800AF8A0.c`); the CARD
  code in other units reaches it as `extern` (`&lbl_803FC620[chan]`).
- **Apploader header** (`lbl_803FB820`, the SDK's `Header` in OSReboot.c; 0x20 bytes, 32-byte
  aligned): date string, entry point, size, reboot-code size (*verified*).

## Flow

- **Reboot / reset to menu (`__OSReboot`, `src/dolphin/os/OSReset.c`; *verified*: byte-matching C).** Disables interrupts,
  re-inits DVD, waits for the drive to finish pending work (with a timeout that falls back to a hot
  reset), stops audio streaming, reads the 0x20-byte apploader header from disc offset 0x2440,
  then reads the reboot code that follows it into 0x81300000, flushes the instruction cache and
  jumps there. Every wait loop gives up after about 1 second and hot-resets instead of hanging.
- **`CARDInit`** (*verified*: byte-matching C). Runs once (both slots' `diskId` still NULL): registers the SDK version string,
  starts the DSP and alarms, initializes each slot's thread queue and timeout alarm, sets the disk
  id to the boot disc's (0x80000000) and registers a reset callback.

## Disc data

`__OSReboot` reads the disc's apploader region (0x2440) directly. Not needed for a PC port (*likely*): a
"reset" there can just restart the game's own boot path.

## Changing it

A PC port can replace whole SDK families behind the same function names (*likely* the simplest route, as other
GameCube ports do). CARD -> save files on disk; DVD -> reads from the player's disc image; OS reset ->
restart the main loop.

## Efficiency

Fixed static control blocks (no allocation); all waits are bounded by timeouts measured with
`OSGetTime` (*verified* for the two functions above).

## Open questions

- Which SDK version exactly.
