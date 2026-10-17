# Changelog

The docs are published in batches. Each entry lists what changed for readers: new pages and notable corrections.

## 2026-10-17: third batch

- New page: **Player appearances** (`colosseum/player-appearances.md`): the places Colosseum shows the player
  character, and what each one loads.
- **Names recovered from the assert text** (`engine/crash-handler.md`): function, variable and constant names the
  game's own error checks reveal.
- **Item friendship amounts are stored twice** (`systems/friendship.md`): once in the item's parameter record and
  again in the item record; a change must update both.
- **Call has no accuracy effect** in Colosseum (`systems/battle-engine.md`, `systems/battle-turn-flow.md`), and much
  more on battle turns and their order from the first battle playing on the PC.
- Additions: how script helpers receive their starting values (script-vm); the Card e font as a supplement to the main
  font (fonts); RNG details (rng); memory and graphics notes (memory, gfx).
- Corrections: which battles the scripts start and which are still untraced (trainer-encounters); battle 561 is the
  Card e card battle (card-e).

## 2026-10-17: second batch (new layout)

- **New layout:** the pages now live in three folders: `engine/` (shared libraries, formats, graphics, memory),
  `systems/` (game systems likely shared with XD) and `colosseum/` (Colosseum-only content). Links to first-batch page
  addresses (`engine/<page>.md`) no longer work for pages that moved.
- New page: **Card e** (`colosseum/card-e.md`): the e-Reader battle cards, their room and menu in the US game, and how
  a card battle would start.
- Every page now starts with an **Applies to** line: how far it holds for Colosseum (verified, USA) and for XD (likely
  or guess: nothing here has been checked against XD yet).
- Corrections: the type labels and Battle Mode's type badges are two different sprite sets (battle-types); the
  `bg*thumbcode` files are the e-Reader's Card e program (gba-upload, gba-link, disc-files); how Pokémon reach the
  Strategy Memo (pda); variable 0x8AE is the game mode (pause-menu); the Card e room's trainers (battle-engine); two
  friendship events are never used (friendship).

## 2026-10-17: first batch

- First publication: 66 pages on the engine of Pokémon Colosseum (USA): the HSD library and the graphics layer,
  memory and resources, the script VM, the disc and file formats (archives, textures, fonts, messages, movies, move
  effects, particles, collision), and the game systems (battles, Shadow Pokémon, snagging, menus, the field, saves,
  trading, the story's start), plus a porting playbook for other decompilation teams.
- Still to come in later batches: Pokémon and trainer data, flags, music, room data, the story route and a few
  more pages (marked "not yet published" where other pages mention them).
