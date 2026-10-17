# Unused and hard-to-see content

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**no**: Colosseum's own content)

Status: **partial** (2026-10-17). Every claim of The Cutting
Room Floor's "Pokémon Colosseum" article (TCRF; read from a saved copy on 2026-10-17), checked against our
decoded US data and code, plus what we found that TCRF doesn't list. TCRF is credited as the source of each claim;
the wording here is ours. Results: **confirmed**, **differs** (TCRF's claim is not quite right; the finding says
how), **not found**, or **not checked yet**. **R** marks a good candidate for a restoration mod.

## TCRF's claims, item by item

### Debug

| TCRF claim | Our finding | Label |
|---|---|---|
| A small debug function (enabled by Action Replay codes): an on-screen bar, L cycles pause / slow / play, R pauses | **Confirmed in part, and bigger than described.** The code has a whole **debug menu** (`dbgMenu*`): camera tools, battle editing (forced win / loss / time-out, moves, weather, trainer AI data and score tables, Pokémon slots), menu tests (name entry, Pokémon menu, evolution, the Mt. Battle "present" room), message tests, a move-animation debugger, and a **`GSvtr`** ("video tape recorder") switch, which **likely** is TCRF's pause / slow / play. The codes themselves were not tested. | verified (functions exist), likely (the VTR match) |
| A room `waza_viewer` (move viewer) that reuses the Phenac Stadium map, with debug mode on | **Confirmed**: the `waza_viewer` archive holds its own module and the `M1_water_colo` map (fsys-catalog.md); the build-kind setting can start the game there (regions.md). | verified |

### Graphics (checked 2026-10-17; gameplay side here)

| TCRF claim | Our finding | Label |
|---|---|---|
| `face412` is a placeholder (a desaturated Bulbasaur portrait, no shiny) | **Confirmed**: the face archive's entry 412 is one past the last species (411); its picture is Bulbasaur's portrait with a black shiny cell, and no face record points to it. | verified |
| Stadium 2 leftover eye and mouth textures | **Partly confirmed.** Expressions are vertical sheets of 4-5 cells per model (for example Meowth's five mouths), and a check by eye confirmed that TCRF's Meowth mouth is one of those cells. Which cells the game never selects isn't proven (that needs the texture-offset animation tracks), and such sheets exist in all three generations, so having one isn't by itself a leftover. No model checked has an image that no material uses. | partly verified |
| Shiny Celebi icons exist but can't be seen | **Confirmed**: Celebi has normal and shiny portraits and body pictures, like every species, and the game's own Celebi gift (`heroPokemonGetCelebi`: level 10, a fixed trainer id) rerolls its personality value with "shiny" off, so it is **never shiny** (the Pikachu gift does the same). | verified |
| Postgame trainer texture variants (Myth, Malpes, Pian) | **Confirmed** (matched by eye to TCRF's descriptions): they are separate field models packed into the room archives where each trainer was meant to appear, and **no NPC placement uses them** (none of the 794): Myth = `thief_m_b3` (dark skin, white suit with blue; in Pyrite Cave, Relic Cave, The Under's subway, Realgam Tower 2F and the Snagem hideout, exactly the rooms of Myth's unstarted battles), Pian = `ojo_b4` (pink dress, red eyes; Pyrite entrance and The Under's house A), Malpes = `obo_b4` (yellow jacket, green eyes; Realgam Tower lobby). A script could still spawn them by model id (not ruled out). **R** (with the postgame trainers) | verified (unplaced), likely (unused) |
| Unused VS-screen portraits | **Confirmed in kind**: of 88 portrait entries, 79 are used (all through the tool-battle menu's portrait lookup) and **9 are referenced nowhere** (`chara_bigl020`, `040`, `043`, `047`, `056`, `059`, `065`, `067`, `069`; for example an elderly man in purple robes and a masked figure in blue armour). Which TCRF portraits these are is not matched one to one. | verified (unreferenced) |
| Marine / Land / Sky ribbon art | Not checked (the menu textures are unnamed sheets; needs browsing by eye). | not checked |

### Rooms

| TCRF claim | Our finding | Label |
|---|---|---|
| Three unused battlefields: Cipher Lab exterior, Mt. Battle lobby, Realgam Tower 2F | **Confirmed.** `D2_pc_1F_bf` (Mt. Battle lobby) and `D4_tower_2F_bf` (Realgam Tower 2F) are named by **no battle record**; `D1_out_bf` (outside the lab) is named by two battle records (459, 511), and **neither is ever started** (both are postgame trainers, below). **R** | verified (battle-arenas.md, trainer-encounters.md) |
| Unused met locations (PRE GYM, Phenac Stadium, Pyrite / Under / Mt. Battle / Orre Colosseum, Relic Cave, Front of Lab, Realgam Tower lobby and 2F) | The location names are all in the US text (**verified**); that no Shadow Pokémon uses them follows from which Shadow ids are used (48 of 96) and the unused battles (**likely**; the Shadow table's location field not checked per entry). | verified (names), likely (unused) |
| A cut dungeon "D3": no D3 maps; map numbers 96-101 are empty and crash if loaded | **Confirmed**: the room table jumps from room 0x5F (Mt. Battle's crater) to 0x66 (Realgam Tower) with **no records for 0x60-0x65**, and there is no `D3_` archive. The archive numbers themselves have no gap, so the cut happened before the archives were numbered (**likely**). | verified |

### Shadow Pokémon and the postgame

| TCRF claim | Our finding | Label |
|---|---|---|
| Extra Shadow Pokémon: a debug list shows the planned line-up; the final table has placeholder slots ("Shadow Extra"), 30 of them for e-Reader cards | **Confirmed in shape**: the Shadow table has **96 slots** (ids 1-0x60) and the story uses **48** (pokemon-data.md (not yet published)). The debug name list was not found in the US executable as Shift-JIS text (**not found**; it may be elsewhere or in the Japanese build). | verified (slots), not found (list) |
| The "snagged" flags are 1262-1357 and the "purified" flags 2126-2221 | **Differs: the other way round.** Each Shadow record's +0x14 flag (1262 + n) is set by the purification code (`fn_801EECD8`, called from `pokemon_relive.c`); its +0x12 flag (2126 + n) is set when the player obtains the Shadow Pokémon (`pokemonSetOnDarkPokemonFlag`) and is what the Repeat Ball reads. | verified |
| An unfinished postgame (the "Unfinished Postgame" sub-page) | **Strong data support.** Trainer records 447-471 and 778 are mostly **untranslated** in the US data (Japanese names such as Milky, Balargas, Pian, Myth), several still carry a **placeholder party** (Willie's two level-24 Zigzagoon: 457, 458, 470, 471), and their battles are **never started** by any script: about **60 battle records** (449-520), which is most of the 93 "story battles with no start call" in trainer-encounters.md. They include a **gang of six riders** with numbered names (Japanese for "one" to "six" in several languages) placed in five areas each, and **Myth (マボロシ, "phantom") placed in eight areas**, fitting TCRF's roaming "Phantom Wanderer". **R** (needs scripts) | verified (records, no calls), likely (that they are the postgame) |
| Rui's companion inventory: placeholder names "N/A 2" / "N/A" for the two characters, and Japanese lines for Rui's pocket | **Confirmed.** The two placeholder strings exist (in `common_rel`'s text bank 4), and Rui's pocket lines are in the US build **still in Japanese**, in the Battle Mode archive's copy of the pocket-menu text (full pocket, "leave it to me", "I already have that one", "received <item> from <Rui>"). The **pocket code exists too**: the bag's page 0 (10 slots, one of each item, give / take-back functions), skipped by the bag's page turning (bag-and-pc.md). **R** (needs translation and the pocket code) | verified |

### Animations (checked from the model slot records, 2026-10-17)
Each model has a **slot table** (17 slots) that maps a role (idle, attack, faint, ...) to one of the model's
animations (animation-catalog.md (not yet published)). TCRF counts a trainer's **animations** 0-9, not slots; in every trainer model the
slots map onto TCRF's numbers the same way: slot 0 = TCRF 1 (field idle), slot 1 = 2 (ball throw), slot 2 = 3 ("arm
held out"), slot 3 = 4 (battle idle), slot 4 = 5 ("fear"), slot 5 = 9 (intro), slot 8 = 7 (hit by a Shadow Pokémon),
slot 9 = 7 then 8 (hit, then guard), slot 10 = 6 (defeat); TCRF's 0 (T-pose) is in **no** slot (**verified**, the
slot tables).

| TCRF claim | Our finding | Label |
|---|---|---|
| Trainer animations 3 ("recall"?) and 5 ("fear") are never played; 7 (hit by a Hyper Mode Pokémon) never seen for trainers only fought in Battle Mode; 9 (intro) only in Colosseum intros | **Confirmed for generic trainers.** Animations 3 and 5 sit in slots 2 and 4 of all 74 trainer models, but every battle animation sequence names the slot it plays (the first field of each of the 1,098 sequence files), and **no sequence plays slot 2 or 4 except Rui's six Shadow-detection "look" sequences** (`hiduki_look_a`-`f`), which use her own model's slots 2-4. The slots that are played: 1 = the 13 ball throws, 5 = the boss intros, trainer entrances and the win pose, 7 = the 13 snag throws and the Shadow-detection look (only Wes and Rui fill it), 9 = damage, 10 = the 13 ball-return / defeat sequences. Caveat: code that switches a model's animation directly, without a sequence, was not scanned. Animation 0 (a short rest pose) is in no slot, matching TCRF's T-pose. **R** (TCRF suggests 3 as a recall and 5 as a flinch when the trainer's Pokémon is hit) | verified (sequence data), likely (no direct use in code) |
| (new) Wes's two extra animations | Wes (`ken_a1`) also fills slots 6 and 7 (animations 11 and 10: the Snag Ball throw and its follow-up, per TCRF), and Rui's model (`hizuki_a1`) fills slot 7 | verified |
| (new) Extra trainer animations | Five trainer models carry one animation that **no slot** plays: `boss777_a1` (1), `gym_c_a2` (3), `ojisan_a1` (7), `ojo_a1` (8), `thief_f_a2` (3). What they show needs eyes. **R** | verified (unreferenced) |
| Hypno's faint is not assigned (it shows its idle; Memento ends at once) | **Matches the data**: Hypno's faint slot (10) and the second part of slot 9 point to its **idle** animation (0), while a 119-frame animation (8) is played by no slot: the unused faint. Memento's battle side not checked. **R** | likely |
| Flying types' mid-air faint (Stadium leftovers, plus new Hoenn ones) | **Plausible**: the Pidgey line has two long animations no slot plays (102 and 59 frames), and Spearow, Farfetch'd, Doduo and Dodrio have similar ones; in all, 90 animations of 40+ frames across the Pokémon models are played by no slot (short unplayed ones, 2-16 frames, may be face animations played by other code). What each shows needs eyes. **R** | guess |
| The Japanese logo | **Not found** in the title archives (`logo_demo.dat` has only the English logo, its glow, sky and desert); the rest of the disc not searched. | not found (title archives) |
| (new) Developer leftover | The "dummy" entry in the movie archives (`pokemon_logo`, `opening_demo`, `auto_demo`, `staffroll`) is the developers' **compiled movie library** (an archive of object files with the original source path), giving the movie player's original function names. | verified |

### Regional differences
The intro, title screen, banner, Rui's clothes, logos, Somek, the Story Mode image and the credits length compare
the Japanese and international releases; this research covers the US version, so these can't be checked from its data
(regions.md). Not checked.

### Oddities

| TCRF claim | Our finding | Label |
|---|---|---|
| Frenzy Plant is named "SOLID PLANT" | **Confirmed** (the move's name in the US text). | verified |
| Teleport's battle animation can't be seen in battle (no wild battles; failed moves don't animate) but appears when Mt. Battle's Abra teleports the player | **Confirmed (likely).** Every Mt. Battle area has an Abra (`casey`), and its script calls `std244_spawnEffectAtPosition`, which plays the move effect whose id is **100 = Teleport's** (the call passes a small record {100, 3}; **verified**) at the player's position: the battle animation reused for the warp. That it is the visible effect when Abra teleports the player is **likely** (not watched). It fails in every trainer battle, as TCRF says. | verified / guess |
| Different Ho-Oh flights in the ending after snagging all / purifying all | **Confirmed, with the exact rule** (`D4_casino_colo`, the ending function): four Ho-Oh models and camera shots; **flight 1** by default; **flight 2** once story flag 1144 (0x478) is set, which the staff-roll code sets after the first ending (`movie.c`), so only on a **later** ending; **flight 3** if, in addition, all 48 Shadow Pokémon were **snagged** (counts flags 2126-2173); **flight 4** if also all 48 were **purified** (1262-1309). So the alternate flights can't appear on the first ending. **R** | verified |

## Found here, not on TCRF's page

| Item | Where | Label |
|---|---|---|
| The **Card e "Virtual Capsule" hall** is complete, translated and unreachable; it is the lab shown in the title's attract video | card-e.md (TCRF has a separate e-Reader sub-page, not checked) **R** | verified |
| The **in-engine opening**: 31 chained scenes in five Snagem hideout rooms that nothing starts | movies.md **R** | verified |
| Room 0x398 `nintendo_logo`: three empty functions, never entered | new-game-flow.md | verified |
| The **footstep mail** machinery is idle (all 32 mails have send rate 0) | field-steps.md | verified |
| The trainer AI's **combo step** is an empty stub (XD fills it) | battle-turn-flow.md | verified |
| The full debug menu (above) | `dbgMenu*` functions | verified |
| A **story-jump and flag-editor** tool: a list of story scenes that jumps the game to any point (setting every earlier story flag, item swaps, party, money and position), or, with a button held on the second pad slot, a flag editor | `fn_800093D0` -> `fn_8018FE30`; nothing references it | verified (code), likely (unreachable) |

## Our own scans (2026-10-17)
- **Untranslated text** (**verified** counts): **2,265 lines in 33 US text banks are still in Japanese**, **likely** a
  sign that the content was cut before the translation (**guess** as to why). The largest groups:
  - **common bank 6 (about 1,750)**: mostly the **debug menu's labels** (camera follow / rotate / move, colour bar,
    processing bar, message test, menu test, party change) and developer names (for example trainer-class labels);
  - **the P★DA (217)**: mail subjects and texts of an unfinished mail thread, including a **Minun hunt** ("found
    Minun?", "where did Minun go?", "being chased!"), matching TCRF's cut Shadow Minun;
  - **room banks (about 200)**, led by Pyrite Town (27), Pyrite Cave (14), outside the Shadow Lab (13), Mt. Battle's
    entrance (13), The Under's house A (13), Outskirt Stand (12) and Phenac City (11). Among them a **password
    quest**: people in Pyrite Town and outside the lab ask "is the password <Pokémon>?" (Exploud, Whismur, Loudred,
    Grimer, Muk) and refuse to battle otherwise; a trainer who "came to battle after reading the mail"; a mail from
    the Pyrite police chief about Miror B.'s henchmen being arrested; someone complaining that the lab is empty.
  TCRF's "Unfinished Postgame" sub-page (not read here) **likely** covers much of this.
- **Dialogue no script shows** (**likely**, a candidate list): of 2,575 lines in room text banks, **767** are never
  passed as a literal message id by a message call in any room script or the common script. Some are shown by code
  (the day-care lines in Agate are an example) or through computed ids, so this overcounts; the untranslated lines
  above are the clearly unused part.
- **Songs no script plays** (candidates): 48 of the 78 songs are never named by a room script; most of them are
  played by code (battle themes, fanfares, the title and level-up songs), so only some are unused. Worth checking
  first: **`virtual_song`** (*updated:* **verified** as played by code: after a Card e battle the music returns to
  `virtual_song` when the Card e menu was entered fresh, otherwise to the world-map song; so it is heard only on the
  Card e path, which US play can't reach: card-e.md) and the ten
  **`chakumelo00`-`09`** songs (着メロ, "ringtones"): no code reference and no id table found (**likely**
  unused, or chosen by a computed id).
- **Trainers in no battle record**: 301 of 820, but most (from 479 up) are **likely** Battle Mode opponents picked by
  code, so this is not an unused list (not pursued).

## Restoration candidates
In rough order of effort: Ho-Oh's alternate flights on the first ending (one script choice); the two unused
battlefields and the lab-front field (data); the debug menu's pause / slow tools (code toggle); Hypno's faint and
the mid-air faints (animation assignment); the Card e hall (needs an entrance and card input); the in-engine
opening (31 scenes exist); the postgame trainers and the password quest (records and Japanese text exist; need scripts, parties and
translation); Rui's item pocket (text exists in Japanese; the pocket code exists, bag-and-pc.md; needs page 0 enabled and translation).
