# Pipeline Root Cause Discovery — Jul 19, 2026

## The Smoking Gun

**The DQ4 HBD is written to the final disc RAW — zero modification.**

Every build version (v19 through v39) extracts the DQ4 HBD from `dq4_heart_v17.bin` and writes it directly to the output disc. No block header patching. No LBA shim. No tree format conversion. No sub-block type/count reinterpretation.

### Evidence

In `psx_binary_ops.cpp` `build_v39()`:

```cpp
// Step 8: Extract
auto hbd_data = extract_hbd_from_disc(dq4_hbd_disc_path, DQ4_HBD_DISC_START, DQ4_HBD_SIZE);

// Step 9: Write RAW to disc
write_raw_data(iso, HBD_LBA, hbd_data);
```

No processing step between extraction and disc write. The `HbdReencoder` class exists with `parse_block()` and `reencode_block()` but is **never called** in any build path.

### What `dq4_heart_v17.bin` Contains

- 368,057,424 bytes (351 MB) — a full DQ4 disc image
- Has English text already encoded with hybrid Huffman tree (from Build 6, Jun 30)
- HBD starts at sector 362 in this disc
- DQ4 HBD block format: `hts=24` (no gap), `treeEnd/textEnd` non-zero, per-block trees
- Sub-block types in lower 16 bits (32, 39, 40, 42, 46) — NOT counts

### What the DW7 EXE Expects

- DW7 HBD block format: `hts≈1366` (gap), `treeEnd=0`, `textEnd=0`, global tree
- Sub-block count in lower 16 bits, tree index in upper 16 bits
- Global Huffman tree at `0x800EF1C8` (relocated to `0x80100000`)

## What Was Actually Done (All Versions)

| Step | Status | Notes |
|------|--------|-------|
| Separate EXE from HBD | DONE | Extract both from source discs |
| Patch DW7 EXE (LBA, tree refs, disc check) | DONE | Multiple EXE patches applied |
| Patch DQ4 HBD (headers, trees, sub-block format) | **SKIPPED** | Raw HBD written to disc |
| Assemble final disc | DONE | EXE + raw HBD + ISO dir + PVD + EDC/ECC |

## What Should Be Done (Localization Team Approach)

### HBD Processing Pipeline (Missing Step)

1. **Parse each DQ4 HBD block** using `HbdReencoder::parse_block()`
2. **Convert block headers to DW7 format**:
   - Set `hts` to DW7 gap size (~1366) or keep at 24 and adjust EXE parser
   - Set `treeEnd = 0`, `textEnd = 0` (use global tree, not per-block)
   - Remove per-block Huffman trees
3. **Fix sub-block type/count fields**:
   - DQ4: lower16 = type (32,39,40,42,46), upper16 = count
   - DW7: lower16 = count, upper16 = tree index
   - Must swap or reinterpret for DW7 EXE compatibility
4. **Fix internal LBA references** within HBD data:
   - DQ4 HBD was at sector 362 in original disc
   - In Frankenstein disc, HBD is at sector 355
   - All internal sector references must be adjusted by delta (-7)
5. **Apply hybrid tree** to text blocks (already done in v17, but tree format must match DW7's global tree convention)
6. **Output processed HBD** as a clean binary, ready for disc assembly

### Build Pipeline (Corrected)

```
1. Extract DW7 EXE → patch (LBA, tree refs, disc check, trampoline)
2. Extract DQ4 HBD → PROCESS (headers, trees, sub-blocks, LBAs) ← THE MISSING STEP
3. Assemble disc: patched EXE + processed HBD
4. Update ISO dir + PVD + EDC/ECC
```

## Available Tools (Unused)

- `HbdReencoder` class in `psx_binary_ops.cpp` — has `parse_block()`, `reencode_block()`, `handle_zero_length_tree()`
- `ParsedBlock` struct — captures all block fields
- Hybrid tree (1480 bytes, 370 leaves) — ready to embed
- `dw7_encoded_translation.json` — 15,318 entries already encoded

## HBD Source Files

| File | Size | Description |
|------|------|-------------|
| `dq4_heart_v17.bin` | 368 MB | Full DQ4 disc with English text (Build 6) |
| `translation/HBD1PS1D.Q41` | 319 MB | Raw DQ4 HBD extracted |
| `translation/HBD1PS1D.Q41_PATCHED.bin` | 319 MB | Patched HBD (Jul 4) |
| `translation/HBD1PS1D.W71` | 618 MB | DW7 HBD extracted |

## Key Constants

- DQ4 HBD disc start: sector 362
- DQ4 HBD size: 319,436,800 bytes (155,975 sectors)
- Frankenstein HBD LBA: 355
- LBA delta: 355 - 362 = **-7** (DQ4 internal refs need -7 adjustment)
- DW7 HBD LBA original: 354
- DW7 gap size: ~1366 bytes
- DQ4 gap size: 0 (hts=24)

## Next Steps

1. Implement HBD processing in C++ (`process_hbd()` method)
2. Call it between extraction and disc write in `build_v39()`
3. Verify processed HBD block headers match DW7 format expectations
4. Build and test in DuckStation
