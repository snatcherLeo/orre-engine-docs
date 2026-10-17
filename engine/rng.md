# RNG (random number generator)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-17). The game's one shared random number generator: its algorithm, its seeding and its
main consumers. A faithful port must reproduce the exact sequence and timing of RNG calls, because
RNG-manipulation tools (PokeFinder) depend on it.

## Algorithm (verified, code)

- **One shared 32-bit LCG**: `state = state * 0x343FD + 0x269EC3`; the output is the **high 16 bits**
  (`fn_801ADCD8`, 0x801ADCD8, in `src/hsd/hsd_pobj_range_801AA608.c`; it is HSD's `HSD_Rand`, the same code as Melee's
  `random.c`, so the generator belongs to the HSD library, not to the Orre Engine). The state lives behind the
  pointer `lbl_80478C94` (Melee: `HSD_RandSeedPtr`, which `_HSD_RandForgetMemory` resets to a static seed).
- Front ends, all advancing the same state once per call: `GSrand` (0x800E0C54, u16),
  `HSD_Randf` (0x801ADC7C, float = (state >> 16) / constant), `fn_801ADC3C(max)` (0x801ADC3C,
  `max * (state >> 16) / 0x10000`), `fn_800E0BA0` (sum of two `HSD_Randf` minus a constant = two
  advances) and `fn_800E0BE4` (one `HSD_Randf`).
- **Seeding:** `fn_800E0C78` sets the state from `OSGetTime` (the low 32 bits of the time base). **Verified in Dolphin
  2026-10-15** (with a scripting build of Dolphin): it runs about 1.0 s after boot (frame 119), and the seed
  follows the console clock: seed = low 32 bits of (seconds since 2000-01-01 x 40,500,000, the time-base rate, + about 40,381,700); with the
  clock at the GameCube epoch the seed is 0x02682D0F. Nothing draws from the RNG before the title / main menu. *For the
  port:* vanilla replays inject the captured seed; normal play seeds from the real clock.
  Which function calls it is open; it runs about 1 s after boot.

## Who calls it

| Caller | Evidence | Status |
|---|---|---|
| Battle code (damage rolls, accuracy, AI, ...): ~120 `GSrand` call sites in `fight_*` files | call sites in the decompiled code | verified (call sites), per-use meaning open |
| Pokémon creation (personality, IVs): `pokemon_range_8011F5FC.c` (8 sites) | call sites in the decompiled code | verified (call sites) |
| Save data (`hero_savedata_exact_80129280.c`: the save's random id) | call sites in the decompiled code | verified |
| Particle system (`ps_range_80168C64.c`: emission angles, spreads, lifetimes) via `HSD_Randf` | call sites in the decompiled code | verified |
| HSD bytecode interpreter (`hsd_range_801920E4.c`: random ranges) via `fn_801ADC3C` | call sites in the decompiled code | verified |
| Menus, `kaisuu.c`, battle camera (`battleCameraStartRandom`) | call sites and names in the decompiled code | verified (sites) |

## Per-frame advances ("noisy" vs "quiet" areas)

Because particles draw from the **same** generator, any room with active particle effects advances
the RNG every frame, by an amount that depends on how many particles spawn: such areas are
"noisy", rooms without per-frame random users are "quiet" (guess, from the particle code calling
`HSD_Randf`; to measure per room). NPC wandering and animated models may also call it (open).

## Shiny checks and the Shadow Pokémon quirk (2026-10-14)

**Background for a shiny-Shadow-Pokémon fix.** A Pokémon is
shiny when `(PID >> 16) ^ (PID & 0xFFFF) ^ (OT >> 16) ^ (OT & 0xFFFF) < 8`, where PID is the
personality value and OT the original trainer's 32-bit ID (secret ID in the high half). It is the
Gen 3 rule.

**Fields** (verified, `pokemonSetStatus` / `pokemonGetStatus` switches in `pokemon_range_8011F5FC.c`):
status `0x6F` = personality value (`pokemonBiosSetRnd` / `GetRnd`), status `0x75` = original
trainer's ID (`pokemonBiosSetCatchTrainerRnd` / `GetCatchTrainerRnd`). The trainer's own ID is
`heroGetStatus(hero, 2, 0)` (`heroBiosGetRnd`).

**Where Colosseum decides and checks shininess** (verified from the C unless marked):

| Step | Where | What it does |
|---|---|---|
| The shiny test | `pokemonCheckRare` (0x80125390, `pokemon_range_exact_801248C4.c`) | the rule above, on the Pokémon's own 0x6F and 0x75. Every display path calls it: portraits and menus (`gs_range_80109C88.c`), the battle model choice. |
| Creating a PID with constraints | `pokemonCreateRndFit(pkm, gender, nature, shinyMode, otId)` (0x80124410, `pokemon_range_8011F5FC.c`) | loop: draw a PID, reject it if gender or nature (`PID % 25`) don't fit, then the shiny test `pokemonCreateRndFitSeedHit` against `otId`: `shinyMode` 1 = must be shiny, **0 = must not be shiny**, -1 = don't care. |
| **Opponents' Pokémon, incl. Shadow Pokémon** | `fightTrainerCreateFightTrainerPokemonDataIdToPokemon` (`fight_trainer_range_801F7F80.c`), called from the trainer setup above it with `heroGetStatus(enemyTrainer, 2, 0)` | calls `pokemonCreateRndFit(..., shinyMode 0, enemy trainer's ID)`: the PID is **rerolled until it is not shiny for the enemy's ID**. Then the Pokémon is added to the enemy's party with `heroCatchPokemon`, which writes the enemy's ID as its OT. |
| Shadow Pokémon met again | same function, Shadow branch | if the Shadow entry's story flag is set (`fn_801EE8F4` -> `std081_flagCheckTrue`), the PID saved at the first meeting is reused (`fn_801EE750`, Shadow runtime slot +8); otherwise a new PID is made as above and saved there (`fn_801EE7BC`). So the PID, and so the shiny outcome for the player, is fixed at the first encounter. |
| **Shininess shown in battle** | battle Pokémon setup (`fight_out_pokemon_suffix_80207C6C.c`, near the `fightOutPokemonGetRndStatus` call) | copies the battle Pokémon's PID and OT (the enemy's ID) into a temporary, calls `pokemonCheckRare` and passes the result to `fn_801DE190` (model choice, shiny recolour). Because of the reroll above this is always "not shiny" for an opponent's Pokémon. |
| **Re-check after the snag** | `heroCatchPokemon` (`hero_savedata_exact_80129280.c`) | copies the Pokémon, then `pokemonSetCatchStatus(..., heroGetStatus(player, 2, 0), playerName)` writes the **player's** ID as the OT; the PID is kept. From then on `pokemonCheckRare` uses the player's ID, so the Pokémon is shiny with the usual 1-in-8192 chance (independent of what battle showed). |

So Colosseum doesn't recalculate the PID after the snag. It only changes the OT, and the same
shiny test gives a new answer. The quirk: battle always shows "not shiny" (the reroll against the
enemy's ID), while the snagged Pokémon can be shiny.

**RNG cost (verified, code).** Each PID candidate is two `GSrand` calls
(`pokemonCreateRndFitRand32`: high half, then low half). Every rejected candidate (wrong gender,
wrong nature, or shiny for the enemy's ID) costs two more calls. The anti-shiny reroll rejects a
candidate about 1 time in 8192, so it rarely adds calls, but vanilla mode must keep it to keep the
exact sequence. Rematches with a Shadow Pokémon whose PID was saved draw no PID at all.

**The mod (Colosseum), sketch:** create Shadow Pokémon with `shinyMode` -1 (or keep 0 but pass the
player's ID), and make the battle-display check use the player's ID for Shadow Pokémon, so battle
and party agree. Either change alters the RNG calls (the rejection pattern differs), so mod only.

**XD (**guess**: from RNG-community research on XD, not from code; assumed to work like Colosseum):** XD makes
Shadow Pokémon PIDs against the **player's** ID and **rerolls any that would be shiny** (the "shiny
lock"). Each reroll is one more PID candidate = **2 RNG calls** (high and low 16 bits), as in
Colosseum's loop. To find in XD: the counterpart of `pokemonCreateRndFit` and its Shadow caller, and
whether the reroll is the same `shinyMode 0` path with the player's ID passed in. Recorded in
`xd-insights.md` (not yet published).

## Port notes

- Implement the exact LCG and call it from exactly the same places, in the same order, at the
  original 60 Hz logic rate; interpolated rendering must not run particle or game logic extra times.
- Keep the shared-generator behaviour (particles and battles share one state) in vanilla mode.

## Open questions

- Who calls the seeding function and when; whether anything reseeds mid-game.
- The per-frame RNG consumers in the overworld (particles per room, NPCs, cameras).
- From the shiny-Shadow-Pokémon section: which story flag each Shadow entry uses for "PID already made"
  (Shadow table field `flag16`), and XD's counterpart.
