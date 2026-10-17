# Orre Engine Docs

Notes on how **Pokémon Colosseum** and **Pokémon XD: Gale of Darkness** work under the hood.

**Use these however you like.** Everything here is released under CC0: no permission and no credit needed. This is just
me (snatcherLeo) documenting what I find about the engine.

## What's here

How the games are built: the shared engine libraries (the HSD / sysdolphin library and Genius Sonority's own engine
layer), the script system, the file formats on the disc, fonts and text, and more.

So far it is mostly Pokémon Colosseum (USA version). The engine-library pages (HSD, graphics, file formats) likely
apply to XD too; the game-specific pages (battles, story, scripts) describe Colosseum.

### Engine: libraries, formats, graphics, memory

- [HSD / sysdolphin (scene graph and rendering library)](engine/hsd.md)
- [HSD models: reading the geometry (maps, objects, NPCs, battle models)](engine/hsd-models.md)
- [Animation tracks (HSD AObj / FObj) and room cameras](engine/animation-tracks.md)
- [Rendering front end (`GSgfx*` / `gfx*`)](engine/gfx.md)
- [Textures, shiny colouring and portraits](engine/textures.md)
- [Particles: GPT1 banks and the particle byte-code](engine/particles.md)
- [Lens flare (GFL, resource type 17)](engine/lens-flare.md)
- [Screen and model effects (GSeffect and the WZX effect types)](engine/effects.md)
- [WZX: move-effect sequences](engine/wzx.md)
- [Collision (CCD files, GScolsys2)](engine/collision.md)
- [Fonts](engine/fonts.md)
- [Disc files (what the player's disc contains)](engine/disc-files.md)
- [Movies (THP video playback)](engine/movies.md)
- [Dolphin SDK (OS, DVD, CARD, ...)](engine/dolphin-sdk.md)
- [Crash handler and fatal errors](engine/crash-handler.md)
- [Memory: the GSmem handle heap](engine/memory.md)
- [Resource registry: naming loaded data by (group, id)](engine/resources.md)
- [Fixed object pools (materials, lights, cameras, gapp tasks)](engine/object-pools.md)
- [Where the game keeps its state](engine/global-state.md)
- [RNG (random number generator)](engine/rng.md)

### Game systems (likely shared with XD)

- [Battle engine: battle setup, move sequences, critical hits and damage](systems/battle-engine.md)
- [Battle engine: turn order, end of turn, weather, AI scoring](systems/battle-turn-flow.md)
- [Battle engine: types and the type chart](systems/battle-types.md)
- [Battle engine: abilities (where each one is checked)](systems/battle-abilities.md)
- [Battle engine: held items (where each effect is checked)](systems/battle-items.md)
- [Battle engine: EXP and prize money](systems/battle-rewards.md)
- [Battle targets (`fightTarget*`)](systems/battle-targets.md)
- [Battle engine: Shadow Pokémon in battle, and the heart gauge](systems/battle-shadow.md)
- [Battle engine: snagging and catching (the ball throw)](systems/battle-snag.md)
- [Battle arenas (where every battle takes place)](systems/battle-arenas.md)
- [The Shadow Pokémon system](systems/shadow-system.md)
- [Evolution: the rules](systems/evolution.md)
- [Friendship](systems/friendship.md)
- [Trading](systems/trading.md)
- [GBA link and Pokémon conversion](systems/gba-link.md)
- [Sending programs to a Game Boy Advance (the PKJB uploader)](systems/gba-upload.md)
- [Player movement in the field](systems/player-movement.md)
- [NPCs (people) in the field](systems/npcs.md)
- [The field camera](systems/field-camera.md)
- [Field: footsteps and step callbacks (poison, friendship, P★DA mail)](systems/field-steps.md)
- [Room changes: links, entry points and arrival](systems/room-changes.md)
- [Story events: what starts them](systems/events.md)
- [Cutscenes (in-engine scenes)](systems/cutscenes.md)
- [Screen transitions (fades and battle-entry wipes)](systems/screen-transitions.md)
- [Script VM (`cms_*`)](systems/script-vm.md)
- [The common script (script group 0x0596)](systems/common-script.md)
- [Messages and text (`GSmsg*`, `msgctrl*`)](systems/messages.md)
- [Message windows (`winMsg*`)](systems/message-windows.md)
- [Menu rooms: full-screen menus are rooms](systems/menu-rooms.md)
- [The field pause menu](systems/pause-menu.md)
- [Name entry](systems/name-entry.md)
- [The P★DA](systems/pda.md)
- [Memory card tasks](systems/memory-card.md)
- [Save data (in memory)](systems/save-data.md)
- [The bag, Rui's pocket and the PC](systems/bag-and-pc.md)

### Colosseum only

- [From power-on to Outskirt Stand: the front-end flow](colosseum/new-game-flow.md)
- [Colosseum challenges (story mode)](colosseum/colosseum-challenges.md)
- [How Rui follows Wes (the partner follower)](colosseum/partner-follow.md)
- [The TV news](colosseum/tv-news.md)
- [Where and when trainers are met](colosseum/trainer-encounters.md)
- [Items placed in the world (treasure)](colosseum/treasure.md)
- [Shops (marts, vending machine, Poké Coupon exchanges)](colosseum/shops.md)
- [Unused and hard-to-see content](colosseum/unused-content.md)
- [FSYS archive catalog (USA disc, GC6E01)](colosseum/fsys-catalog.md)
- [Region and build differences](colosseum/regions.md)
- [Card e: the Virtual Capsule room (Japan-only content left in the US build)](colosseum/card-e.md)
- [Player appearances (where the player is shown)](colosseum/player-appearances.md)

### Porting playbook

- [Bringing a GameCube decomp to life in the PC port (the decompiled code running natively): a playbook](playbook.md)

## How much to trust each claim

Every claim is labelled, so you know how far to rely on it:

- **verified**: checked against the game's code or its behaviour;
- **likely**: strongly supported, but not checked directly;
- **guess**: a working theory, could be wrong.

## How these were made

The docs were researched and written with AI assistance, working from the game's decompiled code and its behaviour,
then reviewed and organised by snatcherLeo. Mistakes are possible; the labels above say how sure each claim is.
Corrections are welcome: open an issue on this repo.

## What's not here

No game files, no extracted game data, no disc images and no game code, only descriptions of formats and behaviour,
plus function and file names from the public decompilations.

## Thanks

Built on the work of the Pokémon Colosseum decompilation (dougchansan and contributors), the Super Smash Bros. Melee
decompilation (the same HSD library), StarsMmd's Colosseum / XD tools and research, and the dtk / objdiff tooling.

Thanks also to TeamOrre and the XD decompilation community.

## License

[CC0 1.0](LICENSE): public domain dedication.

*An unofficial fan research project, not affiliated with Nintendo, The Pokémon Company, Game Freak or Genius Sonority.
Pokémon and the game names are trademarks of their owners.*
