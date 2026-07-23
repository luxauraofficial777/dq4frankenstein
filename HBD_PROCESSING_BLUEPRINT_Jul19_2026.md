# HBD Processing Pipeline Blueprint — Jul 19, 2026

## Author: Cascade (Principal Systems Engineer mode) | Status: Ready for implementation

---

## 1. Problem Statement

The DW7 EXE's HBD parser expects a specific archive structure. DQ4 HBD data has the same hierarchical layout (sectors → folders → files → blocks) but differs at the block level:

| Field | DQ4 HBD | DW7 HBD | Impact |
|-------|---------|---------|--------|
| `hts` (header text start) | 24 (no gap) | ~1366 (gap present) | DW7 parser skips gap bytes; DQ4 has none → misaligned reads |
| `treeEnd` / `textEnd` | Non-zero (per-block tree) | Zero (global tree) | DW7 parser uses global tree; DQ4 has inline trees → wrong decompression |
| Sub-block lower16 | Type ID (32,39,40,42,46) | Count | DW7 reads count, gets type → wild LBA seeks |
| Sub-block upper16 | Count | Tree index | DW7 reads tree index, gets count → wrong tree node |
| Internal sector refs | Relative to LBA 362 | Relative to LBA 354 | Off by -7 in Frankenstein disc (362→355) |

## 2. HBD Archive Structure (Both Games)

```
HBD File (319 MB DQ4 / 618 MB DW7)
├── Sector 0: Binary header (2048 bytes)
├── Sector 1+: Folders
│   ├── Folder Header (16 bytes): file_count, sector_count, size, unknown
│   ├── File Headers (16 bytes each): size, size_uncompressed, unknown, flags, type_id
│   └── File Contents (concatenated, padded to sector boundary)
│       └── Text Block (for type 19/23/24/25/27/32/39/40/42/46)
│           ├── Block Header (24 bytes): end, block_id, hts, reserved, treeEnd, textEnd, unknown
│           ├── Gap (bytes 24..hts) — DW7 only, ~1342 bytes
│           ├── Text Data (bytes hts..textEnd) — Huffman-encoded
│           ├── Tree Header (10 bytes at textEnd): tree_start, tree_middle, tree_nodes
│           ├── Tree Data (tree_start..treeEnd) — DQ4 only
│           ├── Post-tree padding (treeEnd..end)
│           ├── at_end (4 bytes at offset end)
│           ├── dp_count (4 bytes)
│           └── Dialog pointers (dp_count × 8 bytes)
└── Zero/padding sectors
```

## 3. Processing Strategy

### 3.1 What NOT to Change
- **Folder structure**: Both games use identical folder/file hierarchy
- **File headers**: 16-byte file headers are the same format
- **Sector 0 binary header**: Same signature (`2923BE84E16CD6AE...`)
- **Non-text files**: Binary assets (sprites, maps, audio) are game-specific but format-compatible

### 3.2 What TO Change (3 Transformations)

#### Transformation A: Block Header Conversion (per text block)

For each text block inside each file inside each folder:

```
DQ4 header:  end=X  block_id=Y  hts=24  reserved=0  treeEnd=T  textEnd=E  unknown=U
DW7 target:  end=X' block_id=Y  hts=1366 reserved=0  treeEnd=0  textEnd=0  unknown=U
```

**Logic:**
1. Parse 24-byte header from file content
2. Extract text data: `text_bytes = content[24..textEnd]` (DQ4 has no gap, text starts at offset 24)
3. Extract tree data: `tree_bytes = content[tree_start..treeEnd]` (DQ4 per-block tree)
4. Extract dialog pointers: `dp_data = content[end+8..end+8+dp_count*8]`
5. Build new block:
   - Header: `hts=1366`, `treeEnd=0`, `textEnd=0` (signals "use global tree")
   - Gap: 1342 bytes of zeros (bytes 28..1366, matching DW7 convention)
   - Text: re-encoded with hybrid tree (already done in v17, but verify)
   - No inline tree (removed — EXE uses global tree at `0x80100000`)
   - Dialog pointers: preserved as-is
6. Recalculate `end` = 1366 + len(text_bytes) (no tree data)
7. Rebuild file content with new block

**Important**: The gap content in DW7 is NOT all zeros — it contains font/tile mapping data (sequential B0/C0, B1/C1 patterns). We need to either:
- (a) Copy the gap from the corresponding DW7 block (if we can map DQ4→DW7 blocks), OR
- (b) Leave it as zeros and hope the DW7 EXE doesn't read it for DQ4 content, OR
- (c) Determine what the gap actually does and generate appropriate content

**Recommendation**: Start with (b) zeros. The gap is likely font/tile mapping that the DW7 EXE reads separately from the global tree. If the game renders text but not fonts, we'll investigate (c).

#### Transformation B: Sub-Block Field Remapping

Within the DW7 EXE's HBD processing loop at `0x800562E4`, the engine reads a 32-bit word and splits it:
- `s3 = lower16` (DW7: count, DQ4: type)
- `v1 = upper16` (DW7: tree index, DQ4: count)

**Two approaches:**

**Approach 1 (Data-side, preferred)**: In each DQ4 text block, find the sub-block entry words and swap upper/lower 16 bits:
```
Original DQ4:  (count << 16) | type      → e.g., 0x00050020 (count=5, type=32)
DW7 format:   (tree_idx << 16) | count   → e.g., 0x00000005 (tree_idx=0, count=5)
```
This requires knowing WHERE in the block these words are. They're in the file header's `type_id` field (16-bit at offset 14 of each 16-byte file header) and possibly in the block data itself.

**Wait** — re-reading the trace: the sub-block type/count is in the **file header** `type_id` field, not in the block data. The DW7 EXE reads file headers and processes `type_id` as `(count << 16) | type` or `(type << 16) | count` depending on the game.

Actually, from the trace at `0x800562E4`:
- `$s3` = lower16 of a 32-bit word read from the HBD
- `$v1` = upper16 (extracted via `srl $v1, $s3, 16` and `andi $s3, $s3, 0xFFFF`)

This 32-bit word comes from the **file header** structure. In the HBD archive:
- Each file has a 16-byte header: `size(4), size_uncompressed(4), unknown(4), flags(2), type_id(2)`
- The `type_id` is a 16-bit field at offset 14

But the DW7 EXE reads a 32-bit word that combines `flags` and `type_id`:
- `flags` (offset 12, 2 bytes) + `type_id` (offset 14, 2 bytes) = 32-bit word
- DW7: lower16 = type_id (count), upper16 = flags (tree index)
- DQ4: lower16 = type_id (type), upper16 = flags (count)

**Fix**: For DQ4 files with type_id in {32, 39, 40, 42, 46}, swap the `flags` and `type_id` fields in the file header:
```
Original:  flags=COUNT  type_id=TYPE
Fixed:     flags=0      type_id=COUNT  (tree_idx=0 → use root)
```

This is a **2-word patch per file header** — swap flags and type_id, set flags to 0 (tree root index).

**Approach 2 (EXE-side, trampoline)**: The existing trampoline at `0x80101000` does this at runtime. Keep as fallback.

**Recommendation**: Do Approach 1 (data-side) in the HBD processing. Keep the trampoline disabled but compiled in, for debugging.

#### Transformation C: LBA Relocation (delta -7)

The DQ4 HBD was at disc sector 362. In the Frankenstein disc, it's at sector 355. Internal sector references within the HBD need adjustment.

**Where are internal LBAs?**
- Folder headers don't contain LBAs (they use sector_count, relative)
- File headers don't contain LBAs (they use size, relative)
- The DW7 EXE's ABS/REL pointer tables in the EXE contain LBAs — already patched by `patch_lba_references` and `remap_folder_pointers`
- **Within HBD data**: Some file types (e.g., STR/video, map data) may contain absolute sector references back into the HBD

**Logic:**
1. Scan HBD data for 32-bit values in range [362, 362+155975] (DQ4 HBD sector range)
2. For each match, subtract 7 (362→355 delta)
3. Log all patches for verification

**Risk**: False positives — random binary data may contain values in this range. Mitigate by only scanning in known file types that contain sector references (not in text blocks, not in compressed data).

**Recommendation**: Implement as a separate scan pass, log all changes, and verify count is reasonable (<1000 patches expected).

## 4. C++ Implementation: `HbdReencoder::process_hbd()`

### 4.1 Method Signature

```cpp
struct HbdProcessResult {
    uint32_t blocks_processed;
    uint32_t blocks_converted;    // headers converted DQ4→DW7
    uint32_t subblock_fields_swapped; // flags/type_id swaps
    uint32_t lba_patches;         // internal LBA relocations
    uint32_t errors;
    bool success;
};

HbdProcessResult HbdReencoder::process_hbd(
    uint32_t source_lba,    // 362 (DQ4 original)
    uint32_t target_lba     // 355 (Frankenstein)
);
```

### 4.2 Implementation Steps

```cpp
HbdProcessResult HbdReencoder::process_hbd(
    uint32_t source_lba, uint32_t target_lba)
{
    HbdProcessResult r = {};
    int32_t lba_delta = (int32_t)target_lba - (int32_t)source_lba; // -7

    // Phase 1: Parse archive structure (sectors → folders → files)
    // Reuse HBDArchive parsing logic from archive.py, implemented in C++
    uint32_t sector_idx = 0;
    uint32_t total_sectors = (uint32_t)hbd.size() / 2048;

    // Skip sector 0 (binary header)
    sector_idx = 1;

    while (sector_idx < total_sectors) {
        uint8_t* sec = hbd.data() + sector_idx * 2048;

        // Check if this is a folder header
        uint32_t file_count, sector_count;
        memcpy(&file_count, sec, 4);
        memcpy(&sector_count, sec + 4, 4);

        if (file_count > 0 && file_count <= 1000 &&
            sector_count > 0 && sector_count <= 1000) {
            // Parse folder
            uint32_t folder_offset = sector_idx * 2048;
            // Read file headers (16 bytes each, starting at offset 16)
            uint32_t content_start = 16 + file_count * 16;

            for (uint32_t f = 0; f < file_count; f++) {
                uint32_t fh_off = folder_offset + 16 + f * 16;
                if (fh_off + 16 > hbd.size()) break;

                uint32_t f_size, f_size_unc;
                uint16_t f_flags, f_type;
                memcpy(&f_size, hbd.data() + fh_off, 4);
                memcpy(&f_size_unc, hbd.data() + fh_off + 4, 4);
                memcpy(&f_flags, hbd.data() + fh_off + 12, 2);
                memcpy(&f_type, hbd.data() + fh_off + 14, 2);

                // Transformation B: Sub-block field remapping
                // DQ4 types: 32, 39, 40, 42, 46
                // For these, flags=count, type_id=type
                // Swap to: flags=0 (tree root), type_id=count
                if (f_type == 32 || f_type == 39 || f_type == 40 ||
                    f_type == 42 || f_type == 46) {
                    uint16_t count = f_flags;  // DQ4 stores count in flags
                    // Write: flags=0, type_id=count
                    uint16_t new_flags = 0;
                    uint16_t new_type = count;
                    memcpy(hbd.data() + fh_off + 12, &new_flags, 2);
                    memcpy(hbd.data() + fh_off + 14, &new_type, 2);
                    r.subblock_fields_swapped++;
                }

                // Get file content
                uint32_t content_off = folder_offset + content_start;
                // Calculate cumulative content offset
                // (need to sum sizes of all preceding files in this folder)
                // ... [implementation detail: track running offset]

                // Transformation A: Block header conversion
                // Only for text blocks (type 19/23/24/25/27/32/39/40/42/46)
                if (f_size >= 24 && content_off + 24 <= hbd.size()) {
                    uint32_t end, block_id, hts, reserved, tree_end, text_end;
                    memcpy(&end, hbd.data() + content_off, 4);
                    memcpy(&block_id, hbd.data() + content_off + 4, 4);
                    memcpy(&hts, hbd.data() + content_off + 8, 4);

                    if (hts >= 24 && hts < 65536 && end > hts && end < 1048576) {
                        // Valid block header
                        memcpy(&tree_end, hbd.data() + content_off + 16, 4);
                        memcpy(&text_end, hbd.data() + content_off + 20, 4);

                        bool has_per_block_tree = (tree_end != 0 && text_end != 0);

                        if (has_per_block_tree) {
                            // Convert DQ4 block to DW7 format
                            // 1. Save text data
                            // 2. Save dialog pointers
                            // 3. Rebuild with hts=1366, treeEnd=0, textEnd=0
                            // 4. Insert gap (zeros)
                            // ... [see Section 3.2 Transformation A]
                            r.blocks_converted++;
                        }
                        r.blocks_processed++;
                    }
                }
            }

            sector_idx += sector_count;
        } else {
            sector_idx++;
        }
    }

    // Phase 2: LBA relocation scan
    // Scan for 32-bit values in [source_lba, source_lba + total_sectors)
    // Apply delta
    uint32_t hbd_sectors = (uint32_t)hbd.size() / 2048;
    for (size_t i = 0; i + 4 <= hbd.size(); i += 4) {
        uint32_t val;
        memcpy(&val, hbd.data() + i, 4);
        if (val >= source_lba && val < source_lba + hbd_sectors) {
            uint32_t new_val = val + lba_delta;
            memcpy(hbd.data() + i, &new_val, 4);
            r.lba_patches++;
        }
    }

    r.success = (r.errors == 0);
    return r;
}
```

### 4.3 Key Design Decisions

1. **In-place modification**: `process_hbd()` modifies `hbd` vector in-place. The original DQ4 HBD is preserved on the source disc.

2. **Block size changes**: When converting block headers (adding gap, removing tree), block sizes change. This requires:
   - Rebuilding file content within the folder
   - Adjusting file `size` in file header
   - Repacking folder sectors
   - **This is the hardest part** — folder sector boundaries must be maintained

3. **Alternative: Minimal approach**: Instead of full block header conversion, just do:
   - Transformation B (sub-block field swap) — 4 bytes per file header, no size change
   - Transformation C (LBA relocation) — 4 bytes per match, no size change
   - Skip Transformation A (block header conversion) — rely on trampoline for tree handling

   This is **much safer** — no size changes, no repacking, just field patches. The trampoline handles the tree/type reinterpretation at runtime.

4. **Recommended phased approach**:
   - **Phase 1**: Implement B + C only (sub-block swap + LBA fix). No size changes. Test.
   - **Phase 2**: If Phase 1 insufficient, add Transformation A (block header conversion). Requires repacking.

## 5. Pipeline Integration in `build_v39()`

### Current (broken):
```cpp
// Step 8: Extract
auto hbd_data = extract_hbd_from_disc(...);

// Step 9: Write RAW ← BUG: no processing
write_raw_data(iso, HBD_LBA, hbd_data);
```

### Fixed:
```cpp
// Step 8: Extract DQ4 HBD
auto hbd_data = extract_hbd_from_disc(dq4_hbd_disc_path, DQ4_HBD_DISC_START, DQ4_HBD_SIZE);

// Step 8b: Process HBD — convert DQ4 format to DW7-compatible
HbdReencoder reencoder;
reencoder.load(hbd_data.data(), (uint32_t)hbd_data.size(), hybrid_tree.data(), (uint32_t)hybrid_tree.size());
auto proc_result = reencoder.process_hbd(DQ4_HBD_DISC_START, HBD_LBA);
printf("  HBD processed: blocks=%u converted=%u swaps=%u lba=%u errors=%u\n",
    proc_result.blocks_processed, proc_result.blocks_converted,
    proc_result.subblock_fields_swapped, proc_result.lba_patches, proc_result.errors);
if (!proc_result.success) {
    printf("  ERROR: HBD processing failed\n");
    return r;
}

// Get processed HBD
hbd_data.assign(reencoder.raw(), reencoder.raw() + reencoder.size());

// Step 9: Write PROCESSED HBD to disc
write_raw_data(iso, HBD_LBA, hbd_data);
```

### BuildResult Update
Add fields to `BuildResult`:
```cpp
uint32_t hbd_blocks_processed;
uint32_t hbd_blocks_converted;
uint32_t hbd_subblock_swaps;
uint32_t hbd_lba_patches;
```

## 6. Validation (Gate 3)

### Pre-write validation:
```cpp
bool HbdReencoder::validate_processed() {
    // Re-parse archive, verify:
    // 1. All folder headers have valid file_count/sector_count
    // 2. All file headers have valid sizes
    // 3. No file type_id is in {32,39,40,42,46} (should be swapped to counts)
    // 4. All block headers (if converted) have treeEnd=0, textEnd=0
    // 5. No 32-bit values in [362, 362+155975) remain (LBA refs fixed)
}
```

### Post-write validation:
- CyberGrime trace: verify DW7 EXE reads sequential sectors (not wild jumps)
- DuckStation: verify no black screen, Enix logo appears

### DuckStation Gate 3 criteria:
1. EXE loads and boots (already working)
2. HBD parser reads first folder without error
3. No wild LBA seeks in first 5M instructions
4. Enix logo or title screen appears

## 7. Implementation Order

1. **Add `process_hbd()` to `HbdReencoder`** — Phase 1 (B+C only, no repacking)
2. **Add `validate_processed()`** — verify no DQ4 types remain, no stale LBAs
3. **Integrate into `build_v39()`** — between extraction and write
4. **Update `BuildResult`** — track HBD processing stats
5. **Rebuild `psx_ops.dll`** — compile with new code
6. **Build v40** — run `build_frank_v39` with processed HBD
7. **Test in DuckStation** — Gate 3 verification
8. **If Phase 1 insufficient**: implement Phase 2 (Transformation A, block header conversion with repacking)

## 8. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| LBA scan false positives | Medium | Corrupt non-LBA data | Limit scan to non-text, non-compressed files |
| Sub-block swap wrong field | Low | Wrong count/type | Verify with trace data before build |
| Block repacking errors | High (Phase 2) | Corrupt folder structure | Phase 1 first (no repacking) |
| Gap content needed | Medium | Font/rendering issues | Start with zeros, investigate if needed |
| HBD size changes | High (Phase 2) | ISO dir size mismatch | Phase 1 preserves size exactly |

## 9. File Inventory

| File | Role |
|------|------|
| `cybergrime/psx_binary_ops.cpp` | Add `process_hbd()`, `validate_processed()` to `HbdReencoder` |
| `cybergrime/psx_binary_ops.h` | Add method declarations, `HbdProcessResult` struct |
| `translation-tools/frankenstein_builder.py` | Update `BuildResult` ctypes struct, print HBD stats |
| `cybergrime/hbd_diag.cpp` | Diagnostic tool (already created, needs archive-aware parsing) |
