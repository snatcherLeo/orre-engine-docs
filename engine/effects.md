# Screen and model effects (GSeffect and the WZX effect types)

Status: **partial** (2026-10-16). How the game runs the visual effects a move asks for (lightning, glow, colour fades,
after-images, sea and surf, distortion, ...). Read from `gs_effect.c`, `effect_visual.c`, `tracefx.c` and the move
code that starts them; checked by running every move file in the PC port (the decompiled code running natively) (see "Verified").

## What it does

Some move-effect entries (WZX entry kind 4, "screen effect") are not particles or models but small programs: an effect
with its own state block and up to five callbacks. A general **effect system** (`gs_effect.c`) owns a fixed number of
effect slots, starts and stops effects, and calls every running effect's update and render once per frame. Thirteen
effect types exist; the WZX entry says which one and carries its parameters.

## Data structures

**Effect slot** (one per running effect; 32 slots, set up at boot with `GSEffectInit(0x20)`, `fn_80130CE0`): id,
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
| 1 | electron | 0x60 | stop, start, reset, update, render | moving electric points |
| 2 | (`fn_8013A42C`) | 0x24 | start, update, render | arcs between two joints of the owner's model: a curve through four control points with random offsets, redrawn with jitter and a per-frame chance (electric crackle over the body; likely). Its render (`fn_8013A520`) builds curves between model parts |
| 3 | lightning | 0x70 | stop, start, reset, update, render | lightning bolts drawn as strips |
| 4 | trace (`tracefx`) | 0x2C | own callbacks | trails (ribbons following a joint) |
| 5 | leaf | 0x4C | stop, start, reset, update | leaves (an embedded model archive) |
| 6 | sea | 0xAC | stop, start, reset, update | water surface |
| 7 | surf | 0xD0 | stop, start, reset, update | wave model |
| 8 | env map | 0x18 | init, stop, start, reset, update | environment-map shine on the Pokemon |
| 9 | blur | 0x20 | init, stop, start, reset, update, render | two modes (a flag in its block): **after-images**, redrawing the Pokemon at earlier animation frames with falling alpha (Quick Attack and similar); or **screen blur**, capturing the last frame (GSgfx's back-framebuffer capture) and drawing it over the screen with fading alpha (`fn_8013E258`) |
| 10 | aura | 0x34 | init, stop, start, update, render | glow around the Pokemon |
| 11 | billboard (`patchiru`) | 0x24 | init, stop, start, reset, update | camera-facing textured quads ("patchiru" is Spinda's Japanese name: its spots, guess) |
| 12 | distortion | 0xB4 | init, stop, start, update, render | screen distortion (heat haze, ripples) |

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
| 1 electron | 2 | 1 | Thunder (damage half) |
| 5 leaves | 14 | 8 | Petal Dance, Feather Dance, Magical Leaf, Leaf Blade, Razor Leaf |
| 8 env-map shine | 15 | 15 | Harden, Iron Tail, Transform, Camouflage, the frozen status loop |
| 9 blur | 13 | 13 | Quick Attack, Sunny Day, Double Team, Agility, Vice Grip, Mega Kick, Mach Punch |
| 11 billboard | 30 | 30 | the battle-name banners of the Colosseum battles (one file per battle and per scenario) |
| 12 distortion | 39 | 39 | Nightmare, Sweet Scent, Thunder Wave, Imprison, Transform |

Types 2, 3, 4, 6, 7 and 10 appear in no move file: their code is complete but unused by moves (likely used nowhere,
unless another system starts them directly). This also settles the numbering: the moves that ran the blur functions
in the PC sweep carry type 9, and the metallic or icy moves carry type 8.

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
- The reset callback's exact role versus stop.
