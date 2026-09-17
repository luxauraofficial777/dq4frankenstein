# Yamana HeartBeat Data — HBD Disc-Index & Archive Grammar Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V6 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (§2 container spec), Enix Suite PG5 (live HBD verification), `hbd_forensics.py`, `hbd_structure.json`.
**License:** CC BY-NC-SA 4.0
**Status:** Formal grammar for the archive master header, block headers, sub-block records, and the disc-level index (LBA 483–504 class).

---

## 1. Executive Abstract

`HBD1PS1D.Q41` is a 319,436,800-byte, 155,975 × 2,048-byte (Mode 2 Form 1) block archive
starting at LBA 362. It is a **sector-aligned, zero-filesystem** container: every block is
2,048-byte aligned so the engine can issue raw DMA reads straight into memory with no
filesystem driver overhead. The disc-level index (DW7/PSX LBA 483–504 heritage; Q41 reads at
the same class of index cells) is a tree of folders → files → blocks that the engine walks
before seeking to content.

---

## 2. Disc Geometry (MEASURED)

| Property | Value |
|---|---|
| Archive file | `HBD1PS1D.Q41` |
| Size | 319,436,800 B |
| Blocks | 155,975 × 2,048 B |
| Start LBA | 362 |
| Folder count | 3,243 |
| File count | 23,828 |
| Entry count (census) | 44,657 |
| EXE | `SLPM_869.16` @ LBA 24–361 slot (`t_addr 0x800918F4`, `d_addr 0x80017F00`, `d_size 0xA8800`) |
| Total discs | 156,487 LBA |

The 44,657/23,828/3,243 census matches `hbd_structure.json` and was re-derived live by
`hbd_forensics.py` (Enix Suite PG5).

---

## 3. Archive Master Header (16 bytes)

Every HBD block's first sector begins with a fixed 16-byte master header:

```
+0x00  u32 num_subs       number of sub-blocks
+0x04  u32 num_sectors    sectors this block spans
+0x08  u32 total_len      total payload length
+0x0C  u32 reserved       (zero)
```

## 4. Sub-Block Descriptor (16 bytes)

```
+0x00  u32 dlen   disc payload length
+0x04  u32 ulen   uncompressed length
+0x08  u32 extra  engine-specific
+0x0C  u16 flags  compression class flags (0x0500 ⇒ Huffman family)
+0x0E  u16 type   structural class (21/31/35/36/39/40/42/44/06/13/26/46 …)
```

Cumulative packing rule: the engine walks `hdr + cum(dlen)` across sub-blocks — no absolute
offsets are stored per sub-block.

---

## 5. Disc-Level Index (LBA 483–504 class — MEASURED on DW7/Frankenstein)

- The bootstrap reads the HBD index region (Pristine DW7 LBA 483–504, 22 sectors) before
  any seek; it holds the folder/file/block tree the engine walks to resolve a TID to a
  sector.
- In the Q41 sovereign native layout the archive starts at LBA 362 and the index cells are
  the first archive blocks; the engine resolves `TID → block → sector` through the tree
  (`0x8008F7B0` sentinel/directory walk + `0x8008FB48` directory re-init).
- **Zero-filesystem invariant:** the index is read by *raw DMA*, never by the ISO filesystem
  driver (DirectStorage-like Model — see AGENT/LuxArchitect study for the modern analogue).

### 5.1 Directory structure (INFERRED — matches `hbd_structure.json`)

```
HBD tree
 ├── folder (3,243)
 │   └── file (23,828)
 │       └── block (155,975 total sectors) — master header + sub-block records
One file may own many blocks; one block may host many sub-blocks (23,828 subs census).
```

---

## 6. Container Taxonomy Bind (MEASURED)

The sub-block `type` field selects the codec class per the two-key dispatch
(`YAMANA_HBE_COMPRESSION_CODEC_SPEC.md` §1):

| Type band | Class |
|---|---|
| 21, 31, 35, 44 | RAW_PASSTHROUGH (3D meshes / sentinels / cell rosters) |
| 06, 13, 26, 46 | DQLZS (LZSS sliding dictionary) |
| 23, 24, 25, 27, 39, 40, 42 | WIDE_HUFFMAN (dialogue trees / script bytecode) |

`flags==0x0500 ∧ type ∈ {23,24,25,27,39,40,42,44}` is the exact Huffman predicate; any other
combination is RAW or LZSS.

---

## 7. Index-Integrity Invariants

| # | Invariant | Gate |
|---|---|---|
| D1 | Total disc sectors == 156,487; PVD space matches | G2 / EDC-ECC |
| D2 | `num_sectors` walk == physical sector consumption | zero-sector-shift |
| D3 | census (44,657 / 23,828 / 3,243) stable across runs | `hbd_forensics.py` |
| D4 | index cells byte-stable (LBA 483–504 heritage) | pairwise disc diffs |

---

## 8. Open Items

| # | Item | Decides |
|---|---|---|
| D-1 | Full TID→block→sector resolution table extraction | address-space map |
| D-2 | Index-cell version delta between DW7 (`W71`) and Q41 | cross-title census parity |

---

*Rev 1.0 end. Companion: `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` §2 ·
Enix Suite PG5 · `hbd_forensics.py` · `hbd_structure.json`.*