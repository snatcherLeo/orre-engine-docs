# Items placed in the world (treasure)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). How the game places items in rooms (treasure boxes, sparkling items on
the ground, the Ein Files), how picking one up works, and the one place where the contents are **random**: the
Shadow Pokémon Lab's DNA samples. Code: `src/game/floor.c` (room setup), `src/game/floor_event.c`
(`floorEventCtrlTresure`, `floorEventGetTresure`, `floorEventChangeTresure`), the Lab's room scripts.
**Verified** from the decomp's C, the decoded US scripts and the US treasure table unless marked.

## Data: the treasure table (`common_rel`, `lbl_80478EBC`, 74 records of 28 bytes; game-data-tables.md (not yet published))

| Offset | Field |
|---|---|
| +0x00 | one byte: bits 5-7 = **look** (1 = treasure box, 2 = sparkle on the ground, 3 = book); bits 2-4 = **reward kind** (below) |
| +0x01 | quantity |
| +0x02 | facing (degrees) |
| +0x04 | room id |
| +0x06 | **collected flag**: set when picked up; already set = the box shows open / the item is gone |
| +0x08 | **appear flag** (0 = always there): the item is only visible once this story flag is set |
| +0x0C | item id |
| +0x10 | x, y, z |

The three looks are the character models `t_box_00`, `kirari` (with its shadow switched off) and `bbook` (NPC info
59-61), all in `field_common`. In the US table: 45 boxes, 23 sparkles, 5 books; **26** records have an appear flag (the
keys, discs and TMs that turn up once the story gets there, and the Lab's DNA boxes).

**Reward kinds** (`floorEventGetTresure`, after the pick-up jingle):

| Kind | What happens |
|---|---|
| 1 | an item: "found <item>" then "put it in the bag"; if the bag pocket is full, the rest goes to the **PC** with its own message |
| 2 | **money**: the amount is the item field (no US record uses it) |
| 3 | an **Ein File** (items 538-541, 547): it goes to the bag, then the game asks whether to read it now; yes shows the file's text |
| 4 | like 1 without the "found" line (used by a script native that gives items through the same path) |

## Flow
1. **Room setup** (`floor.c`): every record for the current room becomes a character in the room's group (ids
   0x7FFF0000 + n, the n-th treasure of the room) with its model, position and facing. If its collected flag is set,
   it is shown **opened** (box) or hidden (sparkle, book). If it isn't collected and has an appear flag, a small
   **common-script watcher** (common script function 10, common-script.md) keeps it hidden until the flag becomes true.
   Restored rooms skip the setup (menu-rooms.md "fresh / restored").
2. **Examining it** (`floorEventCtrlTresure`, mode 2): already collected = nothing; a box plays its opening animation
   and sound; the item is marked collected (flag set), the look updated, and the reward given.
3. **Scripts can change a record's contents** before the player finds it (`floorEventChangeTresure(index, item,
   count)`; the table lives in RAM, so this lasts until the module reloads).

## The Lab's random DNA samples (**verified**)
The one randomised treasure in the game, and the only use of `floorEventChangeTresure`:
1. When the player uses the **Maingate Key** at the Shadow Pokémon Lab (`D1_out`, function 7), the script draws one
   of **four box layouts** (story flags 2851 / 2852 / 1111 / 1112) and **three different DNA samples** out of nine
   (the "of some sort" samples, items 512-528 even), stored in story variables 1113, 1114, 2721. The random natives
   are named `fadeEffectGetRandom` in the decomp's symbols (a misleading name: it is the game's generic random-number helper with a limit, **likely** returning 0 .. n-1, as the script's checks for 0-3 suggest).
2. Each layout is three treasure records (62-64, 65-67, 68-70, 71-73) spread over `D1_labo_B1` and `D1_labo_B3`;
   on entry both rooms write the three samples into the chosen layout's records. The other layouts' records stay
   empty.
3. The **DNA analyzer** in `D1_labo_B1` identifies a sample (`std205_getDNASampleSepcies`): it swaps the item for its
   named twin (512 -> 513 "BAYLEEF's DNA sample", ...) and says which Pokémon it came from. The nine are Bayleef,
   Quilava, Croconaw, Sudowoodo, Misdreavus, Mightyena, Raikou, Entei and Suicune.
4. The **locked door** in `D1_labo_B2` (function 14) opens the DNA-key screen (menu room 0x387, `DNA_key`,
   menu-rooms.md) with the three drawn samples' species; the screen shows a **3 x 3 grid of the nine species**, and
   the door opens (flag 2637) when the player enters the right three (**likely** for the grid's on-screen use: the
   grid table and its check are verified, the input screen wasn't read).
So every playthrough needs a different code, and the player has to find the samples to learn it.

## For modders
- New or moved items: append records (never insert: room scripts address records by index); look, reward kind,
  appear flag and collected flag are all data. A new item needs an unused collected flag.
- Money pickups (reward kind 2) work but are unused: a mod can place them without code.
- A "randomizer" mod can reuse `floorEventChangeTresure` from a room's entry script, as the Lab does.

## Efficiency
- Treasure isn't loaded per room: the whole table stays in memory with `common_rel`, and each room scans it for its
  own records (74 records, a trivial loop).

## Open questions
- The DNA-key input screen itself (`fn_80087C64`, room 0x387): how the grid is shown and chosen.
- Bits 0-1 of byte 0 (always 0 in the US table).
