# Battle targets (`fightTarget*`)

Status: **partial** (2026-10-16): the full target-id list, read from `fightTargetGetPtr`. Part of the battle engine ([battle-engine.md](battle-engine.md)). Claims are
**verified** from the byte-matched code and the executable's data unless marked.

## What it does

Battle code never passes "the attacker" or "the opponent's partner" around as raw pointers to its scripts. It names
participants by **target id** (a small number), and `fightTargetGetPtr(id, reference, battleType)` resolves the id to
the live object: the battle floor, a side, a trainer or a Pokémon on the field. Scripts (`WS_*` commands take a target
byte), menus and effects all use these ids. A static table describes each id.

## Data structures

### Target-data table (`lbl_80375AC8`, 30 entries of 8 bytes; count 0x1E in `lbl_80478D40`)
- `+0x00` u8 **kind** of object (`fightTargetDataBiosGetStatusKid`): the object kind of the game's generic status
  system (`status.c`): 5 a Pokémon, 6 a move, 7 the battle floor, 8 a side, 9 a trainer, 4 an item (**verified** from `statusSetStatus`). So entry 0x1B (kind 6) names a move and entry 0x1D (kind 4) an item.
- `+0x01` u8: nonzero = an addressable target (the reverse search only considers these).
- `+0x04` s32 **buffer index** (`fightTargetDataBiosGetBuff`): which slot of a per-side array (status panel, effect
  channel) belongs to the target; 0 or 1 for the field slots, 0x49 / 0x4A for the acting / receiving Pokémon's move
  effect channels; negative = none.

### The target ids (`fightTargetGetPtr`; "own" means the side of the reference object)

| Id | Target |
|---|---|
| 0 | nothing |
| 1 | the battle floor (the whole battle) |
| 2 / 3 | the reference's own side / the other side |
| 4 / 5 | side 0 (the host side) / side 1 |
| 6 / 7 | own side's trainer 0 / 1 |
| 8 | own side's other trainer (the partner trainer in a tag battle) |
| 9 / 0x0A | other side's trainer 0 / 1 |
| 0x0B | the first trainer of side 0 |
| 0x0C / 0x0D | own side's Pokémon on the field, slot 0 / 1 |
| 0x0E | own side's other Pokémon on the field (the partner) |
| 0x0F / 0x10 | other side's Pokémon on the field, slot 0 / 1 |
| 0x11 | the **acting** Pokémon (floor status 0x36: attacker, Call target, whoever an effect is about) |
| 0x12 | the **receiving** Pokémon (floor status 0x42: the move's target) |
| 0x13 | floor status 0x44: the Pokémon that ran (set by the run action) |
| 0x14 | floor status 0x45: the Pokémon switching out |
| 0x15 | floor status 0x46: the Pokémon fainting (set by faint handling) |
| 0x16 / 0x17 / 0x18 | floor status 0x47 / 0x48 / 0x49: the Pokémon a secondary effect applies to / whose ability acts / whose held item acts (**verified** from the getters' names, `...GetTuikakoukaPokemonPtr` / `...GetTokuseiPokemonPtr` / `...GetItemPokemonPtr`) |
| 0x19-0x1D | floor status 0x4B-0x4F: the **appointed** (message-subject) Pokémon, trainer, move, side and item (**verified** from `...GetAppointPokemonPtr` ... `...GetAppointItemPtr`; they match the table's kinds 5, 9, 6, 8, 4) |

Ids 2-0x10 are relative: they need a reference object (the side is found from it, `_fightTargetGetTargetPtrToFightSidePtr`)
and count slots using the battle type (trainers per side, Pokémon out per trainer). Ids 0x11-0x1D are "roles" held in
floor status fields that the engine updates as it goes; that is how one script can say "the attacker" and "the
target" without knowing who they are.

### Move target ranges (move record +0x03, `RangeId`)
Read by `fn_8022B2CC` (pick a move's target) and the accuracy check; the meanings are **verified** from the code's
handling plus the moves on the disc that use each value:

| Value | Targets | Example moves (US disc) | Code |
|---|---|---|---|
| 0 | one target the player chooses | Pound, Supersonic | in doubles the chosen opponent (or a random one when the AI or a re-pick chooses), then **Follow Me** redirection, then **Lightning Rod** (ability 31) pulls Electric moves (type 13) to its holder and marks it (status 0x114) |
| 1 | depends on the move (resolved by its script) | Counter | the first opponent, or its partner if that one has fainted |
| 2 | (no move uses it) | | treated like 5 |
| 3 | a random opponent | Thrash | random opposing Pokémon |
| 4 | both opponents | Surf | as 1 for the primary target; a spread move (accuracy and damage x0.5 in doubles) |
| 5 | the user (or its own side) | Swords Dance, Light Screen, Rain Dance | the user; also **Curse** (range 0) when the user is not a Ghost type |
| 6 | every other Pokémon (opponents and the partner) | Earthquake | as 1 for the primary target; spread |
| 7 | the opposing side as a whole | Spikes | as 1 |

Colosseum stores one enumerated value where Gen 3 uses bit flags (`MOVE_TARGET_*`), but the cases are the same
set (**likely**).

### Trainers on a side
Each side has two trainer slots (`fightSideBiosGetFightTrainerPtr(side, idx)`: `side + 0x64 + idx x 0x28E4`, `idx` 0
or 1, NULL otherwise; two because of tag battles). `fightSideGetValidFightTrainerPtr(side, idx)` returns that trainer
only if `fightTrainerCheckValid` accepts it. The original passes `idx` through to the inner call, so the caller's second argument matters.

### Generic field getters
`fightTrainerGetStatus(ctx, slot, field, idx)` and the other `*GetStatus` functions: one function per object kind
(floor, side, trainer, Pokémon) with a field id; the result is a number or a pointer depending on the field. The
target table's "kind" byte says which family applies to a target.

## Flow

1. A script command or menu has a target id; `fightTargetGetPtrAsNowFightType(id, reference)` resolves it with the
   current battle type.
2. The reverse question, "which id is this object, seen from the host side?", is
   `fightTargetGetTragetPtrToRelativeHostSideFightTargetId(object, battleType)`: it scans the table's addressable
   entries for the id whose resolution is that object (0 if none). Move target choices are stored this way (as ids,
   not pointers) and resolved again when the move runs, which is why a fainted target can be re-chosen
   (battle-engine.md, "How a move picks its script").
3. The buffer index picks the status-panel slot or effect channel for the target.

## Disc data

The target table is static data in the executable (`.data`, 0x80375AC8).

## Changing it

More Pokémon per side (a "triple battle" mod) would need new relative ids and buffer indices, and every
fixed-size array the buffers index (8 field slots, 2 per side) to grow; the table count bounds every scan.

## Efficiency

Resolution is a short walk (2 sides x up to 2 trainers x up to 2 Pokémon); the reverse search scans 30 entries, on
events only.

## Open questions

- None known.
