# Pokémon data: species table and individual records

**Applies to:** Colosseum (**verified**, USA GC6E01) · XD (**likely** in structure: the same Gen 3 internal species order; XD's own tables not checked)

Status: **partial** (2026-10-14). Port feature goals: save editor / Pokémon storage and modding tools
(species, moves, Shadow Pokémon). Built from `pokemonGetStatus` (0x8012640C), the master accessor:
`pokemonGetStatus(obj, id, selector, index)`.

## What it does

Everything about a Pokémon is read through **one numbered accessor**. The selector picks the field;
selectors 0x01-0x6C read the **species data table** for species `id`, 0x6E-0xBF read an individual
**Pokémon record** `obj` (the 0x138-byte record stored in the party, PC box and daycare), and
0xC2 onward read battle-time state (Shadow data, in-battle copies, the Pokémon on the field).
`pokemonSetStatus` (0x801254B4, same selectors) writes. The getter names below are the real
symbol names from `symbols.txt`, which carry the original Japanese-English terms (Basis = base stat,
Effort = EV, Rnd = IV, Sinka = evolution, Waza = move, Tokusei = ability, Zokusei = type,
Dark = Shadow, Dp = heart gauge (battle-shadow.md), Tamago = egg, Fusei = illegitimate).

## Selector table

| Selector | Getter | Reads from |
|---|---|---|
| 0x001 | `pokemonDataBiosGetName` | species table (`id`) |
| 0x003 | `pokemonDataBiosGetBasisMaxHp` | species table (`id`) |
| 0x004 | `pokemonDataBiosGetBasisPhyAtk` | species table (`id`) |
| 0x005 | `pokemonDataBiosGetBasisPhyDef` | species table (`id`) |
| 0x006 | `pokemonDataBiosGetBasisSpeAtk` | species table (`id`) |
| 0x007 | `pokemonDataBiosGetBasisSpeDef` | species table (`id`) |
| 0x008 | `pokemonDataBiosGetBasisNimbleness` | species table (`id`) |
| 0x00a | `pokemonDataBiosGetGiveMaxHpEffort` | species table (`id`) |
| 0x00b | `pokemonDataBiosGetGivePhyAtkEffort` | species table (`id`) |
| 0x00c | `pokemonDataBiosGetGivePhyDefEffort` | species table (`id`) |
| 0x00d | `pokemonDataBiosGetGiveSpeAtkEffort` | species table (`id`) |
| 0x00e | `pokemonDataBiosGetGiveSpeDefEffort` | species table (`id`) |
| 0x00f | `pokemonDataBiosGetGiveNimblenessEffort` | species table (`id`) |
| 0x010 | `pokemonDataBiosGetGiveExp` | species table (`id`) |
| 0x011 | `pokemonDataBiosGetGrowDataId` | species table (`id`) |
| 0x012 | `pokemonDataBiosGetGet` | species table (`id`) |
| 0x013 | `pokemonDataBiosGetSexRatio` | species table (`id`) |
| 0x014 | `pokemonDataBiosGetInitFriend` | species table (`id`) |
| 0x015 | `pokemonDataBiosGetItemDataId` | species table (`id`) |
| 0x016 | `pokemonDataBiosGetZokuseiDataId` | species table (`id`) |
| 0x017 | `pokemonDataBiosGetTokuseiDataId` | species table (`id`) |
| 0x019 | `pokemonDataBiosGetSinkaKind` | species table (`id`) |
| 0x01a | `pokemonDataBiosGetSinkaBuff` | species table (`id`) |
| 0x01b | `pokemonDataBiosGetSinkaPokemonDataId` | species table (`id`) |
| 0x01d | `pokemonDataBiosGetGetWazaLevel` | species table (`id`) |
| 0x01e | `pokemonDataBiosGetGetWazaDataId` | species table (`id`) |
| 0x020-0x059 | `pokemonDataBiosGetWazaMcn(obj, n)` | species table: TM/HM compatibility, one selector per machine |
| 0x05a | `pokemonDataBiosGetPokebodyId` | species table (`id`) |
| 0x05b | `pokemonDataBiosGetStatusFaceMenuSpriteId` | species table (`id`) |
| 0x05c | `pokemonDataBiosGetColor` | species table (`id`) |
| 0x05d | `pokemonDataBiosGetTypeName` | species table (`id`) |
| 0x05e | `pokemonDataBiosGetHeight` | species table (`id`) |
| 0x05f | `pokemonDataBiosGetWeight` | species table (`id`) |
| 0x060 | `pokemonDataBiosGetDoc` | species table (`id`) |
| 0x061 | `pokemonDataBiosGetVoice` | species table (`id`) |
| 0x062 | `pokemonDataBiosGetMitaFlag` | species table (`id`) |
| 0x063 | `pokemonDataBiosGetTukamaetaFlag` | species table (`id`) |
| 0x064 | `pokemonDataBiosGetNumZukan` | species table (`id`) |
| 0x065 | `pokemonDataBiosGetNumPokemon` | species table (`id`) |
| 0x066 | `pokemonDataBiosGetPkxDataId` | species table (`id`) |
| 0x068 | `pokemonDataBiosGetKowaza` | species table (`id`) |
| 0x069 | `fn_8011E048` | other |
| 0x06a | `fn_8011E030` | other |
| 0x06b | `fn_8011E018` | other |
| 0x06c | `fn_8011E000` | other |
| 0x06e | `pokemonBiosGetPokemonDataId` | Pokémon record (`obj`) |
| 0x06f | `pokemonBiosGetRnd` | Pokémon record (`obj`) |
| 0x070 | `pokemonBiosGetAttest` | Pokémon record (`obj`) |
| 0x071 | `pokemonBiosGetCatchFloorId` | Pokémon record (`obj`) |
| 0x072 | `pokemonBiosGetCatchLevel` | Pokémon record (`obj`) |
| 0x073 | `pokemonBiosGetCatchBallId` | Pokémon record (`obj`) |
| 0x074 | `pokemonBiosGetCatchTrainerSex` | Pokémon record (`obj`) |
| 0x075 | `pokemonBiosGetCatchTrainerRnd` | Pokémon record (`obj`) |
| 0x076 | `pokemonBiosGetCatchTrainerNamePtr` | Pokémon record (`obj`) |
| 0x077 | `pokemonBiosGetNicknamePtr` | Pokémon record (`obj`) |
| 0x078 | `pokemonBiosGetNicknameOrgPtr` | Pokémon record (`obj`) |
| 0x079 | `pokemonBiosGetExp` | Pokémon record (`obj`) |
| 0x07a | `pokemonBiosGetLevel` | Pokémon record (`obj`) |
| 0x07b | (computed: whether the record's species is 0, an empty slot) | computed |
| 0x07c | `pokemonBiosGetJoutaiPtr` (0x8011F474) | other |
| 0x07d | `pokemonBiosGetConditionAmari` | Pokémon record (`obj`) |
| 0x07f | `pokemonBiosGetPokemonWazaDataId` | Pokémon record (`obj`) |
| 0x080 | `pokemonBiosGetPokemonWazaPp` | Pokémon record (`obj`) |
| 0x081 | `pokemonBiosGetPokemonWazaPpCount` | Pokémon record (`obj`) |
| 0x082 | `pokemonBiosGetItemDataId` | Pokémon record (`obj`) |
| 0x083 | `pokemonBiosGetHp` | Pokémon record (`obj`) |
| 0x087 | `pokemonBiosGetMaxHp` | Pokémon record (`obj`) |
| 0x088 | `pokemonBiosGetPhyAtk` | Pokémon record (`obj`) |
| 0x089 | `pokemonBiosGetPhyDef` | Pokémon record (`obj`) |
| 0x08a | `pokemonBiosGetSpeAtk` | Pokémon record (`obj`) |
| 0x08b | `pokemonBiosGetSpeDef` | Pokémon record (`obj`) |
| 0x08c | `pokemonBiosGetNimbleness` | Pokémon record (`obj`) |
| 0x08d | `pokemonBiosGetMaxHpEffort` | Pokémon record (`obj`) |
| 0x08e | `pokemonBiosGetPhyAtkEffort` | Pokémon record (`obj`) |
| 0x08f | `pokemonBiosGetPhyDefEffort` | Pokémon record (`obj`) |
| 0x090 | `pokemonBiosGetSpeAtkEffort` | Pokémon record (`obj`) |
| 0x091 | `pokemonBiosGetSpeDefEffort` | Pokémon record (`obj`) |
| 0x092 | `pokemonBiosGetNimblenessEffort` | Pokémon record (`obj`) |
| 0x093 | `pokemonBiosGetMaxHpRnd` | Pokémon record (`obj`) |
| 0x094 | `pokemonBiosGetPhyAtkRnd` | Pokémon record (`obj`) |
| 0x095 | `pokemonBiosGetPhyDefRnd` | Pokémon record (`obj`) |
| 0x096 | `pokemonBiosGetSpeAtkRnd` | Pokémon record (`obj`) |
| 0x097 | `pokemonBiosGetSpeDefRnd` | Pokémon record (`obj`) |
| 0x098 | `pokemonBiosGetNimblenessRnd` | Pokémon record (`obj`) |
| 0x099 | `pokemonBiosGetFriend` | Pokémon record (`obj`) |
| 0x09c | `pokemonBiosGetStyle` | Pokémon record (`obj`) |
| 0x09d | `pokemonBiosGetBeautiful` | Pokémon record (`obj`) |
| 0x09e | `pokemonBiosGetCute` | Pokémon record (`obj`) |
| 0x09f | `pokemonBiosGetClever` | Pokémon record (`obj`) |
| 0x0a0 | `pokemonBiosGetStrong` | Pokémon record (`obj`) |
| 0x0a1 | `pokemonBiosGetFur` | Pokémon record (`obj`) |
| 0x0a3 | `pokemonBiosGetChampRibbon` | Pokémon record (`obj`) |
| 0x0a4 | `pokemonBiosGetWinningRibbon` | Pokémon record (`obj`) |
| 0x0a5 | `pokemonBiosGetVictoryRibbon` | Pokémon record (`obj`) |
| 0x0a6 | `pokemonBiosGetBromideRibbon` | Pokémon record (`obj`) |
| 0x0a7 | `pokemonBiosGetGanbaRibbon` | Pokémon record (`obj`) |
| 0x0a8 | `pokemonBiosGetMarineRibbon` | Pokémon record (`obj`) |
| 0x0a9 | `pokemonBiosGetLandRibbon` | Pokémon record (`obj`) |
| 0x0aa | `pokemonBiosGetSkyRibbon` | Pokémon record (`obj`) |
| 0x0ab | `pokemonBiosGetCountryRibbon` | Pokémon record (`obj`) |
| 0x0ac | `pokemonBiosGetNationalRibbon` | Pokémon record (`obj`) |
| 0x0ad | `pokemonBiosGetEarthRibbon` | Pokémon record (`obj`) |
| 0x0ae | `pokemonBiosGetWorldRibbon` | Pokémon record (`obj`) |
| 0x0af | `pokemonBiosGetAmariRibbon` | Pokémon record (`obj`) |
| 0x0b0 | `pokemonBiosGetStyleMedal` | Pokémon record (`obj`) |
| 0x0b1 | `pokemonBiosGetBeautifulMedal` | Pokémon record (`obj`) |
| 0x0b2 | `pokemonBiosGetCuteMedal` | Pokémon record (`obj`) |
| 0x0b3 | `pokemonBiosGetCleverMedal` | Pokémon record (`obj`) |
| 0x0b4 | `pokemonBiosGetStrongMedal` | Pokémon record (`obj`) |
| 0x0b5 | `pokemonBiosGetPokerus` | Pokémon record (`obj`) |
| 0x0b6 | `pokemonBiosGetTamagoFlag` | Pokémon record (`obj`) |
| 0x0b7 | `pokemonBiosGetTokuseiFlag` | Pokémon record (`obj`) |
| 0x0b8 | `pokemonBiosGetFuseiFlag` | Pokémon record (`obj`) |
| 0x0b9 | `pokemonBiosGetFlagAmari` | Pokémon record (`obj`) |
| 0x0ba | (computed: gender, from the personality value and the species' ratio) | computed |
| 0x0bb | `pokemonBiosGetPcboxMark` | Pokémon record (`obj`) |
| 0x0bc | `pokemonBiosGetMailId` | Pokémon record (`obj`) |
| 0x0bd | `pokemonBiosGetPara1Amari` | Pokémon record (`obj`) |
| 0x0be | `pokemonBiosGetAmari` | Pokémon record (`obj`) |
| 0x0bf | (computed: nature = personality value mod 25) | computed |
| 0x0c0 | (computed: level from EXP and the growth rate) | computed |
| 0x0c1 | (computed: the shiny check, from the personality value and trainer id) | computed |
| 0x0c2 | `pokemonBiosGetFightTrainerPokemonDataId` | Pokémon record (`obj`) |
| 0x0c3 | `pokemonBiosGetDarkpokemonDataId` | Pokémon record (`obj`) |
| 0x0c4 | `pokemonBiosGetInitDp` | Pokémon record (`obj`) |
| 0x0c5 | `pokemonBiosGetDp` | Pokémon record (`obj`) |
| 0x0c6 | `pokemonBiosGetPoolExp` | Pokémon record (`obj`) |
| 0x0c7 | `pokemonBiosGetPoolFriend` | Pokémon record (`obj`) |
| 0x0c8 | `pokemonBiosGetDarkFlag` | Pokémon record (`obj`) |
| 0x0c9 | `pokemonBiosGetDarkJoutaiPtr` (0x8011EDC4) | other |
| 0x0cb | `fightPokemonBiosGetMotoPokemonPtr` | battle Pokémon |
| 0x0cc | `fightPokemonBiosGetPokemonBuffPtr` | battle Pokémon |
| 0x0cd | `fightPokemonBiosGetFightJoutaiPtr` | battle Pokémon |
| 0x0ce | `fightPokemonBiosGetEntryId` | battle Pokémon |
| 0x0cf | `fightPokemonBiosGetCatchEntryFlag` | battle Pokémon |
| 0x0d0 | `fightPokemonBiosGetLevelUpFlag` | battle Pokémon |
| 0x0d1 | `fightPokemonBiosGetDarkOutFlag` | battle Pokémon |
| 0x0d2 | `fightPokemonBiosGetHokakuFlag` | battle Pokémon |
| 0x0d5 | `fightOutPokemonBiosGetMotoFightPokemonPtr` | Pokémon on the field (battle) |
| 0x0d6 | `fightOutPokemonBiosGetFightPokemonPtr` | Pokémon on the field (battle) |
| 0x0d7 | `fightOutPokemonBiosGetFightPokemonHensinBuffPtr` | Pokémon on the field (battle) |
| 0x0d8 | `fightOutPokemonBiosGetFightoutJoutaiPtr` | Pokémon on the field (battle) |
| 0x0d9 | `fightOutPokemonBiosGetFightWazaPtr` | Pokémon on the field (battle) |
| 0x0da | `fightOutPokemonGetUseWazaDataId` | Pokémon on the field (battle) |
| 0x0db | `fightOutPokemonGetMotoWazaDataId` | Pokémon on the field (battle) |
| 0x0dc | (computed from the field record) | computed |
| 0x0e5 | `fightOutPokemonBiosGetFightItemPtr` | Pokémon on the field (battle) |
| 0x0e6 | `fightOutPokemonBiosGetAbicntPhyAtk` | Pokémon on the field (battle) |
| 0x0e7 | `fightOutPokemonBiosGetAbicntPhyDef` | Pokémon on the field (battle) |
| 0x0e8 | `fightOutPokemonBiosGetAbicntSpeAtk` | Pokémon on the field (battle) |
| 0x0e9 | `fightOutPokemonBiosGetAbicntSpeDef` | Pokémon on the field (battle) |
| 0x0ea | `fightOutPokemonBiosGetAbicntNimbleness` | Pokémon on the field (battle) |
| 0x0eb | `fightOutPokemonBiosGetAbicntAverage` | Pokémon on the field (battle) |
| 0x0ec | `fightOutPokemonBiosGetAbicntAvoid` | Pokémon on the field (battle) |
| 0x0ed | `fightOutPokemonBiosGetFightoutTurnCount` | Pokémon on the field (battle) |
| 0x0ee | `fightOutPokemonBiosGetSequencePtr` | Pokémon on the field (battle) |
| 0x0ef | `fightOutPokemonBiosGetSketchWazaDataId` | Pokémon on the field (battle) |
| 0x0f0 | `fightOutPokemonBiosGetLastSelectWazaDataId` | Pokémon on the field (battle) |
| 0x0f1 | `fightOutPokemonBiosGetLastUseWazaDataId` | Pokémon on the field (battle) |
| 0x0f2 | `fightOutPokemonBiosGetLastReceiveWazaTargetDataId` | Pokémon on the field (battle) |
| 0x0f3 | `fightOutPokemonBiosGetHitWazaDataId` | Pokémon on the field (battle) |
| 0x0f4 | `fightOutPokemonBiosGetHitWazaZokuseiDataId` | Pokémon on the field (battle) |
| 0x0f5 | `fightOutPokemonBiosGetGamanDamageValue` | Pokémon on the field (battle) |
| 0x0f6 | `fightOutPokemonBiosGetGamanDamageTargetId` | Pokémon on the field (battle) |
| 0x0f7 | `fightOutPokemonBiosGetOumuWazaDataId` | Pokémon on the field (battle) |
| 0x0f8 | `fightOutPokemonBiosGetKeepFightWazaPtr` | Pokémon on the field (battle) |
| 0x0f9 | `fightOutPokemonBiosGetNamakeFlag` | Pokémon on the field (battle) |
| 0x0fa | `fightOutPokemonBiosGetUsedItemDataId` | Pokémon on the field (battle) |
| 0x0fb | `fightOutPokemonBiosGetStockItemDataId` | Pokémon on the field (battle) |
| 0x0fc | `fightOutPokemonBiosGetSuccessCnt` | Pokémon on the field (battle) |
| 0x0fd | `fightOutPokemonBiosGetMeetEnemyFightPokemonEntryId` | Pokémon on the field (battle) |
| 0x0fe | `fightOutPokemonBiosGetFightActionBuffPtr` | Pokémon on the field (battle) |
| 0x0ff | `fightOutPokemonBiosGetZokuseiDataId` | Pokémon on the field (battle) |
| 0x100 | `fightOutPokemonBiosGetTokuseiDataId` | Pokémon on the field (battle) |
| 0x101 | `fightOutPokemonBiosGetWazaMenuCurPtr` | Pokémon on the field (battle) |
| 0x102 | `fightOutPokemonBiosGetDamageAtkValue` | Pokémon on the field (battle) |
| 0x103 | `fightOutPokemonBiosGetDamageAtkTargetId` | Pokémon on the field (battle) |
| 0x104 | `fightOutPokemonBiosGetDamageSpeValue` | Pokémon on the field (battle) |
| 0x105 | `fightOutPokemonBiosGetDamageSpeTargetId` | Pokémon on the field (battle) |
| 0x106 | `fightOutPokemonBiosGetMahiNoAttackFlag` | Pokémon on the field (battle) |
| 0x107 | `fightOutPokemonBiosGetKonranMyselfAttackFlag` | Pokémon on the field (battle) |
| 0x108 | `fightOutPokemonBiosGetOutWazaKoukanaiFlag` | Pokémon on the field (battle) |
| 0x109 | `fightOutPokemonBiosGetTameWazaFlag` | Pokémon on the field (battle) |
| 0x10a | `fightOutPokemonBiosGetItemNigeruFlag` | Pokémon on the field (battle) |
| 0x10b | `fightOutPokemonBiosGetHuuinNoAttackFlag` | Pokémon on the field (battle) |
| 0x10c | `fightOutPokemonBiosGetMeroMeroNoAttackFlag` | Pokémon on the field (battle) |
| 0x10d | `fightOutPokemonBiosGetKanashibariNoAttackFlag` | Pokémon on the field (battle) |
| 0x10e | `fightOutPokemonBiosGetChouhatsuNoAttackFlag` | Pokémon on the field (battle) |
| 0x10f | `fightOutPokemonBiosGetIchamonNoAttackFlag` | Pokémon on the field (battle) |
| 0x110 | `fightOutPokemonBiosGetHirumuNoAttackFlag` | Pokémon on the field (battle) |
| 0x111 | `fightOutPokemonBiosGetPassPpdecFlag` | Pokémon on the field (battle) |
| 0x112 | `fightOutPokemonBiosGetFightActionFlag` | Pokémon on the field (battle) |
| 0x113 | `fightOutPokemonBiosGetDoClearbodyFlag` | Pokémon on the field (battle) |
| 0x114 | `fightOutPokemonBiosGetReceivesWazaHiraishinFlag` | Pokémon on the field (battle) |
| 0x115 | `fightOutPokemonBiosGetVanishoffFlag` | Pokémon on the field (battle) |
| 0x116 | `fightOutPokemonBiosGetDoIkakuFlag` | Pokémon on the field (battle) |
| 0x117 | `fightOutPokemonBiosGetDoTraceFlag` | Pokémon on the field (battle) |
| 0x118 | `fightOutPokemonBiosGetNoPressureFlag` | Pokémon on the field (battle) |
| 0x119 | `fightOutPokemonBiosGetIrekaetaFlag` | Pokémon on the field (battle) |
| 0x11a | `fightOutPokemonBiosGetItemKoraetaFlag` | Pokémon on the field (battle) |
| 0x11b | `fightOutPokemonBiosGetKaigaraDamageValue` | Pokémon on the field (battle) |
| 0x11c | `fightOutPokemonBiosGetMyselfDamageAtkValue` | Pokémon on the field (battle) |
| 0x11d | `fightOutPokemonBiosGetMyselfDamageAtkTargetId` | Pokémon on the field (battle) |
| 0x11e | `fightOutPokemonBiosGetMyselfDamageSpeValue` | Pokémon on the field (battle) |
| 0x11f | `fightOutPokemonBiosGetMyselfDamageSpeTargetId` | Pokémon on the field (battle) |
| 0x120 | `fightOutPokemonBiosGetKizetuFlag` | Pokémon on the field (battle) |
| 0x121 | `fightOutPokemonBiosGetIrekaeTargetEntryId` | Pokémon on the field (battle) |
| 0x122 | `fightOutPokemonBiosGetFightOutPokemonEnemyPtr` | Pokémon on the field (battle) |
| 0x123 | `fightOutPokemonCheckFightOut` | Pokémon on the field (battle) |

## Data structures

- **Species table** (`pokemonDataBios*`): per species: name, 6 base stats, 6 EV yields, EXP yield,
  growth rate, catch rate (`Get`), gender ratio, base friendship, held items, two types, abilities,
  evolutions (kind / parameter / target), level-up learnset (level + move), TM compatibility,
  egg moves (`Kowaza`), body id, menu sprite id, colour, category name, height, weight, Pokédex
  text, cry, seen/caught flags, national and regional dex numbers, model id (`PkxDataId`).
- **Pokémon record** (0x138 bytes, `pokemonBios*`): species, personality value (`Rnd`),
  attestation (`Attest`), catch location / level / ball / OT gender / OT id / OT name, nickname
  (and original), EXP, level, condition, 4 moves with PP and PP-up counts, held item, HP and the
  six stats, EVs, IVs, friendship, contest stats and ribbons, medals, Pokérus, egg flag, ability
  slot, the "fusei" flag, PC box mark, mail id; **Shadow data**: Shadow (Dark) species id, initial
  and current heart gauge (`InitDp` / `Dp`), stored EXP and friendship that are released on purify
  (`PoolExp`, `PoolFriend`), and the Shadow flag (`DarkFlag`).
- **Record layout: `PokemonBios` in `include/game/pokemon_data.h`** (confirmed by the matched
  `pokemonBios*` getters). Key offsets: +0x000 species (u16), +0x004 personality (u32), +0x008
  attest[4], +0x00C catch location (u16), +0x00E catch level, +0x00F ball, +0x010 OT gender,
  +0x014 OT id (u32), +0x018 OT name (0x16), +0x02E nickname (0x16), +0x044 original nickname
  (0x18), +0x05C EXP, +0x060 level, +0x064 condition (0x10), +0x078 four moves (`PokemonWaza`:
  id, PP, PP-ups), +0x088 held item, +0x08A HP, +0x08C stats, +0x098 EVs, +0x0A4 IVs (status ids 0x93-0x98 read them in
  the order HP, Attack, Defense, Sp. Atk, Sp. Def, Speed: verified) (three
  `PokemonStatBlock`s), +0x0B0 friendship, +0x0B2-0x0BC contest stats/medals/sheen, +0x0BD-0x0C9
  ribbons, +0x0CA Pokérus, +0x0CB egg, +0x0CC ability slot, +0x0CD fusei, +0x0CF PC mark,
  +0x0D0 mail id, +0x0D6 trainer-party data id, **+0x0D8 Shadow species data id, +0x0DC heart gauge
  (`dp`, s32), +0x0E0 stored EXP, +0x0E4 stored friendship**, +0x0E8 attest2 (0x10),
  +0x0F8 event flags.
- **Size discrepancy (open):** the struct covers 0xFC bytes, but the save's party slots are 0x138
  bytes each (`HeroSaveData`, `save-data.md`): 0x3C bytes per slot are unaccounted for. In a real save those
  bytes hold run-time addresses (the console's 0x80… RAM addresses) and differ between two saves of the same play,
  so they are working state written along with the record, not stored data (**likely**; save-data.md, "Comparing two
  saves"). Battle Mode's saved Pokémon records (save section 14) show the same thing: three
  plays that started from different saves ended with byte-identical records, including the same RAM addresses, so
  the leftover values come from a fixed code path rather than from earlier play (**verified** by comparing the three
  saves). 

## The data tables in memory

| Table | Getter | Base / count | Entry | Layout |
|---|---|---|---|---|
| Species | `pokemonDataBiosGetPtr(id)` (0x8011E778) | pointer `lbl_80478F94`, count `lbl_80478F90[0]` | **0x11C** bytes | `PokemonData` in `include/game/pokemon_data.h` |
| Moves | `wazaDataBiosGetPtr(id)` (0x8011CA34) | pointer `lbl_80478DFC`, count `lbl_80478DF8[0]` | **0x38** bytes | not yet named; an out-of-range id returns entry 0 instead of NULL |
| Items | `itemDataBiosGetPtr(id)` (0x801440A0) | id -> slot through the static map `lbl_803681E8`, then a count check (`lbl_80478BD8`, `lbl_80478BB0`) | ? | not yet named |

`PokemonData` (species entry, 0x11C): growth rate, catch rate, gender ratio, EXP yield, base
friendship, height, weight, cry, national and regional dex numbers, **message ids** for the name,
category and Pokédex text, seen/caught flag ids, **model id (`pkxDataId`)**, 2 types, 2 abilities,
TM/HM compatibility bytes (0x3A), 2 held items, 8 egg moves, base stats, EV yields, **5 evolutions**,
**20 level-up moves** (`{level, move}`), and 2 face/body entries (colour, menu sprite, body id).

**The species order** (**verified** from the US table): 413 entries in **Gen 3's internal order**
(not the National Pokédex order): 0 is empty, 1-251 are Bulbasaur to Celebi, **252-276 are 25 empty entries** (the
same gap Gen 3 has), 277-411 are the Hoenn species, and 412 is Gen 3's egg slot. The empty entries are the natural first
place for new species without widening any table (**likely**: the other per-species tables must accept those indices
too).

**Reading species numbers: the internal order is not the National Dex** (**verified** against the
US disc's name table). Every species number the game stores or passes around (party and battle records, trainer
teams, the Shadow Pokémon table, script arguments, memory dumps) is the **internal** number, the index into this
table. For 1-251 it equals the National Pokédex number; from 252 on it doesn't. Example: internal **290 is Wurmple and
291 is Silcoon**, while National Dex 290 and 291 are Nincada and Ninjask, so reading a log's 291 as a National Dex
number names the wrong Pokémon.
Checked in play (**verified**, 2026-10-19): a battle logged as internal 323 and 330 showed Barboach and Carvanha on
screen, as this table names them (National Dex 323 and 330 would be Numel and Trapinch).

**Every species has three numbers** (**verified** on the US disc's 413 entries; field names from
the decompiled `PokemonData`):

| Number | Where | What it is | Examples (Treecko / Wurmple / Barboach / Deoxys / Bulbasaur) |
|---|---|---|---|
| **Internal index** | the entry's position in the table | what the game stores and passes everywhere (records, teams, scripts) | 277 / 290 / 323 / 410 / 1 |
| **National Dex number** | +0x12 (`numPokemon`, u16) | the standard national number; 386 distinct values, 0 for the empty entries and the egg | 252 / 265 / 339 / 386 / 1 |
| **Hoenn-first number** | +0x14 (`numZukan`, u16) | Hoenn's regional order first (1-202, Treecko to Deoxys), then every other species from 203 (Bulbasaur) to 386 (Celebi) | 1 / 14 / 127 / 202 / 203 |

So converting is a table lookup, not arithmetic: internal to National Dex through +0x12, never by an offset. Which
of the three the game's own screens show (the strategy memo's "No.") is **not checked yet** (likely the
National number; a look at one Hoenn entry settles it).
Rules for tools and notes: name a species from this table (its name message id), never from a National Dex list;
when a National Dex number is needed, take it from the entry's own national-number field; and write the internal
number with its name ("291 Silcoon") so a reader can't mix the two up. The same order is Gen 3's own (the GBA games
use it too), so it carries over to XD unchanged (**likely**).

**Move entry (0x38 bytes, `wazaDataBios*`):**

| Offset | Field | Offset | Field |
|---|---|---|---|
| +0x00 | s8 priority (`Pri`) | +0x0E | flag: Assist can call it (`Nekonote`) |
| +0x01 | u8 PP | +0x0F | flag: Sleep Talk can call it (`Negoto`) |
| +0x02 | u8 type (`ZokuseiDataId`) | +0x10 | flag: sound move (`Bouon`) |
| +0x03 | u8 target range (`RangeId`) | +0x11 | flag: Pressure applies |
| +0x04 | u8 accuracy (`Avg`) | +0x12 | flag: HM (`Hiden`) |
| +0x05 | u8 secondary-effect chance (`AddFightKoukaAvg`) | +0x13 | flag: "risk" |
| +0x06 | flag: contact (`Dageki`) | +0x16 | s16 base power (`Iryoku`) |
| +0x07 | flag: blocked by Protect (`Mamoru`) | +0x18 | u16, 0 in all 358 moves |
| +0x08 | flag: Magic Coat (`Majikku`) | +0x1A | u16 `SeqId`: the **move-effect id** that picks the battle script (Gen 3's effects 0-213; verified against the scripts) |
| +0x09 | flag: Snatch (`Yokodori`) | +0x1C | u16 index into the move-function tables (AI value / hit / damage, WZX type / variation callbacks) |
| +0x0A | flag: Mirror Move (`Oumugaesi`) | +0x20 | u32 name message id |
| +0x0B | flag: King's Rock (`Oujanosirusi`) | +0x24 / +0x28 | u32 battle message ids ("used X" connector, attack text) |
| +0x0C | flag: Metronome (`Yubiwohuru`) | +0x2C | u32 description message id (`Doc`) |
| +0x0D | flag: Mimic (`Monomane`) | +0x30 | u32 WZX effect data id; +0x34..: type list (`TypeId`, up to 3) |

**Item entry (0x28 bytes, `ItemDataBios` in `src/game/people/people_data.c`):** +0x00 kind (4 =
TM/HM), +0x01 key item, +0x02 usable, +0x04 effect parameter, +0x06 price, +0x08 Poké Coupon price,
+0x0A held-item effect id (`itemSoubiDataId`), +0x0C battle-use effect id, +0x10 name message id,
+0x14 description message id, +0x18 buffer/extra id, +0x1C field-use function id, +0x20 battle-use
function id, +0x24 friendship change per use (3 brackets, s8). Items are looked up through a
**static id -> slot map** (`lbl_803681E8`), so item ids and table rows are not the same. Berries are
item ids 0x85-0xAF (`itemDataBiosGetKinomiNo`). **TM table** `lbl_80368018[machineNo].wazaId`
gives each TM/HM's move (count `lbl_80478BD0`).

All three bases are **pointers filled at runtime** (the tables come from disc data, not the
executable). They live in the REL module `common_rel` (`common.fsys`); its prolog fills the
pointers and counts (413 species, 358 moves, 97 Shadow definitions in the USA game). **Moves:** 358 entries = an empty entry 0, the 354 Gen 3 moves (ids 1..354), an empty
entry 355, **Shadow Rush** (356) and a "????" placeholder (357): 355 real moves (2026-10-15, decoded move table). Full list of
the module's 47 tables: game-data-tables.md (not yet published).

## Models and portraits

- Each species has a **model id** (`pkxDataId`, selector 0x66); models are PKX files, floor
  resource type 15 (`floor-resources.md` (not yet published)).
- **Portrait (face) cache** (`gs_range_80109C88.c`: `fn_8010BD6C` registers a face, `menuPokemonModelIsLoaded` (0x8010A210)
  tests whether a cached face fits a Pokémon): a cached face is reused only if it matches the model
  id, a **Shadow variant** code (0x87 or 0x25 when the Pokémon is Shadow, depending on status
  `pokemonIsJoutaiDataId(pokemon, 0x3E)` at 0x80121ADC; 0 otherwise), the **personality value** (selector 0x6F: decides
  shininess and patterns) and selector 0xC1 (meaning open). So portraits are rendered per individual,
  not per species: a model/avatar swap must also invalidate this cache.
- **Forms (verified from the disc's archive index, 2026-10-15):** species whose forms look different have one model
  archive per form: Castform has four (`pkx_powalen` plus `_f`, `_i`, `_w`), Unown one per letter (`pkx_unknown_e` ... `_z`, `_ex`, `_qu`). **Deoxys has only one model**: archive
  `pkx_deoxys` (archive id 0x0386, one entry `deoxys`, entry id 0x070A1E00) plus the shiny `pkx_rare_deoxys` (0x077E,
  entry id 0x128B1E00), and one species-table entry, so there are no Attack / Defense / Speed Forme models or stats in
  Colosseum (likely shown as Normal Forme even when traded in). The entry id is the u32 at the start of
  the archive's file entry: u16 id, then the file type x 2 (PKX = 0x0F -> 0x1E), then 0 (disc-files.md).
- **Animations per model (verified from data, 2026-10-15):** a Pokémon battle model's scene has one model set
  with a list of 4 to 16 joint animations (8 is the most common, 148 models; Dustox and Shedinja have 4, Pikachu 15,
  Blastoise 16); an animation's length is its end frame. The PKX's **slot table** (16 or 17 slot records: idle, hit,
  faint, ...) points into this list, so several slots can share one animation (models-animation.md (not yet published),
  "Animation slots").
- **Deoxys in detail (researched 2026-10-15; see also gba-link.md "Stats of imported Pokémon"):**
  (verified) The Mew / Deoxys event-flag rule (`menuCBRule_CheckPokemonEventFlag`,
  0x80075FEC) also gates **trading** on both sides (trading.md, the trade selection menu and state 8), not only
  Battle Mode.
  - *Disc (verified):* the one model `deoxys` holds a single scene with one model set (one skeleton of 112 joints,
    8 animations), so no Forme is packed inside it either; the shiny copy has the same layout. The species table has
    one Deoxys entry (index 410) with the **Normal Forme** base stats 50 / 150 / 50 / 150 / 50 / 150 (HP, Atk, Def,
    SpA, SpD, Speed). No text names a Forme.
  - *XD (likely, StarsMmd, XG author, 2026-10-16):* XG did **not** add Defense Forme: StarsMmd changed XD's own
    assembly that chooses which form is used, mapping it to Defense instead of Normal. So XD likely has **form-selection
    code** and probably the other Formes' model data, unlike Colosseum (verified: none, below). A modding tool's
    export of XD's model table has one Deoxys row (archive id 0x386, file identifier 0x6681E00) and no Forme rows,
    while Castform's forms have rows. **To check when we have XD's code:** find the form-selection routine
    (what it reads, where it is called) and the Deoxys Forme models on XD's disc (inside the one Deoxys archive, or
    elsewhere).
  - *Code (verified, byte-matched functions):* there is no form logic: no model or stat switch by form. The stat
    formula (`pokemonSetLevelBasisStatus`, 0x8012795C) has one species special case, Shedinja (0x12F: max HP 1), and
    always uses the species table, i.e. Normal Forme stats. The only Deoxys-specific code is a battle-mode rule
    (`menuRuleCheckPokemon` (0x80076398) check 0): **Mew (0x97) and Deoxys (0x19A) may enter only if their event flag is set** (the Gen 3
    "fateful encounter" bit, bit 31 of the ribbon word, carried over by the GBA converter). Searching the main
    program's assembly for compares against 0x19A / 410 finds only menu sprite ids; switch jump tables were not
    searched (likely complete, not proven). *Rechecked 2026-10-16 (verified):* the decompiled source has no other
    switch or compare on Deoxys's species number (the four `case 0x19A` hits in menu code are copies of the battle-mode
    rule; the GBA-screen ones are on-screen element ids); the battle's Pokemon-model table has one Deoxys model that no
    other entry shares; and the model loader's **variant override table** (`lbl_80374F20`, 35 species, used by
    `fn_801DE190` when its variant argument is set, which the battle sets from "is shiny") has one Deoxys entry, its
    shiny model. Colosseum picks a different model per species only for shininess, Unown's letter (its own table) and
    Castform's weather forms (their own species-table rows).
  - *The "pkx_deoxys_atk" file table (2026-10-16, checked for a table shared on Discord):* the table maps species 408
    Latios, 409 Jirachi, 410 Deoxys -> `pkx_deoxys_atk.fsys` (file id 0x386; model inside still the Normal Forme), 411
    Chimecho (`chirean`), 412 Egg. **On the Colosseum (USA) disc there is no `pkx_deoxys_atk`** (*verified*: all 1,852
    archives and every entry inside them searched; the string "deoxys_atk", and any "atk"-suffixed name, appears
    nowhere, neither in file names, `GSfsys.toc`, main.dol nor common_rel). Colosseum's Deoxys is `pkx_deoxys.fsys`
    (archive id **0x0386**, one entry `deoxys`, entry id 0x070A1E00) plus the shiny `pkx_rare_deoxys.fsys` (0x077E,
    0x128B1E00); "deoxys" occurs only in those two archives and the table of contents. So the **archive id 0x386 is the
    same in both games**, and the species numbers 408-412 are the shared Gen 3 internal order, so neither tells the
    games apart; the **name does: the table is from XD, not Colosseum** (*likely*; consistent with XD having
    form-selection code, StarsMmd above). What "atk" means there is open until XD's code is read. In Colosseum's code nothing refers to an Attack Forme (no form logic at all, below).
  - *A Deoxys traded in from FireRed / LeafGreen / Emerald (verified from code, not tried in Dolphin):* it keeps the
    stats its GBA game computed (the importer copies the six stats) and shows the Normal model. The first time it
    gains EXP in story mode (or gets a Rare Candy, vitamin or evolution-style recalculation), Colosseum recomputes
    its stats from its own table, so they become Normal Forme stats. In battle mode the legality check rejects it:
    every Forme has a base stat above Normal's (Attack Forme Atk 180, Defense Forme Def 160, Speed Forme Speed 180),
    so its stored stat exceeds the recomputed one.

### Items as 3D models (2026-10-17)

Almost every item exists only as data and text: no other item has a 3D model among the disc's object models (type-2
entries), searched by name (**likely**: a model with an unhelpful name could still be missed). The exceptions:
- **The twelve Poké Balls** each have a field model (**verified**, the disc's archive tables): `ball_monster` (Poké
  Ball), `ball_super`, `ball_hyper`, `ball_master`, `ball_safarl` (Safari), `ball_net`, `ball_dive`, `ball_nfst`
  (Nest), `ball_repeat`, `ball_timer`, `ball_gorgeus` (Luxury) and `ball_premiere`. All twelve sit together in the
  same 12 room archives: every Pokémon Center (`M1_pc_1F`, `M3_pc_1F`, `D2_pc_1F`), the Colosseum entrance lobbies
  (`M2_enter_1F`, `M2_enter_1F_2`, `M4_enter_1F`, `M4_enter_B1`), two Realgam Tower rooms (`D4_tower_2F_2`,
  `D4_dome_5`) and three other rooms (`M2_building_1F`, `M2_cave_1F_1`, `M3_houseD_1F`). These are the places where the
  party is healed, so the models are **likely** the balls shown on the healing machine (which ball appears would
  follow each party Pokémon's ball).
- **In battle**, each ball type has its own move-effect sequences (`wzx_<ball>_in`, `_out`, `_open`, `_downin`;
  **verified** names): the ball thrown, opening and the Pokémon coming out or going back. Whether those sequences carry
  their own ball model or reuse the field one is **likely** the former (move effects embed their models, wzx.md),
  not checked per file.
- **Pickups in the field** are generic, not per item (**verified**, treasure.md): a treasure box (`t_box_00`), a
  ground sparkle (`kirari`) and a book (`bbook`, the Ein Files), all in `field_common`.
- **Not models:** the bag and the PC show items as text and 2D pictures (**likely**; the bag's item pictures were not
  traced to their textures), and key items that appear in cutscenes, such as the Snag Machine on the player's arm,
  are part of a character model rather than item models.

## Shadow Pokémon

- **Becoming Shadow:** `pokemonSetDarkPokemonStatus(pokemon, shadowId)` (0x8011FCA4) clears the
  Shadow fields (`pokemonInitDarkPokemon`: Shadow id 0, gauge -100, stored EXP/friendship 0, Shadow
  flag cleared), then stores `shadowId` (selector 0xC3) and sets the heart gauge (0xC5) to a
  constant times the per-Shadow gauge size from save section 15 (`darkPokemonSaveGet4A6(shadowId)`, 0x801EEEB8).
- **Heart-gauge "level"** (`pokemonGetDarkPokemonLevel`, 0x8011F77C): 7 means not Shadow (or not
  flagged, selector 0xC2); otherwise the current gauge (0xC5) relative to the initial gauge (0xC4)
  gives the number of filled bars, 0 = ready to purify. Stored EXP (0xC6) and friendship (0xC7)
  are held back while Shadow and released on purification.
- **Per-Shadow state lives in the save** (section 15, `save-data.md`), one 0x18-byte row
  per Shadow id. A static definitions table (`BattleRangeDef`, `lbl_80478F6C`, ids 1..0x60) gives
  each Shadow entry a type byte and story-flag ids (`flag0C`, `flag0E`, `flag14`, read and set with
  `std081_flagCheckTrue` / `_flagSet`). (**verified**, shadow-system.md "Missed Shadow Pokémon
  and second chances"): +0x12 is "snagged", +0x14 "purified" and +0x16 "met"; what `flag0C` and `flag0E` mean is not
  settled.
  `lbl_80478F6C` is a **pointer** to the table (the disc loads it with `lwz`; the live unit
  `battle_range_exact_801EEC74` matches); entries are **0x38 bytes**. (An older copy of this code in
  `battle_range_801ED640.c` indexes it like an array; that copy is not compiled.)
- **Which Shadow entries the story uses (2026-10-15, from the trainer-Pokémon table):** 108 trainer-Pokémon entries
  carry a Shadow id (byte +0x03), but they use only **48 distinct ids (1..67), each always the same species**: the 48
  snaggable Shadow Pokémon. The extra entries are later re-encounters of the same Shadow Pokémon (each re-encounter is
  its own trainer-Pokémon entry pointing at the same Shadow id). The other **49 of the 97 definition slots are never
  referenced** by a trainer (unused or leftover). For testing and modding,
  "Shadow Pokémon" means the 48 used ids.
- **Port hazard (original-game behavior):** `fn_801EEE44` (Shadow entry type) sets its pointer to
  NULL for an invalid id (0 or > 0x60) and then reads the type byte anyway, i.e. from address 0.
  Harmless on GameCube (low memory is readable), a crash on PC: the port needs a guard here.

### Purification: how the heart gauge goes down (2026-10-15)

One function changes the gauge for every kind of event: `pokemonAddDpFormPokemonDpFilterId(pokemon, item, filter)`
(0x8011F910). **Verified from the code** (formula and clamping):

- The **filter id** names the event. A small table (`pokemonDpFilterDataBios*`) gives each filter a signed base amount.
- The Pokémon's **nature** decides how strongly each event counts: the nature record has one field per filter
  (`pokemonSeikakuDataBiosGetRelive*`: Fightout, Walk, Call, Sodateya, Nadenade), each naming a row of a rate table
  that holds a multiplier and a divisor (`Kake` / `Waru`). Change = base × multiplier / divisor; a divisor of 0 means
  "no change".
- The gauge is kept as an integer of 100 × the working value, and never goes below 0.

| Filter | Event (from the getter names and callers) | Where it is applied |
|---|---|---|
| 0 | sent into battle ("Fightout") | battle code (fight_range_exact_8022DCB8), guarded by the battle Pokémon's "DarkOut" flag (selector 0xD1), so likely once per battle (likely) |
| 1 | walking ("Walk") | field step handler: each party Shadow Pokémon has a step counter in its save row; at 256 steps the change applies and the counter resets (verified) |
| 2 | the **Call** command | battle Call command, when the call brings a player's Shadow Pokémon out of Hyper Mode (likely) |
| 3 | day care ("Sodateya") | same step handler, for the Pokémon in the day care, same 256-step counter (verified) |
| 4 | scent items ("Nadenade") | only for items of kind 6; the base is multiplied by the item's strength value (likely: the Joy / Excite / Vivid Scents) |
| 5 | empties the gauge (base = −1 × current gauge) | no caller found yet (**likely** unused: none in `main.dol` or the room modules built so far, battle-shadow.md) |

**The Call command's success chance** comes from the nature and the gauge "level" (`pokemonGetCallRate` (0x8011F634): the nature record and
`pokemonGetDarkPokemonLevel`, bars left), rolled against a random number 0-99. A Pokémon that is asleep can be woken
by Call instead (half the chance for a Shadow Pokémon; always for others) (likely, from `fightActionCallMain`, 0x80212D6C).

**Port note:** the battle code passed the Pokémon to the chance function through a leftover register; the PC
version now passes it explicitly.

### Hyper Mode (2026-10-15; read from the battle code, labels per claim)

Hyper Mode is battle condition **0x3E** on the battle Pokémon. One function handles it (0x8022EDEC), `_fightCheckDarkPokemonAppear(pokemon, mode)`:

- **Entering (mode 0)**, step 14 of the move sequence (`_wsHitCheckJoutaiSub` (0x8022F2F8)), before a move: only for a Shadow Pokémon that
  is not already in Hyper Mode and not asleep, and only when the move is **Shadow Rush** (move 356). A roll 0-99
  against a chance from the nature and the heart-gauge level (`pokemonGetHyperModeRate` (0x8011F6D8), the same kind of nature-table lookup as
  the Call chance) sets condition 0x3E, plays message 0x771B and redirects the move sequence (likely: the turn's
  action changes). **Likely** (read from the C, not yet watched in a game).
- **While in Hyper Mode (mode 1)**, from `_fightSeqTurnEndJoutaiSub` (0x80230568): shows message 0x771C, the per-turn "... is in HYPER MODE!"
  notice (**verified** from the text bank, 2026-10-20).
- **Leaving by itself (mode 2)**, same move step when already in Hyper Mode: a **1-in-256** roll clears Hyper Mode
  (and sleep and condition 0x17), message 0x771E (**likely**).
- **Leaving by Call**: the Call command (above) clears it with its own chance, and that call also lowers the heart
  gauge (filter 2).
- Whether Hyper Mode can happen at all is a **battle-kind rule** (`fightKindDataBiosGetDarkpokemonHypermodeFlag`,
  read through `fightFloorGetStatus`; `battle-engine.md`).

**What a Pokémon in Hyper Mode does** (`_wsHitCheckSub` (0x8022A6C8), fight_range_8022A504.c; was an open
question). Before each of its moves (unless the move is Shadow Rush), the game rolls once, 0-99, and walks seven bands
whose widths come from the **nature** record (`pokemonSeikakuDataBiosGetPtr`, read with seven getters
`fn_8011CCE4` ... `fn_8011CC54`); the first band the roll falls in decides what happens (structure **verified** from
the code; each action named from the battle message it shows, **likely**):

| Band (getter) | What happens (message) |
|---|---|
| 1 (`fn_8011CCE4`) | uses a move of its own choosing, without orders (0x770C) |
| 2 (`fn_8011CCCC`) | ignores orders and attacks the opposing trainer (0x770D) |
| 3 (`fn_8011CCB4`) | ignores orders and attacks its own trainer (0x770E) |
| 4 (`fn_8011CC9C`) | hurts itself (0x770F) |
| 5 (`fn_8011CC84`) | does nothing, too worked up to act (0x7710; also the fallback) |
| 6 (`fn_8011CC6C`) | tries to use its held item without permission and fails (0x7711 / 0x7712: no item) |
| 7 (`fn_8011CC54`) | tries to return to its Poké Ball (0x7713) or fails to (0x7714) |

So each nature has its own Hyper Mode personality: the seven rates are data, not code. The Call command's messages
fit the same picture: a sleeping Pokémon wakes (0x7706), a Hyper Mode Pokémon comes to its senses (0x7707), or the call
fails (0x7708 / 0x7709). Mode 1's message 0x771C is the "is in Hyper Mode" notice and 0x771E the recovery message
(verified from the text bank).

**For modders:** Hyper Mode behaviour per nature is in the nature table (seven rate bytes), so a mod can make a nature
calmer or wilder without code changes.

**Port note:** the entry check also passed the Pokémon to `pokemonGetHyperModeRate` (0x8011F6D8) through a leftover register; it is on the
leftover-call list.

**For modders:** purification speed per nature and per event is data (the filter table and the nature rate table),
so a mod can make walking, battling or scents count more or less without code changes.

## Flow

A Pokémon record is made by `pokemonCreate` (species, level, personality value fitted to its rules: rng.md), then
stored in the party, a PC box or the daycare. Everything reads and writes it through the selector accessor above.
Its six stats are stored, but recalculated from the species data at the moments listed in gba-link.md ("Stats of
imported Pokémon"): experience gains, level-ups, Rare Candy and vitamins, evolution. In battle, a copy of the record
is used and written back at the end.

(**verified** from code) - **Receiving a Pokémon** (gifts, starters, snags) goes through `heroCatchPokemon(hero, pokemon, location, ball,
  allowBox)`: it sets the catch fields (0x71 location, 0x72 met level = the current level, 0x73 ball, 0x74 trainer
  gender, 0x75 trainer id, 0x76 trainer name, all from the player record), then adds the Pokémon to the party, or to the
  PC box if allowed and the party is full. The starters pass location 0xFE and ball 4 (Poké Ball); the developers'
  debug add passes 0 and 4; the event Pikachu sets a fixed trainer id (0x7991) and name instead.
- **Teaching a move** (`pokemonWazaCreate(slot, move)`) clears the slot, sets the move and sets its PP to the move's
  maximum with no PP Ups. The max-PP read addresses the slot as index + 4, and the slot reader maps an index of 4 or
  more back to index - 4 after a Shadow-move check.

## Disc data

- **Species, moves, items and the related tables** are in `common_rel` (game-data-tables.md (not yet published), "The data tables in
  memory" above).
- **Portraits and body pictures**: `poke_face.fsys` and `poke_body.fsys` (textures.md); **models**: one PKX archive per
  species (models-animation.md (not yet published)); **cries**: the MusyX cry groups (music.md (not yet published)).
- **Individual Pokémon** are save data, not disc data (save-data.md).

## Efficiency

Not measured. One switch-based accessor serves every field, so a read is a table lookup plus a switch; nothing about
a Pokémon is computed per frame except in battle.

## Changing it

For a save editor or modding tool, this table is the field list; selectors are stable ids the
game itself uses, so tools can mirror them. New fields need both `pokemonGetStatus` and
`pokemonSetStatus` cases.

### Mod surface
- **Data it reads:** the species table (413 records), the move table, the item table, the Pokémon record in the save,
  growth tables, portraits and body pictures.
- **Hooks it would need:** species and move tables merged from mods (appended rows, IDs mapped per install), new
  record fields for mods (kept outside the original save format, so the record itself stays unchanged), and a
  National Dex view for tools.
- **Fixed-size limits:** 413 species records (25 empty), 5 evolution slots per species, 4 moves per Pokémon, the
  record's field widths (8-bit abilities, 16-bit species).
- **Must stay vanilla in GameCube-accurate mode:** every record and the Pokémon creation code, including its random
  draws.

## Open questions

- The 0x3C bytes after `PokemonBios` in a 0x138-byte party slot; the species table's location and
  stride in memory / on disc.
- What `Attest` and `Para1Amari` / `Amari` ("remainder") fields are.

## Record facts from the decoded tables (2026-10-16)

- **Shadow Pokémon definition record:** +0x02 species, +0x04 the first trainer that owns it (verified on all 48 ids the
  story uses); +0x08 is **not** the level.
- **Nature record:** +0x05..+0x09 are the stat-rate ids (verified against the standard nature chart for all 25
  natures); +0x00..+0x04 are the purification rows (per `PokemonSeikakuData`).
- Trainer and trainer-Pokémon layouts: see trainers.md (not yet published).
