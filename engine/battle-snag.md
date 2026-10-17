# Battle engine: snagging and catching (the ball throw)

Status: **partial** (2026-10-16). Companion to [battle-engine.md](battle-engine.md). *Hokaku* = capture.

## What it does

Throwing a ball (in Colosseum: a Snag Ball, or any ball with the Snag Machine, at a trainer's Shadow Pokémon) runs
the battle-script command at Gen 3's "handle ball throw" number (0xEF, `fn_80213E94`). It computes Gen 3's catch
odds, rolls up to four "shakes", plays the throw and shake animations, and jumps to the caught or broke-free script.
The formula is **Ruby / Sapphire's**, with one Colosseum change: **a Shadow Pokémon uses its own catch rate from the
Shadow Pokémon table, not its species' rate.** Everything below is **verified** from the byte-matched code unless
marked.

## Data structures

- **Ball:** the thrown item's id (field 0x1E of the item being used, the same field the Bag code reads as the item
  id). The
  balls are items 1-12, in Gen 3's item order: 1 Master, 2 Ultra, 3 Great, 4 Poké, 5 Safari, 6 Net, 7 Dive, 8 Nest,
  9 Repeat, 10 Timer, 11 Luxury, 12 Premier (**verified** that the code switches on the item id; the names are
  **likely**, from Gen 3's item list and the multipliers below).
- **Catch rate:** a normal Pokémon uses the species record's catch rate (`pokemonGetStatus(0, species, 0x12, 0)`); a
  Shadow Pokémon uses byte +0x00 of its **Shadow Pokémon record** (`fn_801EEE44(shadowId)`, the 96-entry table at
  `lbl_80478F6C`). In the US data most Shadow Pokémon have 45, and the range runs from 10 to 255.
- **Repeat Ball's "caught before":** for a Shadow Pokémon, the story flag in its Shadow record (+0x12,
  `fn_801EEAD0`: set once that Shadow Pokémon has been snagged); for others the Pokédex "caught" flag (status 0x63).

## Flow

1. **Ball multiplier** (x10, so 10 = x1.0):
   Ultra 20, Great 15, Poké / Safari 10, Net 30 against Water or Bug types (else 10), Dive 35 when the battle floor's
   terrain (floor status 0x0F) is 3, 4 or 5, the three water terrains (**verified**, battle-arenas.md "Terrain"; else 10), Nest 40 - level (at least 10),
   Repeat 30 if caught before (else 10), Timer 10 + turn count (at most 40), Luxury / Premier 10.
2. **Master Ball** (kind 1), or **battle 17** (floor status 0x0D, the battle id, = 0x11: the battle against TRUDLY,
   an ordinary map battle in room `M1_houseA_1F_bf`, a Phenac City house; **verified** from the battle, arena and room
   tables; it is the first, tutorial snag: Trudly's Makuhita, the first Shadow Pokémon, **verified** from the story
   scripts), skips the roll: always caught.
3. **Odds:** `a = rate x multiplier / 10 x (3 x maxHP - 2 x HP) / (3 x maxHP)`; x2 if asleep or frozen, x1.5 if
   poisoned, badly poisoned, paralysed or burned.
4. If `a >= 255`, caught. Otherwise `b = 0xFFFF0 / sqrt(sqrt(0xFF0000 / a))`, and up to four checks
   `random16 < b`: each passed check is one shake; four means caught.
5. The trainer's throw (`fightTrainerHokakuThrowEffect`, steps 0-6) and the ball's shakes
   (`fightOutPokemonHokakuEffect`, steps 0-4) are played in between; the shake count is stored in sequence flag byte 5
   and picks the "broke free" message from a 4-entry table (`lbl_80279E1C`).
6. **Caught:** sound 0x3F6, message 0x771F, then the caught script (`lbl_80375E51`), which goes on to Gen 3's "give
   caught Pokémon" command (0xF0, `fn_80213A78`): the Pokémon is marked seen / caught (and as a snagged Shadow
   Pokémon), removed from the battle (its battle copy set to 0 HP), its conditions cleared, Hyper Mode ended.
   **Broke free:** the script `lbl_80375E5F`.

### Who can be snagged (**verified** from `fight_menu.c`)
Throwing a ball opens a target choice among the opposing Pokémon (`..SubBallSelectTargetPokemon`). If the chosen one
is **not a Shadow Pokémon** the throw is refused before any ball is used: Rui says it has no black aura (message
0x7716) until story flag 0x99F is set, and afterwards that it is not a Shadow Pokémon (0x7702); then the menu returns
to the target choice. The thrown ball and the "threw a Snag Ball" message (0x7703) only follow for a Shadow Pokémon.

The text has a "Snag Machine is charging after use" message (0x7715), but no code loads that message id (searched in
the target assembly of `main.dol` and the RELs): a cool-down rule that is **likely unused** in the final game.

## Disc data

- Balls and multipliers: the balls are item ids 1-12; the multipliers are code above (no multiplier table).
- Shadow catch rates and "snagged" flags: the Shadow Pokémon table in `common_rel` (pokemon-data.md (not yet published), trainers.md (not yet published)).
- Species catch rates: the species table.

## Changing it

- A Shadow Pokémon's difficulty to snag is one byte in its Shadow record: a mod can change it without touching the
  species.
- A new ball needs a new case in `fn_80213E94` keyed by its item id (the multiplier is code, not data).

## Efficiency

Two square roots per throw (the classic Gen 3 integer-friendly shake formula); everything else is integer maths.

## Region notes

None.

## Open questions

- None known.
