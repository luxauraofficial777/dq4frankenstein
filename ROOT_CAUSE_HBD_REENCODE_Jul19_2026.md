# Root Cause: HBD Text Re-Encode Gap — Jul 19, 2026

## Summary

The Frankenstein build has been failing for weeks because **DQ4 HBD text is never re-encoded from DQ4 Huffman format to DW7 Huffman format**. The `process_hbd()` function strips per-block tree headers and reformats block headers to DW7 style (hts=1366, treeEnd=0, textEnd=0), but **copies the text bytes verbatim** — they remain encoded with DQ4's per-block Huffman trees. The DW7 EXE tries to decode them with the global hybrid tree and gets garbage, causing the hang.

## The Gap (Confirmed)

### What `process_hbd()` currently does:
1. **Transformation A (Block header):** Strips per-block tree, sets hts=1366, treeEnd=0, textEnd=0 — ✅ correct
2. **Transformation B (Sub-block swap):** Swaps flags/type_id fields — ✅ correct
3. **Text re-encoding:** **MISSING** — text bytes are copied as-is from DQ4 format ❌

### What it SHOULD do:
For each DQ4 block with a per-block tree (`has_per_block_tree == true`):
1. **Decode** text bytes using the block's DQ4 per-block tree → list of leaf values (2-byte content)
2. **Re-encode** those leaf values using the hybrid tree (`dw7_hybrid_tree.bin`, 1480 bytes, 370 leaves) → new Huffman-compressed text bytes
3. Store the re-encoded text in the converted DW7-format block

This is the bridge that makes DQ4 data readable by the DW7 EXE.

## Huffman Codec Details (from Python `hbe` library)

### Tree Format (both DQ4 and DW7):
- 2-byte LE nodes, `0x8000` branch bit, `0x7FFF` index mask
- Dual-array layout: Array A at offset 0, Array B at offset `(len+2)//2 - 2`
- Root pointer at `len-4`: value = `0x8000 | root_id`
- DQ4: `root_id = (value & 0x7FFF) + 1` (root_plus_one=True)
- DW7: `root_id = (value & 0x7FFF)` (root_plus_one=False, no +1)

### Bitstream Format:
- Bits within each byte are **reversed** (LSB first after reversal)
- `0` = left child, `1` = right child
- Padded to multiple of 4 bytes (32 bits)

### Decode:
```
bits = reverse_each_byte(text_bytes)  → bit string
node = root
for bit in bits:
    node = (bit == 0) ? node.left : node.right
    if node.is_leaf():
        leaves.append(node.content)  // 2-byte value
        node = root
```

### Encode:
```
codes = walk_tree(root)  // content → bit string mapping
bits = concatenate(codes[leaf.content] for leaf in leaves)
output = reverse_each_byte(bits)  // pad to 4-byte boundary
```

### Leaf Value Classification:
- `value == 0` or `(value & 0xFF00) == 0x7F00` → control character
- Otherwise → character (Shift-JIS code point with high byte stripped)

## Key Files

- **Hybrid tree:** `dw7_hybrid_tree.bin` (1480 bytes, 370 leaves, covers both DQ4+DW7 control codes)
- **Python reference:** `translation-tools/hbe/huffman/{dq4.py, dw7.py, node.py, bitutils.py, schema.py}`
- **C++ build:** `cybergrime/psx_binary_ops.cpp` — `HbdReencoder::process_hbd()` at line 421
- **C++ header:** `cybergrime/psx_binary_ops.h` — `HbdReencoder` class at line 179

## Implementation Plan

### Step 1: Add HuffTree C++ class
Add to `psx_binary_ops.cpp` / `.h`:
- `HuffTree::parse_dq4(tree_bytes)` — parse DQ4 per-block tree (root+1 convention)
- `HuffTree::parse_dw7(tree_bytes)` — parse DW7/hybrid tree (no +1 convention)
- `HuffTree::decode(text_bytes)` → `vector<uint16_t>` leaf values
- `HuffTree::encode(vector<uint16_t> leaves)` → `vector<uint8_t>` text bytes

### Step 2: Integrate into `process_hbd()`
In the block conversion section (line ~539), when `has_tree == true`:
1. Parse the block's per-block tree bytes (`tree_bytes` from `ParsedBlock`)
2. Decode `text_data` using the per-block tree → leaf values
3. Parse the hybrid tree (already loaded as `hybrid_tree`)
4. Encode leaf values using hybrid tree → new text bytes
5. Use new text bytes in the converted block

### Step 3: Strip non-essential EXE patches from `build_v39()`
Remove from `build_v39()`:
- `patch_lba_references()` — no LBA shift needed
- `patch_sequential_sector_table()` — no shift
- `patch_disc_check_variable()` — unnecessary
- `patch_tree_references()` — tree reloc handled by payload append
- `append_reloc_payload_with_thread_fix()` — keep only if BSS clear is still an issue

Keep only:
- `remap_folder_pointers()` — ABS/REL table patches for matched folders
- Tree reloc payload (append hybrid tree to EXE + redirect MIPS refs)

### Step 4: Build and test
- Compile `psx_binary_ops.cpp` → `psx_ops.dll`
- Run `build_v39()` with minimal patching
- Test in DuckStation for Enix logo

## Why Previous Builds Failed

| Build | What happened | Root cause |
|-------|--------------|------------|
| v22 | DW7 EXE + raw DQ4 HBD | Text not re-encoded → DW7 EXE reads garbage |
| v23 | Hybrid HBD (DW7+DQ4) | DQ4 text still not re-encoded → matched folders still garbage |
| v34 | Boot succeeded, no logo | Tree reloc worked but text still DQ4-encoded |
| v39 | Same as v23 + more patches | Same text re-encode gap + EXE corrupted by excessive patches |

## Critical Insight

The hybrid tree (`dw7_hybrid_tree.bin`) was built to contain all leaf values from both DQ4 and DW7. It was used to **encode English translations** in the Python pipeline. But the C++ `process_hbd()` never uses it to re-encode the DQ4 HBD text blocks. The tree is loaded but only passed to `append_reloc_payload()` for EXE patching — it's never used for its primary purpose: re-encoding text.

The fix is straightforward: decode with per-block tree, re-encode with hybrid tree. Same algorithm as `session_f_hybrid_tree.py` but applied to every block in the HBD, not just translated entries.
