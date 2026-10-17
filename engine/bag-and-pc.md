# The bag, Rui's pocket and the PC

Status: **partial** (2026-10-17). How the player's items and stored Pokémon are kept: the bag's pockets
and their limits, a cut **Rui's pocket**, and the PC (three Pokémon boxes plus item storage). The save layout itself is
on save-data.md (sections 2 and 3); this page covers the rules. Code: `src/game/hero_savedata_exact_80129280.c`
(`heroItem*`), `src/game/pcbox.c`, `src/game/gs_pokemon_summary.c` (the bag screen), `src/game/gs_event_exec.c`.
**Verified** from the decomp's C and the US text unless marked.

## The bag: one pocket per item kind
Every item's record has a **kind** (item table, pokemon-data.md (not yet published)). The bag picks the pocket from the kind
(`heroItemGetItemKindToItemAryPtr`); every add, remove, "has it" and "room for it" check goes through that one switch.

| Item kind | Pocket | Slots | Max per slot |
|---|---|---|---|
| 1 | Poké Balls | 16 | 99 |
| 2 | Items | 20 | 99 |
| 3 | Berries | 46 | **999** |
| 4 | TMs | 64 | 99 |
| 5 | Key items | 43 | 99 |
| 6 | Colognes (the three scents, opened through the Cologne Case) | 3 | 99 |

Each slot is `{item id, count}`. Two more per-pocket switches go with each pocket (meaning not read). A pocket being
full is the "no room" case (message 0x4263 "There's no room for that item!").

**The bag screen** (`gs_pokemon_summary.c`, page table `lbl_80266918`, 6 pages of 0x4C bytes) shows pages 1-5:
**ITEMS, BALLS, TMs, BERRIES, KEY ITEMS** (L / R turn the page and wrap between 1 and 5). Each page names its pocket
kind, its draw function, title message and colours. The colognes are not a page; they are reached through the Cologne
Case.

## Rui's pocket (cut; **verified** code and text, cut status **likely**)
Page **0** of the bag table has pocket kind -1, which selects a separate list: **10 slots, one of each item**, stored
in the player's save record next to a second name field ("hizuki" in the code is **Rui's internal name**: her models
are `hizuki_a1` / `hizuki_b1`, her battle effects `DoHizukiAi` / `DoHizukiMiyaburi`). The page title is the partner's name. Two event functions move items:
- **give to Rui** (`fn_800138B4`): if her pocket has room, the item goes in and she agrees to use it
  (a Japanese line, 0x4264); if not, she says she can't hold any more and asks the player to take one back (0x425F);
- **take back** (`fn_8001374C`): the item goes into the player's bag (or "no room"), is removed from her pocket, and
  a "received from Rui" line follows (0x4268).
All these lines are **still in Japanese** in the US build, and the page turning never reaches page 0, so the feature
is unreachable in the US game (unused-content.md). What "leave it to me" was for (Rui using held items in battle or in
the field) is a **guess**.

## The PC (`pcbox.c`)
- **Pokémon**: **3 boxes of 30**, named "BOX 1" to "BOX 3" by default (message 0x32C9 with the box number); a name
  may be up to 8 characters (`pcboxSetPokemonBoxName`). Storing a Pokémon (`pcboxAddPokemon`, or a chosen slot)
  copies its record into the first empty slot of the given box, or of any box, and **fully heals it**
  (`pokemonAllKaihuku`). The party menu's "box is full" checks count empty slots per box.
- **Items**: one list of **235 slots, up to 999 each**, not split by kind. The coupon exchanges deliver their prizes
  here (`fn_80029EF4`: coupon currency -> PC item storage) (**likely**, from the shop code; shops.md), and so does
  any treasure that doesn't fit in the bag (**verified**, `floorEventGetTresure`, treasure.md).
- **Names swapped** (**likely**): `pcboxAddItem` calls the slot-removal helper (`fn_80140ACC`, the one
  `heroItemDecItemDataId` uses) and refuses when the item isn't stored, while `pcboxDelItem` calls the slot-adding
  helper (`fn_80141308`, used by `heroItemAddItemDataId`) and works on items not yet stored. So the two names are
  the wrong way round. Evidence for which helper adds: the bag returns "the amount that didn't
  fit" from `fn_80141308` ("no room" when it's above 0), and treasure passes exactly that remainder to `pcboxDelItem`
  to store it in the PC.

## Adding an item (**verified** from code and a native run)

`heroItemAddItemDataId(hero, item, amount, index)` (what the script native `scriptAddItem` calls) finds the bag pocket
from the item's **kind** (`itemGetStatus(0, item, 2, 0)`: e.g. 2 for a Potion's pocket, 1 for Poké Balls') through
`heroItemGetItemKindToItemAryPtr` (pocket slot count, the per-item maximum 99, sort flags), then inserts with
`fn_80141308`. **It returns how many did not fit** (0 = everything added), or -1 when there is no pocket or the item id
is invalid (an id is valid when it has a name and is below the item count, 548 in the USA game). So a caller checks
"`>= 0` and less than the amount", not "non-zero". New items go in a free slot and the pocket is re-sorted; an item
already present is topped up to the maximum first.

## For modders
- Pocket sizes and caps are constants in one switch; a "bigger bag" option or a new item kind changes it in one
  place (plus the bag page table for a new page).
- More PC boxes: the count (3), the box size (30) and the 0x24A4 stride are constants in `pcbox.c`; the save section
  would grow, so a PC port needs its own save format to add boxes (save-data.md).
- **Restoring Rui's pocket**: add page 0 to the page turning and translate the four lines. The behaviour behind "leave
  it to me" doesn't exist in the code read (nothing found reading her pocket in battle), so it would be new design.

## Open questions
- The two per-pocket switches (which pockets allow sorting / registering, **guess**).
- Whether any bag mode opens directly on page 0 (Rui's pocket).
- The second "hizuki" name field (0x56 bytes) in the player record: Rui's name, though she can't be renamed
  (**guess**).
