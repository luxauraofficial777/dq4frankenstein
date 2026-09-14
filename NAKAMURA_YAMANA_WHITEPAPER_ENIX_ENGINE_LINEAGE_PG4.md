# DRAGON QUEST VI (Super Famicom, 1995) — Engine Architecture Whitepaper

**Paper 4 of 5 — Enix Engine Specification Suite**
**Doc ID:** VW-SNES-WP-005 · 2026-09-14
**Scope:** ドラゴンクエストVI 幻の大地 (Enix/Heartbeat, 1995-12-09) — the first full
HeartBeat 16-bit engine: the global-Huffman dialogue system, the co-opted `BRK`
message primitive, the grouped variable-width font, the talk-object engine, the
42-byte monster struct, and the LC_LZ21 map layer.
**Cartridge analyzed:** `famicom\snes_hbe\Dragon Quest VI\Dragon Quest 6 (tr).sfc`
(4,194,304 B — DQ Translations repackage; Vimm-style header at `$FFC0` is repack junk,
documented below). Reference ROM (No-Intro 0650): HiROM, 4 MiB, FastROM, CRC32
`33304519`, MD5 `ac9955fa4c1aa8ebcc1d09511808c58b` (dq6-sfc/README).
**Method:** values marked "measured" were taken by `snes\study\hbd_forensics.py`
(`HBD_FORENSICS_DATA.json`) against the in-tree ROM and the verified tree tables;
values marked "documented" cite `dq6_sfc_huffman.md`, `dq6_sfc_engine.md`,
the ButThouMust dq6-sfc dumpers, Endo's `dq6decode.c`, and the dqbook (showa-yojyo).

---

## 0. ROM container identity

| Field | Measured (our file) | Reference |
|---|---|---|
| Bytes | 4,194,304 (no copier header) | 4 MiB HiROM |
| `$FFC0` header | repack artifacts (Vimm watermark) — **not game-authentic** | No-Intro: title `DRAGONQUEST VI`, mapmode HiROM |
| Provenance note | the DQ Translations patch rebuilt the script payload; the **tree tables at bank `$C1` are preserved** at their documented addresses (measured, §1) | reference MD5 above |

The tree-position verification below is against the repackage — it confirms the
jp-layout tree survived translation, which is itself useful reauthoring knowledge.

---

## 1. Two text systems — and the DQ5 polarity flip

| System | Mode | Compression | Pointer table | Base |
|---|---|---|---|---|
| Travel mode (walk dialogue, big font) | Huffman | **yes** | `$C15BB5` (raw `0x15BB5`), **870 × 3 B** (measured: entry count; table extent `$C15BB5-$C165E6` = `0xA32` B) | `$F7175B` (raw `0x37175B`) |
| Battle mode (small font) | **raw char codes, NOT Huffman** | no | `$C15AD1` (measured table head this pass) | `+$DEBD` → `$F6DEBD` |

⚠ **DQ5/DQ6 split is inverted**: in DQ5 *both* modes are Huffman; in DQ6 only travel.
In DQ3 SFC the same dialogue/font engine is reused with different pointers (the
`dq6-sfc` README documents that DQ3's dumpers were built on the DQ6 codebase).

### 1.1 The Huffman engine (measured this pass)

**Tree** — two parallel u16-LE node arrays in bank `$C1`:

```c
/* raw 0x167BE ("OFF"/bit-0 side) and raw 0x1700E ("ON"/bit-1 side),
   0x428 = 1064 entries each; root = last entry, index 0x427 */
/* MEASURED: left[0x427] = 0x884A, right[0x427] = 0x884C
   — both MSB-set (inner nodes), 582/557 inner nodes across the arrays */

struct DQ6Node {
    uint16_t value;
    /* bit15 SET   = inner node  (polarity INVERTED vs DQ5/Otogirisou!)
       next index  = (value & 0x7FFF) >> 1  (byte pos p ↔ entry p >> 1)
       bit15 CLEAR = leaf; char code = value & 0x1FFF */
};
```

**Bit order — LSB-first within each byte.** Mask array at `$C02BCC` =
`01 02 04 08 10 20 40 80` read ascending: the first bit read is bit `(ptr & 7)`
(descending within the byte after start), byte advance on wrap. Leaf value `& 0x1FFF`:
`< 0x200` → control code; `≥ 0x200` → glyph via `tbl_charcode_mes[value − 0x200]`.

**Pointer table** — 3-byte LE entries = 24-bit **bit offset** from script base
`$F7175B`; **8 strings per pointer** (id→`ptr[id>>3]`, skip `id&7` terminators; the
dqbook's computed trade-off: 8-strings-per-pointer wastes decode time but saves
~20,880 bytes over per-string pointers). Measured table this pass: 870 entries,
entry 0 = `0x%06X` (see JSON), monotone across the table, blob span
`$37175B … (max_ptr>>3)+base` consistent with the documented script extent.

**Reference decoder state** (ROM routines, documented): `$C02B69` (msg ID → address +
bit), `$C02BD4` (decode one char), `$C02C28` (small→big font conversion, table
`$C11100`), `$C029E6` (control-code ASM dispatcher), `$C02671`/`$C027B0`/`$C02831`
(battle raw-code counterparts, dakuten/handakuten tables `$C0297B`/`$C029A4`).

### 1.2 Control codes (leaf values < 0x200; Endo-verified table)

| Code | Function | | Code | Function |
|---|---|---|---|---|
| `00AC` / `00AE` | string terminator | | `00C9-$00D0` | **fixed character names 1-8** (`value−0xC9` indexes `str_chrname[8]`) |
| `00AD` | line break | | `00D1/00D2/00D3` | context names (self / self-or-leader / leader) |
| `00AF` | input wait (▼) | | `00D4` / `00DE` | **opening speech-mark 「** (suppresses `＊「` prefix) |
| `00B0` | wait (silenced) | | `00D5` | window close |
| `00B3/00C0/00C1` | character-name substitution | | `00D8` | name-entry blank helper |
| `00B5/00C7` | item name | | `00D9-$00DC` | town/location names (♂/♀/star classes) |
| `00BB/00C5` | person (NPC) name | | `00C4/00DF` | glyph-class controls |
| `00C3` | **furigana** (new in DQ6) | | `00B2` | (unknown; dumper open list) |

**Same-byte-different-meaning hazards vs DQ3** (the reauthor trap): `00D4` = tone-silent
in DQ3 but speaker-label suppress + opening 「 in DQ6; `00D9-DC` = town names in DQ6 but
tone registers/merchant/husband in DQ3. Full conflict table in
`DQ3_SFC_DIALOGUE_SYSTEM_MAPPING_Sep14_2026.md` §2.4.

### 1.3 Travel-mode routine map (documented, dqbook table 4.17)

| Address | Purpose |
|---|---|
| `$C02AE2` / `$C02B09` | travel text output type 1 / type 2 |
| `$C02B69` | msg ID → data address + start bit |
| `$C02BD4` | Huffman decode one character |
| `$C02C92` | main handler for one character encoding value |
| `$C029E6` | control-code dispatcher (runs ASM per code) |
| `$C02CB2` | draw one character into the `$7E9585` text-box buffer |
| `$C03255` / `$C033A0` / `$C03286` / `$C03325` | main font-data reader / per-char clear / metadata select / row reader |

---

## 2. Dialogue renderer (documented, dq6-sfc rom layout + dqbook)

### 2.1 Text-box VRAM pipeline

Dialogue font builds in **WRAM `$7E9585`** (0x700 bytes) then DMA to **VRAM `$4900`**
(`DMAPx=$00` 1-byte units, `BBADx=$18`, `VMAIN=$04` 8-bit address translation).
Characters are **16 px tall**; the tilemap stride is 32 bytes per pixel-row step
(2-Dimensional bit-plane layout); per-character variable width comes from font metadata.

### 2.2 Grouped variable-width font

Per group (5-byte metadata, `rom layout.txt` lines 26-45):

```c
struct DQ6FontGroup {
    uint16_t range;        // 12-bit count spanning bytes 0-1
    uint8_t  width_lo;     // byte1 bits: WWWW width | RRRR range-low nibble
    uint16_t data_ptr;     // bytes2-3: bank-$C1 LE pointer to group font data
    uint8_t  height;       // byte4 bits 4-7: HHHH height
};
```

Special glyph `0x200` = space (width 4, height 11). The font + script system carry over
from Otogirisou (Manabu Yamana, Heartbeat founder, was its programming director).

### 2.3 Box model & the 「 convention

3 lines per dialogue window; up to 3 `[AD]`-separated lines then wait; message speed 1/8
accelerates travel text (vanilla SFC speed affects only battle text). The opening 「 is a
**control code** (`0xD4`/`0xDE`), not a typed glyph; the closing `」` renders at string
end (Endo prints `\n」` after input-wait).

### 2.4 BRK = dialogue primitive (documented, dqbook 4.3.2.3)

The 65816 `BRK` is co-opted with a **2-byte operand** (nonstandard!) = travel message ID.
Handler `$C0FFA8` → `JMP $C59942` → pops status, `REP #$30`, fixes the return address (−2),
`JMP $C02E32`/`$C02E1C` outputting the ID. Example from the ROM:
`CA/1F17: 004500 BRK #$0045` = "モーッ モーッ！". **Every literal `BRK #$XXXX` in the
ROM is a message-ID pointer** — the event-script equivalent of the PSX `C021A0` command.

---

## 3. Talk-object engine (documented, dqbook 4.4)

| Table | Shape | Fields |
|---|---|---|
| Simple talk objects | `$FF08DA`, 0x0A B × 0x2BD | `$00` bits `$3FC0` sprite id / `$003F` ?; `$01` facing (0-3); `$02-03` MX/MY (`$01FF`/`$03FE`); `$04` LV altitude (`$001C`) + draw flag; `$05-06` walking-behavior routine; `$08-09` **message ID (LE 16-bit Huffman ID)** |
| Standard talk objects | `$FF243C`, 0x0B B × 0x74E | same + `$04` bit `$0040` **window-erase flag**; `$08-09` **talk handler (event script) pointer**; `$0A` bank byte |

Main talk routine **`$C0C9B9`**: survival check → find facing object at (MX±1, MY±1)
matching LV → rotate NPC to face player (facing codes 0-7 map U,U,R,R,D,D,L,L via
`(dir+4)&6`) → output message ID or jump to handler (`$0006B0`) → increment "talked"
counter `$7E5E8C` → honor window-erase → restore facing.
No-one-there = message `#$1777` ("[D4]その方向には 誰もいない。"); all-dead = `#$17D7`.

---

## 4. Maps, spatial hierarchy, sprites, compression

### 4.1 Map data

- **Uncompressed 4bpp graphics**: raw `0x1000C0-$22351F` (the art blob).
- **Compressed map/data = LC_LZ21** (Lunar Compress v2.00 "Another Enix format";
  spec inside `lc200/DLLcode/LunarDLL.cpp`; whether the LZ21 regions are maps/text is
  unconfirmed — voliol).
- **Overworld encounter-region maps** at `$89138`: 16×16 = `$100` bytes per map, one
  byte = encounter-set id (valid ~`$05-$73`; dungeons have their own sets).

### 4.2 Spatial hierarchy (dqbook 4.18)

| Table | Shape | Contents |
|---|---|---|
| `$C815BF` | 1 B per unit | spatial minimum unit (Floormi spell id → location); current unit at WRAM `$7E5ED9` |
| `$C8997A` | 0x0B B × 0xB8 | Zoom (Rula) destinations: name string, field class (`$01` upper / `$02` lower / `$04` seabed / `$08` はざま), Zoom coords, ship coords 0-2, flying-bed, gourd-island |
| `$C8A188` | 8 B × 0x93 | locations: field class, Tako-eye, Zoom index/behavior, Escape, coords, poison, loud-voice, name |
| `$C8B314` | 5 B × 0x1C9 | coordinate objects: spatial unit `$03FF`, coordinate-array `$000C`, MX `$1FF0`, MY `$3FE0`, LV `$00C0`, roof-region `$001F`, facing `$0060` |

Screen transitions: stairs `$FF8072`, horizontal band `$FF8D5A`, vertical band
`$FFA02A`, rectangle `$FFA0BA`, checkpoints `$FFB692`, destinations `$C834FC`.

### 4.3 Monster struct — 42 bytes (documented, voliol refined posts)

```c
struct DQ6Monster {          // 0x2A bytes at $C20154+ (entry 0 = blank)
    uint8_t  action[6][2];   // $00-$0B: 6 actions (even) + args (odd)
                             //   $42 normal attack, $F7 flee, $FD sultry dance,
                             //   $99 defend …
    uint8_t  initial_status; // $1C bits
    uint8_t  behavior;       // $1D ($02 = spotted slimes flee)
    uint8_t  image_id;       // $1E monster image (palette-swaps share images)
    uint8_t  max_mp;         // $1F
    uint8_t  drop_item;      // $20
    uint16_t palette_ref;    // $21-$22
    uint16_t max_hp;         // $24-$25
    uint16_t exp;            // $26-$27
    uint16_t name_ref;       // $28-$29 "0XXY" (LE) into the $3B8703 strings
};
```

Boss phases swap monster structs mid-battle to exceed the 6-action limit; image IDs
`$3C-$3F`/`$41-$42` = morph (class-change) sprites for Hero/Hassan/Barbara/Chamoro/
Terry-Milly/Amos. **Monster name strings are NOT Huffman** — they live in the
`$3B8703` raw-string area (`$AC`-terminated, referenced by "0XXY" LE refs;
group-start 3-byte relative offsets at `$C165E7`, immediately after the Huffman
pointer table's end — consistent layout measured: table extent ends `$C165E6`).

### 4.4 Battle text = raw small font (documented; measured table)

Pointer table `$C15AD1`, data base `$F6DEBD`; codes `[BC]` message start, `[B4]`
current monster name, `[B8]` enemy-group slot, `[AD]` break, `[AF]` wait-arrow, `[B2]`
name substitution (dqbook 4.3.1: message `#$01A1` = "[BC]なんと [B4]が
おきあがり[AD]なかまに なりたそうに…[AF]"). Measured this pass: first 6 pointer-table
entries + data head hex recorded in `HBD_FORENSICS_DATA.json`.

---

## 5. Byte-level verification performed this pass (tier-1)

1. **Tree tables at the documented addresses**: raw `0x167BE`/`0x1700E`, 1,064 entries
   each, root index `0x427`; **both root entries are inner nodes** (`0x884A`/`0x884C`,
   MSB set) under the DQ6 polarity (MSB set = inner, inverted vs DQ5) — 582/557 inner
   nodes counted across the two arrays.
2. **Pointer table shape**: 870 × 3-B entries read from raw `0x15BB5`; table is
   monotone; script base `$37175B`; 8 strings/pointer with skip-on-terminator chaining
   (`$00AC`/`$00AE`).
3. **Live string decode**: TID 0 chains decode to valid kana terminating at `$AC`
   (recorded in `HBD_FORENSICS_DATA.json → dq6_sfc.tid0_strings`).
4. **Known-plaintext caveat (documented honestly)**: the engine-doc anchors
   (message `#$1777` = "[D4]その方向には 誰もいない。") decode from the **JP** payload;
   our in-tree DQ6 cartridge is the DQ Translations repackage whose script payload was
   re-encoded, so JP-side known-message ID→text checks against THIS file diverge — the
   JP reference ROM (MD5 `ac9955fa…`) is required for the plaintext anchor run. The
   tree tables survived the repackage at their documented addresses (point 1), which
   is the reauthoring-relevant invariant.

## 6. Lineage position (bridging Papers 1-4)

| Dimension | DQ6 SFC (this paper) | → DQ3 SFC (1996) | → DQ4 PSX (Paper 4) |
|---|---|---|---|
| Dialogue | global tree `$C167BE/$C1700E`, 8 strings/ptr, LSB-first, leaf `≥0x200` glyphs | same engine, pointers moved (only), MSB-first | per-block dual-base trees, referrer words, LSB-first |
| Terminators | `00AC/00AE` | `00AC/00AE` | `0000` |
| Event hooks | BRK-as-message-ID (2-B operand) | inline-operand JSL VM (`$C4:2970`) | `C021A0` + `%A/%B` conditionals |
| Fonts | grouped variable-width (5-B metadata) | same family | in-EXE dual-glyph atlases |
| Compression | LC_LZ21 (maps) | HeartBeat LZSS (`$C0:4923`) | LZSSO (`0x0500`) in archive |
| Monster data | 42-B struct, 6 actions | banks `$F0-$F5` far-pointer frame tables | type-44 roster tables in HBD |

The HeartBeat chain (documented): Otogirisou (Yamana) → DQ5 → **DQ6** → DQ3 SFC → DQ7 →
DQ4 PSX. The DQ6 engine is the direct ancestor of the PSX dialogue system: same 2-B
node words, same leaf masking, same 8-strings-per-pointer economics (the referrer-word
packing generalizes it), and the control-code families (name/item/person/furigana)
survive into the PSX `7Fxx` space with the same semantics.

## 7. Open gaps

1. JP-reference-ROM verification run (known-message anchors, pointer table byte
   signature) — needs the No-Intro MD5 `ac9955fa…` file.
2. LC_LZ21 region typing (maps vs text) — unconfirmed (voliol); the format spec lives
   in `lc200/DLLcode/LunarDLL.cpp`.
3. DQ6 battle-mode raw-font code census beyond `[BC]/[B4]/[B8]/[AD]/[AF]/[B2]` — the
   bracket notation is documented; a full table dump is an open item.
4. Per-string width/pagination internals beyond the 3-line + wait model.

---

*Suite: Paper 1 (DW4 NES) · Paper 2 (DQ1+2 SFC) · Paper 3 (lineage) · Paper 4
(DQ7/DQ4 PSX) · Paper 5 (this). Regenerate measurements with `python hbd_forensics.py`
and `python whitepaper_forensics.py`.*
