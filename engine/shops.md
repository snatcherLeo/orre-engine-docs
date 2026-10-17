# Shops (marts, vending machine, Poké Coupon exchanges)

Status: **partial** (2026-10-16): the stock and dialogue tables are decoded; prices, the buy/sell flow and
the coupon balance are not written up yet.

## What it does

Every shop in the game, from the Outskirt Stand counter to The Under's vending machine and the Poké Coupon exchange,
is one record in a small table. A room script opens a shop by number (`menuShopOpen(n)`, the script native that
stores `n` and switches to the shop menu); the record says which kind of shop it is, which clerk lines to show, and
where its stock list starts. The tables live in a tiny loadable module, not in the main executable, so they are only
in memory while the shop menu runs.

## Data structures

All tables are in the **pocket_menu module** (`pocket_menu.fsys`, the type-14 REL entry), data section 3. Its prolog
(`src/rel/pocket_menu/pocket_menu.c`) stores a pointer to each table into a main-executable global, and the shop code in
`src/game/menuShop.c` reads only those globals (*verified*: relocations of the module plus the readers below).

| Global | Section-3 offset | What it is |
|---|---|---|
| `lbl_80478E54` | +0x1D8 | **shop records**, 4 bytes each, indexed by shop number |
| `lbl_80478E44` | +0x0E4 | **stock lists**: u16 item ids, each list ending in 0 |
| `lbl_80478E3C` | +0x210 | **shop dialogue sets**, 0x3C bytes each |
| `lbl_80478E4C` | +0x000 | **coupon-exchange dialogue sets**, 0x4C bytes each |
| `lbl_80478E50` / `E40` / `E38` / `E48` | +0x308 / +0x304 / +0x30C / +0x300 | counts: shops, list u16s, 0x3C sets, 0x4C sets |

Shop record (*verified* from the readers `shopOpenItemList`, `fn_80029FAC`, `fn_8002A0B8`, `std026_unkMenuSequence26`):

| Offset | Size | Meaning |
|---|---|---|
| +0 | u8 | dialogue set number |
| +1 | u8 | **kind**: 0 = mart (menu 0x62), 1 = the vending machine (`fn_8002CE6C`), 2-4 = coupon exchanges (menu 0x83) |
| +2 | u16 | start of the stock list, counted in u16 items from the list base |

Kind names are *likely*: the dispatch is verified from code, the names come from the dialogue text each kind's shops use.
Several shops can share one list (the three coupon-exchange records all point at the same list).

Dialogue set: one byte, then u32 message ids (14 for the 0x3C sets, 18 for the 0x4C sets); 0 means "no line". Kinds 0
and 1 index the 0x3C sets (mart greeting, vending machine, a second clerk style, the herb shop), kinds 2-4 the 0x4C
sets (the coupon counter, GBA-game coupons, Colosseum coupons) (*likely*, from the text of each set). The meaning of the
first byte of each set is open.

## Flow

1. A room script calls `menuShopOpen(n)` (`src/game/menuShop.c`): it stores `n` in `lbl_8047A3FC` and fades to the
   shop menu.
2. The shop sequence (`std026_unkMenuSequence26`) reads record `n` and branches on its kind.
3. `shopOpenItemList` takes the list start from the record, counts items up to the 0 terminator and opens the list menu
   (custom menu 0x60).
4. Clerk lines come from the dialogue set through the text formatters `fn_80029FAC` / `fn_8002A0B8`.

Which room opens which shop is in the room scripts (*verified*, script dump): the Outskirt Stand counter passes a
story-dependent variable (likely shops 0-2, whose stock grows from status heals to balls), Phenac City's mart floors 3
and 4, Pyrite 5, Agate 6, The Under's mart 7, its herb shop 8 and its vending machine 10, Mt. Battle's counter 13. Shops
11 and 12 are opened by the Poké Coupon menu (`menuPokeCouponMain`, `src/game/menu/menu_poke_coupon.c`: choice 0 opens
11, choice 1 opens 12, then it returns to room 0x395, `colosseumbattle_menu`, the Battle Mode menu) (*verified*, code + floor table;
new-game-flow.md): the Poké Coupon exchange of Battle Mode, outside story mode (*likely*). Every shop, 9 included, is also reachable from a
developer test menu (`fn_8000CF68` next to `dbgMenuMenuTestPokemonMenu` in `src/game/gs_party_access.c`, a 14-entry key →
shop table at `lbl_80266700`) (*likely* debug-only). Shop 9 (the TM list alone) is opened by no room script and no
player menu found so far (*guess*: unused in the retail game).

## Disc data

`pocket_menu.fsys` → type-14 module, section 3. Prices are not here: they are the item table's own fields (price and
coupon price). Each stock item's shown price is the coupon price
(item record +0x08) at the coupon exchanges, the normal price (+0x06) elsewhere (*likely*: the coupon-only items such as
Choice Band carry a large coupon price and a token normal price).

## Changing it

- Add or change stock: edit a stock list (keep the 0 terminator), or point a record at a new list; the count at +0x304
  is only the list area size.
- A new shop needs a new record (and the shop count at +0x308) plus a script call with its number.
- Because the module is relocatable, growing a table moves everything after it; a mod tool should rebuild the section
  and its relocations rather than patch in place.

## Efficiency

The whole shop data is under 1 KB and is only loaded with the pocket_menu module.

## Open questions

- Prices and the sell flow (sell price = half the item price is the series rule; not checked here).
- The first byte of each dialogue set; why the coupon exchange has three kinds (2, 3, 4) with one list.
- Whether shop 9 is reachable outside the debug menu.

## Region notes

Message ids only; the tables hold no text, so they should be the same in other regions (*guess*, not compared).
