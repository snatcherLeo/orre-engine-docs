# RNG (random number generator)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-17). The game's one shared random number generator: its algorithm, its seeding and its
main consumers. A faithful port must reproduce the exact sequence and timing of RNG calls, because
RNG-manipulation tools (PokeFinder) depend on it.

## What it does

One random-number generator serves the whole game: battles (damage, critical hits, AI, catch shakes), Pokémon creation
(personality values, IVs), the field (NPCs, particles) and presentation (the battle camera, effects). It is seeded once
from the console clock at boot and never reseeded, so every draw anywhere shifts every later one.

## Data structures

- **The state:** one 32-bit word (`lbl_80478C90`), reached through a pointer (`lbl_80478C94`, HSD's seed pointer); the
  algorithm below.
- Nothing else: there are no separate streams or saved states.

## Algorithm (verified, code)

- **One shared 32-bit LCG**: `state = state * 0x343FD + 0x269EC3`; the output is the **high 16 bits**
  (`HSD_Rand`, 0x801ADCD8, in `src/hsd/hsd_pobj_range_801AA608.c`; it is HSD's `HSD_Rand`, the same code as Melee's
  `random.c`, so the generator belongs to the HSD library, not to the Orre Engine). The state lives behind the
  pointer `lbl_80478C94` (Melee: `HSD_RandSeedPtr`, which `_HSD_RandForgetMemory` resets to a static seed).
- Front ends, all advancing the same state once per call: `GSrand()` (0x800E0C54, u16),
  `HSD_Randf()` (0x801ADC7C, float = (state >> 16) / constant), `HSD_Randi(max)` (0x801ADC3C,
  `max * (state >> 16) / 0x10000`), `fn_800E0BA0` (sum of two `HSD_Randf` minus a constant = two
  advances) and `GSrandf` (0x800E0BE4, one `HSD_Randf`).
- **Seeding:** `fn_800E0C78` sets the state from `OSGetTime()` (the low 32 bits of the time base). **Verified in Dolphin
  2026-10-15** (with a scripting build of Dolphin): it runs about 1.0 s after boot (frame 119), and the seed
  follows the console clock: seed = low 32 bits of (seconds since 2000-01-01 x 40,500,000, the time-base rate, + about 40,381,700); with the
  clock at the GameCube epoch the seed is 0x02682D0F. The start-up does draw. Right after seeding, the game-loop thread's save-data setup
  (`savedataCreate` → `savedataInit` → `memoInit`) fills the 500-entry memo table with one 32-bit random value each,
  made from two draws (`_fadeEffectGetRandom`), so 1,000 draws happen before the first logo frame; about 22 more follow
  before the logo room starts (**verified** in the PC port (the decompiled code running natively) with a caller log; `memoInit` byte-matches the original, so the
  console does the same: **likely**, to confirm in Dolphin). So the generator's state at the logo is the seed advanced
  about 1,022 steps, not the seed itself. (**verified** in Dolphin) The burst runs at about frame
  448, and its exact size **depends on the seed** (1,018 for seed 0x02682D0F, 1,022 for 0xC1F12F2A) because some
  draws are repeated until a gender fits; two of the extra draws place the memory card's work area (memory-card.md).
  The seeding frame measures as 119-120 depending on how frames are counted. *For the port:* vanilla replays inject the
  captured seed; normal play seeds from the real clock.
  **Who seeds, and reseeding** (**verified** from the code): the seed is set
  **once**, during graphics start-up: `GSgfxInit` → `GSmathInit` → `fn_800E0C78`. **Nothing reseeds mid-game**: the
  only other code that writes the seed pointer is HSD's forget-memory hook (`_HSD_RandForgetMemory`), which runs only
  from `HSD_InitComponent` (also called once by `GSgfxInit` at boot) and only moves the **pointer** back to its static
  home (`lbl_80478C90`) if it pointed into freed memory, never the seed value. After boot the state changes only by
  being drawn from.

## Who calls it

| Caller | Evidence | Status |
|---|---|---|
| Battle code (damage rolls, accuracy, AI, ...): ~120 `GSrand()` call sites in `fight_*` files | call sites in the decompiled code | verified (call sites), per-use meaning open |
| Pokémon creation (personality, IVs): `pokemon_range_8011F5FC.c` (8 sites) | call sites in the decompiled code | verified (call sites) |
| Save data (`hero_savedata_exact_80129280.c`: the save's random id) | call sites in the decompiled code | verified |
| **Every save written to the card** : the save task's setup (`memcardSaveSetup`, `memcard.c`) fills a 20-byte padding field after the save block with `GSrand()` bytes, one draw per byte, so each save advances the shared sequence by 20 | the code (`memcardFillRandom`) | verified (code); that the padding is never read back is **likely** (nothing in the load path checks it) |
| Particle system (`ps_range_80168C64.c`: emission angles, spreads, lifetimes) via `HSD_Randf` | call sites in the decompiled code; in battle the largest share of draws (see "Measured in battle" below) | verified |
| HSD bytecode interpreter (`hsd_range_801920E4.c`: random ranges) via `HSD_Randi` | call sites in the decompiled code | verified |
| Menus, `kaisuu.c`, battle camera (`battleCameraStartRandom`) | call sites and names in the decompiled code | verified (sites) |
| **Battle models blinking** (`wazaSequenceOwnerUpdateBlink`, `gs_range_candidate_801DF1D0.c`) | the code plus a PC trace; console counts from Dolphin | verified (mechanism and counts) |

**Idle blinking draws every frame in battle** (**verified** from the code and a PC trace; console
counts from Dolphin). Each battle model runs a blink check every game frame. When the model is not playing a move
sequence, is not already blinking (byte +0x19 = 0) and its blink flag is not off (+0x18 bit 8), a timer at +0x20
counts up by `GSgetFrameStep()` (the screen refreshes the frame took). From 10 on, it draws **one `GSrandf` per game
frame** and starts the blink texture animation if the draw is below a threshold: t(2 - t) / 60 with
t = (timer - 10) / 50 while the timer is under 60, then 1/60, and 1.0 from 180, so a blink is certain by 180.
- **While the command menu waits**, these checks are the only per-frame draws: **3 per game frame** at Willie's first
  menu on the console, rising to **5**.
- **One traced wait, draw by draw** (PC trace, 729 draws from the seed to the end of the first turn; **verified**,
  decoded by function): for the first ~175 frames every draw is a blink check (703 in the window); then **one**
  idle-camera draw (`battleCameraStartRandom`, the skip roll, matching its ~200-refresh timer). Once the commands are
  confirmed, the turn's own draws follow in this order: the opponent AI's move and action choice
  (`fightTrainerAiSelectWaza` 2, `fightTrainerAiSelectFightAction` 1), the speed comparison and its tie-break
  (`fightFloorCmpfightOutPokemonNimbleness` 2), the accuracy check (`fightCheckAccuracy` 1), the move camera's shot
  (motion, timing pattern, position), the critical-hit check (`WS_CRITICAL_CHECK` 1), and another camera position.
  Dolphin's per-frame counts agree on the blink pattern.
- **What it means:** idle blinking feeds the shared generator, so how long a player waits in a menu, and how many
  models are waiting to blink, changes the later damage rolls and AI choices on the console (**likely**: the mechanism
  is verified; the gameplay effect follows from the shared generator).
- **For a port:** the blink timer counts refreshes, so the number of draws per second follows the retrace rule
  (gfx.md): it must stay at the console's rate under frame interpolation and speed settings for replays to match.
- **The order of draws inside a frame matters, not only their number** (**verified** on a
  recorded battle): two runs can draw the same numbers in the same overall sequence and still start different blinks,
  if one draw lands with another caller inside a frame. The generator's state at the end of every frame is then the
  same while each caller got a different value; one such swap started a blink a frame early in the PC port and shifted every
  later blink. In 30 fps battle scenes the console finishes a frame's particle draws in the first refresh and makes the
  blink checks later in the frame (its mid-frame state sits one or two draws behind the frame's end), so where its frame
  logic falls against the refreshes is part of what decides the order (**likely**).
  
- **The same numbers one frame later make the AI choose differently** (**verified**: giving the
  PC the console's state two draws later made it choose exactly what the console chose): a trainer with random target and move picks (AI record flags +0x11 / +0x12,
  battle-turn-flow.md section 7) spends its seven draws in the frame after the command phase ends. In a recorded
  street battle the console and the PC drew the same sequence, with the same lists of targets and moves (checked
  against the disc data), but the PC made its choice in the frame that holds a blink pair and the console one frame
  later, after the next pair. Two draws earlier was enough to change both Pokémon's target and move. So a frame of
  timing before a turn starts decides what the opponent does, not only how much damage it deals.
- **A move's draws, in order** (**verified**: console draw logs with their callers named, and two
  PC traces): the accuracy check (`fightCheckAccuracy`, a 1-100 roll), then, after the move's camera shot has started,
  the critical-hit check (`WS_CRITICAL_CHECK`: the draw modulo the stage's divisor, a critical hit on 0; **likely** 16 at
  stage 0, as in Generation 3: the table itself not read) and the damage spread (`fightApplyDamageVariance`: 100 minus the draw modulo 16, so 85-100%) in the same frame,
  then the side-effect chance (`wsCmd15`, the draw modulo 100 against the move's chance) when the hit lands. The end of
  every turn the battle goes on adds one more (`fightSeqTurnEnd`: the turn random, battle-turn-flow.md).
- **A model animation's particle draws on whichever thread animates the model** (**verified** on
  a recorded battle): particles a model's animation spawns (`HSD_JObjAnimCallback` -> `gsParticlePtclHook` ->
  `gsParticleCreate`) take their numbers from the same generator at the moment the model's animation is updated. Once
  in a long recorded play session this happened on the battle logic's own thread, between two of a turn's checks, so every later
  roll in the battle took the next number along: the same battle without that one spawn would have had a critical
  hit two turns later. Gameplay can therefore depend on exactly when an animation step runs relative to the battle
  logic (**likely** for play; the mechanism is verified).
  
  

### Which thread draws what in a battle (**verified** from a draw-by-draw console capture)

The game runs its logic as cooperative threads (the GS scheduler switches between them inside one game frame), and
all of them share the one generator:
- **The battle thread** (`fightMainThread`, created when the battle starts): every battle-logic roll (accuracy,
  critical hit, damage variance, the trainer AI's picks, the end-of-turn checks) **and the camera of every move**
  (`battleCameraStartWaza`: motion, position, pattern and zoom), since move sequences start from the move script.
  In a double cup battle about 830 draws over the whole battle.
- **The main logic thread**: the idle blink checks of every model (most of a battle's draws), particles, and the
  **idle camera** during command selection (`battleCameraStartRandom`: one 50/50 roll each time its timer runs out,
  and a whole procedural shot when the roll says go).
- **Outside any thread** (the render pass): particle generators.

A thread's run can go on across a frame boundary: the trainer AI's picks for two Pokémon were one unbroken run of 7
draws spread over two console frames. So the order of draws inside one thread is fixed by the game's logic, but how
the threads interleave depends on timing, which is why a frame of difference moves the rolls. Couplings between the
threads go through shared variables, not the generator alone: the idle camera's last motion decides which motion
the next move camera may take (battle-camera.md). 

## Per-frame advances ("noisy" vs "quiet" areas)

Because particles draw from the **same** generator, any room with active particle effects advances
the RNG every frame, by an amount that depends on how many particles spawn: such areas are
"noisy", rooms without per-frame random users are "quiet" (measured per room below). NPC wandering and animated
models may also call it (open).

**The title, the main menu and the Options screen are quiet** (**verified** in the PC port: every draw
counted per room in 60-frame windows over a scripted run, with pictures of the main menu and the Options screen to
confirm where it was): **no draws at all** while waiting on the title (about 110 seconds, its sparkles included), on the
main menu (about 70 seconds) or on the Options screen, nor when moving the cursor or changing Rumble there. The only
draws before the story are the start-up burst (the empty save made at boot: 1,018 draws for that run's seed) and, when
the title hands over to the main menu with no save on the card, the same burst again; loading an existing save draws
none. So how long a player waits on these screens doesn't change their luck; what they choose there (New Game, Continue,
a save made or loaded) does. 

**Measured in battle** (**verified** in the PC port with a caller trace of the generator, and consistent
with a per-frame count in Dolphin): in Willie's battle, between the first command menu and Espeon's first hit, the game
drew **1,844** numbers in the PC port (**1,871** in Dolphin). Most came from visual effects: the one-advance float helper
`GSrandf` **994**, the particle generator's spawner `psExecGenerator` **464**, the particle byte-code interpreter
`psInterpretParticle0` (several sites) **about 220**, and the particle spawn `generateParticle_8017424C` **about 100**.
So **damage rolls, critical hits and the AI's choices depend on how many particles the battle intro and the move
animations spawned before them** (**likely**): visual timing feeds the battle's outcome. This also explains a PC /
Dolphin difference: with the same seed and the same button presses, the PC's first hit still differs from Dolphin's,
and the PC drew 27 fewer numbers before it, almost all of them visual-effect draws (**likely** the cause).
`GSrandf` is a general random-fraction helper that the **battle camera** uses
too (the idle camera and every procedural shot draw through it), so part of its 994 is camera draws, not particles;
the split needs a trace that records the helper's own caller ([battle-camera.md](../systems/battle-camera.md)).

**A port's battle drift: one candidate ruled out** (now the main suspect is **timing**: the PC's near-instant disc reads let the send-out stop
waiting for its portraits sooner (battle-turn-flow.md, the Pokémon entrance), so the battle's per-frame draws start
on different frames). The particle generator decides how many
particles to spawn each frame by adding `random × fraction` to a counter and spawning whenever it passes 1.0. On the
console that addition is **one fused multiply-add** (`fmadds`, **verified** in the assembly), rounded once; a separate
multiply and add rounds twice and could in principle move a spawn by a frame. **Making the PC's version fused did not
change the drift** (**verified**: the same random state at the diner entry and the same battle pattern). So the cause
is **still open**: the move effect starts earlier in the PC port and spawns more particles. The rounding difference stays a
real accuracy item for any reimplementation.

**Measured outside battle** (per-frame counts in Dolphin): the **name-entry screen draws every
frame** until about 75 frames after the name is confirmed (its animated background), then a **burst at New Game**
(the Strategy Memo's 1,000 draws, plus about 16-20 more). The **starters are created before the opening movie**; their
stats match on both sides when the generator is reseeded in the quiet window right after the name confirmation.
(**verified**): The name screen draws 1-5 numbers a frame; after the confirmation come about 30
quiet frames, then the New Game burst (`savedataCreate`): the Memo's 1,000, the trainer id (2), 7 per starter and 2
more per extra attempt when a candidate is rejected. A reseed at the opening movie's start doesn't change the
starters, which confirms they're made in that burst. A personality value is built as `(GSrand & 0xFFFF) |
(GSrand << 16)`, and the console's compiler evaluates the right operand first, so the **first draw is the high half**.

**Measured in the field, per room** (**verified** counts in Dolphin: 300 frames standing still, then
300 frames walking, after arriving in the room; a cause is given only where it is **likely**):

| Room | Standing (300 frames) | Walking (300 frames) | Pattern |
|---|---|---|---|
| Outskirt Stand, outside, on arrival | 422 | 114 | half the frames draw none, the rest mostly 2-3; walking draws fewer |
| Phenac City, outside, on arrival | 16,585 | 15,015 | about **55 per frame** on average, in bursts of 116 / 76 / 246 draws on about half the frames and none on the rest: **likely** a particle effect drawing in bursts |
| Phenac City, inside the mart | 0 | 6 | quiet: three frames with 2 draws |

**Who draws in Phenac** (**verified**: a per-call trace on the PC port during Phenac's arrival
scene, 193,285 draws, resolved to source lines): **about 99% are the particle system**. The non-particle draws are a
few hundred at most.
- **Most come from one emitter shape:** about 147,000 draws from the particle generator's **ring / disc emitters**
  (generator kinds 0, 3 and 4; "ring / disc" is **likely**: kinds 3 and 4 spread particles evenly over a disc's area). Each new particle gets **one random angle** between the generator's minimum and
  maximum, so it leaves in a random direction around the axis. The trace shows no radius draw, so these generators use
  a **fixed radius** (a negative radius in the data means "always this distance": **verified** from the code).
- **About 11,600 particles come from box emitters** (generator kind 5). Each one draws **three numbers in a row, a
  random x, y and z** inside the box (**verified** from the code). The particle-script interpreter (`psInterpretParticle0`)
  draws the same number of times, so **likely** one random script step per box particle.
- **Which effect it is** isn't in the trace.
- This trace was the arrival scene (Folly and Trudly carrying the sack), with different story flags from the Dolphin
  measurement above, and it stopped at the tracer's cap. So its totals don't compare with the 55-per-frame figure.

So **the field's draw rate depends on the room's effects**: none indoors, tens per frame in Phenac. Anything decided at
random in the field (NPC wandering, a gift's personality value) and any port difference in particle timing shifts every
later roll.

**For a port and for RNG tools:** a faithful port must spawn the same particles in the same order, frame by frame; any
change to visual effects changes battle outcomes unless effects get
their own generator.

## Shiny checks and the Shadow Pokémon quirk (2026-10-14)

**Background for a shiny-Shadow-Pokémon fix.** A Pokémon is
shiny when `(PID >> 16) ^ (PID & 0xFFFF) ^ (OT >> 16) ^ (OT & 0xFFFF) < 8`, where PID is the
personality value and OT the original trainer's 32-bit ID (secret ID in the high half). It is the
Gen 3 rule.

**Fields** (verified, `pokemonSetStatus` / `pokemonGetStatus` switches in `pokemon_range_8011F5FC.c`):
status `0x6F` = personality value (`pokemonBiosSetRnd` / `GetRnd`), status `0x75` = original
trainer's ID (`pokemonBiosSetCatchTrainerRnd` / `GetCatchTrainerRnd`). The trainer's own ID is
`heroGetStatus(hero, 2, 0)` (`heroBiosGetRnd`).

**Where Colosseum decides and checks shininess** (verified from the C unless marked):

| Step | Where | What it does |
|---|---|---|
| The shiny test | `pokemonCheckRare` (0x80125390, `pokemon_range_exact_801248C4.c`) | the rule above, on the Pokémon's own 0x6F and 0x75. Every display path calls it: portraits and menus (`gs_range_80109C88.c`), the battle model choice. |
| Creating a PID with constraints | `pokemonCreateRndFit(pkm, gender, nature, shinyMode, otId)` (0x80124410, `pokemon_range_8011F5FC.c`) | loop: draw a PID, reject it if gender or nature (`PID % 25`) don't fit, then the shiny test `pokemonCreateRndFitSeedHit` against `otId`: `shinyMode` 1 = must be shiny, **0 = must not be shiny**, -1 = don't care. |
| **Opponents' Pokémon, incl. Shadow Pokémon** | `fightTrainerCreateFightTrainerPokemonDataIdToPokemon` (`fight_trainer_range_801F7F80.c`), called from the trainer setup above it with `heroGetStatus(enemyTrainer, 2, 0)` | calls `pokemonCreateRndFit(..., shinyMode 0, enemy trainer's ID)`: the PID is **rerolled until it is not shiny for the enemy's ID**. Then the Pokémon is added to the enemy's party with `heroCatchPokemon`, which writes the enemy's ID as its OT. |
| Shadow Pokémon met again | same function, Shadow branch | if the Shadow entry's story flag is set (`darkPokemonCheckFlag16` (0x801EE8F4) -> `std081_flagCheckTrue`), the PID saved at the first meeting is reused (`darkPokemonSaveGet8` (0x801EE750), Shadow runtime slot +8); otherwise a new PID is made as above and saved there (`darkPokemonSaveSet8` (0x801EE7BC)). So the PID, and so the shiny outcome for the player, is fixed at the first encounter. |
| **Shininess shown in battle** | battle Pokémon setup (`fight_out_pokemon_suffix_80207C6C.c`, near the `fightOutPokemonGetRndStatus` call) | copies the battle Pokémon's PID and OT (the enemy's ID) into a temporary, calls `pokemonCheckRare` and passes the result to `wazaSequencePokemonCreate` (model choice, shiny recolour). Because of the reroll above this is always "not shiny" for an opponent's Pokémon. |
| **Re-check after the snag** | `heroCatchPokemon` (`hero_savedata_exact_80129280.c`) | copies the Pokémon, then `pokemonSetCatchStatus(..., heroGetStatus(player, 2, 0), playerName)` writes the **player's** ID as the OT; the PID is kept. From then on `pokemonCheckRare` uses the player's ID, so the Pokémon is shiny with the usual 1-in-8192 chance (independent of what battle showed). |

So Colosseum doesn't recalculate the PID after the snag. It only changes the OT, and the same
shiny test gives a new answer. The quirk: battle always shows "not shiny" (the reroll against the
enemy's ID), while the snagged Pokémon can be shiny.

**RNG cost (verified, code).** Each PID candidate is two `GSrand()` calls
(`pokemonCreateRndFitRand32`: high half, then low half). Every rejected candidate (wrong gender,
wrong nature, or shiny for the enemy's ID) costs two more calls. The anti-shiny reroll rejects a
candidate about 1 time in 8192, so it rarely adds calls, but vanilla mode must keep it to keep the
exact sequence. Rematches with a Shadow Pokémon whose PID was saved draw no PID at all.

**The mod (Colosseum), sketch:** create Shadow Pokémon with `shinyMode` -1 (or keep 0 but pass the
player's ID), and make the battle-display check use the player's ID for Shadow Pokémon, so battle
and party agree. Either change alters the RNG calls (the rejection pattern differs), so mod only.

**XD:** not checked in XD's code yet (xd-insights.md).

## Flow

Boot seeds the state once (above). From then on every consumer advances it when it runs: per frame (particles, the
field's NPCs, the idle battle camera), per event (a battle's damage and AI, a Pokémon's creation) and per menu action
(the name screen, summaries). The order of those calls within a frame is the order the game runs its systems, which
is why visual timing changes battle results ("Who calls it" and "Per-frame advances" above).

## Disc data

None: the generator and its seeding are code; nothing about it is stored on the disc or in the save.

## Changing it (port notes)

- Implement the exact LCG and call it from exactly the same places, in the same order, at the
  original 60 Hz logic rate; interpolated rendering must not run particle or game logic extra times.
- Keep the shared-generator behaviour (particles and battles share one state) in vanilla mode.
- Customised visuals (effects, cameras, weather) should draw from a separate generator, so they never move the
  game's draws.

### Mod surface
- **Data it reads:** none from the disc; the seed comes from the console clock at start-up.
- **Hooks it would need:** a seed override (set or view the seed, for replays and testing); a draw log (every caller
  and value); a **separate generator for cosmetic additions**, so mod visuals never take a draw from the game's own.
- **Fixed-size limits:** one 32-bit state; each draw yields 16 bits.
- **Must stay vanilla in GameCube-accurate mode:** the exact algorithm, every call site and its order, the shared
  generator for visuals and gameplay, the clock seeding, and one logic step per screen refresh at 60 per second.

## Efficiency

One multiply and one add per draw; the cost is negligible even at the thousands of draws per second that busy rooms
make.

## In later Genius Sonority games 

**Pokémon Battle Revolution** (Wii, 2007) keeps the same generator, as its public decompilation (pbr-ret/pbr-decomp,
`src/gs/math/GSrand.cpp`) shows: `GSmath::initRand` seeds it from `OSGetTime()`, the seed starts at 1, and the LCG is
inlined across the code; that project's own comment notes it is equivalent to sysdolphin's `HSD_Rand` and asks whether
that's a coincidence. Colosseum answers it: here the generator **is** HSD's `HSD_Rand` (**verified**, above), and the
game's own `GSrand` front ends sit on top of it. PBR no longer links the HSD library (it uses Nintendo's Wii
libraries), so it most likely carried the same LCG over into the GS engine itself (**likely**). XD, between the two,
almost certainly uses the HSD version like Colosseum (**likely**; to check at the XD kickoff).

## Open questions

- The per-frame RNG consumers in the overworld (particles per room, NPCs, cameras): three rooms measured (above: none indoors, about 55 per frame in Phenac); Phenac's draws are almost all
  particle emitters (ring / disc and box shapes). Still open: which effect in Phenac, and a caller breakdown for the other rooms.
- From the shiny-Shadow-Pokémon section:
  The "PID already made" flag is the Shadow record's +0x16 (`darkPokemonCheckFlag16` (0x801EE8F4) reads it): story flag **1357 + n** for Shadow
  id n (1358-1453, one per Shadow slot, **verified** from the code and the US Shadow table). Still open: XD's
  counterpart.
