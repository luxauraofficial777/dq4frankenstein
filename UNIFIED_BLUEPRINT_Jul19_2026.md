# DQLOSTTRANSLATION — Unified Blueprint: Vector Node + C++ Migration

**Date:** Jul 19, 2026 | **Author:** GLM | **Status:** Blueprint — awaiting execution

---

## Executive Summary

Two synchronization prompts define the next evolution of this project:

1. **Path A — Vector Allocation Node**: Ingest all telemetry, memory maps, and ROM documentation into a structured, queryable vector space indexed by KSEG0 addresses. Enables cross-ROM comparative analysis and crash-to-pointer-dependency queries.

2. **Path B — C/C++ Hot-Path Migration**: Move core binary I/O, patch application, and MIPS pointer calculation from Python to C++ for deterministic memory control. Python remains as orchestrator; C++ handles the binary surgery via ctypes/pybind11.

These two paths are **complementary**: the Vector Node provides the analytical layer to identify what's broken; the C++ migration provides the precision tooling to fix it without Python's abstraction leaks.

---

## Path A: Vector Allocation Node

### A.1 Objective

Transform 50+ telemetry JSON files, 20+ build profiles, and PSX memory map documentation into a unified, queryable index. Every data point is keyed by its `0x80xxxxxx` KSEG0 address and tagged with ROM origin (DQ4_365MB / DW7_711MB).

### A.2 Data Domain Constraints

All ingested data must conform to:

- **Memory Mapping**: Every address indexed using `0x80xxxxxx` KSEG0 convention
- **Structure Identification**: Categorize each data point as one of:
  - `OVERLAY_HEADER` — HBD block headers (24-byte DQ4 / 32-byte DW7)
  - `POINTER_TABLE` — ABS table (EXE 0x4184), REL table (EXE 0x511C), sequential sector table (0xA39AA)
  - `HEAP_ALLOC_LOG` — BIOS malloc/free calls from telemetry BIOS_Call_Log
  - `BIOS_INTERRUPT_HOOK` — CD-ROM intercept, VBlank handler, event descriptors
  - `CODE_CAVE` — Injected trampolines, copy routines, tree relocations
  - `TEXT_BLOCK` — HBD text content blocks (types 23/24/25/27/40/42)
  - `HUFFMAN_TREE` — Per-block (DQ4) or global (DW7) tree nodes
- **ROM Divergence Tag**: Every vector tagged `DQ4_365MB` or `DW7_711MB` or `FRANK_vXX`
- **4-Byte Alignment Verification**: All pointer offsets validated: `offset % 4 == 0`

### A.3 Vectorization Protocol

#### A.3.1 Pointer-to-Base Mapping

For every pointer value found in EXE or HBD:
```
[Pointer_Value] -> [Binary_Base_Address] -> [Resolved_Target] -> [ROM_Tag]
```

Sources to ingest:
- ABS table: 1923 entries at EXE 0x4184 (8-byte entries, max 6000)
- REL table: 168+ entries at EXE 0x511C
- Sequential sector table: 44 entries at EXE 0xA39AA (16-bit)
- 24 MIPS `lui/addiu` pairs referencing global tree at 0x800EF1C8
- 10 disc-check patch sites (EXE offsets 0x6112C, 0x611D8, 0x613B8, etc.)
- CD-ROM Stop intercept at EXE 0x80F64
- BSS clear range: 0x800BC668–0x800F4980
- Copy routine: 14 instructions at 0x800BCCF0
- Hybrid tree: 1480 bytes at 0x800BC700 (on-disc) / 0x80100000 (relocated)

#### A.3.2 Crash/Stall Vector Triples

For every documented crash or stall event in telemetry:
```
[Context: Instruction_Address] -> [Effect: Memory_Violation_Type] -> [Origin: Source_Module]
```

Known crash/stall events to ingest:

| Event | Context (PC) | Effect | Origin | Source |
|-------|-------------|--------|--------|--------|
| v34 freeze | 0x00000058 | VBlank stall, no IRQ delivery | Copy routine ra=0x4 | telemetry_v34_smoke.json |
| v32 crash | 0x800F4F68 | Invalid byte read 0xFFFF8002 | Trampoline corrupted by copy | V32_BUILD_AND_CYBERGRIME_RESULTS |
| v31 crash | 0x800F4F68 | EXE/HBD overlap, tree destroyed | HBD at 354, EXE extends to 354 | BSS_CLEAR_ANALYSIS |
| v33 fail | 0x801005C8 | Invalid byte read, wrong bne offset | Copy routine bne=-4 not -6 | V33_DEBUG_PROGRESS |
| v22 crash | 0x80000000 | ChangeTh returns ra=0x1 | CyberGrime ChangeTh bug | frank_v22_trace.json |
| v38 idle | 0x8009870C | VBlank wait, thread entry zeroed | BSS clears 0x800D9E80 | telemetry_v38b.json |
| DW7 idle | 0x8009674C | Same VBlank wait, same BSS issue | BSS clears 0x800D9E80 | telemetry_baseline_dw7.json |
| v37 hang | LBA 146621 | Boot overlay loads, then GPU idle | Not FMV — overlay calls EXE | SESSION_v37_FMV_SKIP |
| v12 black | — | Unmatched ABS pointers to wrong data | remap_pointers_to_dw7_preserved | GLM_BOOT_FAILURE_ANALYSIS |
| v19 black | — | BSS expansion zeroing data | CD-ROM streaming infinite | DEFINITIVE_BLUEPRINT |

#### A.3.3 BIOS malloc/free Relationship Layer

Map BIOS calls from telemetry to injected hybrid code regions:

```
BIOS_Call (table/fn) -> RAM_Address -> Hybrid_Code_Region -> Collision_Detected?
```

Key BIOS calls to index:
- B:25 (ChangeTh) at ~292K instrs — thread init, ra=0x80096A1C
- B:91 (FlushCache) at ~292K — cache invalidation
- C:10 (ChangeClearRCt) at ~292K — clear rect count
- A:114 (start_card) at ~295K — memory card init
- B:23 (interrupt setup) at ~521K — VBlank/CDROM event registration
- B:53 (DMA transfer) at ~523K — GPU DMA to 0x800F4930
- A:73 (GPU DMA setup) at ~524K — DMA channel config
- B:86/A:68 (timer setup) at ~550K — RCNT mode
- B:8/12 (event setup) at ~564K — F0000009/00000509 event descriptors

Collision zones to flag:
- 0x800F4930 (DMA buffer) — game writes here during init, v32 tree was here
- 0x800F4950, 0x800F4954, 0x800F49A8 — game data section writes
- 0x800D9E80 — BSS clear zeros thread entry, causing idle loop
- 0x800BC668–0x800F4980 — BSS clear range, any injected code here is destroyed
- 0x800EF1C8 — DW7 original global tree (outside loaded EXE, runtime-copied)

### A.4 Telemetry Files to Ingest

| File | ROM Tag | Instructions | Status | Key Data |
|------|---------|-------------|--------|----------|
| telemetry_baseline_dw7.json | DW7_711MB | 5M | COMPLETE | Baseline, VRAM empty, PC=0x8009674C |
| telemetry_baseline_nopreload.json | DW7_711MB | 5M | COMPLETE | No-preload control |
| telemetry_v38a.json | DW7_711MB | 5M | COMPLETE | Disc-check only, identical to baseline |
| telemetry_v38b.json | FRANK_v38b | 5M | COMPLETE | Full engine + shift, identical behavior |
| telemetry_v34_smoke.json | FRANK_v34 | 20.6M | FREEZE | VBlank stall at instr 700K, PC=0x58 |
| telemetry_v32_smoke.json | FRANK_v32 | 5M | COMPLETE | CyberGrime smoke pass, no crash |
| telemetry_v33_smoke.json | FRANK_v33 | 5M | COMPLETE | CyberGrime smoke pass |
| telemetry_v31_smoke.json | FRANK_v31 | 5M | COMPLETE | CyberGrime smoke pass |
| dq4_native_50m.json | DQ4_365MB | 50M | COMPLETE | DQ4 native, 125 VBlanks |
| dq4_native_trace.json | DQ4_365MB | 50M | CRASH | ChangeTh bug at 26.4M |
| dw7_native_trace.json | DW7_711MB | 50M | CRASH | Same ChangeTh bug at 26.4M |
| frank_v22_trace.json | FRANK_v22 | 50M | CRASH | Identical to DW7 native crash |
| telemetry_frank_v14_500m.json | FRANK_v14 | 500M | — | Long-run telemetry |
| v30_gpustat_fix.json | FRANK_v30 | — | — | GPU stat fix run |
| longrun_v30.json | FRANK_v30 | — | — | Long-run v30 |
| frank_diag_*.json (7 files) | FRANK_v14 | — | — | Diagnostic runs |
| telemetry_v33_copy.json | FRANK_v33 | — | — | Copy routine trace |
| telemetry_v33_tramp_debug.json | FRANK_v33 | — | — | Trampoline debug |
| telemetry_v33_wp_only.json | FRANK_v33 | — | — | Write-protect only |

### A.5 Memory Map Documentation to Ingest

#### A.5.1 Proven RAM Layout

| Range | Segment | Source |
|-------|---------|--------|
| 0x80017F00 | EXE load address (BASE) | DEFINITIVE_BLUEPRINT |
| 0x8008E284 | DQ4 BSS clear entry (original PC0) | BSS_CLEAR_ANALYSIS |
| 0x8009674C | VBlank poll loop return | telemetry_baseline_dw7 |
| 0x800986D0 | Outer VBlank wait loop | telemetry_v38b TTY |
| 0x8009870C | VBlank counter check (beq) | telemetry_v38b TTY |
| 0x800B52A0 | Spinlock variable A | telemetry_v38b TTY |
| 0x800B52A4 | Spinlock variable B | telemetry_v38b TTY |
| 0x800B52A8 | Delta computation variable | telemetry_v38b TTY |
| 0x800B63D8 | VBlank count register | telemetry_v38b TTY |
| 0x800BC668 | BSS clear start / GP register | DEFINITIVE_BLUEPRINT |
| 0x800BC700 | Hybrid tree on-disc (within EXE) | DEFINITIVE_BLUEPRINT |
| 0x800BCCF0 | Copy routine (new PC0) | DEFINITIVE_BLUEPRINT |
| 0x800D9E80 | CD-ROM thread entry (ZEROED by BSS) | SESSION_v38_BISECTION |
| 0x800EF1C8 | DW7 original global tree (runtime) | DW7_TREE_CRACK_REPORT |
| 0x800F2C88 | VBlank threshold counter A | telemetry_v38b TTY |
| 0x800F2C8C | VBlank threshold counter B | telemetry_v38b TTY |
| 0x800F4930 | DMA buffer (game writes) | BSS_CLEAR_ANALYSIS |
| 0x800F4950 | Game init write target | BSS_CLEAR_ANALYSIS |
| 0x800F4980 | Game data section start | BSS_CLEAR_ANALYSIS |
| 0x800F9E54 | Game data section end (approx) | BSS_CLEAR_ANALYSIS |
| 0x80100000 | Hybrid tree relocated (safe) | DEFINITIVE_BLUEPRINT |
| 0x801005C8 | CD-ROM Stop trampoline (relocated) | DEFINITIVE_BLUEPRINT |
| 0x80138000 | Dynamic heap start (map geometry) | DEFINITIVE_BLUEPRINT |
| 0x801FFFF0 | Stack top | DEFINITIVE_BLUEPRINT |

#### A.5.2 EXE Patch Sites

| EXE Offset | RAM Address | Patch | Description |
|-----------|-------------|-------|-------------|
| 0x03A94 | 0x8001B894 | LBA 354→355 | Primary HBD LBA reference |
| 0x6112C | 0x8007902C | 0x24020001 | disc_check exit1 return success |
| 0x611D8 | 0x800790D8 | 0x24020001 | disc_check exit2 return success |
| 0x613B8 | 0x800792BC | 0x24020001 | disc_check exit3 return success |
| 0x76B84 | 0x8008E284 | (BSS clear start) | lui $v0, 0x800C |
| 0x76B88 | 0x8008E288 | 0x2442CCC8 | addiu $v0, 0xC668→0xCCC8 (BSS patch) |
| 0x80F64 | 0x80098F64 | j 0x801005C8 | CD-ROM Stop intercept |
| 0x80F68 | 0x80098F68 | nop | Delay slot |
| 0x96FA0 | 0x800AEFA0 | (tree location) | DW7 global Huffman tree (744B) |
| 0xA39AA | 0x800B59AA | 16-bit table | Sequential sector table (44 entries) |
| 0xA5000 | 0x800BC700 | (appended) | Hybrid tree (1480B) + trampoline + copy routine |

#### A.5.3 HBD Block Format Comparison

| Property | DQ4 (365MB) | DW7 (711MB) |
|----------|-------------|-------------|
| HBD LBA | 362 | 354 |
| Header size | 24 bytes | 24 bytes (first 16 identical) |
| hts (header-to-tree) | 24 (no gap) | ~1390 (1366-byte gap) |
| treeEnd | non-zero | 0 |
| textEnd | non-zero | 0 |
| Tree location | After text, per-block | Global in EXE @0x96FA0 |
| Tree format | 2-byte LE, root=(val&0x7FFF)+1 | 2-byte LE, root=val&0x7FFF |
| Text types | 23, 24, 25, 27, 40, 42 | 23, 24, 25, 27 |
| Control codes | 24 (incl. 7F19, 7F1B, 7F2C, 7F2D, 7F32) | 25 (incl. 7F01, 7F05, 7F06, 7F0E, 7F1A, 7F35) |

### A.6 Output: Queryable Vector Space

The indexed vector space supports these query types:

1. **Address Query**: "What is at 0x800D9E80?" → Returns: BSS clear zeros this, CD-ROM thread entry, collision with boot path
2. **Crash Symptom Query**: "Why does GPU stay at 0.00?" → Returns: BSS zeros thread entry → CD-ROM thread never starts → no display data loaded → pointer chain: 0x8008E284 → 0x800D9E80 → VBlank wait at 0x800986D0
3. **Pointer Dependency Query**: "What depends on ABS table entry 901?" → Returns: DW7[901] vs Frank[901] mismatch, remapped to DQ4 data, folder mapping
4. **Cross-ROM Collision Query**: "Where does DQ4 pointer allocation overlap DW7 character subroutine?" → Returns: address conflicts, collision zones, heap boundaries
5. **Patch Impact Query**: "What does patching 0x76B88 affect?" → Returns: BSS clear range change, what memory is no longer zeroed, what previously-zeroed data now survives

### A.7 Implementation

Create `cybergrime/vector_node.py` — a Python module that:
1. Scans all telemetry JSON files in `cybergrime/telemetry/`
2. Parses BIOS call logs, register dumps, TTY traces, error signatures
3. Indexes every address by KSEG0 convention
4. Tags every entry with ROM origin
5. Builds crash vector triples
6. Exposes a query API: `query_address(addr)`, `query_crash(symptom)`, `query_pointer_chain(addr)`, `query_collisions(rom_a, rom_b)`
7. Outputs a JSON index file: `cybergrime/vector_index.json`

---

## Path B: C/C++ Hot-Path Migration

### B.1 Objective

Move the core binary surgery operations from Python to C++ for deterministic memory control, compile-time alignment guarantees, and direct machine code inspection. Python remains as the orchestrator for high-level logic, file selection, and reporting.

### B.2 Current Python Hot Paths to Migrate

| Python Module | Lines | Responsibility | Migration Priority |
|--------------|-------|---------------|-------------------|
| `frankenstein_builder.py` | 3365 | Disc assembly, EXE patching, HBD overlay, EDC/ECC | **P0** (core) |
| `reencode_dq4_blocks.py` | ~300 | HBD text block re-encoding (Huffman) | **P0** (blocker) |
| `patch_dw7_exe.py` | ~600 | Standalone EXE patcher (LBA, tree, disc-check) | **P1** (subset of builder) |
| `cybergrime/psx/exe.py` | 181 | PS-X EXE parser, code cave finder, injector | **P1** |
| `cybergrime/psx/hbd.py` | 183 | HBD archive parser, folder remapper | **P1** |
| `cybergrime/psx/iso.py` | 71 | ISO 9660 sector I/O | **P2** (simple, low risk) |
| `cybergrime/mips/assembler.py` | 11286 | MIPS R3000A assembler | **P2** (already have C++ disasm) |
| `build_hybrid.py` | 65755 | Translation build, Huffman encoding | **P3** (data processing, not binary surgery) |
| `dq4_hbd_patcher.py` | 25330 | HBD text patching | **P1** (binary surgery) |

### B.3 C++ Architecture

#### B.3.1 Core Library: `psx_binary_ops.h`

```cpp
#pragma once
#include <cstdint>
#include <cstring>
#include <vector>
#include <fstream>

// ===== PSX Memory Map Structs (byte-exact to hardware) =====

struct PsxExeHeader {
    char     signature[8];    // "PS-X EXE"
    uint8_t  padding[8];      // 0x08-0x0F
    uint32_t pc0;             // 0x10 — initial PC
    uint32_t gp0;             // 0x14 — global pointer
    uint32_t dest_addr;       // 0x18 — load destination
    uint32_t text_size;       // 0x1C — text section size
    uint32_t data_addr;       // 0x20
    uint32_t data_size;       // 0x24
    uint32_t bss_addr;        // 0x28
    uint32_t bss_size;        // 0x2C
    uint32_t stack_addr;      // 0x30 — initial SP
    uint32_t stack_size;      // 0x34
    // 0x38-0x7F: region info, reserved
};
static_assert(sizeof(PsxExeHeader) == 0x80, "EXE header must be 2048 bytes total");

struct HbdBlockHeader {
    uint32_t end_marker;      // Block end sentinel
    uint32_t block_id;        // Block identifier
    uint32_t hts;             // Header-to-tree offset
    uint32_t reserved1;       // 0
    uint32_t tree_end;        // DQ4: non-zero, DW7: 0
    uint32_t text_end;        // DQ4: non-zero, DW7: 0
    uint32_t unknown;         // Format-specific
};
static_assert(sizeof(HbdBlockHeader) == 24, "HBD block header must be 24 bytes");

struct AbsTableEntry {
    uint32_t folder_id;       // Folder identifier
    uint32_t sector_offset;   // Absolute sector offset
};
static_assert(sizeof(AbsTableEntry) == 8, "ABS table entry must be 8 bytes");

struct RelTableEntry {
    uint32_t folder_id;
    uint32_t relative_offset;
};
static_assert(sizeof(RelTableEntry) == 8, "REL table entry must be 8 bytes");

// ===== MIPS Instruction Encoding (compile-time verified) =====

constexpr uint32_t MIPS_JAL(uint32_t target) {
    return (0x03 << 26) | ((target >> 2) & 0x03FFFFFF);
}
constexpr uint32_t MIPS_J(uint32_t target) {
    return (0x02 << 26) | ((target >> 2) & 0x03FFFFFF);
}
constexpr uint32_t MIPS_LUI(uint8_t rt, uint16_t imm) {
    return (0x0F << 26) | (rt << 16) | imm;
}
constexpr uint32_t MIPS_ADDIU(uint8_t rt, uint8_t rs, uint16_t imm) {
    return (0x09 << 26) | (rs << 21) | (rt << 16) | imm;
}
constexpr uint32_t MIPS_LW(uint8_t rt, uint16_t offset, uint8_t base) {
    return (0x23 << 26) | (base << 21) | (rt << 16) | offset;
}
constexpr uint32_t MIPS_SW(uint8_t rt, uint16_t offset, uint8_t base) {
    return (0x2B << 26) | (base << 21) | (rt << 16) | offset;
}
constexpr uint32_t MIPS_BNE(uint8_t rs, uint8_t rt, int16_t offset) {
    return (0x05 << 26) | (rs << 21) | (rt << 16) | (uint16_t)offset;
}
constexpr uint32_t MIPS_NOP = 0x00000000;
constexpr uint32_t MIPS_JR_RA = 0x03E00008;

// ===== ISO Sector I/O (deterministic, no Python overhead) =====

class IsoImage {
    std::fstream file;
    uint32_t sector_size;  // 2048 or 2352
    uint32_t data_offset;  // 0 or 24 for MODE2 Form1

public:
    bool open(const char* path);
    void close();
    int read_sector(uint32_t lba, void* buf_2048);
    int write_sector(uint32_t lba, const void* buf_2048);
    int read_range(uint32_t lba, uint32_t count, void* buf);
    uint32_t file_size() const;
};

// ===== EXE Patcher (type-safe, alignment-guaranteed) =====

class ExePatcher {
    uint8_t* exe_data;
    uint32_t exe_size;
    PsxExeHeader* header;
    uint32_t load_addr;

public:
    bool load(uint8_t* data, uint32_t size);
    uint32_t ram_to_offset(uint32_t ram_addr) const;
    uint32_t read32(uint32_t ram_addr) const;
    void write32(uint32_t ram_addr, uint32_t value);
    uint16_t read16(uint32_t ram_addr) const;
    void write16(uint32_t ram_addr, uint16_t value);
    // Patch operations
    uint32_t patch_lba_references(uint32_t old_lba, uint32_t new_lba);
    uint32_t patch_sequential_sector_table(uint32_t old_lba, uint32_t new_lba);
    uint32_t patch_tree_references(uint32_t old_addr, uint32_t new_addr);
    uint32_t patch_disc_check(const uint32_t* patches, uint32_t count);
    void append_payload(const uint8_t* data, uint32_t size);
    // Verification
    bool verify_alignment() const;  // Check all pointer offsets % 4 == 0
};

// ===== HBD Re-encoder (the current blocker) =====

class HbdReencoder {
    uint8_t* hbd_data;
    uint32_t hbd_size;
    const uint8_t* hybrid_tree;  // 1480 bytes
    uint32_t tree_size;

public:
    bool load(uint8_t* data, uint32_t size, const uint8_t* tree, uint32_t tree_sz);
    int reencode_block(uint32_t block_offset, const char* english_text, uint32_t budget);
    int handle_zero_length_tree(uint32_t block_offset);  // Write minimal DW7-format block
    bool verify_roundtrip(uint32_t block_offset);
};
```

#### B.3.2 Python Interface: `psx_ops.py`

```python
"""Python wrapper for psx_binary_ops C++ library via ctypes."""
import ctypes
import os

_lib = ctypes.CDLL(os.path.join(os.path.dirname(__file__), "psx_ops.dll"))

# ... ctypes bindings for IsoImage, ExePatcher, HbdReencoder ...
```

### B.4 Migration Strategy: Hybrid Approach

#### Phase B-1: Core C++ Library (Week 1)

**Goal**: Build `psx_ops.dll` with the three critical classes.

1. Create `cybergrime/psx_binary_ops.h` — structs + class declarations
2. Create `cybergrime/psx_binary_ops.cpp` — implementations
3. Create `cybergrime/psx_ops.py` — ctypes wrapper
4. Build: `cl /O2 /LD psx_binary_ops.cpp /Fe:psx_ops.dll`

**Deliverables**:
- `IsoImage` class replaces `cybergrime/psx/iso.py` (71 lines → C++)
- `ExePatcher` class replaces core of `frankenstein_builder.py` patch functions (~800 lines → C++)
- `HbdReencoder` class replaces `reencode_dq4_blocks.py` (~300 lines → C++)

**Verification**: Round-trip test — read EXE with C++, patch same values, compare output byte-for-byte with Python builder output.

#### Phase B-2: Integrate with Python Orchestrator (Week 2)

**Goal**: `frankenstein_builder.py` calls C++ for binary operations, Python handles profile selection, file management, EDC/ECC invocation, reporting.

1. Refactor `frankenstein_builder.py`:
   - Replace `struct.pack_into`/`struct.unpack_from` calls with `ExePatcher` C++ calls
   - Replace sector I/O with `IsoImage` C++ calls
   - Replace HBD re-encoding with `HbdReencoder` C++ calls
   - Keep Python for: profile selection, path management, edcre.exe subprocess, JSON config, logging
2. All 20+ build profiles continue to work identically
3. Add `--use-cpp` flag to builder (default: yes, `--no-cpp` falls back to Python)

**Deliverables**:
- `frankenstein_builder.py` slimmed from 3365 to ~800 lines (orchestration only)
- `psx_ops.dll` handles all binary I/O
- All existing build profiles produce byte-identical output

#### Phase B-3: Fix the Blockers with C++ Precision (Week 3)

**Goal**: Use the C++ tooling to fix the 5 open blockers with deterministic precision.

1. **HBD Re-encoder zero-length tree fix**: `HbdReencoder::handle_zero_length_tree()` writes a minimal DW7-format block with control codes only — no Python guessing about tree topology
2. **BSS clear / thread entry fix**: `ExePatcher::patch_bss_range()` with compile-time-verified MIPS encoding for the post-BSS-clear injection (Option B from v38 bisection)
3. **FMV skip**: `ExePatcher::patch_function_prologue()` with `jr $ra; nop` — compile-time-verified instruction encoding
4. **7 missing blocks**: `HbdReencoder` extracts JP text, re-encodes with hybrid tree, inserts into HBD
5. **Disc extension sectors**: `IsoImage::write_mode2_sector()` guarantees valid sync+header+subheader+EDC/ECC at compile time

#### Phase B-4: Verification & Golden Path (Week 4)

1. Rebuild v36 with C++ builder
2. Run all 3 verify scripts (`verify_e2e_jul19*.py`)
3. CyberGrime smoke test (5M instructions)
4. DuckStation test (title screen check)
5. Golden path automation (`.devin/workflows/golden-path.md`)
6. Victory checkpoint verification

### B.5 Existing C++ Infrastructure to Leverage

| File | Lines | What it provides |
|------|-------|-----------------|
| `cybergrime/psx_emulator_core.cpp` | 255K | Full MIPS R3000A core, HLE BIOS, CD-ROM, DMA, GTE |
| `cybergrime/psx_gpu.h` | 539 | GPU rasterizer, texel modulation, semi-transparency |
| `cybergrime/psx_gte.h` | 32K | All 38 GTE instructions |
| `cybergrime/psx_reference_integrations.h` | 23K | BIOS syscall tables, CD-ROM constants, MIPS disassembler |
| `cybergrime/psx_debugger.h` | 15K | Debugger with breakpoints, memory watch |
| `cybergrime/psx_savestate.h` | 7.5K | Save state serialization |
| `cybergrime/agent_harness.h` | 39K | Agent harness with telemetry, assertions, input macros |
| `cybergrime/orchestration_engine.h` | 13K | Test orchestration, regression management |

The C++ migration builds on this existing codebase — we're not starting from scratch.

---

## Parallel Agent Architecture

Both paths run **simultaneously** as two independent agents. They share a read-only contract (telemetry files + memory map constants) but never write to each other's files. A synchronization point occurs only at the merge phase.

### Agent Assignments

| Agent | Name | Track | Workspace | Output Files |
|-------|------|-------|-----------|-------------|
| **Agent-VA** | Vector Analyst | Path A — Vector Node | `cybergrime/vector_node.py`, `cybergrime/vector_index.json` | Vector index, query API, crash analysis reports |
| **Agent-CB** | Codebase Builder | Path B — C++ Migration | `cybergrime/psx_binary_ops.{h,cpp}`, `cybergrime/psx_ops.py`, `cybergrime/psx_ops.dll` | C++ library, ctypes wrapper, patched builder |

### Shared Contract (Read-Only — Both Agents Can Read)

Both agents operate on the same read-only data. Neither agent modifies these files:

```
SHARED_READ_ONLY:
  - cybergrime/telemetry/*.json          (50+ telemetry files)
  - translation/SLUSP012.06              (DW7 EXE — 675,840 bytes)
  - translation/HBD1PS1D.Q41            (DQ4 HBD — 319,436,800 bytes)
  - dw7_hybrid_tree.bin                  (1,480 bytes)
  - dw7_encoded_translation.json         (15,318 entries)
  - translation/full_translation.json    (15,318 entries)
  - study/*.md                           (all documentation)
  - translation-tools/frankenstein_builder.py  (reference — Agent-CB reads, does NOT modify until Phase 2)
```

### File Ownership (No Overlap)

```
AGENT-VA OWNS (only Agent-VA writes):
  - cybergrime/vector_node.py
  - cybergrime/vector_index.json
  - cybergrime/vector_queries.py        (query helper module)
  - study/VECTOR_ANALYSIS_*.md          (analysis reports)

AGENT-CB OWNS (only Agent-CB writes):
  - cybergrime/psx_binary_ops.h
  - cybergrime/psx_binary_ops.cpp
  - cybergrime/psx_ops.py
  - cybergrime/psx_ops.dll
  - cybergrime/psx_ops_build.bat        (build script)
  - translation-tools/frankenstein_builder_v2.py  (refactored builder — NEW file, does NOT touch original until merge)
  - translation-tools/reencode_dq4_blocks_v2.cpp  (C++ re-encoder)
```

### Parallel Execution Timeline

```
Time    Agent-VA (Vector Analyst)              Agent-CB (Codebase Builder)
----    ---------------------------             ------------------------------
T0      || INGEST: Scan all telemetry JSON      || CREATE: psx_binary_ops.h
        || Parse BIOS_Call_Log from each file   || Define PsxExeHeader, HbdBlockHeader
        || Index addresses by KSEG0             || Define AbsTableEntry, RelTableEntry
        || Tag entries DQ4/DW7/FRANK            || constexpr MIPS instruction encoders
        ||                                      ||
T1      || BUILD: Crash vector triples          || IMPLEMENT: psx_binary_ops.cpp
        || Map 10 known crash/stall events      || IsoImage class (sector I/O)
        || Build BIOS call relationship layer   || ExePatcher class (read/write/patch)
        || Flag collision zones                 || HbdReencoder class (stubs)
        ||                                      ||
T2      || QUERY API: vector_node.py            || COMPILE: psx_ops.dll
        || query_address(addr)                  || Build via cl /O2 /LD
        || query_crash(symptom)                 || Create psx_ops.py ctypes wrapper
        || query_pointer_chain(addr)            || Round-trip test: C++ vs Python output
        || query_collisions(rom_a, rom_b)       || Byte-for-byte comparison
        ||                                      ||
T3      || ANALYZE: Query BSS/thread collision  || FIX: Implement HbdReencoder fully
        || Confirm 0x800D9E80 root cause        || Handle zero-length trees
        || Map full pointer dependency chain    || Implement BSS thread-entry fix
        || Identify any secondary collisions    || (post-BSS-clear injection)
        || Output: VECTOR_ANALYSIS_REPORT.md    ||
        ||                                      ||
        || ====== SYNC POINT ======             || ====== SYNC POINT ======
        ||                                      ||
T4      || MERGE: Agent-VA delivers analysis    || MERGE: Agent-CB receives analysis
        || to Agent-CB                          || Applies any additional patches
        || Vector Node ready for queries        || identified by Vector analysis
        ||                                      ||
T5      || VERIFY: Query post-patch telemetry   || BUILD: Rebuild v36 with C++ builder
        || Compare before/after vectors         || Run verify_e2e_jul19*.py
        || Confirm collision resolved           || CyberGrime smoke test (5M instrs)
        ||                                      || DuckStation title screen test
        ||                                      ||
T6      || REPORT: Final vector diff            || RELEASE: Golden path QA
        || Document all resolved collisions     || FMV skip (if needed)
        || Archive vector index                 || XDelta patch + README
```

### Synchronization Protocol

The two agents operate fully independently from T0-T3. At **T3->T4 SYNC POINT**:

1. **Agent-VA** delivers `study/VECTOR_ANALYSIS_REPORT.md` containing:
   - Confirmed collision zones (BSS vs thread entry, tree placement, etc.)
   - Full pointer dependency chains for each crash event
   - Any NEW collisions discovered during analysis that Agent-CB didn't anticipate
   - Recommended patch sites (EXE offsets + expected values)

2. **Agent-CB** receives the report and:
   - Cross-references Agent-VA's findings against its C++ patch implementations
   - Applies any additional patches identified by the vector analysis
   - Uses `vector_node.py` query API for runtime verification post-build

3. **After sync**, both agents continue in parallel:
   - Agent-VA: queries post-patch telemetry to verify collisions resolved
   - Agent-CB: builds, tests, and releases

### Inter-Agent Communication Contract

Agents communicate exclusively through **files on disk** -- no shared memory, no sockets, no IPC:

```
AGENT-VA -> AGENT-CB:
  study/VECTOR_ANALYSIS_REPORT.md     (T3->T4 sync)
  cybergrime/vector_index.json        (queryable at any time by Agent-CB)
  cybergrime/vector_node.py           (importable by Agent-CB)

AGENT-CB -> AGENT-VA:
  cybergrime/telemetry/telemetry_v39.json  (new telemetry from rebuilt disc)
  dq4_frankenstein_v39.bin/.cue            (new disc for Agent-VA to analyze)
```

### Guard Rails (Conflict Prevention)

| Rule | Enforcement |
|------|-------------|
| Agent-VA never writes to `translation-tools/` | File ownership boundary |
| Agent-CB never writes to `cybergrime/vector_*` | File ownership boundary |
| Neither agent modifies `frankenstein_builder.py` until T4 merge | Agent-CB creates `frankenstein_builder_v2.py` instead |
| Neither agent modifies telemetry files | Read-only shared contract |
| Shared constants (EXE_LBA, HBD_LBA, etc.) are locked | Defined in blueprint appendix; both agents hardcode same values |
| If both agents need the same binary file (e.g., DW7 EXE) | Both read independently; no write contention since neither writes to source ROMs |

### Parallel Architecture Diagram

```
         +---------------------------------------------------+
         |              SHARED READ-ONLY DATA                 |
         |  telemetry/*.json | SLUSP012.06 | HBD1PS1D.Q41   |
         |  hybrid_tree.bin | encoded_translation.json       |
         |  study/*.md | frankenstein_builder.py (ref)       |
         +-----------+-----------------------+---------------+
                     |                       |
    +----------------v----------+  +----------v----------------+
    |    AGENT-VA (Vector)      |  |    AGENT-CB (Codebase)     |
    |                           |  |                            |
    |  T0: Ingest telemetry     |  |  T0: Create C++ headers    |
    |  T1: Build crash vectors  |  |  T1: Implement classes     |
    |  T2: Query API            |  |  T2: Compile + test DLL    |
    |  T3: Analyze collisions   |  |  T3: Fix blockers in C++   |
    |                           |  |                            |
    |  ====== SYNC POINT ====== |  |  ====== SYNC POINT ====== |
    |  T4: Deliver analysis ------>  T4: Receive analysis      |
    |                           |  |      Apply extra patches   |
    |  T5: Verify post-patch    |  |  T5: Build + smoke test    |
    |  T6: Final report         |  |  T6: Golden path + release |
    |                           |  |                            |
    |  OUTPUTS:                 |  |  OUTPUTS:                  |
    |  vector_node.py           |  |  psx_binary_ops.h/.cpp     |
    |  vector_index.json        |  |  psx_ops.dll/.py           |
    |  VECTOR_ANALYSIS_*.md     |  |  frankenstein_builder_v2.py|
    +---------------------------+  +----------------------------+
                     |                       |
    +----------------v-----------------------v-----------------+
    |              VERIFICATION LAYER                          |
    |  CyberGrime smoke -> DuckStation visual -> Golden Path   |
    +----------------------------------------------------------+
```

---

## Execution Priority (Parallel Tracks)

### Track A: Agent-VA (Vector Analyst)

| Phase | Task | Effort | Depends On |
|-------|------|--------|------------|
| **VA-1** | Ingest all 50+ telemetry JSON files, parse BIOS calls, index addresses | 1 session | Nothing |
| **VA-2** | Build crash vector triples (10 known events), BIOS relationship layer, collision zones | 1 session | VA-1 |
| **VA-3** | Implement query API (`query_address`, `query_crash`, `query_pointer_chain`, `query_collisions`) | 0.5 session | VA-2 |
| **VA-4** | Run analysis: confirm BSS/thread-entry collision, map full dependency chain, identify secondary collisions | 0.5 session | VA-3 |
| **VA-5** | **SYNC**: Deliver `VECTOR_ANALYSIS_REPORT.md` to Agent-CB | -- | VA-4, Agent-CB at CB-4 |
| **VA-6** | Post-patch verification: query new telemetry, confirm collisions resolved, final vector diff report | 0.5 session | Agent-CB at CB-6 |

**Agent-VA total: 3.5 sessions** (runs in parallel with Agent-CB)

### Track B: Agent-CB (Codebase Builder)

| Phase | Task | Effort | Depends On |
|-------|------|--------|------------|
| **CB-1** | Create `psx_binary_ops.h` -- structs, constexpr MIPS encoders, class declarations | 0.5 session | Nothing |
| **CB-2** | Implement `psx_binary_ops.cpp` -- IsoImage, ExePatcher, HbdReencoder (stubs) | 1 session | CB-1 |
| **CB-3** | Compile `psx_ops.dll`, create `psx_ops.py` ctypes wrapper, round-trip test | 0.5 session | CB-2 |
| **CB-4** | Implement HbdReencoder fully (zero-tree fix), BSS thread-entry fix (post-BSS injection) | 1.5 sessions | CB-3 |
| **CB-5** | **SYNC**: Receive `VECTOR_ANALYSIS_REPORT.md`, apply any additional patches | 0.5 session | CB-4, Agent-VA at VA-5 |
| **CB-6** | Refactor `frankenstein_builder.py` -> `frankenstein_builder_v2.py` with C++ calls, rebuild v36 | 1 session | CB-5 |
| **CB-7** | Verify: `verify_e2e_jul19*.py`, CyberGrime smoke, DuckStation title screen | 0.5 session | CB-6 |
| **CB-8** | Translate 7 missing blocks, FMV skip (if needed), Golden path QA, XDelta release | 2 sessions | CB-7 |

**Agent-CB total: 7 sessions** (runs in parallel with Agent-VA until sync point)

### Combined Timeline

```
Session:  1       2       3       4       5       6       7
          |       |       |       |       |       |       |
Agent-VA: VA-1   VA-2   VA-3,4  VA-5----------------------VA-6
          |       |       |       |                       |
Agent-CB: CB-1   CB-2   CB-3   CB-4   CB-5,6  CB-7  CB-8
          |       |       |       |       |       |     |
                              SYNC POINT
                          (Session 4)
```

- **Sessions 1-3**: Full parallel -- no dependencies between tracks
- **Session 4**: SYNC POINT -- Agent-VA delivers analysis, Agent-CB receives
- **Sessions 5-7**: Agent-CB continues build/test/release; Agent-VA does post-patch verification

**Total wall-clock time: 7 sessions** (down from 8-9 sequential)

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| C++ migration introduces new bugs | Byte-for-byte comparison with Python output; `--no-cpp` fallback flag |
| Vector Node index becomes stale | Auto-regenerate from telemetry directory; timestamp-based invalidation |
| BSS fix breaks other initialization | Use v34's proven copy-routine pattern (post-BSS injection); CyberGrime smoke test before DuckStation |
| HBD re-encoder still fails on edge cases | Fallback: write minimal control-code-only block; log and continue |
| FMV skip still wrong target | Only attempt after logo renders; use Vector Node to trace actual code path |

---

## File Manifest

### New files to create:
- `cybergrime/vector_node.py` — Vector Allocation Node indexer and query API
- `cybergrime/vector_index.json` — Generated index (auto-updated)
- `cybergrime/psx_binary_ops.h` — C++ structs and class declarations
- `cybergrime/psx_binary_ops.cpp` — C++ implementations
- `cybergrime/psx_ops.py` — Python ctypes wrapper
- `cybergrime/psx_ops.dll` — Compiled C++ library

### Existing files to modify:
- `translation-tools/frankenstein_builder.py` — Refactor to call C++ via ctypes
- `translation-tools/reencode_dq4_blocks.py` — Replace with C++ HbdReencoder calls

### Existing files to preserve (reference only):
- `cybergrime/psx/exe.py` — Python EXE parser (kept as fallback)
- `cybergrime/psx/hbd.py` — Python HBD parser (kept as fallback)
- `cybergrime/psx/iso.py` — Python ISO I/O (kept as fallback)
- All telemetry JSON files in `cybergrime/telemetry/` — Ingested by Vector Node
- All study/*.md documentation — Ingested as memory map reference

---

## Appendix: Key Constants (Locked, do not change)

```
EXE_LBA          = 24
HBD_LBA          = 355  (shifted from 354)
DQ4_HBD_SIZE     = 319,436,800 bytes (155,975 sectors)
DW7_HBD_SIZE     = 618,563,584 bytes (302,033 sectors)
HYBRID_TREE_SIZE = 1,480 bytes (370 leaves)
EXE_EXTENDED     = 677,888 bytes (331 sectors)
ABS_TABLE_START  = 0x4184 (EXE offset)
REL_TABLE_START  = 0x511C (EXE offset)
TREE_RAM_RELOC   = 0x80100000
TRAMPOLINE_RAM   = 0x801005C8
COPY_ROUTINE_RAM = 0x800BCCF0
ORIGINAL_PC0     = 0x8008E284
BSS_CLEAR_START  = 0x800BC668
BSS_CLEAR_END    = 0x800F4980
THREAD_ENTRY     = 0x800D9E80 (zeroed by BSS — ROOT CAUSE of idle)
```
