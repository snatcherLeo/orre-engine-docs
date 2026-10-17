# Orre Engine Docs

Notes on how **Pokémon Colosseum** and **Pokémon XD: Gale of Darkness** work under the hood.

**Use these however you like.** Everything here is released under CC0: no permission and no credit needed. This is just
me (snatcherLeo) documenting what I find about the engine.

## What's here

How the games are built: the shared engine libraries (the HSD / sysdolphin library and Genius Sonority's own engine
layer), the script system, the file formats on the disc, fonts and text, and more.

So far it is mostly Pokémon Colosseum (USA version). The engine-library pages (HSD, graphics, file formats) likely
apply to XD too; the game-specific pages (battles, story, scripts) describe Colosseum.

### Engine pages

- [Animation tracks (HSD AObj / FObj) and room cameras](engine/animation-tracks.md)
- [The bag, Rui's pocket and the PC](engine/bag-and-pc.md)
- [Battle engine: abilities (where each one is checked)](engine/battle-abilities.md)
- [Battle arenas (where every battle takes place)](engine/battle-arenas.md)
- [Battle engine: battle setup, move sequences, critical hits and damage](engine/battle-engine.md)
- [Battle engine: held items (where each effect is checked)](engine/battle-items.md)
- [Battle engine: EXP and prize money](engine/battle-rewards.md)
- [Battle engine: Shadow Pokémon in battle, and the heart gauge](engine/battle-shadow.md)
- [Battle engine: snagging and catching (the ball throw)](engine/battle-snag.md)
- [Battle targets (`fightTarget*`)](engine/battle-targets.md)
- [Battle engine: turn order, end of turn, weather, AI scoring](engine/battle-turn-flow.md)
- [Battle engine: types and the type chart](engine/battle-types.md)
- [Collision (CCD files, GScolsys2)](engine/collision.md)
- [Colosseum challenges (story mode)](engine/colosseum-challenges.md)
- [The common script (script group 0x0596)](engine/common-script.md)
- [Crash handler and fatal errors](engine/crash-handler.md)
- [Cutscenes (in-engine scenes)](engine/cutscenes.md)
- [Disc files (what the player's disc contains)](engine/disc-files.md)
- [Dolphin SDK (OS, DVD, CARD, ...)](engine/dolphin-sdk.md)
- [Screen and model effects (GSeffect and the WZX effect types)](engine/effects.md)
- [Story events: what starts them](engine/events.md)
- [Evolution: the rules](engine/evolution.md)
- [The field camera](engine/field-camera.md)
- [Field: footsteps and step callbacks (poison, friendship, P★DA mail)](engine/field-steps.md)
- [Fonts](engine/fonts.md)
- [Friendship](engine/friendship.md)
- [FSYS archive catalog (USA disc, GC6E01)](engine/fsys-catalog.md)
- [GBA link and Pokémon conversion](engine/gba-link.md)
- [Sending programs to a Game Boy Advance (the PKJB uploader)](engine/gba-upload.md)
- [Rendering front end (`GSgfx*` / `gfx*`)](engine/gfx.md)
- [Where the game keeps its state](engine/global-state.md)
- [HSD models: reading the geometry (maps, objects, NPCs, battle models)](engine/hsd-models.md)
- [HSD / sysdolphin (scene graph and rendering library)](engine/hsd.md)
- [Lens flare (GFL, resource type 17)](engine/lens-flare.md)
- [Memory card tasks](engine/memory-card.md)
- [Memory: the GSmem handle heap](engine/memory.md)
- [Menu rooms: full-screen menus are rooms](engine/menu-rooms.md)
- [Message windows (`winMsg*`)](engine/message-windows.md)
- [Messages and text (`GSmsg*`, `msgctrl*`)](engine/messages.md)
- [Movies (THP video playback)](engine/movies.md)
- [Name entry](engine/name-entry.md)
- [From power-on to Outskirt Stand: the front-end flow](engine/new-game-flow.md)
- [NPCs (people) in the field](engine/npcs.md)
- [Fixed object pools (materials, lights, cameras, gapp tasks)](engine/object-pools.md)
- [Particles: GPT1 banks and the particle byte-code](engine/particles.md)
- [How Rui follows Wes (the partner follower)](engine/partner-follow.md)
- [The field pause menu](engine/pause-menu.md)
- [The P★DA](engine/pda.md)
- [Player movement in the field](engine/player-movement.md)
- [Region and build differences](engine/regions.md)
- [Resource registry: naming loaded data by (group, id)](engine/resources.md)
- [RNG (random number generator)](engine/rng.md)
- [Room changes: links, entry points and arrival](engine/room-changes.md)
- [Save data (in memory)](engine/save-data.md)
- [Screen transitions (fades and battle-entry wipes)](engine/screen-transitions.md)
- [Script VM (`cms_*`)](engine/script-vm.md)
- [The Shadow Pokémon system](engine/shadow-system.md)
- [Shops (marts, vending machine, Poké Coupon exchanges)](engine/shops.md)
- [Textures, shiny colouring and portraits](engine/textures.md)
- [Trading](engine/trading.md)
- [Where and when trainers are met](engine/trainer-encounters.md)
- [Items placed in the world (treasure)](engine/treasure.md)
- [The TV news](engine/tv-news.md)
- [Unused and hard-to-see content](engine/unused-content.md)
- [WZX: move-effect sequences](engine/wzx.md)

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
