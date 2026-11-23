# Evolution: the rules

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). When a Pokémon evolves. Code: `pokemonEvolutionCheck` and its three
helpers in `src/game/pokemon_range_80128524.c` (byte-matched; its jump table checked against the original executable),
so the rules are **verified** unless marked. The evolution scene itself (animation, cancelling, learning moves) is not
covered here. *Sinka* = evolution.

## What it does

Each species record has **5 evolution slots** (species record +0x9C, 6 bytes each: kind, parameter, target species;
pokemon-data.md). The kinds are **Gen 3's evolution ids, unchanged**, and the checks are Gen 3's with one difference:
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

**Beauty (kind 15)** (**verified** from the getter): the check reads the Pokémon record's Beauty
contest stat (+0xB3). Beauty comes from Pokéblocks in the GBA games; Colosseum has no Pokéblocks (**likely**), so a
Feebas can only evolve here if it arrives by trade with its Beauty already high. **Tyrogue (kinds 8-10)** compares the
Attack and Defense stats themselves (not the IVs), so the nature alone can decide it.

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

**Names on evolution** (**verified**, `pokemonEvolution`): a Pokémon's name is stored on the
Pokémon itself. When a Pokémon is created, the species' name is copied from the text table into its nickname field
(`pokemonCreate`). On evolution, the game **compares** the stored name with the old species' name (`GScharCmp`): if
they are equal, the Pokémon "has no nickname" and gets the new species' name; otherwise its nickname is kept. There is
no separate "nicknamed" flag. Shedinja-style extra Pokémon get the new species' name the same way. So anything that
changes the species name table (a translation, a capitalization change) also changes this check: a Pokémon created
under the old names would then count as nicknamed and keep its old species' name after evolving.
(**verified** from the code): Every copy and comparison of a species name here goes through one
function, `GSmsgGetGSchar` (a message taken as a plain string). So a port that shows the game in more than one
language can keep the stored names in the save's own language by answering that function with the original text,
and translate the names only where they are shown.

**The evolution scene** (**verified** from the code: `evolutionOpen`, `evolution`,
`evolutionStart`, `evolutionDemo`):
- **Its own room.** `evolutionOpen` stores its arguments (the Pokémon before and after, whether B may cancel, the moves
  to learn and where to write their slots) in a small global block, warps to menu room 0x386 and waits. That room's
  script calls `evolution`, which runs the scene and writes the result back into the block: 0 evolved, 1 cancelled,
  2 a model could not be made. So the scene always costs a room change and a load. *Likely:* the battle's reward
  step and the Relic Stone's purification (a purified Pokémon that is due to evolve) both reach it this way; a
  recorded purification goes straight from the ceremony into room 0x386.
- **Setup.** Both models (before and after) are built first. The field music and any looping field sound are faded
  out, and their ids and volumes are saved so they can be restored when the scene ends, cancelled or not.
- **The main part** is one loop driven by a move-style effect sequence played on the old model. It runs until 570
  frame steps have passed or the sequence ends. Message 0x4401 opens first (likely "What? ‹Pokémon› is evolving!")
  and the old Pokémon's cry plays. When the cry has finished, the evolution jingle (sound 0x3D3) starts; 30 frame
  steps after that has finished, the second theme (0x3D4) starts.
- **Cancelling.** Only when cancelling is allowed, after 120 frame steps and once the second theme is playing, a
  newly pressed B stops the scene. A short table picks one of three "stopped" sequences by how far the scene had got.
  The old cry plays again, then message 0x43FF (likely "‹Pokémon› stopped evolving") opens.
- **The change.** The screen fades. The old model is hidden, the new one shown, and the "after" sequence plays to its
  end. The theme stops and the new species' cry plays. **The scene waits for that cry to finish**, then opens message
  0x4400 (likely "‹Pokémon› evolved into ‹species›!"). So the time from the start to that message depends on the
  cry's length, not only on fixed timers.
- **Moves.** Each move the new species learns at that level then goes through the usual "learn a new move?" dialogue
  (messages 0x4243 / 0x4242 / 0x4241 / 0x4248 / 0x423D, with the forget-a-move menu), on a copy of the Pokémon that
  is written back. The music and sound saved at the start are restored, and the scene fades out.

On the US disc (counts from the species table, 2026-10-16): 135 level evolutions, 21 item, 7 friendship, 4 trade,
8 trade-with-item, and one each of the rest. The single day and night entries are Eevee's (Espeon and Umbreon), so
in Colosseum Eevee cannot become Espeon or Umbreon by friendship.

## Disc data
The evolution slots are in the species table (`common_rel`, pokemon-data.md).

**What the US disc uses** (**verified**, a count of every filled evolution slot): **184 slots**.
- **Level** (kind 4): 135.
- **Friendship**: 9, including Espeon's kind 2 and Umbreon's kind 3, which never fire here (no clock).
- **Plain trade** (kind 5): 4 (Kadabra, Machoke, Graveler, Haunter).
- **Trade holding an item** (kind 6): 8 (King's Rock twice, Metal Coat twice, Dragon Scale, Up-Grade, DeepSeaTooth,
  DeepSeaScale).
- **Using an item** (kind 7): 21, **all of them evolution stones** (Water Stone 5, Moon Stone 5, Leaf Stone 4, Fire
  Stone 3, Thunderstone 2, Sun Stone 2). So on this disc "stone evolution" and kind 7 are the same thing.
- **Tyrogue's stat comparisons** (kinds 8 to 10): 3.
- **Wurmple's personality split** (kinds 11 and 12): 2.
- **Shedinja** (kind 14): 1.
- **Feebas's Beauty** (kind 15, Beauty 170): 1.

Kinds 0 (empty) and 13 aren't used. An editor can group these by what triggers them (level-up, trade, using an item)
for display; the kind number stays what the game reads.

## Changing it
- New evolutions with the existing kinds: data only (slot kind, parameter, target).
- Making kinds 2 / 3 work (a port option with a real clock), or adding kinds: code (`getEvoPokemonLevelUp`).
- The scene's length : the 570-step loop and the 120-step "no cancel" window are fixed numbers
  in the code; the wait after the change follows the new species' cry, so a modded species with a long cry makes the
  scene longer, and a cry that never reports "finished" would hang it.

**Mod surface** :
- **Data read:** the species table's evolution slots and level-up moves; each species' model and cry; the two
  evolution themes (sounds 0x3D3, 0x3D4); messages 0x4400, 0x4401, 0x43FF and the move-learning texts.
- **Hooks it would need:** "may this Pokémon evolve now?" (before `evolutionOpen`), the cancel rule (B after 120
  steps), and the scene itself (skip or shorten, as a speed-up option).
- **Fixed limits:** two models at a time (before and after); one evolution per scene; the moves-to-learn list given
  by the caller.
- **GameCube-accurate mode keeps:** the timers, the cry waits and the room change, since they decide how many frames
  the scene takes and so everything timed after it.

## Region notes
None found (the same code for all versions, **likely**).

## Open questions
- What the extra Pokémon (kind 14) does in the evolution scene (Shedinja appears only with a free slot; whether a
  Poké Ball is used up as in Gen 3).
