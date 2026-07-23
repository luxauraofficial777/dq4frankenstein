# Frankenstein Build Pipeline — Order of Operations & Toolchain Analysis

## Executive Summary

The DQ4 Frankenstein build takes DW7's EXE (engine) + DQ4's translated HBD (game data),
patches them together into a hybrid disc, and runs it through EDC/ECC. The pipeline has
grown organically across **30+ build profiles** (v12 through v44), two parallel builders
(Python + C++), and multiple legacy scripts. This document maps every step, every tool,
every input, and identifies redundancies, dead code, and cleanroom pathways.

---

## Current Pipeline — Full Order of Operations

### Phase 0: Source Inputs (Pre-build)

| Input | File | Description |
|-------|------|-------------|
| DW7 US Disc | `DW7D1/DW7D1.bin` | Base disc image (MODE2/2352) |
| DW7 US EXE | `translation/SLUSP012.06` | Clean DW7 executable (675,840 bytes) |
| DQ4 JP Disc | `Dragon Quest IV -...bin` | Original Japanese DQ4 disc |
| DQ4 Translated HBD | `dq4_heart_v17.bin` | DQ4 JP disc with English text patched into HBD |
| Translation Data | `translation/full_translation_frankenstein.json` | English text entries |
| Hybrid Huffman Tree | `dw7_hybrid_tree.bin` | 1480-byte merged DQ4+DW7 tree |
| Folder Mapping | `translation/folder_mapping.json` | DW7↔DQ4 sector correspondence |
| EDC/ECC Tool | `edcre/edcre-v1.1.0-.../edcre.exe` | Sector error correction regeneration |
| C++ Library | `cybergrime/psx_ops.dll` | HBD re-encoder + EXE patcher (called via ctypes) |

### Phase 1: Translation Heart Production (Offline, Pre-computed)

**Status: Already produced. `dq4_heart_v17.bin` exists on disk.**

```
Dragon Quest IV JP (Japan).bin
        │
        ▼
dq4_hbd_patcher.py
  (--input DQ4_JP.bin --translations full_translation_frankenstein.json --output dq4_heart_v17.bin)
        │
        ▼
dq4_heart_v17.bin  (DQ4 JP disc with English text injected into HBD at LBA 362)
```

**Tool**: `dq4_hbd_patcher.py` (in translation-tools root)
**Input**: DQ4 JP disc + translation JSON
**Output**: `dq4_heart_v17.bin` (368MB, 141,197 non-zero sectors)
**What it does**: Patches English text into DQ4's HBD archive using Huffman re-encoding.
The DQ4 HBD uses per-block Huffman trees with root_id = (value & 0x7FFF) + 1.

### Phase 2: Folder Mapping Generation (Offline, Pre-computed)

**Status: Already produced. `folder_mapping.json` exists on disk.**

```
DQ4 HBD structure ──┐
                     ├──► build_folder_mapping.py ──► folder_mapping.json
DW7 HBD structure ──┘
```

**Tool**: `translation-tools/build_folder_mapping.py`
**Input**: DQ4 + DW7 HBD folder/file listings (JSON catalogs)
**Output**: `translation/folder_mapping.json` (168 matched folders by block ID)
**What it does**: Matches DW7 folders to DQ4 folders by comparing (type, block_id) signatures.
Produces sector mappings: `dw7_sector → dq4_sector` for ABS/REL pointer remapping.

### Phase 3: Hybrid Huffman Tree Generation (Offline, Pre-computed)

**Status: Already produced. `dw7_hybrid_tree.bin` exists on disk.**

**Tool**: Manual construction (scripts in `_archive_scripts/analyze_scripts/`)
**Output**: `dw7_hybrid_tree.bin` (1480 bytes, signature 0x02800480)
**What it does**: Merges DQ4 and DW7 Huffman tree formats into a single tree that
the DW7 decompressor can use. Bridges root_id calculation difference
(DQ4: +1, DW7: no +1) and offset_b alignment difference.

### Phase 4: Frankenstein Build (Main Pipeline)

**Entry point**: `python frankenstein_builder.py --profile frank_v44`
**File**: `translation-tools/frankenstein_builder.py` (4342 lines, 30+ profiles)

#### Step [1/10] — Disc-Check Bypass (Surgical)

```
DW7 EXE (SLUSP012.06)
    │
    ├── patch_disc_check() → 10 surgical patches
    │     • exit1/2/3: Force return 1 (success) at 3 failure exit points
    │     • nop_timeout: Skip timeout check
    │     • nop_error: Skip error path
    │     • neutralize: Disc-change handler returns immediately
    │     • redirect: 3 infinite-loop traps redirected to success paths
    │     • graft_a/b/c: 3 conditional branches → unconditional (always pass)
    │
    ▼
Patched EXE (disc-check bypassed, CD-ROM hardware still initializes normally)
```

**Key v44 change**: Removed `cd_init_force_success` (which replaced the entire CD-ROM init
function with a stub). Now uses surgical patches that only patch failure exits, letting
CD-ROM hardware initialize normally.

#### Step [2/10] — LBA Reference Patching

```
Patched EXE
    │
    ├── patch_lba_references() → scan for 354, replace with 355
    │     • Scans EXE for all occurrences of HBD_LBA (354)
    │     • Replaces with NEW_HBD_LBA (355) to account for EXE growth
    │
    ▼
EXE with LBA refs updated (354 → 355)
```

**Why**: The hybrid tree is appended to the EXE, growing it by ~2KB. This pushes the HBD
start from sector 354 to 355. All LBA references in the EXE must be updated.

#### Step [3/10] — Sequential Sector Table Patching

```
EXE with LBA refs
    │
    ├── patch_sequential_sector_table() → update sector table at EXE offset 0xA39AA
    │     • Table contains sequential sector values 350-370
    │     • Each value shifted by LBA_DELTA (+1)
    │
    ▼
EXE with sector table updated
```

#### Step [4/10] — ABS/REL Pointer Remapping

```
EXE + folder_mapping.json
    │
    ├── remap_pointers_matched_only()
    │     • ABS table at EXE offset 0x4184 (6000 entries × 8 bytes)
    │     • REL table at EXE offset 0x511C (6000 entries × 8 bytes)
    │     • Matched folders: direct sector replacement (168 matched)
    │     • Unmatched: LBA delta shift (+1)
    │
    ▼
EXE with all folder pointers remapped to DQ4 HBD sectors
```

**Result** (v44): `abs_matched=168, abs_delta=1755, rel=3, tree=24`

#### Step [5/10] — Hybrid Tree Append

```
EXE + dw7_hybrid_tree.bin
    │
    ├── append_hybrid_tree()
    │     • Pad EXE to 4-byte alignment
    │     • Append 1480-byte hybrid tree at end of EXE
    │     • Pad to 2048-byte sector boundary
    │     • Update EXE header text_size field
    │
    ▼
Extended EXE (677,888 bytes, tree at RAM 0x800BC700)
```

#### Step [6/10] — MIPS Tree Reference Patching

```
Extended EXE
    │
    ├── compute_tree_addr() → calculate LUI/ADDIU for 0x800BC700
    │     LUI = 0x800C, ADDIU = 0xB700
    │
    ├── patch_mips_tree_refs() → find all LUI 0x800F + ADDIU 0xF1C8 pairs
    │     • Scans for original DW7 tree reference pattern
    │     • Replaces with new LUI/ADDIU pointing to hybrid tree
    │     • 23 reference pairs found and patched
    │
    ▼
EXE with all Huffman tree refs pointing to hybrid tree
```

#### Step [7/10] — BSS Clear Patching

```
EXE with tree refs
    │
    ├── patch_bss_clear_start()
    │     • Original BSS clear starts at 0x800BC668
    │     • Patch start to 0x800BC700 (after hybrid tree)
    │     • BSS clear end: 0x800F4980 (unchanged)
    │
    ├── Zero pre-tree BSS region
    │     • 152 bytes at file offset 0xA4F68 (RAM 0x800BC668-0x800BC700)
    │     • Manually zeroed to prevent stale data
    │
    ▼
EXE with BSS clear adjusted to skip hybrid tree
```

#### Step [8/10] — HBD Re-encoding (THE CRITICAL STEP)

```
dq4_heart_v17.bin + dw7_hybrid_tree.bin
    │
    ├── detect_hbd_data_size() → binary search for last non-zero sector
    │     Result: 141,197 sectors (289,171,456 bytes)
    │
    ├── extract_hbd_from_disc() → read HBD from sector 362
    │
    ├── psx_ops.dll → HbdReencoder::process_hbd()
    │     │
    │     ├── A) Block header conversion
    │     │     • DQ4: hts=24, per-block trees, treeEnd/textEnd set
    │     │     • DW7: hts=1366, no per-block trees, treeEnd=0, textEnd=0
    │     │     • Converts all block headers to DW7 format
    │     │
    │     ├── B) Sub-block field swap
    │     │     • DQ4: flags=count, type_id=type
    │     │     • DW7: flags=0, type_id=count
    │     │     • Swaps fields for types 32, 39, 40, 42, 46
    │     │     • 1590 subblock swaps performed
    │     │
    │     ├── C) Huffman re-encoding
    │     │     • Decode text with DQ4 per-block tree
    │     │     • Re-encode with hybrid global tree
    │     │     • 391 blocks re-encoded, 162 FAILED ← BLOCKER
    │     │
    │     ├── D) LBA relocation
    │     │     • Internal sector refs adjusted by delta (362→355 = -7)
    │     │
    │     └── E) Validation
    │           • Checks for residual DQ4 types (32, 39, 40, 42, 46)
    │           • 7 validation failures found ← CONFIRMS BLOCKER
    │
    ▼
Re-encoded HBD (287,848,448 bytes, 162 blocks still in DQ4 format)
```

**THIS IS THE BLOCKER**: 162 blocks fail re-encoding. Types 32 and 46 have structures
the re-encoder doesn't fully handle. These unconverted blocks cause the DW7 engine to
populate BSS with garbage pointers, crashing at PC 0x800761DC.

#### Step [9/10] — EXE Size Verification

```
Extended EXE size: 677,888 bytes
Max before HBD overlap: (355 - 24) × 2048 = 677,888 bytes
Result: OK (exactly at limit)
```

#### Step [10/10] — Disc Assembly

```
DW7D1.bin (base disc)
    │
    ├── shutil.copy() → copy DW7 disc as base
    │
    ├── write_raw_data(EXE) → write patched EXE at sector 24
    │     • Sector-by-sector write (2048 bytes per sector)
    │
    ├── write_raw_data(HBD) → write re-encoded DQ4 HBD at sector 355
    │     • Only non-zero sectors written (preserves DW7 FMV at LBA 146621+)
    │
    ├── update_iso_dir() → update ISO 9660 directory
    │     • HBD1PS1D.W71;1 → LBA=355, size=319,436,800
    │     • SLUSP012.06;1 → size=677,888
    │
    ├── update_pvd() → update Primary Volume Descriptor
    │
    ├── run_edcre() → regenerate EDC/ECC for all modified sectors
    │     • edcre.exe processes entire disc image
    │
    ├── write_cue() → generate CUE sheet
    │     • TRACK 01 MODE2/2352, INDEX 01 00:00:00
    │
    ▼
dq4_frankenstein_v44.bin + dq4_frankenstein_v44.cue
```

---

## Parallel / Legacy Pipelines

### Legacy Pipeline A: build_heart.py (DW7 HBD In-Place Patching)

```
full_translation.json → HB_Packer → HB_Linker → HB_Validator → EXE patch → ISO → EDC/ECC
```

**Status**: Superseded. This pipeline patches English text *into DW7's HBD* rather than
swapping DQ4's HBD. It cannot handle DQ4-specific block types.

**File**: `translation-tools/build_heart.py` (346 lines)

### Legacy Pipeline B: frankenstein_build.cpp (C++ Builder)

```
Sources → Parse folders → Patch EXE → Re-encode HBD → Assemble disc → EDC/ECC → Verify
```

**Status**: Parallel to Python builder. Produces identical output (verified 0 byte diff).
Still includes FMV skip + CD-ROM stall bypass (removed in v44 Python).

**File**: `frankenstein_build.cpp` (900 lines)

### Legacy Pipeline C: Java Heart Transplant

```
DQ4 JP disc → TextPatcher.java → HeartTransplantV2.java → dq4_translated_final.bin
```

**Status**: Superseded. This was the original "heart transplant" approach — DQ4 EXE +
patched HBD. Confirmed working but has pointer mapping issues.

**Files**: `translation-tools/src/TextPatcher.java`, `HeartTransplantV2.java`

### Modern Pipeline D: HBE (Heart Beat Engine) Package

```
hbe/archive.py → hbe/parser/ → hbe/compression/ → hbe/huffman/ → hbe/fuser.py → hbe/writer.py → hbe/validator.py
```

**Status**: Partially built. Cleanroom HBD manipulation library with proper archive parsing,
Huffman tree cracking, and ISO9660 writing. Not yet integrated into the main build.

**Directory**: `translation-tools/hbe/` (20+ modules)

### Decompiled EXEs

**Status**: Available. Both DW7 and DQ4 executables have been fully decompiled.

**Directory**: `decompile/` (pseudo_decompiler.py, c_synthesizer.py, symbol_mapper.py, etc.)
**Output**: `decompile/c_synthesis_output/` (3288 items), `decompile/decompile_output/` (8 items)

---

## Tool Inventory

### Active Tools (Used in v44 Build)

| Tool | File | Purpose |
|------|------|---------|
| Frankenstein Builder | `translation-tools/frankenstein_builder.py` | Main build orchestrator (4342 lines) |
| PSX Ops DLL | `cybergrime/psx_ops.dll` | C++ HBD re-encoder + EXE patcher |
| PSX Ops Source | `cybergrime/psx_binary_ops.cpp` | Source for psx_ops.dll (1079+ lines) |
| PSX Ops Wrapper | `cybergrime/psx_ops.py` | Python ctypes bindings |
| Folder Mapper | `translation-tools/build_folder_mapping.py` | DW7↔DQ4 folder sector mapping |
| Heart Builder | `translation-tools/build_heart.py` | Legacy orchestrator (superseded) |
| EDC/ECC | `edcre/edcre-v1.1.0-.../edcre.exe` | Sector error correction |
| Build Verifier | `verify_build.py` | Sector-by-sector patch verification |
| Audit Macro | `audit_macro.py` | Full disc audit |

### Diagnostic Tools (Not in Build Path)

| Tool | File | Purpose |
|------|------|---------|
| ISO Validator | `translation-tools/validate_iso.py` | ISO 9660 directory validator |
| EXE Validator | `translation-tools/validate_exe.py` | EXE header validator |
| LBA Scanner | `translation-tools/scan_hardcoded_lba.py` | DW7 EXE sector value scanner |
| Diagnostic Builder | `translation-tools/build_diagnostic_v2.py` | 6 diagnostic build profiles |
| Hybrid Tree Test | `_archive_scripts/analyze_scripts/test_hybrid_tree.py` | Tree validation |

### Dead/Superseded Code

| Tool | File | Status |
|------|------|--------|
| C++ Builder | `frankenstein_build.cpp` | Superseded by Python (identical output) |
| Frankenstein Clean | `frankenstein_clean.py` | Early version, superseded |
| Heart V17 Builder | `_archive_scripts/cybergrime_legacy/build_heart_v17.py` | One-shot script, already run |
| Java TextPatcher | `translation-tools/src/TextPatcher.java` | Legacy text patching |
| Java HeartTransplant | `translation-tools/src/HeartTransplant*.java` | Legacy heart transplant |
| Java BatchTextPatcher | `translation-tools/src/BatchTextPatcher.java` | Legacy batch patching |
| 30+ Build Profiles | `frankenstein_builder.py` profiles v12-v43 | Superseded by v44 |

---

## Redundancy Analysis

### Redundant Systems

1. **Two builders producing identical output**
   - `frankenstein_builder.py` (Python, 4342 lines) — active
   - `frankenstein_build.cpp` (C++, 900 lines) — produces 0-byte-diff identical output
   - **Recommendation**: Retire C++ builder. Python builder is the canonical path.
   - Keep `psx_ops.dll` (C++ library called by Python via ctypes) — that's different.

2. **30+ build profiles, only 1 active**
   - v12 through v44 defined in `frankenstein_builder.py` PROFILES dict
   - Only `frank_v44` is current. `rc2` is a separate approach (DQ4 EXE).
   - **Recommendation**: Archive v12-v43 profiles. Keep v44, rc2, and diagnostic profiles (test_0/a/b/d/f).

3. **Three disc-check bypass strategies**
   - Full bypass (cd_init_force_success + stall bypass) — v43 and earlier
   - Semi-surgical (8 patches) — v18-v30
   - Surgical (10 patches, no cd_init bypass) — v44 (current)
   - **Recommendation**: Keep only surgical (v44). Others are superseded.

4. **Multiple HBD patching approaches**
   - `dq4_hbd_patcher.py` — patches DQ4 HBD in-place (produces dq4_heart_v17.bin)
   - `build_heart.py` + HB_Packer/Linker — patches DW7 HBD in-place (legacy)
   - `hbe/` package — cleanroom HBD manipulation (not yet integrated)
   - **Recommendation**: Consolidate into HBE package long-term. Keep `dq4_hbd_patcher.py` as current.

5. **Duplicate hybrid tree files**
   - `dw7_hybrid_tree.bin` (root)
   - `frozen/dw7_hybrid_tree.bin` (frozen copy)
   - **Recommendation**: Keep root copy. Verify frozen matches.

### Missing Systems

1. **No automated E2E verification**
   - Build → verify patches → run in emulator → check for crash
   - Currently manual: build, then manually load in DuckStation, visually inspect
   - **Recommendation**: Add automated DuckStation smoke test step

2. **No HBD re-encoder unit tests**
   - The 162 failed blocks (types 32/46) have no test coverage
   - **Recommendation**: Add test fixtures for each block type

3. **No translation completeness check**
   - No automated verification that all Japanese text has English replacements
   - **Recommendation**: Add translation coverage report

4. **No checksum/signature verification of inputs**
   - Source files (DW7 disc, DQ4 disc, EXE) not checksummed
   - **Recommendation**: Add SHA256 verification of all inputs

---

## Cleanroom Pathway Analysis

Now that both DW7 and DQ4 executables are fully decompiled, we have new options.

### Pathway 1: Fix Current Pipeline (Minimal Change)

**Effort**: Low | **Risk**: Low | **Time**: Hours

```
Current Pipeline → Fix HbdReencoder for types 32/46 → Rebuild v44 → Test
```

- Fix `HbdReencoder::process_hbd()` in `psx_binary_ops.cpp` to handle types 32 and 46
- Rebuild `psx_ops.dll`
- Rebuild `dq4_frankenstein_v44.bin`
- Run in DuckStation

**Pros**: Minimal change, fast iteration, all other steps verified working
**Cons**: Still a hybrid approach (DW7 engine parsing DQ4 data through conversion)

### Pathway 2: DQ4 EXE + DW7 Assets (Cleanest)

**Effort**: Medium | **Risk**: Medium | **Time**: Days

```
DQ4 EXE (decompiled) → Patch disc-check for DW7 disc ID → Use DQ4 HBD natively → Add DW7 FMV/audio
```

- Use DQ4's own EXE (which natively understands all DQ4 block types)
- Patch disc-check to accept DW7 disc identity
- Use DQ4 HBD as-is (no re-encoding needed!)
- Map DW7 FMV/audio sectors into DQ4's file table
- The `rc2` profile in `frankenstein_builder.py` already attempts this

**Pros**: No HBD re-encoding needed at all. Engine natively understands all block types.
**Cons**: DQ4 EXE has different CD-ROM handling, may need disc-ID patches, font handling
differences, no DW7 system menu integration.

### Pathway 3: Full Cleanroom via HBE Package

**Effort**: High | **Risk**: Low | **Time**: Weeks

```
HBE Package → Parse DQ4 HBD → Extract all text → Re-encode with DW7 tree → Write HBD → Assemble disc
```

- Use `hbe/` package to parse DQ4 HBD from scratch
- Extract all text blocks with proper type handling (32, 46, etc.)
- Re-encode using DW7-compatible Huffman tree
- Write back to disc using `hbe/iso9660.py`
- Full programmatic control over every block type

**Pros**: Complete control, no black-box C++ re-encoder, testable at each step
**Cons**: Significant engineering effort, HBE package not yet complete

### Pathway 4: Decompilation-Informed Micro-Patches

**Effort**: Medium | **Risk**: Medium | **Time**: Days

```
Decompiled DW7 EXE → Identify block type 32/46 parse logic → Patch to handle DQ4 format natively
```

- Use decompiled DW7 EXE to find the block type switch/dispatch
- Add handling for DQ4 types 32/46 directly in the DW7 EXE
- No re-encoder changes needed — engine handles both formats

**Pros**: Runtime solution, no build pipeline changes, handles all edge cases
**Cons**: Requires precise MIPS patching, adds complexity to EXE, potential for new bugs

### Recommended Path: 1 + 4 (Hybrid)

1. **Immediate**: Fix re-encoder for types 32/46 (Pathway 1) — get a working build
2. **Short-term**: Use decompiled EXE to understand types 32/46 (Pathway 4) — add runtime fallback
3. **Long-term**: Migrate to HBE package (Pathway 3) — cleanroom foundation

---

## Proposed Clean Pipeline (Target State)

```
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1: Translation                                       │
│                                                             │
│  full_translation.json                                      │
│        │                                                    │
│        ▼                                                    │
│  dq4_hbd_patcher.py → dq4_heart_v17.bin                    │
│        │                                                    │
│        ▼                                                    │
│  verify_heart.py (NEW: checksum + block structure check)   │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2: Asset Preparation                                 │
│                                                             │
│  build_folder_mapping.py → folder_mapping.json             │
│  dw7_hybrid_tree.bin (pre-computed)                         │
│  verify_inputs.py (NEW: SHA256 + structure validation)     │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3: EXE Patching                                      │
│                                                             │
│  SLUSP012.06 (DW7 EXE)                                     │
│        │                                                    │
│        ├── [3a] Disc-check bypass (10 surgical patches)     │
│        ├── [3b] LBA references (354→355)                    │
│        ├── [3c] Sequential sector table                     │
│        ├── [3d] ABS/REL pointer remapping                   │
│        ├── [3e] Hybrid tree append                          │
│        ├── [3f] MIPS tree ref patching (23 pairs)           │
│        ├── [3g] BSS clear patching                          │
│        │                                                    │
│        ▼                                                    │
│  verify_exe.py (EXE header + patch verification)           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4: HBD Re-encoding                                   │
│                                                             │
│  dq4_heart_v17.bin + dw7_hybrid_tree.bin                   │
│        │                                                    │
│        ├── [4a] Extract DQ4 HBD (141,197 sectors)           │
│        ├── [4b] Block header conversion (DQ4→DW7)           │
│        ├── [4c] Sub-block field swap (types 32/39/40/42/46) │
│        ├── [4d] Huffman re-encode (per-block → global)      │
│        │     ★ FIX: Handle types 32/46 (162 failures)       │
│        ├── [4e] LBA relocation (delta -7)                   │
│        ├── [4f] Validation (no residual DQ4 types)          │
│        │                                                    │
│        ▼                                                    │
│  verify_hbd.py (NEW: block type + structure validation)    │
│  ★ GATE: Validation must PASS before proceeding             │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 5: Disc Assembly                                     │
│                                                             │
│  DW7D1.bin (base disc)                                     │
│        │                                                    │
│        ├── [5a] Copy DW7 disc as base                       │
│        ├── [5b] Write patched EXE at sector 24              │
│        ├── [5c] Write re-encoded HBD at sector 355          │
│        ├── [5d] Update ISO 9660 directory                   │
│        ├── [5e] Update PVD                                  │
│        ├── [5f] Regenerate EDC/ECC (edcre.exe)              │
│        ├── [5g] Write CUE sheet                             │
│        │                                                    │
│        ▼                                                    │
│  verify_disc.py (sector-by-sector patch verification)      │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 6: Emulator Smoke Test (NEW)                         │
│                                                             │
│  dq4_frankenstein_v44.bin + .cue                           │
│        │                                                    │
│        ├── [6a] Load in DuckStation (headless)              │
│        ├── [6b] Run for 60 seconds (~3600 VBlanks)          │
│        ├── [6c] Check for crash (UnknownReadHandler spam)   │
│        ├── [6d] Check CD-ROM reads at expected LBAs         │
│        ├── [6e] Report: PASS/FAIL + crash PC if failed      │
│        │                                                    │
│        ▼                                                    │
│  BUILD RESULT: PASS or FAIL with diagnosis                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Differences: Current vs Proposed

| Aspect | Current | Proposed |
|--------|---------|----------|
| Build profiles | 30+ (v12-v44) | 1 active + 5 diagnostic |
| Builders | Python + C++ (parallel) | Python only |
| HBD re-encoder | C++ black box (psx_ops.dll) | C++ with unit tests + type 32/46 fix |
| Validation | Post-build manual | Per-phase automated gates |
| Emulator test | Manual DuckStation | Automated headless smoke test |
| Input verification | None | SHA256 + structure checks |
| Dead code | ~15,000 lines across legacy scripts | Archived |

---

## Block Type Reference

| Type | Game | Format | Re-encoder Status |
|------|------|--------|-------------------|
| 19 | DW7 | Standard text | N/A (native) |
| 23 | DQ4 | Text + per-block tree | Re-encoded ✓ |
| 24 | DQ4 | Text + per-block tree | Re-encoded ✓ |
| 25 | DQ4 | Text + per-block tree | Re-encoded ✓ |
| 27 | DQ4 | Text + per-block tree | Re-encoded ✓ |
| 32 | DQ4 | Unknown (likely compressed) | **FAILS** (162 blocks) |
| 39 | DQ4 | Sub-block (field swap) | Field swap ✓, re-encode varies |
| 40 | DW7 | Standard text | N/A (native) |
| 42 | DW7 | Standard text | N/A (native) |
| 46 | DQ4 | Unknown (likely compressed) | **FAILS** (included in 162) |

**Types 32 and 46 are the blocker.** The re-encoder's Huffman decode/re-encode path
fails for these types. They may have:
- Different tree structures (not standard DQ4 per-block trees)
- Different compression schemes (not Huffman at all)
- Different header layouts (tree_end/text_end fields interpreted differently)

---

## Session Meta

- Date: Jul 22, 2026
- Active profile: frank_v44
- Pipeline status: **Blocked at Phase 4, Step 4d** (HBD re-encoder types 32/46)
- Build pipeline: Verified correct (patches persist, EDC/ECC valid, C++/Python identical)
- Decompilation: Both DW7 and DQ4 EXEs fully decompiled
- Next action: Fix `HbdReencoder::process_hbd()` for types 32/46 in `psx_binary_ops.cpp`
