# GBA link and Pokémon conversion

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-14). Code: `src/game/gba/` (`gba_misc.c`, `gba_conv.c`, `GBA*.c`),
`src/game/pokeconv.c`, `gbaCommunication*.c`.

## What it does

Colosseum trades Pokémon with the GBA games (Ruby/Sapphire/FireRed/LeafGreen/Emerald) over the
GameCube-GBA link cable. Pokémon are converted between the GameCube record (`PokemonBios`,
`pokemon-data.md` (not yet published)) and the GBA's 100-byte party format, and sent in byte-swapped packets.

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

- GameCube -> GBA: `fn_8008AE18(gcPokemon, gbaPokemon)` (0xDC4 bytes), then the packet builder.
- GBA -> GameCube: `fn_8008BBDC(gc, gbaPokemon)` (0x9F8 bytes) and `gbaPokemonConditonFromGC`.
- The attestation stamp on each Pokémon (`save-data.md`) records the origin game and language.

## Stats of imported Pokémon (researched 2026-10-15)

- **The importer keeps the GBA's stats (verified, `fn_8008BBDC` byte-matched):** after the boxed data it copies the
  party tail as stored on the GBA: level, mail id, max HP, HP, Attack, Defense, Speed, Sp. Atk, Sp. Def. Nothing is
  recalculated on import. `pokeconv` (`fn_80089048` area) uses it to build a whole trainer (name, gender, trainer id,
  six Pokémon, the 11 GameCube-only ribbons) from a GBA block.
- **Colosseum recalculates later (verified from code):** `pokemonResetBasisStatus` recomputes all six stats from the
  species table at the current level. It runs on every EXP gain (`pokemonGrowBasisStatus`, after battles), Rare
  Candy / vitamins (`item_range_80144574`), evolution, revival of Shadow Pokémon's EXP (`_expRecover`), gift Pokémon
  and new Pokémon (`pokemonCreate`).
- **Battle-mode legality check (verified, `fn_80076398` / `menuCBRule_CheckPokemonErrorAll`, byte-matched):** for
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

## Changing it

For a Pokébank: the two converters plus the stat reorder and the Gen 3 checksum are the core.
Mods that add species or moves must keep the GBA ids in range, or the conversion must refuse them.

## Open questions

- What the 0x32-word and 0x1E-word payloads are (two packet kinds).
- How the GBA checksum and substructure encryption are handled (done on the GBA side?).
- Whether the trade uploads any program to the GBA at all.
