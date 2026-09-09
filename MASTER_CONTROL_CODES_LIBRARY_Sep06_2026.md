# MASTER CONTROL-CODE LIBRARY — ALL DRAGON QUEST ENGINES
**Date:** 2026-09-06  **Author:** Omen Alpha (control-code library lane)
**Scope:** every DQ engine generation — Famicom (DW1/2/3/4-FC), Super Famicom (DQ1+2, DQ3, DQ6),
PlayStation HeartBeat (DQ7 → DQ4 PSX), Nintendo DS (DQ4). Built from verified-in-code extraction
(Endo's decoders, DW1 disassembly, Osteoclave, RadMage FORMAT.md, Wilkens tooling, the consumer
project's own on-disc census).
**Detailed evidence docs (read these for citations):**
- `snes/docs/CONTROL_CODES_SFC_ENGINES.md` — SFC, verified against `dq_analyzer/dq3decode.c` & `dq6decode.c`
- `snes/docs/CONTROL_CODES_PSX_HBD.md` — PSX HBD, RadMage MEASURED + Wilkens tool-mapped + on-disc counts
- `snes/docs/CONTROL_CODES_FC_AND_DS.md` — FC + DS, verified against `Bank01.asm`, `bank22.asm`, concreted's parser
- Machine-readable base: `translation/control_code_mapping.json` (43 PSX code families, on-disc counts/blocks)

---

## 1. THE MASTER CROSS-ENGINE MAP (function → code, per engine)

| Function | FC DW1 | FC DW2 (US) | FC DQ4 | SFC DQ3 | SFC DQ6 | SFC DQ1+2 | **PSX DQ4 (HBD)** | DS DQ4 |
|---|---|---|---|---|---|---|---|---|
| End of string | `$FF` (END2)/`$FC` (END1) | `[FF]`/`<END>` | Huffman `<END>` | `00AC`/`00AE` | `00AC`/`00AE` | `00FF` | **`0x0000`** | EOS |
| Line break | `$FD` (NEWL) | `[FD]`/`<NEWLINE>` | Huffman `<NEWLINE>` | `00AD` | `00AD` | `00F0`/`00FE` | **`7F01` (EXE/UI) · `7F02` (scene)** | `0x0A` |
| Wait for input (▼) | `$FB` (WAIT) | `[FB]` | Huffman `<PAUSE>` | `00AF` | `00AF` | `007A` | **`7F0A`** | — |
| Box/page advance | `$FC` (END1=clear) | — | — | — | — | — | **`7F0B`** (end-of-line) | — |
| Timed wait | `$FE` (pause) | — | — | `00DA` | — | — | — | — |
| Named-dialog opener | — | — | — | `00CD` (suppress ＊「) | `00D4` (suppress ＊「) | — | **`7F04`** (name decorator) | — |
| Hero/player name | `$F8` (NAME) | `[FC]`/`<NAME>` | Huffman `<NAME>` | `00C9` | `00C9` | `00F4` | **`7F1F`** | `%a…`/name tag |
| Party-head name | — | — | — | `00CA/00CB` | `00D2/00D3` | — | — | — |
| Fixed char names | — | — | — | `00C0/00C1/00CC` | `00CA-00D0` (Hassan…Rookiey), `00D1` (sister) | — | **`7F20–7F2F` (15 names)** + `7F31`(Psaro) + `7F32`(Rose) | name tags |
| Item/noun insert | `$F9` (ITEM) | `[FA]` | Huffman `<ITEM>` | `00B5/00C7` | `00B5/00C7` | — | **`7F16/7F18` (receive pair), `7F4B` (noun)** | item tag |
| Spell insert | `$FA` (SPEL) | — | Huffman `<SPELL>` | — | — | — | — | — |
| Number/gold var | `$F6` (AMNT)/`$F7` | — | Huffman `<AMOUNT>` | `00BB/00C5/00DB` | `00BB/00C5` | — | **`7F15` (gold)** | `%H/%M/%O/%L/%D` |
| String var | — | — | Huffman `<STRING>` | `00B2` | `00B2` | — | — | — |
| Bag | — | — | — | `00C3` | `00C3` | — | — | — |
| Plural suffix | — | — | — | `00C4` (たち) / `00DD` (がた) | `00C4` (たち) / `00DF` (がた) / `00E0` (みなさん) | — | — | `%X/%Y/%Z` inside `%H` |
| Tone/register switch | — | — | — | `00D1` ♂…wait: female | `00D9` female / `00DA` male / `00DB` monster / `00DC` silent | — | — | — |
| Tone (DQ3 order) | — | — | — | `00D1` female, `00D2` male, `00D3` monster, `00D4` silent | — | — | — | — |
| Window control | — | — | — | — | `00D5` | — | — | — |
| Custom/town/person | — | — | — | `00D8` name-string, `00D9` merchant, `00DC` husband/lady | — | — | **`7F30/7F33/7F34/7F4C` person, `7F42` town, `7F34` custom name** | `%aNNNNN` |
| Dictionary escape | `$6D+` (dict, FC-DQ1) | 5-bit stream, escapes `0x1C–0x1F` | `7Exx` (per-block, 1-based) | — | — | `00D0–00D2` (2-byte kanji) | **`7Exx`** (158 per-block dictionary refs; inline `FF xx` inside them = `7Fxx`) | — |
| 2-byte glyph escape | — | — | — | — | — | `00D0–00D2` lead+byte → 768-entry kanji table | — | UTF-8 |
| Combining marks | — | — | — | — | — | `0082` ゜/`0083` ゛ compose with next byte | — | — |
| Emphasis glyphs | — | — | — | — | — | `0088` *, `0089` 。, `008A` ,., `008B` 「, `008C` … | **`7F43/7F44/7F45` (emphasis, inferred)** | glyph rewrites |
| Newline reset for EXE UI | — | — | — | — | — | — | **`7F01`** (pen x=8, handler `0x80088A48`) | — |
| VM dialogue load | — | — | — | — | — | — | **`C0 21 A0 <offset> <dialogId>`** (+flag-eval form `C021A0 FFF0 <key>`) | script VM |

## 2. PSX HBD — THE ENGINE THIS PROJECT SHIPS ON (canonical table)

Verified-by legend: **RM** = RadMage FORMAT.md (hardware-measured), **MW** = Wilkens tool-mapped,
**MS** = consumer project (on-disc census / Sovereign build), counts = RM dictionary-expanded /
consumer `control_code_mapping.json`.

| Code | Name | Behavior | Count (RM/CP) | Verified |
|---|---|---|---|---|
| `0000` | END | terminator; real Huffman symbol | all strings / 17,000 | RM·MW·MS |
| `7F01` | newline (EXE/UI renderer) | x=8, y+=line height; hanging indent +16 if prefix latch | 144 EXE | RM |
| `7F02` | newline (scene renderer) | same semantics; **not** "newline+tab" (MW gloss corrected) | 166,134 / 32,894 | RM·MW·MS |
| `7F04` | name decorator | opens named dialog, suppresses engine ＊「 prefix; 20-cell width | 14,084 / 14,084 | RM·MW·MS |
| `7F05` | **enumeration/list-close marker** (RESOLVED Sep-06: closes the equip-name list; runtime splice point — hence "always last before END") | 664 | RT-trace |
| `7F0A` | cursor wait | wait-for-input; box terminator; clears prefix latch | 6,480 | RM·MS |
| `7F0B` | end of line | second box terminator; most-spread code (1,081 blocks) | 3,685 | RM·MS |
| `7F0C` | prologue-narration line end (title crawl, block 0069 only) | 6 | RT-trace |
| `7F11` | **enumerated name slot #1** (equip lists, RESOLVED) | 35,729 / 463 | RT-trace·RM |
| `7F12` | **enumerated name slot #2** | 70,488 / 587 | RT-trace·RM |
| `7F13` | **enumerated name slot #3** | 438 / 250 | RT-trace·RM |
| `7F14` | **enumerated name slot #4** | 166 | RT-trace·RM |
| `7F15` | received gold | `<7F24>は <7F15>Ｇを手に入れた！` | 106 | RM·MW |
| `7F16`/`7F18` | receive-message pair (member + item/quantity, dictionary-bound) | 2/3 | MW·RT |
| `7F17` | **item-name slot** ("the item being handled") | 193 | RT-trace·MW |
| `7F1A` | **wagon-hold member name** (Lucia in Ch5; equip/cure/recipient target) | 3 | RT-trace·MW |
| `7F1F` | player name | `どうした？ <7F1F>。` | 335 | RM·MW |
| `7F20` | ライアン Ragnar | name 1/15 | 1,139 | RM |
| `7F21` | アリーナ Alena | name 2/15 | 1,766 | RM |
| `7F22` | クリフト Kiryl | name 3/15 | 1,505 | RM |
| `7F23` | ブライ Borya | name 4/15 | 1,674 | RM |
| `7F24` | トルネコ Torneko | name 5/15; confirms 7F15 | 2,156 | RM |
| `7F25` | ミネア Meena | name 6/15 | 1,851 | RM |
| `7F26` | マーニャ Maya | name 7/15 | 1,660 | RM |
| `7F27` | — | **never occurs on disc** | 0 | RM (absence) |
| `7F28` | スコット Scott | name 8/15 | 105 | RM |
| `7F29` | アレクス Alex | name 9/15 | 6 | RM |
| `7F2A` | フレア Flora | name 10/15 | 53 | RM |
| `7F2B` | ホイミン Healie | name 11/15 | 189 | RM |
| `7F2C` | オーリン Orlin | name 12/15 | 84 | RM |
| `7F2D` | **wagon coachman** (dynamic NPC — Hoffman in Ch5; NOT one of the chosen; "not always Hoffman" explained) | 459 | RM·RT-trace |
| `7F2E` | パノン Panon | name 14/15 | 291 | RM |
| `7F2F` | ルーシア Lucia | name 15/15 (spoken-dialog fixed name; the *member slot* is `7F1A`) | 222 | RM·RT-trace |
| `7F30` | **Doran — pet dragon's name** (Lucia's dragon; roars グゴゴーン) | 34 | RT-trace |
| `7F31` | ピサロ Psaro | mostly デス{7f31} = Necrosaro | 514 | RM·RT-trace |
| `7F32` | ロザリー Rose | name | 265 | RM·RT-trace |
| `7F33` | **party-leader name** (addressed "you" of the moment; PSX of SFC `00CB`) | 6 | RT-trace·MW·RM |
| `7F34` | **acting party member / searcher** (search/read action actor) | 41 | RT-trace·MW·RM |
| `7F42` | town name | `%a00260` | 63 | MW·RM |
| `7F43` | **tone: loud/shout/announcement** (also dramatic narration boxes; PSX of SFC tone band) | 38 | RT-trace |
| `7F44` | **tone: soft/gentle** (children, Rose, feminine voices) | 8 | RT-trace |
| `7F45` | **tone: dark/menace** (villains, monsters, Psaro; double-repeat = pitch escalation) | 47 | RT-trace |
| `7F47` | **current item name as speaker/title** (equipment-block title box, `{7F04}{7F47}　{7F05}`) | 84 | RT-trace·RM |
| `7F4B` | noun (traded/deal item in the haggle line) — confirmed | 104 | MW·RM |
| `7F4C` | **protagonist/party representative** (the addressed "you-all": `{7F4C}たちに 神のご加護が…` — our church blessing line) | 25 | RT-trace·MW·RM |

**Structures:** `7Exx` = per-block 1-based dictionary reference (158 codes; block-local — `7E08`
means a different phrase in different blocks). Inline `FF xx` inside `7Exx` dictionary phrases
(raw SJIS) = the literal `7Fxx` code. `7Exx` are NOT "untranslated Japanese" and NOT FE-tags —
correcting the old `control_code_mapping.json` note. **SECOND BAND DISCOVERED (Sep-06):** `FExx`
variable/control codes live INSIDE the facility blocks' text streams (047B wagon, 047D church,
047F titles, 0483 memory-card) — leaf contents `0x7Exx` on disc, rendered as `0xFExx`; per-block
semantics (e.g. 0483 `FE01`="Adventure Log", FE0B/FE0D/FE10 = card/slot/log variables; 047D
FE28 = priest box-close; 047F per-title trailing tag). Full evidence:
`snes/docs/CONTROL_CODES_PSX_RESOLVED_Sep06_2026.md` §4.
**Never-occurring in-range codes (ARCHIVE population):** 7F03, 7F06–7F09, 7F0D–7F10, 7F19, 7F1B–7F1E, 7F27,
7F35–7F41, 7F46, 7F48–7F4A. **CORRECTED Sep-08 (cbgrime-ctrl-audit v1):** seven of these DO occur in the
EXE-resident blocks (048C: 7F06, 7F10; 048F: 7F19, 7F1B, 7F1C, 7F1E, 7F35) — semantics UNKNOWN, freeze tokens.
See §9 addendum.

## 3. SFC ENGINES (verified from Endo's decoded switches)

### DQ3 SFC (13-bit Huffman, MSB-first, 8 strings/pointer)
- Controls: `00AC/00AE` end · `00AD` break · `00AF` ▼wait · `00DA` timed wait · `00B0` wait ·
  `00CD` suppress ＊「 · `00B2` string · `00B5/00C7` item · `00BB/00C5/00DB` number ·
  `00C0/00C1/00CC` char name · `00C3` Bag · `00C4` たち · `00DD` がた · `00C9/00CA/00CB` hero/
  hero-or-leader/leader · `00D8` name-string · `00D9` merchant · `00DC` husband/lady ·
  `00D1/D2/D3/D4` tone female/male/monster/silent · silent-output: `00B4/00CE/00CF/00D0/00D6` ·
  unknown: `00AB/00B1/00B3/00B6-00BA/00BC-00BF/00C2/00C6/00C8/00D5/00D7`
- Addresses: pointer table `$C15331`, blob `$3CC258`, fixed strings `$3ECFB7`, tree `$159D3–$1697A`.

### DQ6 SFC (Huffman, LSB-first, polarity inverted vs DQ5)
- Controls: `00AC/00AE` end · `00AD` break · `00AF` ▼ · `00D4` suppress ＊「 (+`00DE` pair) ·
  `00D5` window control · `00B2` string · `00B3/00C0/00C1` char name · `00B5/00C7` item ·
  `00BB/00C5` number · `00C3` Bag · `00C4` たち · `00DF` がた · `00E0` みなさん ·
  **`00C9–00D0` = the 8 party names** (default/ Hassan/Millelucia/Barbara/Terry/Chamoro/Amos/
  Rookiey) · `00D1` sister · `00D2/00D3` hero/leader · `00D8` name-string ·
  `00D9/DA/DB/DC` tone female/male/monster/silent · silent: `00D6/00E1` · unknown:
  `00AB/00B1/00B4/00B6-00BA/00BC-00BF/00C2/00C6/00C8/00D7`
- Addresses: routine `$C02C92`, hook `$C029E6`, pointer table `$C15BB5`.
- **Correction:** the earlier brief's "furigana = 0xC3" is wrong — `00C3` = Bag in BOTH engines
  (Endo prints ふくろ); no furigana code found in any SFC source. "0xD9–0xDC = town names" is
  also wrong — DQ3 has merchant/name-string/husband-lady there; DQ6 has tone controls there.

### DQ1+2 SFC (NO Huffman — plain bytes)
- `00F0/00FE/00FF` newline/end variants · `00F4` name · `007A` ▼ (single-byte mode) ·
  `0078/0079` ?/! · `0088–008C` *。「「… · `0082/0083` ゜゛ (compose) · `00D0–00D2` 2-byte kanji
  escape (768-entry table).

## 4. FAMICOM ENGINES

### DW1 FC (verified from `dragon-warrior-disassembly/Bank01.asm`, `LB662–LB6AF`)
17 code set `$EF–$FF`: PLRL/PNTS/ENM2/DESC/AMTP/ENMY/AMNT/SPEL/ITEM/NAME/COPY/SUBEND/WAIT(`$FB`)/
END1(`$FC`)/NEWL(`$FD`)/NOP/END2(`$FF`); charset `$0A`=a…`$23`=z, `$24`=A…`$3D`=Z, space=`$5F`;
19 TextBlocks × 16 entries, block/entry nibble-encoded in the dialog byte. Plus the FC-DQ1
**dictionary system**: codes ≥`$6D` → pointer table (bank-3 `$F150`, `$FA`-separated word lists;
atwiki places the table at `$8858` — community value).

### DW2 FC (US, Osteoclave)
5-bit dictionary bitstream: dict `0x0B44B` (word-length nybbles + words), 48 pointers at `0x0B762`
→ 48×16 strings; escape indices `0x1C–0x1F` = sub-dictionaries (names/punctuation/common words/
phrases); tokens `[F2]–[FF]` (incl. `<SUN>/<STAR>/<MOON>`).

### DQ4 FC/US (two verified layers)
- **Dialogue = bit-level Huffman** (Osteoclave code table: `000`=e … `100010`=`<END>`; 86 blocks
  ×32 lines; block pointers `0x58961–0x58A10`; stream spliced banks 0–4 + `0x68010`/`0x6F79A`).
- **Byte layer (verified vs `bank22.asm:1786–1840`):** `$FD`=LINE, `$FE`=CTRL, `$FF`=END,
  `$F0–$FC` clamped; real DTE table at bank-22 `$B3A4` (menu text).
- Huffman tokens: `<END> <NEWLINE> <PAUSE> <NAME> <PROMPT> <AMOUNT> <ITEM> <SPELL> <STRING> <S>
  <DISAPPEAR> <NEWPARAGRAPH>`.
- **Fabrication rejected:** `TEXT_ENCODING.md`'s `$FE=[LINE]/$FD=[CLEAR]` contradicts bank22.asm.
- FC char table: kana `$01–$3E`, katakana `$40–$5F`, ▼=`$80`, ▶=`$81` (DQバイナリ改造@Wiki).
- 22-byte monster records with 10-bit packed stats/resistances.

## 5. DS DQ4 (Nitro)
- Container: `.mpt` (`MPT0` magic, 6-byte pointers, UTF-8, `0xFE` pad, `0x0A` newline).
- Tag grammar (from concreted's parser — de-facto spec): `@a/@b` · `@c0@–@c3@` ·
  **`%A/%B/%C` gender triples · `%H/%M/%O/%L/%D` conditionals with `%X/%Y/%Z` delimiters** ·
  `%0` · `%aNNNNN` · `%N<n>`. This is the only public DQ4 dialogue-branching reference in any
  version, and the authoritative model for the PSX `%A{ID}%X/%B{ID}%X/%Z` family.

## 6. CROSS-ENGINE CORRECTIONS & CONFLICTS LEDGER

1. **Same byte, different game** (all verified in Endo's code): `0xD4` (DQ3 tone-silent vs DQ6
   suppress ＊「), `0xD9` (DQ3 merchant vs DQ6 tone-female), `0xDA` (timed-wait vs tone-male),
   `0xDB` (number vs tone-monster), `0xC9–0xCC` (hero-family vs Hassan/Milly/Barbara), `0xB3`,
   `0xDD` (がた vs unknown), `0xCF/0xD0` (no-output vs Amos/Lizzie).
2. `7F02` ≠ "newline+tab" — that was Wilkens' gloss; RadMage measured a plain newline
   (range-tested with `7F01` at `0x800886D8`).
3. `0xC3` = Bag (both SFC engines), NOT furigana. No furigana code exists in any engine studied.
4. The PSX `7F20–7F2F` band is the 15 chapter party names with 5 names already re-mapped by the
   consumer project (`7F20` Ragnar vs RawMage's ライアン = same; consistent).
5. `7Exx` PSX dictionary refs are block-local; treat them as data, not as control codes.
6. The old `control_code_mapping.json` "note" claiming `7Exx` = untranslated JP is wrong
   (dictionary refs); and the FE-range claim there is wrong for PSX (FE exists only in DS/FC worlds).

## 7. UNKNOWN-CODE BACKLOG (updated Sep-06 after roundtrip context tracing)

- **RESOLVED (RT-trace, see `CONTROL_CODES_PSX_RESOLVED_Sep06_2026.md`):** 7F05, 7F11–7F14,
  7F47 (enumeration system), 7F17, 7F1A, 7F2D, 7F30, 7F33, 7F34, 7F43–7F45 (tone band), 7F4C.
- **PSX underdetermined (tiny counts, dictionary-bound):** `7F16/7F18` (receive pair), `7F0C`
  (prologue). Best probe: sentinel-string injection + DuckStation screenshot.
- **PSX new queue:** the `FExx` facility band (047B/047D/047F/0483 semantics per code).
- **SFC:** DQ3 `00AB/00B1/00B3/00B6-00BA/00BC-00BF/00C2/00C6/00C8/00D5/00D7`; DQ6 same band.
- **DS:** NFTR ruby/color tags still open (decomp doesn't cover the text renderer yet).

## 8. FILES

| File | Content |
|---|---|
| `snes/docs/CONTROL_CODES_PSX_HBD.md` | full PSX table with FORMAT.md/autoTranslator citations |
| `snes/docs/CONTROL_CODES_SFC_ENGINES.md` | DQ3/DQ6/DQ1+2 with dq3decode.c/dq6decode.c switch quotes |
| `snes/docs/CONTROL_CODES_FC_AND_DS.md` | DW1/2/4 FC + DQ4 DS with asm/parser citations |
| `translation/control_code_mapping.json` | on-disc PSX census (43 families) |
| this file | cross-engine master map + corrections + backlog |
| copies in `study/` and `study/cybergrime/` | same master file for pipeline lanes |

*One library, five engines, every code with an owner and a confidence level. The unknowns are
now a finite, testable list.*

---

## 9. ADDENDUM — EXE-RESIDENT CONTROL BAND (Sep 08 2026, cbgrime-ctrl-audit v1)

The Sep-06 library was built from the **archive population** (1,528 HBD text sub-blocks).
An exhaustive leaf census of the **EXE-resident** Font-1 blocks (pristine `SLPM_869.16`,
`DQ4Schema` decode) extends the PSX band:

- **51 unique control codes now measured on disc** (43 archive + 8 EXE).
- **Never-occurring list falsified for EXE blocks.** Occurring with UNKNOWN semantics:
  - `7F06` ×3 and `7F10` ×11 — block **048C** (field-menu strings)
  - `7F19` ×10, `7F1B` ×1, `7F1C` ×13, `7F1E` ×15, `7F35` ×2 — block **048F** (priest/save menu)
  - Prime candidates for the menu submenu/dialect controls; **freeze tokens** (pass through
    verbatim, position-locked) until RT-traced.
  - **EXONERATED Sep-08 (second directive):** all seven verified passing through verbatim in
    the active `eef2ccb2` 048F encode AND in-RAM (dump 174); the Cynthia-speech hard freeze
    on `75fe6aff` is **caused by an HBD archive-mount sector write clobbering the resident
    overlay module at RAM 0x11F00** (see FREEZE_MECHANISM_174_175_CLOBBER_Sep08_2026.md) —
    unrelated to these codes. Status: **known freeze tokens, pass-through verified;
    semantics still UNKNOWN (RT-trace pending).**
- **Fullwidth-82xx romaji band is the engine's native Latin for Font 1**: 048C 3,833 glyphs
  (+355 fw-81xx punct), 048F 2,742 (+418). The fullwidth-leaf encode path is canonical.
- **048F carries 910 DICT (7Exx) refs pristine**; the menu-framed restoration scope covered
  309 (~34%). Dialogue blocks 0021/0023 carry 18,173/10,306 dict refs (translated inline — by design).
- **Corpus token gaps (dialogue-side, 18 blocks):** 7F4B dropped (0168/0169/0224/022A/02BA/02F7),
  7F0B dropped (02D7/03EE/0422/0423/0424/042F/043E/043F), 7F13 dropped (0021/0023),
  7F12 dropped + 7F0A inserted (0022), 7F20 dropped (015B). G8 does not currently cover these.
- 048D (192 B) layout differs (txe=0) — census gap, own parse path required.

Evidence: `study/CONTROL_CODE_DEEP_AUDIT_Sep08_2026.md` · `cybergrime/forensics/ctrl_census.json`
· `cybergrime/forensics/ctrl_audit.py` · `snes/docs/CONTROL_CODES_PSX_HBD.md` (EXE band section).
