# Sending programs to a Game Boy Advance (the PKJB uploader)

Status: **partial** (2026-10-16). How the game sends a program to a GBA plugged into a controller port,
so the GBA can act as a second screen or controller without a cartridge. The original source file name survives in
the executable's strings: `pkjb_uploader.c`, prefix **PKJB**. Code: `fn_80073E8C` (`PKJB_Prepare`, in
`menu_range_8007109C.c`) and the transfer helpers (`pkjb_candidate_80071AE4.c`, `menu_candidate_80074324.c`).
**Verified** from the code unless marked; nothing was watched running. For the rest of the GBA link (trading and
battle-mode parties from the GBA games) see gba-link.md.

## What it does

The GameCube can boot a GBA over the link cable by sending it a small program ("multiboot"). Colosseum keeps such
programs as plain files on the disc and sends them through the GBA serial interface. Before sending, it **stamps the
GBA program with the GameCube disc's game code** and fixes the GBA header checksum, so the GBA BIOS accepts the
program and the program can tell which GameCube game sent it (**likely** the purpose of the stamp).

## Data structures

| Global (proposed) | Address | What |
|---|---|---|
| `gPkjbImageA` | 0x803D6E40 | 128 KB buffer for the first program |
| `gPkjbImageB` | 0x803B6E40 | 128 KB buffer for an optional second program |
| `gPkjbSizeA` / `gPkjbSizeB` | 0x8047A608 / 0x8047A604 | their sizes, rounded up to 32 bytes |
| `gPkjbGameCode` | 0x8047A60C | the disc's 4-byte game code, OR-ed with 0x00202020 before stamping (**likely**: makes the last three letters lower case) |
| `gPkjbCrcTable` | 0x803FAEF8 | a 256-entry table (0x400 bytes) used for a checksum over the data sent (**likely** CRC-32) |
| per-port state | 0x803B6E08 / 0x803B6E18 | a flag and a callback per controller port (4 ports) |

**The programs on the disc** (root files, sizes from the US disc): `colbtl.bin` (92,412 bytes) and `pkjb_test.bin`
(6,948 bytes) both carry a valid GBA ROM header (an entry branch at offset 0 and the fixed byte 0x96 at 0xB2), so
they are GBA programs. `colbtl.bin` is named in `menuGBAC.c`, the battle-mode GBA code: **likely** the program a GBA
runs when used as a controller in battle mode (the menus accept a GBA as input on any port, global-state.md).
`pkjb_test.bin` is **likely** a developer test program. `coupon.bin` (59,964 bytes) is a third GBA program (valid header), but its name does not
appear in `main.dol`'s strings (**guess**: sent by another code module, perhaps the coupon features of
`menuExDiscCoupon.c` / `menuPokeCoupon.c`, or unused). The three `bg0/1/2thumbcode.bin` files have no GBA header:
they are the payload of the second kind of upload (below).

## Flow (`PKJB_Prepare(pathA, pathB)`)
1. Read the disc id's game code (`fn_800A7BCC`) and set its case bits; initialise the GBA library (`GBAInit`).
2. Open and read program A (error "PKJB_Prepare: not found" / "empty or too big size" over 128 KB / "read error").
3. **Stamp** the 4-byte code into the GBA header's game-code field (0xAC-0xAF) and recompute the GBA header
   **complement check** at 0xBD: 0xE7 minus the sum of bytes 0xA0-0xBC (the GBA BIOS refuses a program whose check
   is wrong: public GBA knowledge, not from this code).
4. The same for program B if given.
5. The transfer helpers then send the image per port in blocks (command 0x22 and friends), checking the GBA's replies
   and keys (`_PKJBUpValidateKeyB: invalid key` messages).

## The second kind of upload: a payload for a GBA game (`fn_8007B114`, `fn_8007B350`)
(**verified** from the code; what the payload does is **likely** / **guess** as marked):
- `fn_8007B114` builds the checksum table (`gPkjbCrcTable`), then picks **one of the three `bg0/1/2thumbcode.bin` files
  at random** (`OSGetTick % 3`, names from the table `lbl_802EE608`), allocates a buffer (`gMenuLoadedBuffer`,
  size `lbl_8047A650`), reads the file into it **after a 0x34-byte header gap** and zero-pads the end; `lbl_8047A64C`
  marks it loaded.
- `fn_8007B350` ("prepare the GBA upload context and worker") fills that header: the GBA device code (4 bytes), a
  **4-letter code chosen by region: "PSAJ" for Japan, "PSAE" otherwise**, a name copied in at +4, a variant byte at
  +0x29, and a checksum over the data with the table; then starts a worker thread for the transfer.
- So these files are not multiboot programs (no ROM header) but THUMB code sent to a GBA that is **already running
  a game** (**likely** the Ruby / Sapphire link used for trading and battle-mode parties, gba-link.md). Why there are
  three variants picked at random is not known (**guess**: three builds of the same payload, to make copying or
  patching the transfer harder).

## Disc data
Root files `colbtl.bin`, `pkjb_test.bin`, `coupon.bin`, `bg0thumbcode.bin`, `bg1thumbcode.bin`, `bg2thumbcode.bin` (disc-files.md).

## Changing it
- For a PC port there is no GBA to boot: the GBA-as-controller feature would be replaced by an on-screen or
  second-window menu, and these files are not needed. Emulating a linked GBA (as Dolphin can) would need the programs
  and this protocol unchanged.
- A modded GBA program must keep a valid header; the game rewrites the game code and the check itself.

## Region notes
The stamped game code is the disc's own ("GC6E" here, "GC6J" on the Japanese disc; regions.md), so the same GBA
program file can serve every region's disc (**likely**; only the US disc checked).

## Open questions
- Which caller sends which file (the caller passes paths from a work block: `gbaCommunication.c`, `p + 0x4344`).
- What the three `thumbcode` variants differ in, and what "PSA" stands for.
- The exact checksum (CRC-32 or another table checksum) and the key exchange.
