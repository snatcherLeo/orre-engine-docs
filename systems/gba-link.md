# GBA link and Pokémon conversion

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-14). Code: `src/game/gba/` (`gba_misc.c`, `gba_conv.c`, `GBA*.c`),
`src/game/pokeconv.c`, `gbaCommunication*.c`.

## What it does

Colosseum trades Pokémon with the GBA games (Ruby/Sapphire/FireRed/LeafGreen/Emerald) over the
GameCube-GBA link cable. Pokémon are converted between the GameCube record (`PokemonBios`,
`pokemon-data.md`) and the GBA's 100-byte party format, and sent in byte-swapped packets.

## Data structures

- **`GbaPokemon`** (`src/game/gba/gba_misc.c`): the Gen 3 party Pokémon with its four encrypted
  substructures already decoded into one layout (0x64 bytes): personality and OT id, nickname, language
  and egg / bad-egg bits, OT name (7), markings, **checksum**, then growth (species, held item, EXP,
  PP bonuses, friendship), attacks (4 moves, 4 PP), EVs and condition (HP, Atk, Def, **Speed**, SpA,
  SpD, then the contest stats), misc words (IVs, ribbons, Pokérus...), and the party-only tail
  (status, level, mail id, HP and the six stats).
- **Stat order differs:** Gen 3 stores EVs as HP/Atk/Def/Speed/SpA/SpD; the GameCube record uses
  HP/Atk/Def/SpA/SpD/Speed (`PokemonStatBlock`). Any converter must reorder.
- **Link packet** (`fn_80089048`): 0x10-byte header (three words, plus a u16 payload length in words
  and a u16 from `fn_8011F5C8(pokemon)`), then the payload, **every 32-bit word byte-swapped**
  (GameCube is big-endian, GBA little-endian). Payload length is asserted to be 0x32 or 0x1E words
  (200 or 120 bytes). When a Pokémon is attached, the converted GBA Pokémon is appended, followed by
  11 bytes of extra ribbons (`exribbonGetNo(i)`, the GameCube-only ribbons).

## Flow

- GameCube -> GBA: `pokeconvPokemonToGBA(gcPokemon, gbaPokemon)` (0x8008AE18) (0xDC4 bytes), then the packet builder.
- GBA -> GameCube: `pokeconvPokemonFromGBA(gc, gbaPokemon)` (0x9F8 bytes) and `gbaPokemonConditonFromGC`.
- The attestation stamp on each Pokémon (`save-data.md`) records the origin game and language.

### How the game looks for a GBA 

Seen by logging every GBA driver call and every device probe, with the calling function, while Gang Battle and the
Phenac trade counter were opened with no GBA plugged in (**verified** on the PC port; the functions **verified** in
the code):
1. **Getting ready:** `pkjbPrepare` (the "PKJB uploader", from its error strings) starts the GBA driver
   (`GBAInit`), reads the disc ID for the boot key and loads the program a GBA will be sent from the disc.
2. **The "Game Boy Advance connection" screen** (the console, the controller and a GBA in socket 2, "You must save
   at a PKMN CENTER to connect!", "1P B Cancel") then waits in `fn_80087AE8`. That loop asks the serial port what
   is plugged into the chosen socket (`SIProbe`) until it answers **0x40000, a GBA**, and it also leaves on the B
   button (result 2, cancel) or when the message window is answered.
3. **Nothing reaches the GBA driver** until a GBA is found: with "no device" (0x8) on socket 2, no `GBAGetStatus`,
   `GBAJoyBoot`, `GBARead` or `GBAWrite` call was made. The program upload (`GBAJoyBoot`) and the data transfers
   come only after detection.

Gang Battle and the trade counter show the same connection screen and logged the same calls. So one shared module,
`pkjb_uploader` plus the wait loop, serves every GBA feature (**likely**).

**For a port** (and an emulator link later): a GBA link would have to answer the device probe with 0x40000 on the
socket, then serve the boot upload and the reads and writes. Answering "no device" keeps the game on this screen
until the player cancels, which is what the console does with nothing plugged in.

## Stats of imported Pokémon (researched 2026-10-15)

- **The importer keeps the GBA's stats (verified, `pokeconvPokemonFromGBA` (0x8008BBDC) byte-matched):** after the boxed data it copies the
  party tail as stored on the GBA: level, mail id, max HP, HP, Attack, Defense, Speed, Sp. Atk, Sp. Def. Nothing is
  recalculated on import. `pokeconv` (`fn_80089048` area) uses it to build a whole trainer (name, gender, trainer id,
  six Pokémon, the 11 GameCube-only ribbons) from a GBA block.
- **Colosseum recalculates later (verified from code):** `pokemonResetBasisStatus` recomputes all six stats from the
  species table at the current level. It runs on every EXP gain (`pokemonGrowBasisStatus`, after battles), Rare
  Candy / vitamins (`item_range_80144574`), evolution, revival of Shadow Pokémon's EXP (`_expRecover`), gift Pokémon
  and new Pokémon (`pokemonCreate`).
- **Battle-mode legality check (verified, `menuRuleCheckPokemon` (0x80076398) / `menuCBRule_CheckPokemonErrorAll`, byte-matched):** for
  each party member it copies the Pokémon, recomputes the copy's stats, and rejects the Pokémon if any stored stat is
  higher than the recomputed one, if the EVs total more than 510 or one EV is above 255, if it is Mew or Deoxys
  without the event flag, a Shadow Pokémon, an egg, or has an invalid move or data. So stats from another game's
  formula (a Deoxys Forme) or edited stats can't be used in battle mode. That the GBA-loaded parties of battle mode go
  through this same check is likely (the per-slot check loop in `menu_range_8007109C` runs it for the hero's party;
  not traced end to end).

## Disc data

The programs the GameCube sends to a GBA are root `.bin` files on the disc (gba-upload.md): `colbtl.bin` for a GBA
used as a controller (**likely**); the `bg*thumbcode` files are the e-Reader's (Card e battles), not part of
trading.

## The GBA side (from the GBA games' decompilations)

Facts about the GBA games come from the pret decompilations (pokeruby, pokeemerald,
pokefirered), not from Colosseum's code.
- **The GBA games can receive a program from a GameCube while the cartridge game runs** (**verified** from the
  decompiled source: a GameCube multiboot receiver library linked into the GBA game). It accepts the image over the
  serial link, checks it, copies it into the GBA's work RAM and jumps into it. A program running there has the
  cartridge and its save at hand, so the GameCube **likely** reads and writes the GBA save through a helper program
  it sends. Which file Colosseum sends for trades is **not yet found** (gba-upload.md lists the GBA programs on the
  disc).
- **The save** (**verified** from the decompiled source): 128 KB of flash in 32 sectors of 4 KB. Two save slots of 14
  sectors are written alternately; the rest holds the Hall of Fame and a few extra records. Each sector has 3,968
  bytes of data, then a footer with the sector's id, a 16-bit checksum, a fixed signature and a save counter (the
  slot with the higher counter is the newer one). In a slot, sector 0 holds the trainer data, sectors 1-4 the world
  data (party, bag, flags) and sectors 5-13 the PC boxes.
- **A Pokémon in the save** (**likely**, from the decompiled source and long-standing community documentation): 80
  bytes in a box, 100 in the party. Its four 12-byte substructures are stored in one of 24 orders chosen by the
  personality value, XOR-encrypted with the personality value and trainer id, and covered by a 16-bit checksum.
  Colosseum's converters work on the decoded layout above (`GbaPokemon`).

## Changing it

- **Trading through a save file instead of the cable** (a PC port with no GBA): read the GBA save, decode the
  Pokémon, then run the same converters and the battle-mode check as a cable transfer, so the result matches.
  Writing back must keep the slot rotation, the sector checksums and the Pokémon checksum valid.

For a Pokémon storage tool: the two converters plus the stat reorder and the Gen 3 checksum are the core.
Mods that add species or moves must keep the GBA ids in range, or the conversion must refuse them.

### Mod surface
- **Data it reads:** the GameCube Pokémon record and the GBA (Gen 3) record formats, item exportability, the
  event-flag rule.
- **Hooks it would need:** a save-file transfer in place of the cable (read and write a GBA save), and a refusal
  (or stand-in) for species, moves and items that have no GBA id.
- **Fixed-size limits:** Gen 3's id ranges for species, moves and items.
- **Must stay vanilla in GameCube-accurate mode:** both converters, the checks and the checksums.

## Efficiency

Everything moves as small fixed records: one Gen 3 Pokémon is 0x64 bytes, a link packet 200 or 120 bytes plus the
attached Pokémon and 11 bytes of extra ribbons (**verified**, above). Each conversion runs once per transfer, so the
cost is the link's transfer time, not the processing (**likely**).

## Open questions

- What the 0x32-word and 0x1E-word payloads are (two packet kinds).
- How the GBA checksum and substructure encryption are handled (done on the GBA side?).
- Whether the trade uploads any program to the GBA at all. The GBA games have a receiver for
  one (above), so **likely** yes; which file is still open.
