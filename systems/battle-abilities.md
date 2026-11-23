# Battle engine: abilities (where each one is checked)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **nearly complete** (2026-10-16): every Gen 3 ability with a battle role is located in code or scripts, with the matching
behaviour. Companion to [battle-engine.md](battle-engine.md) and [battle-turn-flow.md](battle-turn-flow.md).
*Tokusei* = ability.

## What it does

An ability is just an id on the Pokémon. The ability table (`pokemonTokuseiDataBiosGetPtr`, `pokemon_tokusei_data`) holds
only a **name and a description** message per ability; **all behaviour is code** that compares the id at the place
where the ability matters (damage, accuracy, speed, switching in, end of turn, ...). There is no ability script or
hook table.

**The ids are Generation 3's** (Ruby / Sapphire, 1 = Stench ... 77 = Air Lock). Evidence: every id in the table
below is checked in the function where that Gen 3 ability acts (Battle Armor in the critical check, Sturdy in the
one-hit-KO check, Compound Eyes in the accuracy check, ...): **verified** as a pattern over 40 ids. The behaviour
itself was read line by line for: the weather abilities, Intimidate and Trace (switch-in), Rain Dish (heals 1/16 in
rain), Early Bird, Truant, Swift Swim / Chlorophyll, and the damage-formula abilities (battle-engine.md). For the other
rows the "what it does" column is Gen 3's behaviour at that check (**likely**).

## Data structures

- Pokémon field ability: `fightOutPokemonGetTokuseiDataId(p)` or `pokemonGetStatus(p, 0, 0x100, 0)`.
- Field-wide check: `fightFloorCheckFightOutPokemonPtrAryPokemonTokuseiDataId(floor, id)` (is any Pokémon on the field
  holding it) and `...TokuseiDataIdFirst` (the first holder, for redirection such as Lightning Rod).
- Once-per-entry flags on the Pokémon: status 0x116 (Intimidate done), 0x117 (Trace done).

## Flow: where each ability is checked (survey of `src/game/fight*.c`)

| Id | Ability (Gen 3) | Checked in | What the code does there |
|---|---|---|---|
| 2 | Drizzle | switch-in (`fightOutPokemonTokuseiCheckAppear` (0x8022E410)) | sets permanent rain (field condition 0x50) and its message |
| 3 | Speed Boost | end of turn (`fn_8022DF08`) | |
| 4 / 75 | Battle Armor / Shell Armor | `WS_CRITICAL_CHECK` | no critical hits |
| 5 | Sturdy | `fightCheckOneHitKO` | blocks one-hit-KO moves |
| 6 | Damp | explosion command (0x78, `wsCmd78` (0x8021D090)) | the move fails if any Pokémon on the field has Damp |
| 8 | Sand Veil | `fightCheckAccuracy`, weather damage | in sandstorm, moves against it have accuracy x0.8; no sandstorm damage |
| 12 | Oblivious | status-immunity group (`_fightTokuseiCureJoutaiSub` (0x8022D394)) | can't be infatuated |
| 13 / 77 | Cloud Nine / Air Lock | weather getter, switch-in | weather effects off while on the field (battle-turn-flow.md) |
| 14 | Compound Eyes | `fightCheckAccuracy` | accuracy up |
| 19 | Shield Dust | `wsStatBuffChange` (0x8021B910) (secondary effects) | blocks secondary effects |
| 22 | Intimidate | switch-in | once per entry (status 0x116), queues the Attack drop (condition 0x3B) |
| 23 / 71 / 42 | Shadow Tag / Arena Trap / Magnet Pull | the switch check | trap opponents: all, those touching the ground (not Flying, not Levitate), Steel types |
| 25 | Wonder Guard | `fightSeqWazaTypeCalc` (0x802274F0), `fightTrainerAiGetAisyou` (0x8023793C) (type calc, AI) | only super-effective moves hit |
| 26 | Levitate | type calc, AI, switching (`...IrekaeSelect`) | immune to Ground moves; not trapped by Arena Trap (**likely**) |
| 28 | Synchronize | `_wsCheckSynchronizeAttacker` (0x8022D20C), `fn_8022D2CC` | passes status back |
| 29 / 73 | Clear Body / White Smoke | stat-drop code (`wsStatBuffChange` (0x8021B910), `wsCmd48` (0x80220868)) | block stat drops |
| 31 | Lightning Rod | target choice (`fightSeqSelectTarget` (0x8022B2CC)) | redirects Electric moves (the first holder on the field). **Verified in play** (console recordings, double battles): the message "<Pokémon>'s LIGHTNINGROD took the attack!" comes first, then the move resolves against the holder; with a Ground-type holder (an opposing Rhydon taking a Thunderbolt) it then shows "It doesn't affect RHYDON...", so the redirect happens before the type check and a Ground-type holder cancels the move outright |
| 32 | Serene Grace | `wsCmd15` (0x80224820) (effect chance) | doubles the secondary-effect chance (**verified**) |
| 33 / 34 | Swift Swim / Chlorophyll | effective Speed | x2 in rain / sun |
| 36 | Trace | switch-in | once per entry (status 0x117), queues the copy (condition 0x3C) |
| 37 / 74 | Huge Power / Pure Power | damage (`wsCmd80` (0x8021C900)) | x2 Attack |
| 43 | Soundproof | `WS_HITCHECK`, `fightSeqCheckTokuseiBlock` | blocks sound moves (move status field 0x17) |
| 44 | Rain Dish | end of turn | heals 1/16 in rain, if not at full HP |
| 45 | Sand Stream | switch-in | permanent sandstorm (0x51) |
| 46 | Pressure | `WS_HITCHECK`, `wsCmd03` (0x8022808C), `_fightCheckYokodoriSub` (0x8022A504) | extra PP use |
| 47 | Thick Fat | damage | halves Fire and Ice |
| 48 | Early Bird | move checks (`_wsHitCheckJoutaiSub` (0x8022F2F8)) | sleep counter advances two at a time |
| 51 / 52 | Keen Eye / Hyper Cutter | stat-drop code | block accuracy / Attack drops |
| 53 | Pickup | the "pickup" script command (0xE5, `wsCmdE5` (0x80214B58)) | the script command does **nothing** (a 16-byte stub that only steps the cursor, **verified**); Pickup runs **after the battle** instead: `fightActionFlowSyuuryouPost` calls `heroCheckSetMonohiroiAllTemotiPokemon` for the party when the battle kind's Pickup rule (status 0x30) is on (**verified**). Per Pokémon with Pickup and no held item, a 1-in-10 roll (`GSrand() % 10 == 0`), then `GSrand() % 100` picks from a table of 10 {u16 item, u16 cumulative chance} pairs: the first pair whose chance is above the roll (**verified** from code) |
| 54 | Truant | move checks, end of turn | acts every other turn |
| 55 | Hustle | damage, accuracy | x1.5 Attack, lower accuracy |
| 57 / 58 | Plus / Minus | damage | x1.5 Sp. Atk with the partner ability |
| 59 | Forecast | switch-in | changes Castform's form by weather (`fightGetForecastForm` (0x8022EC40)) |
| 21 | Suction Cups | move-effect scripts (`jumpifability`) | can't be forced to switch |
| 30 | Natural Cure | switch-out command (0xE2, `wsCmdE2` (0x80214C04)) | status cured on switching out |
| 39 | Inner Focus | secondary-effect code (`fightSeqSetTsuikaKouka` (0x802249B8)) | no flinching |
| 60 | Sticky Hold | `wsCmdD2` (0x80215CF0) (item theft) | item can't be taken |
| 61 | Shed Skin | end of turn | may cure status |
| 62 / 63 | Guts / Marvel Scale | damage | x1.5 Attack / Defense when statused |
| 64 | Liquid Ooze | move-effect scripts | draining moves hurt the drainer |
| 65-68 | Overgrow / Blaze / Torrent / Swarm | damage | x1.5 for their type at a third of HP or less |
| 69 | Rock Head | secondary-effect scripts | no recoil |
| 70 | Drought | switch-in | permanent sun (0x4F) |

The ability groups, each one `switch` statement whose case list is exactly one of Gen 3's ability-effect groups
(**verified** from the case lists):

| Group (Gen 3 name) | Function | Abilities |
|---|---|---|
| on switch-in | `fightOutPokemonTokuseiCheckAppear` (0x8022E410) | Drizzle 2, Cloud Nine 13, Intimidate 22, Trace 36, Sand Stream 45, Forecast 59, Drought 70, Air Lock 77 |
| end of turn | `fn_8022DF08` | Speed Boost 3, Rain Dish 44, Truant 54, Shed Skin 61 |
| contact (after a contact move hits) | `_wsTokuseiAfterDamage` (0x8022D6BC) | Static 9, Color Change 16, Rough Skin 24, Effect Spore 27, Poison Point 38, Flame Body 49, Cute Charm 56 |
| status immunity (cure a status the ability forbids) | `_fightTokuseiCureJoutaiSub` (0x8022D394) | Limber 7, Oblivious 12, Insomnia 15, Immunity 17, Own Tempo 20, Magma Armor 40, Water Veil 41, Vital Spirit 72 |
| absorbing (move of the absorbed type) | `fightCheckTokuseiWazaKyuushuu` (0x8022DCB8) | Volt Absorb 10, Water Absorb 11, Flash Fire 18 |

**Inside the contact group** (**verified** from code): Static (9) triggers on a contact hit
while the attacker is still on the field, with a **1-in-3 roll** (`GSrand() % 3`, one draw from the shared
generator); it then sets the status to inflict (0x45) and runs the shared "ability inflicts status" battle script
(`lbl_8037992F`, **likely** used by all of them). Flame Body (49 = 0x31) is the same with 0x43 (burn). A byte list
{1, 2, 5} filled next to this code is **likely** Effect Spore's pick of status.

**Checks inside battle scripts** (**verified** from the executable's scripts): some abilities are
tested by the scripts themselves, through Gen 3's "jump if ability" commands (0x1E `jumpifability`, 0x43
`jumpifabilitypresent`), so they don't appear in the C. Following the main jump and call commands from the 214 move-effect scripts
(2,272 commands reached) finds these ability ids in such commands: **Suction Cups** (21, against forced switching),
**Liquid Ooze** (64, draining moves hurt the drainer), Soundproof (43), and the status-immunity group again (Limber 7,
Insomnia 15, Immunity 17 (twice), Own Tempo 20 (four times), Water Veil 41, Vital Spirit 72), which the scripts check
before trying to inflict a status. The **secondary-effect scripts** (a second table, `lbl_80399F58`, 39 entries, run by the
secondary-effect code) add **Rock Head** (69: no recoil).

A wider scan, from every script address the battle code references (178 roots, 3,541 commands
reached), also finds Clear Body (29), Hyper Cutter (52) and White Smoke (73) checked in the stat-drop scripts, as
well as in code.

Found through address aliases of the ability getters (`fn_80207BF4`,
`fn_801F349C`) that the first text search missed: **Damp** (6) in the explosion command (0x78, `wsCmd78` (0x8021D090): the
move fails if any Pokémon on the field has Damp), **Inner Focus** (39) in the secondary-effect code (`fightSeqSetTsuikaKouka` (0x802249B8): no
flinching), **Natural Cure** (30) in the switch-out command (0xE2, `wsCmdE2` (0x80214C04): the status is cured on switching
out). With these, every Gen 3 ability that has a battle role is located (**verified**). **No battle role expected:** Stench and Illuminate (wild-encounter rates in
Gen 3), Run Away (escaping wild battles; Colosseum has no wild battles and no escape roll), Cacophony (unused even in
Gen 3) (**likely**).

## Disc data

The ability table (name and description message per ability) is in the executable (`pokemon_tokusei_data`, 0x3A8 bytes); the
behaviour is all in code. Which species has which ability is in the species table (pokemon-data.md).

## Changing it

- A new ability needs code at every place it matters; there is no data hook. For a port, an "ability hook" layer
  (one callback table per event: switch-in, damage, accuracy, end of turn) would make abilities moddable; the
  switch statements above are the natural places to dispatch from.
- Renumbering is unsafe: the ids are compared directly in dozens of places.

### Mod surface
- **Data it reads:** each species' two ability slots and the ability ids the battle code compares against.
- **Hooks it would need:** one callback table per ability event (switch-in, damage, accuracy, end of turn), dispatched
  from the existing switch statements, so new abilities register behaviour instead of editing code.
- **Fixed-size limits:** ability ids are 8-bit; the record stores only which of the two slots is used.
- **Must stay vanilla in GameCube-accurate mode:** every ability's behaviour and the order its checks run in.

## Efficiency

Direct id comparisons at the point of use: no lookups, no allocation. Field-wide checks loop over the 8 field slots.

## Region notes

None.

## Open questions

- Whether any ability behaves differently from Gen 3 (none seen so far).
