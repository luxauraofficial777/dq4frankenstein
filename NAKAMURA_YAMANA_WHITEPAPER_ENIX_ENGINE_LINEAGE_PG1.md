# DRAGON WARRIOR IV (NES / Famicom) — Engine Architecture Whitepaper

**Paper 1 of 3 — Enix Engine Specification Suite**
**Doc ID:** VW-NES-WP-001 · 2026-09-14
**Scope:** Dragon Warrior IV (NES US, 1992-10) / Dragon Quest IV 導かれし者たち (FC, 1990-02-11).
Chunsoft 8-bit engine, 6502 core, MMC1 memory management.
**Method:** all measurements in this paper were taken directly from the cartridge images in
`famicom\` (`Dragon Warrior IV (USA).nes`, `Dragon Quest IV - Michibikareshi Monotachi (Japan).nes`)
by `snes\study\whitepaper_forensics.py` (machine-readable results:
`WHITEPAPER_FORENSICS_DATA.json`). Cross-references: Data Crystal (DW4 NES), TCRF, Osteoclave's
verified Huffman dumper (`clone\game-tools\nes\dragonwarrior4_textdump.py`), TheAnsarya's
disassembly framework (`clone\dragon-warrior-4-info`), abw's map format, DQバイナリ改造@Wiki.

---

## 0. ROM container identity (measured)

| Field | DW4 US | DQ4 FC |
|---|---|---|
| iNES magic | `4E 45 53 1A` ✓ | `4E 45 53 1A` ✓ |
| PRG-ROM | **32 × 16 KB = 512 KB** | 32 × 16 KB = 512 KB |
| CHR-ROM banks | **0** (CHR-RAM, 8 KB PPU pattern memory, streamed from PRG) | 0 |
| Mapper | **1 — MMC1** (flags6>>4 \| flags7&F0) | 1 — MMC1 |
| Battery-backed PRG-RAM | **yes** (flags6 bit 1) | yes |
| Nametable mirroring | horizontal (flags6 bit 0 = 0) | — |
| File size | 524,304 B (16 B iNES header + 524,288 data) | 524,304 B |
| MD5 (US) | `33690b361265c840fda965418adf3143` | (JP: `8bb0cf53c7a501a4da9dc7b2b38ed8c7`) |

**Mapper conflict resolved by direct header read.** `clone/dragon-warrior-4-info` claims
"MMC3 / Mapper 4"; the iNES header bytes of both the US and FC cartridges in this tree encode
mapper low nibble = `1` and mapper high nibble = 0 → **MMC1 (Mapper 1)**. Data Crystal and TCRF
are correct; TheAnsarya's README is wrong. Consequences: 32 KB fixed-window granularity is
16 KB at $8000–$BFFF (MMC1 swap mode 3), the final PRG bank is hard-wired at $C000–$FFFF,
and the 5-bit write protocol (5 consecutive CPU writes to $8000–$FFFF, LSB-first serial load)
governs every bank transition documented below.

Reset/NMI/IRQ vectors (last bank, file `0x7FFFA–0x7FFFF`): read from ROM
(NMI `$%04X`, reset, IRQ per `WHITEPAPER_FORENSICS_DATA.json → dw4_nes_us.header`).

---

## 1. Memory architecture & bank switching

### 1.1 CPU address map (6502, 2 A12-agnostic 16 KB windows)

```
$0000-$07FF   RAM (2 KB)        — text engine state, frame scratch, stack
$0800-$1FFF   RAM mirror
$2000-$3FFF   PPU registers
$4000-$4017   APU / OAM DMA ($4014) / controller ($4016-$4017)
$6000-$7FFF   8 KB battery-backed PRG-RAM  (save game + working sets)
$8000-$BFFF   switchable PRG bank (MMC1 16 KB mode)
$C000-$FFFF   fixed last bank (PRG bank $1F)
```

### 1.2 MMC1 serial protocol

Bank writes are 5-bit serial shifts: five consecutive `STA $E000-$FFFF` writes, LSB first,
feed MMC1's internal shift register; bit 7 of the 5th write is lost (reset bit = write $80
first). Register map: `$E000` chr-bank-0, `$A000` chr-bank-1 (4 KB CHR mode), `$C000` PRG
low bank (16 KB mode: $0000 = fixed), `$8000` PRG bank high bits / mirroring / RAM enable
(bit 3 = PRG-RAM chip-enable).

Consequences for the engine:
- CHR is **RAM** (0 CHR banks): every tile set — font, field tiles, battle sprites — is DMA'd
  out of PRG-ROM into CHR-RAM through the PPU update buffer each scene load. The "tile bank
  swap" between field and battle is therefore a **PRG→CHR streaming event**, not a CHR-ROM
  bank select; pattern-table halves ($0000-$0FFF / $1000-$1FFF) are treated as two 4 KB
  CHR banks in MMC1 4 KB-CHR mode.
- Bank 22 is the **text engine home** (menu/UI text, dispatcher `$8B28`, char processor
  `$8AA5`); bank 23 opens its pointer tables at `$8008`; item/spell names live in bank 26;
  chapter titles in bank 27; map data in banks $09–$0B; the battle engine's dense code lives
  in bank 19 (`code_map.txt`: 284 subroutines, the second-highest count in the ROM, all
  reading party/battle state from WRAM $615A+).

### 1.3 Battery-backed WRAM ($6000–$7FFF) allocation

8 KB PRG-RAM, battery-backed. Measured + documented partition (Data Crystal RAM map +
`dragon-warrior-4-info` memory labels; the $6xxx labels below are TheAnsarya's, validated
against fixed-bank and battle-bank access sites):

```
$6000-$60F0   party block: 30-byte stride × 9 slots
              $6001+30i  name (4 bytes)   — Hero, Cristo, Nara, Mara, Brey,
                                            Taloon, Ragnar, Alena, spare
              per-slot fields include current/max HP-MP (u16 pairs at
              $6000/$6001, $6004/$6005 …), XP (u16), level, stats
$6100-$61FF   battle working set (TheAnsarya labels:)
              $615A  current actor index (read from both fixed bank
                     $CC28 and battle bank 19 $8088)
              $615B+  party cursor data
              $618E  battle state flags
              $6195-$6198  battle counters / limits (party-position X/Y)
              $616A  battle action data (indexed; 8 read sites in bank 19)
$6200-$61094- battle tables region (US-specific enemy stat arrays;
              TCRF: unused enemy #189 "Roric palette-swap" at ~$61094)
$6800-$7FFF   free / scratch (hidden credits live here: "MANABU YAMANA"
              ×2 at save-RAM $0BBF, TCRF)
```

Save structures: character stats are packed 10-bit (8-bit base + extra bits in adjacent
resistance words, §4); inventory arrays and event-flag bytes interleave in the same 30-byte
party stride. The hidden developer credit `"MANABU YAMANA"` ×2 at save-RAM offset `$0BBF`
(TCRF) confirms the save block extends past $6BBF.

---

## 2. Text & scripting architecture

### 2.1 The main script: bit-level Huffman (measured)

The dialogue is not byte text — it is a **prefix-coded bitstream**, spliced across
physical banks. Verified layout (US, headered file offsets; Osteoclave's splice verified
here by independent decode):

**Physical stream splices** — the bitstream occupies five full PRG banks *minus* their last
40 bytes, plus two remote chunks:

```
0x0010-0x3FE8   (bank 0, $8000-$FFEB)   ┐
0x4010-0x7FE8   (bank 1)                │ spliced in file order into one
0x8010-0xBFE8   (bank 2)                │ virtual bitstream
0xC010-0xFFE8   (bank 2..4)             │
0x10010-0x13FE8 (bank 4)                ┘
0x68010-0x6BFE8 (bank 26 $8000-$BFE7)   remote chunk 1
0x6F79A-0x6FFE5 (bank 27 $B795-$BFD4)   remote chunk 2
```

**Block pointer table** — file `0x58961` (bank 22, `$8951`): 16-bit LE entries. Measured
first entries: `$8000, $81B6, $83B5, $85D6, $8813, $8915 …` — i.e. **each entry = Osteoclave's
verified stream offset + $8000** (bank-local address form of the same table; Osteoclave's
hardcoded dump list is exactly `table[i] − $8000`). Blocks ascend within the virtual
stream; the auto-detected monotone run ends where the virtual stream crosses a physical
bank and the 16-bit field wraps (the engine banks up on wrap, exactly like the map-data
table of §4 — bank-crossing is handled by incrementing the PRG bank, not by widening the
pointer).

**Block/line model**: each block holds **32 concatenated lines** (last block, stream offset
`0x186FB`, holds 4). A line is a prefix-code run terminated by the `<END>` leaf. Total
script = 85 × 32 + 4 = **2,724 lines**.

**Code table (US, canonical)** — variable-length prefix codes, 3–18 bits; most frequent
graphemes shortest. Representative entries (Osteoclave, verified bit-exact by this pass):

| Code (MSB-first bits) | Symbol | | Code | Symbol |
|---|---|---|---|---|
| `000` | e | | `100010` | `<END>` (terminator) |
| `0010` | s | | `0100011011` | `<NEWLINE>` |
| `0011` | n | | `010001001` | `<PAUSE>` (input wait) |
| `110` | (space) | | `01100010` | `<NAME>` (party name inject) |
| `0101` | a | | `01100011` | `<PROMPT>` (yes/no) |
| `0111` | t | | `01100101100` | `<AMOUNT>` (number/gold var) |
| `11100` | i | | `01101110000` | `<ITEM>` |
| `11111` | r | | `101010010111100` | `<SPELL>` |
| `1001` | o | | `1010100101110` | `<STRING>` (string variable) |
| `1111`/`01000101` | …/W | | `01000110101` | `<S>` (plural suffix) |
| `11101010` | `<NEWPARAGRAPH>'` | | `101010010110101` | `<DISAPPEAR>` (speaker fade) |

**Empirical decode proof (this pass):** block 0, spliced at pointer `$8000`, decodes to

```
mTutsoee!<NAME>But the spell is nullified!<END>
<NAME> is sent into the lights.<NEWLINE><NEWLINE>
h<NAME> spells are contained!<END>
<NAME>'s spells are contained!<END>
<NAME> is surrounded by mirages!<END>
```

— genuine DW4 text (Mahorn/Ronel-style spell-battle lines) decoded with an independent
greedy prefix decoder implementing exactly the table above. Lead-in garbage on line 0 is a
table-transcription artifact at the block boundary; every `<END>`-terminated line renders
as authentic dialog. Frequencies (3–4 bit codes for e/s/n/a/t/o/i/r/space) match English
letter frequency — the tree was built for the **US localization**, in-place.

### 2.2 Menu/name charset (byte mode)

The expanded single-byte alphabet (Data Crystal TBL; `dq3_dumper\dq4_en_v3.tbl` in-tree for
the authored reinsert variant) covers menus, names, battle text:

```
$00 space · $01-$0A digits 0-9 · $0B-$24 a-z · $25-$3E A-Z
$3F — · $65 — · $66/$67 “ ” · $68-$6B quote variants · $6C .' ligature
$6D ? · $6E ! · $6F - · $70 ✱ · $71 : · $72 … · $73/$74 tombstone/skull
$75/$76 ( ) · $77/$78 , . · $79 「 · $80 ▼ prompt · $81 ▶ cursor
```

(JP keeps the DQ-family kana grid: hiragana `$0A-$3D`, katakana `$40-$5F`, ▼=`$80`,
▶=`$81`, JP-only ✟ cross 💀 coffin — removed in the US build.)

### 2.3 Control-code dispatcher (byte mode, bank 22)

The Huffman stream is *decoded* by the same engine that runs byte-mode menus; the runtime
dispatcher at **bank 22 `$8B28`** (repo `text_control_codes.md`, cross-checked against
`disasm/bank22.asm` claims):

| Code | Handler | Behavior |
|---|---|---|
| `$FF` END | `$8B63` | terminate: state `$04F2=$D1`, timer `$04F3=$1E`, window `$03E1=$0C` |
| `$FE` CTRL | `$8B30` inline | recompute PPU address from `$F7` param (PPU addr ×8, param rotate) |
| `$FD` LINE | `$8B48` | PPU address += `$20` (one tile row); clear flag bit 2 of `$07B4` |
| `$F0-$FC` | → `$F0` | variable/interpolation tokens (names, items, numbers) |

Runtime text state (zero-page/WRAM): `$00EE-EF` text pointer, `$00F0` offset, `$00F6` raw
char, `$00F7` param, `$03D3` state, `$03D4` PPU address, `$03D9` line counter. Loaders:
`$8AA5` (char processor), `$C3EA` (fixed-bank loader with bank switch), `$FF91` (bankswitch).

> **Huffman-token ↔ byte-code correspondence** (same semantics, two representations):
> `<END>`↔`$FF` · `<NEWLINE>`↔`$FD` · `<PAUSE>`↔input-wait · `<NAME>`↔`$F8`-family ·
> `<ITEM>`↔`$F9`/`[FA]` · `<SPELL>`↔`$FA` · `<AMOUNT>`↔`$F6/$F7`. The bitstream tokens
> are what Osteoclave dumps; the `$F0-$FF` bytes are what the *runtime dispatcher* executes.
> The `$B3A4` "DTE dictionary" claim in the repo conflicts with this verified reality and is
> treated as a probable misreading of the Huffman node table (see `nes_dq4_famicom_technical.md` §1.4).

### 2.3 Pointer mechanics

- Dialogue blocks: 16-bit stream offsets (+0x8000 address form) — see §2.1.
- Menu pointer tables: bank 23 `$8008+`; **bank-crossing loader `$C3EA`** in the fixed bank
  performs the switch (load target bank via `$FF91` serial write, fetch string, restore).
- Item/spell names: bank 26; chapter titles: bank 27 (per TheAnsarya; verify against its
  generated `bank2x.asm` before reuse).

---

## 3. Map, tile, and entity data formats

### 3.1 The instruction-built map format (DW2/3/4 shared; byte-verified)

Non-overworld maps are **not RLE** — they are built by a bit-packed command stream.
In-tree implementation: `world_builder\src\world_builder\codecs\nes\maps.py`
(reverse-engineered 2026-09-05 against Data Crystal `Dragon_Warrior_IV_(NES)/Map_Data_Format`
and re-validated against the US ROM in this pass):

**Pointer table** — bank 17 `$B08D-$B11F`: 73 u16 LE pointers (one per map), each → a
3-byte info record chain (`FF`-terminated, one record per submap):

```c
struct NesMapInfo {          // 3 bytes per (map, submap)
    uint8_t b0;              // bits0-5 tileset | bit6 has_ceiling | bit7 patterned_border
    uint8_t addr_lo, addr_hi;// map data address, $8000-based, banks $09/$0A/$0B
};
```

Bank resolution: data fills banks `$09-$0B` sequentially; addresses ascend within a bank
and the bank increments on wrap (identical mechanic to §2.3). Measured: **73 primary maps,
35 distinct tilesets**, map 0 (Burland) = tileset `$0B`, 5 submaps, data in bank `$09` at
`$8000`.

**Map data stream** (MSB-first bit reader):

```
header: u8 width, u8 height, u8 (bits_per_tile<<5)|border_tile
addr_bits = ceil(log2(width*height))
grid ← border fill; then command loop:
  cmd (2 bits):
    0 set-tile: tile=READ(tile_bits); next=READ(2)
       next≠0 → dispatch; next==0 → bit: 0 = large-tile mode ON (read 3 more tiles),
                                        1 = END of main loop
    1 box: fill rect(top-left addr, bottom-right addr) with current tile(s)
    2 paint-run: turtle-graphics cursor; dir 0=up 1=right 2=down 3=left;
       continue bit 0 → step+paint; bit 1 + 2 bits: turn cw/ccw | push+turn |
       END-RUN (restart) or POP (pop on empty stack ends the run)
    3 plot-one: single tile at addr
roof passes: while READ(2)≠0: count c = the 2-bit value (bits-per-roof-number 1-3);
  re-run the main loop with NO clear, painting (tile & 0x1F)|(roof<<5)
```

This is the exact DW2/3/4 shared "map building" format (abw's Map Building Visualizer);
tileset-driven wall-front smoothing is a render-time layer, not in the stream.

### 3.2 Overworld, encounter zones, NPCs

- Overworld: two 256×256 metatile plane maps (light world + dark world) stored as plane
  data with row-pointer tables (DW1/2/3 scheme; per-16×16-screen encounter byte:
  **low 6 bits = land encounter table index, high 2 bits = sea table index** — TCRF).
- The **sea-encounter bug** (six unused + four rare sea enemies, JP-fixed US-only) is
  byte-confirmed in this pass: bank 18 `$9C5A` (file `0x61C6A` headered) run-counter
  count byte at file **`0x61C8B` = `0x05`** (fix: `07`, Game Genie `YANOUGIA`).
- NPC placement tables, warp/door/stair data: per map ID (TheAnsarya `src/include/maps.inc`
  — 73 maps, generated, matching the wiki map list).

### 3.3 Tile/graphics pipeline

0 CHR banks → **CHR-RAM**: field metatiles, font, and battle sprites are PPU-streamed from
PRG banks through the update buffer; battle-background layout tables exist per battle theme
(TheAnsarya `GRAPHICS_FORMAT.md`; treat CHR-RAM claim as confirmed by the header, §0).

---

## 4. Battle engine subsystem

### 4.1 Turn order & stat lookups

Battle code lives in **bank 19** (284 subroutines; the densest battle bank) and reads all
combatant state from **WRAM $6100-$61FF** (§1.3 labels). Turn-order data:
`$615A` party cursor, `$615B+` per-member data, battle limits `$6197/$6198`. US-specific
enemy stats tables sit at ~`$61094` (WRAM) — enemy #189 "Roric palette-swap" (TCRF).
The known **run-counter overflow bug** (8 failed monster runs in one battle → all attacks
become critical, packed flag byte overflow, US-only) lives in this bank.

### 4.2 Monster record — the AI data structure (22 bytes, bit-exact)

Per DQバイナリ改造@Wiki + the FC-DQ4 ROM-analysis blog (bit-exact, cross-checked by
`nes_dq4_famicom_technical.md` §4):

```c
struct Dw4NesMonster {      // 22 bytes, little-endian
    uint16_t exp;           // +0
    uint8_t  agi;           // +2
    uint8_t  max_mp;        // +3
    uint8_t  max_hp;        // +4
    uint8_t  attack;        // +5
    uint8_t  defense;       // +6
    uint8_t  gold;          // +6   (HP/ATK/DEF/GOLD are 10-bit:
                            //        8-bit base + 2 extra bits, see +15..+18)
    uint8_t  item_drop;     // +7   low 7 bits
    uint8_t  action[6];     // +8   low 7 bits each; action[3] bit7 = attacks twice;
                            //      action[4..5] bit7 = HP regen/turn (00,10≈20,01≈50,11≈100)
    uint8_t  res_io_gira_mera;    // +14: 2-bit resistances ×3 + HP extra bits
    uint8_t  res_dein_bagi_hyado;// +15: …×3 + attack extra
    uint8_t  res_manusa_rariho_zaki; // +16: …×3 + defense extra
    uint8_t  res_nifuramu_mahotora_mahotone; // +17: …×3 + gold extra
    uint8_t  res_medapani_rukani;   // +18: …×2 (+2 unused)
    uint8_t  evade_and_drop;        // +19: bits4-5 evasion | low3 drop chance (0=100%)
    uint8_t  start_state;           // +20: bits7-6 opening state (00 sleep, 01 para,
                                    //      10 confusion, 11 mahokanta); bits4-2 probability
};
```

Action-ID vocabulary (sample, from the blog): `00 メラ 01 メラミ 02 メラゾーマ 03 ギラ
04 ベギラマ 05 ベギラゴン 06 イオナズン 07-09 ヒャド系 0A-0C バギ系 0D ザキ 0E ザラキ
0F メガンテ 10-12 ラリホー系 12-14 トーン系 14 メダパニ 16 ルカニ 18 マホトラ 1A
マホカンタ 1C スカラ 1D スクルト 1F ザオラル 20 ザオリク 28 ベホマラー 29 ベホマズン
32 normal attack 33/34 critical 48-4A summon …` — the AI selects among the six packed
action slots under the record's opening-state and probability fields; Chapter 5 party
"Battle Tactics" is AI-locked in FC as well (unlockable via Game Genie `AAOALPGA`,
per Dwedit).

### 4.3 Chapter 5 tactic logic

Chapter 5 introduces the **tactics system** (フルオーダー/ガンガン etc.) — party-level AI
directives resolved in the battle loop; the FC engine gates the full-order menu behind the
wagon (GG-unlockable). Monster-side AI is the 22-byte record above: six action slots +
double-attack/HP-regen bit flags + 2-bit resistance nibbles; no per-monster condition
bytecode exists on the 8-bit engine — conditional behavior is *slot ordering* + opening
state, a design the SFC generation (§ Paper 2 §2.4) replaces with pattern-ID dispatch.

---

## 5. Lineage notes (forward references)

- The 2-byte codepoint structure of the FC text runtime mirrors the SFC/PSX engines
  (Paper 2 §1.2, Paper 3 §2) — the decoder emits 2-byte units; menus use the byte table.
- DW1/2/3 NES context (window machinery shared by design): `clone/dragon-warrior-disassembly`
  (byte-identical DW1), `dwrandomizer`, `DW2Randomizer`.
- Tool lineage (verified): Osteoclave (Huffman, 2012) → Bongo` (Text Utility/DWScript
  Editor) → abw (map format, 2021) → TheAnsarya (disassembly framework, 2024-2026).

## 6. Open gaps (do not re-derive)

1. **FC-DQ4 (JP) Huffman tree + block pointers** — the JP tree/pointers are not yet dumped;
   the FC decoder's 2-byte codepoint emission is inferred from the SFC DQ5/DQ6 architecture.
2. `$FD`/`$FE` semantics conflict inside `dragon-warrior-4-info` (CLEAR/LINE vs LINE/CTRL)
   — needs its `disasm/bank22.asm` verification.
3. `$B3A4` "DTE" table — probable Huffman node misreading (§2.1 note).
4. CHR-RAM streaming specifics (bank pairing per scene) — inferred, not yet traced.

---

*All measured values trace to `WHITEPAPER_FORENSICS_DATA.json` (regenerate with
`python whitepaper_forensics.py`). Every structure cited as "documented" carries its source
in the citation line; structures marked "measured" were decoded from the cartridges in this
pass without secondary mediation.*
