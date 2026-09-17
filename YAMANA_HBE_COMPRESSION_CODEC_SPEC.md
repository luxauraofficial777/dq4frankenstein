# Yamana HeartBeat Engine — Compression Codec Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V3 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (§3), `YAMANA_NAKAMURA_HEARTBEAT_ENGINE_GENERATIONAL_ARCHITECTURE.md` (§6), `WELL_FLOOR_COMPRESSION_FORENSICS_Sep13_2026.md`, `DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md`, Enix Suite PG3/PG5.
**License:** CC BY-NC-SA 4.0
**Status:** Consolidated codec reference — one authoritative document for Huffman `HTS-0x18`, dqlzs, and the RAW passthrough boundary.

---

## 1. Executive Abstract

HBE does not apply uniform compression. Every HBD sub-block belongs to exactly one of three
disjoint structural classes, chosen by a **two-key dispatch rule** — never a single
yes/no `compressed` flag:

| Class | Flag / Type | Codec | Engine Dispatch |
|---|---|---|---|
| **WIDE_HUFFMAN** | `flags=0x0500`, type ∈ {39, 40, 42} (+ tree-carriers 23/24/25/27 as applicable) | 12-bit dual-array Huffman (length-limited) | `0x8008F59C` bitstream walker |
| **DQLZS_COMPRESSED** | `flags=0x0500`, type ∈ {06, 13, 26, 46} | LZSS sliding dictionary (raw mode) | `0x8008F9A0` cell unpacker |
| **RAW_PASSTHROUGH** | `flags≠0x0500` or payload is 3D geometry/sentinels; type ∈ {21, 31, 35, 44} | none (direct DMA copy) | bypasses codecs |

A naive global `compressed = (flags == 0x0500)` corrupts non-text containers; raw-mesh
geometry (type 21), cell rosters (type 44, e.g., File 9 / Container 18683 Sub 2), and
fixed-stride sentinel tables legitimately use `flags != 0x0500` or non-Huffman payloads.

**The formal two-key dispatch rule:**

```
compressed_HUFF = (flags==0x0500) ∧ (type ∈ {23,24,25,27,39,40,42,44})
```

---

## 2. Huffman Codec (`HTS-0x18`)

### 2.1 Tree serialization (MEASURED)

- Header size is **24 bytes fixed** (`HTS 0x18`); the header precedes the payload, and the
  on-disc tree is serialized in the block header (`HeaderTreeSize` double-word units).
- Referrer arithmetic depends on it: `ReferrerWord = (BlockID ≪ 20) | (BitOffset +
  HeaderTreeSize × 8)` — the tree skip is **byte**-scaled to land on the first code bit.
- DQ4's per-block trees are **length-limited Huffman** (depths 14..9). Re-encoders use
  length-limited Huffman to honor the `{0000}`-terminator and 20-bit/128 KB per-block
  budget. (Binary-isomorphism proof vs DW7 global hybrid tree: root_id `+1`, odd-vs-even
  `offset_b`; fix ≈ 3–5 MIPS instructions — per `FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md`.)

### 2.2 Bit conventions across the lineage (MEASURED)

| Title (year) | Convention | Evidence |
|---|---|---|
| DQ1+2 SFC (1993) | plain-byte text + `0xD0–D2` kanji escape (**no Huffman**) | header-checksum-verified; corrected-myth ledger |
| DQ3 SFC (1996) | 13-bit coefficient Huffman, MSB-first | ptr `79 12 00` → steps `011111101001` → leaf `0x0521` (ツ); bit=1→`$161A7` |
| DQ6 SFC (1995) | global tree, 1,064 nodes × 2 parallel arrays at raw `0x167BE`/`0x1700E`, root `0x427`; **MSB polarity inverted vs DQ5** | measured arrays, both roots inner |
| DQ4 PSX (2001) | per-block length-limited Huffman `HTS-0x18` | census 1,358 blocks re-encoded (depth 14..9) |

### 2.3 Wide-tree walker (`0x8008F59C`)

- Dual-array (left/leaf, right/) look-up per bit; consumes from the bitstream at the SID's
  BitOffset.
- On decode side: emits character stream + control tokens to the messaging VM (§4).
- **Overrun contract:** generators may overrun at most `+3` decoded bytes past target length.

---

## 3. dqlzs (LZSS sliding dictionary)

### 3.1 Structure (MEASURED)

- Sliding-window LZSS with literal/copy token boundaries; operates on raw byte cells.
- Used by type 06/13/26/46 sub-blocks: script bytecode (Type 39 script bodies are LZSS →
  remap → recompress; 612 Type-39 blocks), overlays, and algorithmic cells (`0x048B` battle
  overlay heritage).
- Unpacker family `0x8008F9A0` (planar/cell unpacker). Streaming path feeds it from the
  uncached KSEG1 alias `0xA00E8008`.

### 3.2 Stream discipline (INFERRED, consistent with `0x8008F810` sync)

1. DMA3 delivers sectors to ring `0x800E8000`.
2. `0x80018C20` invalidates stale L1 D-cache lines (R3000A has no DMA bus snooping).
3. Decompression executes from `0xA00E8008` (uncached mirror) so cache staleness cannot
   corrupt output.
4. On drain, `0x8008F8E4` asserts `[0x800F84E4]=1`.

---

## 4. RAW_PASSTHROUGH Boundary (MEASURED)

- **Auld Well floor containers `0x0066`–`0x006C`** (B1F Descent 15 subs/104 sectors/LBA
  107,143; B3F Flying Shoes 15/46/106,980; Healie Recruitment 15/44/106,864) fail a naive
  `DQ4Schema.parse_tree` because they are **heterogeneous composite blocks**: type 21
  collision meshes, type 35/36 entity-trigger matrices, type 39 script bytecode, type 40
  dialogue strings. They are RAW passthrough **by law** (compile-time contract I-1).
- The dialogue work buffer `0x800F4DF0` preserves unstreamed recruitment strings
  `{7f43}Come here...{7f0b}` (SIDs `0x0384`/`0x0387`) across functional *and* stalled
  states — proving the stall is a runtime re-seed failure, not corrupted string text.

### 4.1 Sector-aware write & EDC/ECC regeneration (land fix)

The Sovereign Build 12 "ghost build" defect: `_repair_edc_ecc` in
`hbe_sovereign/pipeline.py:432` hardcoded `range(22, 30)` and repaired exactly 8 of 2,794
modified sectors — 2,786 left with invalid EDC and uncomputed Reed-Solomon P/Q.

**Land fix (§ DMA_SUBBLOC spec §6.2):** replace with `_modified_form1_sectors(ref, out)`
covering the full modified-Form-1 set (`sb[18]&0x20` skips Form-2): EDC over header
(16..23)+user data (24..2071), P/Q vectors (2076..2351). Gate on
`edc_ecc_repaired == touches(Form1)`.

---

## 5. Codec Invariant Checklist

| # | Invariant | Gate |
|---|---|---|
| K1 | three-class exclusivity (HUFF / DQLZS / RAW) | dispatch census G3 |
| K2 | `HTS-0x18` header consumed before payload | referrer arithmetic |
| K3 | `≤ +3` overrun bound | every generator |
| K4 | mandatory `{0000}` terminator | VM contract |
| K5 | 20-bit / 128 KB per-block ceiling, zero sector shift | budget gates |
| K6 | every modified Form-1 sector EDC/ECC repaired @@ touches | sovereign build gate |

---

## 6. Open Items

| # | Item | Decides |
|---|---|---|
| H-1 | Per-block tree metadata census (tree sizes per TID) for full block-map validation | re-encode budget proof |
| H-2 | Exact literal/copy token width table for dqlzs cells | byte-exact independent decoder |

---

*Rev 1.0 end. Companion: `DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md` ·
`YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` §3 · Enix Suite PG3/PG5.*