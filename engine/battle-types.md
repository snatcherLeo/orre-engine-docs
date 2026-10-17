# Battle engine: types and the type chart

Status: **partial, well covered** (2026-10-16). How types are stored, how the type
chart works, the unused "???" type, and what adding a type would need. Claims are **verified** from the byte-matched
code and the executable's data unless marked. *Zokusei* = type.

## What it does

A type is a small number, 0-17, Gen 3's numbering: 0 Normal, 1 Fighting, 2 Flying, 3 Poison, 4 Ground, 5 Rock, 6 Bug,
7 Ghost, 8 Steel, **9 "???"**, 10 Fire, 11 Water, 12 Grass, 13 Electric, 14 Psychic, 15 Ice, 16 Dragon, 17 Dark.
Moves have one type, species two. One table in the executable describes each type: whether its moves are physical
or special, its icon, its name, and its row of the type chart. The chart is **identical to Gen 3's**.

## Data structures

### The type table (`lbl_8035B500`, 18 records of 0x2C bytes; count 18 in `lbl_80478B38`; accessors in `gs_colsys.c`)

| Offset | Field | Accessor |
|---|---|---|
| +0x00 | u8 **category**: 1 physical, 2 special, 0 neither | `fn_8010C4A0` |
| +0x02 | u16 **icon** id (used by the GBA-link screens) | `fn_8010C46C` |
| +0x04 | u32 **name** message (developer-style short names: NORMAL, FIGHT, ..., ELECTR, PSYCHC, ?) | `fn_8010C4D4` |
| +0x08 | u16[18] **type-chart row**: the result of this type attacking each defending type | `fn_8010C508` (`zokuseiGetWazaJoutai`) |

Category on the disc: types 0-8 physical, 10-17 special, "???" 0. So **the physical / special split is data** in
Colosseum (Gen 3 hard-codes "type below ???" = physical); the damage formula, the type-boosting items, Hustle's
accuracy rule and the AI all ask this byte.

### The chart values
Each entry is a **result id** (shared with the move-condition ids the battle uses for messages): 0x3F neutral (x1),
0x41 super effective (x2), 0x42 not very effective (x0.5), 0x43 no effect (x0). The type check applies one entry per
defending type (`fn_802279F8`: damage x 20 / 10, x 5 / 10 or x 0, at least 1 unless immune) and records the result,
which drives the "super effective" / "not very effective" messages, Wonder Guard (only result 0x41 hits) and the
"no effect" handling. **Verified identical to Gen 3's chart**: all 110 non-neutral pairs match the public Emerald
source's `gTypeEffectiveness` (pret/pokeemerald, read 2026-10-16), with no differences. Gen 3 keeps the chart as a
list of (attacker, defender, multiplier) entries with a Foresight separator; Colosseum stores the full 18 x 18 matrix
and handles Foresight in code (battle-engine.md, "Type effectiveness").

### Menu icons (`lbl_802EDB40`, 18 u16 sprite ids)
The battle and summary menus draw a move's type with `windowDrawSprite(.., lbl_802EDB40[type], ..)`: sprite ids
0x2C9-0x2D9, one per type. The type's colour is part of the sprite (**likely**: no separate colour table was found).

### Where type ids appear
Move record +0x02 (u8), species record +0x30 / +0x31 (two u8), the battle Pokémon's current types (changed by
Conversion, Camouflage, Color Change, Transform), the terrain table (Camouflage), the type-boosting held-item table
(17 item / type pairs, `lbl_80279ED0`), and many direct comparisons in code (below).

## The "???" type (id 9)

- Name "?", category 0 (neither physical nor special), icon: **none of its own**, it reuses Steel's sprite (0x2D5).
- Its chart row and column are all neutral: nothing is strong or weak against it, and its moves hit everything
  normally.
- Used by exactly one move in the data, **Curse** (move 174), and by no species (both checked against the US disc's tables); Curse's script deals no damage, as in Gen 3. Struggle and
  Shadow Rush are not "???": their data type is Normal (0), and they skip the type check entirely (battle-engine.md).
- Code skips it on purpose: the AI's type-quality score loops over all types except 9 (`fn_8010C54C`).
- So slot 9 is a complete, unused type entry: the natural place for a new type (below). XD, the sequel, does add a
  Shadow type; whether it uses this slot is for the XD decomp to show (**guess**).

### Hidden Power (`pokemonGetMezamerupower`, 0x80120B00; **verified** from the target assembly)
Gen 3's formula, unchanged. It reads the six IVs (status 0x93-0x98); with *a* = the lowest bit of each IV and *b* the
second-lowest, taken in the order HP, Attack, Defense, Speed, Sp. Atk, Sp. Def as bits 0-5:
- **type** = *a* x 15 / 63 + 1, then +1 if that reaches 9: Fighting (1) to Dark (17), never Normal or "???";
- **power** = *b* x 40 / 63 + 30: 30 to 70.
The bit order also fixes the IV selectors: 0x93 HP, 0x94 Attack, 0x95 Defense, 0x96 Sp. Atk, 0x97 Sp. Def, 0x98 Speed
(the same order as the species record's base stats).
(**verified** from every caller in the C): Only **battle code** calls it: the move's type and
power when it is used, and the AI's damage, hit and value functions (so the AI knows a Pokémon's real Hidden Power
type). **No menu code** calls it, so the summary and move screens show Hidden Power with the move record's own
type (Normal), as in the Gen 3 games (**likely**; not seen on screen). A port option "show Hidden Power's type"
would call it from the move list's type icon.

## Flow
1. A move's type comes from its record (or is replaced: Hidden Power, Weather Ball, ...).
2. Damage: STAB if the type matches either of the user's current types; the category byte picks Attack / Defense
   or Sp. Atk / Sp. Def; type-boosting items by the item / type table; then the chart, per defending type
   (battle-engine.md, "Damage" and "Type effectiveness").
3. The AI estimates damage with the same code, so the chart reaches the AI automatically; it also rates a type with
   `fn_8010C54C` (+10 / -10 per strong / weak matchup, skipping "???") and checks specific types in many value
   functions (for example Rain Dance counts the Water types on its side).

## Changing it

**Changing the chart or a type's category: data only.** One u16 per pair in the type table, one byte per type for
physical / special.

**Adding a type: what it needs.**
- *Reusing slot 9 ("???"): mostly data.* Set its category, give it a name message, fill its chart row and its column
  in the other 17 rows, give it a menu sprite (a new sprite id in `lbl_802EDB40[9]` plus the sprite itself; today it
  points at Steel's) and an icon id for the GBA screens. Then give moves and species that type id. Code that would
  still treat it specially: the AI's `fn_8010C54C` (skips 9), Curse (the only "???" move: it would become the new type, which
  matters for STAB, type-boosting items and the menu icon, so re-type it, e.g. to Ghost as later games did), and Hidden Power's type formula (it never produces Normal or ???, see "Hidden
  Power": a type in slot 9 would need that formula changed to be reachable).
- *A 19th type: code changes.* The chart is a fixed-width matrix: the lookup refuses defending types >= 18
  (`typeB >= 0x12`) and every record is 0x2C bytes (8 + 18 x 2), so widening it means a new record size and a new
  bound in `fn_8010C508`; the count (`lbl_80478B38`) and the menu sprite table (0x24 bytes = 18 entries) must grow;
  every 18-entry or type-indexed array elsewhere must be found and widened (the type-boosting item table has 17
  rows; the AI's per-type checks; messages).
- *Behaviour tied to type ids in code* (would not apply to a new type unless added): the type-based abilities
  (Overgrow / Blaze / Torrent / Swarm, Thick Fat, Flash Fire, Volt / Water Absorb, Lightning Rod, Levitate), weather
  (rain / sun scaling Water and Fire), Sand / hail immunities (Rock, Steel, Ground / Ice), Mud Sport / Water Sport,
  Charge (Electric), the Net Ball (Water, Bug), Arena Trap / Magnet Pull (Flying, Steel), Foresight (Normal and
  Fighting into Ghost), Curse's Ghost check, Camouflage's terrain table, and the AI value functions that test types.
- *For the port:* keep the table-driven parts as data files (the type table, the sprite table) so a mod can add a
  type by data, and route the hard-coded type checks above through one "type traits" layer.

## Disc data
Type table, chart and menu sprite table: in the executable (`main.dol` data). Type names: message ids into the game
text. Sprites: the window / menu sprite resources (which archive not traced).

## Efficiency
One array lookup per (attacking type, defending type); the full matrix trades 648 bytes for no searching (Gen 3
walks a list).

## Region notes
The type names are message ids, so they follow the language of the text archives; the sprite ids are shared
(**likely**; the type icons in the US version show English text: a localised resource).

## Open questions
- Which sprite archive holds the type icons (0x2C9-0x2D9), and whether any palette colour per type exists in code.
- Whether Shadow Rush is shown as Normal or "???" in the move menu (data says Normal: a visual check).
