# DW7 EXE Decompressor Patch Target Disassembly — Session Progress

Date: 2026-07-23

## Work completed
- Wrote C++ MIPS disassemblers:
  - `decompile/decompressor_disasm.cpp` — 516 bytes at RAM 0x80073670 / file 0x5B970
  - `decompile/find_patch_sites_vanilla.cpp` — full-EXE scan for patch candidates
  - `decompile/disasm_region.cpp` / `disasm_region2.cpp` / `disasm_region3.cpp` — focused region dumps
- Compiled and ran all tools with `g++`.
- Produced output files in `decompile/`:
  - `decompressor_disasm.txt`
  - `patch_sites_slusp.txt`
  - `patch_sites_dw7us.txt`
  - `region_73000.txt`
  - `region_6F000.txt`
  - `region_misc.txt`
  - `patch_targets.md`

## Key findings

### 1. Tree pointer LUI/ADDIU pairs
The runtime global Huffman tree lives at 0x800EF1C8. It is materialized by:

| RAM (lui)  | File off | RAM (addiu) | File off | Effective address |
|------------|----------|-------------|----------|-------------------|
| 0x8006F0FC | 0x571FC  | 0x8006F100  | 0x57200  | 0x800EF1C8        |
| 0x8006F144 | 0x57244  | 0x8006F148  | 0x57248  | 0x800EF1C8        |
| 0x80056AEC | 0x392EC  | 0x80056AF0  | 0x392F0  | 0x800EF1C8        |
| 0x80056B34 | 0x39334  | 0x80056B38  | 0x39338  | 0x800EF1C8        |

The first two pairs are inside the file/block loader (around 0x8006F000), which builds/walks the tree. The 516-byte block at 0x80073670 does **not** contain the tree-pointer load; it is a bit-manipulation helper/state-machine dispatcher.

### 2. root_id +1 patch target
Tree node index is extracted at 0x8006F1CC / file 0x572CC:
```
0x8006F1CC: andi $v0,$v1,0x7FFF
```
This instruction is in the branch delay slot of the preceding `beq`. Adding an explicit `addiu $v0,$v0,1` after it would require code insertion.

Equivalent single-instruction fix: the tree base is set at 0x8006F1A4 / file 0x572A4:
```
0x8006F1A4: addiu $t2,$v0,-3640   # $t2 = 0x800EF1C8
```
Because each node is 2 bytes, adding +1 to the index is equivalent to adding +2 to the base. Patch candidate:
```
0x244AF1C8 -> 0x244AF1CA   (addiu $t2,$v0,-3638)
```

### 3. offset_b mask removal target
The most likely `andi ... 0xFFFE` site for offset_b alignment is at:
```
0x80032EB8 / file 0x1AFB8: andi $v0,$v0,0xFFFE
```
It is followed by `ori $v0,$v0,0x0002` and `sh $v0,48($s2)`. Replacing the `andi` with `nop` removes the `& ~1` alignment, allowing odd offset_b values as required by DQ4.

Other `andi ... 0xFFFE` candidates:
- 0x800344F8 / 0x1C5F8
- 0x800949A4 / 0x2519A4

## Open issue

The tree-pointer patch is more than a simple LUI/ADDIU substitution because the per-block tree address varies. The decompressor must either:
1. Load the tree base from a runtime variable set by the block loader, or
2. Copy each block's inline tree to a fixed RAM buffer before decompression.

Option 2 aligns with the build-plan wording "load tree from block data instead of fixed 0x800BC700" and can reuse the existing `patch_tree_references` mechanism by retargeting all 0x800EF1C8 references to the fixed copy address, then injecting a small copy routine.

## Next steps
1. Decide on the tree-pointer strategy.
2. Implement root_id +1 patch (single instruction at 0x572A4 or code insertion at 0x572CC).
3. Implement offset_b mask removal (nop at 0x1AFB8).
4. Apply patches in `cybergrime/psx_binary_ops.cpp` and regenerate the build.
