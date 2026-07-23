# HBD Sub-Block Type Trampoline Diagnostic — Jul 19, 2026

## Status: Analysis Complete — Patch Designed — Implementation Pending

---

## 1. Schroeder HBD Schema Alignment

From `HBD1PS1D.java`, the DQ4 HBD archive structure:

```
VeryFirstBlock (sector 0, 2048 bytes)
  → StarZeros blocks (* 00 00 00 header)
    → StarZerosSubBlock entries (16-byte headers):
      [fsize:4] [fsize_unc:4] [unknown:4] [flags:2] [type:2]
  → H60010180 blocks (audio/ADPCM)
```

### Sub-Block Type Classification

| Type | Category | DQ4 Count | DW7 Equivalent |
|------|----------|-----------|----------------|
| 32 | Scenario/script data (uncompressed, flags=0) | 32 | Not present in DW7 |
| 39 | Cutscene script / dialog commands (flags=1280, compressed) | 976 | DW7 uses types 23/24/25/27 |
| 40 | DQ4 text block (Huffman-compressed dialog) | 1315 | DW7 uses type 23/24 |
| 42 | DQ4 text block variant | 213 | DW7 uses type 25/27 |
| 44 | Script data | 152 | DW7 uses type 22 |
| 46 | Message/event data (flags=1280, compressed) | 612 | DW7 uses type 26 |

**Critical mismatch:** DW7 EXE's parser recognizes `dq7TextTypes = [23, 24, 25, 27]` as text blocks. DQ4 HBD contains types `40, 42` (text) and `39, 46` (cutscene/message). The DW7 parser has no handler for types 39, 40, 42, 46 — it falls through to default case, computing wrong sector offsets.

---

## 2. DW7 EXE Dialog-Fetch Kernel — Instruction Trace

### Layer 1: Mount/Dispatcher — 0x8002EC2C (EXE offset 0x16D2C)
- Main HBD mount entry. Reads folder table at HBD offset 0x804.
- Calls folder count dispatcher at 0x8004BBE4 with $a0=2.
- `sltiu $v0, $a2, 4` at 0x8004BDC8 — supports up to 3 folders. No patch needed.

### Layer 2: Resource Dispatcher — 0x8002F238 (6 instructions)
```mips
0x8002F238: lui $v0, 0x800E         # load base
0x8002F23C: lw  $v0, 1360($v0)     # read [0x800E0550] — HBD mount status
0x8002F240: nop
0x8002F244: bne $v0, $zero, 0x8002F258  # if mounted, branch
0x8002F248: nop
0x8002F24C: jr  $ra                # return 0 (not mounted)
```
Return == 1 → path A (jal 0x8010BCA8), Return != 1 → path B (jal 0x8010F31C).

### Layer 3: Block Header Reader — 0x800429B0–0x80042A00 (21 instructions)
```mips
0x800429B0: addiu $sp, $sp, -32
0x800429B4: sw    $s0, 16($sp)
0x800429B8: addu  $s0, $a0, $zero     # $s0 = block data pointer
0x800429BC: sw    $s1, 20($sp)
0x800429C0: sw    $ra, 24($sp)
0x800429C4: jal   0x8002F238          # call dispatcher
0x800429C8: addu  $s1, $a1, $zero     # $s1 = dest buffer (delay slot)
0x800429CC: addiu $v1, $zero, 1
0x800429D0: bne   $v0, $v1, 0x800429EC  # path select
0x800429D4: addu  $a0, $s0, $zero
0x800429D8: jal   0x8010BCA8          # path A: standard decompression
0x800429DC: addu  $a1, $s1, $zero
0x800429E0: j     0x800429F0
0x800429E4: nop
0x800429E8: jal   0x8010F31C          # path B: alternate decompression
0x800429EC: addu  $a1, $s1, $zero
0x800429F0: lw    $ra, 24($sp)        # epilog
0x800429F4: lw    $s1, 20($sp)
0x800429F8: lw    $s0, 16($sp)
0x800429FC: jr    $ra
0x80042A00: addiu $sp, $sp, 32
```

### Layer 4: Huffman Decompressor — 0x80073670 (516 bytes, 129 instructions)
- 9 call sites in 0x8007EF00–0x800907A0 range.
- 6 `andi 0x7FFF` sites: 0x8007EFF0, 0x8007F0F8, 0x8007F278, 0x8007F898, 0x80080FE4, 0x80081198.

### Layer 5: Tree Loader — 0x800561C8–0x800563D0 (131 instructions)
- Loads Huffman tree from HBD data into global tree buffer at 0x800EF1C8.
- $s3 register holds sub-block type/count info.
- At 0x800562EC–0x800562F0: `lui $v0, 0x800F` + `addiu $v0, $v0, -3640` → 0x800EF1C8.

---

## 3. Divergence Point

**Address: 0x800562E4** (EXE offset 0x3E3E4)

The DW7 parser at 0x800561C8 reads the sub-block entry from the folder table. The 16-byte sub-block header (Schroeder format):

```
[fsize:4] [fsize_unc:4] [unknown:4] [flags:2] [type:2]
```

At 0x80056278: `lw $s3, 0($s1)` loads the first word (fsize).
At 0x800562E0: `andi $s3, $s3, 0xFFFF` extracts lower 16 bits.
At 0x800562E4: `beq $s3, $zero, 0x80056320` branches if count == 0.

DW7 parser uses $s3 (lower 16 bits) as a COUNT of entries. DQ4 HBD encodes sub-block TYPE (39, 46) in this field. When type=39, DW7 interprets it as 39 entries → computes 39 * 32 = 1248 bytes offset → wild lseek calls.

### CyberGrime Trace Evidence
- 176 wild lseek calls starting at instruction ~22.8M
- DW7 reads sequential sectors (0x26A, 0x26B, 0x26C...)
- Frank jumps wildly (0x39F9, 0x1A3FC, 0x1A394...)
- Confirms: DW7 EXE misinterprets DQ4 block headers → incorrect sector offsets

---

## 4. Target Memory Addresses

| Address | EXE Offset | Role | Patch Target |
|---------|-----------|------|-------------|
| 0x80056278 | 0x3E378 | lw $s3, 0($s1) — loads sub-block header | Reference |
| 0x800562E0 | 0x3E3E0 | andi $s3, $s3, 0xFFFF — extracts type/count | Reference |
| **0x800562E4** | **0x3E3E4** | **beq $s3, $zero, 0x80056320** — branch if count==0 | **Primary injection point** |
| 0x800562E8 | 0x3E3E8 | sh $v1, 1336($v0) — stores type index | Displaced by JAL |
| 0x800429D0 | 0x2A9D0 | bne $v0, $v1 — decompression path select | Secondary |
| 0x80073670 | — | Huffman decompressor entry | Not patched |
| 0x800EF1C8 | — | Global Huffman tree (RAM) | Already redirected to 0x800BC700 |

---

## 5. MIPS Trampoline Patch

### Injection Point
- Address: 0x800562E4 (EXE offset 0x3E3E4)
- Original: `beq $s3, $zero, 0x80056320` (0x1260000D)
- Replace with: `jal 0x80101000` + `nop`

### Binary Patch Bytes

| Offset | Original (LE) | Patched (LE) | Instruction |
|--------|--------------|-------------|-------------|
| 0x3E3E4 | 0D 00 60 12 | 00 04 04 0C | jal 0x80101000 |
| 0x3E3E8 | 38 05 43 A4 | 00 00 00 00 | nop |

### Trampoline at 0x80101000

```mips
# DQ4 HBD Sub-Block Type Handler Trampoline
# Placed at 0x80101000 (high RAM, outside BSS clear 0x800BC668-0x800F4980)

# Entry: $s3 = lower 16 bits (DW7 treats as count, DQ4 encodes type)
#         $v1 = upper 16 bits (index/flags)
#         $s1 = current sub-block entry pointer
#         $s4 = context struct pointer
#         $s2 = loop counter

# Step 1: Load actual type field from sub-block header
lw    $t0, 12($s4)          # sub-block header array pointer
nop
addu  $t0, $t0, $s2         # current entry
nop
lhu   $t1, 0x0E($t0)        # type field at offset 14
nop

# Step 2: Type dispatch — check DQ4-specific types
addiu $t2, $zero, 39
beq   $t1, $t2, .dq4_handler
nop
addiu $t2, $zero, 40
beq   $t1, $t2, .dq4_handler
nop
addiu $t2, $zero, 42
beq   $t1, $t2, .dq4_handler
nop
addiu $t2, $zero, 46
beq   $t1, $t2, .dq4_handler
nop
addiu $t2, $zero, 32
beq   $t1, $t2, .dq4_handler
nop

# Step 3: Not DQ4 type — restore original behavior
lui   $v0, 0x800E          # restore $v0 base
sh    $v1, 1336($v0)       # original 0x800562E8 behavior
beq   $s3, $zero, .original_skip
nop
jr    $ra                   # count != 0, continue original processing
nop

.original_skip:
j     0x80056320            # count == 0, skip
nop

# Step 4: DQ4 handler — reinterpret type as type, not count
.dq4_handler:
srl   $s3, $v1, 0           # $s3 = upper 16 bits = real entry count
addiu $v1, $zero, 0         # clear consumed field
lui   $v0, 0x800E           # restore $v0 base
sh    $v1, 1336($v0)        # original 0x800562E8 behavior (store 0)
jr    $ra                    # continue with corrected $s3
nop
```

---

## 6. Implementation Status

- [x] Schroeder HBD schema analysis (types 32, 39, 40, 42, 46)
- [x] DW7 EXE dialog-fetch kernel traced (5 layers identified)
- [x] Divergence point isolated at 0x800562E4
- [x] MIPS trampoline designed
- [x] Binary patch bytes computed
- [x] Trampoline injected into build pipeline (psx_binary_ops.cpp)
- [x] Binary patch applied to EXE in build (build_v39 step 6b)
- [x] Python ctypes wrapper updated (frankenstein_builder.py)
- [x] C++ compilation verified (psx_ops.dll built successfully)
- [ ] Build and test in DuckStation
- [ ] CyberGrime telemetry verification

### Build Integration Details

**C++ Implementation** (`cybergrime/psx_binary_ops.cpp`):
- `ExePatcher::patch_hbd_type_trampoline()` — patches 0x800562E4 to `jal 0x80101000`, pads EXE to 0x80101000, appends 34-instruction trampoline (136 bytes)
- Called in `FrankensteinBuilder::build_v39()` as step 6b (after tree refs, before reloc payload)
- `BuildResult.trampoline_patches` field added to track success

**Python Wrapper** (`translation-tools/frankenstein_builder.py`):
- `BuildResult` ctypes struct updated with `trampoline_patches` field
- Print output includes trampoline patch count

**Trampoline at 0x80101000** (34 instructions):
- Instructions 0-5: Load type field from sub-block header
- Instructions 6-20: Type dispatch (beq for types 39, 40, 42, 46, 32)
- Instructions 21-26: .not_dq4 — original DW7 behavior (sh $v1, beq $s3, jr $ra)
- Instructions 27-28: .skip — j 0x80056320 (original skip target)
- Instructions 29-33: .dq4_handler — addu $s3, $v1, $zero (use upper 16 bits as count)
