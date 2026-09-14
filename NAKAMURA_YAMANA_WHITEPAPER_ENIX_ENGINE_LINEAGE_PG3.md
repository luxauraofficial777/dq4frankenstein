# THE ENIX ENGINE LINEAGE — Cross-Generation Comparative Analysis

**Paper 3 of 3 — Enix Engine Specification Suite**
**Doc ID:** VW-SNES-WP-003 · 2026-09-14
**Scope:** the architectural evolution from Chunsoft's 8-bit Famicom engine (DW1-DW4,
1986-1992) through Chunsoft's 16-bit transition (DQ5 SFC 1992, DQ I+II SFC 1993) into the
HeartBeat 16-bit engines (DQ6 1995, DQ3 SFC 1996), the 32-bit HeartBeat engine (DQ7 2000 →
DQ4 PSX 2001), and the DS sidestep (DQ4 DS 2007, Arsys/CREAM Nitro stack) — with the
storage, text, and AI-paradigm contrasts made byte-explicit.
**Evidence hierarchy:** (1) byte-level measurements taken for Papers 1-2 of this suite
(`WHITEPAPER_FORENSICS_DATA.json`); (2) verified-in-code reconstructions (s-endo decoders
`dq3decode.c`/`dq12decode.c`/`dq6decode.c`, DW1 commented disassembly, Osteoclave's DW4
Huffman dumper, RadMage's hardware-proven PSX FORMAT.md) cloned in `snes\clone\`;
(3) atwiki/DQバイナリ改造 + Data Crystal tables. Claims are labelled by tier.

---

## 1. The five engine generations

| # | Engine | Games | CPU / RAM | Defining mechanism |
|---|---|---|---|---|
| G1 | Chunsoft 8-bit "table era" | DW1-DW4 (FC/NES 1986-92) | 6502-class, 2 KB RAM + 8 KB battery WRAM, MMC1 bank-switched PRG | direct pointer tables, per-block Huffman (DW4), instruction-built maps |
| G2 | Chunsoft 16-bit transition | DQ5 SFC (1992), **DQ1+2 SFC (1993)** | 65C816 SlowROM, LoROM | raw byte text (DQ1+2: no compression at all), flat pattern-dispatch AI |
| G3 | HeartBeat 16-bit "Huffman era" | DQ6 (1995), DQ3 SFC (1996) | 65C816 FastROM, HiROM/ExHiROM | global 13-bit Huffman script + per-resource streams + event VM |
| G4 | HeartBeat 32-bit "archive era" | DQ7 (2000), DQ4 PSX (2001) | MIPS R3000, 2 MB RAM, CD | HBD1PS1D.Q41 container, LZSSO, per-block Huffman + referrer words |
| G5 | DS sidestep (2007-08) | DQ4 DS | Nitro (ARM7/ARM9) | NFTR fonts, .mpt message packs, full %A/%B/%H conditional grammar |

G2's DQ1+2 SFC is the bridge cartridge whose *absence* of Huffman is the datum that dates
the Huffman era's introduction to HeartBeat's own G3 engines. Chunsoft's DQ5 SFC also
predates the Huffman script (same engine family as G2 — the series convention
`0x83`/`0x82` trailing dakuten is shared, tier-2).

---

## 2. Storage paradigms: from bank-switching to virtual containers

### 2.1 G1 — everything is an address

The 8-bit engine has no data abstraction: every table is a raw ROM offset reached through
explicit bank arithmetic (measured, Paper 1):

- text blocks: u16 stream offsets (+$8000) with **bank-up-on-wrap** resolution (DW4);
- map data: u16 addresses, banks `$09-$0B`, bank-up-on-wrap (DW4, 73 maps/35 tilesets);
- monster records: fixed 22-byte strides; windows/PTs per bank.

Bank-crossing is handled by *protocol*, not by pointers: the loader serializes a 5-bit
MMC1 write and increments the bank when an address wraps. The battery WRAM holds save
blocks AND hot working sets.

### 2.2 G2 — still direct, wider cells

DQ1+2 SFC keeps the direct-table model with 16-bit cells and bank windows
(`0x4EA5F` item-price pointer table × 87; shared 122-record monster table at `0x5DA0E`;
zones at `0x5B52D`/`0x5BEC5`; all measured). No container, no compression, no indirection
beyond the table.

### 2.3 G3 — per-resource streams (the container precursor)

DQ3 SFC introduces **resource-level indirection**: pointer tables whose targets are opaque
compressed streams (measured, Paper-2 companion run):

| Resource | Pointer table | Target format | Status |
|---|---|---|---|
| Dialogue script | `$C1:5331`, 506 × u24 **bit-addressed** (21-bit byte offset \| 3-bit bit offset), base `$FC:C258` | 13-bit Huffman bitstream, 8 strings/pointer | **byte-verified** (worked example reproduced) |
| Map archives | candidate table at raw `0x9385`, 15 × u24 far (bank-$DD uniform family, 0x400 stride + one bank-$E8 target) | LZSS-class per the HeartBeat engine `$C0:4923` (1024-B window, init `$03BE`, 2-byte literals, 10-bit offset + 6-bit len+3) | **stream framing OPEN** — targets did not decode under the dialogue-stream model; container size-words undocumented |
| Monster frames | banks `$F0-$F5`, 163 monsters | monotone far-pointer tables → `$F6` descriptor streams | documented (tier-2) |
| Scene palettes | raw `0x370000 + k*0x200` | CGRAM image per scene | documented (tier-2) |

The architecture lesson: G3 already has "many streams behind one table" — the HBD
container is this, regularized.

### 2.4 G4 — the completed archive

`HBD1PS1D.Q41` (PSX, hardware-proven FORMAT.md, tier-2): 16-B main header; then 16-B
sub-block headers (`csize/dsize/offset`, **flags uint16 = 0x0500 compressed**, type
uint16) + payload. Types: 1 font · 6 map chips · 8/10/13 sprites · 21 qQES geometry ·
32 scene index · **39 Huffman dialogue script** · 42 linear text · 44 roster tables ·
26 records · **46 LZS overlay modules** (≥20.2% of the script lives in type-46 string
pools). Compression = HeartBeat LZSSO (LZSS, 4096-B zero-initialized ring, 8 flag
commands, offset=W>>4, len=(W&0xF)+3). The DQ3 SFC `$C0:4923` stream codec (§2.3) is this
engine's direct 16-bit ancestor — same LZSS family, smaller window, no container header.

### 2.5 The through-line

```
G1  bank arithmetic + raw tables            (no abstraction)
G2  + 16-bit cells, direct tables           (no abstraction)
G3  per-resource pointer tables + streams   (abstraction begins)
G4  container file with typed sub-blocks    (abstraction complete)
G5  standard platform containers (Nitro)    (abstraction outsourced)
```

---

## 3. Text dispatch evolution

### 3.1 The Huffman bloodline (table, tier-1 where measured)

| Generation | Game | Tree layout | Bit order | Node test | Strings/pointer | Verification |
|---|---|---|---|---|---|---|
| G1 | DW4 NES (US) | per-block bitstreams, 3-18 bit prefix codes | MSB-first bit reader | prefix-greedy | 32 lines/block, 86 blocks | **decoded this pass** (real lines reproduced) |
| G2 | DQ1+2 SFC | none — plain bytes + `0xD0-D2` kanji escape | — | — | — | **negative result, confirmed** |
| G3 | DQ3 SFC | 2 × 1002-entry u16-LE parallel arrays at `$C1:59D3/$C1:61A7`; root `0x3E9` | **MSB-first**; **bit=1 → `$161A7` table, bit=0 → `$159D3`** | MSB set = inner | **8 per pointer** (SID = TID×8 + slot) | **byte-verified**: ptr `79 12 00` → steps `011111101001` → leaf `0x0521` (ツ), exactly as documented |
| G3 | DQ6 | 2 × 1064-entry arrays `$C1:67BE/$C1:700E`; root `0x427` | **LSB-first**; MSB set = inner (polarity inverted vs DQ5) | MSB set = inner | 8 per pointer | **tree measured this pass** (1064 nodes; roots inner) |
| G4 | DQ4 PSX (HBD) | dual-base per-block tree (`pair[NN]`/`pair[m+NN]`), `tree_len = 4N+6` | per-block | `>= 0x8000` = inner | per-block referrer words `(id<<20)\|(bit+hts*8)` | hardware-proven (tier-2, FORMAT.md) |
| G4 | DQ7 | same family, global tree | per-block | `>= 0x8000` | — | same engine family (DQ4's direct parent) |
| G5 | DQ4 DS | NFTR font system | — | — | 6-byte pointers, UTF-8 | standard Nitro (tier-2) |

**Invariants carried across the whole bloodline** (each byte-proven where claimed):

1. Leaf values `≥ 0x200` = glyphs; `< 0x200` = controls (G3/G4 identical threshold).
2. Terminator: `0x00AC`/`0x00AE` (G3) → `0x0000` (G4); 8-strings-per-pointer and the
   3-byte `offset|bitoffset` pointer word survive G3 → G4 until the PSX generalizes them
   into per-block 6-I headers + packed referrer words.
3. Every G3 string ends at the next 8-boundary share — the exact physics that became
   the PSX `(id<<20)` packed referrers and its 4-bytes-per-symbol tree cost rule.
4. `0x7Exx` per-block dictionary references appear G1 (DW2 5-bit stream escapes) and G4
   (158 per-block refs) — the *dictionary idea never dies*, it only changes encoding.
5. Bit order is a **local property, not a family property**: DQ3 MSB-first vs DQ6
   LSB-first with inverted polarity — proof that the same node-table paradigm was
   re-tuned per game, and the single most dangerous cross-engine reauthor hazard.

### 3.2 Control-code evolution (tier-2 master library)

| Function | G1 FC (DW1/DW2/DQ4-FC) | G2 DQ1+2 SFC | G3 DQ3/DQ6 SFC | G4 PSX HBD | G5 DS |
|---|---|---|---|---|---|
| End of string | `$FC/$FF` | `0x00FF` | `00AC/00AE` | `0000` | EOS |
| Line break | `$FD` | `00F0/00FE` | `00AD` | `7F01/7F02` (EXE vs scene) | `0x0A` |
| Input wait | `$FB` | `007A` | `00AF` | `7F0A` | — |
| Name inject | `$F8`/`[FC]` | `00F4` | `00C9/00CA/00CB/00C0-CC` | `7F1F` + `7F20-7F2F` (15 names) + `7F30-33` | `%a…` / tags |
| Item/noun | `$F9`/`[FA]` | — | `00B5/00C7` | `7F16/7F18` receive pair, `7F4B` noun | item tag |
| Number/gold | `$F6/$F7` | — | `00BB/00C5/00DB` | `7F15` | `%H/%M/%O/%L/%D` |
| Speaker label | — | — | `00CD` (DQ3) / `00D4` (DQ6) | `7F04` | — |
| Tone register | — | — | `00D1-D4` (DQ3) / `00D9-DC` (DQ6) | `7F43/44/45` | — |
| Conditional text | — | — | — | `%A{ID}%X…%Z`, `%B{ID}%X…%Z`, `%H…%Ys` | full `%A/%B/%C/%H/%M/%O/%L/%D %X/%Y/%Z` |
| 2-byte glyph escape | — | `00D0-D2` (768-entry) | — | inline `FF xx` in `7Exx` phrases | UTF-8 |
| Dictionary escape | `$6D+` (DQ1) / 5-bit stream (DW2) / Huffman tokens (DQ4-FC) | — | — | `7Exx` per-block (158; block-local) | — |

Read vertically, the table is the thesis: **each generation adds one dispatch dimension
while preserving every earlier function** — name → item → number → tone → conditional —
until the DS grammar (`%A/%B/%C/%H/%M/%O/%L/%D`) completes the set that the PSX
conditional family (`%A/%B`) only partially implements.

### 3.3 The G2 "missing features" are lineage data

DQ1+2 SFC has **no** embedded pointer, no conditional, no tone, no pluralization, no
furigana, no Huffman — every one of those appears for the first time in G3/G4/G5. This
negative space is precisely how the engines interlock: G3's engine is not a refactor of
G2's renderer (different tree, different terminators, different pointer scheme), it is a
replacement — while its *control-code semantics* (name/item/number/wait families) stay
compatible with G1's design, inherited through Chunsoft's shared renderer ancestry.

---

## 4. Map, AI, and event-VM evolution

| Era | Overworld | Towns/interiors | Compression | AI/event VM |
|---|---|---|---|---|
| G1 DW1 | nibble RLE (tile:count-1) at `$1D6D` | 2 tiles/byte, `$F` = 2×2 metatiles | — | 19 TextBlocks × 16 entries, nibble-encoded |
| G1 DW2/3/4 | 256×256 plane maps, per-16×16 encounter bytes | **instruction-built maps** (turtle-graphics + push/pop, shared across 3 games; measured: 73 maps/35 tilesets) | — | 22-byte monster records; slot-ordering AI |
| G2 DQ1+2 | raw large tilemaps; 16×16 encounter zones + tile-level flee rule | raw tilemaps | none (raw) | 8 pattern slots → flat u16 dispatch at `$0B:BD8C` (byte-verified) |
| G3 DQ3 | 256×256 → 4×4 metatiles → 1×1 | palette/table refs (sprites `$2E560D+`, palettes `$36602C`, NPC `$808DA`, walkability `$37C74C`) | adaptive 0x400-entry RAM dictionary (`$7F0000`, flag = 8×2-bit commands); LC_LZ21 (DQ6) | **event VM**: inline-operand dispatch `$C4:28FE` (stack-return rewrite), 6-entry handler table `$C4:2970`, WRAM param FIFO `$7E40B5-A`; catalogue `$C42AD2` show-dialogue … `$C439C0` scripted battle |
| G4 PSX DQ4 | rotating GTE 3D grid + billboard 2D sprites | type-6 chipsets, type-21 qQES meshes streamed to heap `0x80138000` via DMA Ch4 (`open_file 0x80076040` / `mopen_file 0x80082598`) | LZSSO (flags `0x0500`) + type-46 LZS overlays | bit-packed event-flag matrix `0x8000F800-$80010000` (same AI-decision weights as SFC engines, 32-bit form) |
| G5 DS | NSCR/NCGR Nitro tiles | standard Nitro | Nitro built-ins | Nitro stack |

The G3 event VM is the direct conceptual parent of the G4 script VM: inline-operand
native-dispatch (65816 `JSL` + inline bytes, return-address rewriting `$C4:28FE`)
generalizes on MIPS into the byte-aligned 3-byte dialogue command `C0 21 A0 <offset>
<dialogId>` inside the HBD event stream. The AI weights persist across both: the SFC
pattern-slot dispatch (G2's `$0B:BD8C` jump table) and the PSX bit-packed flag matrix are
the same design in 16-bit and 32-bit form.

---

## 5. The monster-table thread (one continuous structure, four formats)

| Engine | Record | Pack highlights |
|---|---|---|
| DW4 FC/NES | 22 B | EXP u16; HP/ATK/DEF/GOLD 10-bit (8+2 spillover); 6 action slots, low-7-bit each; double-attack/HP-regen in slot bit-7s; 2-bit×9 resistances; opening-state/probability byte |
| DQ1+2 SFC | 18 B | AGI/STR/DEF/HP/MP/GOLD raw bytes; **8 AI pattern slots** (hi3 flags \| lo5 id); evade%/intelligence nibbles; XP u16 — shared DQ1+DQ2 table, ground truth Metal Slime `99 12 fe 04 06 06 e0 e5 e0 e5 e0 e5 00 06 00 00 07 03` |
| DQ3 SFC | — (frames are banks `$F0-$F5` monotone far-pointer tables; stats per wiki tables) | far-pointer per-monster frame tables |
| DQ4 PSX | type-44 roster tables + type-26 records inside HBD | 10-bit packed stat/resistance nibbles again — the FC record's packing survives into the PSX roster format |

---

## 6. Negative results / corrected myths (do-not-rechase list)

1. **DQ1+2 SFC text is not Huffman** (plain bytes + `0xD0-D2` kanji escape) — confirmed.
2. **DQ6 battle text is not Huffman** (raw small-font, base `$F6DEBD`); DQ5's dialogue
   Huffman polarity is inverted vs DQ6 — do not reuse one tree-walk for both.
3. **No FC→EN DQ4 fan translation exists**; the DW4 NES tools lineage is
   Osteoclave → Bongo` → abw → TheAnsarya (all verified in-tree).
4. GBC remakes are **TOSE** — a separate lineage, not Chunsoft/HeartBeat ancestry.
5. The hidden 5-language script lives in the **DS JP ROM**, not NES/FC.
6. The `$B3A4` "DTE dictionary" (dragon-warrior-4-info) is a probable Huffman-node
   misreading — the main script is 3-18-bit prefix-coded, not byte DTE (measured).
7. DQ3 map-archive **container framing is OPEN**: the candidate pointer table measured
   at raw `0x9385` (15 × u24 far, bank-$DD family) did not decode as LZSS — stream size
   words are undocumented; do not claim a byte-verified archive container for the JP
   original until that framing is traced.

---

## 7. What this lineage means for the consumer project

1. **The Sovereign Huffman engine's spec is DQ3 SFC's** — now byte-verified end-to-end in
   this pass (tree walk, pointer decode, string chain, worked example). Unit-test hbe's
   parser against `dq3decode.c` + `dq3mining/text.py` for edge cases: bit order, MSB
   polarity, 8-string pointer strides, per-block root re-initialization.
2. **The referrer word and 8-boundary physics** are G3→G4 continuous: any change to the
   packed `(id<<20)` referrers must respect the 8-boundary share rule.
3. **The DS grammar table is the authoritative spec** for the PSX `%A/%B` conditional
   family; the SFC tone-band codes (`00D1-D4` / `00D9-DC`) and the PSX tone band
   (`7F43-7F45`) are the same feature family at three points.
4. **The `[row][plane]` tile interleave** (G2) vs `[plane][row]` SNES standard vs the
   NES `[tile,count]` and instruction-built streams — any cross-era asset pipeline must
   encode platform-specific byte orders explicitly (the `world_builder` codecs already
   do; keep them canonical).
5. **Integrity posture**: ROM checksums are the only verified integrity checks (SFC
   header complement⊕checksum = `0xFFFF` measured twice); save-block checksums remain
   open for both G1 (DW4 battery WRAM) and G2 (8 KiB SRAM) — the honest gap list.

---

*Suite: Paper 1 (DW4 NES), Paper 2 (DQ1+2 SFC), Paper 3 (this lineage). Regenerate all
measurements with `python whitepaper_forensics.py`. Every claim is tier-labelled;
byte-proven results are reproducible from the cartridges in `famicom\`.*
