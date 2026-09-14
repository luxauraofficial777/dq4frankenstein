# DRAGON QUEST I & II (Super Famicom, 1993) — Engine Architecture Whitepaper

**Paper 2 of 5 — Enix Engine Specification Suite**
**Doc ID:** VW-SNES-WP-002 · 2026-09-14
**Scope:** 「DRAGONQUEST 1・2」 Chunsoft/Enix, 1993-12-18 — the 16-bit bridge between the
Famicom table era and the HeartBeat Huffman era.
**Cartridge analyzed:** `famicom\snes_hbe\Dragon Quest I & II (J) [T-Eng2.0DQ_RPGOne].smc`
(2,097,152 B, headerless — the RPGOne v2.0 repack; data tables are JP-original bytes,
verified against the community randomizer's ground truth). The JP original is 1.5 MiB
(12 Mbit) per Data Crystal (CRC32 `98BB6853`); this image expands to 2 MiB and updates the
header size code (§0).
**Method:** every "measured" value below was taken by `snes\study\whitepaper_forensics.py`
from the cartridge bytes; machine-readable output in `WHITEPAPER_FORENSICS_DATA.json`.
Cross-references: `snes\docs\dq12_sfc_technical.md`, `DQ12_SYSTEMS_MINING_SNAPSHOT_Sep6_2026.md`
(offsets re-verified against OUR ROM), Shingo Endo's `dq_analyzer` (`dq12decode.c`,
`dq1analyzer`/`dq2analyzer`), the atwiki DQバイナリ改造 wiki (SFC-DQ1·2 page 18; its
addresses include the +0x200 SMC header — subtract 0x200 for our headerless file), and the
`braves_dqdata` table archive.

---

## 0. ROM container identity (measured)

| Field | Value (from header at `$00:7FC0`) |
|---|---|
| Title | `DRAGONQUEST 1・2` (21-byte field, title-case as shipped) |
| Map mode (`$7FD5`) | **`0x20` — LoROM** |
| Cart type (`$7FD6`) | **`0x02` — ROM + RAM + battery** |
| ROM size (`$7FD7`) | `0x0B` → 2 MiB (RPGOne repack; JP original 1.5 MiB / `0x0A`-class) |
| RAM size (`$7FD8`) | **`0x03` → 8 KiB battery SRAM** |
| Country (`$7FD9`) | Japan |
| Header checksum | **VALID** — complement XOR checksum = `0xFFFF` over the 2 MiB image (measured) |
| Copier header | none (size exactly 2,097,152) |

**LoROM bank model (verified):** SNES `$00-$3F:8000-$FFFF` → file
`bank*0x8000 + (addr-0x8000)`; linear, no copier header. 64 banks × 32 KiB = 2 MiB. The
addressable code/data window of the slow 65C816 (SlowROM, 200 ns class per Data Crystal)
plus mirrored banks `$80-$FF` (FastRAM region semantics do not apply — this is a SlowROM
LoROM).

---

## 1. Memory mapping & cartridge topology

### 1.1 Address map

```
$00-$3F:$8000-$FFFF   PRG-ROM banks (LoROM; 32 KiB per bank)
$40-$7D:$0000-$7FFF   WRAM mirror window (7E direct-page $7Exxxx native)
$70-$7D:$0000-$7FFF   SRAM window (8 KiB used; battery)
$7E0000-$7FFFFF       WRAM 128 KiB (stack, engine state, name buffers)
$00-$3F:$0000-$7FFF   mirrors of $7E-$7F low halves + I/O
$40-$43:xxxx          MMIO (PPU $2100-$21FF, APU $2140-$217F via $40xx window)
```

WRAM native addressing is the engine's working set: name entry writes 1 byte per char at
`$7E0156-$7E015A` (5-char name buffers, cheat-verified), DQ2 party stat blocks stride from
`$7E0C01` (level/HP/MP/STR/AGI/DEF per char, 3-character hero block shared with the DQ1
hero), inventory array at `$7E0CBE+` (10 slots). Battle background id at `$7E0081`;
no-encounter `$7E0027`;トヘロス latch `$7E00E2` (cheat-derived RAM map, roadbikebeginners).

### 1.2 SRAM allocation & integrity — status (gap-fill updated, Sep 14)

- Size: 8 KiB (`$70:0000-$70:1FFF` window; header byte `0x03`).
- **Save routines LOCATED (measured, `gapfill_forensics.py → dq12_sram`).** A 65816
  long-addressing census of the `$70-$7D` bank byte over the full image (576 hits,
  8 clusters of ≥3) places the save/load machinery at:
  - **bank `$00:$DF1C-$DF5C`** (raw `0x5F1C-0x5F5C`) — paired `LDA long,X` /
    `STA long,X` over `$70:0000-$70:0004`: the save-slot core read/write block;
  - **bank `$09:$B1A3-$B5xx`** (raw `0x4B1A3+`) — the dense save/state cluster with a
    `$46`-byte stride pattern (`$70:0014/$70:0059/$70:009E/$70:00E3` reads at
    `$09:$B431+`) and coverage up to `$70:0AB3`.
- **Remaining open item (precise):** the exact slot-layout map and the checksum
  arithmetic. Notably, no `LDA #$0070 / PHA / PLB` bank-load sequence occurs anywhere
  (0 hits) — the engine changes DB by other means — so the checksum computation was
  not isolatable statically in this pass; it now has concrete routine banks to trace
  instead of a blank. Known constraints stand: the cartridge is `ROM+RAM+battery`
  (type `0x02`), 8 KiB SRAM, two games' saves coexist, and the community cheat tooling
  manipulates WRAM, not SRAM.
- ROM-side integrity is measured and green (§0): complement⊕checksum = `0xFFFF` over the
  full 2 MiB image — the RPGOne repack recomputed the header checksum, so the cart boots
  with internal SRAM-check paths intact.

---

## 2. Compression & data storage

### 2.1 The dialogue text is NOT compressed (corrected assumption — now triple-confirmed)

Unlike DQ3 SFC (Paper 3 §2), the DQ1+2 SFC script is **plain byte-per-glyph streams** with
a 2-byte escape mechanism. Verified structure (s-endo `dq12decode.c` `dec_string()`):

```c
/* single-byte mode (menus, battle text, fixed strings) */
emit tbl_charcode[byte];                     /* 256-entry kana table */

/* message mode (dialogue / "message font") */
if (0xd0 <= b && b <= 0xd2)                  /* 2-byte kanji escape */
    emit tbl_charcode_mb2[((b - 0xd0) << 8) | next_byte];   /* 768 entries */
else
    emit tbl_charcode_mb1[b];                /* 256 entries, message font */

/* dakuten composition (both modes) */
if (next == 0x83 && tbl_dakuten[cur])     emit tbl_dakuten[cur], consume 2 bytes;
else if (next == 0x82 && tbl_handakuten[cur]) emit tbl_handakuten[cur], consume 2;
```

Design consequences:
- `0x00` terminates/pads strings — no `0xAC`-style box terminators, no per-string window
  control codes, no furigana (the game has none; windows are palette-swapped frames).
- The 768-entry 2-byte table is the SFC's only "wide" character space — no pointer table
  into a compressed blob, because the text is not compressed.
- 5-char name buffers are fixed width (cheat write sites at `$7E0156-A`, 1 byte/char).

### 2.2 Control codes (from the master cross-engine library, SFC-DQ1·2 column)

| Code | Function |
|---|---|
| `0x00FF` | end of string |
| `0x00F0` / `0x00FE` | line break (two variants) |
| `0x007A` | wait for input (▼) |
| `0x00F4` | hero/player name inject |
| `0x00D0-$00D2` + byte | 2-byte kanji lead (768-entry message-font table) |
| `0x0082` / `0x0083` | handakuten / dakuten compose with preceding kana |
| `0x0088-$008C` | emphasis glyphs (*, 。, ., 「, …) |

No embedded pointers, no conditional grammar, no tone registers exist in this engine —
the first appearance of any of those is DQ3 SFC (Paper 3 §3).

### 2.3 Planar graphics: the `[row][plane]` tile layout (measured, content-proven)

Monster art occupies banks **`$2D-$32` + `$37-$38`** (art-likeness census: mean
bit-transitions/byte-row ≈ 1.83 for art vs ≈ 18.5 for code; banks `$35/$36` empty —
project re-verification). The tile format is standard **4bpp 8×8, 32 B/tile**, but bytes
are stored **`[row][plane]` interleaved** — byte *k* of a tile = row *k/4*, plane *k%4* —
NOT the SNES-standard `[plane][row]`. This was settled by content, not by decode quality
alone (transposing an 8×8 tile preserves art-likeness): mirror-pair adjacency
(e.g. bank `$2E` tiles 853/854 are left/right mirror pairs), 4-colour ground/shadow bands,
and left-edge idle-wiggle fragments all decode coherently only under `[row][plane]`.

Storage unit = **0x800-byte chunks (64 tiles)**, matching the engine's WRAM staging copy
(`$7EF800` ← bank `$2F` tail `$E000-$FFFF`, 4 dense chunks, indexed by WRAM `$7EF449`
which cycles 0-3 = battle-BG variant slots; `INC`/`CMP #$04` at `0x1636A6`).

### 2.4 Battle graphics banks & the battle VM

- **Battle engine home = bank `$2C`** (code: hit-flash routines ~`0x161E57+`).
- **Battle-animation VM command table**: bank `$2C` `$B5C0-$B820` — u24 handler table
  (`$2CB626 / $2CB662 / $2CB6C3 / $2CB6FD / $2CB758 / $2CB7A2 …`); the bytecode at
  raw `0x161Cxx-0x1626xx` uses ops `83/8C/92/95/9C/9F` with `0x3C`-strided VRAM args.
- **DMA sites with static ROM sources**: bank `$2C:$C28C` → VRAM `$4C00` (0x800 B, setup
  at `0x16424C`) and bank `$30:$9200` → VRAM `$1400` (0x1600 B, setup at `0x183E36`).
- The `(x,y,tile)` triplet list at raw `0x161DD0-0x161E22` (tiles `0x02-0x33`,
  FF-terminated) = battle-BG row-build table (lo/hi tile pairs per row).

---

## 3. Dialogue & event system

### 3.1 Text storage model

Strings are `0x00`-separated byte streams in dedicated string areas; menu strings form
separate blocks. The one **published pointer table** in this engine's text layer:

| Region (headerless raw) | Content |
|---|---|
| `0x4EA5F` (`04EC5F`−0x200) | item-price string pointer table, **2 B × 87 entries** |
| `0x4EB14+` (`04ED14`−0x200) | item-price strings (variable: item-name chars + digit chars, concatenated) |
| `0x4E210-$4E220` | DQ1 equipment attack/defense values, 1 B each (たけざお…みかがみのたて) |
| `0x4E233-$4E254` | DQ2 equipment values (ひのきのぼう…ふしぎなぼうし) |

Measured in this pass: the 87-entry table decodes; ordering is not globally monotone
(`$D6DC, $D6EC, $D596 …`) — string clustering groups by name-family, so "monotone pointer
run" (the DQ3 standard) does NOT hold here; the table is bank-window addressed.

### 3.2 Verb/token system

No conditional-grammar tokens exist; name injection is a fixed code (`0xF4` hero name);
pronoun/noun interpolation does not exist as text codes (the JP engine's gender nouns are
baked into the string, not parameterized). The dialogue system is therefore a
**fixed-width glyph renderer + fixed code set** — the evolutionary antecedent whose
*absence of features* is itself the lineage data point (Paper 3 §3.2).

### 3.3 Event system

- **Event script region ≈ raw `0xE0000-0xE5000` (bank `$1C`)**: treasure item ids are
  EMBEDDED in event scripts (49 DQ1 script addresses archived in `dq12mining/content.json`;
  specials `0x4AE33/0x4AE38/0x4AE43`). Hero-growth patch site `0xDA03D` (32 B, bank `$1B`).
- Battle trigger hooks: princess/armor monster instantiation ids in event data
  (`0x5084`, `0x500F`); battle block: HP `0x59BC7`, MP `0x59BCF`, AGI `0x59BD7`,
  DEF `0x59BDC`, STR `0x59BE4`, XP `0x591CE` (u16).
- Encounter-rate hooks: `0x5CFD4`, `0x59739`.

---

## 4. Map & event engine

### 4.1 Encounter zones & tile level

- Overworld = one large tilemap per game, partitioned into **16×16-tile encounter zones**
  (`X00-X15 × Y00-Y15`, value = area number; `dq12s-2field.txt` family).
- **Tile-level table** (`dq1s-2tilelevel.txt` / `dq2s-2tilelevel.txt`): tile level vs
  party level — `hero_level ≥ tile_level*5` → 100% flee; `M00-M04` = group spawn,
  `M05` = solo spawn.
- Floor data (`dq12s-2floor.txt`): per-floor encounter area + depth count.
- No byte-level tilemap compression is published for this game; the 12 Mbit cart carries
  raw tilemaps (the RPGOne repack to 16 Mbit adds English banks, measured §0).

### 4.2 Monster/zone content tables (all re-verified against THIS cartridge)

**Monster stat table — SHARED by both games, one table:**

```
base 0x5DA0E, stride 18, 122 records:
  DQ1 = records 0..40,  DQ2 = records 41..121

struct Dq12Monster {          // 18 bytes
    uint8_t agi;              // +0
    uint8_t atk;              // +1
    uint8_t def;              // +2
    uint8_t hp;               // +3
    uint8_t mp;               // +4
    uint8_t gold_lo;          // +5   (gold high 2 bits live at +14 bits6-7)
    uint8_t slot[8];          // +6..13  AI pattern slots:
                              //   hi 3 bits = flags, lo 5 bits = pattern id;
                              //   +12 hi3 = action-pattern id,
                              //   +13 hi3 = item-drop difficulty (DQ2 only)
    uint8_t evade_int_goldhi; // +14: bit0-3 physical-evade %, bit4-5 intelligence,
                              //       bit6-7 gold high bits
    uint8_t held_item;        // +15
    uint16_t exp;             // +16..17 LE
};
```

**Ground-truth record (byte-exact, this pass):** record 16 = Metal Slime —
raw `99 12 fe 04 06 06 e0 e5 e0 e5 e0 e5 00 06 00 00 07 03` →
AGI=153, STR=18, DEF=254, **HP=4**, XP=**775**, pattern slots alternating
flags%7/id **0,5,0,5,0,5** then `0,0` / `0,6` — exactly Metal Slime's flee-alternating
AI. Record 105 (DQ2 metal class) XP = 10,150; XP range observed 0..10,150.

**AI pattern vocabulary (IDs 0-31):** 0 normal attack · 1 critical · 2 poison · 3 sleep ·
4 defend · 5 flee · 6 ギラ · 7 ベギラマ · 8 イオナズン · 9-11 ホイミ/ベホイミ/ベホマ ·
15 ザオリク · 16 ルカナン · 17 スクルト · 18 ラリホー · 19 マホトーン · 20 マヌーサ ·
21 ザラキ · 22 メガンテ · 23-25 fire-breath tiers · 26 poison breath · 27 sweet breath ·
28 summon · 29 double attack · 30 concentrate · 31 MP-drain dance.

**Dispatch: the AI decision "tree" is a jump table.** Pattern IDs index a **u16 dispatch
table at `$0B:BD8C`** (raw `0x5BD8C`) = AI pattern routines; measured first entries
`$DA55, $DA93, $DA5D, $DA6B, $DA50 …` (entry 0 = null). The dispatch state machine reads
the record's pattern slots in order under the hi-3-bit flags — a strictly flatter design
than the NES 22-byte slot-ordering model and far flatter than the DQ3/PSX event VMs.

**Boss HP > 255 hardcode — byte-proven this pass** (atwiki 059C04+ family):

```
0x059C14  A2 40 01   LDX #$0140   (ベリアル/Berial  HP 320)
0x059C19  A2 CC 01   LDX #$01CC   (ハーゴン/Hargon  HP 460)
0x059C1E  A2 D6 06   LDX #$06D6   (シドー/Sidoh     HP 1750)
```

The engine's 8-bit HP cell cannot hold these values; the code `CMP #$50/$51/$52 … LDX
#$xxxx` patches them in — the exact byte sequence located at the three offsets above.

**Zones** (measured, matching the mining snapshot):

| Table | Base (raw) | Shape | Used slots |
|---|---|---|---|
| DQ1 zones | `0x5B52D` | 20 zones × 5 slots | **100** (IDs 1-37) |
| DQ2 zones | `0x5BEC5` | 68 zones × 8 B (6 used) | **378 raw / 348 in-slot** (0xFF = empty) |
| Weapons | `0x4EFE1` | 103 B, 0xFF-terminated groups | **16** groups (DQ1 recs 0-41, DQ2 42-102) |
| Items | `0x4F0C1` | 59 B | **13** groups (DQ1 0-18, DQ2 19-58) |

**Scene table** — 100-record, 3-byte interleaved `[f0][f1][tag]` at raw `0x5C322`
(tags 1..100 ascending; parsed clean from tag 25; consumer at `0x5CAB0`:
`LDA $0BC323,X; JSR $C8A6` … `(0x1F − f0)` = screen-Y arithmetic → per-zone
scene-layout/battle-position data). Related parallel tables: `$0B:C131`, `$0B:C99A/B`.

---

## 5. Integrity model

- **ROM checksum**: measured valid on the shipped 2 MiB image (§0). The SFC header's
  complement/checksum pair is recomputed by the repack — the only published integrity
  check for this title.
- **SRAM save integrity**: layout + checksum = partially RESOLVED by the Sep-14
  structural scan (`gapfill_forensics.py → dq12_sram`). A 65816 long-addressing census
  of the `$70-$7D` bank byte over the whole image finds **576 SRAM-window accesses in
  8 clusters of ≥3**, locating the save/load machinery at:
  - **bank `$00:$DF1C-$DF5C`** (raw `0x5F1C-0x5F5C`) — paired `LDA long,X`/`STA long,X`
    over `$70:0000-$70:0004`: the save-slot core read/write block;
  - **bank `$09:$B1A3-$B5xx`** (raw `0x4B1A3+`) — the dense save/state cluster:
    `$70:0002/$70:0008/$70:000E/$70:0014/$70:0059/$70:009E/$70:00E3/$70:01B2…`
    strided read/write groups (stride `$46` = 70 bytes between `$70:0014/$70:0059/
    $70:009E/$70:00E3` reads at `$09:$B431+` — the per-record save stride candidate),
    extending to `$70:0AB3`.
  No `LDA #$0070 / PHA / PLB` bank-load sequence occurs (0 hits) — the engine changes
  DB by other means (PHB/RTL frames or TCD-page conventions), so the exact checksum
  computation was not isolated statically in this pass. Status: **save-routine banks
  LOCATED (measured); slot-layout map and checksum arithmetic remain the two precise
  remaining artifacts** — both now have concrete routine addresses to trace, replacing
  the earlier blank.

---

## 6. Lineage notes (bridge position)

1. **Storage paradigm**: raw tables + raw text, no container abstraction — every system
   is a direct ROM-offset table with an explicit base/stride. The "archive" idea does not
   exist yet; it appears in DQ3 SFC (per-resource streams) and completes in the PSX
   HBD container (Paper 3 §4).
2. **Text dispatch**: fixed glyph codes + a 768-entry 2-byte kanji escape — no Huffman.
   This is the *negative result* that defines the Huffman era's starting line.
3. **Monster AI**: 8 pattern slots → flat u16 dispatch table in the same bank. The
   HeartBeat PSX carries the same AI-decision-weight concept into a 32-bit bit-packed
   event-flag matrix (`0x8000F800-0x80010000`, Paper 3 §4).
4. **Tile format**: `[row][plane]` interleave (not SNES-standard `[plane][row]`) —
   a Chunsoft-local convention worth keeping in any cross-generation tile converter.
5. **Engine facts that bind generations**: the `$7EF800` staging copy + `$7EF449`
   battle-BG variant cycling (4 slots) is the 16-bit direct ancestor of the PSX engine's
   per-sector background streaming into heap `0x80138000` (Paper 3 §4).

---

*Measured values regenerate via `python whitepaper_forensics.py` +
`python gapfill_forensics.py`; see `WHITEPAPER_FORENSICS_DATA.json → dq12_sfc` and
`GAPFILL_FORENSICS_DATA.json → dq12_sram`. Documented-but-unverified structures carry
their source citation; the remaining SRAM artifacts are the slot-layout map and the
checksum arithmetic, both now anchored to measured routine banks.*
