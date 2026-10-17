# The P★DA

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's P★DA is larger)

Status: **partial** (2026-10-16). The player's handheld menu: the Strategy Memo, mail, and the other
screens. This page collects what is known so far; the P★DA's screen-by-screen structure is not yet traced.
**Verified** from the decomp's C and the disc's tables unless marked.

## What it does

The P★DA is a **menu room** of its own (room 0x392, `pda_menu`), pushed on the room stack when the player opens it
from the pause menu (`menuPdaOpen`; menu-rooms.md, pause-menu.md). Its room script is a stub: everything is C
(`menu/pda*.c`, `menu/menu_pda_mail*.c`). The code is a reduced version of the sequel's P★DA (**likely**: XD's code
clusters are found here with some cut out).

## Data structures
- **Scene work** `gPdaSceneWork` (0x803A6818, `PdaSceneWork`, 0x248 bytes): the P★DA's menu models and camera.
- **Model windows** `gPdaModelWindow` (`PdaModelWindow`, 0xA0): a window with its own 3D model, alpha and fade.
- **Strategy Memo list** `gPdaMemoSpecies` (a u16 per memo entry, filled from the memo's save data, section 12, by
  `memoDataGetPokemonID`; bit 0x8000 marks a **new** entry, the species is the low 14 bits) and the memo's
  **temporary Pokémon** `gPdaMemoPokemon`: for the selected entry the P★DA builds a level-10 Pokémon of that species
  with the **personality value and trainer id recorded in the memo** (`memoDataGetPokemonRndFromID`,
  `...TrainerRndFromID`), so the shown Pokémon looks like the one recorded (gender, shininess and other
  personality-based traits, **likely**).
- **A Shadow Pokémon list** `gPdaBattleList`. The entry's id
  is a **Shadow id**: `pdaBattleCaption` asks the Shadow definitions table which trainer holds that Shadow Pokémon
  (`fn_801EEFAC`, record +0x04) and captions the entry with that trainer's nickname for it, or with the species name
  when the trainer field is 9, which is the **Card e trainer placeholder** (trainer 9 in the trainer table: a
  Japanese "Card e trainer" name and no party; **verified** from the table), so Card e Shadow Pokémon have no
  fixed nickname to show (**likely**; the helper read is a second, unused copy in the decompiled code; pokemon-data.md (not yet published)). So this is **likely** the P★DA's Shadow Pokémon list.
- **Number input** `gPdaNumberValue`: clamped to a range, built digit by digit.

**The memo's save data** (save section 12; **verified**, `memo.c`): a u16 count, then 12-byte
entries from +4: the species (u16, bit 0x8000 = new), the personality value and the original trainer's id.
`memoDataSet(pokemon)` adds an entry **only if that species is not in the memo yet**, so the memo keeps the **first
individual of each species** the game records. Its callers (**verified**, every `memoDataSet` call in the C) are the **new game** (both starters, `hero.c`), the four C-side **gifts** (Celebi, Pikachu,
Ho-Oh, Plusle in `hero_pokemon.c`), **evolution** (the evolved form, and the extra Pokémon an evolution can add,
**likely** Shedinja), and the end of a **trade**, plus
`pokemonSetOnZukanFlag(pokemon, caught)` (0x80123368; **verified** from the target assembly), which adds the memo
entry, sets the Pokédex "seen" status (0x62) and, when `caught` is set, "caught" (0x63). That function runs in two
ways: when a Pokémon that is not the player's own enters battle (`fightOutPokemonSetOnZukanFlag(pokemon, 0)`, from
the battle-start entry `fightActionFlowKaisiNyuujouPokemon` and a later send-out, `fn_8021F458`), but only when the
battle kind's "record as seen" rule is on (floor status 0x1E, the `DoZukanMita` flag); and from the snag command
(`fn_80213A78`) with `caught` = 1. So the memo records the first individual of each species the player **sees** in
battle (opponents' Pokémon, Shadow ones included), plus starters, gifts, evolutions and trades; snagging needs no
separate path.
`memoDataSetMemoFlag` clears every "new" bit and returns whether any entry was new (script command
`scriptSetMemoFlag` and the party code call it, **likely** to tell the player that the memo was updated).

## Flow and features
- **Opening:** `menuPdaOpen` pushes room 0x392 and the caller's thread waits until the P★DA is closed.
- **Mail** (field-steps.md for when mail arrives and the mail record): the mailbox list (`pdaMailGetMailID(index)`),
  sorted by subject with the standard `qsort` and a comparison on the subjects' text (`fn_8004BE90`); the first time a
  mail is viewed its read effects run (story flag, mail stage, Shadow records, script, hero event).
- **Strategy Memo:** the list above, with the 3D view of the recorded Pokémon and its cry (`pdaPushCry`) and type
  icons (`pdaGetTypeIcon`).
- **Text speed:** P★DA animations scale with the text-speed setting (`gPdaTextSpeedScale` =
  characters per frame / frame rate).

## Disc data
The P★DA's models, textures and menu layouts are in its room archive (`pda_menu.fsys`) and the menu tables; mail
texts are in the game text.

## Changing it
- The memo shows recorded individuals, not species defaults: a mod that changes how Pokémon look by personality
  (gender, shininess) changes the memo too.
- New P★DA screens: code (the P★DA's screens are C, not script).

## Region notes
- The Shadow list's caption has a branch for Shadow Pokémon held by the Card e placeholder trainer (9). Card e is
  the Japanese e-Reader content (regions.md); in the US build the branch is **likely** unreachable or only reached
  through Card e data, which US play may not load (battle-engine.md, Card e notes).

## Open questions
- The full list of P★DA screens and their menu ids.
- The Shadow list screen's menu id; whether US play ever lists a Card e Shadow Pokémon; what the "People" 3D screen shows.
