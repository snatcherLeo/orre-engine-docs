# Screen and model effects (GSeffect and the WZX effect types)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely**: the same engine layer; not checked against XD)

Status: **partial** (2026-10-16). How the game runs the visual effects a move asks for (lightning, glow, colour fades,
after-images, sea and surf, distortion, ...). Read from `gs_effect.c`, `effect_visual.c`, `tracefx.c` and the move
code that starts them; checked by running every move file in the PC port (the decompiled code running natively) (see "Verified").

## What it does

Some move-effect entries (WZX entry kind 4, "screen effect") are not particles or models but small programs: an effect
with its own state block and up to five callbacks. A general **effect system** (`gs_effect.c`) owns a fixed number of
effect slots, starts and stops effects, and calls every running effect's update and render once per frame. Thirteen
effect types exist; the WZX entry says which one and carries its parameters.

## Data structures

**Effect slot** (one per running effect; 32 slots, set up at boot with `GSEffectInit(0x20)`, `GSeffectInit` (0x80130CE0)): id,
state (uninitialised / active / stopping), links in a free list and an active list, the effect's user data (its state
block, allocated per effect from the game's memory system with the size the type asks for), and the callbacks:

| Callback | When it runs |
|---|---|
| init | when the slot is registered (some types) |
| stop | when the effect ends or is stopped |
| start | when the move entry starts it |
| (reset) | second stop-type callback, cleanup |
| update | every frame while active; returning 0 ends the effect |
| render | every frame while active, after all updates |

**Parsed parameters** (0xD8 bytes, built per entry when the move file loads, `fn_801364A8`): +0 effect type, +4
duration in frames (scaled to the frame rate), +8 the type's own block. In the file the parameters are big-endian
words; some types end with an embedded file (a texture or a model archive) aligned to 32 bytes.

**The 13 types** (number = WZX effect type; names are the decomp's names, likely right; state size = the per-effect
block; callback roles follow the function-pointer types in the source, likely):

| Type | Name in the source | State | Callbacks | What it does (likely, from code and frame captures) |
|---|---|---|---|---|
| 0 | filter (`filterStart`) | 0x5C | stop, start, reset, step, update | colour fade over time on models: the attacker, or the **battle room's models** (darkens or tints the arena); colour keys interpolated |
| 1 | bolt (`boltEffectCreate` (0x80139820)) | 0x60 | stop, start, reset, update, render | Thunder's lightning from above: a branching bolt split again and again and drawn as strips (`_boltSubdivide`, `_boltDrawStrip`; **verified**, "The bolt" below) |
| 2 | (`electronEffectCreate` (0x8013A42C)) | 0x24 | start, update, render | arcs between two joints of the owner's model: a curve through four control points with random offsets, redrawn with jitter and a per-frame chance (electric crackle over the body; likely). Its render (`_electronRender` (0x8013A520)) builds curves between model parts |
| 3 | lightning (`lightningEffectCreate` (0x801380D4)) | 0x70 | stop, start, reset, update, render | not checked; no move on the disc uses it |
| 4 | trace (`tracefx`) | 0x2C | own callbacks | trails (ribbons following a joint) |
| 5 | leaf | 0x4C | stop, start, reset, update | leaves (an embedded model archive) |
| 6 | sea | 0xAC | stop, start, reset, update | water surface |
| 7 | surf | 0xD0 | stop, start, reset, update | wave model |
| 8 | env map | 0x18 | init, stop, start, reset, update | environment-map shine on the Pokemon |
| 9 | blur | 0x20 | init, stop, start, reset, update, render | two modes (a flag in its block): **after-images**, redrawing the Pokemon at earlier animation frames with falling alpha (Quick Attack and similar); or **screen blur**, capturing the last frame (GSgfx's back-framebuffer capture) and drawing it over the screen with fading alpha (`_blurDrawCapture` (0x8013E258)) |
| 10 | aura | 0x34 | init, stop, start, update, render | glow around the Pokemon |
| 11 | billboard (`patchiru`) | 0x24 | init, stop, start, reset, update | a model scaled each frame to fill the camera's view (Sandstorm, Sunny Day, the battle-name banners; "The billboard in detail" below) ("patchiru" is Spinda's Japanese name; Spinda's spots are painted elsewhere, by `pachiruEffectSetupModel` (verified, models-animation.md (not yet published), "Spinda's spots")) |
| 12 | distortion | 0xB4 | init, stop, start, update, render | screen distortion (heat haze, ripples) |

**Which function each type starts** (**verified**: the move-file loader `_wazaSequenceEffectEntryLoad` switches on the
type and calls one create function per type, in the order of the table above: filter, bolt, electron, lightning,
trace, leaf, sea, surf, env map, blur, aura, billboard, distortion). Type 1's create function prints "Could not start
electron effect" when it fails. That message was copied from the electron effect's code in the game itself, which is
how type 1 came to be called "electron".

**The after-images in detail** (type 9, mode 0; **verified** from the code; a rebuild of Quick
Attack matches the game): the effect's keys (*from, to, ticks*) advance on the effect's own clock, 2 ticks per frame.
The current value *v* gives *round(30 × v)* copies of the Pokémon at earlier animation frames, half a frame apart; the
*j*-th of *n* copies is drawn at alpha *0.5 × (j / n)²*, so older copies fade, and all are drawn before the Pokémon
itself.

**Moves seen using type 9** (verified: these move files ran the two blur functions in the PC sweep before they were
fixed): after-images in Quick Attack, Vice Grip (and its Pinsir variant), Mega Kick, Skull Bash and Extreme Speed
(attack halves); screen blur in Sunny Day, Double Team, Agility, and the damage halves of Mega Kick, Mega Punch and
Dragon Rage. Type 0's room tint: Thunderbolt (seen in the PC port). Type 5 (leaves): Petal Dance; its attack half alone
can use up all 64 model slots with leaf copies, so some leaves fail to appear (seen in the PC port with the game's own
pool size; likely the same in the original).

**Which types the moves use** (**verified**: every screen-effect entry of all 1,098 move files on
the disc counted with a parser of the move-file format):

| Type | Entries | Files | Examples |
|---|---|---|---|
| 0 colour fade | 955 | 855 | nearly every move (its flash or room tint) |
| 1 bolt | 2 | 1 | Thunder (damage half) |
| 5 leaves | 14 | 8 | Petal Dance, Feather Dance, Magical Leaf, Leaf Blade, Razor Leaf |
| 8 env-map shine | 15 | 15 | Harden, Iron Tail, Transform, Camouflage, the frozen status loop |
| 9 blur | 13 | 13 | Quick Attack, Sunny Day, Double Team, Agility, Vice Grip, Mega Kick, Mach Punch |
| 11 billboard | 30 | 30 | the battle-name banners of the Colosseum battles (one file per battle and per scenario); also Sandstorm and Sunny Day, in other effect files (below) |
| 12 distortion | 39 | 39 | Nightmare, Sweet Scent, Thunder Wave, Imprison, Transform |

(**verified** from the code and the effect files): The billboard (type 11) is used by more than
the battle-name banners. **All three Sandstorm effect files** start one, carrying a 3D model archive of about 527 KB (its geometry is only two
20 × 20 cards, below), and
**Sunny Day** uses the same effect. The stadium intro and name titles use it too. Each frame `_billboardUpdate` places
the model just in front of the camera, sized to the camera's own aspect ratio, so on the GameCube's 4:3 picture it
fills the screen; at a wider aspect it covers only the 4:3 middle unless widened. `_billboardUpdate` draws no random
numbers (verified). So the sandstorm is a screen-filling model, not particles; rain and hail are particles placed in
the world.

**The billboard in detail** (**verified** from the code; a rebuild matches the game's Sandstorm):
- **Its settings** (the descriptor, from +0x0C): the model archive's size at +0x00 (0 = no model), the animation at
  +0x08, a particle sub-id at +0x0C, and the animation kind at +0x10 (0 or 1; likely once or looping). The archive
  sits at the next 32-byte boundary after +0x18.
- **Start** (`billboardEffectStart`): the model goes into the **overlay layer** ([gfx.md](gfx.md), "The two layers and
  the depth reset"), and its animation and texture animation start from frame 0 at half speed.
- **Every frame** (`_billboardUpdate`): the model is placed just past the camera's near plane (at 1.1 × the near
  distance, along the view direction) and scaled to exactly fill the view at that distance: width 2 × *d* ×
  tan(fov / 2) × aspect ÷ the model's bounding width, height 2 × *d* × tan(fov / 2) ÷ its bounding height. The effect
  runs as long as its animation does.
- **Sandstorm's model** is two 20 × 20 camera-facing cards.
- Likely, from the game's pictures: the attack half's billboard ends when the damage half starts. The game's Sandstorm
  is clear at the cut to the target, then the damage half's own haze fades in.

**What a colour fade tints** (type 0; **verified** from `_wazaSequenceEffectEntryStart` and
`_filterApplyColor`): target 1, "the stage", is **only the room's own models**: the models listed in the room's scene
data, taken from the current room's resource. A move's own models (Surf's water, for example) and the Pokémon are never
tinted by it. Target 2 is the Pokémon playing the half; when its shiny colour swap is on, the fade keeps that swap as
its base. Each listed model gets the fade's colour as its modulation colour.
**How that colour is applied** (**verified** from the material code): the model's modulation stage
**doubles the colour but not the transparency**. So the neutral colour is 0x80 (and the default modulation is 7F, 7F,
7F, FF), while transparency is used as it is: 0xFF fully solid, 0x00 invisible, a value of *n* meaning *n* / 255.
Fades that combine two transparencies multiply them (*a* × *b* / 255). With this, Camouflage's fade (FF to 00: the
Pokémon see-through from the start) and Teleport's (80 to 00) match the game. Still open on Camouflage: in the game
the Pokémon takes on the room's grey, likely from its env-map shine reflecting the room.

**The env-map shine (type 8) in detail** (**verified** from the code unless marked; a rebuild of
Iron Tail and Steel Wing matches the game side by side):
- **What it is:** a real-time reflection. Every frame the game draws the battle scene, minus the shining Pokémon, from
  the battle camera into one screen-sized picture (`envMapEffectInit` makes it once and shares it). It then maps that
  picture onto the Pokémon's surfaces as an extra layer drawn over its lit colour, so the Pokémon reflects the room.
- **Settings:** a mode (how the picture is mapped: 0 reflection, 1 gradient, other values backlight), a type (0
  multiply, otherwise blend), and the strength over time: either a list of keys (*from, to, frames*), or one value
  that rises over the first third of the effect, holds, then drops to nothing for the last third. The strength is how
  much of the reflection shows. Past the last key the effect stops and the layer is removed.
- **What it reflects:** every visible model except the shining Pokémon itself; no particles.
- **Who uses it** (**verified** from the disc's 15 effect files with a type-8 entry): in the
  attack halves, Steel Wing, Iron Tail, Metal Claw, Mirror Coat and Magic Coat (all mode 0, reflection, blended);
  in the self-aimed moves' third part (the `_special` files), **Harden**, Iron Defense, Transform, Camouflage,
  Conversion and Conversion 2; plus Iron Defense's Torkoal version, two Iron Tail variants, and the frozen status loop.
  So Harden does shine: it lives in its third part, which a scan of only the attack and damage halves misses (a
  rebuild of it matches the game to the end; its shine keys run 195 ticks, past the animation, added 2026-11-20). It
  costs a second drawing of the scene every frame, which is likely why so few effects use it.

**The screen warp (type 12) in detail** (**verified** from the code unless marked:
`_distortionUpdate`, `_distortionEffectUpdateMatrices`, `_distortionRender`; a rebuild matches the game's Confusion and
Psychic side by side):
- **What it is:** a ripple in the picture. The game draws the scene again into a screen-sized picture, then draws one
  square facing the camera over the Pokémon, showing that picture shifted by a **warp picture** (128 × 128, one
  brightness value per pixel, dark at its edges), using the GameCube's indirect texturing. Where the warp picture is
  bright the scene is pushed further, so a hump with ripples appears; its edges are dark, so no seam shows.
- **Settings:** whether particles are drawn into the captured scene too; whether it follows the Pokémon or stays put;
  its size and strength; the warp picture's size; and when it peaks (mode 1: at the end; mode 2: at a set point).
- **Over time:** it starts at the Pokémon's centre, size 0, grows to full size at the peak and shrinks after it (a peak
  of 0 shrinks from the start); its strength fades evenly to nothing by the end.
- **Drawing:** no depth test, no culling, no blending; the square simply replaces what's under it with the shifted scene.
- Likely, from the game's pictures: the shift is *strength × brightness* pixels on both axes.
- **Moves:** in the attack and damage halves, Confusion, Psychic, Thunder Wave, Night Shade, Future Sight, Disable,
  Hypnosis, Psywave, Nightmare, Curse, Spite, Sweet Scent, Imprison, Metal Sound, Tickle and Extrasensory; in the
  self-aimed third parts, Meditate, Transform, Acid Armor, Rest, Conversion and Conversion 2 (39 effect files in all).

**The leaves (type 5) in detail** (**verified** from the code: `leaffxStartEffect`,
`_leaffxGenerateLeafData`, `_leaffxUpdate`, `_leaffxMoveLeaf`; a rebuild matches the game's Razor Leaf and Petal Dance
side by side):
- **Its settings:** ten numbers (a leaf's size, base and range; how far each flight goes and how far its bend reaches,
  base and range each; a lift added to each new end point's height, negative on the disc so the leaves sink; the
  speed; a factor for the first flight, base and range), then the number of leaves (at most 32), the tick when they
  start to fade, the attach point, and the leaf model.
- **Where they start:** at an attach point of the Pokémon playing that half (its slot's attach table), read once at the
  start; their size follows that Pokémon's effect scale.
- **One leaf model, copied** once per leaf. Each copy gets its own size (base plus range × a random offset) and flies
  along smooth curves (cubic Bézier curves), turning gradually between two random orientations along each one. The
  first flight always heads upward.
- **Each next flight** starts where the last one ended and leaves in the same direction, so the path never kinks; its
  end is a random direction at the set distance, plus the lift.
- **Speed and fade:** before the fade tick they fly faster (up to twice as fast at the start, slowing evenly); after it,
  every leaf fades together to nothing by the effect's end (one shared transparency on the model). A leaf that drops
  below the floor (height 0) disappears.
- **Moves:** Razor Leaf, Magical Leaf and Leaf Blade (damage half), Petal Dance (both halves, two sets each), Feather
  Dance (both halves): 7 of the 8 files the count above found; the eighth isn't identified yet.
- **The curve, exactly** (**verified** from `GSbezierCalculateVector` and `_leaffxUpdate`): each
  leaf keeps four points, the start (+0x00), the end (+0x0C), and two bend points (+0x18 near the start, +0x24 near
  the end), and a place on the curve *t* (+0x54). Its position is the standard cubic Bézier: start × (1−*t*)³ +
  3*t*(1−*t*)² × bend 1 + 3*t*²(1−*t*) × bend 2 + *t*³ × end, with *t* clamped to 0..1. Its orientation is a slerp
  (smooth turn) between two quaternions (+0x30 to +0x40) at the same *t*, then scaled by the leaf's size and moved to
  the position. When *t* passes 1, the next flight is set up and *t* drops by 1.
- **How many leaves really show** (**likely** the same on the console): every leaf is a copy of
  the leaf model, and the game holds at most 64 models at once. Feather Dance's target half starts three sets of 32,
  so with the two Pokémon and the room's models loaded the second set gets about 22 copies and the third none (the
  game logs "Could not create leaf clone model" for each one it can't make). The effect simply flies the copies it got.
  Petal Dance's attack half hits the same limit.
- **The leaves move the battle's random rolls** (**verified**: a console recording of Razor
  Leaf in Dolphin and the corrected PC build match over the whole turn, 17,382 draws and all 127 flights): each new
  flight a leaf sets up draws **13 numbers from the game's one shared random sequence** (rng.md), and a leaf that
  drops below the floor is freed and draws no more. So how many flights the leaves fly, which depends on the exact
  curve, decides where the next damage roll lands: the leaves are not look only, and anything that changes their
  paths (a different curve, a mod's new settings, a rounding difference) changes later rolls.
- Likely, not checked: the leaves draw in the first layer (the code sets no overlay flag on them), so the floor and the
  Pokémon can hide them.

Types 2, 3, 4, 6, 7 and 10 appear in no move file: their code is complete but unused by moves (likely used nowhere,
unless another system starts them directly). This also settles the numbering: the moves that ran the blur functions
in the PC sweep carry type 9, and the metallic or icy moves carry type 8.

### The bolt (type 1, Thunder), in detail

**verified** from the code (`effect_visual.c`, `tracefx.c`) unless marked. "A random offset"
below means one draw of *randpm* (`fn_800E0BA0`): two `HSD_Randf()` calls added, minus 1, so a value from -1 to 1 that
favours the middle.

- **Its settings** (the descriptor, layout in [wzx.md](wzx.md)): three vectors (jitter, spread, scale) and a step;
  a height, a width and a branch chance; the number of strikes and the branching depth; a colour; the owner's part
  the bolt lands on; and a texture. Thunder's damage half has two such entries.
- **When the strikes come** (`boltStartEffect`): each strike lasts *span = frames / (count − 1)*, and strikes start
  *step = (frames − span) / count* apart: strike *i* runs from *i × step* to *i × step + span*. They overlap.
- **The shape** (`_boltSubdivide`): a strike starts at height *height* above its landing point and adds points
  downward, each moved by *spread × a random offset* plus the step, until a point falls below 0 (at most 200 points).
  Then it may fork along the new stretch, with a chance of *branch × (max depth − depth) / max depth*: deeper forks
  are rarer. A fork starts at half the width and tapers. It ends below a quarter of its starting height, or by chance
  (*1 − 1 / (depth + 1)*).
- **Every frame** (`_boltUpdate`, advanced by the frame step): every point of each active strike wanders by *jitter ×
  a random offset*, adding up over time (the crackle). Past its middle, the strike's colour fades evenly to nothing.
- **Drawing** (`_boltDrawStrip`): over the strike's first half the bolt is revealed from the top down. Each segment is
  a flat strip turned toward the camera, half its width on each side. The strike is placed so the trunk's last point
  lands on the owner's part (scaled by the scale vector). One quirk: the direction to the camera is measured from the
  segment's position *before* that placement to the camera's real position, so the strips don't face the camera
  exactly.
- **Blending** (`_boltRender`): additive (source alpha plus what's already there), with depth testing and depth writes
  on and no back-face culling.
- **The bolt draws from the game's one shared random generator** (`HSD_Randf`, [rng.md](rng.md)) when it builds a
  strike and on every frame it runs. So, like the particles, it shifts every later random roll; a port or editor that
  changes it needs the separate generator for looks.

### Overlay effects draw after a depth reset

**verified** (a frame capture and a depth log of the PC build; the functions are fully matched):
the frame is drawn in two layers ([gfx.md](gfx.md), "The two layers and the depth reset"). Between them a full-screen
quad pushes the depth of everything drawn so far to the back. Layer 0 holds the battle room, the Pokémon and ordinary
particles; layer 1 holds models flagged 0x200 and the particles they spawn, and everything the screen effects spawn.
So **overlay effects such as a hit burst are never hidden by the arena floor or the room**, while within each layer
depth works normally (particles test less or equal unless their "no depth compare" flag is set, and write depth only
when their texture-edge flag is set). How a move chooses the layer (**verified** from the code; Fire Punch and
Flamethrower drawn this way match the game's pictures):
- **Particle entries:** bit 0 of the entry's mode word (the "draw link", [wzx.md](wzx.md)) starts its generators in
  layer 1, so the room never hides a hit's effects.
- **The Pokémon playing the half:** in layer 1 while its sequence runs when bit 1 of the sequence header's flags is
  set (`wazaSequenceStart` → `wazaSequencePokemonMotionStart` → `fn_800E3CC8`), back in layer 0 when it stops.
- **Model entries:** in layer 1 when bit 0 of the entry's flags is set.
- **An example:** both halves of Flamethrower leave header bit 1 clear, so Espeon and Umbreon stay in layer 0 and the
  flames are never hidden behind Umbreon. That is also why the flames light up Umbreon's body in the game.

## Flow

1. Boot: `GSEffectInit(32)` builds the slot table and registers two background tasks: one runs every active effect's
   update, the other its render.
2. Move file load: each kind-4 entry's parameters are parsed (`fn_801364A8`) and the type's start function (above)
   takes a slot, allocates the state block and registers the callbacks. The slot id goes into the entry.
3. Entry start (its time point comes): the entry fills the effect's per-battle fields (the owner's model, its scale,
   for type 0 the battle room's models from the current floor) and starts it.
4. Every frame: update, then render, for every active effect; an update returning 0 stops it.

## Disc data

Effects have no files of their own: their parameters (and any texture or model archive they need) are embedded in
the move's WZX file, entry kind 4.

## Changing it

New effect types would need a new start function and an entry type number; the parser's switch and the start
switch in the move code both list the 13 types. Parameters can be changed per move in the WZX entry.

## Efficiency

A fixed slot table (no allocation when an effect starts, only its state block), effects parsed once at load, and
textures or models embedded in the move file, so starting a move loads nothing.

## Verified (2026-10-16)

In the PC port, all 1,098 move files run their effects for 60 frames without crashing (after fixes to the after-image,
screen-blur and colour-key code); Thunderbolt's colour fade darkens Outskirt Stand's battle stage, and its glow
and lightning draw (frame captures, checked by eye). Not yet compared with the original game side by side.

## Open questions

- Whether anything outside the move files starts types 2, 3, 4, 6, 7 or 10.
- A rebuilt bolt shows its first strike a few frames before the game's at the same tick count;
  the cause is not known yet.
- The reset callback's exact role versus stop.
