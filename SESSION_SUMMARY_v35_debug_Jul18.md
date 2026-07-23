# Session Summary: v35 Boot Debug — Jul 18, 2026 8:35 AM

## Objective
Fix the v33 boot crash at `PC=0x801005C8` by relocating the hybrid Huffman tree from unmapped `0x80100000` to an in-place append at the end of the EXE text segment (`0x800BC700`).

## What Was Done

### 1. In-Place Tree Approach (v35)
- Replaced `append_reloc_payload()` (copy routine → `0x80100000`) with `append_tree_inplace()` — appends tree + trampoline directly to EXE text segment
- Tree at RAM `0x800BC700` (file offset `0xA5000`), trampoline at `0x800BCCC8`
- Patched BSS clear start from `0x800BC668` → `0x800BCCF0` via `patch_bss_clear_start()`
- No copy routine needed — PC0 unchanged at `0x8008E284`
- 24 MIPS tree references patched to `0x800BC700`
- EXE size: 677,888 bytes (exactly max before HBD overlap)

### 2. BSS Clear Patch Fix
- Removed second LUI/ADDIU patch at `0x76BA8/0x76BAC` — that pair loads `0x800BC668` for `$gp` setup after BSS clear, not for BSS clear itself. Patching it would break all global variable access.

### 3. Build Completed Successfully
- File: `dq4_frankenstein_v35.bin` (711,567,024 bytes)
- EDC/ECC: SUCCESS
- Re-encoding: **SKIPPED** (round-trip verification failed — blocks with tree length=0)
- Patches: name=2, lba=1, seq=44, abs=1923, rel=167, disc=10, tree=24, headers=1, audio=4

### 4. DuckStation Boot Test
- Launched `dq4_frankenstein_v35.cue` in DuckStation
- **Result: Black screen, no Enix logo, no boot**
- FPS: 0.00 (no GPU output), VPS: ~59.82 (CPU running)
- Game hangs after CD-ROM data loading

## Root Cause Analysis

### v35 Crash: Game BSS Variables Overwrite Tree
The DuckStation log reveals the exact failure sequence:

```
[11.7080] D/CodeCache: Ignoring fault due to RAM write @ 0x800BC6CC
[11.7262] D/CodeCache: Ignoring fault due to RAM write @ 0x800BC7B8
```

The game **writes to `0x800BC6CC` and `0x800BC7B8`** — addresses within our tree region (`0x800BC700`–`0x800BCCC8`). These are game BSS variables that get initialized **after** BSS clear, during game startup. The tree is being overwritten by game variable writes.

Then:
```
[11.7442] E(ReadBlockInstructions): Instruction read failed at PC=0x800BDE60
[11.7603] E(CompileOrRevalidateBlock): Failed to read block at 0x800BDE60
```

The game tries to execute code at `0x800BDE60` — in the zeroed BSS region past our trampoline — crash.

**Conclusion**: The in-place approach fundamentally doesn't work because `0x800BC700` is inside the game's BSS variable region. The game writes to these addresses after BSS clear completes.

### v33 Crash Revisited: NOT Unmapped RAM
Previous hypothesis was that `0x80100000` was unmapped. DuckStation source code analysis disproves this:

- `RAM_2MB_SIZE = 0x200000` — DuckStation maps full 2MB RAM
- `0x80100000` = offset `0x100000` into RAM — well within 2MB range
- EXE loader (`bus.cpp:999-1008`) loads `min(file_size, actual_data)` bytes to `load_address` and sets PC
- No t_size-based RAM mapping restriction — RAM is always 2MB

The v33 crash at `0x801005C8` was **not** caused by unmapped RAM. The copy routine likely had a bug, or the BSS clear zeroed the copy routine before it could execute.

### v33 Copy Routine Bug Analysis
v33 layout:
- Copy routine at `0x800BCCF0` (PC0) — **inside BSS clear range** (`0x800BC668`–`0x800F4980`)
- BSS clear starts at `0x800BC668` and zeros up to `0x800F4980`
- The copy routine at `0x800BCCF0` is **within the BSS clear range**
- But PC0 is set to the copy routine, so it executes **before** BSS clear
- After copy routine jumps to original PC0 (`0x8008E284`), BSS clear runs and zeros `0x800BC668`–`0x800F4980`
- This would zero the copy routine's location but not `0x80100000` (outside BSS range)

**The v33 crash was likely a copy routine bug** — possibly the load delay slot NOP was missing or the branch offset was wrong, causing the tree/trampoline to not be copied correctly to `0x80100000`.

### BSS Clear Patch Offset Verification
Checking the actual bytes on disc at the BSS clear patch offsets revealed a **byte order issue** in the verification script. The LUI/ADDIU instructions read back as:
- LUI: `op=0x09 imm=0x0011` (should be `op=0x0F imm=0x800B`)
- ADDIU: `op=0x0F imm=0x800B` (should be `op=0x09 imm=0xC668`)

The opcodes are **swapped** — LUI and ADDIU are in reverse order. This means either:
1. The patch offsets `0x76B84`/`0x76B88` are wrong (ADDIU comes first, then LUI)
2. Or the verification script reads them in the wrong order

This needs further investigation — if the patch writes LUI to the ADDIU offset and vice versa, the BSS clear address would be garbage.

### CD-ROM Intercept Not Patched
On both v33 and v35 discs, the CD-ROM intercept at `0x80F64`/`0x80F68` shows **original unpatched instructions**:
- `0x80F64: 0x2406FFFF` (addiu $a2, $zero, -1) — original instruction, not `j trampoline`
- `0x80F68: 0x90A20000` (sw $v0, 0($a1)) — original instruction, not nop

This means the CD-ROM stop intercept is **not being written to disc at all**. The patches are applied to the in-memory EXE bytearray but may not survive to the disc write, or the offsets are wrong relative to the sector write.

## Key Findings Summary

| Issue | Status | Details |
|-------|--------|---------|
| In-place tree at `0x800BC700` | **FAILS** | Game BSS variables overwrite tree after BSS clear |
| `0x80100000` unmapped hypothesis | **DISPROVED** | DuckStation maps full 2MB RAM; `0x100000` is within range |
| v33 copy routine crash | **Likely bug** | Copy routine itself may have been correct but tree/trampoline not properly copied |
| BSS clear patch offsets | **Needs verification** | LUI/ADDIU may be swapped at `0x76B84`/`0x76B88` |
| CD-ROM intercept on disc | **NOT PATCHED** | Original instructions still at `0x80F64`/`0x80F68` on both v33 and v35 |
| Re-encoding | **SKIPPED** | Round-trip verification fails (blocks with tree length=0) |

## DuckStation Source Code Findings

- **RAM size**: Always 2MB (`RAM_2MB_SIZE = 0x200000`), no t_size-based restriction
- **EXE loader** (`bus.cpp:985-1023`): Loads `min(file_size, actual_data)` to `load_address`, sets PC/GP/SP
- **No memfill**: DQ4 EXE header has `memfill_start=0, memfill_size=0` — BSS clear is purely software
- **File**: `@/c:/LuxAura/VoidWalkers_Project/DQLOSTTRANSLATION/study/duckstation-src/src/core/bus.cpp:985-1023`

## Disc Layout
- **Format**: MODE2/2352 (2352 bytes per sector, 2048 user data at offset 24)
- **EXE**: LBA 24, PS-X EXE signature confirmed
- **HBD**: LBA 355, size 319,436,800 bytes
- **PVD**: Not at standard sector 16 (all zeros there); CD001 found at sector 18 offset 793
- **SYSTEM.CNF**: Found at sectors 25-26

## Next Steps

1. **Fix CD-ROM intercept not reaching disc** — The `write_raw_data` function writes sector-by-sector with 2048-byte user data. The CD-ROM intercept at EXE file offset `0x80F64` is in sector `24 + (0x80F64 - 0x800) / 2048 = 24 + 148 = 172`. Verify the write actually reaches this sector.

2. **Verify BSS clear patch offsets** — Disassemble the actual MIPS instructions at file offsets `0x76B84` and `0x76B88` on the **original** unpatched EXE to confirm which is LUI and which is ADDIU.

3. **Return to copy-routine approach with `0x80100000`** — Since DuckStation maps full 2MB RAM, `0x80100000` is valid. Fix the v33 copy routine bug and ensure:
   - Copy routine executes before BSS clear (PC0 → copy → original PC0)
   - Tree and trampoline are correctly copied to `0x80100000`
   - BSS clear does not zero the copy routine (it's at `0x800BCCF0`, within BSS range, but executes before BSS clear)
   - CD-ROM intercept actually reaches disc

4. **Alternative: Hook after BSS clear** — Instead of placing tree before BSS clear, hook into the post-BSS-clear path. Find where BSS clear ends and insert a copy routine there, so BSS clear runs first (zeroing everything including `0x800BC700`), then copy routine copies tree to `0x80100000`.

5. **Debug re-encoding** — `reencode_dq4_blocks.py` `verify_roundtrip()` finds blocks with tree length=0. Block map may have stale offsets.

## Files Modified This Session
- `translation-tools/frankenstein_builder.py` — Removed second LUI/ADDIU BSS clear patch
- `study/PROGRESS_v35_localization_graft_Jul18.md` — Updated with v35 build results
- `study/SESSION_SUMMARY_v35_debug_Jul18.md` — This file

## Files Created (Temporary, should clean up)
- `_check_files.py`, `_check_iso.py`, `_check_iso2.py`, `_check_iso3.py`, `_check_headers.py`
