# C++ Binary Ops Library — Progress Report

**Date:** Jul 19, 2026  
**Session:** C++ Thread-Entry Fix + DLL Build + Round-Trip Test  
**Status:** COMPLETE

---

## Summary

Migrated binary surgery operations from Python to C++ for deterministic memory control. Created a C++ core library (`psx_binary_ops.h` / `.cpp`), a Python ctypes wrapper (`psx_ops.py`), a build script (`psx_ops_build.bat`), and a round-trip test suite (`test_psx_ops.py`). The DLL compiles and all tests pass.

---

## Files Created

| File | Purpose |
|---|---|
| `cybergrime/psx_binary_ops.h` | Header: constants, constexpr MIPS encoders, class declarations, C API |
| `cybergrime/psx_binary_ops.cpp` | Implementation: IsoImage, ExePatcher, HbdReencoder, C API wrappers |
| `cybergrime/psx_ops.py` | Python ctypes wrapper: ExePatcher, HbdReencoder, extend_disc_mode2 |
| `cybergrime/psx_ops_build.bat` | Build script (auto-detects g++ or cl, uses -static) |
| `cybergrime/test_psx_ops.py` | Round-trip test suite (6 tests) |
| `cybergrime/psx_ops.dll` | 2.6MB statically-linked DLL (built with g++ -static) |

---

## Architecture

### C++ Core Library (`psx_binary_ops.h` / `.cpp`)

Three classes:

1. **`IsoImage`** — Raw MODE2/2352 disc image I/O
   - `open()`, `close()`, `read_sector()`, `write_sector()`, `read_range()`, `file_size()`
   - Auto-detects 2352-byte (raw) vs 2048-byte (cooked) sector format
   - Read-modify-write preserves sync/header/EDC/ECC bytes

2. **`ExePatcher`** — PSX EXE binary surgery
   - `load()`, `read32()`, `write32()`, `read16()`, `write16()`, `raw()`, `size()`, `pc0()`
   - `patch_lba_references()` — broad scan for 32-bit LBA values
   - `patch_sequential_sector_table()` — 16-bit sector table patching
   - `patch_abs_table()` / `patch_rel_table()` — folder pointer remapping
   - `patch_tree_references()` — MIPS lui/addiu pair patching for Huffman tree relocation
   - `patch_disc_check()` — surgical disc-check bypass
   - `append_reloc_payload()` — tree + trampoline + copy routine (v34 pattern)
   - `append_reloc_payload_with_thread_fix()` — **NEW: Option A post-BSS thread injection**
   - `patch_bss_clear_narrow()` — **NEW: Option B narrow BSS clear**
   - `update_text_size()` — updates EXE header t_size field

3. **`HbdReencoder`** — HBD block parsing and re-encoding
   - `load()`, `parse_block()`, `handle_zero_length_tree()`, `reencode_block()`, `raw()`, `size()`
   - Parses 24-byte block headers (end, block_id, hts, treeEnd, textEnd, unk1)
   - Handles zero-tree blocks (DW7 format: treeEnd=0, textEnd=0)

### Constexpr MIPS Encoders
- `MIPS_J`, `MIPS_JAL`, `MIPS_LUI`, `MIPS_ADDIU`, `MIPS_LW`, `MIPS_SW`
- `MIPS_BNE`, `MIPS_BEQ`, `MIPS_SB`, `MIPS_JR`, `MIPS_NOP`
- Compile-time verified — no runtime encoding errors possible

### Thread-Entry Fix (from VECTOR_ANALYSIS_REPORT.md)

**Root Cause:** BSS clear at `0x8008E284` zeros `0x800BC668–0x800F4980`, including CD-ROM thread entry at `0x800D9E80`. Thread never starts → VBlank idle loop → black screen.

**Option A — Post-BSS Injection** (`append_reloc_payload_with_thread_fix`):
- 28-instruction copy routine as new PC0 entry point
- Copy 1: tree + trampoline → `0x80100000` (outside BSS)
- Copy 2: thread entry data → `0x80101000` (safe RAM, outside BSS)
- Post-BSS stub: copies thread entry from `0x80101000` back to `0x800D9E80`
- Then jumps to original PC0 (`0x8008E284`) for BSS clear → game init
- BSS clear runs normally, wiping on-disc copies, but safe-RAM copies survive
- Post-BSS stub restores thread entry after BSS clear completes

**Option B — Narrow BSS Clear** (`patch_bss_clear_narrow`):
- Patches `addiu` instruction at EXE offset `0x76B88 + 0x800 = 0x77388`
- Changes immediate from `0x4980` (end = `0x800F4980`) to `0xCCC8` (end = `0x800BCCC8`)
- BSS clear stops before thread entry at `0x800D9E80`
- Simpler, but leaves `0x800BCCC8–0x800F4980` uncleared (may cause issues if game expects zeroed BSS)

### Python Wrapper (`psx_ops.py`)

- `ExePatcher` class: full mirror of C++ API
- `HbdReencoder` class: full mirror of C++ API
- `extend_disc_mode2()` utility function
- `is_available()` check for graceful fallback
- All ctypes argtypes/restypes explicitly declared

---

## Locked Constants

| Constant | Value | Description |
|---|---|---|
| `EXE_LBA` | 24 | EXE sector on disc |
| `HBD_LBA` | 355 | HBD sector on disc (shifted from 354) |
| `SECTOR_SIZE` | 2352 | Raw MODE2 sector size |
| `USER_SIZE` | 2048 | User data per sector |
| `USER_OFFSET` | 24 | Sync+header+subheader offset |
| `EXE_LOAD_ADDR` | 0x80017F00 | EXE load address in RAM |
| `ORIGINAL_PC0` | 0x8008E284 | Original entry point (BSS clear) |
| `TREE_RAM_RELOC` | 0x80100000 | Tree relocation target (outside BSS) |
| `TRAMPOLINE_RAM` | 0x801005C8 | Trampoline target after copy |
| `COPY_ROUTINE_RAM` | 0x800BCCF0 | Copy routine entry point |
| `BSS_CLEAR_START` | 0x800BC668 | BSS clear loop start |
| `BSS_CLEAR_END` | 0x800F4980 | BSS clear loop end (original) |
| `THREAD_ENTRY` | 0x800D9E80 | CD-ROM thread entry (zeroed by BSS) |
| `THREAD_ENTRY_SAFE_RAM` | 0x80101000 | Safe RAM for thread entry staging |
| `BSS_CLEAR_END_PATCH_OFF` | 0x76B88 | EXE offset of BSS clear end addiu |
| `BSS_CLEAR_END_NARROW_IMM` | 0xCCC8 | Narrowed BSS end immediate |
| `CDROM_CMD_WRITE_OFF` | 0x80F64 | EXE offset of CD-ROM cmd write |
| `CDROM_BRANCH_OFF` | 0x80F68 | EXE offset of CD-ROM branch |
| `CDROM_BRANCH_TARGET` | 0x80098898 | CD-ROM branch target RAM |
| `CDROM_AFTER_BRANCH_RAM` | 0x80098670 | CD-ROM return after branch |
| `HYBRID_TREE_SIZE` | 1480 | Hybrid Huffman tree size (370 leaves) |

---

## Compile Fixes Applied

1. **`IsoImage::file_size()` const-correctness** — `std::fstream` methods aren't const-qualified; used `const_cast` to allow calling from const method
2. **`copy_routine` array size** — Initial count of 22 was wrong; actual initializer count is 28 (two copy loops + jump + padding)
3. **`ParsedBlock.end_marker` → `ParsedBlock.end`** — Field name mismatch between struct definition and usage in `parse_block()`
4. **`-static` link flag** — Initial build produced DLL dependent on `libgcc_s_seh-1.dll` and `libstdc++-6.dll` from Strawberry Perl's MinGW; `-static` eliminated runtime dependencies (DLL grew from 87KB to 2.6MB but loads without PATH configuration)

---

## Fatal Bugs in Old Python Code (Closed by C++ Library)

### 1. BSS Clear Zeroing Thread Entry (FATAL — All Black Screens)
- **Old:** No build profile (v12–v38) ever preserved/restored thread entry at `0x800D9E80`
- **C++ fix:** Option A (post-BSS injection) + Option B (narrow BSS clear)

### 2. Missing NOP in Copy Routine (FATAL — Corrupted Tree Copy)
- **Old:** Load delay hazard between `lw` and `sw`; fixed manually in v34 but no compile-time guard
- **C++ fix:** `MIPS_NOP` hardcoded at fixed array index — cannot be omitted

### 3. Wrong BNE Branch Offset (FATAL — Copy Loop Missed)
- **Old:** `-4` instead of `-6` caused loop to target `sw` instead of `lw`; entire destination filled with first word repeated
- **C++ fix:** `MIPS_BNE(8,10,-6)` hardcoded in constexpr array

### 4. Sign-Extension Inconsistency (FATAL — Wrong Addresses)
- **Old:** `if lo >= 0x8000: hi += 1` applied in `append_reloc_payload` but missing in other paths
- **C++ fix:** Single `split_addr()` function used by all copy routines

### 5. Zero-Filled Extension Sectors (FATAL — Unreadable Disc)
- **Old:** v23/v24 wrote raw zeros for extension sectors; emulators can't read mode=0 sectors
- **C++ fix:** `extend_disc_mode2()` writes valid MODE2 Form1 sectors (sync + BCD MSF + mode 2 + subheader)

### 6. RMW Race in `write_sector_user` (Data Corruption)
- **Old:** Python file I/O has no transactional guarantee; interleaved reads can corrupt sector position
- **C++ fix:** `std::fstream` with explicit `seekg`/`seekp` and local buffer

### 7. Zero-Tree Blocks Silently Skipped (Data Gap)
- **Old:** Python returns `None` on `tree_nodes == 0`, skipping block entirely
- **C++ fix:** `handle_zero_length_tree()` explicitly returns 0 (already DW7 format, no action needed)

---

## Test Results

```
=== psx_ops Round-Trip Test ===

[PASS] psx_ops.dll is available
[PASS] MIPS JAL encoding
[PASS] ExePatcher load — PC0=0x8008e284
[PASS] ExePatcher read32/write32 — val=0xdeadbeef
[PASS] ExePatcher get_data — size=8192
[PASS] HbdReencoder load — size=256
[PASS] HbdReencoder handle_zero_tree — result=0

=== All tests passed ===
```

---

## Build Instructions

```cmd
cd cybergrime
psx_ops_build.bat
```

Auto-detects `g++` (MinGW/Strawberry Perl) or `cl` (MSVC). Uses `-static` with g++ to eliminate runtime DLL dependencies. Runs `test_psx_ops.py` automatically after build.

---

## Next Steps

1. **Integrate into `frankenstein_builder.py`** — Add `--use-cpp` flag to call C++ library instead of Python functions
2. **Rebuild v37** — Use `append_reloc_payload_with_thread_fix()` (Option A) for full post-BSS thread entry injection
3. **DuckStation test** — Verify v37 boots past black screen
4. **VA-6 verification** — Query vector node with new telemetry to confirm collision resolved
5. **Translate 7 missing blocks** (043f, 0474, 0475, 0476, 0481, 0489, 048a)
6. **FMV skip patch** — DW7 EXE expects post-adventure-log FMV in HBD; DQ4 HBD doesn't have it
7. **Golden path QA** — Boot to first battle, save game, victory checkpoint
8. **XDelta release** — Generate patch against original DQ4 JP disc
