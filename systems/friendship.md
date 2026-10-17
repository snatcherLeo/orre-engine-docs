# Friendship

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). How a Pokémon's friendship (*natsuki*, status 0x99) changes. The one
function that changes it is `pokemonGetFriendFormPokemonFriendFilterId(pokemon, held item effect, event)`
(0x80122370; read from the target assembly, a near copy is in the decompiled C).
**Verified** from that code unless marked.

## What it does

Gen 3's friendship rules (pokeemerald's `AdjustFriendship`), with one Colosseum addition for Shadow Pokémon.
Every change is "event x friendship band -> amount", read from a small table.

## Data structures

**The friendship table** (`lbl_8035F988` in the executable, count `lbl_80478B70` = 9; accessors
`pokemonFriendFilterDataBiosGetPtr(event)` and `...GetValue(record, band)`): one record of three signed bytes per
event, one amount per friendship band. **All 9 records are identical to Gen 3's table** (`sFriendshipEventModifiers`,
compared byte for byte, 2026-10-16), and the event numbers are Gen 3's:

| Event | Gen 3 name | Where Colosseum uses it |
|---|---|---|
| 0 | level up | a level gained in battle (the EXP command, `fn_802232F4`) and levels gained when purification applies the stored EXP (`_expRecover`) |
| 1 | vitamin | **never used**: no caller passes it (below) |
| 2 | battle item | **never used**: no caller passes it (below) |
| 3 | league battle (Gen 3: gym leader, Elite Four, champion) | **at battle start**, for each of the player's Pokémon entering, when the battle kind's "friendship changes" (status 0x27) and "boss battle" (0x2E) switches are both on (battle-engine.md, "The full picture") and the trainer is the player (`fightActionFlowKaisiNyuujouTrainer`); also for a Pokémon snagged during such a battle (`fn_80213A78`) |
| 4 | learn TM / HM | teaching a move from the bag's TM screen (`fn_80022834`, after `pokemonWazaCreate`) |
| 5 | walking | every 256 footsteps (field-steps.md) |
| 6 | faint (small) | fainting in battle when no opponent is 30 or more levels higher (`fn_80224158`) |
| 7 | faint from field poison | field poison (field-steps.md) |
| 8 | faint (large) | fainting in battle when the strongest opponent is **30 or more levels higher** (Gen 3's rule, same number) |

Callers in C (2026-10-16): field poison (7), walking (5), the TM screen (4), purification's EXP (0), and the battle
(0, 3, 6, 8) through the wrapper `fightPokemonGetFriendFormPokemonFriendFilterId(battle Pokémon, event)`, which passes the event from
the battle code and the held item's effect (or none when a check on condition 0x3D says the item does not count; not
traced).

**Events 1 and 2 are never used** (**verified**: every call to this function and to its
battle wrapper in `main.dol` and the code modules checked passes one of 0, 3, 4, 5, 6, 7 or 8). Items change friendship
another way, as Gen 3's item effects do: the item-use code (`fn_80144574`) reads three amounts from the item's own
parameters (`itemParamGetFriend1Up` / `2Up` / `3Up`, one per band: below 100, 100-199, 200 and up) and applies the
one for the Pokémon's band (`friendXUp`). So vitamins and similar items take their friendship change from item data,
not from this table. `friendXUp` (the item-use path) applies the same rules as steps 1-5 below (**verified** from its
code): no change for a Shadow Pokémon below gauge level 3; the band check counts the Shadow pool; a positive amount is
scaled x 150 / 100 by the Soothe Bell (effect 27) and gets +1 for the Luxury Ball (item 11); the result is kept in
0-255 and goes into the Shadow pool (0xC7) for a Shadow Pokémon, into friendship otherwise.

**Items with friendship amounts** (**verified** from the US item and item-parameter tables; amounts per band, below
100 / 100-199 / 200 and up, the same as Gen 3's): the vitamins (HP Up, Protein, Iron, Carbos, Calcium, Zinc), Rare
Candy, PP Up and PP Max raise it by 5 / 3 / 2; the battle items X Attack, X Defend, X Speed, X Accuracy, X Special,
Guard Spec. and Dire Hit by 1 / 1 / 0; the bitter herbal medicines lower it: Energy Powder and Heal Powder by
5 / 5 / 10, Energy Root by 10 / 10 / 15, Revival Herb by 15 / 15 / 20. No other item carries an amount. The amounts are
three signed bytes (+5 to +7) in the item's parameter record, which the item record's byte +0x04 selects. The item
record itself also holds the same three values at +0x24 (pokemon-data.md (not yet published) "Item entry"; identical for every item
checked: **verified**), but the item-use code reads the parameter record, so a mod must change both copies together
(**verified** that the use code reads the parameter record; whether anything reads +0x24 is not traced).

**Status ids used** (per-Pokémon status, battle-turn-flow.md): 0x99 friendship (0-255), 0xC2 "is Shadow",
0xC7 the Shadow friendship pool, 0x73 the ball it was caught in, 0x6E species, 0x7B fainted, 0x70 / 0xB6 / 0xB8 validity
checks (below).

## Flow

1. **Skip** the change when the Pokémon is not valid: no species, species 412 (Gen 3's egg slot), status 0xB6 or 0xB8
   set (**likely** egg / bad-egg flags), or the check on status 0x70 (`gamedataAttestCheckValid`, **likely** the
   Pokémon's data checksum) fails. For every event except the three faint events, also skip when the Pokémon has
   fainted (status 0x7B: current HP is 0, `pokemon_get_status_exact_8012640C.c`): a fainted Pokémon gains nothing.
2. **Band:** friendship below 100 = band 0, 100-199 = band 1, 200+ = band 2. For a Shadow Pokémon the band uses
   friendship plus its Shadow pool (0x99 + 0xC7).
3. **Walking:** event 5 is skipped half the time (`GSrand & 1`), as in Gen 3.
4. **Amount** = the table's value for (event, band). If it is positive: x 150 / 100 when the held item's effect is
   27 (the Soothe Bell, Gen 3's friendship-boost effect), then +1 more if the Pokémon was caught in **item 11, the
   Luxury Ball** (item 11 and the Soothe Bell's effect 27 both checked in the US item table). (Gen 3 also adds +1 when the Pokémon is in the area where it was met; Colosseum does not.)
5. **Apply:**
   - normal Pokémon: friendship += amount, kept in 0-255;
   - **Shadow Pokémon** (0xC2 = 1): nothing changes while the heart gauge level is below 3; from level 3 (gauge under
     60%) the amount goes into the **Shadow friendship pool** (0xC7, kept in 0-255), not into friendship. On
     purification the pool is added to friendship (battle-shadow.md, "Purification").

**Which battles** (**verified** from the US disc's battle-kind table): friendship changes in
battle in every story kind (1-11, 17) and in no battle-mode kind (12-16, 18); the boss bonus (event 3) applies in
kinds 1, 5, 7 and 9 (the bosses on the map, in the story Colosseums, in Phenac's Colosseum and the 100th Mt. Battle
opponent) and in the test kind 4.

**Trades** (**verified**, `gs_npc_event.c`, the exchange's final step): both traded Pokémon's
friendship is set to **70** (`pokemonBiosSetFriend(.., 0x46)`), whatever it was before.

## What friendship is used for (partial)
- **Return and Frustration** (**verified**): power = friendship x 10 / 25 (Return, 0-102) and
  (255 - friendship) x 10 / 25 (Frustration), Gen 3's formulas, in two small helpers (`fn_80217BEC`, `fn_80217BD0`);
  the power step (`fight_trainer_ai_waza_damage.c`) writes the result as the move's power (move status 0x2F). The
  friendship read is status 0x99 of the Pokémon's party record (`fn_8023842C`), so a Shadow Pokémon's pool (0xC7) does
  **not** count until purification adds it.
- **Friendship evolutions** (evolution.md): at 220 or more, on level-up; the day / night ones never happen (no clock).

## Disc data
The friendship table is in `main.dol`. Held-item effects come from the item table (`itemDataBiosGetItemSoubiDataId`).

## Changing it
- Amounts: data only (the 9 x 3 table; item friendship: the three bytes of the item's parameter record).
- The 50% walking skip, the Soothe Bell scale, the Luxury Ball bonus and the band thresholds (100, 200) are code.
- Port idea: a "met location" bonus (Gen 3's) would need a met-location check added at step 4.

## Efficiency
One table read and a few status reads per change.

## Region notes
None found.

## Open questions
None known.
