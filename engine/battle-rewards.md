# Battle engine: EXP and prize money

Status: **partial** (2026-10-16). Companion to [battle-engine.md](battle-engine.md). Both rewards are
battle-script commands at Gen 3's numbers ("get exp" 0x23 = `fn_802232F4`, "get money reward" 0x5D =
`fn_8021E754`). Everything is **verified** from the byte-matched code unless marked; ids are Gen 3's held-item
effects.

## What it does

When an opponent faints, its EXP is shared among the Pokémon that faced it (and Exp Share holders) and shown with the
EXP bar animation and level-ups; Shadow Pokémon bank their EXP instead. When a trainer is beaten, prize money is
computed from the trainer's class and the strongest Pokémon's level, or, in some battle kinds, paid as a pooled
reward instead of money.

## Data structures

Battle-kind rules read here (`fightFloorGetStatus` ids, battle-engine.md "Battle kind"): 0x24 EXP is given, 0x2D the
EXP-boost rule, 0x27 friendship changes, 0x25 prize money is given, 0x26 money is pooled.

## Flow

### EXP (`fn_802232F4`)
1. No EXP if the fainted Pokémon is the player's own, or the battle kind gives none (rule 0x24).
2. **Base:** `species base EXP x the fainted Pokémon's level / 7` (species field 0x10).
3. **Split** among the winning side's Pokémon that are still able to battle and below level 100: those that **faced**
   it (`...CheckMeetEnemyFightPokemon`) share the base; if anyone holds an **Exp Share** (held-item effect 25), the
   base is halved first and the other half is split among the holders (a holder that also fought gets both parts).
   Each share is at least 1.
4. **Boosts, each x1.5:** Lucky Egg (held-item effect 40); the battle kind's EXP-boost rule (0x2D: **likely** what
   that rule means); a Pokémon from another trainer (traded), which also switches the message to the "boosted" one.
   (Gen 3's x1.5 for trainer battles is not applied separately; Colosseum's rule 0x2D plays that role: **guess**.)
5. **Shadow Pokémon:** no EXP is applied. If its heart-gauge level (`pokemonGetDarkPokemonLevel`) is **3 or more**, i.e. the
   gauge is **below 60%** of its starting value (battle-shadow.md), the amount is **added to a stored-EXP field** (Pokémon status 0xC6), given back when it is purified (**verified**, the purification ceremony applies it, battle-shadow.md "Purification"); above 60% the EXP is simply lost (**verified**).
6. Otherwise: EVs from the fainted species (`fightPokemonGetEffortFromPokemon`), the EXP message (0x7532), then a
   level-up loop: fill the bar to the next level (growth-rate tables), raise stats, friendship (rule 0x27, player's
   Pokémon only), run the level-up script (`lbl_80378724`: messages, new moves), carry the remainder over.

### Prize money (`fn_8021E754`)
1. Only if the battle kind gives money (rule 0x25) and there is a winner to pay.
2. `amount = multiplier x class base x highest level x the winner's money factor`, where the class base comes from
   the beaten trainer's class record (`fn_801FBD40`), the highest level is taken over the beaten trainer's Pokémon
   and over the winner's Pokémon with status 0xCF set, which marks a Pokémon **snagged in this battle** (`fightPokemonBiosGetCatchEntryFlag`, set when the snagged Pokémon joins the party, `fn_80213A78`; **verified**): so snagging a trainer's strongest Pokémon does not lower the prize. The money factor is the winning
   trainer's field 0x48: 1 normally, set to 2 when a Pokémon holding an **Amulet Coin** enters battle (**verified**:
   `fn_8022D084`, battle-items.md). The multiplier is 1 with one Pokémon
   out per side, 2 in doubles, and **4** when the pool rule (0x26) is on.
3. With the pool rule on, the amount goes to `scriptAddPremium` instead of the wallet. The pool
   rule is on in the four story Colosseum battle kinds (battle-engine.md, "The 19 battle kinds"), so it is the
   Colosseum challenge's prize, held back and paid together. The pool is the story variable **0xA9E**
   (`scriptAddPremium` adds to it, `scriptGetPremium` / `scriptSetPremium` are the script side: **verified**), so the
   payout happens in a field script (**likely**; which script not traced); otherwise `heroAddPokedoru` adds it as money and the message shows it.

Differences from Gen 3 (Colosseum side **verified**; Gen 3 side **likely**, from Gen 3's published formula,
not checked against its code): Gen 3 uses the *last* Pokémon's level and a fixed factor 4; Colosseum uses the
*highest* level and 1 / 2 / 4.

### After the battle (`fightActionFlowSyuuryouPost`; **verified**)
For the player's party: every Pokémon that gained a level (status 0xD0) gets an **evolution check** and evolves
right away if it can; if the battle kind allows Pickup (status 0x30), **Pickup** runs for the party; and if the battle
kind's write-back rule (status 0x1C, `BackSaveData`) is on, the battle's copy of the party is **written back to the
save** (`heroBiosCopy`). On the disc that rule is on in the story kinds and off in the battle-mode kinds, so battle
mode never changes your saved party (**verified** from the battle-kind table). The battle kind's Pokérus rule (status 0x28) is
read at that point too, but nothing is done with the value: **Pokérus never spreads** in Colosseum (**verified**:
the only writers of a Pokémon's Pokérus value are Pokémon creation (0), the debug menu's Pokémon editor and the
import of a Pokémon from a GBA game, `gba_misc.c`; so the only way to have Pokérus is to bring it from a GBA game).
When `fn_8006B57C` is 1 (**guess**: battle
mode or link play) the trainers'
Pokémon are put back into their parties (`fightTrainerBackFightPokemonToTemotiPokemon`).

## Disc data

Species base EXP (species table), growth-rate tables (pokemon-data.md (not yet published)), trainer class money base (trainer-class table,
trainers.md (not yet published)), held-item effect ids (item table).

## Changing it

- EXP and money multipliers are code (`fn_802232F4`, `fn_8021E754`); a port option ("EXP x2", "no Exp Share split")
  hooks these two functions.
- Battle kinds switch EXP and money on and off by data (rules 0x24-0x27, 0x2D).

## Efficiency

All integer maths; the level-up loop runs once per level gained, with the EXP bar animation waited on between levels.

## Region notes

None.

## Losing a battle (**verified** from `fightStartEncount` and the US battle table)
Each encounter record has a "whole team fainted" (*zenmetu*) flag (+0x02). When it is set and the player's side does
not win, `fightStartEncount` runs the **blackout** right after the battle: half the money is lost, the party is
healed and the game warps to the return point (field-steps.md). When it is clear, the battle simply returns its
result and the calling script decides what happens. On the US disc the flag is set for all 20 story boss battles
(kind 1), 338 of the 344 ordinary trainer battles (kind 2; the six without it are battles 22-26 and 36) and the test
kind's 8; it is clear for every story Colosseum and Phenac round, Mt. Battle and Battle Mode battle, which (**likely**) handle a
loss themselves. The first battle (battle 10, Willie) has it set.

## Open questions

- Which field script pays out the pooled money (variable 0xA9E).
