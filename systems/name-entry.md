# Name entry

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure; XD's own data and additions not checked)

Status: **partial** (2026-10-17). The screen where the player types a name: the player's own name at New
Game, the partner's name, Pokémon nicknames and PC box names. Code: `menuNameEntry*.c` (the screen),
`menuNameEntry_exact_80029558.c` (the openers). **Verified** from the decomp's C and the US text and data unless
marked.

## What it does

Name entry is a **menu room** of its own, room 0x390 `name_entry_menu`, pushed on the room stack like the bag or the
P★DA (menu-rooms.md). The caller fills a small request block, pushes the room and waits; when the room pops, the
caller reads the result. One screen serves four kinds of name, chosen by a **mode**:

| Mode | Used for | Title (paraphrased) | Max length | Preset names | Caller |
|---|---|---|---|---|---|
| 0 | the player's name | "Your name?" | 7 | three (the hero's default name and two others) | New Game, `gs_title.c` (new-game-flow.md step 5) |
| 1 | the partner's name | "Your name?" | 7 | three (her default name and two others) | Phenac City's script (`M1_out`), when she first introduces herself and asks the player's name back |
| 2 | a party Pokémon's nickname (the second argument is the party slot; refused if the slot is empty) | "<species>'s nickname?" | 10 | none | the PC / party tools (`etctool.c`) and after purification (`pokemon_relive.c`, without the fade) |
| 3 | a PC box's name (the second argument is the box, 0-2) | "Name of the BOX?" | 8 | none | (no caller found in the C yet) |

The lengths are in characters. (Mode 1's script call is the only `menuNameEntryOpen` in the decoded US scripts.)

## Data structures

- **The request block** (`lbl_803A2068`): the name being edited (a GS character string, so the block starts with
  the text), then the mode (+0x18), the index (+0x1C), the **result** (+0x20: 1 if the name changed, 0 if not), a
  "fade" flag (+0x24: 1 for `menuNameEntryOpen`, 0 for `menuNameEntryOpenNoFade`) and a "running" flag (+0x28).
  `menuNameEntryGetLastName` copies the finished name out.
- **The per-mode table** (`lbl_80266DD8`, 4 rows of 16 bytes): title message, maximum length, pointer to the preset
  names (message ids) and how many there are (3 for modes 0 and 1, none for 2 and 3).
  (**verified** from the code and a native run): The whole screen's setup data is
  one read-only block the code reaches by offsets from `lbl_80266DC0`: +0x00 the three player presets (`lbl_80266DC0`),
  +0x0C the partner's three presets (`lbl_80266DCC`), +0x18 the four per-mode rows above, and +0x88-+0x94 **one
  model id per mode** (modes 0 and 1: the same model, 0x00F70400; mode 2: none, the Pokémon itself is shown; mode 3:
  0x11470400). The preset list's callbacks reach the mode's id list through the window's menu argument (+0x60).
- **The letter pages**: each page is a message whose text is the grid itself (rows like "ABCDE FGHIJ 789", spaces
  as gaps), one set for upper case and one for lower case, plus a row of symbols; X switches pages.
  In detail (**verified** from the linked code):
  - **Each page is four messages, one per grid line:** page 0 (green) 0x2F04-0x2F07, page 1 (blue) 0x2F08-0x2F0B.
    They come from a two-row page table (`lbl_80266E18`: tint colour, the X button's label, four line messages).
  - **The grid is 15 letters wide and 4 lines tall**, with a 16th column for the three buttons. Letters are 27 px
    apart, lines 35 px apart.
  - **The cursor never wraps:** it stops at the edges.
  - A gap in a line types a space.
- **The case-pair table** (`lbl_8047B920`, two message ids): R looks up the last letter in these pair lists and
  swaps it for its partner. On the US disc the first slot is empty and the second is a message of upper / lower
  pairs ("AaBbCc..."), so **R toggles the last letter's case**; the screen's own label for R says so ("A⇔a").
  The code's name for this is `exchangeDakuon` ("swap the voiced mark"): see Region notes.

## Flow

1. The caller: `menuNameEntryOpen(mode, index)` checks the arguments (mode 0-3; for mode 2 the slot must hold a
   Pokémon; for mode 3 the index must be 0-2), clears the name and the result, pushes room 0x390 and yields until
   the room is popped. `menuNameEntryOpenNoFade` does the same and sets the shared fade-in script (`0x05960008`)
   for the way back.
2. The room: a 3D model is shown next to the text: the model of the mode's model id: **the same model for modes 0 and 1**
   (both use model 0x00F70400, **likely** the
   player character; with a motion), the Pokémon itself for mode 2, another model for mode 3 (**likely** a box). The animated "ball" background runs
   behind (message-windows.md, "Efficiency").
3. **Modes 0 and 1 start with a short list** (menu 0x70): the three preset names and an entry for typing a new one (**likely** the first
   item: choosing it opens the keyboard). Picking a
   preset asks "Is <name> OK?" (a yes / no box that starts on **Yes**); yes takes it, no returns to the list.
   Backing out of the keyboard returns to the list (**likely**, from the loop).
   **B does nothing on this list** (**verified** from the code, the menu record and a run): the
   list's own button callback (`menuNameEntrySelectButton`, set as menu 0x70's button handler in its menu record)
   closes the list on A, and only while B isn't held; it never reads B. So the player can't leave the name menu: the
   game gives no way back to the top menu once New Game's question is answered, and the partner must be named too.
   (`menuNameEntry` still has a branch for a list result of -1, the value a normal menu returns on B, which opens the
   list again; with this callback it isn't reached from B.)
4. **The keyboard** (modes 2 and 3 go straight here): the letter grid with the cursor; A types the letter under the
   cursor (once the name is at its maximum length the cursor jumps to a fixed slot: **verified**,
   DONE; typing at the maximum overwrites the last letter; an empty or all-blank name becomes the
   default, the first preset in modes 0 and 1), B deletes,
   X switches the letter page, R toggles the last letter's case, START finishes. The on-screen buttons are
   "DONE", "BACK" and the R hint. Key bits: message-windows.md, "Window input keys".
5. **Finishing** stores the name by mode: mode 0 only keeps it in the request block (the title code then passes it
   to `savedataCreate`, so it becomes the save's player name); mode 1 writes it to the player record's status 0x17
   (the partner's name); mode 2 sets the Pokémon's nickname; mode 3 renames the PC box. The result says whether it
   changed, then the room fades out and pops.

### Frame by frame (**verified** against the console on the New Game screen)
- The room's script makes one native call, `menuNameEntry` (script native 23); the whole screen runs inside it,
  so the script itself executes nothing while the player chooses.
- **The background particles** (`menuNameEntryBackCtrl`, a menu tick callback inside `menuDaemon`): on the screen's
  first frame they run 600 spawn-and-age steps at once (so the screen opens already full), then one step per frame.
  Each step draws the shared random generator once to decide whether to spawn, and four more times when it spawns
  (position, size, lifetime). So **every frame the screen stays open moves the shared random numbers**, which is why
  the New Game burst that rolls the starters' stats depends on exactly when the player confirms the name.
- **The way out:** after "Is <name> OK?" is answered, the list window closes (`menuCloseSync` waits until it is gone),
  then a 0.5 s fade to black (`fadeSet(3, 0.5)` and `fadeCheck(1)`), then the model and the screen are torn down.
- **`fadeCheck(1)` waits one frame more than the fade:** the fade advances in `menuDaemon` (the frame task with
  priority 0xFE), after the scripts and menus that wait on it (task 0x0A); the waiting thread sees the fade finished
  only the next frame and then yields once more. So the screen tears down two frames after the fade's last step,
  and the particles step during both.
- **Frame pacing on the console** (**verified** from Dolphin's per-frame capture and the game's
  frame-rate setter): the game asks for 60 frames a second the whole time the screen is open, but the console needs two
  refreshes to draw it, so the screen (and its particles) runs at 30 a second; when the fade to black starts the frames
  get light enough for one refresh each, so the fade and the particles' last steps run at 60. Anything that counts
  frames here therefore counts at 30, then 60, and a replay must copy both stretches to keep the particles' random
  draws in step.
- **After the screen's second use in the opening** (right after the battle with Folly; **likely** the partner's name;
  timings **verified** from the console capture): the black screen after the fade lasts about two seconds. The
  game reads a ~0.65 MB archive in about 19 frames, then has three long frames (4, 3 and 6 refreshes, **likely** its
  unpacking and setup), and the following cutscene runs at 30 frames a second.
- Per-frame task order (`GSgappCreate` priorities, ascending; **verified** from code): 0x00 and 0x01 (system), 0x0A
  `mainLogicTask` (the game threads: scripts and menus), 0x13 the disc error task, 0x7F the field, 0xFC a disc task,
  0xFD, 0xFE `mainRetraceLogic` (`menuDaemon` with the window animations, menu ticks and `fadeDaemon`), 0xFF drawing
  and the reset check.

## Disc data
The titles, presets, page grids and button labels are messages in the game text (the menu's own text bank); the
models come from the room's archive (`name_entry_menu`). The PC box screen has its own copy of the room
(`pcbox_name_entry_menu`, fsys-catalog.md).

## Names in the text: storage and insertion (renaming characters)

How every kind of character name reaches the screen. **Verified** from `msgctrl*.c`, `gs_msg.c`, `people.c`,
`fight.c`, `fight_action.c` and counts over the decoded US text, unless marked.

**How inserts work.** A message contains a control code; its handler returns either a **string** (insert mode 1, the
table word 0x58...) or a **message key** that is looked up and expanded (mode 2, 0x98...); inserts can nest three
deep. Most handlers just return a value a caller stored beforehand with `msgctrlSetValue(code, value)`: the slot
number **is** the control code.

| Who | How the text names them | Where the name lives | Notes |
|---|---|---|---|
| *Names in memory and in the save* (**verified** from data) | | names are **UTF-16 code units, big-endian** ("WES" = 00 57 00 45 00 53) in the player record and in each Pokémon's original-trainer name; the trainer-name setter and getter (`pokemonBiosSet/GetCatchTrainerNamePtr`) copy up to 11 units | |
| **Player (Wes)** | code **0x2B** (`msgctrlHero`), about 325 uses | the player save record's name field (0x2C bytes = 22 UTF-16 units; name entry allows 7) | read straight from the save every time: renaming the save renames every line |
| **Partner (Rui)** | code **0x2C** (`msgctrlHizuki`), about 230 uses (speaker labels included) | the player save record's partner-name field (0x56 bytes; entry allows 7) | same, from the save |
| "WES" / "RUI" literally | only as the **preset names** in the name-entry screen's text | message bank of the name-entry menu | no dialogue line spells either name out (**verified**, a search of all US text) |
| **NPCs speaking in the field** | most lines **start with the speaker's name written literally** ("WILLIE: ...", about 1,800 page starts, close to 390 distinct names, an approximate count); unnamed speakers start with **U+2031**, which the main font draws as a **speech-bubble icon** (about 1,500 pages) | in each message's text | renaming such an NPC means editing every line that names them |
| NPCs through the name table | code **0x59** (`msgctrlNpc`, a message key): `characterTalk` fills it from the speaker's **character-name record** (the NPC placement's name index), but only once that record's **"heard" flag** is set; until then it inserts message 0xFA3, the bubble icon | **character-name table** (`common_rel`, 607 records of 8 bytes: heard flag, name message; game-data-tables.md (not yet published)) | used by shared lines, e.g. Mt. Battle's generic trainer talk (`tool_fight`); one name message serves every line |
| **Trainers in battle** | codes **0x22** (trainer class), **0x23** (trainer name), 0x24-0x27 (losing line, the opposing names), **0x13** (the player's trainer name) | the trainer record's name message (+0x08) and its class's name message (trainers.md (not yet published)) | set per battle by `fight_action.c`; starting a battle also sets the opponent's **heard flag** (`fight.c`), so their name shows in later field lines that use 0x59 |
| Pokémon nicknames | the Pokémon codes (0x14-0x19 in battle, 0x32 / 0x33, 0x36 in menus) | the Pokémon record's nickname field (0x16 bytes = 11 units; entry allows 10) | |
| PC boxes | menu text | the box header (up to 8 characters, `pcboxSetPokemonBoxName` refuses longer) | default "BOX 1" .. "BOX 3" |

**The name buffer** (**verified** from the linked code):
- **Where the name is typed:** straight into `lbl_803A2068`, which holds 12 units with the mode right after them.
  Typing writes a terminator after each letter, so **11 letters is the most this buffer can take** without
  overwriting the mode.
- **The name field's frame** (its underscores and caret) exists only for maximum lengths 7, 8 and 10. Any other value
  hides the frame and the typed name.
- **So a longer limit needs** a bigger buffer and a name field drawn some other way. A table edit alone is not enough.

**Length limits.**
- Entry limits: player 7, partner 7, nickname 10, box 8 (the per-mode table above); the save fields hold more for the
  player and partner (22 and 43 units), exactly 10 for nicknames, 8 for boxes.
- **Dialogue lines are broken by hand** (line-break codes in each message). The field window has a fixed width, so a
  name longer than the designers allowed for (7) can push a line past the window's edge (**likely**; how the
  renderer handles an overlong line wasn't read). The battle window sizes itself to its message (type 4,
  message-windows.md).
- Trainer and NPC names are ordinary messages: no length field, only the window and line widths limit them.

**For a renaming mod.**
- Player and partner: change the save fields (raise the entry limit only together with a check of line widths).
- Named NPCs: either edit the literal names in their lines, or (cleaner, port-side) convert the literal
  "NAME:" prefixes to the 0x59 code with the character-name table, so one entry renames them everywhere.
- Trainers: edit the trainer's name message; the battle text follows automatically.

## Changing it
- **Longer names**: the maximum length is one byte per mode in the per-mode table, but the save keeps names in
  fixed-size fields (player record, Pokémon nickname): raising it beyond what those fields hold would overflow them.
- **Different presets**: edit the three message ids per mode, or the messages themselves (a text edit).
- **Other alphabets** (a translation): the page grids are plain messages, so new letters need only text and font
  edits, plus a pair list for R if the alphabet has cases or marks.

## Efficiency
- One generic screen with a mode table instead of four screens.
- The floating background (30 "ball" overlays in a fixed table; **verified**, `menuNameEntry.c`) is **simulated for
  600 ticks before the first frame** when the screen opens, so it appears already in motion instead of filling up
  from empty; afterwards it spawns at most one new ball per frame. A fixed-size table and a pre-run are a cheap way to
  get a "steady state" effect without saving any state.

## Region notes
- The R key's code is the **Japanese voiced-mark swap** (`exchangeDakuon`, two pair lists). The US build keeps the
  code and changes the data: the first pair list is empty and the second holds upper / lower case pairs (**verified**:
  the table holds 0 and message 0x2EF7), so the same key
  becomes the case toggle. (**verified** from the Japanese and European discs' name-entry banks): In the Japanese
  build the first list pairs each kana with its voiced form (ka / ga style) and the second with its semi-voiced form
  (ha / pa style), so R adds or removes the marks; the European builds keep the US arrangement, and German adds its
  umlaut pairs (Ä ä Ö ö Ü ü) to the case list.
- **The maximum lengths per region** (**verified**: read from each disc's own copy of the per-mode
  table, found by its title message keys): US and Europe 7 / 7 / 10 / 8 (player, partner, nickname, box); **Japan
  5 / 5 / 5 / 8**. The table sits at a different address in each build (US 0x80266DD8; the others found by pattern).
- **The letter pages per region** (**verified** from the banks):
  - **Japan:** three pages, chosen with X: katakana, hiragana and full-width Latin (upper and lower case, digits,
    Japanese punctuation, the gender signs), each four rows of three blocks; buttons labelled in kana; R adds the
    voiced marks.
  - **US and Europe:** pages of upper case and lower case Latin (three letter rows with digits, then a row of
    symbols); the button labels and the preset names are each language's own. **German** adds Ä Ö Ü (and ä ö ü) to
    the third row; French uses « » and German „ “ in the symbol row. **French, Spanish and Italian offer no accented
    letters** on this screen (no é, no ñ).
  - The US bank still carries the Japanese build's kana and full-width pages and its voiced-mark list under the same
    keys (left over; the US screen never shows them); the Japanese bank has no US-style Latin pages.

## Open questions
- Who opens mode 3 (box names): no C caller found; the PC box screen may use its own room copy.
- The exact slot the cursor jumps to when the name is full (likely DONE).
- What happens when the player finishes with an empty name in mode 0 (whether a default is substituted).
