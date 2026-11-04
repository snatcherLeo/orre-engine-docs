# Movies (THP video playback)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** for the player; XD's videos differ)

Status: **partial** (2026-10-17): every video, its sound, the player loop and the hand-off after each video are
known; the credits' cue table is still to confirm. Code: `src/game/movie.c`; the player itself is Nintendo's THP SDK
(`src/dolphin/thp/`), wrapped as `GSmovie*`.

## What it does

Plays the game's few prerendered videos: the boot logos, the opening cinematic, the credits and the
title-screen attract demo. Everything else that looks like a cutscene (story scenes, Shadow Pokémon
reveals, admin exits) is **rendered in-engine** by scripts, not video.

## The complete list (read from the disc's file table)

| File | Size | Frames | Rate | Length | Played by |
|---|---|---|---|---|---|
| `movie/gs_logo.thp` | 2.3 MB | 130 | 29.97 | 0:04.3 | `movieGsLogoStart` (room 0x39A `genius_logo`, boot) |
| `movie/tpc.thp` | 1.9 MB | 75 | 29.97 | 0:02.5 | `movieTpcStart` (room 0x399 `pokemon_logo`, boot) |
| `movie/openingdemo.thp` | 61.1 MB | 1926 | 29.97 | 1:04.3 | `movieOpeningStart` (new game / intro) |
| `movie/autodemo01.thp` | 96.8 MB | 2700 | 29.97 | 1:30.1 | `movieStartAttract` (title-screen idle demo, BGM 0x494) |
| `movie/staffroll.thp` | 104.1 MB | 5250 | 29.97 | 2:55.2 | `fn_80035F64` / `staffrollStart` (0x800361C0) (credits) |

Five files, about 266 MB together, the bulk of the disc's size budget for video. The two logos are
played by their rooms' start functions (**verified**, new-game-flow.md, "Boot").

## Where each video's sound comes from (verified 2026-10-14 late: THP headers + `src/game/movie.c`)

A video's sound can come from three places: the THP file's own audio track, a **background song** the game starts
through MusyX at the same moment it opens the video (`std104_soundSetBackgroundMusic(id, 0, 0x7F)` right after
`GSmovieOpen`), and **frame-cued sound effects** (a table of `{frame, sound A, sound B}` that the wait loop fires with
`soundStartSE` when the video reaches each frame).

| Video | THP audio track (header) | Song started with it | Frame-cued effects | Started by |
|---|---|---|---|---|
| `gs_logo` | **none** (video only) | **0x4D1** | none seen | `movieGsLogoStart` |
| `tpc` | **yes**: 2 ch, 32 kHz, 80,152 samples (2.5 s) | none | none seen | `movieTpcStart` (opens with loop = 1) |
| `openingdemo` | **yes**: 2 ch, 32 kHz, 2,058,287 samples (64.3 s) | **0x495** | yes: 5 cues (`lbl_802E51A8`, `fn_80035E04`) | `movieOpeningStart` |
| `autodemo01` | **none** (video only) | **0x494** | none seen | `movieStartAttract` |
| `staffroll` | **none** (video only) | **0x4C9** | probably the 33-cue table `lbl_802E50E0` (`fn_80036240`, which follows the staff-roll start in the file; to confirm) | `staffrollStart` (0x800361C0) (wait / post-game handling in `fn_80035F64`) |

So **the game plays most of a video's sound itself**: three videos are silent files whose music is a MusyX song, and
even the opening's file audio is only one layer (measured: quiet, well under the song) under
song 0x495 and five cued effects. Only the Pokémon Company logo's sound is entirely in the file.
Two files carry audio (`tpc` and `openingdemo`); the other three have none. Sounding like the game needs the song plus the cue
effects mixed in at their frames; a port remake must start the same song and cues.

### How the ending reaches the staff roll

(**verified** from the decoded room script and the movie code unless marked):
1. **The final scene is a room script:** Realgam Tower Colosseum (`D4_casino_colo`, room 118 = 0x76), function 13,
   which runs after the battle with Evice (the story flag 1143's step). It plays the closing dialogue and the Ho-Oh
   flight. The flight has **variants**: on a later ending (flag 1144 already set), the script counts the 48 Shadow
   Pokémon's "snagged" flags (2126 + n) and "purified" flags (1262 + n), and with all 48 of each it switches to a
   different model and camera animation (shadow-system.md). It starts song `end_ev_2_song`, then **sets flag 1143**
   and **links to room 0x39F, `staffroll`**.
2. **The staff-roll room** opens `staffroll.thp` and starts song 0x4C9 (`staffrollStart` (0x800361C0)), then waits until the video
   ends (`fn_80035F64`).
3. **After the roll**, if the previous room was 118 and flag 1142 is set (set during the final battle's steps), the
   game **sets flag 1144** (the "game cleared" flag the postgame and the Ho-Oh gift check), marks the clear in the save
   data and **asks to save** until the player declines or the save succeeds. A replay from another room skips this.
4. It then links to **Phenac City (room 2, entry 1)** with a fade script: the postgame starts there.

So the ending has two editable joints: the room script's link to the staff-roll room (a script change) and the
movie room's return to Phenac with its flag and save step (code: a hook point for a port).

## How a movie plays (found while running the movie player natively)

- **Two decoder threads, above the game** (*verified* from the code): an on-memory movie (the whole file in RAM, as for
  the logo movies) runs an audio decode thread at priority 12 and a video decode thread at priority 20, both above the
  main game thread (16). Each decodes ahead into **3 rotating buffers**.
- **They stop themselves at the end** (*verified*, code and probes): for a non-looping movie each decoder suspends its
  own thread on the last frame; the later stop / close cancels the threads.
- **Sound goes through MusyX** (*verified*): the sound mixer calls the movie's stream callback
  (`StreamUpdateCallback`, `fn_801E260C`) from the audio interrupt; it hands used buffers back to the decoder with a
  message, which wakes the decoder.
- (*likely*) The game asks to open `tpc.thp` again while the previous movie is still open; the open is refused ("Fail
  to open the thp file") and nothing else happens.

## Data structures

- **Sound cue table** `lbl_802E50E0`: 0x21 entries of 6 bytes `{u16 frame, u16 sound A, u16 sound B}`,
  used by `fn_80036240`. When the movie's current frame reaches an entry's frame, both sounds (if
  nonzero) are started with `soundStartSE` and the cursor moves to the next entry. So sound effects are
  **not** baked into the video's audio track; the game fires them on frame numbers.

## Flow

Every movie function has the same shape (`fn_80035E04`, `fn_80036240`, `fn_80035F64`):
1. `GSmovieOpen(path, loop)` (THPPlayerOpen + play).
2. Loop while `GSmovieIsActive` returns "playing", yielding with `_threadSwitch` each frame (the
   movie runs inside a cooperative GS thread, see the script VM / thread notes).
3. Each frame: skip if the game is in states 0xB / 0x5 (`fn_8017B1AC`, probably paused or a system
   dialog); read the pad (`GSinputButtonsChanged(1) & fn_800F7BC4(1)` (0x800F7AF0))
   and stop the movie (`GSmovieClose`) if A, B or Start (0x100 | 0x200 | 0x1000 = 0x1300) is pressed;
   otherwise step the sound cue table.
4. After the movie: play a sound, go to the next room and set a fade script. The opening goes to room 1, Outskirt
   Stand (`floorLink(1, 0)`); the attract demo and the Genius Sonority logo go to room 0x384, the title (**verified**,
   table below).

## Disc data

Standard THP files (Nintendo's Motion-JPEG container with interleaved audio), header magic `THP\0`,
frame rate at +0x10, frame count at +0x14. Read in place from the disc by the THP SDK's own DVD
streaming.

## Changing it

- Replacing a video: any THP with the same name works, but the **sound cue table is keyed to frame
  numbers**, so a re-cut video needs a new cue table or the sounds fire at the wrong moments.
- For in-engine remakes of the videos (so swapped player models appear): only
  `openingdemo`, `staffroll` and `autodemo01` show characters. The story cutscenes are already
  in-engine, so they pick up model swaps for free.

### Mod surface
- **Data it reads:** the THP videos on the disc and the opening's sound cue table (frame number, two sound ids).
- **Hooks it would need:** a video substitution by name (higher-resolution or re-cut videos), the cue table per video,
  and an option to replace a video with an in-engine scene.
- **Fixed-size limits:** the cue table's entry count; the video size the player was built for (its buffers).
- **Must stay vanilla in GameCube-accurate mode:** the original videos and their cue timing.

## The hideout cutscene: in-engine opening scenes that the game never plays (found 2026-10-14 late)

The opening video's characters are pre-rendered (THP frames; nothing in it is a live model). But the opening also has
**in-engine scenes**: the Team Snagem Hideout rooms `S2_out_1`, `S2_hallway_1`, `S2_bossroom_1`, `S2_snatchroom_1`,
`S2_out_3` (room ids 0x81-0x89, floor class 0xA0 "special") hold cutscene-only models (`ken_c1` = Wes in high detail,
`boss_s_c1` / `c2` = Gonzap, `snach_zako_c*` grunts, `blacky_c1` Umbreon, `bike_c1`, `airmd` Skarmory and 33
`opening_characters_NN` props) and up to 20 camera shots each, with nearly empty scripts. The scenes are sequenced
from **C code**, not scripts: `demoScene14` (0x80090720), `demoScene13` (0x800909E4), `demoScene12` (0x80090D34), `demoScene11` (0x8009100C), `demoScene10` (0x80091564),
`demoScene06` (0x80091DA4), ... play map-object animations (`std115_mapObjectPlayAnimation`), camera shots (`cameraPlayAnime`,
`cameraWaitSyncAnime`), attach characters to each other (`std070_characterAttachToCharacter`) and then
`floorRequestChange(0x81..0x89)` to the next hideout room. Those functions sit in `src/game/gbaCommunication*.c`,
which is a **misleading file name** for this range (they are the opening cutscene, not GBA link code).

**What loads them (code + data):**
- The five hideout rooms share one room "main" function in the floor table (`common_rel`, floor entry +0x40):
  `fn_8008C700` (+0x3C pre = `fn_8008C78C`, +0x44 post = empty `fn_8008C6FC`). It reads story flag **0xB5D**, calls
  entry *flag* of the 31-entry scene table `lbl_802EEC70` (`demoScene30` (0x8008C7B0) ... `demoScene00` (0x80092B2C), stored last-to-first),
  then sets the flag to flag + 1 (wrapping at 31). Each scene plays its camera / map-object animations and ends with
  `floorRequestChange` to the next hideout room, so once started the 31 scenes chain through the five rooms.
- **Nothing in the retail flow starts the chain:** no code in main.dol or in any of the 171 REL modules loads room
  0x81 / 0x82 / 0x83 / 0x87 / 0x89 except the scene functions themselves; no room script warps there (scripts only warp
  to the PC-box menu room 0x391); flag 0xB5D is written only by the dispatcher. The opening video's code
  (`movieOpeningStart`) hands off to room 1, Outskirt Stand (`floorLink(1, 0)`), and the title's idle timer goes to
  room **0x39C** (the attract demo), not to the hideout.
- **Verified in Dolphin (2026-10-14):** a new game plays the pre-rendered opening video and then
  loads straight into Outskirt Stand with Wes; no in-engine hideout scene appears.
- **Conclusion:** the hideout scenes are **unused content**, very likely the in-engine source the opening video was
  rendered from (the cutscene Wes matches the video: "Verified in Blender" below). They are only reachable by forcing a load of one of
  the hideout rooms (a debug floor select or a mod). Not checked exhaustively: per-room door / warp tables
  (no reference found, but those tables are only partly decoded), so "unused" is a strong finding, not a proof.
- **What already exists for an in-engine remake (2026-10-14, verified in Blender):** the cutscene Wes `ken_c1` (below), together
  with the 31 chained hideout scenes (cameras, map-object animations, character attachments), nearly everything an
  in-engine opening needs is on the disc: models, rig, props, rooms, cameras and animations.
- **Why GS shipped a video instead (guesses, not verified):** effects and lighting too expensive in real time;
  memory and loading cost for a one-time intro; guaranteed picture quality.
- For a remake: these 31 scenes are a ready-made starting point: the cameras, map-object animations and character
  attachments already exist on the disc.

**Verified in Blender (2026-10-14):** the cutscene Wes `ken_c1` in its default pose
matches the Wes of the pre-rendered opening video exactly, so the video was almost certainly rendered from these same
assets. Its coat has its own bones; its face is one painted texture with a grin (no mouth joints, no face swaps in this
model). This makes an in-engine remake of the video practical from the disc's own models (*likely*).

## For in-engine remakes (2026-10-14)

An in-engine remake of a video would slot in where the original plays. What a remake has to replace, per video:

| Video | Story point | Code that plays it | What happens after | Remake notes |
|---|---|---|---|---|
| `openingdemo` | new game: the opening cinematic (Wes blows up the Team Snagem hideout and steals the Snag Machine) | `movieOpeningStart` | a sound, then `floorLink(1, 0)` = `S1_out`, Outskirt Stand, entry 0 (in `fn_80035E04`; room 0x384 is `title`) and a fade script | frame-keyed sound cues (`lbl_802E50E0`) must become timeline events; skippable with A / B / Start |
| `autodemo01` | title screen left idle (attract demo): started when the **title song 0x46A stops playing** (`fn_8001EF78` / `fn_800203B4` link room 0x39C `auto_demo`; new-game-flow.md step 3) | `movieStartAttract` | back to the title (to confirm) | plays with BGM 0x494 instead of its own audio (to confirm) |
| `staffroll` | credits after the ending | `fn_80035F64` / `staffrollStart` (0x800361C0) | to confirm | 2:55 of footage; the longest remake |
| `gs_logo`, `tpc` | boot logos | `movieGsLogoStart` / `movieTpcStart` (rooms 0x39A / 0x399) | the next boot room; the Genius Sonority logo goes to the title (0x384) | no characters; remake optional |

**Hook point:** each player function opens the movie (`GSmovieOpen`), loops on `GSmovieIsActive`
yielding each frame, then hands off. A remake replaces the open / loop part with an in-engine scene and keeps the
hand-off, so the story continues exactly as before.

**How in-engine cutscenes are driven (what a remake would be built from):**
- **Scripts:** the room script (bytecode, `script-vm.md`) sequences the scene: entering a room runs function 0 of
  its script (`flags.md` (not yet published)).
- **Character animation:** `std051_characterSetAnimation` / `std154_characterPlayAnimation(group, resource,
  anim_index, frame, loop)`, waiting with `peopleWaitSyncMotion` (`models-animation.md` (not yet published), "Original limits").
- **Cameras:** room camera files (`cam_<room>_shot_NN`, HSD camera animations: `floor-resources.md` (not yet published) "Room cameras",
  `animation-tracks.md`); which script native selects a shot is still to document.
- **Movement:** NPC walking and object attachment (`std166_mapObjectAttachToObject`, as in the bike travel scene).

## Efficiency

THP streams from disc with a small ring of read-ahead buffers (SDK), decoding JPEG frames on the CPU.
The game thread only polls and yields, so the movie costs one cooperative thread and no extra
allocation per frame.

## Open questions

- What states 0xB / 0x5 of `fn_8017B1AC` are.
