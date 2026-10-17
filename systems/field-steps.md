# Field: footsteps and step callbacks (poison, friendship, P★DA mail)

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-16). How the overworld counts the player's steps and what runs on each step.
Claims are **verified** from the byte-matched C in `hero_move.c` and `mailMain.c` unless marked; nothing here was
watched running.

## What it does

The overworld has no tile grid, so "a step" is distance: the player's movement work adds up the distance walked, and
every **7.0 units** counts as one footstep. Each footstep calls up to **8 registered step callbacks**. Four are known:

| Callback | Registered by | What it does per step |
|---|---|---|
| `cbPoison` (`cbPoison__Fl15FootStepCounterl`) | hero movement init | field poison (below) |
| `cbTsureFriend` | hero movement init | friendship walk counter (below) |
| `cbStep` (`mailMain.c`) | `mailMainInit` | P★DA mail delivery (below) |
| `fn_801ED3B8` | `fn_801ED388` (id kept in `lbl_8047B5B8`) | heart gauge while walking, the day care, a story step counter (below) |

The callback names are the original ones (mangled C++ names survive for two: `FootStepCounter` is the original name of
the step counter). *Tsure* = "companion": `cbTsureFriend` is the party's walking friendship.

**Healing by script** (**verified**, `gs_range_801C766C.c`): script command 216
(`std216_partyStatusUpdate_8ae(mode)`) heals the whole party (mode 0, the same `fn_801D0AFC` the blackout uses) or
cures only poison and bad poison (mode 1); it does **nothing while variable 0x8AE is set** (the Mt. Battle challenge run
of pause-menu.md: no healing between battles there).

## Data structures

In the hero movement work (`HeroMoveWork`, `lbl_80426BD0`):

| Field | Type | Meaning |
|---|---|---|
| `stepAccum` | `f32` | distance walked since the last footstep; footsteps are taken off in 7.0 chunks |
| `stepCallback[8]` | `{func, arg}` | the step callbacks; `heroMoveAddStepCallback(func, arg)` takes a free slot and returns its index |
| `poisonSteps` | `s32` | footsteps since the last poison tick (0-3) |
| +0x184 | `u32` | footsteps since the last friendship tick (0-255) |

## Flow

1. **Every movement update** adds the distance moved to `stepAccum`; while it is at least 7.0, all registered
   callbacks run once and 7.0 is taken off (so a fast move can produce several footsteps in one frame).
2. **Field poison (`cbPoison`), every 4 footsteps:** each party Pokémon that is poisoned or badly poisoned
   (conditions 3 and 4, battle-turn-flow.md) loses **1 HP** (status 0x83). If any HP changed, the screen plays the
   poison flash (`fadeEffectDokuStart`). A Pokémon that reaches 0 HP faints: message 0x444E names it, and its
   friendship takes the "fainted" change (event 7). If no party Pokémon has HP left, the player **blacks out**:
   message 0x444F, **half the money is lost** (`heroDecPokedoru(money / 2)`), the whole party is healed
   (`fn_801D0AFC(1)`: `pokemonAllKaihuku` on each), and condition 0x3E (Hyper Mode, likely) is cleared from every
   Pokémon. Then the game **warps to a return point** (`fn_80113FE8`): a script-set one if a script has set it
   (script command 161, which stores a room id, a second id and an entry index in the field state `lbl_80408378`
   +0x48-0x50 and flags it at +0x51), otherwise the one in the save header (fields 5, 7 and 8: room id, second room id,
   entry index; save-data.md "Save header"), and requests that room (`floorRequestChange`). So the save header's
   "floor" fields are a return point the scripts keep up to date. (**verified** from the decoded
   room scripts of the US disc): Command 128 (`std128_mapSetReturnMap`) appears in **one** room script, Phenac City's
   Pokémon Center basement (**verified**: the shared common script uses it too, in its PC-box
   function 13, which records the return point before opening the box; common-script.md), where it sets the return point to the current room just before the save menu opens;
   command 161 (`std161_fieldSetPending`) appears 27 times, in story areas (the Shadow Pokémon Lab, Pyrite Town, Agate
   Village's shrine, The Under, the Team Snagem hideout, ...), each setting a fixed return point (room ids written in
   the script; in the example read, under a story-flag condition). A third command, 246 (`std246_saveSetStatus578`), writes
   the save header's return point directly; it too occurs in only one script. So inside those story areas a blackout
   returns the player to the spot the area's script chose (**likely**); elsewhere it uses the save header's fields. (**verified**): **every save** writes those fields from the field state's "return map" (+0x3C-0x44,
   read with `fn_8011418C`; set by command 128; whether room changes also set it is not traced) just before memory-card task 4 (the save menu,
   `std235_menuShowSaveMenu`, and the P★DA's save), so a blackout and a Continue both go to the return map stored at
   the last save. This is Gen 3's field-poison rule (1 HP per 4 steps, poison can faint; **likely** the same source, as with
   the battle engine).
3. **Friendship (`cbTsureFriend`), every 256 footsteps:** every valid party Pokémon gets the friendship change for
   event 5 (`pokemonGetFriendFormPokemonFriendFilterId(mon, held item's effect, 5)`, so the Soothe Bell scales it: friendship.md). Half of these ticks are skipped at random, and the amount comes from Gen 3's friendship table (friendship.md).
4. **Shadow Pokémon and the day care (`fn_801ED3B8`), every footstep:**
   - each party Shadow Pokémon (not yet purified, gauge above 0) counts footsteps in its Shadow record; at 256 the
     gauge takes the "walking" drop (event kind 1, battle-shadow.md) and the count restarts;
   - the **day-care** Pokémon (save section 0x0B, present when its first byte is set): a Shadow Pokémon there counts
     the same way and takes the "day care" drop (kind 3) every 256 steps; any other Pokémon below level 100 gains
     **1 EXP per footstep** and learns any level-up move it reaches (Gen 3's day-care rule);
   - while story flag 0xD0 is set, story variable 0xD1 counts footsteps up to 10,000, and the count is also written
     into the day-care block (+2). When flag 0xD0 is clear the value written there is whatever the variable held.
     (**verified** from the assembly): That variable lives in a saved register
     (r31) which, on the flag-clear path, still holds the caller's value. The only caller is the step-callback loop
     inside `heroMoveMain`, which has loaded the hero-move work's first word, `leader` (the leading party member, 0
     or 1), into that register. So with 0xD0 clear the day-care block's +2 receives the **leader index (0 or 1)**;
     with it set, the incremented step value. A deterministic original quirk, not random memory; the PC port
     reproduces it.
     What 0xD0 / 0xD1 are for is not traced (**guess**: the day care's "steps since deposit").
5. **P★DA mail (`cbStep`), every footstep:** for each mail entry that has not been received yet and whose send rate
   is not 0xFFFF, it draws a random number (`HSD_Randf`, 0-1) and the mail **arrives with chance rate / 64,000**
   per footstep (`rand < rate x (1/64) / 1000`), provided its condition holds: the mail **stage** value (`fn_801D1338`: mail save section
   +0x444, below) compared with the entry's value by one of seven tests (=, >, <, >=, <=, !=, always), and a second
   condition by type (**likely** a story flag or a script check: `chkMailSend` uses both; not every type read). Arriving mail goes into the mailbox and starts the **new-mail notice** unless one is
   already running (`mailMainStartSe`): the music is paused and the mail's jingle plays (its own 64 KB sound
   resource), and a mail icon (texture 0x0C881200) **blinks in the top-left corner** (30 frames on, 30 off) for
   600 frames, 10 seconds at 60 fps; opening the pause menu ends the notice early (`mailMainReceiveTerminate`, called
   by the pause menu). Scripts can also send mail directly (`mailMainSendByScrpt`).
6. **Reading a mail** (**verified**): the first time the player views a mail in the P★DA
   (`menu_pda_mail`: not yet read -> mark read, then `fn_801D228C`), the mail's effects run: it can set a story flag,
   set the **mail stage** (mail section +0x444) to its own value, update listed Shadow Pokémon records (mark them and
   raise each one's Shadow state to at least a given value, `std243_shadowPokemonSetState`: **likely** how mails
   reveal Shadow Pokémon to the player's records), run a script and queue a hero event. Because later mails' send
   conditions compare against the stage, **mails form chains**: reading one can make the next ones possible. (The
   decomp's field name `sortMode` for +0x444 and the comments on `fn_801D228C` / `fn_801D1364` are wrong.) The `mailMain.c` comments calling `cbStep` "Waza multi-hit effect setup" are wrong.

## Disc data
**The mail record** (**verified** from its accessors in `mail.c`; table `lbl_80478E9C`,
count `*lbl_80478E98`; 0x2C bytes; the decomp types it as `WazaEntry`, a wrong name). The table is in its own small module, entry `mail` of `common.fsys` (a REL), whose
prolog points the four globals at its data section: the Shadow lists at +0x000 (count at +0x6B8), the mail table
at +0x138 (count at +0x6BC) (**verified**: found from every module's prolog stores, checked here
against the module's relocations).

| Offset | Field | Accessor |
|---|---|---|
| +0x00 | u8 second condition's type (0xFF = none) | `mailGetSendCondType` |
| +0x01 | u8 stage comparison: 0 =, 1 >, 2 <, 3 >=, 4 <=, 5 !=, 6 always (0xFF = never) | `fn_801D142C` |
| +0x02 | u16 first row of its Shadow Pokémon list (0xFFFF = none) | `fn_801D1734`, `fn_801D1864` |
| +0x04 | u16 stage value to compare with (0xFFFF = never) | `fn_801D13E4` |
| +0x06 | u16 send rate: chance rate / 64,000 per footstep (0xFFFF = never by steps) | `mailGetSendRate` |
| +0x08 | u16 mail stage set when it is read (0xFFFF = no change; the decomp's "f32 duration" is wrong) | `fn_801D139C` |
| +0x0C / +0x10 / +0x14 | u32 messages: sender, subject, contents | `mailGetSenderName`, `mailGetSubject`, `mailGetContents` |
| +0x18 | u32 attached file group (a picture shown with the mail, **likely**) | `mailGetAttachFileGroup` |
| +0x1C | s32 second condition's value (a flag id for the flag type, ...) | `mailGetSendCondition` |
| +0x20 | s32 story flag **set** when read (0 / -1 = none; the code calls `std078_flagSetFalse`, whose name is swapped and which sets the flag) | `fn_801D1504` |
| +0x24 | s32 script run when read | `fn_801D14C0` |
| +0x28 | s32 hero event queued when read | `fn_801D147C` |

**What the US data shows** (**verified** by reading the module; counts only):
- **32 mails** from **9 senders**; 78 Shadow-list rows in 17 lists, raising Shadow states 1 to 5.
- **No mail arrives by walking:** every record has send rate 0 and the stage test "always", the second condition
  is type 0 with value 0 throughout. So in Colosseum the footstep mail check (step 5) never fires on its own. Mail
  is **sent by the room scripts** (`mailMainSendByScrpt(n)`, e.g. Outskirt Stand and its shop, under flag 1144),
  and the step machinery is **likely** shared code that XD uses more (a guess until XD's table is read).
- When read: 24 mails set a story flag, 16 update Shadow records, 3 show an attached picture, 2 run a shared-script
  function, none queues a hero event or changes the mail stage.

**The mail Shadow Pokémon lists** (`lbl_80478EA4`, 4-byte rows {u8 Shadow state to raise to, u8 -, u16 Shadow
Pokémon id}, each list ending at id 0): the Shadow records a mail updates when read (step 6). Messages 0x444E and 0x444F are in the
field text.

## Changing it
- **Step length:** the 7.0 constant (float data) sets how far one footstep is for all callbacks at once.
- **Poison:** the "4" and the 1-HP loss are in `cbPoison`; a port option "poison leaves 1 HP" (the Gen 4+ rule) would
  change only the `poison == 0` branch.
- **New step effects** (a step counter for mods, Repel-style timers): register a callback; only 8 slots exist, and 3-4
  are used.
- **Mail timing:** data only (each entry's rate and conditions).

## Efficiency
Distance is accumulated as one float and callbacks are a fixed array of 8; nothing is allocated per step.

## Open questions
- What story flag 0xD0 and variable 0xD1 mean (the day-care step count, a guess).
- Whether the money loss shows the amount; whether other Pokémon Centers set the return point another way.
- Whether field poison really happens in normal play: it is in the code, but whether any field poisoning survives a
  battle in Colosseum (conditions are kept after battle in Gen 3) is a visual check.
