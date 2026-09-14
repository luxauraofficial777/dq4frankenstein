# Formal Architectural Whitepaper & Engine Specification
## The HeartBeat Engine (HBE) and HeartBeat Data (HBD) Archive Architecture — `SLPM_869.16` / `HBD1PS1D.Q41`

**Document ID:** VW-DQLOST-TECHRPT-008 · **Rev** 1.0 · **Date:** 2026-09-14
**Authoring role:** Principal Reverse-Engineering Architect & PlayStation 1 Systems Analyst (Big Pickle, VoidWalkers Research Project)
**System architect:** Lux Aura
**Target invariants:** Zero Sector Shift · Native MIPS R3000A Stride Execution · Clean-Room Re-Authoring
**Evidence corpus:** `study/DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md`, `study/MASTER_FIX_WELL_THIRDCLASS_AND_FACILITY_ORDERING_Sep14_2026.md`, `study/BUG_TRIAGE_AND_FIX_PLAN_Sep14_2026.md`, `study/WELL_FLOOR_COMPRESSION_FORENSICS_Sep13_2026.md`, `study/CHAPTER_{1,2,4,5,6}_FORENSIC_RAM_AND_MIPS_OVERLAYS.md`, `translation-tools/hbe/` (in-tree parser code), and the `YAMANA_HBE_MASTER_*_LIBRARY` registers (mirrored to [`luxauraofficial777/dq4frankenstein`](https://github.com/luxauraofficial777/dq4frankenstein)).
**License:** CC BY-NC-SA 4.0

---

## Table of Contents

1. [Executive Overview & Architectural Philosophy](#1-executive-overview--architectural-philosophy)
2. [Physical Container Specification (.HBD Archive Architecture)](#2-physical-container-specification-hbd-archive-architecture)
3. [Multi-Class Compression & Decompression Dispatch Vectoring](#3-multi-class-compression--decompression-dispatch-vectoring)
4. [MIPS Disassembly & Runtime Execution Architecture](#4-mips-disassembly--runtime-execution-architecture)
5. [The Scripting & Dialogue Subsystem (Types 39, 40, 42)](#5-the-scripting--dialogue-subsystem-types-39-40-42)
6. [Mathematical Invariants & Verification Pipeline Rules](#6-mathematical-invariants--verification-pipeline-rules)
7. [Appendix A — C Struct Definitions](#7-appendix-a--c-struct-definitions)
8. [Appendix B — Dispatch & Census Tables](#8-appendix-b--dispatch--census-tables)
9. [Appendix C — Source Traceability](#9-appendix-c--source-traceability)

---

## 1. Executive Overview & Architectural Philosophy

### 1.1 The system

The HeartBeat Engine (HBE) shipping in *Dragon Quest IV – Michibikareshi Mono Tachi* (PlayStation, 2001) is a 33.8688 MHz MIPS R3000A application built on a proprietary optical container — the **HeartBeat Data** (`HBD`) archive — stored in `HBD1PS1D.Q41` (319,436,800 B, 155,975 × 2,048-B Mode 2 Form 1 blocks, starting LBA 362) and directed by the executable `SLPM_869.16` (692,224 B at sector 24; PS-X EXE header `t_addr(+0x10)=0x800918F4`, `d_addr(+0x18)=0x80017F00`, `d_size(+0x1C)=0xA8800`).

The engine's design philosophy, as recovered, is that of a **compiler for game dialog under hard byte budgets**:

- **Direct hardware execution.** Text is not interpreted from a separate language runtime; it is decoded by hand-written MIPS that walks the disc read-back stream and rasterizes glyphs through the GPU draw-list.
- **Fixed VRAM budgets.** Two font planes coexist under a singular page latch: the 8×14 half-width ASCII plane (Font 1) and the proportional 16×16 Shift-JIS plane (Font 2). No dynamic texture allocation exists.
- **Zero-heap runtime paradigms.** All message working state lives in fixed-residency globals: work buffer `0x800F4DF0`, roster `0x800F83C0`, message directory pointers `0x800F51EC..0x800F5208`, unblank latch `0x800F84E4`, DMA3 ring `0x800E8000`. There is no `malloc`; decompression targets are statically placed slabs.
- **Bare-metal disc sector streaming.** Text may arrive mid-DMA: the streaming path (`dma3_sync_stream_decompress` @ `0x8008F810`) synchronizes on INT3, flushes the D-cache (`0x80018C20`) because the R3000A has **no DMA bus snooping**, and unblanks via `0x800F84E4` only after the whole sector funnel drains.

### 1.2 Historical failure modes of blunt romhacking passes

The corpus's 2020–2026 attempt history documents four canonical failure classes, all reproduced in Rebuild C forensics:

| Failure class | Mechanism | Disc/RAM witness |
|---|---|---|
| **TOC boundary breaks** | Rewriting a block header value (e.g., `d`, the tree-end/font-record start) shifts the engine's per-block boundary computation; `divu`-by-modulus traps fire | `0x048B` phantom font record `[10908, 11112)`; RadMage `d=0` blocks (191 total) |
| **Split-shift kana/English desyncs** | Referrer points mid-glyph; the Shift-JIS renderer resumes a few bits late after a control/terminator (`+6/+8/+19` measured), producing `ＮＴ　ＷＲＭ`-class fragments | `0x8008F3BC` decode resumption; dumps 65–80/72–80; battle tactics garble `EずンてるタELL` |
| **LBA stream clobbers** | A patched sub-block inside a streamed LBA window overwrites the *framing* the CD controller uses; 11-bit sector-completion masks never fire | Auld Well LBA 19082–19118; sector `18683` sub 2 corruption |
| **Transition directory corruption** | `0x8008FB48` re-zeroes the 12-slot message directory at transition; a non-recognized container is skipped by repopulation, stranding slab pointers | dumps 196/197: self-referential ring `0x0F51E8→0x0F51EC→0x0F51F0`, unblank latched `1` yet descent dead-locked |

The central engineering conclusion (proven Sep-14): **HBE does not apply uniform decompression, and a single yes/no `compressed` flag applied to every sub-block type is mathematically incapable of expressing the engine's real dispatch domain.** The rest of this paper formalizes the container, the dispatch layer, the runtime, the scripting VM, and the verification invariants that a clean-room re-author must hold.

---

## 2. Physical Container Specification (.HBD Archive Architecture)

### 2.1 Archive master header (16 bytes)

Every HBD block's first sector begins with a fixed 16-byte master header:

```
Offset   Size   Field    Meaning
+0x00    4      nsub     sub-block (record) count
+0x04    4      nsec     sector count / span of the archive
+0x08    4      tlen     archive total length (bytes)
+0x0C    4      zero     padding; must be 0
```

Parser evidence (in-tree, verified against `SLPM_869.16`/`Q41`): `hbe/referrers.py:50` reads `num_subs, num_secs, span_len, zero_chk = struct.unpack_from("<4I", ...)`; `hbe/archive.py:188-189` reads `num_files` and `sector_count` from the same two leading words with `size` at `+8`.

### 2.2 Per-sub-block metadata record (16 bytes)

The record table immediately follows the master header, `nsub` records × 16 B:

```
+0x00    4      dlen    compressed slot size (packed length budget)
+0x04    4      ulen    decompressed logical length (declared)
+0x08    4      extra   extra header parameter (per-type)
+0x0C    2      flags   compression flags (0x0500 = DQLZS)
+0x0E    2      type    sub-block type code (23..46)
```

Parser evidence: `hbe/referrers.py:62-63` (`comp_len, uncomp_len, extra = "<3I"` at `item_loc`; `flag_val, type_code = "<2H"` at `item_loc + 12`).

### 2.3 Packing mechanics

Sub-block payloads are packed **sequentially** by cumulative compressed size:

```
payload_n = hdr_size + Σ(dlen_k)   for k = 0 .. n-1          (hdr_size = 16 + 16·nsub)
```

`tlen` must bound `payload_{n} ≤ sector_aligned_boundary` and the block must not spill past its declared `nsec` span. Padding runs to the sector boundary are mandated to preserve the **pristine LBA topology** (zero sector shift is a hard invariant, §6.3). A sub whose `dlen` shrinks must therefore *not* re-pad into the following record — the slot's tail bytes stay untouched (the "honest `dlen` shrink" Fallback B) or the stream is grown inside `[ulen_effective, dlen]`.

### 2.4 Nested text-block container (Huffman payloads)

A *compressed text* sub-block carries a nested container at its payload start (24-byte header + 10-byte tree header), specified in Appendix A.7 and §3.1. This nesting is why parse-level tools must never confuse the 16-byte *record* (`dlen`/`ulen`/...) with the 24-byte *text header* (`end`/`id`/`hts`/...).

---

## 3. Multi-Class Compression & Decompression Dispatch Vectoring

### 3.1 Model: three disjoint container classes + a canary class

The engine's decompression layer is **heterogeneous and type-selective**. A global Huffman re-encode pass, or a yes/no `compressed` flag keyed on one `flags` value, corrupts the engine's own validation at dungeon entry — the exact collision reproduced at the Auld Well stairs.

#### Class 1 — Wide-tree Huffman streams

Large town/overworld dialogue containers (Endor mega-block `0x0021`; 1,087 strings / 93,140 B) governed by a binary prefix tree with a special "two-region" structure:

```
TEXT_HEADER (24 B):  end | id | hts(=0x18) | treeEnd | textEnd | unk1
TREE_HEADER (10 B):  treeStart | treeMiddle | numNodes
NODES: 2-byte LE; 0x8000 = branch bit; 0x7FFF = child/base mask; root at end−4
DP_TABLE: marker(4) count(4) then <count> u32 leaf-bit offsets (binding entries to SIDs)
```

Packaged SID addressing is exact:

```
ReferrerWord = (BlockID << 20) | (BitOffset + HeaderTreeSize × 8)
BlockID       ∈ [0x020 .. 0x4FF]     (12-bit sub-block/stream slot selector)
HTS           = 0x18
```

String `N` lives at `ReferrerWord` where `BitOffset = DP_OFFSETS[N]`; SID/`dp` tables must remain **monotonic non-decreasing** (§6.2).

#### Class 2 — Algorithmic raw cells & overlays

Planar 2-bit graphic/roster modules, the battle message overlay (`0x048B`), and dungeon overlay cells (type-26/44/46/39). `DQ4Schema.parse_tree` **fails** on these (root-at-end−4 is not a branch); the decode path is the 2-bit planar unpacker (`0x8008F9A0`, §4.4) / 2bpp RLE expander (`0x8008FA30–0x8008FB40`), **not** Huffman. **LZSS re-encoding destroys cell stride**: re-packing a raw 2-bpp cell stream realigns run boundaries and corrupts glyph/cell indexing. 148 such blocks (`flags == 0x0000`, `type == 44`) were misidentified by the legacy single-flag test and corrupted in Rebuild C (sector `18683` sub 2); they are now hard-excluded (§6.4).

#### Class 3 — Fixed-stride sentinel lookup tables

Facility/priest/menu interaction arrays (type-23/24/25/26/27/46). These are **not compressed streams at all**: they are fixed-stride entry arrays traversed by sentinel-terminated walk chains and modulo indexing (`divu`-remainder + linked walk, §4.3). Rewriting them in-place is safe only when the fixed-stride and the record layout are preserved; re-encoding them as LZSS is a category error.

#### Boundary canaries — pristine RAW passthrough

Specific dungeon-transition containers — the Auld Well floor `0x0066–0x006C` (LBA `19067–19118`, crash stream `19082–19118`) — form a *third* (uncatalogued) class: `parse_tree` fails, they are not rosters, and any structural header shift trips the engine's transition **directory repopulation heuristics**. They must remain 100% byte-identical to pristine (`WELL_THIRD_CLASS_IDS` in `dq4_hbd_patcher.py` `HOLD_IDS`; `EXCLUDED_TARGETS` in `patch_table_refs.py`; gate `gate_well_thirdclass_parity.py`).

### 3.2 The fatal flaw of the single-flag check vs. the two-key rule

Legacy tooling selected the driver by one value:

```python
compressed = (flag_val == 0x0500)          # SINGLE-FLAG: applied to every type  — WRONG
```

This cannot express Class-1/Class-3/raw-class blocks whose `flags != 0x0500` yet are not LZSS — and worse, it *misreads* raw blocks (`flags == 0x0000`, `type == 44`) as plain rosters, corrupting 2,649 bytes in sector 18683 sub 2 (107 runs, 58 × `0x6DB6DB6D` fixed-slot filler words) — the Auld Well black screen.

The container-aware dispatch (implemented Sep-14 in `hbe/referrers.py` + `patch_table_refs.py`) is a **two-key heuristic**:

```python
LZSS_TYPES = {23, 24, 25, 27, 39, 40, 42, 44}

compressed = (flags == 0x0500) and (type_code in LZSS_TYPES)
# guard: raw cell blocks must be skipped outright
if sb.type == 44 and not sb.compressed:  continue  # 148 blocks preserved pristine
```

High-level dispatcher (canonical project form):

```
def container_decoder(lba, tid, sub):
    # 1) Domain carve-out by archive-LBA range (ENDOR mega-stream 0x0021)
    if lba in ENDOR_LBA_RANGE or tid == 0x0021:   return DRIVER.WIDE_HUFFMAN
    # 2) Battle/dungeon overlays: raw 2-bit cells; never LZSS
    if tid in (0x048B, 0x048F):                    return DRIVER.RAW_CELLS
    # 3) Facility/menu: fixed-stride sentinel tables (LZSS only if flagged)
    if sub.type in (23, 24, 25, 26, 27, 46):
        return DRIVER.DQLZS if sub.flags == 0x0500 else DRIVER.LOOKUP_SENTINEL
    # 4) Text/dialogue Type-39/40/42/44 with the two-key rule
    if sub.flags == 0x0500 and sub.type in LZSS_TYPES:
        return DRIVER.DQLZS
    return DRIVER.RAW
```

Authority order for writes: (1) verify-only upstream selector — two-key; (2) remap decisions bind on `(archive_lba, sub.off)`, never on a global re-encode; (3) font-safe injection — a 1-byte control prefix (`{7F0B}`) routes the renderer into the ASCII plane *only where the corpus already places it* (a blanket prefix was measured to alter ~19,153/19,158 strings and was **rejected** as build-breaking).

---

## 4. MIPS Disassembly & Runtime Execution Architecture

EXE mapping (verified by disassembly): `file_off = addr − 0x80017F00 + 0x800` — e.g. `0x8008F810` → file `0x78110`. Boot: EXE load `0x80017F00`, thread PC0 `0x8008E284`; engine entry `0x80010000`.

### 4.1 `0x8008F280` — Packed SID resolver & directory walk

`src: study/DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md; digests extract from dumps 196/197`:

```
srl  v1, a0, 20            # TID = high 12 bits of the 32-bit referrer word
# walk the 12-slot directory at 0x80100168, entry stride 0x20:
for slot in 0..11:
    if slot.tid == v1:                       # lhu +0x0A slot filter
        flag = lhu  (slot + 0x1C)            # +28: block flag register
        P    = lw   (slot + 0x0C);  M = lw(slot + 0x10)   # .P/.M pointer pair
        base = (P or M)                      # plane selected per cell by t1>>28
        return (base + (a0 & 0xFFFF))        # block_base + low16 = bit offset
# packed return form (0x8008F318-344):
#   (addr >> 3) | ((a0 & 7) << 28) | (slot << 24)
```

This reproduces the `(BlockID<<20)|(BitOffset+HTS×8)` indexing exactly: the high 12 bits are a **sub-block/stream slot selector**, not a scalar font flag.

### 4.2 `0x8008F3BC` — String renderer & font-plane latches

```
read byte b
if b in [0x81,0x9F] or [0xE0,0xFC]:   consume lead byte → SJIS two-byte glyph plane
if b == 0x10:                         8-byte control command (control token interpreter)
if b >= 0xFE:                         two-byte Shift-JIS pair
else:                                 single-byte glyph (ASCII plane)
per-cell plane select: bit (t1 >> 28) chooses between .P and .M pointer
```

This is the *Latin vs SJIS plane latch*: single-byte ASCII `0x01–0x0F, 0x11–0xFD` (excluding SJIS leads) renders on the ASCII plane; SJIS leads switch the two-byte plane. A string missing its one-byte plane selector renders as SJIS lead bytes → the **split-shift kana/ASCII garble**.

### 4.3 `0x8008F7B0` — Fixed-stride lookup-table walk

```
12 slots at 0x80100168; per-slot entry array:
    filter = lhu(node_base + 0x0A)          # slot selector
    r      = divu(target, entry_stride)      # modulo indexing remainder
    while node != SENTINEL:                  # sentinel-terminated chain
        if lhu(node + 2) == (target & 0xFFFF): return node
        node = next(node)                    # linked walk
```

Class-3 sentinel tables (§3.1) are consumed here. The Font 1 glyph atlas is a sibling structure at this region (chained hash table, modulus 137, per RadMageIRL `fonts.py`): codes miss → nothing drawn, codes hit → atlas cell.

### 4.4 `0x8008F9A0` — 2-bit planar cell unpacker

```
Group of 32 cells: masks 0xCCCC / 0x3333 reorder the two bit-planes
per cell decode:  read 2 bits; value != 0 → run = 1; else read 2 more bits,
                  run = (those + 1)   # 1..4 zero-run
                  emit `run` pixels of value until width×height exhausted
```

Sibling for the 2bpp RLE glyph stream: `0x8008FA30–0x8008FB40`; descriptor runtime patching lives at `0x8008F8E8–0x8008F900` (masks `0xF00FFFFF` / `0x0FFFFFFF`).

### 4.5 `0x8008FB48` — Map-transition directory zeroing & re-chain

```
for slot in 0..11:  zero 0x80100168 + slot*0x20           # directory reset
write empty self-referential ring: 0x0F51E8 → 0x0F51EC → 0x0F51F0 ...
# then re-run the directory REPOPULATION pass against the overlay list
```

Dumps 196/197 show RAM `0x800F51F0` pointing into the `0x8019xxxx` slab pre-descent, and the empty self-ring post-reset — with repopulation **skipping the canary third-class blocks** (§3.1) → frozen `{7f43}Come here...{7f0b}` at `0x800F4DF0` and unblank left at `1`. The engine's own re-init cannot be stopped; the re-author's job is to make repopulation *succeed* by keeping canary containers pristine.

### 4.6 `0x8009A120` / `0x8009A240` — CD-ROM sector-slot DMA dispatch & completion masks

```
0x8009A120  read-timeout evaluator/spindle recovery:
    if INT3 not asserted within 120 VBlank frames:
        write 0x06 (CdlPause) → 0x1F801801;  reset param FIFO 0x1F801800
        re-issue CdlReadN
0x8009A240  Mode 2 Form 1 checksum / EDC-ECC trap:
    fires sub-handlers per 11-bit sector-completion mask;
    on controller error: log status; branch to 0x8009A240-class trap handler
```

Supporting stream machinery: `0x8008F810 dma3_sync_stream_decompress` (DQLZS, 120-frame INT3 timeout on `$I_STAT` bit 2; D3_MADR `0x1F8010B0`; dcache flush `0x80018C20`; unblank store `sw $v1, −0x7B1C($v0)` → `0x800F84E4`), and `0x8008F59C`/`0x8008F214`/`0x8008F178` (decoder / block registration). Because D3 writes DRAM without snooping, decompression must read the uncached mirror `0xA00E8000` or explicitly flush.

### 4.7 Runtime memory map (fixed residency)

| KSEG0 address | Resident object |
|---|---|
| `0x80010000` | Engine boot / entry |
| `0x80011F00` | **Freeze-trap vector** (message module overlay; a collision here dead-locks dispatch, it never corrupts DRAM first) |
| `0x80017F00` | EXE load base (`t_addr 0x800918F4`, `d_size 0xA8800`) |
| `0x80019CE4` | FONT1 table secondary (16 entries, stride 8) |
| `0x8008E284` | Thread PC0 / BSS entry |
| `0x800A9FA0` | FONT1 table primary (48 entries, strides 20/24) |
| `0x800E8000` | **CD-ROM DMA Channel 3 streaming ring buffer (16 KB)**; uncached alias `0xA00E8000` |
| `0x800F4AC0` | Cached message-block pointer lookup table |
| `0x800F4DF0` | **Text work / decompression buffer** (frozen-stream witness) |
| `0x800F51EC–0x800F5208` | **Message directory pointer block** (reset ring `0x0F51E8→0x0F51EC→0x0F51F0`) |
| `0x800F83C0` | Position / roster table |
| `0x800F84E4` | **Video display blanking latch** (0=blackout, 1=unblank) |
| `0x80100168` | **12-slot message directory** (entry stride 0x20; `.P` +0x0C, `.M` +0x10, filter lhu +0x0A, flags lhu +0x1C) |
| `0x8019xxxx` | Streamed slab region (dump-196 directory targets) |

DMA registers in play: `D3_MADR 0x1F8010B0`, `D3_BCR 0x1F8010B4`, `D3_CHCR 0x1F8010B8`, `DICR 0x1F8010F0`; CD: `0x1F801800` (param FIFO), `0x1F801801` (command), INT3 = `I_STAT` bit 2.

---

## 5. The Scripting & Dialogue Subsystem (Types 39, 40, 42)

### 5.1 Encoding & control tokens

Strings are decoded at `0x8008F3BC` (§4.2) with a token grammar:

| Token family | Semantics |
|---|---|
| `0x10` + 7-byte payload | 8-byte control command (multi-armed) |
| `{7F..}` family | Block-local control (page advance `{7F0B}`, name substitution `{7F3F}`/`{7F3D}`, etc.) |
| `{0000}` | **Mandatory terminator** (block-scoped; Q7 END-fill pads use END-codes, never `0x00`) |
| `%A` / `%B` / `%H` | Conditional string composition (e.g., Alena ID 120 dialogue dispatch) |
| particle swap `は/を/った` | Grammar particle handlers resolved through the Type-39 item-use token order |

Speaker attribution and character interpolation run through the name layer: `{7F3F}`/`{7F3D}` write a name via a placeholder slot resolved by `0x8008F280`'s name sub-decode — the sub-decode that produced the famous `+8` first-glyph-drop artifacts when a name sequence lacked a pristine marker leaf.

### 5.2 Dynamic pointer resolution (split-immediates)

The MIPS build addresses packed words two ways:

1. **Direct words** — `(w >> 20) == BlockID` patterns; rewritable in place.
2. **Split-immediates** — `lui id` + `ori/addiu bitoff` instruction pairs whose low 20 bits resolve to a seq start. A **32-instruction dataflow scanner** plus an **`ADDIU` sign-carry rewriter** (`hbe/splitimm.py`, 248 lines) proves pair provenance and carries resolved immediates through every re-pack. The canonical failure — a `lui`/`ori` pair straddling a rewritten boundary smashing fastmem with `0xFFFFFFFF` — is the `numNodes=0` church-freeze class (§6.4).

Remap correctness gates: resolve-to-pristine-start; 1-bit-shift noise control; table-context ordinals at ±4/±8/±44 (strided record tables); exclusion of embedded text blocks; slot-fit ladder `full(4,8,44) → dense(4,8) → none → ABORT`. For `0x048B` the gate requires **1,625/1,625 direct words and 368/368 split-immediates resolve** (`gate_battle_commands_048b.py`).

### 5.3 Facility overlay remapping & pipeline dependency ordering

Facility overlays are multi-copy: Burland `0x048C`/`0x048F` across **5 sites** — 4 compressed sites (type-44, `dlen=109512` / `ulen=234576`) plus 1 raw site (sec `105026` sub 1, `flags=0x0000`) — 362 facility referrers total. Rules the pipeline must obey:

1. **Ordering is dependency-sensitive:** STEP `4f-2` (facility overlay) must run **after** `4g-3` (table re-encoding, incl. Endor `0x0021`) and **before** `4g-4` (type-39 recommit); otherwise `0x048F` SID offsets written by 4f-2 are stale after 4g re-encodes (`048F 337→338` once shifted 1,086 refs across 3 uncompressed `0x002C` tables, `dlen=177340`).
2. **Multi-copy parity:** embedded `type-46` copies inherit remaps via STEP `4c`; byte-parity 100% required across all copies of a given module site.
3. **The 362-pair gate** (`gate_g2b_facility_362.py`) must PASS: 362/362 referrers resolve, multi-copy byte-parity 100%.

---

## 6. Mathematical Invariants & Verification Pipeline Rules

The formal acceptance criteria for re-authoring an HBE title are:

### 6.1 Delta-lock preservation & boundary constraints

```
∀ SID edits:  ∀ k ≥ i:  ── leaf-bit boundary of SID k must not move ──
delta(i) = codebudget shift, constrained to the (44, 36, 50, 66) measured lifetimes
            across SIDs 773–778 for Font 1 lock groups
round-trip:  |natural_out − ulen| ≤ +3      (pristine parity: 0..+3)
slot:        len(rec) ≤ dlen
```

Pristine streams legally overrun `ulen` by `0..+3` bytes (the "≤ +3 decode contract"); padded streams that emit `+3..+4,764` bytes corrupt the next block's tree header — the church-freeze & NumNodes-0 crash. `0x048C` delta-locks: 57/57; `0x048F`: 8/8; battle SIDs 102/114.

### 6.2 Monotonicity of string offset tables

All `DP_OFFSETS[]` (SID→bit offset) tables must stay **strictly monotone non-decreasing**, and every resolver target must land **exactly on a seq start** — no residue, no interior leaf. A table that regresses reads as "document freezes"; a table that returns non-exactly produces the `+6/+8/+19` bit-resume first-glyph drops (§4.2).

### 6.3 Zero-displacement LBA disc-topology invariance

- Full disc image is Mode 2 Form 1, 156,487 LBA, 368,057,424 B.
- All 67 distinct overlay modules fit within original `dlen` (measured margins +10 B..+3,376 B; battle overlay +2,586 B) — `len(recompressed) ≤ orig_dlen`, hence **zero sector shift**.
- Payload deltas are confined to `manifest`-listed sectors: Rebuild B `DQ4_REBUILD_B.vpbin` v3 `RB1-Hotfix1` — 33,794,548 B, **14,344 patched sectors**, pristine SHA-1 `85064625AFA12219880FC8D07047A3CC1C595CB9`, output SHA-256 `055D37CE649BB1156D82670CF793D42A930062DD832F17D4E9DB5B926877761D`.
- Third-class canaries `0x0066–0x006C` + container `18683` sub 2 must be **byte-identical to pristine** (gate `gate_well_thirdclass_parity.py`); 148 raw type-44 cell blocks must never be round-tripped.

### 6.4 Headless telemetry invariants

Automated (no-controller) checks must observe:

- **Directory pointer integrity:** after every transition, the 12-slot dir at `0x80100168` must repopulate fully — never an empty self-referential ring at `0x800F51EC..0x800F5208`.
- **Unblank latch timing:** `0x800F84E4` must complete `0 → 1` before the F3 dining/descent watchpoint (Healie well Ch1 B1F→B2F, sector `106502`, TID `006C`); a steady `0` with a ~354 FPS spin-loop after DMA3 drain = repopulation failure.
- **EDC/ECC:** every edited Mode 2 Form 1 sector re-encodes its EDC/ECC (8,280 sectors in Rebuild B).
- Gate suite: `G2 church facility 872/872`, `G3 Endor 0x0021` (1,086 refs × 3 tables), `G4 multicopy 19/19`, `G8b varcode 33/33`, `G9 EDC/ECC`, `F9 gate_type39`, `F10 gate_monster_name_table`, plus the Sep-14 additions `gate_well_thirdclass_parity.py`, `gate_battle_commands_048b.py`, `gate_g2b_facility_362.py`.

---

## 7. Appendix A — C Struct Definitions

```c
/* PS1 DQ4 HBD on-disc archive master header — 16 bytes                    */
typedef struct {
    uint32_t nsub;   /* sub-block (record) count                            */
    uint32_t nsec;   /* sector count / span of archive                      */
    uint32_t tlen;   /* archive total length in bytes                       */
    uint32_t zero;   /* padding; MUST be 0                                  */
} hbd_archive_header_t;

/* Per-sub-block metadata record — 16 bytes, follows master header          */
typedef struct {
    uint32_t dlen;   /* compressed slot size (publish budget)               */
    uint32_t ulen;   /* decompressed logical length (declared)              */
    uint32_t extra;  /* extra per-type header parameter                     */
    uint16_t flags;  /* 0x0500 => DQLZS-compressed                          */
    uint16_t type;   /* sub-block type code: 23,24,25,26,27,39,40,42,44,46  */
} hbd_subblock_t;

/* Nested Huffman text-block container header — 24 bytes                    */
typedef struct {
    uint32_t end;      /* absolute end offset of block                      */
    uint32_t id;       /* block id / TID                                    */
    uint32_t hts;      /* header+tree store size; 0x18 for DQ4              */
    uint32_t treeEnd;  /* end of serialized tree region                     */
    uint32_t textEnd;  /* end of compressed text window                     */
    uint32_t unk1;     /* reserved                                          */
} hbe_text_header_t;

/* Huffman tree serialization header — 10 bytes, at hts region              */
typedef struct {
    uint32_t treeStart;
    uint32_t treeMiddle;
    uint16_t numNodes; /* 2-byte LE nodes; 0x8000 branch bit; root at end-4 */
} hbe_tree_header_t;

/* DP table entry (SID → leaf-bit offset) — follows marker(4)+count(4)      */
typedef struct { uint32_t bit_offset; } hbe_dp_entry_t;

/* Font 2 (proportional Shift-JIS) chain record — 8 bytes, code at +4!      */
typedef struct {
    uint32_t descriptor; /* bits 0..19 pixel index; 20..27 slot; 28..31 rec */
    uint16_t code;       /* char code (offset +4, NOT +2)                   */
    uint8_t  width;      /* real widths: caps 9.3px, lc 7.4px, kanji 11.9  */
    uint8_t  height;
} hbe_font2_chain_t;

/* 12-slot message-directory entry — stride 0x20 at 0x80100168             */
typedef struct {
    uint8_t  pad[0x0C];
    uint32_t P;          /* +0x0C  plane P pointer                          */
    uint32_t M;          /* +0x10  plane M pointer                          */
    uint8_t  pad2[0x04];
    uint16_t filter;     /* +0x1A  slot filter (TID high slice)             */
    uint16_t flags;      /* +0x1C  block flag register                      */
} hbe_dir12_entry_t;     /* total 0x20 = 32 B                               */
```

---

## 8. Appendix B — Dispatch & Census Tables

### 8.1 EXE dispatch table (verified against `SLPM_869.16`)

| Addr | Function |
|---|---|
| `0x8008F280` | Packed SID resolver — `srl 20`; 12-slot dir walk (`0x80100168`, stride 0x20); bit-shift + directory return (`0x8008F318-344`) |
| `0x8008F3BC` | String renderer — single/double-byte SJIS dispatch; `0x10` = 8-byte control; plane latch `.P`/`.M` by `t1>>28` |
| `0x8008F7B0` | Fixed-stride sentinel lookup walk — `divu` remainder + linked sentinel compare |
| `0x8008F9A0` | 2-bit planar cell unpacker (`0xCCCC`/`0x3333` reorder, 32-cell groups) |
| `0x8008FA30–0x8008FB40` | 2bpp RLE glyph expander |
| `0x8008FB48` | Map-transition directory zeroing & re-chain loop |
| `0x8008F810` | `dma3_sync_stream_decompress` (DQLZS via DMA3 @ `0x800E8000`) |
| `0x8009A120` | CD read-timeout evaluator / spindle recovery (`CdlPause` 0x06 @ `0x1F801801`) |
| `0x8009A240` | Mode 2 Form 1 checksum & error trap — 11-bit sector-completion mask sub-handlers |

### 8.2 Census

| Measure | Count |
|---|---|
| SIDs | 19,193 |
| HBD blocks | 1,108 (archive) + 4 overlays |
| Type-39 scripts | 403 |
| TID census (Sep-14 audit) | 1,128 = 893 type-40 + 213 type-42 + 22 EXE/facility |
| VM control tokens | 51 (43 archive + 8 EXE) |
| Overlay module types / sites | 67 / 267 |
| Endor `0x0021` | 93,140 B / 1,087 strings |
| `0x048B` battle overlay | 817 sequences; 1,625 dir words + 368 split-imm |
| `0x048C` delta-locks | 44/36/50/66 across SIDs 773–778; 57/57 resolved |
| Facility `0x048C`/`0x048F` | 5 Burland sites / 362 referrers; 8/8 (048F) |
| Raw type-44 cell blocks (preserved) | 148 |
| Third-class canaries (preserved) | `0x0066–0x006C` |

---

## 9. Appendix C — Source Traceability

- Format: `translation-tools/hbe/referrers.py:50-63` (archive header + record), `hbe/archive.py:188-237`, `hbe/textblock.py:26-82`, `hbe/parser/text.py:58-74`, `hbe/splitimm.py`, `hbe/huffman/length_limited.py`, `hbe/tree_cracker.py:175-253`, `hbe/mips.py`, `snes/clone/DQIV_PSX_TOOLS/docs/FONTS.md`.
- Dispatch audit: `study/DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md` (function bodies, header parse, container classes, font-page latch, two-key rule).
- Well & canary: `study/MASTER_FIX_WELL_THIRDCLASS_AND_FACILITY_ORDERING_Sep14_2026.md`, `study/WELL_FLOOR_COMPRESSION_FORENSICS_Sep13_2026.md`, `study/BUG_TRIAGE_AND_FIX_PLAN_Sep14_2026.md`, `study/MASTER_FIX_REBUILD_C_RECONCILED_Sep14_2026.md`, `study/RAM_DUMP_FINDINGS_196_197_Sep14_2026.md`.
- RAM/MIPS forensics chapters: `study/CHAPTER_{1,2,4,5,6}_FORENSIC_RAM_AND_MIPS_OVERLAYS.md`, `study/DEFINITIVE_REBUILD_C_BLUEPRINT_CH1-6_Sep12_2026.md`, `study/COUNTER_AGENT_VERIFICATION_BRIEF{,_BUILD_READY}_Sep14_2026.md`.
- Registers & lineages: `YAMANA_HBE_MASTER_TID_LIBRARY.md`, `YAMANA_HBE_MASTER_SID_LIBRARY.md`, `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md`, `HBE_GENERATIONAL_TRANSLATION_HISTORY.md` (public mirror: [`dq4frankenstein`](https://github.com/luxauraofficial777/dq4frankenstein), branch `main`).
- Builds: Rebuild B `AC9F94A1…` / payload `DQ4_REBUILD_B.vpbin` v3 `RB1-Hotfix1`; Rebuild C playtest-ready `FF1AF239`; post-fix `A9CD6903E39B8A6C85345DF125500E80099178F4A971D6B582DB8ED4EBD8322B`.

---

*End of specification. Prepared for the VoidWalkers Research Project under CC BY-NC-SA 4.0. Offsets, opcodes, LBAs, and counts trace to the cited study corpus of 2026-09-14.*