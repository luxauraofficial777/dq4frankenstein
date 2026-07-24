# CYBERGRIME
https://github.com/luxauraofficial777/cybergrime
CYBERGRIME V0.1 Developed by Lux Aura
Custom PSX EMU harness for QA/TEST/DEBUG http://facebook.com/LuxAuraOfficial http://youtube.com/LuxAuraOfficial http://luxaura.bandcamp.com

CYBERGRIME CUSTOM EMULATOR HARNESSS TEST/QA/DEBUG
PSX Emulator Harness & Binary Patching Toolkit for Dragon Quest IV Translation
CyberGrime is a from-scratch PlayStation (PSX) emulator, automated testing harness, and MIPS binary patching toolkit built specifically for reverse-engineering and localizing PSX games. It was developed as part of the DQ4 Frankenstein Project — an effort to translate the Japan-exclusive PSX remake of Dragon Quest IV into English by grafting the English-capable DW7 executable onto the DQ4 disc body.

Features
Custom PSX Emulator
R3000A CPU core — Full MIPS R3000A instruction set with load-delay slot modeling, CP0 system coprocessor, and exception handling
HLE BIOS — High-level emulation of PSX BIOS calls (A0/B0/C0 tables) with event system, thread management, and interrupt dispatch (ported from PCSX-Redux / OpenBIOS reference)
Hardware emulation — GPU (polygon rasterizer, VRAM), GTE (geometry transformation engine), SPU (ADPCM audio), MDEC (FMV decoder), XA-ADPCM, CD-ROM controller, DMA channels, timers
Memory map — Full 2MB RAM, scratchpad, I/O registers, expansion regions, BIOS ROM
Deterministic execution — Fixed-cycle instruction stepping for reproducible test runs
Testing & Automation
Headless agent runner — Run disc images for N instructions with full telemetry capture, stall detection, and crash analysis
GDB remote stub — Attach GDB to the running emulator for interactive debugging (--gdb 3333)
Lua scripting — Automate gameplay, inject inputs, and hook memory via embedded Lua engine
Interactive console — Live breakpoints, memory inspection, register dumps, and stepping
Save/load state — Serialize and restore emulator state for reproducible test scenarios
Rewind buffer — Circular snapshot buffer for time-reverse debugging
Event timeline — JSON export of every BIOS call, DMA transfer, CD-ROM command, and GPU operation
Screenshot capture — Dump framebuffer to BMP at any point during execution
Binary Patching Pipeline
EXE patcher — Type-safe MIPS instruction patching with patch recording, validation, and revert
ISO 9660 toolkit — Sector-level read/write, directory entry manipulation, PVD updates, SYSTEM.CNF editing
Frankenstein builder — Full build pipeline that patches a DW7 EXE and swaps it into a DQ4 disc image
Reverse Frankenstein builder — Alternative approach: DQ4 disc as base, patched DW7 EXE swapped in, HBD left unmodified at native sector
HBD re-encoder — Heart Beat Disc archive parser, Huffman tree codec, and text block re-encoder
Huffman tree parser — Dual-format parser supporting both DQ4 (per-block trees, root_id + 1) and DW7 (global tree, root_id & 0x7FFF) conventions
MIPS assembler — Compile-time instruction encoding (MIPS_JAL, MIPS_LUI, MIPS_ADDIU, etc.) with branch offset calculation
EDC/ECC regeneration — Integrates with edcre.exe for MODE2/2352 disc image error correction
Telemetry & Diagnostics
Structured telemetry — JSON output with instruction counts, VBlank counts, BIOS call logs, CD-ROM command traces, GPU command traces, memory access patterns, and crash diagnostics
Baseline comparison — Save and diff telemetry runs across build versions
Regression runner — Automated test profiles with pass/fail criteria
Crash analyzer — Post-mortem PC tracking, register dumps, and call stack reconstruction
CD-ROM trace comparison — Side-by-side diff of CD-ROM command sequences between builds
DuckStation bridge — Launch and supervise DuckStation for visual verification alongside headless testing
VectorDB
SQLite FTS5 search engine — Indexed corpus of all project files (source, docs, study notes, telemetry) with BM25 ranking
Cross-reference analysis — Semantic search across 21,000+ chunks to find related code, patches, and documentation
Telemetry ingestion — Auto-indexes new telemetry JSON files for searchable crash/stall history
Quick Start
Prerequisites
Windows 10/11 (primary platform)
MSVC 2022 Build Tools + Windows SDK 10
Python 3.10+ (for tooling scripts)
DuckStation (optional, for visual verification)
Build the Emulator
cd cybergrime
.\build_agent.bat clean       # clean + release build
.\build_agent.bat debug       # debug build with extra logging
Build the Frankenstein Pipeline (C++)
.\build_psx_ops.bat           # builds psx_binary_ops.dll (EXE patcher + ISO toolkit)
.\build_frank.bat             # builds frankenstein_build.exe (full build pipeline)
Run a Headless Test
# Smoke test: 5M instructions, 60s timeout
psx_agent_runner.exe disc.bin smoke_test telemetry.json 60000

# Full boot: 55M instructions with timeline + screenshot
psx_agent_runner.exe disc.bin full_test telemetry.json 600000 55000000 --timeline timeline.json --screenshot title.bmp

# Interactive debugger
psx_agent_runner.exe disc.bin debug_test telemetry.json 600000 --console

# GDB remote debugging
psx_agent_runner.exe disc.bin gdb_test telemetry.json 600000 --gdb 3333
# Then: gdb-multiarch -ex "target remote :3333"

# Lua automation
psx_agent_runner.exe disc.bin lua_test telemetry.json 600000 --lua bot.lua
Build a Hybrid Disc
# Reverse Frankenstein (DQ4 disc base + patched DW7 EXE)
frankenstein_build.exe --reverse ^
    --dq4-disc  dq4_translated.bin ^
    --dw7-exe   SLUSP012.06 ^
    --tree      dw7_hybrid_tree.bin ^
    --output    dq4_reverse.bin ^
    --cue       dq4_reverse.cue ^
    --edcre     edcre.exe
Regression Testing
# Run all test profiles
python regression_runner.py

# Run a specific profile
python regression_runner.py quick_smoke

# Save a telemetry baseline
python baseline_manager.py save build_v42 telemetry_smoke.json

# Compare a run against baseline
python baseline_manager.py compare build_v42 telemetry_smoke.json
ISO Toolkit
python iso_toolkit.py list disc.bin
python iso_toolkit.py info disc.bin
python iso_toolkit.py extract disc.bin SLUS_012.06 exe.bin
python iso_toolkit.py diff disc_a.bin disc_b.bin
python iso_toolkit.py edc disc.bin
python iso_toolkit.py patch disc.bin patches.json out.bin
CLI Flags
Flag	Description
--bp <addr>	Add log-only breakpoint
--bp-halt <addr>	Add halting breakpoint
--wp <addr>	Add memory watchpoint
--trace-from <addr>	Start instruction trace from address
--trace-to <file>	Trace output file
--inject <json>	RAM injection spec file
--monitor <json>	Buffer monitor spec file
--symbols <json>	Load symbol table for named addresses
--console	Enable interactive stdin debugger
--savestate-out <file>	Save emulator state on exit
--loadstate <file>	Load emulator state before boot
--screenshot <file.bmp>	Dump framebuffer to BMP on exit
--timeline <file.json>	Export event timeline as JSON
--gdb [port]	Start GDB remote stub (default 3333)
--rewind	Enable rewind snapshot buffer
--lua <file.lua>	Load Lua automation script
--psn00b-align	Map watchpoints to BIOS vectors + GTE registers
--huffman-trace	Active trace on decompression + text cache buffer
--rebuild-iso <xml>	Generate ISO rebuild manifest from layout XML
Architecture
cybergrime/
├── Emulator Core
│   ├── psx_agent_runner.cpp      — Entry point, CLI, emulation loop
│   ├── psx_emulator_core.cpp     — CPU step, memory, GPU, CD-ROM, DMA, timers
│   ├── psx_test_station.h        — MIPS CPU, PSX memory, hardware register defs
│   ├── psx_gpu.h                 — Polygon rasterizer and VRAM
│   ├── psx_gte.h                 — Geometry Transformation Engine
│   ├── psx_spu.h                 — Sound Processing Unit (ADPCM)
│   ├── psx_mdec.h                — Motion DECoder (FMV)
│   ├── psx_xa.h                  — XA-ADPCM audio
│   ├── psx_debugger.h            — Interactive CLI debugger
│   ├── psx_savestate.h           — Save/load state serialization
│   ├── psx_timeline.h            — Event timeline recorder
│   ├── psx_rewind.h              — Rewind snapshot circular buffer
│   ├── psx_gdb_stub.h            — GDB remote serial protocol stub
│   ├── psx_lua.h                 — Lua scripting engine bindings
│   ├── psx_cdrom_tracer.h        — CD-ROM command trace recorder
│   ├── psx_exe_disasm.h          — MIPS disassembler
│   ├── psx_reference_integrations.h — Reference test vectors
│   └── txrt_hle.h                — HLE BIOS call interception
│
├── Binary Patching
│   ├── psx_binary_ops.cpp/.h     — EXE patcher, ISO I/O, Frankenstein builder
│   ├── frankenstein_driver.cpp/.h — Driver framework, MIPS codegen, disc assembler
│   ├── frankenstein_build_main.cpp — CLI for build pipeline
│   ├── huffman_compat_hook.c     — Huffman tree compatibility shim
│   ├── rom_graft_verifier.h      — ROM graft validation
│   └── vfs_compare.cpp           — VFS comparison tool
│
├── Testing & Automation
│   ├── agent_harness.cpp/.h      — TTY interception, VFS logging, telemetry
│   ├── CybergrimeUI.cpp/.h       — UI rendering (VRAM viewer, heatmap, CRT)
│   ├── instrumentation_bus.cpp/.h — Instrumentation event bus
│   ├── orchestration_engine.h    — Blueprint-driven orchestration
│   ├── harness_runner.py         — Python harness runner
│   ├── regression_runner.py      — Regression test runner
│   ├── baseline_manager.py       — Telemetry baseline comparison
│   ├── closed_loop_controller.py — Closed-loop build-test-analyze cycle
│   ├── iteration_harness.py      — Iterative build-and-test harness
│   ├── duckstation_bridge.py     — DuckStation supervisor + automation
│   └── test_definitions.json     — Test profiles
│
├── Python Tooling
│   ├── iso_toolkit.py            — ISO 9660 manipulation
│   ├── psx_ops.py                — Python bindings for psx_binary_ops.dll
│   ├── mips_asm.py               — MIPS assembler/disassembler
│   ├── psx_exe.py                — PS-X EXE parser
│   ├── telemetry_encoder.py      — Telemetry serialization
│   ├── telemetry_diff_analysis.py — Telemetry diff engine
│   ├── cdrom_log_compare.py      — CD-ROM trace comparison
│   ├── crash_analyzer.py         — Post-mortem crash analysis
│   ├── pipeline_audit.py         — Build pipeline auditor
│   ├── constraints_engine.py     — Patch constraint validator
│   ├── patch_controller.py       — Patch application controller
│   ├── patch_injector.py         — Binary patch injector
│   ├── build_orchestrator.py     — Full build orchestration
│   ├── runtime_comparator.py     — Runtime behavior comparison
│   ├── tree_isomorphism_audit.py — Huffman tree isomorphism checker
│   ├── dq_vectordb.py            — VectorDB (SQLite FTS5 search)
│   └── examples/                 — Example scripts and configs
│
├── Diagnostics
│   ├── diag_regions.py           — Memory region diagnostics
│   ├── live_analysis.py          — Live runtime analysis
│   ├── react_debugger.py         — Reactive debugging engine
│   ├── dump_stall.py             — Stall state dumper
│   ├── verify_*.py               — Build verification scripts
│   └── read_*.ps1                — PowerShell diagnostic readers
│
└── Data
    ├── thread_code_blob.bin      — CD-ROM thread entry code (2048 B)
    ├── dw7_hybrid_tree.bin       — Hybrid Huffman tree (1472 B)
    ├── constraint_map.json       — Patch constraint definitions
    ├── golden_state.json         — Golden test state
    ├── surgical_disc_bypass.json — Disc check bypass patches
    └── dw7_function_map.json     — DW7 function address map
The Frankenstein Approach
The core challenge: Dragon Quest IV (PSX, Japan-only) uses the Heart Beat Engine (HBE), the same engine as Dragon Warrior VII (PSX, US release). DW7's EXE contains English font rendering and text display code; DQ4's does not. The Frankenstein approach grafts the DW7 EXE onto the DQ4 disc to gain English text capabilities.

Forward Frankenstein (legacy)
DW7 disc as base, DQ4 HBD (Heart Beat Disc archive) re-encoded and placed at DW7's expected sector. Required 37% block re-encoding, LBA remapping, and folder pointer updates. Abandoned due to re-encode failure rate and fragility.

Reverse Frankenstein (current)
DQ4 disc as base, DW7 EXE patched and swapped into the EXE slot (sector 24). DQ4 HBD stays unmodified at its native sector 362. The DW7 EXE is patched to:

Tree pointer redirect — Point Huffman decompressor at a hybrid tree placed in BSS-adjacent memory
Root ID +1 — Match DQ4's (value & 0x7FFF) + 1 convention vs DW7's value & 0x7FFF
Offset_b mask removal — Allow odd offset_b values (DQ4 requirement, DW7 forces even)
BSS clear narrowing — Preserve thread entry code and hybrid tree from being zeroed
Disc check bypass — 10 surgical patches to skip PSX disc authentication
HBD type trampoline — Runtime type dispatch for DQ4-specific sub-block types (32, 39, 40, 42, 46)
FMV skip — Stub XA/STR and MDEC init functions (DQ4 HBD has no FMV data)
LBA reference update — Redirect HBD sector references from 354 (DW7) to 362 (DQ4)
Thread code injection — Copy CD-ROM thread entry code to RAM 0x800D9E80 via boot stub
ISO metadata update — Rename EXE entry, update SYSTEM.CNF boot filename
Credits
Lux Aura — Project direction, LIMINAL LORE agentic toolchain, emulator development, binary patching, build pipeline Referenced repos: PSXOxide, PSXMister, pcstation, PSXN00B, duckstation https://github.com/EBonura/PSoXide https://github.com/Lameguy64/PSn00bSDK/ https://github.com/VelocityRa/pctation https://github.com/stenzek/duckstation
License
This project is an internal research and development toolkit. The emulator code, patching pipeline, and testing harness are developed by Lux Aura. Reference implementations (PCSX-Redux, OpenBIOS, MAME) are credited to their respective authors and licensed under their original terms.

No copyrighted game ROMs or disc images are included in this repository. Users must provide their own legally-obtained disc images.
