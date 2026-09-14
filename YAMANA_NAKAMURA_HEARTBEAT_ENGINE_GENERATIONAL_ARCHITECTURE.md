# YAMANA–NAKAMURA HEARTBEAT ENGINE
## Generational Architecture: From the Super Famicom String VM to the PlayStation TID/SID Dispatch Machine

**Document ID:** VW-DQLOST-TECHRPT-007 · Rev 1.0 · 2026-09-13
**Author:** Big Pickle, for the VoidWalkers Research Project — *Dragon Quest IV: The Zenithian Chronicles*
**Engineering Lead & System Architect:** Lux Aura
**Repository:** [`luxauraofficial777/dq4frankenstein`](https://github.com/luxauraofficial777/dq4frankenstein) (remote HEAD `aecc8523`)
**Target Binary:** Japanese PlayStation 1 `SLPM_869.16` / `HBD1PS1D.Q41` (2001 HeartBeat/Koichi Nakamura build)
**Peer Formats:** `DQLOSTTRANSLATION/study/`, `DQLOSTTRANSLATION/snes/`, `DQLOSTTRANSLATION/shipB/`, `DQLOSTTRANSLATION/shipC/`, `DQLOSTTRANSLATION/voidpatcher/`
**License:** CC BY-NC-SA 4.0
**Status:** Rebuild C playtest-ready master `FF1AF239`

---

## Table of Contents

1. [Executive Abstract](#1-executive-abstract)
2. [Lineage: Nakamura → Yamana](#2-lineage-nakamura--yamana)
3. [Generation I — The Super Famicom Baseline (16-bit)](#3-generation-i--the-super-famicom-baseline-16-bit)
4. [Generation II — The PlayStation Paradigm Shift (32-bit)](#4-generation-ii--the-playstation-paradigm-shift-32-bit)
5. [The Messaging VM: String Tables, Control Codes, and the TID/SID Dispatch Cycle](#5-the-messaging-vm-string-tables-control-codes-and-the-tidsid-dispatch-cycle)
6. [Compression Containers: Huffman HTS-0x18 and LZSS/DQLZS Bitstreams](#6-compression-containers-huffman-hts-0x18-and-lzssdqlzs-bitstreams)
7. [Overlay Residency, Swapping, and the 2 MB Working Set](#7-overlay-residency-swapping-and-the-2-mb-working-set)
8. [Rebuild C Engineering Lessons: Desync, Handshake, and Budget Failures](#8-rebuild-c-engineering-lessons-desync-handshake-and-budget-failures)
9. [Re-Authoring Architecture: The 100% Native In-Place Pipeline](#9-re-authoring-architecture-the-100-native-in-place-pipeline)
10. [Verification, Gate Calculus, and the FF1AF239 Master](#10-verification-gate-calculus-and-the-ff1af239-master)
11. [References — Local Study Corpus and GitHub Publication Corpus](#11-references--local-study-corpus-and-github-publication-corpus)
12. [Appendix A — Hex-Level Structural Layouts](#12-appendix-a--hex-level-structural-layouts)
13. [Appendix B — Census Tables](#13-appendix-b--census-tables)

---

## 1. Executive Abstract

The HeartBeat Engine is not a single program. It is a *lineage*: a message-and-script virtual machine first engineered by Manabu Yamana's HeartBeat team on the Super Famicom for *Dragon Quest III* (1996) and *Dragon Quest VI* (1995), and later recompiled, retargeted, and crowded onto the PlayStation's MIPS R3000A for *Dragon Quest VII* (2000) and *Dragon Quest IV* (2001). The translate flows from artifact to artifact: token-compressed character streams on the 65816 and a hardware-addressable string VM on the 5A22; on the 32-bit side, a flexible text-machine driven by two 32-bit identifier families — the **TID** (Text Identifier) and **SID** (String Identifier) — a handle that survived directly into *Dragon Quest IV*'s `HBD1PS1D.Q41`.

This document presents the generational architecture of that engine at hex- and instruction-level fidelity, as recovered under the VoidWalkers Research Project. It covers (1) the Nakamura-era 16-bit string VM — banking, PPU transfer, and the physical dialogue tables of DQ3/DQ6 SFC; (2) the Yamana-era 32-bit rebuild — the MIPS R3000A memory choreography, DMA discipline, dynamic overlay banking, and the `TID/SID → referrer word → Huffman/LZ block` retrieval path; and (3) the Rebuild C reverse-engineering lessons — Shift-JIS lead-byte desync at `0x800F4DF0`, the Auld Well LBA `19082–19118` streaming handshake, the `0x047B` facility jump-table segregation, and the decompressor overrun contract (`≤ +3` decoded bytes) that every generator in the pipeline must respect.

The work is published in two interlocked corpora: the local repository under `DQLOSTTRANSLATION/study/` and its public mirror at `https://github.com/luxauraofficial777/dq4frankenstein`, whose contents are enumerated and cited in [§11](#11-references--local-study-corpus-and-github-publication-corpus).

All sector, LBA, hash, and address figures below are taken from the shipped artifacts: the `DQ4_REBUILD_B.vpbin` payload manifest, the `FF1AF239` Rebuild C master, the `YAMANA_HBE_MASTER_*_LIBRARY` registers, and the live disassembly corpus.

---

## 2. Lineage: Nakamura → Yamana

### 2.1 Two engineers, one message machine

| | Nakamura era (16-bit) | Yamana era (32-bit) |
|---|---|---|
| Console | Super Famicom (5A22/65816) | PlayStation (MIPS R3000A) |
| Reference titles | *DQ III* SFC (1996), *DQ VI: Maboroshi no Daichi* (1995) | *DQ VII* (2000), *DQ IV* PS1 (2001) |
| Per-chapter code | `dq3_sfc.asm`, `dq6_sfc.asm` | `HBD1PS1D.Q41`, `SLPM_869.16` |
| String engine | Bank-resident physical dialogue tables | `TID`/`SID` logical name spaces over overlay banks |
| Text compression | Token/macro-gapped Huffman (DQ6 schema) | Heterogeneous: Huffman `HTS 0x18`, LZ77-family (dqlzs), raw geometry |
| SW personality | Chapter-managed narrative VM | Multi-armed messaging VM (`3d`-family opcodes) |

The continuity thesis — and the finding on which this paper rests — is that HeartBeat did **not** discard its Super Famicom messaging approach when it moved to the PlayStation. It *re-homed* it: the SFC engine's "character stream + control embedded in stream" model becomes, on the R3000A, the "character stream addressed by a 32-bit compound `TID`/`SID` handle, decompressed through an engine-wide Huffman tree, dispatched through a 51-opcode control VM." The historical lineage is documented in full in `HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE.md` and the `CASE_STUDY_NAKAMURA_YAMANA_HEARTBEAT_ENGINE_IEEE_2026.md`.

### 2.2 What the 2001 PS1 situ pick failed to deliver, and why

Between 2001 and 2025 no hardware-accurate English home release of *DQ IV* PS1 existed. Historical attempts grafted the North American *Dragon Warrior VII* executable onto the *DQ IV* disc — the "Frankenstein" grafting approach. The failure is architectural, not textual: the *DW7* binary drives different overlay residency vectors, different memory-card layouts, and a different `type-46` duplication set (`67 module types / 267 sites`) than `HBD1PS1D.Q41`. Threads collide, MIPS pipeline stalls, and camera scripts desynchronize. The VoidWalkers project therefore rejects grafting in favor of a **100% native in-place re-authoring engine** against the authentic Japanese HeartBeat binary with **zero sector shift**, documented at `FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md`, `HBD_PROCESSING_BLUEPRINT_Jul19_2026.md`, and `HLRM_INTEGRATION_BLUEPRINT.md` in the public corpus.

---

## 3. Generation I — The Super Famicom Baseline (16-bit)

### 3.1 ROM topology and banking

The SFC optical heritage is fully mapped in `DQLOSTTRANSLATION/snes/`:

| Title | Cartridge type | Header | Notes |
|---|---|---|---|
| *DQ VI* SFC | HiROM, 4 MiB, FastROM | `$FFD5` LoROM bank `00` clearing | 8 KiB SRAM save |
| *DQ III* SFC | HiROM, 4 MiB | Map mode 0x30-class sibling | Source for ExHiROM host |
| *DQ1+2* SFC | LoROM, 1.5 MiB | `$FFD5 = 0x20`-class | Dual-cart compilation host |
| **DQ3 SFC re-authoring host** | **ExHiROM, 6 MB, FastROM** | **`$FFD5 = 0x35`** | Zenithian Forge Track B |

The 6 MB ExHiROM decision (`0x35`) grants the translator ~1.5 MB of free expansion above the base 4 MiB image at the cost of a FastROM switching write (`STA $420D`) on every bank activation — the archival note that became `snes/docs/HOST_DECISION.md`.

### 3.2 The 65816 string VM

All SFC-target strings live in **bank-resident physical tables** — there is no TID/SID indirection on the 65816. The engine resolves:

```
effective = Bank << 16 | (base + tokenIndex)
writes char → VRAM buffer (BG1/BG3 area)
```

- **Control codes** inhabit the stream as single- or double-byte escapes; the SFC engine family's code space is enumerated in `snes/docs/CONTROL_CODES_SFC_ENGINES.md` and cross-referenced against the PS1 51-opcode space in `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md`.
- **Chapter management** is table-driven (`snes/src/chapter_manager.asm`), the direct ancestor of the PS1 `type-39` script module.
- **Warp/collision** live in `snes/src/warp_collision_table.asm`.
- **Symbols** are centralized in `snes/src/symbols.asm` — the 65816 analogue of a linker map.

### 3.3 PPU path and DMA choreography

Lines are committed to the display via three DMA channels under NMI:

| Channel | Target | Payload |
|---|---|---|
| 7 | `$2104` (OAMDATA) | OAM sprite tables |
| 0 | `$2118` (VMDATAL) | VRAM tilemap/BG3 |
| 1 | `$2122` (CGDATA) | CGRAM palette rows |

PPU controller space occupies `$2100–$2122`; the reset/bootstrap handler is at `$00:FF98` (hardware vector `$00:FFE4`). `Mode 7` transforms are available but the messaging path uses only BG1/BG3.

### 3.4 Tile, map, and LOCN formats

The tile/map/LOCN schemas are documented at `snes/docs/TILE_FORMAT.md`, `snes/docs/MAP_FORMAT.md`, and `snes/docs/LOCN.md`. Two properties matter for the cross-generational story: (1) tile indices are 8×8, 2-bpp/4-bpp planed, indexed into 256-color CGRAM rows; (2) the "LOCN" record is the SFC ancestor of the PS1 `position/roster` record at `0x800F83C0`.

### 3.5 DQ6 Huffman schema (the SFC compression ancestor)

DQ6 SFC already compresses its dialogue with a **Huffman packing** layer — the direct ancestor of the PlayStation `HTS 0x18` engine. The difference is capacity: SFC trees are per-package and small; the PS1 engine hoists **one global tree** at `0x800EF1C8` (see [§6](#6-compression-containers-huffman-hts-0x18-and-lzssdqlzs-bitstreams)). The full DQ6 tree layout is reproduced in `snes/docs/dq6_sfc_huffman.md`.

---

## 4. Generation II — The PlayStation Paradigm Shift (32-bit)

### 4.1 MIPS R3000A, the 2 MB ceiling, and caching

The PS1's CPU runs at 33.8688 MHz with a unified 2 MB DRAM working set and 4 KB I-cache / 4 KB D-cache (not a real split for these purposes). The engine's critical region is **KSEG0** (`0x80000000+`): cacheable, and therefore where all message decompression runs on hot paths. Uncached KSEG1 (`0xA0000000+`) is reserved for CD-ROM/DMA register work.

### 4.2 Fixed-residency memory map (from `RAM_DUMP_FINDINGS_Sep13_2026.md` and the disassembly corpus)

| KSEG0 address | Resident object |
|---|---|
| `0x80010000` | MIPS entry / engine boot |
| `0x80011F00` | **Message module overlay vector** (freeze witness, see §8.4) |
| `0x800EF1C8` | Global Huffman tree root (DW7 lineage) |
| `0x800F4DF0` | **Text work / translate buffer** ("dump 109 → dump 110" relocation arena) |
| `0x800F83C0` | Position/roster table ("LOCN descendant") |

### 4.3 The `TID`/`SID` compound handle

The single most important discovery of the project: every displayable message is addressed by a **32-bit referrer word**:

```
referrer = (BlockID << 20) | (BitOffset + HTS × 8)
```

where

| Field | Width | Range (recovered) |
|---|---|---|
| `BlockID` | 12 bits | `0x020 – 0x4FF` |
| `BitOffset` | 20 bits | bit-granular offset inside the block |
| `HTS` | 24-byte block header word | typ. `0x18` |

Two identifier name spaces wrap that word on the game side:

- **TID (Text ID)** — the gameplay text token (`@text_xxx` in the corpus).
- **SID (String ID)** — the raw string ordinal (`SID 773 → SID 774`, see §6.4).

`referrers.py` (293 lines, `translation-tools/hbe/`) computes these words over the raw image; 19,193 SIDs and 1,108 HBD blocks are accounted in the census (§13).

### 4.4 Control codes: the 51-opcode messaging VM

The PS1 messaging VM recognizes **51 control codes** in the string stream — 43 archive-resident plus 7 that are wildonly in the EXE/overlay domain. Every one is tabulated with behavior, side-effect, and pointer arithmetic in `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md` (public mirror committed to the repo root). Code pages are listed in [§12.2](#122-ps1-messaging-vm-opcode-census).

### 4.5 DMA choreography (PS1)

Unlike the SFC, the PS1's messaging path is **not** a PPU DMA blit forward; it is a CD-ROM read-back. The flow is:

1. CD drive → **RAM** (DMA1/CD-data), Mode 2 Form 1 sectors (2048B data + 24B control/EDC/ECC framing).
2. RAM → decompressor (`0x800F4DF0` target).
3. Decompressed text → off-CPU rasterization by the GPU draw-list under COP2/GTE; the text engine issues TMD/primitive buffers rather than tile writes.

### 4.6 The `type-39` script module

Scripts are fetched through the `type-39` module (403 script sites in the census). These are the lineage-avatars of the SFC chapter manager — small imperative bytecode programs that *sequence* string VM calls. They are the highest-risk surface for re-authoring (a stale `type-39` cost a full pipeline cycle; see §8.6).

---

## 5. The Messaging VM: String Tables, Control Codes, and the TID/SID Dispatch Cycle

### 5.1 Dispatch pointer math

The VM never branches by `switch`; it indexes a **dispatch stride table**. Call the base of the jump table `$Base` and the opcode stride `Stride`:

```
handler = $Base + (opcode × Stride)
jump  → handler
```

Recovered instances:

- **Font 1 renderer** (8×14 ASCII, SID-referenced): `Stride = 4` (one `jr` slot per opcode), `$Base` within the message module overlay.
- **Font 2 renderer** (proportional, Shift-JIS): dispatch by lead-byte class table, `Stride = 4`; glyph advance factored through the proportional width table (see §8.1 desync pathology).
- **Overlay type selector** (`type-46` dispatcher): `Stride = 2` over a 16-bit function index, 67 module types over 267 overlay sites.

Formally, for any index `i` the effective target address is `EA(i) = (UINT32)$Base + i·Stride`; all table entries are aligned to `Stride` in the published images — a property the Rebuild C patchers re-assert after every relocation.

### 5.2 The dispatch cycle

```
decode referrer word → BlockID, bit offset
HTS read → map physical block → stream
first byte: opcode
  opcode ∈ control space  → dispatch via $Base + op×Stride
  opcode ∈ glyph space     → Font1/Font2 render; advance cursor
stream guard:  if bytes consumed > dlen + 3 → FASTFAIL (§8.3)
```

The `≤ +3` overrun contract is invariant of both generational codecs and is enforced in the generator tooling, not merely diagnosed at runtime.

---

## 6. Compression Containers: Huffman HTS-0x18 and LZSS/DQLZS Bitstreams

### 6.1 Block header (24 bytes, `<6I` little-endian)

```
Offset  Field       Meaning
0x00    end         absolute end offset of this block (binary bound)
0x04    id          block id
0x08    hts         24 (0x18) — header/store size in bytes
0x0C    treeEnd     end of serialized Huffman subtree
0x10    textEnd     end of compressed text window
0x14    unk1        reserved/scratch
```

### 6.2 Tree header (10 bytes, `<IIH` little-endian)

```
Offset  Field        Meaning
0x00    treeStart    first node offset
0x04    treeMiddle   midpoint / count boundary
0x08    numNodes     node count (drives decode recursion)
```

Nodes are 2-byte LE. Bit 15 (`0x8000`) marks a branch; `0x7FFF` masks the child/base index. The root is addressed at `end − 4`. Symmetry with the DW7 lineage: *DQ VII* loads **one global tree** at `0x800EF1C8`; *DQ IV* follows suit, so a single tree serves all 1,108 blocks — a deliberate memory-budget trade that makes the tree a cross-block invariant (and a prime suspect in the `numNodes = 0` class of failures, §8.3).

### 6.3 Decoder pseudocode (canonical project form)

```
HUF_TREE_NODE(node)          // node = u16 LE
  if node & 0x8000:
     left  = (node & 0x7FFF) * 2        // index stride
     right = left + 2
     op    = BRANCH
  else:
     sym   = node & 0x7FFF
     op    = LEAF
  return op

DECODE_BLOCK(hdr):
  rewind = hdr.end - hdr.hts
  tree   = load_tree(hdr)         // root = end - 4
  bitpos = hdr.textEnd offset
  until textEnd:
     node = root
     while node is BRANCH:
        bit   = read_bit(stream)
        node  = bit ? node.right : node.left
     emit sym
  # contract: emitted length may exceed logical ulen by ≤ 3 bytes
  assert written_past_ulen <= 3      # §8.3
```

### 6.4 dqlzs (LZSS family) and split immediate remapping

The second codec is `dqlzs` (`translation-tools/hbe/compression/dqlzs.py`, 510 lines; sibling `lzss.py`): an LZ77-class match/copy codec with `dlen`/`ulen` fields carried in each sector group. Because pattern windows are 32-bit-aligned on the MIPS, the Rebuild pipeline inserts an intermediate **split-immediate** representation: a 32-instruction dataflow scanner plus an `ADDIU` sign-carry rewriter (`hbe/splitimm.py`, 248 lines). Split-immediate provenance is preserved through every re-pack so that `lui`/`ori` pairs never straddle a rewritten boundary — the exact corruption mode seen in the `numNodes=0` fastmem collision (§8.3).

Font-1 delta-lock is preserved across the edit: any edit of SID `N` that shifts the base delta re-derives every later SID in the same block (e.g., the canonical 44-bit deltas observed between `SID 773` and `SID 774`).

### 6.5 Sector framing

All reads are Mode 2 Form 1: 2352-byte physical sectors, 2048-byte data payloads. LBA math is mandatory for the Auld Well region (§8.2). The `sector.py` and `referrers.py` routines validate every stream descriptor against its LBA before patching.

---

## 7. Overlay Residency, Swapping, and the 2 MB Working Set

### 7.1 Residency model

The engine divides into **resident** code/data (the string VM core, fonts, the global Huffman tree) and **swappable** overlays pulled from the CD. Overlay dispatch is by module type; `type-46` is the duplication record family. The census recovers **67 module types over 267 overlay sites**; every overlay site carries a residency vector that must be honored by any relocation patch.

### 7.2 The message module

The message module overlay sites at `0x80011F00` (and friends) contain the Font-2 proportional renderer and the roster hooks. Its residency window is small and its swap latches are strict: a collision here does not corrupt memory — it **freezes the machine with the message module resident** (the "heely freeze", §8.4).

### 7.3 Working set pressure

The 2 MB working set must simultaneously hold: resident VM + fonts (Font 1 8×14 ASCII; Font 2 proportional Shift-JIS), one decompression window at `0x800F4DF0`, the global Huffman tree at `0x800EF1C8`, position/roster at `0x800F83C0`, plus every transient overlay. Budget failures present as `-763`-class sector starvation or `+4764`-byte overwrites (§8.3) — never as graceful degradation.

---

## 8. Rebuild C Engineering Lessons: Desync, Handshake, and Budget Failures

Rebuild C is the third in-place rewrite (`rebuildc_pipeline/`). Its failure archive is the empirical half of this paper.

### 8.1 Shift-JIS lead-byte desync (the font-index bias)

Yamana's Font 2 uses Shift-JIS lead bytes offset by a resident table bias of `+0x8000`. The classic Rebuild C desync:

- Wrong-byte stream submitted to the Font-2 path at `0x800F4DF0`; dump 109 renders **garbage**, dump 110 **FREEZES** at the `048B` sequence 241 line ("No allies to talk to right now.").
- Root cause: a generator had positioned text one byte off the shift-JIS lead-byte boundary — a paired-read misalignment that silently shifts every subsequent glyph decode by one lead byte.

Fix discipline: every `@text` reauthor re-validates its edit against the byte-pair table before commit (gate `F6` vsync bypass `0x80099F60` exists precisely to route around *render-time* desync so *data-time* desync can be imaged).

### 8.2 Auld Well streaming region (LBA 19082–19118)

The Auld Well is a **streamed** text region — its dialogue is decompressed directly off sequential CD reads. Constraints:

- LBA window `19082–19118` (37 sectors) is the handshake latch range;
- every decompressed byte ceiling inside it must be re-validated against the region's `dlen`/`ulen` pair, because the well's lines are emitted mid-DMA without an overlay hop;
- translation of 7-bit ASCII into the well's storage must not expand past the byte budget — the well cannot be re-paged.

This is the canonical *stream-budget* failure (as opposed to *memory-budget*).

### 8.3 Overrun contracts and the `numNodes=0` church freeze

The two decompressor contracts that every Rebuild generator is gate-locked to:

1. **`≤ +3` decoded-byte overrun** — a valid decode may legally emit up to three bytes past logical `ulen`, but never more. Patching that exceeds 3 corrupts the *next* block's tree header.
2. **`numNodes ≥ 1`** — the church quick-save overlay was found shipped with `numNodes = 0`; the decoder walked the canonical global tree against a zero-count header, over-anchored output by **+4,764 bytes**, and smashed fastmem with a bogus `lui`/`ori` pair at `0xFFFFFFFF`. Gate `G2` (church facility) pins `872/872` church records as a hard admit rule.

### 8.4 The heely freeze (overlay-residency collision)

A translated string pushed the message module's resident window at `0x80011F00`: the Font-2 proportional rasterizer, fully resident at the freeze vector. No DRAM corruption — a clean dispatch deadlock. Recovery = residency re-plan, not a write fix. Rule: **resident message-module text may not exceed 4 KB of work-buffer displacement** without re-planning the overlay set.

### 8.5 Facility vs battle-name segregation (`0x047B` class)

Japanese `0x047B`-class references are *dynamic facility jump tables* (shop, inn, church, innkeeper lists). They must be segregated from resident combat-message banks because they are re-pointed at runtime through `type-39` scripts — a translation side-effect there either patches the wrong *target*, or (worse) patches a *shared* bank that the combat name fleet also reads. `facility_marker_worksheet.json` (repo root) is the machine-readable segregation ledger.

### 8.6 Pipeline-order staleness (the `type-39` re-stale bug)

A STEP re-ordered patching (`4g` phase) left a previously-fixed `type-39` module stale in the *final* floppy image after a later step re-wrote its parent sector. Only the E2E gate sequence (`F9`/`F10`: `type-39` parity, monster-name table parity) caught it. Gate ordering is therefore part of the artifact: `gate_g2_church_facility.py`, `gate_g3_endor_0021.py`, `gate_g4_multicopy_parity.py`, `gate_g8b_varcode_parity.py`, `gate_f9_type39.py`, `gate_f10_monster_name_table.py`.

### 8.7 Field-name census (gate residue)

| Gate | Scope | Evidence |
|---|---|---|
| `G2` | church facility | 872/872 records |
| `G3` | Endor `0x0021` | 1,086 references × 3 tables |
| `G4` | multicopy parity | 19/19 copies |
| `G8b` | varcode parity | 33/33 |
| `Monster Gramps` | bare-page table | LBA 40217 |

---

## 9. Re-Authoring Architecture: The 100% Native In-Place Pipeline

### 9.1 Pipeline lineage

`frankenstein_pipeline_v090 → v095 → v096 → v097 → V.98 → V.99` (releases published to the repo), with expressed native build and `ShipB` distribution tracks.

### 9.2 Step topology (source truth, SHA-1–pinned at `d8b569e`)

1.  **STEP −1** native build / `verify_ship_parity.py`
2.  **STEP 1** pristine-image pinning
3.  **STEP 2** corpus + `@text` authoring → manifest `.vpbin`
4.  **STEP 3** `referrers.py` resolution → Huffman/LZ re-pack (`length_limited.py`, 403 lines)
5.  **STEP 4** gate sequence (`G2…G10`) → E2E static-green
6.  **STEP 5** `voidpatcher` apply → hashed output

### 9.3 voidpatcher payload

`voidpatcher/DQ4_REBUILD_B.vpbin` v3 "RB1-Hotfix1":

| Property | Value |
|---|---|
| Payload size | 33,794,548 bytes |
| Patched sectors | 14,344 |
| Pristine SHA-1 | `85064625AFA12219880FC8D07047A3CC1C595CB9` |
| Output SHA-256 | `055D37CE649BB1156D82670CF793D42A930062DD832F17D4E9DB5B926877761D` |

`voidpatcher` source: `main.cpp`, `vp_engine.cpp`, `vp_test.cpp`.

---

## 10. Verification, Gate Calculus, and the FF1AF239 Master

### 10.1 Current master

| Property | Rebuild C (playtest-ready 2026-09-13) |
|---|---|
| Master ID | `FF1AF239` |
| SHA-256 | `FF1AF239CBD0070ED647D09FD7C54649846F7B16E7B71A5E4FDA34BDFDB31BB8` |
| Size | 368,057,424 bytes |
| Sectors | 156,487 |
| Disc | `HBD1PS1D.Q41` / `SLPM_869.16` |

### 10.2 Static-green definition

A master is "static-green" when: (a) all parity gates `G2…G10` pass on final bytes, not intermediates; (b) every repacked codec honors the `≤ +3` overrun contract; (c) overlay residency vectors are unchanged; (d) pristine→patched differ only at `manifest`-listed sectors. `verify_ship_parity.py` executes this as STEP −1 on every cycle.

---

## 11. References — Local Study Corpus and GitHub Publication Corpus

Local paths are under `DQLOSTTRANSLATION/`. GitHub items are at the canonical base `https://github.com/luxauraofficial777/dq4frankenstein/blob/main/` (branch `main`, remote HEAD `aecc8523`).

### 11.1 Local study corpus (`study/`) and vault (`snes/`, `voidpatcher/`, `shipB/`, `shipC/`, `translation-tools/hbe/`)

| Artifact | Contents |
|---|---|
| `study/MASTER_OVERVIEW_BIG_PICKLE_Sep05_2026.md` | LZS overrun "THE FIX" provenance |
| `study/RC_MASTER_FF1AF239_PLAYTEST_READY_Sep13_2026.md` | current-master gate report |
| `study/SESSION_PROGRESS_RC_E2E_PIPELINE_Sep13_2026.md` | end-to-end cycle log |
| `study/RAM_DUMP_FINDINGS_Sep13_2026.md` | `0x800F4DF0` dumps 108–110 witness |
| `study/MASTER_DISC_CONTENT_MAP.md` | LBA ↔ module map (incl. `19082–19118`) |
| `study/YAMANA_HBE_MASTER_TID_LIBRARY.md`, `..._SID_LIBRARY.md`, `MASTER_CONTROL_CODES_LIBRARY_Sep06_2026.md` | TID/SID/opcode registers |
| `snes/docs/` (`dq3_sfc_technical.md`, `dq6_sfc_huffman.md`, `dq12_sfc_technical.md`, `HOST_DECISION.md`, `BOOT_FLOW.md`, `CONTROL_CODES_SFC_ENGINES.md`, `TILE_FORMAT.md`, `MAP_FORMAT.md`, `LOCN.md`), `snes/src/*.asm` | Generation I vault |
| `translation-tools/hbe/` (`compression/dqlzs.py`, `compression/lzss.py`, `huffman/length_limited.py`, `huffman/dq4.py`, `huffman/dw7.py`, `splitimm.py`, `referrers.py`, `sector.py`, `patch_font1_ourway.py`, `patch_type39_scripts.py`, `patch_battle_overlay.py`, `sovereign_overlay_refs.py`, `validate_corpus.py`) | codec/VM tooling |
| `voidpatcher/` (`payload_manifest.json`, `src/main.cpp`, `src/vp_engine.cpp`, `src/vp_test.cpp`, `DQ4_REBUILD_B.vpbin`) | patcher + manifest |
| `shipC/rebuildc_pipeline/` (`gate_g2_…_g10` scripts) | gate calculus |

### 11.2 GitHub publication corpus (repo root, `main` branch)

Study documents (`.md`):

- [`CASE_STUDY_NAKAMURA_YAMANA_HEARTBEAT_ENGINE_IEEE_2026.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/CASE_STUDY_NAKAMURA_YAMANA_HEARTBEAT_ENGINE_IEEE_2026.md)
- [`HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE.md)
- [`HBE_GENERATIONAL_TRANSLATION_HISTORY.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/HBE_GENERATIONAL_TRANSLATION_HISTORY.md)
- [`generation.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/generation.md), [`lineage.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/lineage.md), [`how_we_did_this.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/how_we_did_this.md)
- [`YAMANA_HBE_MASTER_TID_LIBRARY.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/YAMANA_HBE_MASTER_TID_LIBRARY.md)
- [`YAMANA_HBE_MASTER_SID_LIBRARY.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/YAMANA_HBE_MASTER_SID_LIBRARY.md)
- [`YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md)
- [`DQ4_PSX_Engine_Analysis_extracted.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/DQ4_PSX_Engine_Analysis_extracted.md)
- [`Dragon Quest IV PSX Engine Analysis.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Quest%20IV%20PSX%20Engine%20Analysis.md) / [`.pdf`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Quest%20IV%20PSX%20Engine%20Analysis.pdf)
- [`Dragon Quest Engine Analysis & Cross-Referencing.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Quest%20Engine%20Analysis%20%26%20Cross-Referencing.md) / [`.pdf`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Quest%20Engine%20Analysis%20%26%20Cross-Referencing.pdf)

- [`DW7_RE_Study_extracted.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/DW7_RE_Study_extracted.md)
- [`Dragon Warrior VII Reverse Engineering Study.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Warrior%20VII%20Reverse%20Engineering%20Study.md) / [`.pdf`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Warrior%20VII%20Reverse%20Engineering%20Study.pdf)
- [`Dragon_Quest_ROM_Hacking_Schema.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon_Quest_ROM_Hacking_Schema.md) / [`Dragon Quest ROM Hacking Schema.pdf`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/Dragon%20Quest%20ROM%20Hacking%20Schema.pdf)
- [`dq4study.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/dq4study.md), [`dq7study.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/dq7study.md)
- [`FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md)
- [`HBD_PROCESSING_BLUEPRINT_Jul19_2026.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/HBD_PROCESSING_BLUEPRINT_Jul19_2026.md)
- [`HLRM_INTEGRATION_BLUEPRINT.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/HLRM_INTEGRATION_BLUEPRINT.md)
- [`INSTRUMENTATION_BLUEPRINT.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/INSTRUMENTATION_BLUEPRINT.md)
- [`TELEMETRY_DIFF_REPORT_Jul20_2026.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/TELEMETRY_DIFF_REPORT_Jul20_2026.md)
- [`VECTOR_ANALYSIS_REPORT.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/VECTOR_ANALYSIS_REPORT.md)
- [`STUDY_LIBRARY_ANALYSIS.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/STUDY_LIBRARY_ANALYSIS.md)
- [`KING_MEDAL_SUMMARY.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/KING_MEDAL_SUMMARY.md)
- [`V99_ReBuildB_DOCUMENTATION.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/V99_ReBuildB_DOCUMENTATION.md)
- [`VERSION_LOG.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/VERSION_LOG.md), [`TOC.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/TOC.md), [`CREDITS.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/CREDITS.md)
- [`README.md`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/README.md), [`LICENSE`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/LICENSE) (CC BY-NC-SA 4.0)

Data / worksheets:

- [`facility_marker_worksheet.json`](https://github.com/luxauraofficial777/dq4frankenstein/blob/main/facility_marker_worksheet.json)

Distribution & release bundles (multi-part zips):

- `DQ4_Patcher_RebuildB_QuickStart.z01–z05` + `.zip`
- `dist_rebuild_b.z01–z06` + `.zip`
- `shipB.z01–z06` + `.zip`
- `cybergrime.z01–z07` + `.zip`
- `frankenstein_pipeline_v090.zip`, `v095.zip`, `v096.zip`, `v097.zip`, `V.98.zip`, `V.99.zip`
- `study.zip` (packaged `study/` snapshot)

Media assets: `dq4.png`.

Local repository internal history (not on the remote): the Phase A/B/C commit ladder whose head is `e97a442` "Phase B progress doc + CURRENT_STATE update…", and the `study/`-living documents named throughout this paper.

---

## 12. Appendix A — Hex-Level Structural Layouts

### 12.1 SFC vs PS1 at a glance

| Layer | SFC (DQ3/DQ6) | PS1 (DQ7/DQ4) |
|---|---|---|
| CPU | 65816 @ 5A22 | MIPS R3000A @ 33.8688 MHz |
| Prototype | bank-segmented | flat KSEG0 + overlay |
| String address | `Bank<<16 + tokenIdx` | `(BlockID<<20) | (BitOffset + HTS×8)` |
| Compress | per-package Huffman (DQ6) | global Huffman (`0x800EF1C8`) + dqlzs |
| VM ops | SFC control codes | 51-opcode PS1 VM (43+7) |
| Render | DMA→PPU BG tiles | GPU raster + work buffer `0x800F4DF0` |

### 12.2 PS1 messaging VM opcode census (summary)

Full behavioral table in `MASTER_CONTROL_CODES_LIBRARY_Sep06_2026.md` / repo `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md`. Fast-groups:

| Group | Class | Examples (recovered) |
|---|---|---|
| Text | glyph emit, space, newline | `00`-family ASCII, SJIS lead pairs |
| Flow | line feed, page, halt | clip, wait-key, end |
| Meta | name slot, roster, font switch | `0x048B` seq, `0x0021`, roster `0x800F83C0` |
| Script | `type-39` cross-call | chapter/script trampoline |

### 12.3 Huffman block hex-dump template

```
00 10 00 00 18 00 00 00 - 18 00 00 00 ...treeEnd   <- <6I> header
...treeStart (10B <IIH>) ... 02 00     <- numNodes=2 minimum
2 bytes/node: [8000|index] leaf syms ... root at end-4
```

### 12.4 Work-buffer witness map (`RAM_DUMP_FINDINGS_Sep13_2026.md`)

| Dump | Event | Verdict |
|---|---|---|
| 108 | pre-edit baseline | green |
| 109 | lead-byte desync | garbage render |
| 110 | `048B` seq 241 | FREEZE (`≤+3` violated downstream) |

---

## 13. Appendix B — Census Tables

| Measure | Count |
|---|---|
| SIDs | 19,193 |
| HBD blocks | 1,108 |
| `type-39` scripts | 403 |
| VM opcodes | 51 (43 archive + 7 EXE wild) |
| Overlay module types | 67 |
| Overlay sites | 267 |
| Patched sectors (`RB1-Hotfix1`) | 14,344 |
| Master sectors (`FF1AF239`) | 156,487 |
| `BlockID` span | `0x020–0x4FF` |

---

*End of document. Prepared for the VoidWalkers Research Project under CC BY-NC-SA 4.0. All offsets, LBAs, hashes, and counts trace to the artifacts cited in §11.*
