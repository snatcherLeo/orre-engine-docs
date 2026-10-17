# Battle engine: held items (where each effect is checked)

Status: **partial** (2026-10-16). Companion to [battle-engine.md](battle-engine.md) and
[battle-abilities.md](battle-abilities.md). *Soubi* = held item.

## What it does

Every item has a **held-item effect id** (item table +0x0A, `itemSoubiDataId`, pokemon-data.md (not yet published)) and a parameter (the
item's +0x04 value: the HP a berry restores, Quick Claw's percentage, a type item's boost). Battle code compares the
effect id where it matters. **The effect ids are Generation 3's** (`HOLD_EFFECT_*`: 1 restore HP ... 66 Stick);
**verified** as a pattern: each id below is checked where that Gen 3 effect acts, and the dispatchers' case lists are
exactly Gen 3's groups. Like abilities, held items have no data hooks: behaviour is code.

## Flow

### The item dispatchers (Gen 3's `ItemBattleEffects` split by moment; case lists **verified** from the code)

| Function | When | Effect ids handled |
|---|---|---|
| `fn_8022D084` | switch-in (`KaisiPost` and send-out) | 23 White Herb (resets lowered stat stages to neutral), 32 **Amulet Coin** (sets the trainer's money factor, field 0x48, to 2: battle-rewards.md) |
| `fn_8022BE2C` | end of turn and HP checks | 1 HP-restoring berries, 2-6 status-cure berries (paralysis, sleep, poison, burn, freeze), 7 PP berry, 8 confusion cure, 9 any-status cure, 10-14 the five flavour HP berries that confuse a Pokémon that dislikes the flavour, 15-21 pinch stat-up berries (Attack, Defense, Speed, Sp. Atk, Sp. Def, critical rate, random stat), 23 White Herb, 28 Mental Herb (cures infatuation), 43 **Leftovers** |
| `fn_8022B5C8` | right after a status is inflicted | 2-6, 8, 9 (cure berries), 23 White Herb, 28 Mental Herb |
| `fn_8022BB84` | after a hit | 30 King's Rock (flinch chance), 62 Shell Bell (heals part of the damage dealt) |

Effect names are Gen 3's for those ids (**likely** for each name; the grouping is **verified**).

### Direct checks elsewhere

| Id | Item | Checked in |
|---|---|---|
| 22 | BrightPowder (evasion) | `fightCheckAccuracy` |
| 24 | Macho Brace | effective Speed (halved) |
| 25 | Exp Share | `fn_802232F4` (EXP) |
| 26 | Quick Claw | effective Speed (battle-turn-flow.md) |
| 29 | Choice Band | damage (x1.5 Attack), move lock (`fightOutPokemonCheckCanOutOkWazaBanme`) |
| 34 / 35 / 36 | Soul Dew / Deep Sea Tooth / Deep Sea Scale | damage |
| 39 | Focus Band | `fightCheckOneHitKO`, and survival at 1 HP with the item's value as the chance (`fightApplyDamageVariance`, **verified**) |
| 40 | Lucky Egg | EXP x1.5 |
| 41 / 63 / 66 | Scope Lens / Lucky Punch / Stick | critical check |
| 45 / 64 / 65 | Light Ball / Metal Powder / Thick Club | damage |
| 0 (none) and item 175 | holding nothing / the Enigma Berry | item theft (`fn_80215CF0`): an Enigma Berry can't be stolen (**likely**: item 175 is the Enigma Berry in Gen 3's item list) |

The type-boosting items (31, 42, 46-60: Silver Powder, Metal Coat, Soft Sand, ...) go through the 17-row item / type
table in the damage formula (battle-engine.md).

### Using an item from the Bag (**verified** from code unless marked)
`fightActionFlowFightTrainerUseItem`: the item (item field 0x1E) is applied to the chosen party member, or, for items
whose kind field (item field 2) is 1, to a target picked like a move's target (stored as a target id, item field
0x1F); then the use script runs (`fn_80211E18`). For scoring and menus the battle sorts items into **battle-use
kinds** (`fightSeqGetItemType`, from the item's effect data):

| Kind | Items |
|---|---|
| 1 | item 19 only (**Full Restore**: HP and status, by Gen 3's item numbering, **likely**) |
| 2 | HP restoring |
| 3 | status curing (six separate cure flags: **likely** sleep, poison, burn, freeze, paralysis, confusion) |
| 4 | one effect flag (**guess**: revival or PP) |
| 5 | five effect flags (**guess**: the X items raising a stat) |
| 6 | one effect flag (**guess**: Guard Spec. or Dire Hit) |
| 7 | no battle effect (not usable in battle) |

**Shell Bell** (**verified**, `fight_range_exact_8022BB84.c`): heals the holder by the damage dealt divided by the
item's parameter (at least 1 HP), after a hit that did damage to another Pokémon, unless the holder's HP is full or it
is no longer out in battle (`fightOutPokemonCheckFightOut`). The divisor is data (Gen 3's value is 8, a 1/8 heal; the
US value not read).

**Field and evolution effects:** 27 (Soothe Bell) is the friendship boost (x 1.5, friendship.md) and 38 (Everstone)
blocks every evolution (evolution.md), both outside battle as in Gen 3. 37 (Smoke Ball) has nothing to act on:
running has no escape roll in Colosseum (battle-turn-flow.md 3b), so it does nothing (**likely**: no check for it
was found).

## Disc data

Held-item effect id and parameter per item: the item table (pokemon-data.md (not yet published), "Item entry"). The dispatchers and
multipliers are code.

## Changing it

- Changing what an existing item does: change its effect id or parameter in the item table (data only), as long as
  the new combination is one of the effects above.
- A new effect needs code in the right dispatcher; the four dispatchers are the natural hook points for a port's
  "item hook" layer.

## Efficiency

One `switch` per moment, called per Pokémon; no allocation.

## Region notes

None.

## Open questions

- Effects 33 (Cleanse Tag), 44 (Dragon Scale) and 61 (Up-Grade): whether Colosseum's battle code checks any of
  them was not surveyed. 44 and 61 are trade-evolution items, matched by item id (evolution.md, kind 6), not by
  effect.
