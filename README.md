# dq4frankenstein Progress ~97%
DQ4/DW7 Frankenstein Project V.9,V.95,V.96,V.97
===============================================================
  FRANKENSTEIN PIPELINE — DQ4/DW7 Binary Re-authoring Toolkit
===============================================================

  A PlayStation EXE patching and disc re-authoring framework
  for the Dragon Quest IV / Dragon Warrior VII Frankenstein
  translation project.

  Created by Lux Aura with the help of previous work by Markus Projects, Mandy Wilkens, and ChickenKnife.
  http://markus-projects.net/dragon-hackst-iv/
  https://github.com/mwilkens/dq4psxtrans
  https://www.romhacking.net/hacks/4275/
  https://luxaura.bandcamp.com
  https://www.facebook.com/LuxAuraOfficial
  https://www.youtube.com/LuxAuraOfficial


  ===============================================================================
    REVERSE FRANKENSTEIN BUILD PIPELINE — DQ4 PSX TRANSLATION
    Dragon Quest IV (PSX, SLPM-869.16) → English Translated Disc
===============================================================================
https://github.com/luxauraofficial777/dq4frankenstein
CONCEPT
-------
The DQ4 PSX translation is COMPLETE — all text strings are translated and
packed into the HBD archive in the Japanese DQ4 disc. The problem is that
the DQ4 executable (SLPM_869.16) uses a different Huffman tree format than
the DW7 US executable (SLUS-012.06). The "Reverse Frankenstein" approach
swaps the DW7 US EXE onto the DQ4 translated disc, then patches the EXE
to bridge the format differences. Think of it as: "Use the American engine
to drive the Japanese data — but the data is already in English."

    ┌─────────────────────┐       ┌─────────────────────┐
    │   DW7 US Disc       │       │   DQ4 JP Disc       │
    │   SLUS-012.06       │       │   SLPM-869.16       │
    │                     │       │                     │
    │  ┌───────────────┐  │       │  ┌───────────────┐  │
    │  │ DW7 US EXE    │──┼──┐    │  │ DQ4 JP EXE    │  │  (discarded)
    │  │ PC0:8008E284  │  │  │    │  │ PC0:800918F4  │  │
    │  │ Size:0xA5000  │  │  │    │  └───────────────┘  │
    │  └───────────────┘  │  │    │                     │
    │                     │  │    │  ┌───────────────┐  │
    │  ┌───────────────┐  │  │    │  │ HBD1PS1D.Q41  │──┼──┐
    │  │ HBD1PS1D.W71  │  │  │    │  │ (TRANSLATED)  │  │  │
    │  │ (English)     │  │  │    │  │ LBA: 362      │  │  │
    │  └───────────────┘  │  │    │  └───────────────┘  │  │
    └─────────────────────┘  │    └─────────────────────┘  │
                             │                             │
                             ▼                             │
    ┌─────────────────────────────────────────────────────┐ │
    │           FRANKENSTEIN BUILD TOOL                    │ │
    │           (frankenstein_build.exe --reverse)         │ │
    │                                                     │ │
    │  INPUTS:  DW7 US EXE  +  Hybrid Tree  +  DQ4 Disc   │◄┘
    │  OUTPUT:  dq4_reverse.bin / dq4_reverse.cue         │
    └─────────────────────────────────────────────────────┘
                             │
                             ▼
    ┌─────────────────────────────────────────────────────┐
    │           PATCHED OUTPUT DISC                       │
    │                                                     │
    │  Sector 16: PVD (volume ID)                         │
    │  Sector 22: ISO Directory (EXE renamed, LBA updated)│
    │  Sector 23: SYSTEM.CNF (boot=SLUSP012.06)           │
    │  Sector 24: DW7 US EXE (patched, 680,100 bytes)     │
    │  Sector 362: HBD1PS1D.Q41 (UNMODIFIED — translated) │
    └─────────────────────────────────────────────────────┘


EXE PATCH PIPELINE (25 STEPS) FRANKENSTEIN V0.95 (REV_FRANK) --reverse
=============================

 ┌─────────────────────────────────────────────────────────────────┐
 │  STEP  WHAT IT DOES                          RAM/FILE ADDR     │
 ├─────┼─────────────────────────────────────────────────────────┤
 │  1  │ Load DW7 US EXE into patcher           0x80017F00 load   │
 │  2  │ Load hybrid Huffman tree (bridge       0x800BC700 target │
 │     │   DQ4 per-block format → DW7 global)                     │
 │  2b │ Set trampoline base past tree          0x800BCF00        │
 │  3  │ Append hybrid tree to EXE payload      file 0xA5000      │
 │     │   Pad to sector boundary               +2048 align       │
 │     │   Update text_size in EXE header       offset 0x1C       │
 │  4  │ Patch all tree pointer references      0x800EF1C8→BC700  │
 │     │   (LUI/ADDIU pairs, N sites)                             │
 │  5  │ Narrow BSS clear range to preserve     START: 0x800BD100 │
 │     │   tree + trampolines + thread blob     END:   0x800D9E80 │
 │  6  │ Zero pre-tree BSS region (file bytes)  0xA5000 backfill  │
 │  7  │ Decompressor patches (3 targets):                        │
 │     │   T4: tree pointer variable init       0x800BC6F8        │
 │     │   T5: root_id +1 (DQ4 offset format)   decomp site 3     │
 │     │   T6: offset_b mask (16-bit vs 15-bit) decomp site 1&2   │
 │  8  │ Disc check bypass (9 surgical NOPs     0x6112C etc.     │
 │     │   + 1 variable patch, NO brute force)  0x387CC           │
 │  9  │ CD-ROM stall bypass — SKIPPED          (removed BAD-3)  │
 │     │   (was causing FIFO overflow in        DuckStation)      │
 │ 10  │ HBD type trampoline                    0x800BCF00        │
 │     │   (intercepts HBD header parsing)                         │
 │ 11  │ FMV stubs (replace XA/STR + MDEC)      0x8008AEF4,       │
 │     │   (BAD-2: function stubs, not Setmode) 0x8008CAD0        │
 │ 12  │ CD-ROM command intercept — SKIPPED     (removed BAD-1)  │
 │ 12b │ Patch HBD filename string in EXE:      file 0xF600       │
 │     │   'hbd1ps1d.w71' → 'hbd1ps1d.q41'      RAM 0x80026D00    │
 │     │   (MISS-2: 3-byte edit, no ISO rename)                   │
 │ 12c │ THREAD BLOB INJECTION (MISS-1):                          │
 │     │   Append thread blob (2048 bytes)      RAM 0x800BCF6C    │
 │     │   Append boot copy stub (56 bytes)     RAM 0x800BD76C    │
 │     │   Redirect PC0 → boot stub            0x8008E284→BD76C   │
 │     │   Stub copies blob → 0x800D9E80        (BSS region)      │
 │     │   then jumps to original PC0          → 0x8008E284       │
 │     │   BSS clear [BD100, D9E80) PRESERVES   the blob copy     │
 │ 12d │ UPDATE text_size in EXE header         offset 0x1C       │
 │     │   (CRITICAL: must include blob+stub    0xA506C→A58A4)    │
 │ 12e │ Record final EXE size                  680,100 bytes     │
 │ 13  │ Patch LBA references: 354→362          (HBD sector)      │
 │     │   Broad scan for immediate values                        │
 │ 14  │ Patch sequential sector table:         354→362           │
 │     │   (contiguous HBD read table)                            │
 │ 15  │ Verify EXE fits before HBD overlap     max 692,224 bytes │
 │ 16  │ Copy DQ4 disc as base (unmodified)     → dq4_reverse.bin │
 │ 17  │ Write patched DW7 EXE at sector 24     680,100 bytes     │
 │ 18  │ Update SYSTEM.CNF: boot filename       SLPM_869.16→      │
 │     │                                       SLUSP012.06       │
 │ 19  │ Update ISO directory:                                    │
 │     │   EXE: rename + LBA=24 + size          SLPM→SLUSP        │
 │     │   HBD: LBA=362 + size (no rename)      HBD1PS1D.Q41      │
 │ 20  │ Update PVD volume descriptor           sector 16         │
 │ 21  │ Write CUE sheet                         dq4_reverse.cue  │
 │ 22  │ Run EDC/ECC regeneration (edcre.exe)   all Mode 2 sects  │
 │ 23  │ Record final disc size                  ~368 MB          │
 │ 24  │ Record new PC0                          0x800BD76C       │
 │ 25  │ Integrity verification + patch summary  post-build check │
 └─────┴─────────────────────────────────────────────────────────┘


MEMORY MAP AFTER PATCHING
=========================

  RAM ADDR         CONTENTS                     SOURCE
  ───────────────  ───────────────────────────  ──────────────────
  0x80017F00       DW7 US EXE text (code)       Loaded from sector 24
  0x8008E284       Original PC0 (DW7 entry)     Jumps here after stub
  0x8008AEF4       FMV stub 1 (XA/STR skip)     BAD-2 patch
  0x8008CAD0       FMV stub 2 (MDEC skip)       BAD-2 patch
  0x800BC700       Hybrid Huffman tree           Appended payload
  0x800BCF00       HBD type trampoline           Step 10 patch
  0x800BCF6C       Thread blob (2048 bytes)      MISS-1 injection
  0x800BD100       BSS clear START               Narrowed from 0x800BCCC8
  0x800BD76C       Boot copy stub (new PC0)      14 MIPS instructions
  0x800D9E80       Thread blob destination       Copied here by stub
                   (BSS clear END — preserved)   BSS stops before this
  0x800EF1C8       Old tree pointer (patched)    → 0x800BC700


BOOT SEQUENCE
=============

  1. BIOS loads EXE from CD-ROM sector 24 into RAM at 0x80017F00
     └─ Loads text_size (0xA58A4) bytes = 678,052 bytes

  2. BIOS sets PC = PC0 = 0x800BD76C (boot copy stub)
     │
     ▼
  3. BOOT COPY STUB executes (14 instructions):
     │  lui  t0, <blob_src_hi>     # source = blob in loaded EXE
     │  addiu t0, t0, <src_lo>
     │  lui  t1, 0x800D            # dest = 0x800D9E80 (BSS region)
     │  addiu t1, t1, 0x9E80
     │  addiu t2, t0, <blob_size>  # end = src + 2048
     │  loop:
     │    lw   t3, 0(t0)           # load word from blob
     │    nop
     │    sw   t3, 0(t1)           # store to 0x800D9E80+
     │    addiu t0, t0, 4          # advance src
     │    addiu t1, t1, 4          # advance dst
     │    bne  t0, t2, loop        # until done
     │    nop
     │  j    0x8008E284            # jump to original DW7 entry point
     │  nop
     │
     ▼
  4. DW7 EXE INITIALIZATION (PC=0x8008E284):
     │  BSS clear: zero [0x800BD100, 0x800D9E80)
     │  └─ Thread blob at 0x800D9E80 SURVIVES (outside clear range)
     │  └─ Hybrid tree at 0x800BC700 SURVIVES (before clear start)
     │  └─ Trampolines at 0x800BCF00 SURVIVE (before clear start)
     │
     ▼
  5. DW7 game logic starts:
     │  CD-ROM init → reads SYSTEM.CNF → finds SLUSP012.06
     │  Opens HBD1PS1D.Q41 at LBA 362 (patched string 'q41')
     │  HBD parser uses hybrid tree (patched pointers)
     │  Decompressor bridges DQ4 per-block → DW7 global format
     │
     ▼
  6. GAME RUNS — translated text displays via DW7 engine


CURRENT STATUS
==============

  ✅ Translation: COMPLETE (all text in HBD1PS1D.Q41)
  ✅ EXE patches: ALL 25 steps applied and verified
  ✅ text_size fix: Header updated to 0xA58A4 (includes blob+stub)
  ✅ Boot stub: EXECUTES correctly (BSS clear runs full range)
  ✅ DuckStation: US BIOS (SCPH-5501) loads, POST sequence passes
  ✅ CyberGrime: 50M instruction stress test, no crash
  ✅ EDC/ECC: Regenerated, passes validation

  ❌ DuckStation: Stalls after BSS clear at PC=0xBFC0E5E8
     └─ BIOS exception handler area — likely disc region mismatch
  ❌ PVD volume ID: Still 'DRAGONQUEST4_EN' (Japanese)
     └─ DuckStation reports: disc=NTSC-J, console=NTSC-U
     └─ DW7 CD-ROM driver may reject disc based on region

  REMAINING WORK:
  1. Patch PVD volume ID to US-compatible string
  2. Trace 0xBFC0E5E8 exception (enable TTY logging)
  3. Possibly patch disc region check in DW7 EXE


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

