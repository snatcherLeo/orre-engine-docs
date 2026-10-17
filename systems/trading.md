# Trading

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). How a Pokémon trade works on the GameCube side. Collected from the
menu-room, NPC-event and Pokémon code; **verified** from the decomp's C and the decoded room scripts unless marked.
The GBA side of the link is in gba-link.md and gba-upload.md.

## What it does

Trading is a full-screen menu room, `pokemonchange_menu` (room 0x394), opened by script command 198
(`std198_menuEnterTradingMenu`). On the US disc the only script that opens it is Phenac City's Pokémon Center
basement (`M1_pc_B1`), right after the player saves, and the trade's own messages speak of the GBA being disconnected, so the partner is
a **linked GBA game** (**verified** from the messages) (the
partner's party is read from a second player record filled by the link code).

## Data structures

| Global (proposed) | Address | What |
|---|---|---|
| `gTradeState` | 0x8047A42C | the trade's state (0 = leave, 1-19 as in Flow) |
| `gTradeCursorSlot` | 0x8047A428 | the slot picked in the selection menu (-1 = none) |
| `gTradeSlotOwn` | 0x8047A424 | the player's party slot offered (-1 = none) |
| `gTradeSlotPartner` | 0x8047A420 | the partner's party slot (-1 = none) |
| `gTradePartnerHero` | 0x803A2688 | the partner's player record (0xB60 bytes; its party is read with `heroGetStatus(record, 3, slot)`) |

The selection screen shows two groups of six entries (`fn_800318D8`): group 1 is the player's side (highlighted by
`gTradeSlotOwn`), group 2 the partner's (`gTradeSlotPartner`).

## Flow
(**verified**, the exchange state machine in `gs_npc_event.c`; message contents paraphrased):
1. The script saves, then opens the trading room (command 198).
2. (**verified**, `gs_npc_event.c` and `menuPokemonChange.c`): **choosing.** State 1 asks the
   link code to start (`fn_800849B4`; a failure ends the trade). State 2 is the selection menu (0xD9,
   `fn_8002FC58`): one list holding the player's six party slots and the partner's six; a slot can be picked only if
   it holds a valid Pokémon that passes the event check (`menuCBRule_CheckPokemonEventFlag`, 0x80075FEC: **Mew (species 0x97) and Deoxys
   (0x19A) pass only with their event flag set**, the Gen 3 "fateful encounter" bit; every other species passes;
   the same rule as Battle Mode's, pokemon-data.md (not yet published) "Deoxys in detail"). Picking a Pokémon
   opens a two-choice menu (state 3): offer it (state 6) or view its summary (state 5, `fn_80097CD0`); B goes to the
   quit question (state 4: quit -> end, otherwise back to the list).
3. **Checks on the player's offer** (state 6, `fn_8002F79C`), each with its own refusal message and a return to the
   list: the held item must be one the item table marks **exportable** (`itemDataBiosCheckExportable`), the Pokémon
   must **not be a Shadow Pokémon**, and the player must **keep at least one other usable Pokémon**
   (`menuPokemonChangeHasOther`; it skips eggs and Pokémon flagged illegitimate, **likely** from the helpers it calls).
   Passing records the slot and moves to the partner's side: **state 7** (`fn_8002F284`) reopens the same selection
   menu with input taken from **port 2** (`menuSetEnablePort(2)`), so the partner picks with the linked GBA's buttons
   (the window system does read a linked GBA's keys as a port's input, message-windows.md
   "Window input keys", **verified**; that the GBA is on port 2 is **likely**); cancelling there goes to state 9, which clears the player's
   offer and returns to the list.
4. **Checks on the partner's Pokémon** (state 8, `fn_8002EE74`): refused if it fails the event check or is an egg
   (message 0x43E1) or carries the **illegitimate** ("fusei") flag (message 0x44BE; npc-event-windows.md (not yet published)). (**verified**, `gba_misc.c`): That flag is the Gen 3 **Bad Egg** bit, copied from the GBA game's data when a
   Pokémon is converted (`pokemonBiosSetFuseiFlag(.., isBadEgg)`; also status id 0xB8), so this check refuses
   Bad Eggs; then the
   same offer / summary choice for the partner's Pokémon (state 10 shows its summary through input port 2). A lost
   link in these states shows the "GBA disconnected" message (0x4448) and ends the trade.
5. States 11-13: the confirmation screen (menu 0xDE), "start trading" naming both Pokémon and both trainers, and a
   farewell to both Pokémon.
6. State 14: the exchange itself (`stateFunctionExChangeMain`); state 15 shows "the trade failed, check the
   connections" on the failure path.
7. State 16 (**the final step**): both traded Pokémon's **friendship is set to 70**, the Pokémon the player receives
   is **added to the Strategy Memo** (pda.md, if its species is new), and the "was traded to" message is shown.
8. State 17: **evolution** (`stateFunctionEvolution`): the trade check, `pokemonEvolutionCheck(.., mode 2)`, on
   **both** traded Pokémon (evolution.md, kinds 5 and 6).
9. State 18: **save** (`stateFunctionSaveReport`; **verified** from its messages): memory-card
   task kind 8 saves the card in slot A and the GBA Game Pak **together**; only result 4 confirms the trade, any
   failure restores the save image copied before the exchange and says the trade was not saved (memory-card.md).
10. State 19: asks whether to trade other Pokémon.

## Changing it
- What can't be traded: Mew and Deoxys without the event flag (either side), Shadow Pokémon, Pokémon holding non-exportable items, the player's last usable Pokémon, and
  from the partner eggs and Bad Eggs. These rules are code (state 6 and 8 checks); the exportable bit is
  item data.
- A port without a GBA: the partner record must be filled some other way (another save file, a network peer); the
  rest of the exchange works on two player records.
- The friendship reset to 70 is in the exchange code, not in data.

## Open questions
- Whether the GBA is always on port 2 in a trade (a Dolphin check with a linked GBA would settle it).
