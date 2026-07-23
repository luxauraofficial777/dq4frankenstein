# HLRM Integration Blueprint: CyberGrime Comparative Simulation Environment

**Date:** Jul 19, 2026  
**Author:** Cascade (Systems Architecture)  
**Status:** Specification — Ready for Implementation

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CYBERGRIME HARNESS LOOP                       │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌────────────────────┐     │
│  │ MIPS CPU │───▶│ Trace Ring   │───▶│ Comparator Module  │     │
│  │ (C++     │    │ (64K frames) │    │ (C++ inline)       │     │
│  │  emulator)│   │              │    │                    │     │
│  └────┬─────┘    └──────────────┘    └────────┬───────────┘     │
│       │                                        │                │
│       │    ┌──────────────┐                    │                │
│       └───▶│ VFS / CD-ROM │                    │                │
│            │ Hooks        │                    │                │
│            └──────────────┘                    │                │
│                                               ▼                │
│                                    ┌────────────────────┐      │
│                                    │ Shared Memory IPC  │      │
│                                    │ (mmap or pipe)     │      │
│                                    └────────┬───────────┘      │
│                                             │                  │
└─────────────────────────────────────────────┼──────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PYTHON SIDECAR (HLRM)                          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐     │
│  │ Constraints  │  │ HLRM State   │  │ Divergence Reporter│     │
│  │ Engine       │  │ Simulator    │  │ (JSON → telemetry) │     │
│  │ (static map) │  │ (dynamic)    │  │                    │     │
│  └──────────────┘  └──────────────┘  └────────────────────┘     │
│                                                                  │
│  Input: constraint_map.json (pre-computed)                       │
│  Input: trace_stream (live, from C++ harness)                    │
│  Output: divergence_report.json (structured log)                 │
│  Output: valid_paths.json (C++ heartbeat spec)                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Specifications

### 2.1 Python Constraints Engine (Spec-Generator)

**Purpose:** Consume the DW7 EXE as a static map. Calculate where every expected header, Huffman tree, and memory boundary must be. Output a **Structural Constraint Map** that serves as the source of truth for C++ surgery.

**File:** `cybergrime/constraints_engine.py`

#### Data Structures

```python
# constraint_map.json — Output of the Constraints Engine
{
    "exe_header": {
        "load_address": "0x80017F00",
        "original_file_size": "0x000A4800",   # 673,792 bytes
        "original_pc0": "0x8008E284",
        "max_exe_size": "0x000B3400",         # (HBD_LBA - EXE_LBA) * 2048
        "text_section_end": "0x800BC700"      # load + file_size
    },
    "bss_clear": {
        "start_address": "0x800BC668",
        "end_address": "0x800F4980",
        "instruction_offset": "0x76B84",       # lui $v0, 0x800C
        "instruction_offset_addiu": "0x76B88", # addiu $v0, 0xC668
        "narrow_patch_target": "0x800BCCC8",   # tree end if placed at 0x800BC700
        "constraint": "tree_addr MUST NOT be in [0x800BC668, 0x800F4980) unless BSS is narrowed"
    },
    "huffman_tree": {
        "original_dw7_ram": "0x800EF1C8",
        "original_dw7_lui_imm": "0x800F",
        "original_dw7_addiu_imm": "0xF1C8",
        "hybrid_tree_size": 1480,
        "hybrid_tree_file": "dw7_hybrid_tree.bin",
        "ref_count": 24,                       # lui/addiu pairs to patch
        "constraint": "all 24 refs MUST be patched to same target addr"
    },
    "hbd_layout": {
        "original_dw7_lba": 354,
        "frankenstein_lba": 355,
        "lba_delta": 1,
        "dw7_hbd_size": "0x24D80000",          # 618,563,584 bytes
        "dq4_hbd_disc_start": "sector offset in DQ4 disc",
        "dq4_base_sector": 302115,             # HBD_LBA + DW7_HBD_SECTORS
        "constraint": "all LBA refs 354→355, seq table entries patched, unmatched ABS get +1 delta"
    },
    "disc_check": {
        "patch_count": 11,
        "offsets": ["0x6112C", "0x611D8", "0x613B8", "..."],
        "constraint": "ALL disc-check patches MUST be applied or game hangs at disc identity check"
    },
    "memory_regions": [
        {"name": "EXE_TEXT",    "start": "0x80017F00", "end": "0x800BC700", "type": "loaded"},
        {"name": "BSS_CLEAR",   "start": "0x800BC668", "end": "0x800F4980", "type": "zeroed_at_boot"},
        {"name": "TREE_ORIG",   "start": "0x800EF1C8", "end": "0x800EF788", "type": "runtime_copied"},
        {"name": "STACK",       "start": "0x801FFFF0", "end": "0x80200000", "type": "stack"},
        {"name": "SAFE_HIGH",   "start": "0x80100000", "end": "0x801F0000", "type": "free"}
    ],
    "folder_pointers": {
        "matched_count": 168,
        "unmatched_delta_count": 1755,
        "rel_matched_count": 167,
        "constraint": "matched→DQ4 base sector, unmatched→+1 LBA delta to DW7 data"
    },
    "valid_tree_placements": [
        {"addr": "0x800BC700", "requires_bss_narrow": true,  "risk": "BSS variables unzeroed"},
        {"addr": "0x800F4980", "requires_bss_narrow": false, "requires_copy": true, "risk": "BSS boundary edge"},
        {"addr": "0x80100000", "requires_bss_narrow": false, "requires_copy": true, "risk": "none (safe high RAM)"}
    ]
}
```

#### Constraints Engine API

```python
class ConstraintsEngine:
    """Static analysis of DW7 EXE — produces constraint_map.json."""
    
    def __init__(self, exe_path: str, disc_path: str):
        self.exe = open(exe_path, 'rb').read()
        self.disc_path = disc_path
        
    def build_constraint_map(self) -> dict:
        """Produce the complete structural constraint map."""
        return {
            "exe_header": self._map_exe_header(),
            "bss_clear": self._map_bss_clear(),
            "huffman_tree": self._map_huffman_tree(),
            "hbd_layout": self._map_hbd_layout(),
            "disc_check": self._map_disc_check(),
            "memory_regions": self._map_memory_regions(),
            "folder_pointers": self._map_folder_pointers(),
            "valid_tree_placements": self._enumerate_tree_placements(),
        }
    
    def validate_patch(self, patch_spec: dict) -> list[str]:
        """Check a candidate patch against constraints. Returns list of violations."""
        violations = []
        # Check tree addr not in BSS range (unless BSS narrowed)
        # Check LBA delta consistent
        # Check EXE size doesn't overflow into HBD
        # Check disc-check offsets are valid
        return violations
    
    def _enumerate_tree_placements(self) -> list[dict]:
        """Enumerate all valid tree placement addresses with risk assessment."""
        ...
```

### 2.2 C++ Surgical Tool (Executor)

**Purpose:** Read the constraint map and perform bit-level surgery on the DW7 EXE. This is the existing `psx_binary_ops.cpp` `FrankensteinBuilder::build_v39()`, now driven by the constraint map.

**File:** `cybergrime/psx_binary_ops.cpp` (existing, extended)

#### Constraint Map Loader

```cpp
// New: Load constraint_map.json and drive patching from it
struct ConstraintMap {
    uint32_t load_address;
    uint32_t original_file_size;
    uint32_t original_pc0;
    uint32_t max_exe_size;
    uint32_t bss_start, bss_end;
    uint32_t bss_narrow_target;
    uint32_t tree_original_ram;
    uint16_t tree_original_lui, tree_original_addiu;
    uint32_t tree_size;
    uint32_t tree_ref_count;
    uint32_t hbd_original_lba, hbd_new_lba;
    uint32_t dq4_base_sector;
    // ... disc check offsets, folder mappings, etc.
    
    static ConstraintMap load(const char* json_path);
};

// Modified build function: constraint-driven
BuildResult FrankensteinBuilder::build_from_constraints(
    const char* constraint_map_path,
    const char* dw7_disc_path,
    const char* dw7_exe_path,
    const char* dq4_hbd_disc_path,
    const char* hybrid_tree_path,
    const char* output_bin_path,
    const char* output_cue_path,
    const char* edcre_path,
    int tree_mode  // 0=C, 1=B, 2=A, 3=current
);
```

### 2.3 Harness Synchronization (Live Trace Feed)

**Purpose:** Feed real-time register states and memory dumps from the C++ emulator into the Python HLRM for comparative simulation.

**Mechanism:** Named pipe (`\\.\pipe\cybergrime_trace`) with JSON-line protocol.

#### C++ Side: Trace Emitter

```cpp
// In agent_harness.h — new TraceEmitter class
class TraceEmitter {
    HANDLE m_pipe;
    bool m_connected;
    uint64_t m_emit_interval;  // emit every N instructions
    uint64_t m_last_emit;
    
public:
    bool start(const char* pipe_name, uint64_t emit_interval);
    void emit_state(uint64_t instr_count, uint32_t pc, 
                    const uint32_t regs[32], uint32_t hi, uint32_t lo,
                    uint32_t cp0_status, uint32_t vblank_count);
    void emit_mem(uint32_t addr, uint32_t value, const char* label);
    void emit_vfs(const char* resource, uint32_t lba, bool resolved);
    void stop();
};
```

#### Trace Protocol (JSON Lines, newline-delimited)

```json
{"type":"state","instr":12345678,"pc":"0x8008E284","regs":{"v0":"0x00000001","a0":"0x800B5312","sp":"0x801FFFF0","ra":"0x8009F828"},"hi":"0x0","lo":"0x0","sr":"0x40000000","vblank":42}
{"type":"mem","instr":12345679,"addr":"0x800EF1C8","value":"0x00000000","label":"huffman_tree_root"}
{"type":"vfs","instr":12345680,"resource":"cdrom:sector:355","lba":355,"resolved":true}
{"type":"divergence","instr":12345700,"expected_pc":"0x8008E290","actual_pc":"0x80000000","severity":"FATAL","detail":"Branch to unmapped memory"}
```

#### Python Side: Trace Consumer

```python
# cybergrime/hlrm_sidecar.py
class HLRMSidecar:
    """Live trace consumer + comparative simulator."""
    
    def __init__(self, constraint_map: dict, pipe_name: str):
        self.constraints = constraint_map
        self.pipe_name = pipe_name
        self.expected_state = self._init_expected_state()
        self.divergences = []
        self.valid_paths = []
        
    def run(self):
        """Main loop: read trace lines, compare against expected state."""
        with open(self.pipe_name, 'r') as pipe:
            for line in pipe:
                event = json.loads(line)
                handler = {
                    'state': self._handle_state,
                    'mem': self._handle_mem,
                    'vfs': self._handle_vfs,
                    'divergence': self._handle_divergence,
                }.get(event['type'])
                if handler:
                    handler(event)
    
    def _handle_state(self, event):
        """Compare register state against HLRM expected state."""
        pc = int(event['pc'], 16)
        expected = self.expected_state.get(pc)
        if expected:
            for reg, exp_val in expected.items():
                actual = event['regs'].get(reg, '0x0')
                if actual != exp_val:
                    self._record_divergence(event, reg, exp_val, actual)
        # Track valid execution path
        self.valid_paths.append({
            'instr': event['instr'],
            'pc': event['pc'],
            'regs': event['regs'],
        })
```

### 2.4 Comparator Module (Divergence Detection)

**Purpose:** Run in parallel with the PSX trace. Trigger alerts when simulated output differs from actual MIPS execution.

**Architecture:** The comparator operates at three levels:

#### Level 1: PC-Stream Comparator (Fast, C++ inline)

```cpp
// In psx_emulator_core.cpp — inline check after each instruction
inline void Comparator::check_pc(uint32_t actual_pc, uint64_t instr_count) {
    if (m_expected_pc_queue.empty()) return;
    uint32_t expected_pc = m_expected_pc_queue.front();
    m_expected_pc_queue.pop();
    if (actual_pc != expected_pc) {
        // Divergence! Record and emit
        g_trace_emitter.emit_divergence(instr_count, expected_pc, actual_pc,
                                        "PC_STREAM_MISMATCH");
    }
}
```

#### Level 2: Register State Comparator (Medium, Python sidecar)

```python
def _handle_state(self, event):
    """Compare full register file against expected values."""
    pc = int(event['pc'], 16)
    if pc in self.expected_state:
        expected = self.expected_state[pc]
        for reg in ['v0','v1','a0','a1','a2','a3','t0','t1','ra','sp','gp']:
            actual = event['regs'].get(reg, '0x00000000')
            expected_val = expected.get(reg, None)
            if expected_val is not None and actual != expected_val:
                self._record_divergence(
                    event['instr'], pc, reg, expected_val, actual,
                    severity='WARN' if reg in ['t0','t1','v1'] else 'FAIL'
                )
```

#### Level 3: Memory Constraint Validator (Deep, Python sidecar)

```python
def _handle_mem(self, event):
    """Validate memory accesses against constraint map."""
    addr = int(event['addr'], 16)
    for region in self.constraints['memory_regions']:
        start = int(region['start'], 16)
        end = int(region['end'], 16)
        if start <= addr < end:
            if region['type'] == 'zeroed_at_boot':
                # Check that BSS is actually zeroed (or narrowed)
                if int(event['value'], 16) != 0:
                    self._record_constraint_violation(
                        event['instr'], addr, event['value'],
                        f"Non-zero value in BSS region {region['name']}"
                    )
            elif region['type'] == 'loaded':
                # Check that code section hasn't been corrupted
                pass
            elif region['type'] == 'runtime_copied':
                # Check that tree has been copied here
                if int(event['value'], 16) == 0:
                    self._record_constraint_violation(
                        event['instr'], addr, event['value'],
                        f"Tree not copied to {region['name']} — decompression will fail"
                    )
            break
```

### 2.5 Huffman/LBA Normalization Layer

**Purpose:** Dynamic translation shim within the Python sidecar that performs Huffman tree and LBA pointer re-calculation to "speak English" (DW7 format) for DQ4 data.

```python
class HuffmanLBANormalizer:
    """Translation shim: DQ4 data → DW7-compatible format."""
    
    def __init__(self, hybrid_tree: bytes, lba_delta: int):
        self.tree = HuffTree.parse_dw7(hybrid_tree)
        self.lba_delta = lba_delta
        self.tree_addr = None  # Set from constraint map
        
    def normalize_lba(self, lba: int) -> int:
        """Apply LBA delta: DQ4 sector → Frankenstein sector."""
        return lba + self.lba_delta
    
    def validate_tree_ref(self, lui_imm: int, addiu_imm: int) -> bool:
        """Check that a lui/addiu pair points to the correct tree address."""
        addr = (lui_imm << 16) + sign_extend(addiu_imm)
        return addr == self.tree_addr
        
    def decode_text_block(self, encoded: bytes) -> list[int]:
        """Decode a Huffman-encoded text block using the hybrid tree."""
        return self.tree.decode(encoded)
    
    def encode_text_block(self, leaves: list[int]) -> bytes:
        """Encode leaf values using the hybrid tree."""
        return self.tree.encode(leaves)
    
    def validate_block_roundtrip(self, block_data: bytes) -> dict:
        """Validate that a block can be decoded and re-encoded losslessly."""
        leaves = self.decode_text_block(block_data)
        reencoded = self.encode_text_block(leaves)
        return {
            'leaf_count': len(leaves),
            'original_size': len(block_data),
            'reencoded_size': len(reencoded),
            'roundtrip_ok': reencoded[:len(block_data)] == block_data,
        }
```

### 2.6 Telemetry Pipeline

**Purpose:** Pipe HLRM comparison results back into the existing CyberGrime telemetry dashboard.

#### Existing Infrastructure (Already Built)

- `agent_harness.cpp` → writes `telemetry_<profile>.json` with VFS records, error signatures, register dumps, memory snapshots
- `agent_telemetry.py` → parses telemetry JSON + triage logs into condensed agent brief
- `harness_runner.py` → Python orchestrator with `--compare` mode for diffing two telemetry files
- `vector_node.py` → 2270-address index with crash vector queries

#### New: HLRM Telemetry Extension

```python
# Extension to agent_telemetry.py
def build_hlrm_brief(telemetry: dict, hlrm_report: dict) -> dict:
    """Merge HLRM divergence report into standard telemetry brief."""
    brief = build_agent_brief(telemetry)
    brief['hlrm'] = {
        'divergences': hlrm_report.get('divergences', []),
        'constraint_violations': hlrm_report.get('violations', []),
        'valid_paths': hlrm_report.get('valid_paths', []),
        'tree_placement': hlrm_report.get('tree_placement', {}),
        'huffman_validation': hlrm_report.get('huffman_validation', {}),
    }
    # Add HLRM-specific diagnostic hints
    for div in brief['hlrm']['divergences'][:5]:
        brief['diagnostic_hints'].append(
            f"HLRM: {div['type']} at instr {div['instr']} — "
            f"expected {div.get('expected','?')}, got {div.get('actual','?')}"
        )
    return brief
```

#### Telemetry Dashboard Output

```json
{
    "verdict": "FREEZE_TIGHT_LOOP",
    "profile": "frank_v40a",
    "hlrm": {
        "divergences": [
            {
                "instr": 12345678,
                "type": "MEM_CONSTRAINT_VIOLATION",
                "addr": "0x800BC700",
                "expected": "0x00000000 (BSS zeroed)",
                "actual": "0x2923BE84 (Huffman tree data)",
                "detail": "Tree placed in BSS clear range without narrowing"
            }
        ],
        "tree_placement": {
            "mode": "Option A (0x80100000)",
            "copy_routine_verified": true,
            "tree_refs_patched": 24,
            "bss_narrowed": false
        },
        "huffman_validation": {
            "blocks_checked": 1810,
            "roundtrip_pass": 1648,
            "roundtrip_fail": 162,
            "failed_blocks": ["1666", "1670", "..."]
        }
    }
}
```

---

## 3. Flow Control: End-to-End Pipeline

### Phase 1: Pre-Build (Python Constraints Engine)

```
DW7 EXE ──▶ constraints_engine.py ──▶ constraint_map.json
                                      │
                                      ▼
                              validate_patch() ──▶ violations[] (empty = OK)
                                      │
                                      ▼
                              constraint_map.json ──▶ C++ Surgical Tool
```

### Phase 2: Build (C++ Surgical Tool)

```
constraint_map.json ──┐
DW7 disc ─────────────┼──▶ build_from_constraints() ──▶ frankenstein.bin
DW7 EXE ──────────────┤                                 frankenstein.cue
DQ4 HBD disc ─────────┤
hybrid_tree.bin ──────┘
```

### Phase 3: Test (CyberGrime Harness + HLRM Sidecar)

```
frankenstein.bin ──▶ psx_agent_runner.exe ──▶ trace_pipe ──▶ hlrm_sidecar.py
                         │                                        │
                         ├──▶ telemetry.json                      ├──▶ divergence_report.json
                         ├──▶ triage.log                          ├──▶ valid_paths.json
                         └──▶ screenshot.bmp                      └──▶ hlrm_brief.json
                                  │
                                  ▼
                         agent_telemetry.py + hlrm_report
                                  │
                                  ▼
                         Unified Dashboard (JSON)
```

### Phase 4: Iterate

```
divergence_report.json ──▶ constraints_engine.py (update constraints)
                                │
                                ▼
                         Rebuild with corrected constraints
                                │
                                ▼
                         Re-test → new divergence report
```

---

## 4. C++ Heartbeat Engine Specification

The `valid_paths.json` output from the HLRM sidecar serves as the data-specification for a C++ heartbeat engine — a lightweight inline validator that runs inside the emulator without the Python dependency.

### valid_paths.json Format

```json
{
    "profile": "frank_v40a",
    "tree_mode": 2,
    "expected_checkpoints": [
        {
            "instr_min": 0,
            "instr_max": 100000,
            "pc": "0x800BC700",
            "description": "Copy routine entry (PC0)",
            "required_regs": {"t0": "0x800BC738", "t1": "0x80100000"}
        },
        {
            "instr_min": 100000,
            "instr_max": 200000,
            "pc": "0x8008E284",
            "description": "Original entry point (after copy)",
            "required_regs": {"v0": "0x00000000"}
        },
        {
            "instr_min": 500000,
            "instr_max": 5000000,
            "pc_range": ["0x800BC668", "0x800BC700"],
            "description": "BSS clear loop",
            "required_mem": {"0x800BC700": "0x00000000"}
        },
        {
            "instr_min": 5000000,
            "instr_max": 50000000,
            "pc_range": ["0x80041CE8", "0x80041D00"],
            "description": "Huffman decompression entry",
            "required_mem": {"0x80100000": "non-zero (tree present)"}
        }
    ],
    "forbidden_states": [
        {"pc": "0x00000000", "description": "NULL jump"},
        {"pc_range": ["0xA0000000", "0xA0200000"], "instr_min": 0, "instr_max": 1000000, "description": "BIOS region access too early"},
        {"mem": {"0x800EF1C8": "0x00000000"}, "instr_min": 5000000, "description": "Tree not copied — decompression will fail"}
    ]
}
```

### C++ Heartbeat Engine

```cpp
// cybergrime/heartbeat_engine.h
class HeartbeatEngine {
    struct Checkpoint {
        uint64_t instr_min, instr_max;
        uint32_t pc;           // exact PC or 0 for range
        uint32_t pc_range_start, pc_range_end;
        std::string description;
        std::unordered_map<int, uint32_t> required_regs;  // reg_idx → value
        std::unordered_map<uint32_t, uint32_t> required_mem;
        bool hit;
    };
    
    std::vector<Checkpoint> m_checkpoints;
    std::vector<Checkpoint> m_forbidden;
    size_t m_next_checkpoint;
    
public:
    bool load(const char* json_path);
    void tick(uint64_t instr_count, uint32_t pc, const uint32_t regs[32],
              PSX_Memory& mem);
    bool all_checkpoints_hit() const;
    void report() const;
};
```

---

## 5. Integration with Existing CyberGrime Components

### Existing Components (No Changes Needed)

| Component | File | Role |
|-----------|------|------|
| MIPS CPU emulator | `psx_emulator_core.cpp` | Executes instructions, fills trace ring |
| AgentHarness | `agent_harness.cpp` | TTY capture, freeze detection, JSON telemetry |
| VFS hooks | `psx_agent_runner.cpp` | `hooked_find_cd_file`, `hooked_cd_read` |
| Trace ring | `psx_test_station.h` | 64K-frame ring buffer with full register snapshots |
| Telemetry parser | `agent_telemetry.py` | Parses JSON + triage into agent brief |
| Vector index | `vector_node.py` | 2270-address crash vector database |
| Frankenstein builder | `psx_binary_ops.cpp` | C++ surgical tool (build_v39) |
| Harness runner | `harness_runner.py` | Python orchestrator, batch mode, compare mode |

### New Components to Build

| Component | File | Priority |
|-----------|------|----------|
| Constraints Engine | `cybergrime/constraints_engine.py` | P0 |
| HLRM Sidecar | `cybergrime/hlrm_sidecar.py` | P1 |
| Trace Emitter (C++) | `cybergrime/trace_emitter.h/.cpp` | P1 |
| Heartbeat Engine | `cybergrime/heartbeat_engine.h/.cpp` | P2 |
| Huffman/LBA Normalizer | `cybergrime/hlrm_normalizer.py` | P1 |
| HLRM Telemetry Extension | `agent_telemetry.py` (extend) | P1 |
| Constraint Map Loader (C++) | `psx_binary_ops.cpp` (extend) | P2 |

---

## 6. Implementation Priority

### Sprint 1 (Immediate: Unblocks Black Screen Debug)

1. **Constraints Engine** — Generate `constraint_map.json` from DW7 EXE static analysis. This is pure Python, no emulator dependency. Validates tree placement options A/B/C against memory regions.

2. **Constraint Map Loader** — Extend `build_v39` to read constraint map and validate patch decisions against it before applying.

### Sprint 2 (Next: Comparative Testing)

3. **Trace Emitter** — Add named pipe output to `agent_harness.cpp`. Emit state/mem/vfs events at configurable interval (default: every 10,000 instructions).

4. **HLRM Sidecar** — Python process that reads trace pipe, compares against constraint map, outputs divergence report.

5. **Huffman/LBA Normalizer** — Python module that validates Huffman roundtrip and LBA normalization on processed HBD blocks.

### Sprint 3 (Future: Automated Regression)

6. **Heartbeat Engine** — C++ inline validator that loads `valid_paths.json` and checks checkpoints without Python dependency.

7. **Telemetry Dashboard Extension** — Merge HLRM reports into existing `agent_telemetry.py` brief output.

8. **Automated Iteration Loop** — `harness_runner.py` extension that reads divergence reports, feeds back into constraints engine, triggers rebuild.

---

## 7. Key Constraints (Current State, Jul 19 2026)

| Constraint | Value | Source |
|------------|-------|--------|
| EXE load address | `0x80017F00` | EXE header offset 0x18 |
| EXE original size | `0xA4800` (673,792) | EXE header offset 0x1C |
| Original PC0 | `0x8008E284` | EXE header offset 0x10 |
| BSS clear start | `0x800BC668` | EXE offset 0x76B88: `addiu $v0, 0xC668` |
| BSS clear end | `0x800F4980` | EXE offset 0x76B90: `addiu $v1, 0xF4980` |
| DW7 tree original RAM | `0x800EF1C8` | 24 lui/addiu pairs: `lui 0x800F; addiu 0xF1C8` |
| Hybrid tree size | 1480 bytes | `dw7_hybrid_tree.bin` |
| Tree ref count | 24 pairs | `patch_tree_references()` count |
| HBD original LBA | 354 | ISO directory |
| HBD new LBA | 355 | Shift by 1 to avoid EXE overlap |
| Max EXE before HBD | `(355-24)*2048 = 676,864` | Sector math |
| Stack pointer | `0x801FFFF0` | EXE header offset 0x30 |
| Safe high RAM | `0x80100000`–`0x801F0000` | Above BSS, below stack |
| Disc-check patches | 11 (9 aligned + 2 variable) | `patch_disc_check()` + `patch_disc_check_variable()` |
| Folder pointer matched | 168 | `remap_folder_pointers()` |
| Folder pointer delta | 1755 | Unmatched ABS → +1 LBA shift |

---

## 8. Current Black Screen Diagnosis (Context)

The black screen hang occurs after LBA 146621 read (normal DW7 boot). Three tree placement options are being tested:

- **Option C** (`v40c`): No tree, no BSS patch — game uses own tree at `0x800EF1C8`. Built, pending DuckStation test.
- **Option B** (`v40b`): Tree at `0x800F4980` via copy routine. Built, pending DuckStation test.
- **Option A** (`v40a`): Tree at `0x80100000` via copy routine (safe high RAM). Built, pending DuckStation test.

The HLRM blueprint above will enable automated verification of which option succeeds, with byte-level divergence detection pinpointing the exact failure point.
