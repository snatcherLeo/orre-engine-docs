# Evolution: the rules

Status: **partial** (2026-10-16). When a Pokémon evolves. Code: `pokemonEvolutionCheck` and its three
helpers in `src/game/pokemon_range_80128524.c` (byte-matched; its jump table checked against the original executable),
so the rules are **verified** unless marked. The evolution scene itself (animation, cancelling, learning moves) is not
covered here. *Sinka* = evolution.

## What it does

Each species record has **5 evolution slots** (species record +0x9C, 6 bytes each: kind, parameter, target species;
pokemon-data.md (not yet published)). The kinds are **Gen 3's evolution ids, unchanged**, and the checks are Gen 3's with one difference:
Colosseum has no clock, so the day / night friendship evolutions never happen.

## Data structures

| Kind | Gen 3 name | Check in Colosseum | Trigger |
|---|---|---|---|
| 0 | (empty slot) | never | |
| 1 | friendship | friendship **220 or more** (friendship.md) | level-up |
| 2 / 3 | friendship by day / by night | **never**: the code reads friendship and does nothing (no clock) | level-up |
| 4 | level | level >= parameter | level-up |
| 5 | trade | always | trade |
| 6 | trade holding an item | the held item = parameter | trade |
| 7 | item used | the item used = parameter | using an item |
| 8 / 9 / 10 | level, Attack > / = / < Defense (Tyrogue) | level >= parameter and the comparison | level-up |
| 11 / 12 | level, personality value split (Wurmple: Silcoon / Cascoon) | level >= parameter and (personality high half) mod 10 below 5 / 5 or more | level-up |
| 13 | level, Ninjask (Gen 3) | **never** here (not handled; Colosseum's Nincada uses kind 4 for Ninjask and kind 14 for Shedinja: verified on the US disc) | |
| 14 | level, Shedinja | level >= parameter **and a free party slot**: returned as an *extra* Pokémon next to the main evolution | level-up |
| 15 | beauty (Feebas) | the Beauty condition >= parameter | level-up |

**Everstone:** if the held item's effect is 38 (the **Everstone**'s held effect on the US disc, Gen 3's "prevents
evolution" effect), no trigger evolves the Pokémon.

## Flow
`pokemonEvolutionCheck(pokemon, mode, item, &extra, &evolution)`: Everstone check, then by mode 0 level-up
(`getEvoPokemonLevelUp`), 1 item (`getEvoPokemonItem`, kind 7 only), 2 trade (`getEvoPokemonTrade`, kinds 5 and 6).
Every slot is checked in order and the **last** matching slot wins (Eevee's item slots, for example, each match only
their own stone). Returns the target species (0 = none, 0xFFFF = no data) and fills the matched kind and parameter.

**After a battle** (**verified**, `fight_action.c`): for each of the player's
Pokémon that was out in the battle and has per-Pokémon status 0xD0 set (the **level-up flag**, `fightPokemonBiosGetLevelUpFlag`,
set by the EXP command when it gains a level: **verified**; as in Gen 3, only those evolve), the level-up check runs (mode 0) and, on a match, `pokemonEvolutionAll` plays the
evolution; then Pickup runs (battle-rewards.md).

**After a trade** (**verified**, `menuPokemonChange.c`): the trade's evolution step runs the
trade check (mode 2) on **both** traded Pokémon, the one received and the one sent (trading.md).

On the US disc (counts from the species table, 2026-10-16): 135 level evolutions, 21 item, 7 friendship, 4 trade,
8 trade-with-item, and one each of the rest. The single day and night entries are Eevee's (Espeon and Umbreon), so
in Colosseum Eevee cannot become Espeon or Umbreon by friendship.

## Disc data
The evolution slots are in the species table (`common_rel`, pokemon-data.md (not yet published)).

## Changing it
- New evolutions with the existing kinds: data only (slot kind, parameter, target).
- Making kinds 2 / 3 work (a port option with a real clock), or adding kinds: code (`getEvoPokemonLevelUp`).

## Region notes
None found (the same code for all versions, **likely**).

## Open questions
- What the extra Pokémon (kind 14) does in the evolution scene (Shedinja appears only with a free slot; whether a
  Poké Ball is used up as in Gen 3).
