# Session V42 Build — Jul 20, 2026

## Root Cause
The DQ4 HBD source (`dq4_heart_v17.bin`) contains 319MB of allocated data, but only **141,197 sectors (289MB)** are non-zero. The builder was writing all 319MB, overwriting valid DW7 FMV data at LBA 146621 with zeros. The game hardcodes a seek to LBA 146621 for FMV/CD-ROM initialization, finds zeros, and stalls in a timeout loop at PC 0x800986F0.

## Three Fixes Applied

### 1. HBD Truncation Fix (Primary)
- Added `detect_hbd_data_size()` function: binary-searches for last non-zero sector
- Builder now only writes 141,197 sectors of DQ4 HBD data (sectors 355-141551)
- DW7 data preserved from sector 141552 onwards, including FMV at LBA 146621
- Verified: v42 LBA 146621 matches DW7 original exactly (2004 non-zero bytes)

### 2. Pre-Tree BSS Zero (Secondary)
- Region 0x800BC668-0x800BC700 (152 bytes) was originally cleared by BSS
- v41 moved BSS start to 0x800BCCC8 (past tree), skipping this region
- Non-zero garbage at 0x800BC6CC causes thread status polling stall at PC 0x80048004
- Fix: explicitly zero the 152-byte region at build time
- Verified: 0 non-zero bytes on disc

### 3. CD-ROM Stall Bypass (Tertiary)
- Timeout loop at 0x800986E0 polls event flag up to 3.9M iterations
- `bne $v1, $zero, +12` (0x1460000C) → `beq $zero, $zero, +12` (0x1000000C)
- Always skips the wait loop as if the event already fired
- Verified: 0x1000000C on disc

## C++ Integration (psx_binary_ops.cpp/h)
- `ExePatcher::zero_pre_tree_bss()` — zeros 152-byte pre-tree BSS region
- `ExePatcher::patch_cdrom_stall_bypass()` — patches bne→beq at 0x800986E0
- `ExePatcher::patch_fmv_skip()` — searches for Setmode 0xA0 pattern (none found in EXE)
- C API wrappers: `exe_zero_pre_tree_bss()`, `exe_patch_cdrom_stall_bypass()`, `exe_patch_fmv_skip()`
- New constants: `PRE_TREE_BSS_START/END/LEN`, `CDROM_POLL_BNE_OFF`, `CDROM_POLL_TIMEOUT_OFF`

## FMV Skip
- No `addiu $reg, $zero, 0xA0` (Setmode double-speed) found in EXE
- FMV skip is achieved indirectly by preserving DW7 FMV data at LBA 146621
- The game can now read valid FMV data instead of zeros

## Build Output
- File: `dq4_frankenstein_v42.bin` (711,567,024 bytes)
- CUE: `dq4_frankenstein_v42.cue`
- Patches: disc=11, lba=1, seq=44, abs_matched=168, abs_delta=1755, rel=3, tree=24
- EXE: 677,888 bytes, PC0=0x8008E284 (unchanged)
- Tree: 0x800BC700, BSS: 0x800BCCC8-0x800F4980

## Testing
- Ready for DuckStation test with `-nogui -fastboot`
- Expected: game boots past CD-ROM init, reaches Enix logo or title screen
