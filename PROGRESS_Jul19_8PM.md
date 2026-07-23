# Progress Notes — Jul 19, 2026 8:00 PM

## Current Task: Black Screen Debug — Tree Placement Experiment

### Root Cause Analysis
- LBA 146621 read is **normal DW7 boot behavior**, not a bug
- v38a (disc-check only, no tree) boots past this read → reaches disc check screen
- v40 (tree@0x800BC700 + BSS narrow) hangs after this read → **tree placement is the problem**
- Tree at `0x800BC700` sits inside BSS clear range (`0x800BC668`–`0x800F4980`)
- BSS narrow patch (start→`0x800BCCC8`) may break game init by leaving BSS variables unzeroed

### Three-Option Experiment

| Option | Profile | Tree Location | BSS Patch | PC0 | Status |
|--------|---------|---------------|-----------|-----|--------|
| C | `frank_v40c` | None (game uses own at `0x800EF1C8`) | None | `0x8008E284` (orig) | **Built** |
| B | `frank_v40b` | `0x800F4980` via copy routine | None | `0x800BC700` (copy) | **Building** |
| A | `frank_v40a` | `0x80100000` via copy routine | None | `0x800BC700` (copy) | **Pending** |

### Code Changes Made

**`cybergrime/psx_binary_ops.cpp`**:
- `build_v39()` now accepts `int tree_mode` parameter (0=C, 1=B, 2=A, 3=current)
- tree_mode=0: No tree appended, no tree ref patches, no BSS patch, original EXE size
- tree_mode=1: Copy routine (14 instr) at `0x800BC700` as PC0, tree at `0x800BC738`, copies to `0x800F4980`, then jumps to `0x8008E284`
- tree_mode=2: Same copy routine, copies tree to `0x80100000` (high RAM, safe from BSS)
- tree_mode=3: Current behavior (tree@`0x800BC700`, BSS narrowed to `0x800BCCC8`)
- Added C API wrappers: `build_frankenstein_v40c`, `build_frankenstein_v40b`, `build_frankenstein_v40a`

**`cybergrime/psx_binary_ops.h`**:
- Updated `build_v39()` declaration with `int tree_mode = 3` default

**`translation-tools/frankenstein_builder.py`**:
- Added `_build_v40_variant()` shared ctypes launcher
- Added `build_frank_v40c()`, `build_frank_v40b()`, `build_frank_v40a()`
- Registered profiles: `frank_v40c`, `frank_v40b`, `frank_v40a`

### DLL
- `cybergrime/psx_ops.dll` rebuilt with g++ `-static` (2.68MB, 8:04 PM)

### v40c Build Output
- EXE: 675,840 bytes (original, no tree), PC0=`0x8008E284`
- Patches: tree=0, lba=1, seq=44, abs=1923(168 matched, 1755 delta), rel=167, disc=11
- Disc: 1,076,550,384 bytes (extended for DW7+DQ4 HBD)
- EDC/ECC: FAILED (expected for extended disc — edcre can't handle >original size)
- File: `dq4_frankenstein_v40c.bin` / `.cue`

### DuckStation CLI
Correct usage (from GitHub wiki):
```
duckstation-qt-x64-ReleaseLTCG.exe -fastboot dq4_frankenstein_v40c.cue
```
Parameters use `-` prefix (not `--`). No `--noshadercache` or other invalid flags.

### Key Memory Addresses
- EXE load: `0x80017F00`, original end: `0x800BC700` (file_size `0xA4800`)
- BSS clear range (code-level at `0x8008E284`): `0x800BC668`–`0x800F4980`
- Original DW7 tree dest: `0x800EF1C8` (outside loaded EXE, game copies at runtime)
- High RAM safe zone: `0x80100000` (~1MB above data section, ~1MB below stack `0x801FFFF0`)
- Stack: `0x801FFFF0`

### Pending
- v40b build completing (background)
- v40a build
- Test all three in DuckStation with correct CLI args
- v34 previously booted successfully with tree at `0x80100000` + copy routine + trampoline
