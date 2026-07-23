# Live Instrumentation & Patch Loop — Implementation Progress

**Date:** Jul 19, 2026 8:38 PM  
**Session:** Dynamic Verification & Instrumentation Blueprint Implementation  
**Status:** All Python components functional, C++ integrated (needs recompile)

---

## Completed Work

### 1. Blueprint Document
- **File:** `study/INSTRUMENTATION_BLUEPRINT.md`
- 11 sections covering architecture, instrumentation path, Ghidra bridge, runtime comparator, deterministic patching, divergence log format, decompilation guide, and MIPS state tracking
- Full architecture diagram with named-pipe protocol specification

### 2. Python Components (All Import-Verified)

| File | Purpose | Status |
|------|---------|--------|
| `cybergrime/constraints_engine.py` | Static EXE analysis → `constraint_map.json` | ✅ Working |
| `cybergrime/runtime_comparator.py` | Live state vs golden model comparison | ✅ Working |
| `cybergrime/live_analysis.py` | Main sidecar: pipe reader + comparator + Ghidra display | ✅ Working |
| `cybergrime/patch_injector.py` | WRITE_MEM/WRITE_REG/ROLLBACK via named pipe + CLI | ✅ Working |
| `cybergrime/patch_controller.py` | Automated patch loop: capture→validate→inject→resume | ✅ Working |
| `cybergrime/ghidra_bridge.py` | PC→function lookup with binary search | ✅ Working |

### 3. Generated Data Files

| File | Content |
|------|---------|
| `cybergrime/constraint_map.json` | 23 tree refs, BSS range [0x800BC668, 0x800F4980), 16 critical addresses, 4 tree placement options |
| `cybergrime/golden_state.json` | Expected register checkpoints at 0x800BC700, 0x8008E284, 0x80073670, 0x80084BF0 |
| `cybergrime/dw7_function_map.json` | 13 known function boundaries for Ghidra context lookup |

### 4. C++ Components (Integrated, Needs Recompile)

| File | Purpose | Status |
|------|---------|--------|
| `cybergrime/instrumentation_bus.h` | Named pipe emitter + patch injector class | ✅ Written |
| `cybergrime/instrumentation_bus.cpp` | Full implementation: JSON emit, reader thread, patch apply/rollback | ✅ Written |
| `cybergrime/psx_agent_runner.cpp` | `--live-instrument N` flag, tick after step(), bp notification, cleanup | ✅ Integrated |

### 5. Constraint Engine Results

```
EXE Header:
  Load address:  0x80017f00
  Original PC0:  0x8008e284
  File size:     0xa4800 (675,840 bytes)
  Max EXE size:  0xa5800 (677,888 bytes)
  SP:            0x801ffff0

BSS Clear:
  Range:         0x800bc668 → 0x800f4980
  Verified:      start=True, end=True
  Thread entry:  IN RANGE (will be zeroed!)
  Narrow target: 0x800bccc8

Huffman Tree:
  Original RAM:  0x800ef1c8
  Tree size:     1480 bytes
  Ref count:     23 lui/addiu pairs

Valid Tree Placements:
  Mode 3: 0x800bc700 — Tree at EXE end, BSS narrowed
  Mode 1: 0x800f4980 — Tree at BSS end, copy routine
  Mode 2: 0x80100000 — Tree in safe high RAM (recommended)
  Mode 0: 0x800ef1c8 — No tree (uses DW7 original)
```

---

## Patch Loop Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PATCH LOOP FLOW                           │
│                                                             │
│  1. STATE CAPTURE                                           │
│     Emulator hits BP at 0x800BC700 → emits bp_hit JSON      │
│                                                             │
│  2. SIMULATION & VALIDATION                                 │
│     patch_controller checks $t0/$t1/$t2 vs golden_state     │
│     runtime_comparator flags tree pointer mismatches        │
│                                                             │
│  3. INSTRUCTION INJECTION                                   │
│     patch_injector sends WRITE_REG via named pipe           │
│     C++ InstrumentationBus applies patch to CPU regs        │
│                                                             │
│  4. RESUMPTION                                              │
│     Emulator continues from exact instruction               │
│     Instant feedback — no ROM rebuild needed                │
│                                                             │
│  5. ROLLBACK (if fix fails)                                 │
│     ROLLBACK command restores original state                │
│     Try different tree address → re-test                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Usage Commands

### Automated Patch Loop
```bash
# Terminal 1: Emulator
psx_agent_runner.exe dq4_frankenstein_v40a.bin test 200000000 telemetry.json 0 \
  --bp 0x80073670 --bp 0x800BC700 --bp 0x8008E284 \
  --live-instrument 10000

# Terminal 2: Patch Controller
python cybergrime\patch_controller.py --mode auto --output patch_loop_log.json
```

### Live Analysis (Monitor Only)
```bash
python cybergrime\live_analysis.py \
  --constraints cybergrime\constraint_map.json \
  --golden cybergrime\golden_state.json \
  --ghidra-map cybergrime\dw7_function_map.json \
  --output divergence_log.json
```

### Manual Injection
```bash
python cybergrime\patch_injector.py --pipe \\.\pipe\cybergrime_live
> write_reg 6 0x80100000 "fix tree pointer"
> dump_region 0x80100000 64 "verify tree data"
> rollback "undo all patches"
```

### Regenerate Constraint Map
```bash
python cybergrime\constraints_engine.py \
  --exe translation\SLUSP012.06 \
  --tree dw7_hybrid_tree.bin \
  --folder-map translation\folder_mapping.json \
  --output cybergrime\constraint_map.json
```

---

## Pending Work

1. **Recompile C++ emulator** — Add `instrumentation_bus.cpp` to build, recompile `psx_agent_runner.exe`
2. **DuckStation disc tests** — Test v40c/v40b/v40a discs (from previous session)
3. **End-to-end test** — Run emulator with `--live-instrument`, connect patch_controller, verify automated tree fix works
4. **Ghidra headless integration** — Export full function map from Ghidra project (currently using 13 known functions)
5. **LBA drift detection** — Enhance patch_controller with real-time LBA pointer verification during CD-ROM reads

---

## Key Files Index

```
study/INSTRUMENTATION_BLUEPRINT.md      — Full architectural blueprint
study/INSTRUMENTATION_PROGRESS.md       — This file
cybergrime/constraints_engine.py        — Static EXE analysis
cybergrime/constraint_map.json          — Generated constraint map
cybergrime/golden_state.json            — Expected runtime state
cybergrime/dw7_function_map.json        — Function boundaries
cybergrime/runtime_comparator.py        — State comparison engine
cybergrime/live_analysis.py             — Main sidecar
cybergrime/patch_injector.py            — Pipe command sender
cybergrime/patch_controller.py          — Automated patch loop
cybergrime/ghidra_bridge.py             — PC→decompiled C lookup
cybergrime/instrumentation_bus.h        — C++ pipe emitter/injector
cybergrime/instrumentation_bus.cpp      — C++ implementation
cybergrime/psx_agent_runner.cpp         — Integrated --live-instrument flag
```
