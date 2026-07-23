# dq4frankenstein
DQ4/DW7 Frankenstein Project
===============================================================
  FRANKENSTEIN PIPELINE — DQ4/DW7 Binary Re-authoring Toolkit
===============================================================

  A PlayStation EXE patching and disc re-authoring framework
  for the Dragon Quest IV / Dragon Warrior VII Frankenstein
  translation project.

  Created by Lux Aura with the help of previous work by Markus Projects, awilles, and ChickenKnife.
  http://markus-projects.net/dragon-hackst-iv/
  https://www.romhacking.net/hacks/4275/
  https://luxaura.bandcamp.com
  https://www.facebook.com/LuxAuraOfficial
  https://www.youtube.com/LuxAuraOfficial

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

