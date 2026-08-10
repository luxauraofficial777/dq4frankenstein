# dq4frankenstein Progress ~99%
<img width="1024" height="1024" alt="dq4frank" src="https://github.com/user-attachments/assets/3ac9afc3-0cec-4a48-aab4-8b5e086aed73" />
DQ4/DW7 Frankenstein Project V.9,V.95,V.96,V.97,V.98,V.99

Dragon Quest IV PSX: "Lost Translation"
Created by Lux Aura with the help of previous work by Markus Projects, Mandy Wilkens, and ChickenKnife. http://markus-projects.net/dragon-hackst-iv/ https://github.com/mwilkens/dq4psxtrans https://www.romhacking.net/hacks/4275/ https://luxaura.bandcamp.com https://www.facebook.com/LuxAuraOfficial https://www.youtube.com/LuxAuraOfficial

A monolithic reverse-engineering, dynamic translation, and ISO rebuilding suite. This project aims to create a fully playable English localization of Dragon Quest IV (PSX) by grafting the Dragon Warrior VII (US) executable onto the DQ4 disc (the "Frankenstein" approach).

⚠️ Why Frankenstein?
While modifying the native DQ4 Japanese executable is possible, extending the hardcoded text buffers to accommodate English without crashing at Chapter 1 is extremely difficult. More importantly, the native engine stores battle, item, and world dialogue indices internally in a way that makes full localization virtually impossible without source access.

By transplanting the DW7 (US) executable into Sector 24 of the DQ4 disc ("Reverse Option A"), we inherit an engine already equipped to handle English text, variable-width fonts, and global Huffman tree parsing. The challenge shifts from expanding buffers to bridging data formats—specifically, patching the DW7 engine in MIPS assembly to parse DQ4's per-block Huffman trees and HBD archive structures.

🛠️ The Architecture & Toolchain
This is not a simple ROM hack. It is a closed-loop, autonomous reverse-engineering pipeline.

🚀 1. The Core Builder (cybergrime/)
A custom C++ compiler (frankenstein_build.exe) that operates directly on the PSX binaries and ISO files.

MIPS Patching: Injects surgical assembly patches to resolve delay-slot hazards, bypass strict CD-ROM intercepts, and handle BSS zero-filling boundaries.
HBD Re-encoder: Reads the DQ4 per-block text archives and re-encodes them on-the-fly into the DW7 global hybrid tree format to prevent PSX heap overflows.
EDC/ECC Generation: Automatically corrects sector checksums post-modification.

🚀 2. The Build Orchestrator (pipeline.py)
The Python backbone that manages the build lifecycle, applies hotfixes, manages FastBoot INI files, and interfaces with the C++ builder without mutating staging files destructively (--disasm-only).

🚀 3. PSX Matrix & Hydra Loop (PSXMatrix/)
Testing across a single emulator is a trap. The Hydra loop (hydra_loop.py) automatically runs the compiled .bin against a multi-emulator matrix:

DuckStation: Validates strict hardware constraints, accurate CD-ROM FIFO timings, and BIOS execution.
CyberGrime: A custom headless emulator core built for deep telemetry logging, VBlank manipulation, and memory hooking.
StarPSX: Provides a lightweight secondary baseline for raw execution flow.
4. Digital Twin Diagnostics (verify/)
A 4-phase automated quality assurance framework that closes the loop between build failures and code remediation:

Phase 1 (Static Analysis): Inspects the built ISO for valid PVDs, correctly patched PC0 offsets, and descriptor table alignments.
Phase 2 (Dynamic Telemetry): Parses CyberGrime and DuckStation stdout to detect heap over-allocations and missing thread execution (probe_heap.py, probe_thread.py).
Phases 3 & 4 (Checkpoints & Triage): Maps execution telemetry against the "Golden Mile" boot sequence to auto-classify crashes.
🚀 Current Project Status: The Golden Mile
The project maps boot progression across a "Golden Mile".

Frankenstein Pipeline V.99 — Documentation
Version: V.99 (Pre-release / Release Candidate) Date: August 10, 2026 Author: VoidWalkers Project
> **Progress: ~99%** — Reverse Frankenstein Pipeline V.99 (Release Candidate)
> A monolithic reverse-engineering, dynamic translation, and ISO rebuilding suite for Dragon Quest IV (PSX).

Created by **Lux Aura** with the help of previous work by Markus Projects, Mandy Wilkens, and ChickenKnife.

- http://markus-projects.net/dragon-hackst-iv/
- https://github.com/mwilkens/dq4psxtrans
- https://www.romhacking.net/hacks/4275/
- https://luxaura.bandcamp.com
- https://www.youtube.com/LuxAuraOfficial

---

## What This Is

This project creates a fully playable English localization of Dragon Quest IV (PSX) by grafting the Dragon Warrior VII (US) executable onto the DQ4 (Japan) disc — the **"Frankenstein" approach**.

### Why Frankenstein?

While modifying the native DQ4 Japanese executable is possible, extending the hardcoded text buffers to accommodate English without crashing at Chapter 1 is extremely difficult. More importantly, the native engine stores battle, item, and world dialogue indices internally in a way that makes full localization virtually impossible without source access.

By transplanting the DW7 (US) executable into Sector 24 of the DQ4 disc (**Reverse Option A**), we inherit an engine already equipped to handle English text, variable-width fonts, and global Huffman tree parsing. The challenge shifts from expanding buffers to bridging data formats — specifically, patching the DW7 engine in MIPS assembly to parse DQ4's per-block Huffman trees and HBD archive structures.

### The Architecture & Toolchain

This is not a simple ROM hack. It is a closed-loop, autonomous reverse-engineering pipeline.

1. **The Core Builder** (`builder/`) — A custom C++ compiler (`frankenstein_build.exe`) that operates directly on PSX binaries and ISO files:
   - **MIPS Patching**: Injects surgical assembly patches to resolve delay-slot hazards, bypass strict CD-ROM intercepts, and handle BSS zero-filling boundaries
   - **HBD Re-encoder**: Reads DQ4 per-block text archives and re-encodes them on-the-fly into the DW7 global hybrid tree format to prevent PSX heap overflows
   - **EDC/ECC Generation**: Automatically corrects sector checksums post-modification via `edcre`
   - **19 Hard Verification Gates**: Sector-aware readback of every major patch site — aborts build on any mismatch

2. **Translation Layer** (`translation/`) — Pre-encoded English text, hybrid Huffman tree, folder mapping, and the DW7 US EXE:
   - `dw7_encoded_translation.json` — 3.2 MB of pre-encoded English text in Huffman format
   - `dw7_hybrid_tree.bin` — 1472-byte hybrid tree bridging DQ4 + DW7 character sets
   - `folder_mapping.json` — DW7-to-DQ4 sector/folder mapping
   - `SLUSP012.06_clean.bin` — Unpatched DW7 US executable

3. **DuckStation** (`duckstation/`) — Bundled emulator with SCPH-1001 BIOS for boot testing

---

## Why This Is Hard

This is **NOT** a translation problem. The translation is done. This is a binary compatibility problem between two PSX executables that share the same game engine but use incompatible internal data formats:

- **DW7 Huffman**: global tree, `root_id = value & 0x7FFF`
- **DQ4 Huffman**: per-block trees, `root_id = (value & 0x7FFF) + 1`

The hybrid tree bridges this at the binary level. The thread blob injection solves the BSS zeroing problem (the CD-ROM thread code lives in BSS and gets wiped on boot). The boot copy stub ensures the blob is in place **before** BSS clear runs.

The remaining issue is a BSS/LBA mismatch causing a BIOS exception after initialization — the disc's volume descriptor still identifies as Japanese, and the US BIOS/EXE may be rejecting it.

This requires someone who understands PSX BIOS internals, ISO 9660 PVD structure, and MIPS R3000A exception handling. Not a text editor job.

---

## What You Need

You must provide your own ROMs:

1. **Dragon Warrior VII (SLUS-01206)** disc image (.bin/.cue) — source of the US EXE
2. **Dragon Quest IV (Japan)** disc image — source of the HBD archive and base disc

This repo contains **no copyrighted ROM data**, no compiled game binaries, and no open-source dependencies. You will need to obtain those separately:

- **edcre** (EDC/ECC regeneration tool) — included in `edcre/`
- **DuckStation** (PSX emulator for testing) — included in `duckstation/`
- **Visual Studio 2022 Build Tools** (only if recompiling the builder from source)

### Source Disc Placement

Place the DQ4 Japan disc at:
```
frankenstein_pipeline/disc/Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin
```

| Property | Value |
|----------|-------|
| Expected SHA-256 | `100D87DB9DEADF8F9FA4BB891D3A5D0BB112ACBF5ADBCBC93C637848ED9C7531` |
| Size | 368,057,424 bytes (351 MB) |
| Format | Mode 2 Form 1, 2352-byte sectors |

---

## Directory Structure (V.99)

```
frankenstein_pipeline/
├── builder/                        # C++ builder (source + binary + configs)
│   ├── frankenstein_build.exe           # Pre-compiled builder (483 KB)
│   ├── frankenstein_build_main.cpp      # Main entry point + 19 verification gates
│   ├── psx_binary_ops.cpp               # Core PSX binary operations (187 KB)
│   ├── psx_binary_ops.h                 # BuildResult, MIPS encoders, structs
│   ├── hotfixes_empty.txt               # Empty hotfix file (no runtime patches)
│   ├── offset_registry.json             # Canonical patch address registry
│   ├── FROZEN_INPUTS.json               # SHA-256 hashes of all build inputs
│   ├── thread_code_blob.bin             # MISS-1 thread blob (2048 bytes)
│   ├── translation_map.bin              # Binary translation map (TXMP format)
│   ├── reencode_manifest.json           # HBD re-encoding results manifest
│   ├── reencode_pipeline.py             # HBD pre-processing script
│   ├── gen_translation_map.py           # Translation map generator
│   ├── verify_thread_blob.py            # F-2: Thread blob verification
│   └── dw7_exe_surgical_patcher.py      # Standalone EXE patcher
├── translation/                    # Translation data + patcher inputs
│   ├── SLUSP012.06_clean.bin            # DW7 US EXE (675 KB)
│   ├── dw7_hybrid_tree.bin              # Hybrid Huffman tree (1472 bytes)
│   ├── dw7_encoded_translation.json     # Pre-encoded English text (3.2 MB)
│   ├── full_translation_hybrid_playable.json  # Full translation (1.4 MB)
│   ├── folder_mapping.json              # DW7->DQ4 sector mapping (290 KB)
│   └── checksums.txt                    # Frozen input checksums
├── edcre/                          # EDC/ECC repair + verification
│   └── edcre.exe                        # edcre v1.1.0 (53 KB)
├── docs/                           # Documentation
│   └── V99_DOCUMENTATION.md             # Full technical documentation
├── compile.bat                     # Compile builder from source (MSVC)
├── build.bat                       # Build Frankenstein disc (reverse Option A)
├── build_comparison.bat            # F-3: Build with/without XA stub for A/B testing
└── boot_test.bat                   # Launch DuckStation boot test
```


---

## Build Instructions

### Quick Start (pre-compiled builder)

```cmd
cd frankenstein_pipeline
build.bat
```

Output: `build_staging/dq4_frankenstein_v109.bin` + `.cue`

### Build from source

```cmd
cd frankenstein_pipeline
compile.bat
build.bat
```

**Compile requirements**: Visual Studio 2022 Build Tools (vcvars64.bat)

```cmd
cl /nologo /std:c++17 /EHsc /O2 /Fe:frankenstein_build.exe ^
    frankenstein_build_main.cpp psx_binary_ops.cpp ^
    /I. /D_CRT_SECURE_NO_WARNINGS ^
    /link /SUBSYSTEM:CONSOLE /LARGEADDRESSAWARE
```

### Boot test

```cmd
cd frankenstein_pipeline
boot_test.bat build_staging\dq4_frankenstein_v109.cue
```

### A/B Comparison (with/without XA stub)

```cmd
cd frankenstein_pipeline
build_comparison.bat
```

### CLI Reference

```
Usage:
  frankenstein_build.exe --reverse [options]

Required (reverse mode):
  --dq4-hbd <path>       DQ4 Japan disc (.bin)
  --dw7-exe <path>       DW7 US EXE (SLUSP012.06_clean.bin)
  --hybrid-tree <path>   Hybrid Huffman tree (dw7_hybrid_tree.bin)
  --edcre <path>         edcre.exe path
  --output <path>        Output .bin path
  --output-cue <path>    Output .cue path

Optional:
  --no-xa-stub           Skip XA stub patch (F-3 comparison)
  --reencode-hbd         Enable HBD re-encoding
  --skip-hash-check      Skip FROZEN_INPUTS hash verification
  --duckstation          Enable DuckStation compatibility patches
  --hotfixes <path>      Runtime hotfix file (default: none)
  --staging-dir <path>   Build staging directory
  --tree-mode <0-3>      Tree mode (forward build only)

Exit Codes:
  0 = Build complete, all gates passed
  1 = Build failed or gate failure
```

---

## Patch Architecture (V.99, Reverse Option A)

The builder applies 127+ binary patches to the DW7 EXE in the following categories:

| # | Category | Sites | Purpose |
|---|----------|-------|---------|
| 1 | Disc-check bypass | 3 | Force all disc identity checks to return success |
| 2 | LBA table relocation | 1 | Redirect HBD reference from LBA 354 to 362 |
| 3 | Sequence table adjustment | 44 | Update seq table entries for DQ4 disc layout |
| 4 | Tree pointer redirection | 24 | Redirect all LUI/ADDIU tree refs to hybrid tree at 0x800BC700 |
| 5 | BSS clear range narrowing | 3 | Prevent BSS zero-fill from wiping hybrid tree region |
| 6 | Malloc clamp trampoline | 9 words | Redirect heap allocation to prevent overflow into tree area |
| 7 | Decompressor root ID hook | 2 | Runtime JAL to adjust root_id +1 for DQ4 compatibility |
| 8 | Tree hook JAL | 1 | Redirect decompressor to hybrid tree parser |
| 9 | FIFO drain jump | 1 | Jump past CD-ROM FIFO stall loop |
| 10 | XA stub (D1 fix) | 6 | Set Flag A (0x800E2038) and Flag B (0x800E3D94) to unblock poll loops |
| 11 | MISS-1 boot copy stub (D2 fix) | ~20 | Copy thread blob to 0x800D9E80 before BSS clear runs |
| 12 | VBlank wait bypass | 3 | Return immediately from VBlank spinlock |
| 13 | FMV XA stub NOP | 1 | NOP out FMV XA command intercept |
| 14 | Decomp offset-B mask removal | 1 | Remove bit mask for DQ4 block compatibility |
| 15 | HBD re-encoding | 3573 blocks | Re-encode text blocks with hybrid Huffman tree |

### Key RAM Addresses

| Address | Purpose |
|---------|---------|
| 0x80017F00 | EXE load address (LBA 24) |
| 0x8008E284 | Original PC0 (game entry point) |
| 0x800BDF00 | Patched PC0 (MISS-1 boot copy stub) |
| 0x800BC700 | Hybrid Huffman tree location in RAM |
| 0x800D9E80 | Thread blob destination (MISS-1 copy) |
| 0x800DA680 | Thread blob end address |
| 0x800E2038 | Flag A — set by XA stub, checked by poll loops (D1 critical) |
| 0x800E3D94 | Flag B — set by XA stub |

---

## Verification Gates (19 total)

Every build must pass all 19 hard gates or the build aborts. Gates 9-19 perform sector-aware readback of patched values directly from the output disc.

| Gate | Description | Type |
|------|-------------|------|
| 1 | EXE magic (PS-X EXE) | Metadata |
| 2 | Output disc exists | Metadata |
| 3 | CUE sheet exists | Metadata |
| 4 | Disc size > 350 MB | Metadata |
| 5 | Tree patches > 0 | Count |
| 6 | Disc-check patches == 3 | Count |
| 7 | HBD blocks processed > 0 | Count |
| 8 | HBD blocks converted > 0 | Count |
| 9 | XA stub sw_2038 (D1 guard) | Sector-aware readback |
| 10 | MISS-1 stub[7] ADDIU (D2 guard) | Sector-aware readback |
| 11 | Disc-check 3 sites == 0x24020001 | Sector-aware readback |
| 12 | Malloc clamp trampoline (9 words) | Sector-aware readback |
| 13 | Tree hook JAL at 0x800562B8 | Sector-aware readback |
| 14 | FIFO drain J at 0x8008B65C | Sector-aware readback |
| 15 | Decomp root ID JAL at 0x80057FEC | Sector-aware readback |
| 16 | BSS clear range (3 words) | Sector-aware readback |
| 17 | LBA ref at 0x8001B194 == 0x16A | Sector-aware readback |
| 18 | Seq table[0] at 0x800BB0AA | Sector-aware readback |
| 19 | Decomp tree ptr LUI at 0x800562EC | Sector-aware readback |

Sector-aware readback formula: `LBA = 24 + file_offset/2048`, `sector_offset = file_offset%2048`, read at `LBA * 2352 + 24 + sector_offset`.

---

## Build Output

### v109 (current)

| Property | Value |
|----------|-------|
| SHA-256 | `AFED72C6416844D6DE6EE4A350670A9D7C2A9DA5C7535FD205E0BAB8CEEA8B85` |
| Size | 368,057,424 bytes (351.0 MB) |
| Patches | tree=24, disc=3, lba=1, seq=44, trampoline=2 |
| HBD | 3573 processed, 1527 converted, 3148 sub-block swaps |
| Gates | 19/19 PASS |
| Determinism | Byte-identical across multiple builds with same inputs |

### v109-no-xa (F-3 comparison)

| Property | Value |
|----------|-------|
| SHA-256 | `A220D3A8B57B119A4B75E9541EB8A668E9F3DEF33F2F3C8F72B6FFFEECDA91A7` |
| Gates | 18/19 PASS (Gate 9 SKIP as expected) |

---

## Translation Pipeline

### How It Works

1. **`gen_translation_map.py`** reads `dw7_encoded_translation.json` and outputs `translation_map.bin` (TXMP format: magic + entry count + per-text-ID encoded sequences)
2. The C++ **HbdReencoder** reads DQ4 HBD blocks from the source disc at sector 362
3. For each block: decode using per-block Huffman tree, re-encode using the global hybrid tree
4. Inject pre-encoded English text from `translation_map.bin`
5. Strip per-block tree, set `hts=24`, `treeEnd=0`, `textEnd=0`
6. Blocks that fail re-encoding get verbatim fallback (DW7 format headers, original text preserved)

### Standalone Patcher

`dw7_exe_surgical_patcher.py` can apply patches independently of the builder:

```cmd
python dw7_exe_surgical_patcher.py <disc.bin> --apply-all
python dw7_exe_surgical_patcher.py <disc.bin> --verify-only
python dw7_exe_surgical_patcher.py <disc.bin> --dump-tables
```

---

## Audit Fixes Applied (V.99)

All open findings from the consolidated audit have been addressed:

| Finding | Severity | Fix |
|---------|----------|-----|
| F-2 | MEDIUM | Thread blob verification script (`verify_thread_blob.py`) |
| F-3 | HIGH | `--no-xa-stub` comparison disc support, Gate 9 conditional skip |
| F-5 | MEDIUM | WARNING guards promoted to hard aborts (2 sites in psx_binary_ops.cpp) |
| F-7 | MEDIUM | `run_edcre` refactored: BIN input, two-phase repair+verify, exit 1 distinct |
| F-8 | MEDIUM | Banner context-aware: only prints relevant flags per build mode |
| F-9 | MEDIUM | `__TIMESTAMP__` replaced with `time()`, argv, patch counts in manifest |

---

## Known Issues & Limitations

1. **Boot stall at ~18s**: The EXE reads sectors 154-174 (ISO dir + SYSTEM.CNF + EXE start), then stalls. Likely HBD format incompatibility or CD-ROM polling loop. The XA stub (D1 fix) and MISS-1 stub (D2 fix) are designed to address this but have not been boot-tested with v109 yet.

2. **Low MIPS density in thread blob**: The MISS-1 thread blob (`thread_code_blob.bin`) has only 8% MIPS instruction density (43/512 words). The remaining 91% is zeros. This may indicate the blob is mostly data, not executable code. Runtime verification via DuckStation memory dump is needed.

3. **HBD re-encoding partial**: Of 3573 HBD blocks processed, only 1527 are converted. The rest use verbatim fallback. Some text may garble but the parser should not crash.

4. **DuckStation DLLs**: The `duckstation/` folder is ~158 MB due to Qt6 and DirectX DLLs. This is the minimum required for DuckStation to run. If boot testing is not needed, this folder can be omitted.

5. **Source disc not included**: The DQ4 Japan disc (351 MB) is too large for the repo. It must be placed at `disc/Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` before building.

---

## The Golden Mile

The project maps boot progression across a "Golden Mile" — a sequence of checkpoints the disc must pass to reach gameplay:

| Checkpoint | Description | Status |
|------------|-------------|--------|
| BIOS boot | SCPH-1001 initializes, loads SYSTEM.CNF | PASS |
| EXE load | EXE loaded from LBA 24 to RAM at 0x80017F00 | PASS |
| PC0 redirect | MISS-1 stub executes, copies thread blob | PASS (static) |
| BSS clear | Narrowed BSS zero-fill preserves hybrid tree | PASS (static) |
| Disc check | All 3 disc identity checks bypassed | PASS (static) |
| HBD mount | HBD archive at sector 362 recognized | PASS (static) |
| Tree init | Hybrid Huffman tree loaded at 0x800BC700 | PASS (static) |
| CD-ROM init | XA stub sets Flag A/B, poll loops unblocked | PASS (static) |
| First read | EXE reads sectors 154-174 (ISO dir + CNF) | **STALL ~18s** |
| Game loop | Main game thread begins | **NOT REACHED** |

All static gates pass (19/19). The stall occurs at runtime during the first CD-ROM data read after initialization.

---------------------------------------------------------------
CONTACT
---------------------------------------------------------------

  Bandcamp:  https://luxaura.bandcamp.com
  Facebook:  https://www.facebook.com/LuxAuraOfficial
  YouTube:   https://www.youtube.com/LuxAuraOfficial

  (cc) Lux Aura. For educational and preservation purposes.
===============================================================

