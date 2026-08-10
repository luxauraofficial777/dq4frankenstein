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

1. The Core Builder (cybergrime/)
A custom C++ compiler (frankenstein_build.exe) that operates directly on the PSX binaries and ISO files.

MIPS Patching: Injects surgical assembly patches to resolve delay-slot hazards, bypass strict CD-ROM intercepts, and handle BSS zero-filling boundaries.
HBD Re-encoder: Reads the DQ4 per-block text archives and re-encodes them on-the-fly into the DW7 global hybrid tree format to prevent PSX heap overflows.
EDC/ECC Generation: Automatically corrects sector checksums post-modification.
2. The Build Orchestrator (pipeline.py)
The Python backbone that manages the build lifecycle, applies hotfixes, manages FastBoot INI files, and interfaces with the C++ builder without mutating staging files destructively (--disasm-only).

3. PSX Matrix & Hydra Loop (PSXMatrix/)
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

1. Overview
The Frankenstein Pipeline is a build system that creates a playable English-translated Dragon Quest IV (PSX) disc by grafting a Dragon Warrior VII (PSX) executable onto a DQ4 (Japan) disc base. The DW7 EXE provides English-compatible text rendering and game logic; the DQ4 disc provides the HBD (Huffman Block Data) archive which is re-encoded with English text using a hybrid Huffman tree.

Build Strategy: Reverse Frankenstein (Option A)
Take the DQ4 Japan disc as the base (351 MB, Mode 2 Form 1)
Swap in the DW7 US executable (SLUSP012.06_clean.bin)
Apply 127+ binary patches to the EXE:
Disc-check bypass (3 sites)
LBA table relocation (1 site)
Sequence table adjustment (44 sites)
Tree pointer redirection (24 sites)
BSS clear range narrowing (3 sites)
Malloc clamp trampoline (9 words)
Decomp root ID runtime JAL (2 sites)
Tree hook JAL (1 site)
FIFO drain jump (1 site)
XA stub (D1 fix — Flag A poll loop)
MISS-1 boot copy stub (D2 fix — load-delay slot)
VBlank bypass (3 sites)
FMV XA stub NOP (1 site)
Re-encode the HBD at sector 362 with English text using the hybrid Huffman tree
Run EDC/ECC repair and verification on the output disc
Verify all 19 hard gates pass
2. Folder Structure
frankenstein_pipeline/
├── builder/                    # C++ builder (source + binary)
│   ├── frankenstein_build.exe       # Pre-compiled builder (483 KB)
│   ├── frankenstein_build_main.cpp  # Main entry point
│   ├── psx_binary_ops.cpp           # Core PSX binary operations
│   ├── psx_binary_ops.h             # Header (BuildResult, MIPS encoders, structs)
│   ├── hotfixes_empty.txt           # Empty hotfix file (no runtime patches)
│   ├── offset_registry.json         # Canonical patch address registry (M3)
│   ├── FROZEN_INPUTS.json           # SHA-256 hashes of all build inputs (M1)
│   ├── thread_code_blob.bin         # MISS-1 thread blob (2048 bytes)
│   ├── translation_map.bin          # Binary translation map (TXMP format)
│   ├── reencode_manifest.json       # HBD re-encoding results manifest
│   ├── reencode_pipeline.py         # HBD pre-processing script
│   ├── gen_translation_map.py       # Translation map generator
│   ├── verify_thread_blob.py        # F-2: Thread blob verification
│   └── dw7_exe_surgical_patcher.py  # Standalone EXE patcher
├── translation/                # Translation data + patcher inputs
│   ├── SLUSP012.06_clean.bin        # DW7 US EXE (675 KB)
│   ├── dw7_hybrid_tree.bin          # Hybrid Huffman tree (1472 bytes)
│   ├── dw7_encoded_translation.json # Pre-encoded English text (3.2 MB)
│   ├── full_translation_hybrid_playable.json  # Full translation (1.4 MB)
│   ├── folder_mapping.json          # DW7→DQ4 sector mapping (290 KB)
│   └── checksums.txt                # Frozen input checksums
├── edcre/                      # EDC/ECC repair tool
│   └── edcre.exe                    # edcre v1.1.0 (53 KB)
├── duckstation/                # Emulator for boot testing
│   ├── duckstation-qt-x64-ReleaseLTCG.exe  # DuckStation (10 MB)
│   ├── bios/US/SCPH1001.BIN         # SCPH-1001 US BIOS (512 KB)
│   ├── settings/duckstation-qt.ini  # DuckStation config
│   ├── portable.ini                 # Portable mode flag
│   ├── qt.conf                      # Qt plugin config
│   ├── *.dll                        # Required DLLs (~120 MB)
│   ├── QtPlugins/                   # Qt platform plugins
│   ├── resources/                   # UI resources
│   └── translations/                # Qt translations
├── docs/                       # Documentation (this file)
├── compile.bat                 # Compile builder from source
├── build.bat                   # Build Frankenstein disc
├── build_comparison.bat        # F-3: Build with/without XA stub
└── boot_test.bat               # Launch DuckStation boot test
Total footprint: ~168 MB (158 MB is DuckStation + DLLs; builder/translation/edcre = ~7 MB)

3. Prerequisites
For Building (compile.bat)
Visual Studio 2022 Build Tools (vcvars64.bat)
Install: https://visualstudio.microsoft.com/visual-cpp-build-tools/
Select: "Desktop development with C++"
The compiler path must be: C:\Progra~2\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat
For Building (pre-compiled)
None. frankenstein_build.exe is a standalone Windows x64 binary.
For Boot Testing
DuckStation is included in the duckstation/ folder
SCPH-1001 BIOS is included at duckstation/bios/US/SCPH1001.BIN
MD5: 924E392ED05558FFDB115408C263DCCF
Source Disc (NOT included — too large for repo)
Place the DQ4 Japan disc at:

frankenstein_pipeline/disc/Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin
Expected SHA-256: 100D87DB9DEADF8F9FA4BB891D3A5D0BB112ACBF5ADBCBC93C637848ED9C7531 Size: 368,057,424 bytes (351 MB)

4. Quick Start
Build the disc (using pre-compiled builder)
cd frankenstein_pipeline
build.bat
Output: build_staging/dq4_frankenstein_v109.bin + .cue

Build from source
cd frankenstein_pipeline
compile.bat
build.bat
Boot test
cd frankenstein_pipeline
boot_test.bat build_staging\dq4_frankenstein_v109.cue
F-3 Comparison (with/without XA stub)
cd frankenstein_pipeline
build_comparison.bat
5. Builder Architecture
5.1 frankenstein_build_main.cpp
Entry point: parses CLI args, validates inputs, orchestrates build
Build modes: Forward (DW7 base) and Reverse (DQ4 base, Option A)
Verification gates: 19 hard gates that abort the build on failure
Manifest: writes build_hash_manifest.json with SHA-256, build time, argv, patch counts
5.2 psx_binary_ops.cpp
ExePatcher class: sector-aware EXE extraction and patching
HBD re-encoder: decodes DQ4 text, re-encodes with hybrid tree
Patch functions: disc-check bypass, LBA table, tree pointers, BSS clear, malloc clamp, XA stub (D1), MISS-1 stub (D2), tree hook, FIFO drain, VBlank bypass
EDC/ECC: two-phase edcre call (repair + verify)
5.3 Key Constants
Constant	Value	Description
EXE_LBA	24	EXE start sector on disc
EXE_LOAD_ADDR	0x80017F00	PSX RAM load address
ORIGINAL_PC0	0x8008E284	DW7 original entry point
NEW_PC0	0x800BDF00	Patched entry point (MISS-1 stub)
HBD_TARGET_SECTOR	362	HBD location on output disc
SECTOR_SIZE	2352	Mode 2 Form 1 raw sector
SECTOR_DATA	2048	User data per sector
SECTOR_HEADER	24	Header + subheader bytes
6. Verification Gates (19 total)
Gate	Description	Type
1	EXE magic (PS-X EXE)	Metadata
2	Output disc exists	Metadata
3	CUE sheet exists	Metadata
4	Disc size > 350 MB	Metadata
5	Tree patches > 0	Count
6	Disc-check patches == 3	Count
7	HBD blocks processed > 0	Count
8	HBD blocks converted > 0	Count
9	XA stub sw_2038 (D1 guard)	Sector-aware readback
10	MISS-1 stub[7] ADDIU (D2 guard)	Sector-aware readback
11	Disc-check 3 sites == 0x24020001	Sector-aware readback
12	Malloc clamp trampoline (9 words)	Sector-aware readback
13	Tree hook JAL at 0x800562B8	Sector-aware readback
14	FIFO drain J at 0x8008B65C	Sector-aware readback
15	Decomp root ID JAL at 0x80057FEC	Sector-aware readback
16	BSS clear range (3 words)	Sector-aware readback
17	LBA ref at 0x8001B194 == 0x16A	Sector-aware readback
18	Seq table[0] at 0x800BB0AA	Sector-aware readback
19	Decomp tree ptr LUI at 0x800562EC	Sector-aware readback
Gates 9-19 use sector-aware reads: LBA = 24 + file_off/2048, offset = file_off%2048, read at LBA*2352 + 24 + sec_off.

7. Audit Fixes Applied (V.99)
All open findings from BUILDER_CONSOLIDATED_AUDIT_Aug10_2026.md have been addressed:

Finding	Severity	Fix
F-2	MEDIUM	Thread blob verification script (verify_thread_blob.py)
F-3	HIGH	--no-xa-stub comparison disc support, Gate 9 skip logic
F-5	MEDIUM	WARNING guards promoted to hard aborts (2 sites in psx_binary_ops.cpp)
F-7	MEDIUM	run_edcre refactored: BIN file, two-phase (repair + verify), exit 1 distinct
F-8	MEDIUM	Banner context-aware: only prints relevant flags in reverse mode
F-9	MEDIUM	__TIMESTAMP__ replaced with time(), argv, patch counts in manifest
8. Build Output
v109 (latest)
SHA-256: AFED72C6416844D6DE6EE4A350670A9D7C2A9DA5C7535FD205E0BAB8CEEA8B85
Size: 368,057,424 bytes (351.0 MB)
Patches: tree=24, disc=3, lba=1, seq=44, trampoline=2
HBD: 3573 blocks processed, 1527 converted, 3148 sub-block swaps
Gates: 19/19 PASS
Determinism: byte-identical across multiple builds with same inputs
v109-no-xa (F-3 comparison)
SHA-256: A220D3A8B57B119A4B75E9541EB8A668E9F3DEF33F2F3C8F72B6FFFEECDA91A7
Gates: 18/19 PASS (Gate 9 SKIP as expected)
9. Translation Pipeline
Input Files
File	Size	Description
dw7_encoded_translation.json	3.2 MB	Pre-encoded English text in Huffman-encoded format
full_translation_hybrid_playable.json	1.4 MB	Human-readable translation entries
dw7_hybrid_tree.bin	1472 bytes	Hybrid Huffman tree (DQ4+DW7 character set)
folder_mapping.json	290 KB	DW7→DQ4 sector/folder mapping
SLUSP012.06_clean.bin	675 KB	DW7 US EXE (unpatched)
Translation Map Generation
cd builder
python gen_translation_map.py
Reads dw7_encoded_translation.json, outputs translation_map.bin (TXMP format).

HBD Re-encoding
The builder's C++ HbdReencoder:

Reads DQ4 HBD blocks from the source disc at sector 362
For each block: decode using per-block Huffman tree
Re-encode using the global hybrid tree
Inject pre-encoded English text from translation_map.bin
Strip per-block tree, set hts=24, treeEnd=0, textEnd=0
Blocks that fail re-encoding get verbatim fallback (DW7 format headers, original text)
Standalone Patcher
dw7_exe_surgical_patcher.py can apply patches independently:

python dw7_exe_surgical_patcher.py <disc.bin> --apply-all
python dw7_exe_surgical_patcher.py <disc.bin> --verify-only
10. Known Issues & Limitations
Boot stall at ~18s: The EXE reads sectors 154-174 (ISO dir + SYSTEM.CNF + EXE start), then stalls. Likely HBD format incompatibility or CD-ROM polling loop. The XA stub (D1 fix) and MISS-1 stub (D2 fix) are designed to address this but have not been boot-tested with v109 yet.

Low MIPS density in thread blob: The MISS-1 thread blob (thread_code_blob.bin) has only 8% MIPS instruction density (43/512 words). The remaining 91% is zeros. This may indicate the blob is mostly data, not executable code. Runtime verification via DuckStation memory dump is needed.

HBD re-encoding partial: Of 3573 HBD blocks processed, only 1527 are converted. The rest use verbatim fallback. Some text may garble but the parser should not crash.

DuckStation DLLs: The duckstation/ folder is ~158 MB due to Qt6 and DirectX DLLs. This is the minimum required for DuckStation to run. If boot testing is not needed, this folder can be omitted.

Source disc not included: The DQ4 Japan disc (351 MB) is too large for the repo. It must be placed at disc/Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin before building.

11. File Inventory (with checksums)
Builder (1.2 MB)
File	Size	Purpose
frankenstein_build.exe	483 KB	Pre-compiled builder binary
frankenstein_build_main.cpp	42 KB	Builder source (main)
psx_binary_ops.cpp	187 KB	Builder source (ops)
psx_binary_ops.h	26 KB	Builder header
offset_registry.json	9 KB	Patch address registry
FROZEN_INPUTS.json	2.5 KB	Input checksums
translation_map.bin	445 KB	Binary translation map
thread_code_blob.bin	2 KB	MISS-1 thread blob
hotfixes_empty.txt	119 B	Empty hotfix file
reencode_manifest.json	3 KB	Re-encoding results
reencode_pipeline.py	17 KB	HBD pre-processor
gen_translation_map.py	2 KB	Translation map generator
verify_thread_blob.py	10 KB	F-2 verification
dw7_exe_surgical_patcher.py	35 KB	Standalone EXE patcher
Translation (5.6 MB)
File	Size	Purpose
SLUSP012.06_clean.bin	675 KB	DW7 US EXE
dw7_encoded_translation.json	3.2 MB	Pre-encoded English text
full_translation_hybrid_playable.json	1.4 MB	Full translation
folder_mapping.json	290 KB	Sector mapping
dw7_hybrid_tree.bin	1.5 KB	Hybrid Huffman tree
checksums.txt	1 KB	Frozen checksums
EDC/ECC (53 KB)
File	Size	Purpose
edcre.exe	53 KB	EDC/ECC repair + verify
DuckStation (158.5 MB)
File	Size	Purpose
duckstation-qt-x64-ReleaseLTCG.exe	10 MB	Emulator
SCPH1001.BIN	512 KB	US BIOS
*.dll + resources	~148 MB	Qt6, DirectX, SDL3, etc.
12. CLI Reference
frankenstein_build.exe
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
Exit Codes
0: Build complete, all gates passed
1: Build failed or gate failure



WHY THIS IS HARD
================

This is NOT a translation problem. The translation is done. This is a
binary compatibility problem between two PSX executables that share the
same game engine but use incompatible internal data formats:

  - DW7 Huffman: global tree, root_id = value & 0x7FFF
  - DQ4 Huffman: per-block trees, root_id = (value & 0x7FFF) + 1

The hybrid tree bridges this at the binary level. The thread blob
injection solves the BSS zeroing problem (the CD-ROM thread code lives
in BSS and gets wiped on boot). The boot copy stub ensures the blob
is in place BEFORE BSS clear runs.

The remaining issue is a BSS/LBA mismatch causing a BIOS exception
after initialization — the disc's volume descriptor still identifies
as Japanese, and the US BIOS/EXE may be rejecting it.

This requires someone who understands PSX BIOS internals, ISO 9660 PVD
structure, and MIPS R3000A exception handling. Not a text editor job.

===============================================================================

---------------------------------------------------------------
WHAT THIS IS
---------------------------------------------------------------

This pipeline treats PSX EXEs as fully programmable bootloaders.
The EXE is the driver. The HBD is the payload. We control both.

The Frankenstein approach transplants a DQ4 (Dragon Quest IV)
translated HBD payload onto a DW7 (Dragon Warrior VII) EXE
engine, patching the binary at the MIPS instruction level to
resolve format incompatibilities between the two games'

Huffman compression, block type dispatch, and CD-ROM handling.

---------------------------------------------------------------
WHAT YOU NEED
---------------------------------------------------------------

You must provide your own ROMs:

  1. Dragon Warrior VII (SLUS-01206) disc image (.bin/.cue)
  2. Dragon Quest IV (Japan) disc image with translated HBD

This repo contains NO copyrighted ROM data, no compiled
binaries, and no open-source dependencies. You will need to
obtain those separately:

  - edcre (EDC/ECC regeneration tool)
  - DuckStation (PSX emulator for testing)
  - Ghidra (for decompilation, if re-running analysis)

---------------------------------------------------------------
DIRECTORY STRUCTURE
---------------------------------------------------------------

  src/            C++ source — EXE patcher, emulator, builder
  scripts/        Python tools — orchestration, verification,
                  DuckStation bridge, telemetry, patching
  decompile/      Ghidra decompilation output and analysis
                  tools for DQ4 and DW7 EXEs
  data/           Config files, assembly source, patch specs
  docs/           Architecture docs and roadmaps
  mips/           MIPS assembler/disassembler Python package
  psx/            PSX EXE/ISO/HBD parsing Python package
  examples/       Example stubs, Lua scripts, blueprints

---------------------------------------------------------------
BUILD INSTRUCTIONS
---------------------------------------------------------------

  Native C++ build (MinGW or MSVC):

    g++ -std=c++17 -O2 -o frankenstein_build.exe \
        frankenstein_build_main.cpp psx_binary_ops.cpp

  Then run with your disc paths:

    frankenstein_build.exe \
        --dw7-disc <DW7.bin> \
        --dq4-hbd-disc <DQ4.bin> \
        --hybrid-tree <tree.bin> \
        --folder-map <mapping.json> \
        --output <output.bin> \
        --tree-mode 3 \
        --duckstation

  See frankenstein_build_main.cpp --help for full options.

---------------------------------------------------------------
PATCH ARCHITECTURE (build_v39, tree_mode=3)
---------------------------------------------------------------

  1. Disc check bypass (11 patches)
  2. BSS narrow (preserve hybrid tree region)
  3. Tree references (LUI/ADDIU -> 0x800BC700)
  4. Decompressor: tree pointer patch (per-block)
  5. Decompressor: root ID +1 patch
  6. Decompressor: offset_b mask removal
  7. CD-ROM stall bypass
  8. HBD type trampoline
  9. FMV skip
  10. CD-ROM command intercept
  11. Disc assembly + EDC/ECC + CUE sheet

---------------------------------------------------------------
CONTACT
---------------------------------------------------------------

  Bandcamp:  https://luxaura.bandcamp.com
  Facebook:  https://www.facebook.com/LuxAuraOfficial
  YouTube:   https://www.youtube.com/LuxAuraOfficial

  (cc) Lux Aura. For educational and preservation purposes.
===============================================================

