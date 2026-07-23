# DQLOSTTRANSLATION — Reverse Engineering, QA & Debugging Blueprint

**Author:** Lead Reverse Engineer / QA Architect (Cascade)  
**Date:** July 21, 2026  
**Scope:** `DQLOSTTRANSLATION/` and all subdirectories (`cybergrime/`, `translation-tools/`, `translation/`, `study/`)  
**Status:** ACTIVE — canonical reference for PS1 localization QA and debugging

---

## TABLE OF CONTENTS

1. [Target Environment & Architecture](#1-target-environment--architecture)
2. [Project Components & Pipeline](#2-project-components--pipeline)
3. [Debugging Taxonomy](#3-debugging-taxonomy)
4. [Test Harness Initialization Checklist](#4-test-harness-initialization-checklist)
5. [Core Session Rules & Engineering Directives](#5-core-session-rules--engineering-directives)
6. [Operational Ruleset](#6-operational-ruleset)
7. [Reasoning Hierarchy](#7-reasoning-hierarchy)

---

## 1. Target Environment & Architecture

### 1.1 Platform

| Property | Value |
|---|---|
| Target Hardware | PlayStation 1 (PS1/PSX) — MIPS R3000A @ 33.87 MHz |
| RAM | 2 MB main RAM (0x80000000–0x801FFFFF), 1 KB scratchpad (0x1F800000) |
| VRAM | 1 MB (1024×512 16bpp) |
| CD-ROM | MODE2/Form1, 2352-byte sectors, 2048-byte payload |
| Audio | 24-voice SPU, ADPCM, CD-DA, XA-ADPCM |
| GPU | 1024×512 16bpp, 4-bit/8-bit/16-bit CLUT, semi-transparency |
| GTE | Geometry Transformation Engine (COP2) |
| MDEC | Motion Decoder (JPEG-like, DMA ch1) |
| BIOS | HLE (A0/B0/C0 call tables) + real BIOS support (SCPH-39001, SCPH-70012, etc.) |

### 1.2 Emulation Stack

The project uses a **dual-emulator architecture**:

**CyberGrime Custom Emulator** (`cybergrime/`):
- MIPS R3000A interpreter with CP0 exception handling, branch delay slots, trace ring (65536 entries)
- HLE BIOS: A0/B0/C0 function tables, event system (EvCB), thread management (OpenTh/ChangeTh), CD-ROM file I/O
- Hardware: RAM, VRAM, scratchpad, CD-ROM registers (0x1F801800 series), DMA, timers, PAD stubs
- GTE (`psx_gte.h`), GPU (`psx_gpu.h`), SPU (`psx_spu.h`), MDEC (`psx_mdec.h`), XA-ADPCM (`psx_xa.h`)
- Debugger: CLI breakpoints/watchpoints (`psx_debugger.h`), GDB remote stub on port 3333 (`psx_gdb_stub.h`)
- Save states: Full snapshot/restore (`psx_savestate.h`)
- Timeline: Event recording for VBlanks, CD-ROM, DMA, IRQs, BIOS calls (`psx_timeline.h`)
- Rewind: State history for rollback (`psx_rewind.h`)
- Lua scripting: `psx_lua.h`
- Instrumentation bus: Named-pipe JSON Lines protocol for live Python sidecar (`instrumentation_bus.h`)
- Orchestration engine: Instruction-level hooks, register watches, audit log, bulk opcode search (`orchestration_engine.h`)

**DuckStation** (`DuckStation/`):
- Production-grade x64 emulator (v0.1.11580, ReleaseLTCG build)
- Used as reference validator and visual verification target
- Bridge: `duckstation_bridge.py` — ReadProcessMemory/WriteProcessMemory to live DuckStation process
- Supervisor: `duckstation_supervisor.py` — PC polling, stall detection, breakpoint/watchpoint management, patch injection
- Closed-loop controller: `closed_loop_controller.py` — DuckStation telemetry → orchestrator analysis → memory injection → verification

### 1.3 Key Architectural Decisions

- **CyberGrime is the acceptance gate.** All DQ4 project validation runs inside CyberGrime. DuckStation is optional reference only (per `BLUEPRINT_CyberGrime_Jul16_2026.md:7`).
- **Five Laws of the Frankenstein Pipeline** (enforced by `pipeline_rulebook.py`):
  1. **Zero Blind Commits** — Every patch must pass `golden_state.json` verification + telemetry justification + simulation backing
  2. **Hardware Ground Truth** — All telemetry must originate from the live DuckStation bridge (no synthetic data)
  3. **Historical Vector Pre-Check** — Crash traces must be queried against the vector database before debugging
  4. **Deterministic C++ Surgery** — Python may only simulate/scan/parse. Binary writes to disc images must go through C++ core with bounds checking
  5. **Pipeline Continuity** — Every asset transformation must maintain symbol and pointer resolution. Drift halts the build

### 1.4 Critical PSX Memory Map

| Region | Address Range | Size | Purpose |
|---|---|---|---|
| EXE Load | 0x80017F00–0x800BCF00 | ~675 KB | DW7 US EXE code + data |
| BSS | 0x800BC668–0x800F4980 | ~230 KB | Zeroed at boot (narrowed to 0x800BCCC8 in Frankenstein) |
| Thread Entry | 0x800D9E80 | — | CD-ROM loading thread (loaded from HBD, beyond EXE range) |
| Code Cave | 0x800B4A68–0x800B4E7C | 1057 B | Bootstrap + Huffman shims + vector shadow |
| Huffman Tree (DW7) | 0x800EF1C8 | ~1472 B | Global tree, root_id = val & 0x7FFF |
| Huffman Tree (DQ4) | per-block | variable | Per-block trees, root_id = (val & 0x7FFF) + 1 |
| Relocated Tree | 0x80100000 | 1480 B | Hybrid tree destination |
| BIOS Vectors | 0x800E2000–0x800E2400 | 1 KB | Exception handler vectors (shadow-protected) |
| HBD Filename | 0x80026D00 | — | "hbd1ps1d.w71" string check |
| Event Flag | 0x800B679C | 4 B | VBlank event polling stall address |
| CD Status | 0x800B9164, 0x800B91CC, 0x800B91CE | — | Pre-set CD-ROM status flags |

### 1.5 ISO / Disc Layout

| File | LBA | Sectors | Size |
|---|---|---|---|
| SYSTEM.CNF | 23 | 1 | 68 B |
| SLUSP012.06 (EXE) | 24 | 330 (DW7) / 331 (Frank) | 675,840 / 677,888 B |
| HBD1PS1D.W71 (HBD) | 354 (DW7) / 355 (Frank) | 155,975 | 618,563,584 B |

**Sector size:** 2352 bytes (MODE2/Form1), 2048-byte payload  
**EDC/ECC:** Must be regenerated after ANY binary modification (`edcre.exe`)  
**EXE header:** `PS-X EXE` magic at offset 0x00, entry PC at 0x10, load addr at 0x10, text_size at 0x14, data_size at 0x1C, SP at 0x30

---

## 2. Project Components & Pipeline

### 2.1 Translation Pipeline

#### 2.1.1 Translation Assets

| Asset | Location | Entries | Status |
|---|---|---|---|
| `translation_mapping.json` | `translation/` | 10,117 JP→EN | Complete |
| `full_translation.json` | `translation/` | 15,318 entries / 1,105 blocks | 100% coverage, 0 placeholders |
| Disemvowel pass | Applied to 13,704/15,318 lines | — | Stability layer (vowels removed to fit buffers) |
| `dq4_block_map.json` | Root | Block structure map | 278 KB |
| `dw7_block_catalog.json` | Root | DW7 block catalog | 903 KB |

#### 2.1.2 Translation Rules

- **No apostrophes** in translations (ShiftJIS encoding conflict)
- **All dialog lines end with `{0000}`** (null terminator)
- **No fullwidth spaces** (U+3000 → ASCII space, 166 entries fixed)
- **No tildes** (~ → hyphen, 2 entries fixed — tilde maps to wrong Unicode in ShiftJIS)
- **Every patch build MUST follow EDC/ECC regeneration** via `edcre.exe`
- **Use XDelta3 or PPF 3.0** for final patch (never IPS)
- **Do not distribute original .bin files**

#### 2.1.3 Java Patcher (`translation-tools/`)

The Java-based patcher (`dq4psx-patcher.jar`) handles:
- Huffman tree compression/decompression
- Text block extraction and re-insertion
- Pointer table updates (folder sector addresses)
- `TextPatcher.java` — `selectiveTranslation` skips individual sequences with missing referrers (not entire blocks)
- `HeartBeatDataWriter.java` — checks both DQ4 (`HBD1PS1D.Q41`) and DW7 (`HBD1PS1D.W71`) HBD filenames
- `DragonQuestBinaryFileWriter.java` — dual-format HBD/EXE file resolution

#### 2.1.4 Python HBD Patcher (`translation-tools/dq4_hbd_patcher.py`)

- 935/1,156 blocks patched (212 grew, skipped)
- Handles compressed script blocks with Huffman encoding
- Block types 23/24/25/27 extracted via `dq4_extract_234_25.py`

#### 2.1.5 Frankenstein Builder (`translation-tools/frankenstein_builder.py`)

193 KB unified build system with 7 profiles:
- `test_0`: Plain DW7 copy (baseline)
- `test_a`: DQ4 HBD at 354, no EXE patches (HBD incompatibility proof)
- `test_b`: DQ4 HBD at 355 + tree patches
- `test_d`: DW7 HBD + tree patches only
- `test_f`: ISO dir update only
- `frank_v12`: Full Frankenstein with DW7-preserved redirect pointer strategy
- `rc2`: RC2 fallback build

### 2.2 CyberGrime Test Harness

#### 2.2.1 Agent Runner (`cybergrime/psx_agent_runner.cpp`)

**Entry point:** `psx_agent_runner.exe <disc.bin> <profile_name> [max_instrs] [json_path] [wallclock_ms]`

Architecture:
- VFS hooks wrap `find_cd_file` and `cd_read` to log all file/sector access
- TTY pipe redirection (1 MB capture buffer)
- Freeze detection: PC history ring (256 entries), <16 unique PCs in 200M instrs = frozen
- VBlank stall detection: 20M instrs without VBlank = stalled
- JSON telemetry output with full register snapshots, BIOS call log, VFS access log
- Thread blob injection: pre-loads `thread_code_blob.bin` at 0x800D9E80
- HBD linear pre-load: 200 sectors past EXE range
- BIOS vector stubs at 0x800E2000–0x800E2400

#### 2.2.2 Agent Harness (`cybergrime/agent_harness.h`)

- **PassStatus enum:** RUNNING, COMPLETE, CRASH, FREEZE, BOOT_FAIL, TIMEOUT
- **VFS Access Records:** 512 max entries (resource ID, LBA, size, resolved, calling PC, instruction count)
- **Error Records:** 128 max (category: VFS/CDROM/CPU/BIOS/FREEZE, signature, PC, instruction count)
- **Register Snapshots:** Full GPR[32] + PC + HI/LO + CP0 registers
- **Memory Dump Records:** 256-byte raw memory snapshots at arbitrary addresses
- **Assertion system:** ASSERT_INFO, ASSERT_WARN, ASSERT_FAIL, ASSERT_PARITY
- **BIOS call log:** 512 max entries for triage

#### 2.2.3 Regression Runner (`cybergrime/regression_runner.py`)

- Loads `test_definitions.json` (16 tests, 5 profiles)
- Runs each test via `psx_agent_runner.exe`
- Evaluates pass/fail criteria against telemetry JSON
- Records results in SQLite database (`test_results.db`)
- Exit code: 0 if all pass, 1 if any fail

#### 2.2.4 Test Definitions (`cybergrime/test_definitions.json`)

**Profiles:**
| Profile | Tests | Purpose |
|---|---|---|
| `quick_smoke` | v36_smoke, v18_smoke, jp_original | 2-minute smoke test |
| `full_regression` | All 16 tests | Complete regression suite |
| `v43_closed_loop` | v43_hbd_reencode_verify, v43_smoke, v43_extended | V43 closed-loop |
| `emulator_only` | 8 boot/stability tests | No unit tests |
| `unit_only` | 7 unit tests | No disc required |

**Test categories:**
- **Boot/stability:** v36_smoke, v18_smoke, v18_extended, v26_stability, v43_smoke, v43_extended, jp_original, dw7_boot
- **Unit:** gte_rtps, gte_nclip, gpu_triangle, cdrom_readn, cdrom_getid, save_state_roundtrip
- **Python:** hbe_huffman_roundtrip, v43_hbd_reencode_verify

**Pass criteria examples:**
- `no_crash: true` — final_status must not be CRASH
- `min_vblank: N` — VBlank count must reach N
- `thread_executes: true` — thread entry at 0x800D9E80 must be reached
- `cdrom_sectors_delivered: N` — at least N CD-ROM sectors must be read
- `no_freeze: true` — freeze_type must be NONE

#### 2.2.5 Harness Runner (`cybergrime/harness_runner.py`)

- Single test, batch mode, and compare mode
- Parses `emulator_triage.log` into structured dict
- DW7 key address map for memory snapshot interpretation
- Breakpoint support: `--bp 0x80078FF4 --bp 0x80078EA4`

#### 2.2.6 DuckStation Bridge (`cybergrime/duckstation_bridge.py`)

- Windows process memory inspection via `ReadProcessMemory`/`WriteProcessMemory`
- Discovers PSX RAM (2 MB) by scanning DuckStation's virtual address space
- Discovers CPU::Core struct by matching known GPR values
- `read_mem(addr, size)` / `write_mem(addr, data)` — PSX address space
- `read_regs()` / `write_reg(name, value)` — MIPS GPR[32] + PC + HI/LO + CP0
- Telemetry stream to `build_orchestrator.py`

#### 2.2.7 DuckStation Supervisor (`cybergrime/duckstation_supervisor.py`)

- Launches DuckStation with `--nogui --fastboot`
- Polls PC/registers at configurable interval (default 50ms)
- Stall detection: 128-sample window, PC range tracking
- Software breakpoints (polled, not hardware)
- Memory watchpoints (polls for value changes)
- Patch injection queue
- Full session logging to JSON

#### 2.2.8 Closed-Loop Controller (`cybergrime/closed_loop_controller.py`)

**The "Circle of Truth"** — binds DuckStation telemetry → orchestrator analysis → memory injection → verification:

1. DuckStation bridge captures PC/regs/mem
2. Orchestrator analyzes telemetry against historical failures and golden state
3. Patch commands sent back via `write_mem`
4. Verification: 3 clean cycles required for "BUILD READY"

**Golden state expectations:**
- Boot sequence: copy_routine → tree_copy → bss_clear → thread_entry
- Thread entry at 0x800D9E80 must be non-zero
- Tree at 0x80100000 must be non-zero
- No stall at event flag poll (0x80098780, 0x8009885C, 0x80098840, 0x8009883C)

#### 2.2.9 Build Orchestrator (`cybergrime/build_orchestrator.py`)

- Historical failure signatures (HF-001 through HF-NNN) extracted from 500 MB of session logs
- State synthesis: cross-reference telemetry against historical signatures
- Constraint enforcement: golden_state.json verification
- Surgical threshold: every patch requires telemetry basis + simulation proof
- Pipeline exit: "Build Ready" only after 3 clean boot cycles with zero divergence
- Output format: `[OBSERVATION] | [ROOT CAUSE ANALYSIS] | [SIMULATED VERIFICATION] | [C++ PATCH INSTRUCTION]`

#### 2.2.10 Pipeline Auditor (`cybergrime/pipeline_audit.py`)

5-stage audit:
1. Asset packaging & LBA alignment
2. Huffman tree compression & decompression bounds
3. Cross-engine symbol resolution (DQ4 heart → DW7 EXE)
4. EXE patch coordinate verification (file offset → RAM address)
5. Runtime memory coordinate verification (DuckStation bridge)

**Critical addresses tracked:**
- `bss_clear_entry`: 0x8008E284
- `bss_clear_loop`: 0x8008E294
- `thread_entry`: 0x800D9E80
- `copy_routine_entry`: 0x800BCCF0
- `huffman_decompress`: 0x80041CE8
- `tree_dest`: 0x80100000
- `stall_event_flag`: 0x8009885C

#### 2.2.11 TXRT HLE Bypass (`cybergrime/txrt_hle.h`)

Surgical decompression bypass:
- Intercepts at function entry (0x80073670) before Huffman tree is accessed
- Reads block ID from caller's registers
- Looks up pre-decoded 2-byte DW7ASCII leaf values in TXRT payload
- Writes directly to output buffer, bypassing decompression entirely
- Returns to caller via `$ra` — engine never touches the incompatible tree

#### 2.2.12 Instrumentation Bus (`cybergrime/instrumentation_bus.h`)

Named-pipe protocol (`\\.\pipe\cybergrime_live`):
- **Emulator → Python:** state, mem, vfs, bp_hit, divergence events (JSON Lines)
- **Python → Emulator:** WRITE_MEM, WRITE_REG, SET_PC, SET_BREAKPOINT, DUMP_REGION, ROLLBACK
- Patch history with rollback for all RAM modifications

### 2.3 Build Inventory

| Build | Size | Status | Key Finding |
|---|---|---|---|
| DW7 US (baseline) | 711 MB | Boots on real hardware | Custom emu: 0 sectors read (emulator bug) |
| DQ4 JP (original) | 351 MB | Boots on real hardware | Reference for HBD format |
| v12 | 711 MB | PASS 193M custom, BLACK SCREEN real emu | Deprecated |
| v14 | 711 MB | CRASH 34.5M | Off-by-one pointer table entries |
| v16 | 711 MB | CRASH 4.55M | jr $zero at 0x800E23C4 (HBD format incompatibility) |
| v17 (heart) | 351 MB | Exists | Heart build |
| v36 | 711 MB | Smoke test defined | v34 engine + HBD overlay |
| v38a | 711 MB | Closed-loop tested | BSS narrow patch candidate |
| v40a | 1 GB | Extended disc | HBD sector extension |
| v42/v43/v44 | 711 MB | Latest builds | HBD re-encode to DW7 format |
| v45 | 1 GB | Latest extended | — |
| Level 6 | 711 MB | 200M+ no crash, engine executing | Bootstrap + Huffman shims + vector shadow |
| RC2 | 678 MB | Survives 50M, 125 VBlanks | DQ4 EXE fallback (abandoned per user directive) |
| Native DQ4 | 351 MB | Survives 50M, 125 VBlanks | DQ4 EXE (abandoned per user directive) |

### 2.4 Frankenstein Architecture

**Frankenstein ROM** = DW7 US EXE + DQ4 translated HBD + DW7 ROM body

**Two critical blockers:**
1. **DW7 EXE cannot parse DQ4 HBD format** — different Huffman tree (global vs per-block), different HBD archive structure, different pointer tables/LBA references
2. **Custom emulator CD-ROM pipeline incomplete** — game stuck in CD-ROM wait loops, status register returns 0x42 instead of 0x03

**HBD format differences:**
| Property | DW7 | DQ4 |
|---|---|---|
| Decompression addr | EXE 0x429F0 | EXE 0x2A5E8 |
| Tree format | Global tree at 0x800EF1C8 | Per-block trees |
| root_id | val & 0x7FFF | (val & 0x7FFF) + 1 |
| Dominant block type | Type 4 | Type 40/42 |
| hts (tree size) | ~1407 | 24 |
| Entries | — | 44,657 |
| Folders | — | 3,243 |

**Disc check bypass patches (6):**
- 0x80078FF4, 0x80078EA4, 0x800793A4, 0x800793F8, 0x80079458, 0x80079660

**Bootstrap code cave (0x800B4A68–0x800B4E7C):**
- 0x800B4A68: Bootstrap (130 instrs, 520 B)
- 0x800B4CD4–0x800B4D10: 4 Huffman shims (20 B each)
- 0x800B4D24: Vector shadow routine (13 instrs, 52 B)
- 0x800B4D58: Trampoline (4 instrs, 16 B)

---

## 3. Debugging Taxonomy

### 3.1 Crash Categories

| Category | Signature | Detection Method | Example |
|---|---|---|---|
| **CRASH_DIV** | PC in data/BSS region, jr $zero | PC outside EXE code range [0x80017F00, 0x800BCF00] | v16: jr $zero at 0x800E23C4 |
| **CRASH_PTR** | Bad function pointer from HBD misinterpretation | PC = data address, $ra in EXE range | v14: 0x8004BD71 (off-by-one pointer) |
| **CRASH_BIOS** | Unhandled BIOS call or missing HLE function | BIOS fn number not in A0/B0/C0 table | Unknown A:0x72 |
| **CRASH_OVERFLOW** | Text buffer overflow, compressed data exceeds allocation | Compressed size > buffer size | Script 552/3: 1001 > 1000 bytes |
| **CRASH_VECTOR** | BIOS vector region corrupted by HBD DMA | Memory at 0x800E2000–0x800E2400 is non-zero | HBD raw data overwrites exception handlers |

### 3.2 Freeze Categories

| Category | Signature | Detection Method | Threshold |
|---|---|---|---|
| **FREEZE_PC_STUCK** | PC unchanged for extended period | PC history ring, <16 unique PCs | 200M instrs |
| **FREEZE_TIGHT_LOOP** | Small PC set cycling | PC history ring, ≤32 unique PCs | 200M instrs |
| **FREEZE_VBLANK_STALL** | No VBlank IRQ delivered | VBlank counter unchanged | 20M instrs |
| **FREEZE_EVENT_POLL** | Game polling event flag at stall PCs | PC in {0x80098780, 0x8009885C, 0x80098840, 0x8009883C} | Any hit |
| **FREEZE_CDROM_WAIT** | Game waiting for CD-ROM event | CD-ROM status reg unchanged, WaitEvent looping | 50M instrs |

### 3.3 Text/Translation Defect Categories

| Category | Detection Method | Severity |
|---|---|---|
| **TEXT_OVERFLOW** | Compressed byte count > original buffer size | Critical — crash or truncation |
| **TEXT_MISSING_TERM** | Line does not end with {0000} | Critical — text bleed |
| **TEXT_APOSTROPHE** | Translation contains ' character | High — ShiftJIS encoding conflict |
| **TEXT_FULLWIDTH_SPACE** | Translation contains U+3000 | Medium — rendering artifact |
| **TEXT_JP_REMNANT** | Japanese characters in translated output | High — incomplete translation |
| **TEXT_PLACEHOLDER** | Translation is "..." or empty | Medium — needs real translation |
| **TEXT_BLOCK_SKIP** | Block skipped during patching (missing referrers) | Medium — block remains Japanese |
| **TEXT_FONT_GLYPH** | Character not in game's font table | High — renders as garbage or blank |

### 3.4 VRAM/GPU Defect Categories

| Category | Detection Method | Severity |
|---|---|---|
| **VRAM_CORRUPTION** | Framebuffer diff between expected and actual | Critical |
| **VRAM_OVERFLOW** | Sprite/tile data exceeds VRAM page bounds | High |
| **GPU_BLACK_SCREEN** | All framebuffer dumps identical (zero pixels written) | Critical |
| **GPU_NO_DMA** | GPU DMA channel never triggers | High |
| **VWF_RENDER_ERROR** | Variable-width font exceeds text box bounds | Medium |

### 3.5 Historical Failure Signatures

From `build_orchestrator.py` HISTORICAL_FAILURES:

| ID | Name | Root Cause | Verified Fix | Status |
|---|---|---|---|---|
| HF-001 | BSS_ZEROS_THREAD_ENTRY | BSS clear loop zeros thread entry at 0x800D9E80 | Narrow BSS end from 0x800F4980 to 0x800BCCC8 | UNRESOLVED (EXE patch) |
| HF-002 | HBD_SECTOR_EXTENSION_CORRUPTION | Disc extension LBA 302537 has raw zeros | — | — |
| HF-003+ | Additional signatures in build_orchestrator.py | — | — | — |

---

## 4. Test Harness Initialization Checklist

### 4.1 Environment Prerequisites

- [ ] **Python 3.x** on PATH (for regression runner, harness runner, bridge, supervisor, orchestrator)
- [ ] **Java JDK + Maven** installed (for Java-based patcher)
  - `DQLOSTTRANSLATION/maven/` contains Maven distribution
  - `DQLOSTTRANSLATION/maven_build.log` for build verification
- [ ] **MSVC build tools** (for CyberGrime emulator compilation)
  - `cybergrime/build_agent.bat` — single `cl` invocation
- [ ] **edcre.exe** available at `edcre/edcre-v1.1.0-windows-x86_64-static/edcre.exe`
- [ ] **xdelta3.exe** available at `DQLOSTTRANSLATION/xdelta3.exe`
- [ ] **DuckStation** at `DuckStation/duckstation-qt-x64-ReleaseLTCG.exe`
- [ ] **PSX BIOS files** in `DQLOSTTRANSLATION/bios/` (SCPH-39001, SCPH-70012, scph1001-us, etc.)

### 4.2 CyberGrime Emulator Build

- [ ] `cybergrime/build_agent.bat` succeeds
- [ ] `psx_agent_runner.exe` produced (25 MB)
- [ ] `psx_headless.exe` produced (8.7 MB)
- [ ] `psx_test_station.exe` produced (237 KB)
- [ ] `psx_autoplayer.exe` produced (166 KB)
- [ ] `vfs_compare.exe` produced (197 KB)
- [ ] `hbd_diag.exe` produced (255 KB)

### 4.3 DuckStation Configuration

- [ ] `DuckStation/settings.ini` exists with correct paths
- [ ] BIOS SearchDirectory set to `DQLOSTTRANSLATION\bios\`
- [ ] `DuckStation/portable.txt` exists (portable mode)
- [ ] `DuckStation/cheats/SLUS-01206.pnach` exists (if cheat codes needed)
- [ ] `DuckStation/savestates/` writable
- [ ] `DuckStation/memcards/` writable
- [ ] DuckStation version: 0.1.11580 (verify via `--version`)

### 4.4 Disc Image Verification

- [ ] **Original DQ4 JP:** `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` — MD5: `F3D566CCE1AE08A9FFA4699EFDF1590E`
- [ ] **DW7 US baseline:** `DW7D1\DW7D1.bin` — MD5: `AEB42BF45ABDE4716BF833F00ECC5325`
- [ ] **DW7 EXE:** `translation\SLUSP012.06` — MD5: `2F135DF6CC653B21F73A0DE5D919B0D9`
- [ ] **Frankenstein build under test:** verify MD5 against `ARTIFACT_MANIFEST.md`
- [ ] **ISO 9660 directory:** PVD valid, root directory parses, file bounds within disc image
- [ ] **EXE header:** Magic = `PS-X EXE`, entry PC = 0x80017F00, text_size = 0xA4800
- [ ] **EDC/ECC:** Regenerated after last binary modification (check `edcre_*.log`)

### 4.5 Test Suite Initialization

- [ ] `cybergrime/test_definitions.json` parses without error
- [ ] `cybergrime/test_results.db` (SQLite) initialized
- [ ] All disc paths in test definitions resolve to existing files
- [ ] `cybergrime/golden_state.json` loads (constraint map for orchestrator)
- [ ] `cybergrime/constraint_map.json` loads (critical addresses)
- [ ] `cybergrime/thread_code_blob.bin` exists (2048 B, for thread injection)

### 4.6 Pre-Test Smoke

- [ ] Run `python regression_runner.py unit_only` — all 7 unit tests pass
- [ ] Run `python regression_runner.py quick_smoke` — 3 smoke tests pass
- [ ] Verify `emulator_triage_*.log` files are produced
- [ ] Verify `telemetry_*.json` files are produced with valid structure
- [ ] Verify `diag_traps.log` is written (diagnostic log bypasses harness stdout pipe)

### 4.7 DuckStation Bridge Pre-Flight

- [ ] `duckstation_bridge.py` can find DuckStation executable
- [ ] `duckstation_bridge.py` can launch DuckStation with a disc
- [ ] `wait_for_ram(timeout=30)` succeeds — PSX RAM base discovered
- [ ] `read_pc()` returns a valid MIPS address (0x80000000–0x801FFFFF or 0xA0000000–0xA01FFFFF)
- [ ] `read_all_regs()` returns 32 GPR values + PC + HI/LO + CP0
- [ ] `read_mem(0x80017F00, 8)` returns `PS-X EXE` magic

---

## 5. Core Session Rules & Engineering Directives

### 5.1 Byte-Alignment & Checksum Integrity

**R-B1: EDC/ECC regeneration is mandatory.** Every ISO/BIN modification — no matter how small — must be followed by `edcre.exe` regeneration. A single byte change without EDC/ECC update will cause DuckStation to reject sectors or produce silent data corruption.

```powershell
edcre\edcre-v1.1.0-windows-x86_64-static\edcre.exe --repair dq4_frankenstein_vXX.bin
```

**R-B2: ISO 9660 directory consistency.** After any file size change in the ISO, the directory entry must be updated with correct LBA and size (both little-endian and big-endian). Use `validate_iso.py` to verify PVD, root directory, and file bounds.

**R-B3: EXE header integrity.** The EXE header at disc LBA 24 must have:
- Magic: `PS-X EXE` at offset 0x00
- Entry PC: 0x80017F00 at offset 0x10
- Load addr: 0x80017F00 at offset 0x10
- text_size: 0xA4800 at offset 0x14 (or updated value if EXE extended)
- data_size: covers full loadable image at offset 0x1C
- SP: 0x801FFFFC at offset 0x30

Use `validate_exe.py` to verify before and after patching.

**R-B4: File size preservation.** The final patched binary must match the original file size (368,057,424 bytes for DQ4 JP, 711,567,024 bytes for DW7/Frankenstein). Size changes indicate buffer overflow or data loss.

**R-B5: MD5/SHA256 tracking.** Every build must be hashed and recorded in `ARTIFACT_MANIFEST.md`. No unversioned binaries ship. The manifest is the canonical provenance record.

### 5.2 Logging & Telemetry

**R-L1: Hardware breakpoint logging.** All breakpoint hits must log:
- PC at hit time
- Full register snapshot (GPR[32] + PC + HI/LO + CP0 Status/Cause/EPC)
- Instruction count
- Breakpoint label/description
- Hit count

**R-L2: Script overflow logging.** When a compressed script block exceeds its buffer:
- Block ID
- Original compressed size
- New compressed size
- Buffer capacity
- Action taken (truncated/skipped/grew)
- Pointer table impact (did folder shift?)

**R-L3: VRAM corruption logging.** During text-heavy scene transitions:
- Framebuffer diff (before/after)
- VRAM page write addresses
- DMA channel 2 (GPU) transfer log
- Text box render coordinates
- VWF advance metrics

**R-L4: CD-ROM trace logging.** All CD-ROM register writes and commands:
- `cdrom_trace.log` (capped at 20,000 entries via `CDLOG_MAX`)
- `CDROMTracer` (`psx_cdrom_tracer.h`) records command, response, IRQ, data FIFO
- Sector read log: LBA, success/fail, calling PC

**R-L5: BIOS call logging.** All HLE BIOS calls logged with:
- Function table (A0/B0/C0)
- Function number
- Arguments ($a0-$a3)
- Return value ($v0)
- Instruction count
- Calling PC

**R-L6: Telemetry JSON schema.** Every test run must produce a telemetry JSON with:
- `final_status`: RUNNING/COMPLETE/CRASH/FREEZE/BOOT_FAIL/TIMEOUT
- `instr_count`: total instructions executed
- `vblank_count`: VBlank IRQs delivered
- `final_pc`: last PC before termination
- `crash_reason`: if CRASH, human-readable reason
- `freeze_type`: if FREEZE, PC_STUCK/TIGHT_LOOP/VBLANK_STALL
- `cdrom_sectors_read`: total sectors delivered
- `cdrom_cur_lba`: last LBA read
- `bios_calls`: array of {fn_table, fn_num, args, instr_count}
- `vfs_access`: array of {resource, lba, size, resolved, calling_pc}
- `register_snapshot`: full GPR + CP0 dump at termination

### 5.3 Symbol & Offset Documentation

**R-D1: Symbol offset mapping.** Every patched address must be documented with:
- RAM address (0x800xxxxx)
- File offset (RAM - 0x80017F00 + 0x800)
- Original instruction/value (hex + disassembly)
- Patched instruction/value (hex + disassembly)
- Reason for patch
- Historical failure ID (if applicable)

**R-D2: Pointer table shift tracking.** When text patching shifts folder locations:
- Original folder sector
- New folder sector
- Pointer table entry index
- Pointer table location (absolute at EXE 0x4184, relative at EXE 0x511C)
- Old pointer value → New pointer value

**R-D3: Compression algorithm documentation.** Any Huffman tree modification:
- Tree type (global vs per-block)
- Tree location (RAM address)
- Tree size (bytes)
- root_id formula (DW7: `val & 0x7FFF`, DQ4: `(val & 0x7FFF) + 1`)
- Leaf count
- Block IDs affected

**R-D4: Test harness execution steps.** Every test run must record:
- Command line invoked
- Disc image path
- Profile name
- Max instructions / wall-clock timeout
- Telemetry JSON path
- Triage log path
- Pass/fail verdict with reason
- Duration (ms)

**R-D5: Session progress.** All progress documented in `study/` with dated filenames (`*_Jul21_2026.md` convention). Single-file progress tracking per session.

---

## 6. Operational Ruleset

### Session Rules

| ID | Rule | Enforcement |
|---|---|---|
| SR-1 | **No binary writes from Python.** Python may scan, parse, and simulate. All binary disc writes must go through C++ core with bounds checking. | `pipeline_rulebook.py` Law 4 enforcement |
| SR-2 | **Vector database pre-check before debugging.** Crash traces must be queried against `vector_db.py` before any new analysis. Historical fixes take priority. | `pipeline_rulebook.py` Law 3 enforcement |
| SR-3 | **Golden state verification before patches.** Every patch must pass `golden_state.json` verification AND have telemetry justification. | `pipeline_rulebook.py` Law 1 enforcement |
| SR-4 | **3 clean cycles for BUILD READY.** No build is declared ready until 3 consecutive clean boot cycles with zero divergence. | `closed_loop_controller.py` enforcement |
| SR-5 | **No telemetry older than 5 seconds.** Stale telemetry is rejected by the orchestrator. | `pipeline_rulebook.py` Law 2 enforcement |
| SR-6 | **Frankenstein approach ONLY.** Native DQ4 and RC2 are abandoned. All effort goes to DW7 EXE + DQ4 HBD. | User directive |
| SR-7 | **No original .bin distribution.** Use XDelta3 or PPF 3.0 for release patches. | Legal constraint |
| SR-8 | **Every build is hashed and manifest-tracked.** No unversioned binaries. | `ARTIFACT_MANIFEST.md` |

### Pipeline Rules

| ID | Rule | Rationale |
|---|---|---|
| PR-1 | **EDC/ECC after every binary modification.** | DuckStation rejects sectors with bad EDC/ECC |
| PR-2 | **ISO directory consistency after file size changes.** | LBA/size must be correct in both endianness |
| PR-3 | **EXE header integrity verified pre/post patch.** | Corrupted header = boot failure |
| PR-4 | **File size must match original.** | Size change = buffer overflow or data loss |
| PR-5 | **Pointer table continuity.** Every text shift must update pointer tables. | Drift halts build per Law 5 |
| PR-6 | **Huffman tree format must match EXE parser.** | DW7 global tree vs DQ4 per-block trees = architectural incompatibility |
| PR-7 | **Disc check bypass applied to all Frankenstein builds.** | 6 patches at known addresses |
| PR-8 | **BSS clear range must preserve thread entry.** | BSS must not zero 0x800D9E80 |

### QA Rules

| ID | Rule | Rationale |
|---|---|---|
| QR-1 | **Quick smoke before extended runs.** Always run `quick_smoke` profile before `full_regression`. | Catch build-breaking issues early |
| QR-2 | **Unit tests are disc-independent.** Run `unit_only` profile to verify emulator core before testing ROMs. | Isolate emulator bugs from ROM bugs |
| QR-3 | **Telemetry diff on regression failure.** When a previously-passing test fails, diff old and new telemetry JSON. | Isolate the specific change that broke |
| QR-4 | **DuckStation bridge for visual verification.** CyberGrime cannot render graphics. Use DuckStation for visual checks. | CyberGrime GPU is minimal |
| QR-5 | **Closed-loop controller for iterative patching.** Use `closed_loop_controller.py` for automated patch-test-verify cycles. | Reduces manual iteration |
| QR-6 | **Historical failure signature check.** Before debugging a new crash, check `build_orchestrator.py` HISTORICAL_FAILURES. | Avoid re-investigating known issues |
| QR-7 | **Pipeline audit before build sign-off.** Run `pipeline_audit.py --full` before declaring a build ready. | 5-stage audit catches alignment/symbol/coordinate errors |

### Debugging Rules

| ID | Rule | Rationale |
|---|---|---|
| DR-1 | **Root cause over workaround.** Identify the root cause before implementing a fix. Avoid masking failures (e.g., ChangeTh skip-switch). | Skip-switch masked v16 failure for 50M instrs |
| DR-2 | **Differential analysis on crash.** Compare crash telemetry to last known good state. Isolate the specific instruction divergence. | `build_orchestrator.py --diff` |
| DR-3 | **Diagnostic PC dump before crash.** Use `--diag-pc` or `--diag-pc-quit` to capture register/memory state before the target instruction. | Post-crash state is unreliable |
| DR-4 | **Trace from divergence point.** When PC diverges from expected path, enable trace from the divergence instruction. | `psx_debugger.h` trace_from_addr |
| DR-5 | **GDB stub for interactive debugging.** Connect `gdb-multiarch` to port 3333 for step-through debugging. | `psx_gdb_stub.h` |
| DR-6 | **Instrumentation bus for live analysis.** Use named-pipe protocol for real-time Python sidecar analysis. | `instrumentation_bus.h` |
| DR-7 | **Save state before risky patches.** Snapshot emulator state before applying runtime patches. Rollback if patch fails. | `psx_savestate.h` + `psx_rewind.h` |

### Documentation Rules

| ID | Rule | Rationale |
|---|---|---|
| DR-1 | **All patches documented with RAM address, file offset, old/new values.** | Provenance and reproducibility |
| DR-2 | **All test runs documented with command, disc, profile, verdict, duration.** | Regression tracking |
| DR-3 | **All crash investigations documented with telemetry, triage log, root cause.** | Historical knowledge base |
| DR-4 | **Study documents dated and scoped.** `*_Jul21_2026.md` convention. | Chronological ordering |
| DR-5 | **ARTIFACT_MANIFEST.md updated with every new build.** | Canonical provenance |
| DR-6 | **False hypotheses recorded.** Document rejected theories to avoid re-investigation. | HF-001 lists 4 false hypotheses |

---

## 7. Reasoning Hierarchy

```
DQLOSTTRANSLATION — PS1 Localization & QA
│
├── Target Layer
│   ├── Platform: PS1 (MIPS R3000A, 2MB RAM, 1MB VRAM)
│   ├── Emulation: CyberGrime (custom, acceptance gate) + DuckStation (reference)
│   ├── Disc: MODE2/Form1, 2352-byte sectors, EDC/ECC mandatory
│   └── Focus: Frankenstein ROM (DW7 EXE + DQ4 HBD + DW7 body)
│
├── Translation Pipeline
│   ├── Assets: 15,318 entries, 1,105 blocks, 10,117 JP→EN mappings
│   ├── Java Patcher: Huffman compression, pointer updates, text re-insertion
│   ├── Python Patcher: 935/1,156 blocks, HBD format handling
│   ├── Frankenstein Builder: 7 profiles (test_0 through frank_v12, rc2)
│   ├── Rules: No apostrophes, {0000} terminators, EDC/ECC after every build
│   └── Output: XDelta3 or PPF 3.0 patch (never IPS, never raw .bin)
│
├── CyberGrime Test Harness
│   ├── Agent Runner (psx_agent_runner.exe)
│   │   ├── VFS hooks (find_cd_file, cd_read, read_cd_sector)
│   │   ├── Freeze detection (PC ring, VBlank stall, event poll)
│   │   ├── Thread blob injection (0x800D9E80)
│   │   ├── HBD pre-load (200 sectors past EXE)
│   │   └── JSON telemetry output
│   │
│   ├── Agent Harness (agent_harness.h)
│   │   ├── PassStatus: RUNNING/COMPLETE/CRASH/FREEZE/BOOT_FAIL/TIMEOUT
│   │   ├── VFS records (512 max), Error records (128 max)
│   │   ├── Register snapshots, Memory dumps (256 B each)
│   │   ├── Assertion system (INFO/WARN/FAIL/PARITY)
│   │   └── BIOS call log (512 max)
│   │
│   ├── Regression Runner (regression_runner.py)
│   │   ├── test_definitions.json: 16 tests, 5 profiles
│   │   ├── SQLite results database
│   │   └── Pass/fail evaluation against telemetry
│   │
│   ├── Debugger (psx_debugger.h)
│   │   ├── Breakpoints (halt or log-only)
│   │   ├── Watchpoints (1/2/4 byte)
│   │   ├── Single-step, trace, hex dump, backtrace
│   │   └── Symbol table (addr ↔ name)
│   │
│   ├── GDB Stub (psx_gdb_stub.h)
│   │   ├── TCP port 3333
│   │   ├── gdb-multiarch / lldb compatible
│   │   └── Read/write regs, read/write mem, breakpoints, step, continue
│   │
│   ├── Save State (psx_savestate.h)
│   │   ├── Full snapshot: CPU, RAM, BIOS, scratchpad, HW regs, GTE, GPU, SPU, CD, MDEC
│   │   └── Magic: 0x50535853 ("SXPS"), version 1
│   │
│   ├── Timeline (psx_timeline.h)
│   │   ├── 17 event types (VBlank, CDROM, DMA, IRQ, BIOS, BP, WP, freeze, inject, etc.)
│   │   ├── 100,000 event cap
│   │   └── JSON export for analysis
│   │
│   ├── Rewind (psx_rewind.h)
│   │   └── State history for rollback
│   │
│   ├── Instrumentation Bus (instrumentation_bus.h)
│   │   ├── Named pipe: \\.\pipe\cybergrime_live
│   │   ├── JSON Lines protocol
│   │   ├── Emit: state, mem, vfs, bp_hit, divergence
│   │   └── Inject: WRITE_MEM, WRITE_REG, SET_PC, SET_BREAKPOINT, DUMP_REGION, ROLLBACK
│   │
│   ├── Orchestration Engine (orchestration_engine.h)
│   │   ├── 64-slot instruction hook table
│   │   ├── PC auto-corrector for illegal opcodes
│   │   ├── Register watches at specific PCs
│   │   ├── Audit log with rollback
│   │   └── DQ4 ↔ DW7 dual-binary address map
│   │
│   └── TXRT HLE Bypass (txrt_hle.h)
│       ├── Intercepts decompression at function entry
│       ├── Pre-decoded 2-byte DW7ASCII leaf values
│       ├── Bypasses Huffman tree entirely
│       └── 2 MB payload (txrt_payload.bin)
│
├── DuckStation Integration
│   ├── Bridge (duckstation_bridge.py)
│   │   ├── ReadProcessMemory / WriteProcessMemory
│   │   ├── RAM discovery (2 MB scan)
│   │   ├── CPU::Core struct discovery (GPR matching)
│   │   └── read_mem / write_mem / read_regs / write_reg
│   │
│   ├── Supervisor (duckstation_supervisor.py)
│   │   ├── PC polling (50ms default)
│   │   ├── Stall detection (128-sample window)
│   │   ├── Software breakpoints + memory watchpoints
│   │   └── Patch injection queue
│   │
│   └── Closed-Loop Controller (closed_loop_controller.py)
│       ├── DuckStation → Orchestrator → Injection → Verification
│       ├── Golden state expectations
│       ├── 3 clean cycles for BUILD READY
│       └── Pre-verified runtime patches
│
├── Build Orchestration
│   ├── Orchestrator (build_orchestrator.py)
│   │   ├── Historical failure signatures (HF-001+)
│   │   ├── State synthesis (telemetry ↔ history)
│   │   ├── Constraint enforcement (golden_state.json)
│   │   └── Output: OBSERVATION | ROOT CAUSE | VERIFICATION | PATCH
│   │
│   ├── Pipeline Auditor (pipeline_audit.py)
│   │   ├── LBA alignment verification
│   │   ├── Huffman tree bounds checking
│   │   ├── Cross-engine symbol resolution
│   │   ├── EXE patch coordinate verification
│   │   └── Runtime memory coordinate verification
│   │
│   ├── Rulebook (pipeline_rulebook.py)
│   │   ├── 5 Laws enforcement (Zero Blind Commits, Hardware Ground Truth,
│   │   │   Historical Vector Pre-Check, Deterministic C++ Surgery, Pipeline Continuity)
│   │   └── Gate verdict: APPROVED or REJECTED with violations
│   │
│   └── Vector Database (vector_db.py)
│       ├── 7.1 MB cache (vector_db_cache.json)
│       ├── 1.0 MB index (vector_index.json)
│       └── Historical crash signature matching
│
├── Debugging Taxonomy
│   ├── Crashes: DIV, PTR, BIOS, OVERFLOW, VECTOR
│   ├── Freezes: PC_STUCK, TIGHT_LOOP, VBLANK_STALL, EVENT_POLL, CDROM_WAIT
│   ├── Text: OVERFLOW, MISSING_TERM, APOSTROPHE, FULLWIDTH_SPACE, JP_REMNANT
│   ├── VRAM: CORRUPTION, OVERFLOW, BLACK_SCREEN, NO_DMA, VWF_RENDER_ERROR
│   └── Historical: HF-001 (BSS zeros thread), HF-002 (HBD sector corruption), ...
│
└── Critical Blockers
    ├── Blocker A: DW7 EXE cannot parse DQ4 HBD format
    │   ├── Huffman: global tree vs per-block trees
    │   ├── root_id: val & 0x7FFF vs (val & 0x7FFF) + 1
    │   ├── Archive: different folder/file formats
    │   └── Pointers: different LBA references
    │
    └── Blocker B: Custom emulator CD-ROM pipeline incomplete
        ├── Status register: 0x42 instead of 0x03
        ├── DMA ch3: never triggered
        ├── Game never sends Setloc(0x02) or ReadN(0x06)
        └── INT2 second responses / GetID sequence / CD event delivery
```

---

## Appendix A: Key File Reference

| File | Purpose |
|---|---|
| `cybergrime/psx_agent_runner.cpp` | Test harness main entry — VFS hooks, freeze detection, telemetry |
| `cybergrime/psx_emulator_core.cpp` | MIPS R3000A emulator core — CPU, memory, HLE BIOS, CD-ROM, DMA |
| `cybergrime/psx_test_station.h` | PSX hardware model — RAM, VRAM, scratchpad, CD-ROM, DMA, timers, PAD |
| `cybergrime/agent_harness.h` | Agent testing harness — telemetry, triage, freeze detection, assertions |
| `cybergrime/psx_debugger.h` | CLI debugger — breakpoints, watchpoints, step, disasm, hex dump |
| `cybergrime/psx_gdb_stub.h` | GDB remote stub — TCP port 3333, gdb-multiarch compatible |
| `cybergrime/psx_savestate.h` | Save state system — full snapshot/restore |
| `cybergrime/psx_timeline.h` | Event timeline — 17 event types, JSON export |
| `cybergrime/psx_rewind.h` | Rewind system — state history for rollback |
| `cybergrime/psx_lua.h` | Lua scripting engine |
| `cybergrime/instrumentation_bus.h` | Live instrumentation — named-pipe JSON Lines protocol |
| `cybergrime/orchestration_engine.h` | Dynamic MIPS control — hooks, watches, audit, address map |
| `cybergrime/txrt_hle.h` | TXRT HLE bypass — surgical decompression interception |
| `cybergrime/psx_gte.h` | GTE — Geometry Transformation Engine |
| `cybergrime/psx_gpu.h` | GPU — polygon rasterizer, VRAM, CLUT |
| `cybergrime/psx_spu.h` | SPU — 24-voice sound, ADPCM, reverb |
| `cybergrime/psx_mdec.h` | MDEC — motion decoder, IDCT, YUV-to-RGB |
| `cybergrime/psx_xa.h` | XA-ADPCM — CD-XA audio decoding |
| `cybergrime/psx_cdrom_tracer.h` | CD-ROM command tracer |
| `cybergrime/psx_exe_disasm.h` | EXE disassembler with DW7 symbols |
| `cybergrime/regression_runner.py` | Regression test runner — SQLite results |
| `cybergrime/harness_runner.py` | Python orchestration wrapper — single/batch/compare |
| `cybergrime/duckstation_bridge.py` | DuckStation process memory bridge |
| `cybergrime/duckstation_supervisor.py` | DuckStation live supervisor |
| `cybergrime/closed_loop_controller.py` | Circle of Truth closed-loop controller |
| `cybergrime/build_orchestrator.py` | Build orchestrator — historical failures, golden state |
| `cybergrime/pipeline_audit.py` | 5-stage pipeline auditor |
| `cybergrime/pipeline_rulebook.py` | 5 Laws enforcement gate |
| `cybergrime/vector_db.py` | Vector database for crash signature matching |
| `cybergrime/test_definitions.json` | 16 tests, 5 profiles |
| `cybergrime/golden_state.json` | Golden state constraints |
| `cybergrime/constraint_map.json` | Critical address map |
| `translation-tools/frankenstein_builder.py` | Unified build system (193 KB, 7 profiles) |
| `translation-tools/dq4_hbd_patcher.py` | Python HBD patcher |
| `translation-tools/patch_dw7_exe.py` | DW7 EXE surgical patcher |
| `translation-tools/qa_check.py` | QA validation script |
| `ARTIFACT_MANIFEST.md` | Canonical provenance — MD5/SHA256, LBA, file inventory |

---

## Appendix B: Command Reference

```powershell
# Build CyberGrime emulator
cd DQLOSTTRANSLATION\cybergrime
build_agent.bat

# Run regression tests
python regression_runner.py quick_smoke
python regression_runner.py full_regression
python regression_runner.py unit_only
python regression_runner.py v43_closed_loop

# Single test run
psx_agent_runner.exe ..\dq4_frankenstein_v43.bin v43_smoke 5000000 telemetry\v43_smoke.json 60000

# Harness runner (Python wrapper)
python harness_runner.py --disc ..\dq4_frankenstein_v43.bin --profile v43_smoke --max-instrs 5000000

# DuckStation bridge
python duckstation_bridge.py  # Interactive

# DuckStation supervisor
python duckstation_supervisor.py --cue ..\dq4_frankenstein_v43.cue --monitor 60

# Closed-loop controller
python closed_loop_controller.py --cue ..\dq4_frankenstein_v43.cue --max-iterations 10

# Pipeline audit
python pipeline_audit.py --disc ..\dq4_frankenstein_v43.bin --full

# EDC/ECC regeneration
edcre\edcre-v1.1.0-windows-x86_64-static\edcre.exe --repair dq4_frankenstein_vXX.bin

# XDelta3 patch generation
xdelta3.exe -e -s original.bin patched.bin output.xdelta

# GDB connection
gdb-multiarch
(gdb) target remote localhost:3333
(gdb) set architecture mips
```

---

*End of document. This blueprint is the canonical reference for DQLOSTTRANSLATION reverse engineering, QA, and debugging within the `DQLOSTTRANSLATION/` scope.*
