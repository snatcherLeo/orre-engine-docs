# Battle engine: Shadow Pokémon in battle, and the heart gauge

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: XD keeps the Shadow mechanics and adds to them; not checked)

Status: **partial** (2026-10-16). A map of every Shadow-specific battle rule, with links to where each is
described, plus how the heart gauge goes down. Shadow data structures (Shadow ids, the Shadow table, the Pokémon's
gauge fields) are in pokemon-data.md (not yet published). *Dark* = Shadow, *Dp* = heart gauge, *relive* =
purification, *seikaku* = nature. All claims **verified** from the byte-matched code unless marked; nothing here has
been watched running yet.

(The whole Shadow loop, field side included: shadow-system.md.)

## What it does

Colosseum's battle engine is Ruby / Sapphire's (battle-engine.md) with a thin Shadow layer on top: one extra move
(Shadow Rush), one extra condition (Hyper Mode), one extra action (Call), one extra move check, a different EXP rule,
Shadow catch rates for snagging, and a heart gauge that the battle lowers.

## The Shadow rules, in one list

| Rule | Where | Page |
|---|---|---|
| **Shadow Rush** is move 356 (effect 48, recoil of a quarter of the damage: **likely**; uses no PP); move 357 is the locked-move placeholder, substituted by the move getter by gauge level ("Locked moves" below) | move data, `fn_802128D0` | battle-engine.md, "How a move picks its script" |
| Shadow Rush in Hyper Mode crits 90% of the time | critical check | battle-engine.md, "Critical hit" |
| Shadow Rush skips the type check: always neutral, no STAB | `fn_802274F0` | battle-engine.md, "Type effectiveness" |
| **Hyper Mode** (condition 0x3E) starts only when Shadow Rush is used: nature x gauge-level chance | `fn_8022EDEC` mode 0 | battle-turn-flow.md 3d |
| In Hyper Mode: seven misbehaviours before other moves (nature +0x0B..+0x11); 1 / 256 self-recovery; reminder each turn; refuses items | `fn_8022A6C8`, `fn_8022EDEC` | battle-turn-flow.md 3d |
| **Call** (action priority 2): wakes a sleeper; ends Hyper Mode (always, in the US data) | `fn_80212D6C` | battle-turn-flow.md 3c |
| Rui spots Shadow Pokémon at battle start | `fightCheckRuiShadowSense` | below |
| The battle kind's Hyper Mode rule (floor status 0x31) switches all Hyper Mode behaviour off | | battle-turn-flow.md 3d |
| Shadow Pokémon bank EXP (gauge level 3 or more) instead of gaining it | `fn_802232F4` | battle-rewards.md |
| Snagging uses the Shadow record's own catch rate | `fn_80213E94` | battle-snag.md |
| The heart gauge drops when the player's Shadow Pokémon enters battle, and on a successful Call | below | |

## Rui's detection (`fightCheckRuiShadowSense(pokemon, side)`, run for every Pokémon at battle start; **verified**)
- **The player's own Shadow Pokémon** (once per battle, status 0xD1): the "entered battle" heart-gauge drop (kind 0
  below) and a status-panel refresh.
- **An opponent's Shadow Pokémon:** if the battle kind's detection rule is on (status 0x2B, battle-kind flag offset
  0x0F), the player has Rui with them (player trainer field 0x4C), and this Pokémon hasn't been pointed out yet in
  this battle (status 0xD1): Rui's model (trainer model 0x32) is swapped in for the player's on the battle field, she
  says it (message 0x7717, "a Pokémon with a black aura", until story flag 0x9A0 is set; after that 0x770A, which
  names it a Shadow Pokémon), and the player's model returns.
- **One special story battle** (floor status 0x0D = 0x11) gets a longer scene instead: Rui recognises the Pokémon she
  was kidnapped for seeing and warns that it attacks people (messages 0x7729, 0x772A), and if the player has no Poké
  Balls, she reacts to that (0x772B). The same battle value makes the catch automatic (battle-snag.md), so this is a
  scripted first-snag battle (**likely**): battle 17, against TRUDLY (battle-snag.md).

## Data structures

- **Gauge value:** Pokémon status 0xC5, stored as hundredths (the code divides by 100.0 to work in floats).
- **Gauge level** (`pokemonGetDarkPokemonLevel`; **verified**): the gauge as a percentage of the
  Pokémon's starting gauge (status 0xC4), in 20% bands: 100% or more = 0, 80-99% = 1, 60-79% = 2, 40-59% = 3,
  20-39% = 4, above 0 = 5, empty = 6 (ready to purify), not a Shadow Pokémon = 7. This is the index into the nature
  tables (Hyper Mode chance, Call chance), and it gates EXP banking (level 3 or more: the gauge is below 60%). The five
  bands above "empty" match the gauge's five bars on screen (**likely**).
- **The 60% threshold (level 3)** gates three things (**verified** from code): EXP is banked
  only from level 3 (battle-rewards.md); a Shadow Pokémon's **friendship** is a separate value (status 0xC7) that
  only changes from level 3, from any source including items (`pokemonGetFriendFormPokemonFriendFilterId`,
  `itemUse2PokemonSimulation`); and its **nature** is shown only from level 3, a placeholder message (0x934) before
  (the GBA-link status screen, `fn_8009567C`; the console menus likely do the same).
- **Gauge drop table** (`pokemonDpFilterDataBiosGetPtr`, in `common_rel`): one signed amount per event kind
  ("filter id").
- **Nature purification rates:** each nature names a **rate record** per event kind
  (`pokemonSeikakuDataBiosGetRelive{Fightout,Walk,Call,Sodateya,Nadenade}`); a rate record is a multiply / divide pair
  (`pokemonSeikakuRateDataBiosGet{Kake,Waru}`, *kake* = times, *waru* = divide).

## Flow: lowering the heart gauge (`pokemonAddDpFormPokemonDpFilterId(pokemon, item, filterId)`)
1. Only for a Shadow Pokémon (status 0xC2 set).
2. `delta` = the drop table's amount for the event kind.
   - Kind 4 (**scents**): only for an item of kind 6, and `delta` is multiplied by the item's own value (the scent's
     strength).
   - Kind 5: `delta = -1 x the current gauge`, i.e. it **empties the gauge** (**likely**: the final purification).
3. `delta = delta x kake / waru` from the nature's rate record for that event kind (no change if *waru* is 0).
4. `gauge = max(0, gauge + delta)`.

| Kind | Event | Called from |
|---|---|---|
| 0 | the Shadow Pokémon is sent into battle (once per battle: status 0xD1; only the player's own) | battle start / send-out (`fightCheckRuiShadowSense`) |
| 1 | walking: every 256 steps, counted per Shadow record | `field_exact_801ED3B8.c` (the step handler) |
| 2 | a successful Call ends its Hyper Mode | `fn_80212D6C` |
| 3 | daycare (step count, like walking) | `field_exact_801ED3B8.c` |
| 4 | a scent is used on it | `gs_title.c` (menu side) |
| 5 | empties the gauge | no caller found (main.dol and the 25 room modules the decomp builds, of about 170 on the disc): **likely** unused |

The names of kinds 0-4 come from the getters the code picks for each (`Fightout`, `Walk`, `Call`, `Sodateya`
(daycare), `Nadenade` (*nadenade* = stroking; used for scents)).

## Locked moves and how they unlock (`pokemonBiosGetPokemonWazaPtr`, 0x8011F260; **verified** from the byte-matched C and the executable's data)

A Shadow Pokémon's real moves are **stored all along**: nothing in its data is locked, and nothing is written when a
move "unlocks". Instead, the move getter has a **Shadow view** (its `mode` argument = 1) that substitutes two fixed
move records from the executable:

- `lbl_80478B58` = move **356, Shadow Rush** (PP 5, PP-ups 0), and
- `lbl_80478B5C` = move **357, the "?" placeholder** (PP 5, PP-ups 0).

The substitution applies when the Pokémon has a Shadow record, its gauge is not negative, and it does **not** belong
to an enemy trainer (its trainer-Pokémon id is 0: the player's own, after a snag; **likely** cleared by the snag), and
it depends on the gauge level (`pokemonGetDarkPokemonLevel`: 0 = full, then one level per 20% down, 6 = empty):

| Slot (code index 0-3 = slots 1-4) | Shown as | Until |
|---|---|---|
| 1 | Shadow Rush, always (the real slot-1 move stays hidden) | purification |
| 2 | "?" | the gauge drops **below 80%** (level 2) |
| 3 | "?" | the gauge drops **below 40%** (level 4) |
| 4 | "?" | the gauge drops **below 20%** (level 5) |

The everyday move readers all use the Shadow view (`pokemonBiosGetPokemonWazaDataId`, `...WazaPp`, `...WazaPpCount`),
so menus and battle see Shadow Rush and "?" moves; copying and writing moves use the real view (mode 0). Two effects
of reading the fixed records: Shadow Rush's PP never goes down (the record is constant data, which is how "uses no PP"
works), and setting PP-ups on a slot that is still shown as a move with 4 PP or fewer is refused
(`pokemonBiosSetPokemonWazaPpCount`). After purification the substitution stops, so the stored slot-1 move takes
Shadow Rush's place and every move is visible.

**For the port and mods:** the unlock thresholds (80 / 40 / 20%) and the slot order are code, not data; a mod that
wants different unlock points changes `pokemonBiosGetPokemonWazaPtr`. The two fixed records are data in `main.dol`.

**The "purified" flag** (**verified**): each Shadow record (`common_rel` Shadow table, 0x38
bytes) names a story flag at **+0x14**; `fn_801EECD8(id, 1)` sets it at purification and `fn_801EEC74(id)` reads it
("is this Shadow Pokémon purified?", used for example by the walking and day-care gauge code). `fn_801EE398` checks
that all 48 Shadow Pokémon of the list `lbl_80375240` are purified: one of the conditions of the Ho-Oh gift
(menu-rooms.md, "Gift Pokémon").

## Purification (`reliveCeremonyAll`, the Relic; **verified** from code)
1. The Shadow record is marked purified (`fn_801EECD8(shadowId, 1)`), Hyper Mode is cleared if set, and the gauge is
   set to -1.0 (the "purified" value, below empty).
2. Sound 0x3F7 and the ceremony messages play.
3. **The banked EXP is announced and applied** (`_expRecover`: levels gained, with their moves), the EXP from
   battles fought while the gauge was below 60% (battle-rewards.md).
4. **Friendship becomes: current + the banked Shadow friendship (status 0xC7) + 70**, and the bank is cleared.
5. An **evolution check** runs immediately (`pokemonEvolutionCheck` / `pokemonEvolutionAll`), so a purified Pokémon
   that qualifies evolves on the spot.

Kind 5 of the gauge-drop function (empty the gauge) has no caller in `main.dol` or any of the 25 room modules the decomp
builds (of about 170 on the disc; checked in the target assembly): purification sets the gauge directly instead (**verified**); kind 5 is unused (**likely**).

## Disc data

The drop table, the nature records and the rate records are in `common_rel`; values are not listed here (disc data).

## Changing it

- Purification speed is data: the drop table (per event) and the nature rates. A port's "faster purification" option
  can scale `delta` in this one function.
- Hyper Mode can be switched off per battle kind by data (rule 0x31).

## Region notes

None found.

## Open questions

- Whether a snag clears the Pokémon's trainer-Pokémon id (the locked-move view assumes so).
