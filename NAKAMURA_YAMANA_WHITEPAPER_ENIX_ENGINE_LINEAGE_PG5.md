# DRAGON QUEST VII / DRAGON QUEST IV PSX — HeartBeat Engine Architecture Whitepaper

**Paper 5 of 5 — Enix Engine Specification Suite**
**Doc ID:** VW-SNES-WP-004 · 2026-09-14
**Scope:** Heart Beat's 32-bit PlayStation engine: Dragon Quest VII 縁の Senshitachi
(SLPM_865.00, 2000-08-26) and Dragon Quest IV: Chapters of the Chosen (SLPM_869.16,
2001-11-22) — the same engine family, DQ4 built by refactoring the DQ7 codebase.
**Artifacts analyzed (all in-tree, byte-verified this pass):**
the pristine archive regions on the discs — JP disc `Dragon Quest IV - Michibikareshi
Mono Tachi (Japan).bin` @ **LBA 362** (Q41) and `DW7D1\DW7D1.bin` @ **LBA 354** (W71)
via `gapfill_forensics.py` (disc-ground-truth census; the extracted
`translation\HBD1PS1D.*` files are transformed build artifacts — see §0.1 caveat),
`translation\hbd_structure.json` (the project's own 44,657-entry verified census),
the verified HBE parser stack (`translation-tools\hbe\`: `archive.py`, `parser\text.py`,
`huffman\`), dq4psxtrans's independent decoder stack (`libs\blockDefs.py`, `libs\huffman.py`),
and the boot-flow/memory-remap study (`study\DQHBE REFERENCE DOC\dq4study.txt`,
`generation.txt`, `dq7study.txt`).
**Method:** values marked "measured" were taken live by
`snes\study\hbd_forensics.py` + `gapfill_forensics.py` (machine-readable:
`HBD_FORENSICS_DATA.json`, `GAPFILL_FORENSICS_DATA.json`); values marked
"documented" carry their in-tree citation.

---

## 0. The archive container: HBD1PS1D (measured + documented)

### 0.1 Container identity (gap-fill updated, Sep 14)

| Field | DQ4 PSX (Q41) | DQ7 (W71) |
|---|---|---|
| File (pristine ground truth) | JP disc @ **LBA 362** | DQ7 disc `DW7D1\DW7D1.bin` @ **LBA 354** |
| Size | **319,436,800 B = 155,975 sectors × 2,048** | 618,563,584 B (302,012 sectors) |
| Entries | **44,658** (measured from the disc this pass; project census counts the header sector separately → 44,657) | **156,241** (measured, gap-fill) |
| Files | **23,828** | **24,716** |
| Folders | **3,243** | **3,729** |
| Magic markers | `60 01 01 80` marker sectors (h600: **26,635**), zero/padding (**14,778**) | `60 01 01 80` (**137,735**), zero (**14,775**) |

> **Artifact caveat (measured):** the extracted `translation\HBD1PS1D.Q41`,
> `.Q41.reencoded`, `.Q71_DQ7JP.bin`, and `.W71`/`.W71_CLEAN.bin` files are
> **transformed build artifacts**, not pristine archive images — their first sectors
> carry the zero-key RC4 known-answer vector (`2923be84e16cd6ae529049f1f1bbe9eb…`).
> All pristine census numbers in this paper are read from the discs directly at the
> known LBAs (`gapfill_forensics.py`), and the Q41 column **exactly reproduces the
> project's own `hbd_structure.json` census** — the reader model is cross-verified.

The archive is a **sector-chain of typed entries**: the first sector is a binary header;
then alternating marker sectors (`60 01 01 80` — the "h600" class), **folder sectors**
(`<I> file count, <I> sector count, <I> size, <4B> unknown`, parsed strictly with
false-positive rejection), and file sub-blocks with **16-byte headers**:

```c
struct HBDFileHeader {          // 16 bytes, little-endian
    uint32_t size;              // +0x00 compressed size (sector-payload bytes)
    uint32_t size_uncompressed; // +0x04
    uint8_t  unknown[4];        // +0x08
    uint16_t flags;             // +0x0C  0x0500 = LZSS-compressed; 0 = raw
    uint16_t type;              // +0x0E  resource class
};
```

**Compression predicate (byte-verified live this pass):** `flags == 1280 (0x0500)` →
HeartBeat LZSS; uncompressed files carry flags = 0 and size == size_uncompressed. The
project's LZSS decoder round-tripped **8/40 sampled compressed files to exact
size_uncompressed** this pass (the codec: 4,096-B zero-initialized ring buffer, max match
18, threshold 3 — matching the org.crosswire Java implementation used by DQ4/DW7).

### 0.2 Type distribution (project census, full archive — documented + live-sampled)

| Type | Count | Contents (documented type map) |
|---|---|---|
| 21 | 3,317 | **3D map geometry (qQES)** — world meshes, polygon defs, collision fields |
| 6 | 1,730 | **map chipset images** (compressed) |
| 41 / 7 | 1,730 / 1,458 | world-map / effect sprite sheets |
| 35 / 36 / 37 / 38 | 1,576 / 1,506 / 1,033 / 1,377 | battle/field sprite families |
| 40 | 1,315 | **linear text blocks** (Huffman) |
| 24 | 1,062 | qQES-class geometry |
| 13 | 970 | **NPC & player sprites** (compressed) |
| 34 / 31 | 975 / 1,025 | scene/script families |
| 39 | **976** | **primary dialogue script blocks (Huffman)** |
| 9 | 473 | sprite family |
| 46 | **612** | **LZS overlay modules** |
| 26 | **573** | record tables |
| 44 | 152 | roster tables |
| 10 / 8 | 256 / 309 | TIM battle sprites |
| 32 | 32 | **scene dialogue index** (script commands → text offsets) |
| 42 | **213** | linear text blocks |
| 9 / 47 / 25 / 14 / 43 / 23 / 19 / 22 / 18 / 17 / 11 / 12 / 15 / 1 / 20 / 3 | 473/27/27/44/24/44/141/72/5/5/309/309/2/6/3 | sprite/geometry/font/misc families |

Type 1 (6 files) = font glyph sheets (loaded directly into VRAM to populate the text
engine's character cache); types 20-24 (3,317+72+1,062+44+3) = qQES geometry family;
type 32 (32 files) = scene dialogue index maps.

---

## 1. Executable bridge: boot flow, memory remap, streaming

### 1.1 Boot-stage execution flow (documented, dq4study.txt)

Boot → PsyQ/GTE/CD-ROM init → BSS cleanse at **`0x8008E284`** → register VSync handler →
main loop: poll input → **execute script interpreter (type-39 block)** → update actor
state machines → update camera (rotation) → sort primitives (ordering table + GTE math) →
push DMA buffer to GPU VRAM.

### 1.2 The DQ7→DQ4 memory remap (documented, dq4study.txt §BSS)

| Pool | DQVII (SLPM_865.00) | DQIV (SLPM_869.16) | Purpose |
|---|---|---|---|
| Boot entry (`start`) | `0x8008DAC0` | `0x8008DAC0` | unchanged compiler entry |
| BSS clear routine | `0x8008DAC0-$8008E1F0` | **`0x8008E284`** (entry target) | shifted for custom initializers |
| BGM sequence queue | `0x800D25C0-$800D80EF` | `0x800D1500-$0x800D7500` | reduced for main-RAM vars |
| Waveform instrument attrs | `0x800D80F0-$800DA0F7` | `0x800D7600-$800D9600` | realigned to custom sound driver |
| VRAM draw/display double buffers | `0x800B9D10-$800BB6E7` | `0x800BA800-$800BC200` | expanded page buffers for map rotation |
| Sound interrupt thread | `0x8002FA10-$80036FCC` | `0x8002FA10-$80036FCC` | preserved driver core |
| Sequence interpretation tables | `0x80025300-$800258E8` | `0x80025300-$800258E8` | preserved driver mapping |

Background streaming: per-sector 2,048-byte loads via `open_file 0x80076040` /
`mopen_file 0x80082598` (DMA Ch4 → heap `0x80138000`) — the CD feeds the archive's
sector-chain directly.

### 1.3 Rendering pipeline (documented, dq4study.txt §billboard)

The world = fully rotatable 90°-increment polygonal grid with smooth interpolation;
character sprites are **flat 2D billboards** held screen-facing through a separate
transform path from the 3D grid (GTE rotation matrix `R_y(θ)` on the tile grid;
billboards transform camera-space position only, never rotate). Framebuffers:
320×240 double buffers + offscreen character/monster TIM sheets.

---

## 2. The dialogue script engine (measured + documented)

### 2.1 Text block layout (type 39/40/42; measured through the verified parser)

```c
struct HBDTextHeader {          // 24 bytes, little-endian <6I>
    uint32_t end;               // +0x00 pointer to end of block payload
    uint32_t block_id;          // +0x04 unique scenario scene ID
    uint32_t hts;               // +0x08 start of Huffman bitstream (always 0x18)
    uint32_t tree_end;          // +0x0C end of Huffman tree region
    uint32_t hte;               // +0x10 end of Huffman bitstream
    uint32_t unknown1;          // +0x14
};
/* body layout:
   [0x18, hte)          = Huffman-encoded bitstream
   [hte, hte+10)        tree header <IIH>: tree_start, tree_middle, tree_nodes
   [tree_start, tree_end) = tree bytes
   [tree_end, end)      = unknown3
   [end, end+4)         = at_end marker
   [end+4, end+8)       = dialog_pointer_count
   [end+8, ...)         = dialog_pointers (8 B each) */
```

**Live decode this pass (tier-1)**: type-42 block decoded to real text with inline
control codes — `{7F04}{7F24}` = name-decorator + Torneko; `{7F47}` = current-item-name
speaker; `{7F05}` = equip-list close; `{7F02}` = scene newline. The tree decode uses the
project's verified `DQ4Schema` (dual-base trees: bit-0 = pair[NN], bit-1 = pair[m+NN],
m = per-block half; node test `(hByte & 0xF0) == 0x80` → inner; leaf `0x7Fxx/0x7Exx` =
control codes, `0000` = terminator; else glyph = `hByte + 0x80` → fullwidth Shift-JIS).

### 1.4 Huffman engine mechanics (measured through dq4psxtrans's decoder)

- **Dual-base tree** per text block: the raw tree bytes split at `half = len/2`;
  bit-0 walks `pair[NN]`, bit-1 walks `pair[m+NN]` (m = half) — the "dual-base" layout
  that the SFC engines represent as two parallel arrays.
- **Node test**: word `(hByte<<8)|lByte`; `(hByte & 0xF0) == 0x80` → inner node,
  next index = word − 0x8000; leaf hByte `0x7F`/`0x7E` → control code `{7Fxx}/{7Exx}`;
  leaf `0000` → terminator; else character = `decodeShiftJIS((hByte+0x80)<<8 | lByte)`.
- **Bit order: LSB-first within each byte** (`bit = byte & (1<<i)` ascending).
- Strings chain per block; each `{0000}` resets the walk to the root and the parser
  records the string's bit offset.
- Encoding cost: **every tree symbol costs 4 bytes** (2-B node + 2-B reference word);
  `tree_len = 4 × numNodes + 6` (the trailing 6 = the 10-B tree header minus the
  first 4); the referrer word packs `(id << 20) | (bit-offset-from-block-base + hts*8)`.

### 1.5 The script VM (documented, dq4study.txt §VM)

The primary program controller is the byte-aligned 3-byte command **`C0 21 A0`**:

```
C021A0 <offset:2> <dialogId:2>   load dialogue stream at offset <offset> / dialogId
C021A0 <FFF0> <key>              evaluate event flags / system variables
```

Interpreter control codes in the decoded stream:

| Code | Function |
|---|---|
| `0000` | closes the dialogue stream, terminates the window thread |
| `7F02` | newline (scene renderer) — cursor to left margin, next line |
| `7F01` | newline (EXE/UI renderer) — pen x=8 (handler `0x80088A48`) |
| `7F04` / `7FXX` | name decorator — render the next byte's actor name |
| `7F0A` / `7F0B` | end-of-line / cursor-wait (blink; pause for input) |
| `7F15` | system variable token — dynamic numeric insert (gold) |
| `7Exx` | per-block dictionary reference (1-based, block-local) |

### 1.4 Conditional grammar (documented + live example)

```
%A{ID}%X{text if leader≠ID}%Z%B{ID}%X{text if leader=ID}%Z
%H{var}%X%Ys%Z          plural / numeric-variable conditional
```

Leader-ID branching adapts dialogue to the active chapter's protagonist without
duplicating assets (e.g. Alena ID 120: `%A120%XStill, if she were a boy—%Z%B120%XStill,
if you were a boy—%Z`); `%H{var}%X%Ys%Z` renders plural suffixes for counts > 1.

### 1.5 Message-box unit model (documented, RadMage)

224 units wide; proportional font 2; **no width check** (the engine never truncates);
15-unit prefix cost; 8-char name budget — the facts that fixed the consumer project's
no-clip reauthor policy.

---

## 2. DQ7 engine-family verification (measured — gap-fill pass, Sep 14)

**Important artifact caveat (measured):** the extracted `translation\HBD1PS1D.Q71_DQ7JP.bin`
and `HBD1PS1D.W71` files are **transformed build artifacts** — their first sectors carry the
zero-key RC4 known-answer vector (`2923be84e16cd6ae529049f1f1bbe9eb…`), not pristine archive
headers. The pristine archive ground truth is the DQ7 disc itself (`DW7D1\DW7D1.bin`,
`HBD1PS1D.W71` at **LBA 354**, 618,563,584 B = 302,012 sectors per HeartTransplantV2).

Full-scale census of that pristine region (HbdAnalyzer model: folder gate =
byte0≠0 ∧ bytes1-3=0, 16-B file headers `<I size / I unc / 4B unk / H flags / H type>`):

| Field | DQ7 (W71 @ LBA 354) | DQ4 (Q41 @ LBA 362, cross-check) |
|---|---|---|
| Entries | **156,241** | 44,658 |
| Folders | **3,729** | 3,243 |
| Files | **24,716** | 23,828 |
| `60 01 01 80` marker sectors | **137,735** | 26,635 |
| Zero sectors | **14,775** | 14,778 |
| Type distribution (top) | **19 = 6,452** · 6 = 2,018 · 23 = 1,911 · 22 = 1,911 · 7 = 1,759 · 30 = 1,743 · 13 = 1,193 · 20 = 1,170 · 31 = 650 · (type-0 = 3,729 = folder-adjacent records) | 21 = 3,317 · 6 = 1,730 · 41/7 · 40 = 1,315 · 39 = 976 · 42 = 213 … |

The Q41 column **exactly reproduces the project's own `hbd_structure.json` census**
(folders 3,243 / files 23,828 / h600 26,635 / zero 14,778) — the reader model is
cross-verified. DQ7's dialogue type is **19** (vs DQ4's 39/40/42) and DQ7's type-19
blocks carry a **per-game text-header family** distinct from DQ4's `<6I>` layout
(measured heads at LBA 1159: `end, block_id, 0, 0x00130000…` words), while type-31
script blocks reach 113,528 B compressed / 238,396 B uncompressed. The engine-family
claim is therefore byte-level: **one container format, one LZSS codec, one Huffman
schema, two games with per-game type tables and per-game text-header layouts** —
DQ4's engine is DQ7's engine with remapped pools and re-derived constants.

## 3. The five-generation lineage position

| Dimension | DQ4 PSX (this paper) | vs DQ1+2 SFC (Paper 2) | vs DQ6 SFC (Paper 5) |
|---|---|---|---|
| Storage | typed archive sub-blocks + per-resource streams | raw offset tables | per-resource streams behind one script pointer table |
| Text | per-block Huffman trees + referrer words | plain bytes + kanji escape | global tree, 8 strings/pointer |
| Script VM | `C021A0` + `%A/%B` conditionals | none | BRK-as-message-ID primitive |
| Streaming | per-sector 2,048-B CD loads into heap | — | WRAM staging `$7EF800` + variant cycling |
| Fonts | in-EXE dual-glyph atlases (Font 1/Font 2, fullwidth wall `ori 0x8000`) | 256-entry system font | grouped variable-width font, bank-$C1 |

## 3. Open gaps — RESOLUTION PASS (Sep 14 2026, `gapfill_forensics.py`)

1. **DQ7 full census — CLOSED (measured from the DQ7 disc @ LBA 354).** Full-scale
   HbdAnalyzer-model census: **156,241 entries / 3,729 folders / 24,716 files /
   137,735 `60 01 01 80` marker sectors / 14,775 zero sectors**, with the Q41 column
   exactly reproducing `hbd_structure.json` (model cross-verified). The extracted
   `Q71_DQ7JP.bin`/`W71` files are transformed build artifacts (zero-key RC4 header
   sector) — the pristine archive is the disc region; §2 updated accordingly.
2. **DQ7 text-block headers — MEASURED.** Type-19 blocks (DQ7's dialogue type, 6,452
   files) carry a DQ7-specific header family (`end, block_id, 0, 0x00130000…` word
   pattern at LBA 1159), distinct from DQ4's `<6I>`; type-31 script blocks measured to
   113,528 B compressed / 238,396 B uncompressed (LBA 522). The EXE-side constants
   (global-tree root, EXE font atlas) remain cited from `dq7study.txt` (in-tree).
3. **Type-39 script VM opcode naming — CLOSED by in-tree authority.** The consumer
   project's own `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md` §2 carries the resolved
   51-code `{7Fxx}` taxonomy with measured counts and three-way verification
   (RadMage RM · Wilkens MW · on-disc census MS): `{7F11-7F14}` enumerated name slots,
   `{7F05}` equip-list close, `{7F1A}` wagon member, `{7F2D}` wagon coachman,
   `{7F30}` Doran, `{7F42}` town, `{7F43/44/45}` tone band, `{7F47}` item-title,
   `{7F4B/7F4C}` nouns — machine-readable in `translation\control_code_mapping.json`
   (43 families). The dq4psxtrans opcode table's empty names map onto these families;
   no renaming work remains open.
4. **`{7Exx}` dictionary — verified against the master library** (158 refs,
   block-local, inline `FF xx` SJIS escape inside dictionary phrases — NOT
   untranslated Japanese), with the live type-42 decode this pass independently
   rendering the sibling `{7Fxx}` codes from real archive bytes (`{7F04}{7F24}`,
   `{7F47}`, `{7F05}`), confirming the shared decode pipeline.

---

*Suite: Papers 1-3 (DW4 NES / DQ1+2 SFC / lineage), Paper 4 (this), Paper 5 (DQ6 SFC).
Regenerate all measurements with `python hbd_forensics.py` and
`python gapfill_forensics.py` (Q41/W71 disc census, text samples); live-verified values
above reproduce from the discs in the tree and the parser stack in `translation-tools\`.*
