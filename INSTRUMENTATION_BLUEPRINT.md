# Live Dynamic Verification & Instrumentation Blueprint

**Date:** Jul 19, 2026  
**Author:** Cascade (Systems Architecture)  
**Status:** Specification — Ready for Implementation  
**Depends on:** HLRM_INTEGRATION_BLUEPRINT.md (Sprint 1 constraints engine)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                     CYBERGRIME EMULATOR (C++)                        │
│                                                                      │
│  ┌───────────┐   ┌───────────────┐   ┌──────────────────────┐       │
│  │ MIPS CPU  │──▶│ Trace Ring    │──▶│ Instrumentation Bus  │       │
│  │ Core      │   │ (64K frames)  │   │ (new)                │       │
│  └─────┬─────┘   └───────────────┘   └──────────┬───────────┘       │
│        │                                         │                   │
│  ┌─────▼─────┐   ┌───────────────┐               │                   │
│  │ PSX Mem   │──▶│ Watchpoint    │───────────────┤                   │
│  │ (RAM/VRAM)│   │ Engine        │               │                   │
│  └───────────┘   └───────────────┘               │                   │
│                                                  │                   │
│  ┌──────────────────────────────────────┐        │                   │
│  │ Live Patch Injector (new)            │◀───────┤                   │
│  │ • In-memory EXE writes               │        │                   │
│  │ • Breakpoint-suspended patching      │        │                   │
│  │ • Rollback on crash                  │        │                   │
│  └──────────────────────────────────────┘        │                   │
│                                                  │                   │
└──────────────────────────────────────────────────┼───────────────────┘
                                                   │
                    ┌──────────────────────────────┼───────────────┐
                    │         INSTRUMENTATION BUS (JSON Lines)      │
                    │         Named pipe: \\.\pipe\cybergrime_live  │
                    └──────────────────────────────┼───────────────┘
                                                   │
                    ┌──────────────────────────────▼───────────────┐
                    │           PYTHON LIVE ANALYSIS SIDEAR         │
                    │                                               │
                    │  ┌─────────┐  ┌──────────┐  ┌────────────┐  │
                    │  │ Ghidra  │  │ Runtime  │  │ Divergence │  │
                    │  │ Bridge  │  │ Comparator│  │ Logger     │  │
                    │  │ (MCP)   │  │          │  │            │  │
                    │  └─────────┘  └──────────┘  └────────────┘  │
                    │                                               │
                    │  ┌──────────────────────────────────────┐    │
                    │  │ Golden Model (constraint_map.json)    │    │
                    │  └──────────────────────────────────────┘    │
                    │                                               │
                    │  ┌──────────────────────────────────────┐    │
                    │  │ Patch Injector (JSON commands → pipe) │    │
                    │  └──────────────────────────────────────┘    │
                    └───────────────────────────────────────────────┘
```

---

## 2. Live Instrumentation Path

### 2.1 Existing Infrastructure (Already Built)

The CyberGrime emulator already has these instrumentation hooks:

| Component | File | What It Does |
|-----------|------|-------------|
| Trace Ring | `psx_test_station.h:127-185` | 64K-frame ring buffer, each frame has full 32-register snapshot + PC + raw instruction + disasm |
| Freeze Detection | `agent_harness.cpp:428-464` | PC-stuck, tight-loop, and VBlank-stall detection |
| VFS Hooks | `psx_agent_runner.cpp:51-84` | `hooked_find_cd_file`, `hooked_cd_read`, `hooked_read_cd_sector` |
| Watchpoints | `psx_agent_runner.cpp:173-176` | `mem.add_watchpoint(addr, mask, read, write)` |
| Monitors | `psx_agent_runner.cpp:196-198` | `harness.add_monitor(addr, size, type, ...)` — value change, overflow, nonzero |
| Breakpoints | `psx_agent_runner.cpp:163-171` | `cpu.add_breakpoint(addr)` + `--bp-halt` for hard stop |
| Register Dump | `agent_harness.h:94-103` | `RegisterSnapshot` struct with all 32 GPRs + CP0 |
| Memory Dump | `agent_harness.h:106-110` | `MemoryDumpRecord` with 256-byte capture |
| TTY Capture | `agent_harness.cpp:172-211` | 1MB pipe-redirection buffer for stdout |
| JSON Telemetry | `agent_harness.h:548` | `write_json()` — full state to telemetry JSON |
| Orchestration | `orchestration_engine.h` | Auto-correct jumps, redirect PC, skip instructions |
| Huffman Trace | `psx_agent_runner.cpp:287-307` | `--huffman-trace` flag: breakpoints at 0x80073670, 0x80084BF0, monitors on tree root + text cache |
| Render Trace | `psx_agent_runner.cpp:308-352` | `--render-trace` flag: breakpoints at GPU dispatch, font table monitors |

### 2.2 New: Instrumentation Bus (C++ Side)

**File:** `cybergrime/instrumentation_bus.h` / `.cpp`

```cpp
// Named-pipe JSON-line emitter for live state streaming
class InstrumentationBus {
    HANDLE m_pipe;
    bool m_connected;
    bool m_enabled;
    uint64_t m_emit_interval;      // emit state every N instructions
    uint64_t m_last_emit_instr;
    std::mutex m_write_mutex;
    
    // Patch injection (reverse direction: Python → C++)
    std::queue<PatchCommand> m_pending_patches;
    std::mutex m_patch_mutex;
    bool m_patch_thread_running;
    std::thread m_patch_reader;
    
public:
    // ── Lifecycle ──
    bool start(const char* pipe_name, uint64_t emit_interval);
    void stop();
    
    // ── Emit (C++ → Python) ──
    void emit_state(uint64_t instr_count, uint32_t pc,
                    const uint32_t regs[32], uint32_t hi, uint32_t lo,
                    uint32_t cp0_status, uint32_t vblank_count);
    void emit_mem_access(uint32_t addr, uint32_t value, bool is_write,
                         uint32_t pc, uint64_t instr_count);
    void emit_vfs(const char* resource, uint32_t lba, bool resolved,
                  uint32_t pc, uint64_t instr_count);
    void emit_divergence(uint64_t instr_count, uint32_t pc,
                         const char* expected, const char* actual,
                         const char* detail, const char* severity);
    void emit_breakpoint_hit(uint32_t bp_addr, uint64_t instr_count,
                             const uint32_t regs[32]);
    
    // ── Inject (Python → C++) ──
    bool has_pending_patches();
    PatchCommand pop_patch();
    void apply_patch(MIPS_CPU& cpu, PSX_Memory& mem);
    
    // ── Called from emulator main loop ──
    void tick(uint64_t instr_count, uint32_t pc, MIPS_CPU& cpu, PSX_Memory& mem);
};

// Patch command from Python sidecar
struct PatchCommand {
    enum Type { WRITE_MEM, WRITE_REG, SET_PC, SET_BREAKPOINT, 
                REMOVE_BREAKPOINT, DUMP_REGION, ROLLBACK };
    Type type;
    uint32_t addr;       // target address (RAM or register index)
    uint32_t value;      // value to write
    uint32_t size;       // for DUMP_REGION: bytes to dump
    char description[128];
};
```

### 2.3 Integration Point in Emulator Loop

In `psx_emulator_core.cpp`, the main execution loop already calls `check_freeze()` and `trace_ring_push()`. We add the instrumentation bus tick:

```cpp
// In psx_emulator_core.cpp — main instruction loop (simplified)
void MIPS_CPU::step(PSX_Memory& mem) {
    uint32_t instr = read32(pc, mem_ctx);
    // ... execute instruction ...
    
    // Existing: trace ring capture
    if (instr != 0 || is_branch) {
        trace_ring_push(cur_pc, instr, disasm_str, false, false);
    }
    
    // NEW: instrumentation bus tick
    if (g_instr_bus && g_instr_bus->is_enabled()) {
        g_instr_bus->tick(instructions, pc, *this, mem);
        
        // Check for pending patches from Python sidecar
        while (g_instr_bus->has_pending_patches()) {
            g_instr_bus->apply_patch(*this, mem);
        }
    }
    
    // Existing: freeze detection
    if (g_harness && g_harness->check_freeze(pc, instructions)) {
        // ... freeze handling ...
    }
}
```

### 2.4 CLI Flag

```cpp
// In psx_agent_runner.cpp — new CLI argument
} else if (strcmp(argv[i], "--live-instrument") == 0) {
    if (i + 1 >= argc) break;
    uint64_t interval = strtoull(argv[i + 1], nullptr, 10);
    g_instr_bus = new InstrumentationBus();
    g_instr_bus->start("\\\\.\\pipe\\cybergrime_live", interval);
    harness.tty_printf("[INSTR] Live instrumentation enabled (interval=%llu)\n", interval);
    i++;
}
```

---

## 3. Dynamic Trace-to-Decompile Mapping

### 3.1 Ghidra MCP Bridge

**Purpose:** When the emulator hits a breakpoint or divergence, automatically query Ghidra for the decompiled C function at the current PC.

**File:** `cybergrime/ghidra_bridge.py`

```python
import json
import subprocess
import os
from typing import Optional, Dict, Any

class GhidraBridge:
    """Bridge between live emulator PC and Ghidra decompiled output."""
    
    def __init__(self, ghidra_project_path: str, ghidra_headless_path: str):
        self.project = ghidra_project_path
        self.headless = ghidra_headless_path
        self._cache = {}  # PC → decompiled function (LRU cache)
        self._function_map = {}  # function entry → (start, end, name)
        
    def load_function_map(self, map_path: str):
        """Load pre-exported function boundary map from Ghidra.
        
        This is generated once by running Ghidra's auto-analysis on the
        DW7 EXE and exporting the function list as JSON.
        """
        with open(map_path) as f:
            self._function_map = json.load(f)
    
    def query_pc(self, ram_addr: int) -> Optional[Dict[str, Any]]:
        """Given a RAM address, return the containing function's decompiled C.
        
        Returns:
            {
                "function_name": "FUN_80073670",
                "entry": "0x80073670",
                "end": "0x80073800",
                "decompiled": "void FUN_80073670(...) { ... }",
                "disasm_at_pc": "lw $t3, 0($t0)",
                "source_line": 42
            }
        """
        # Find containing function
        func = self._find_function(ram_addr)
        if not func:
            return None
        
        # Check cache
        if func['entry'] in self._cache:
            result = self._cache[func['entry']].copy()
            result['offset_in_function'] = ram_addr - func['entry']
            return result
        
        # Query Ghidra headless for decompiled output
        decompiled = self._ghidra_decompile(func['entry'])
        result = {
            'function_name': func['name'],
            'entry': func['entry'],
            'end': func['end'],
            'decompiled': decompiled,
            'offset_in_function': ram_addr - func['entry'],
        }
        self._cache[func['entry']] = result
        return result
    
    def _find_function(self, ram_addr: int) -> Optional[dict]:
        """Binary search the function map for containing function."""
        for func in self._function_map:
            start = int(func['entry'], 16)
            end = int(func['end'], 16)
            if start <= ram_addr < end:
                return func
        return None
    
    def _ghidra_decompile(self, entry_hex: str) -> str:
        """Call Ghidra headless analyzer for decompiled C of function at entry."""
        # Uses Ghidra's postScript to export decompiled C
        script = f"""
        // Ghidra postScript: decompile_function.py
        from ghidra.app.decompiler import DecompInterface
        from ghidra.util.task import ConsoleTaskMonitor
        
        addr = toAddr("{entry_hex}")
        func = getFunctionAt(addr)
        if func is None:
            func = createFunction(addr, None)
        if func:
            decomp = DecompInterface()
            decomp.openProgram(currentProgram)
            result = decomp.decompileFunction(func, 30, ConsoleTaskMonitor())
            if result.decompileCompleted():
                print("===DECOMPILE_START===")
                print(result.getDecompiledFunction().getC())
                print("===DECOMPILE_END===")
        """
        # Run headless Ghidra
        output = subprocess.run([
            self.headless, self.project, "-process", "SLUSP012.06",
            "-postScript", "decompile_function.py",
            "-scriptPath", os.path.dirname(script)
        ], capture_output=True, text=True, timeout=30)
        
        # Extract decompiled C between markers
        text = output.stdout
        start = text.find("===DECOMPILE_START===")
        end = text.find("===DECOMPILE_END===")
        if start >= 0 and end >= 0:
            return text[start + len("===DECOMPILE_START==="):end].strip()
        return "// Decompilation failed"
```

### 3.2 Pre-Exported Function Map

Instead of running Ghidra headless on every query (slow), we pre-export the function map once:

```json
// dw7_function_map.json — Generated by Ghidra auto-analysis
[
    {"entry": "0x80017F00", "end": "0x80018000", "name": "_start"},
    {"entry": "0x80041CE8", "end": "0x80041E00", "name": "huffman_decompress_inline"},
    {"entry": "0x80073670", "end": "0x800738A0", "name": "huffman_decompress_primary"},
    {"entry": "0x80084BF0", "end": "0x80084D00", "name": "huffman_decompress_secondary"},
    {"entry": "0x800562C0", "end": "0x80056400", "name": "hbd_subblock_parser"},
    {"entry": "0x8008E284", "end": "0x8008E300", "name": "bss_clear_entry"},
    {"entry": "0x80098898", "end": "0x80098900", "name": "cdrom_cmd_handler"},
    {"entry": "0x800D9E80", "end": "0x800DA000", "name": "cdrom_thread_entry"},
    {"entry": "0x8009F790", "end": "0x8009F900", "name": "gpu_display_setup"},
    {"entry": "0x8009F828", "end": "0x8009F830", "name": "gpu_cw_call_site"},
    {"entry": "0x800A5DA0", "end": "0x800A5E00", "name": "gpu_cw_trampoline"}
]
```

### 3.3 Live Context Display

When the emulator hits a breakpoint, the Python sidecar queries Ghidra and displays:

```
┌─ BREAKPOINT HIT ────────────────────────────────────────────────────┐
│ PC: 0x80073670  Instruction: 123,456,789                            │
│ Function: huffman_decompress_primary (0x80073670 - 0x800738A0)     │
│ Offset in function: 0x0 (entry)                                     │
│                                                                     │
│ Decompiled C:                                                       │
│   void huffman_decompress_primary(uint8_t* src, uint8_t* dst,       │
│                                    uint16_t* tree, int tree_size) { │
│       int node = tree_root;                                         │
│       while (1) {                                                   │
│           int bit = *src >> (bit_pos++ & 7) & 1;                   │
│           node = tree[node + bit];                                  │
│           if (!(node & 0x8000)) {                                   │
│               *dst++ = node & 0xFF;                                 │
│               *dst++ = (node >> 8) & 0xFF;                          │
│               node = tree_root;                                     │
│           }                                                         │
│       }                                                             │
│   }                                                                 │
│                                                                     │
│ MIPS at PC:                                                         │
│   0x80073670: lui $gp, 0x800F    # $gp = tree segment base          │
│                                                                     │
│ Registers:                                                          │
│   $a0 = 0x80120040  (src: encoded text buffer)                     │
│   $a1 = 0x80120100  (dst: decoded output buffer)                   │
│   $a2 = 0x800EF1C8  (tree: Huffman tree pointer)                   │
│   $a3 = 0x000005C8  (tree_size: 1480 bytes)                        │
│   $ra = 0x80072500  (return to caller)                             │
│                                                                     │
│ Memory at $a2 (tree root):                                          │
│   0x800EF1C8: 00 00 00 00  ← EXPECTED: hybrid tree data            │
│   ⚠ DIVERGENCE: Tree is zeroed! BSS clear wiped it.                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Verification Loop: Runtime Comparator

### 4.1 Golden Model

The Golden Model is the `constraint_map.json` from the HLRM blueprint, extended with runtime expectations:

```python
# cybergrime/runtime_comparator.py

class RuntimeComparator:
    """Compares live emulator state against the Golden Model."""
    
    def __init__(self, constraint_map: dict, golden_state: dict):
        self.constraints = constraint_map
        self.golden = golden_state  # Expected runtime state at checkpoints
        self.divergences = []
        self.checkpoints_hit = set()
        
    def check_memory_region(self, addr: int, actual_value: int,
                           instr_count: int, pc: int) -> list:
        """Check a memory access against constraint map."""
        divergences = []
        
        for region in self.constraints['memory_regions']:
            start = int(region['start'], 16)
            end = int(region['end'], 16)
            if not (start <= addr < end):
                continue
            
            rtype = region['type']
            name = region['name']
            
            if rtype == 'zeroed_at_boot':
                # BSS region — should be zero unless tree placed here
                tree_addr = self.golden.get('tree_placement', {}).get('addr')
                tree_size = self.constraints['huffman_tree']['hybrid_tree_size']
                if tree_addr and int(tree_addr, 16) <= addr < int(tree_addr, 16) + tree_size:
                    # Tree is here — check it's not zero
                    if actual_value == 0 and instr_count > 500000:
                        divergences.append(self._div(
                            instr_count, pc, 'TREE_ZEROED_IN_BSS',
                            addr, 'non-zero (tree data)', '0x00000000',
                            f"Tree at {hex(addr)} was zeroed by BSS clear"
                        ))
                elif actual_value != 0 and instr_count < 100000:
                    # BSS not yet cleared — normal
                    pass
                elif actual_value != 0 and instr_count > 200000:
                    # BSS should have cleared this
                    divergences.append(self._div(
                        instr_count, pc, 'BSS_NOT_ZEROED',
                        addr, '0x00000000', hex(actual_value),
                        f"BSS region {name} not zeroed at instr {instr_count}"
                    ))
            
            elif rtype == 'runtime_copied':
                # Tree should be copied here by now
                if actual_value == 0 and instr_count > 500000:
                    divergences.append(self._div(
                        instr_count, pc, 'TREE_NOT_COPIED',
                        addr, 'non-zero (tree data)', '0x00000000',
                        f"Tree not copied to {name} — decompression will fail"
                    ))
            
            break
        
        self.divergences.extend(divergences)
        return divergences
    
    def check_register(self, reg_name: str, actual: int,
                       instr_count: int, pc: int) -> list:
        """Check a register value against golden state for this PC."""
        expected = self.golden.get('register_checkpoints', {}).get(pc, {}).get(reg_name)
        if expected is not None and actual != expected:
            div = self._div(
                instr_count, pc, 'REGISTER_MISMATCH',
                reg_name, hex(expected), hex(actual),
                f"${reg_name} expected {hex(expected)} but got {hex(actual)}"
            )
            self.divergences.append(div)
            return [div]
        return []
    
    def check_huffman_decode(self, tree_addr: int, encoded_bytes: bytes,
                            expected_leaves: list, instr_count: int, pc: int) -> list:
        """Verify Huffman decoding produces expected leaf values."""
        # Read tree from emulator memory (via instrumentation bus)
        # Decode locally using Python HuffTree
        # Compare against expected leaves
        divergences = []
        if not expected_leaves:
            return divergences
        
        # The actual check happens when we receive a mem dump of the tree
        # and the encoded bytes from the emulator
        # For now, just log the decode attempt
        return divergences
    
    def _div(self, instr, pc, dtype, addr, expected, actual, detail):
        return {
            'instr_count': instr,
            'pc': hex(pc),
            'type': dtype,
            'address': hex(addr) if isinstance(addr, int) else addr,
            'expected': expected,
            'actual': actual,
            'detail': detail,
            'timestamp': time.time(),
        }
```

### 4.2 Golden State Checkpoints

```json
// golden_state.json — Expected state at specific instruction counts
{
    "tree_placement": {
        "mode": "Option A",
        "addr": "0x80100000",
        "copy_routine_pc0": "0x800BC700"
    },
    "register_checkpoints": {
        "0x800BC700": {
            "t0": "0x800BC738",
            "t1": "0x80100000",
            "t2": "0x800BC738",
            "_description": "Copy routine entry: src=tree, dst=high RAM"
        },
        "0x8008E284": {
            "v0": "0x00000000",
            "_description": "Original PC0 after copy: BSS clear entry"
        },
        "0x80073670": {
            "a2": "0x80100000",
            "_description": "Huffman decompress: tree pointer should be relocated addr"
        }
    },
    "memory_checkpoints": [
        {
            "instr_min": 500000,
            "addr": "0x80100000",
            "expected": "non-zero",
            "description": "Tree must be copied to high RAM before first decompression"
        },
        {
            "instr_min": 100000,
            "addr": "0x800D9E80",
            "expected": "non-zero",
            "description": "Thread entry must not be zeroed by BSS clear"
        },
        {
            "instr_min": 5000000,
            "addr": "0x800EF1C8",
            "expected": "0x00000000",
            "description": "Original tree location should be zeroed (we use relocated tree)"
        }
    ],
    "huffman_checkpoints": [
        {
            "pc": "0x80073670",
            "tree_reg": "a2",
            "src_reg": "a0",
            "dst_reg": "a1",
            "description": "Primary decompression entry — verify tree pointer matches placement"
        },
        {
            "pc": "0x80084BF0",
            "tree_reg": "a2",
            "description": "Secondary decompression entry"
        }
    ]
}
```

---

## 5. Deterministic Patching via Injection

### 5.1 Live Patch Injector

**Purpose:** Test patches in emulator memory before burning them to ROM. The emulator suspends at a breakpoint, the Python sidecar sends a patch command, and execution resumes with the patch applied.

```python
# cybergrime/patch_injector.py

class PatchInjector:
    """Send patch commands to the emulator via instrumentation bus."""
    
    def __init__(self, pipe_name: str):
        self.pipe = open(pipe_name, 'w', buffering=1)  # Line-buffered
    
    def write_mem(self, ram_addr: int, value: int, size: int = 4,
                  description: str = ""):
        """Write value to RAM address in emulator memory."""
        cmd = {
            'type': 'WRITE_MEM',
            'addr': hex(ram_addr),
            'value': hex(value),
            'size': size,
            'description': description
        }
        self.pipe.write(json.dumps(cmd) + '\n')
        self.pipe.flush()
    
    def write_reg(self, reg_idx: int, value: int, description: str = ""):
        """Write value to CPU register (0-31)."""
        cmd = {
            'type': 'WRITE_REG',
            'addr': reg_idx,
            'value': hex(value),
            'description': description
        }
        self.pipe.write(json.dumps(cmd) + '\n')
        self.pipe.flush()
    
    def set_pc(self, pc: int, description: str = ""):
        """Set program counter."""
        cmd = {
            'type': 'SET_PC',
            'addr': hex(pc),
            'description': description
        }
        self.pipe.write(json.dumps(cmd) + '\n')
        self.pipe.flush()
    
    def set_breakpoint(self, addr: int, description: str = ""):
        """Add a breakpoint."""
        cmd = {
            'type': 'SET_BREAKPOINT',
            'addr': hex(addr),
            'description': description
        }
        self.pipe.write(json.dumps(cmd) + '\n')
        self.pipe.flush()
    
    def dump_region(self, addr: int, size: int, description: str = ""):
        """Request a memory dump from the emulator."""
        cmd = {
            'type': 'DUMP_REGION',
            'addr': hex(addr),
            'size': size,
            'description': description
        }
        self.pipe.write(json.dumps(cmd) + '\n')
        self.pipe.flush()
    
    def rollback(self, description: str = ""):
        """Rollback all injected patches (restore original memory)."""
        cmd = {
            'type': 'ROLLBACK',
            'description': description
        }
        self.pipe.write(json.dumps(cmd) + '\n')
        self.pipe.flush()
    
    # ── High-level patch helpers ──
    
    def patch_lui_addiu(self, exe_offset: int, new_lui: int, new_addiu: int,
                        description: str = ""):
        """Patch a lui/addiu pair in EXE memory (loaded at 0x80017F00)."""
        ram_addr = 0x80017F00 + exe_offset
        lui_instr = 0x3C000000 | (new_lui & 0xFFFF)  # lui $reg, imm
        addiu_instr = 0x24000000 | (new_addiu & 0xFFFF)  # addiu $reg, $reg, imm
        self.write_mem(ram_addr, lui_instr, 4, f"lui patch: {description}")
        self.write_mem(ram_addr + 4, addiu_instr, 4, f"addiu patch: {description}")
    
    def patch_tree_refs(self, new_tree_addr: int, old_lui: int = 0x800F,
                        old_addiu: int = 0xF1C8):
        """Patch all 24 lui/addiu tree reference pairs to point to new addr."""
        new_lui = (new_tree_addr >> 16) & 0xFFFF
        new_addiu = new_tree_addr & 0xFFFF
        if new_addiu >= 0x8000:
            new_lui += 1  # Sign extension correction
        
        # Scan EXE memory for all lui/addiu pairs matching old values
        # This would be done via a DUMP_REGION command first, then
        # patch each match individually
        pass
    
    def inject_copy_routine(self, ram_addr: int, src: int, dst: int,
                            size: int, jump_to: int):
        """Inject a MIPS copy routine at ram_addr in emulator memory."""
        src_hi = (src >> 16) & 0xFFFF
        src_lo = src & 0xFFFF
        dst_hi = (dst >> 16) & 0xFFFF
        dst_lo = dst & 0xFFFF
        if dst_lo >= 0x8000: dst_hi += 1
        if src_lo >= 0x8000: src_hi += 1
        
        instructions = [
            0x3C080000 | src_hi,       # lui $t0, src_hi
            0x25080000 | (src_lo & 0xFFFF),  # addiu $t0, $t0, src_lo
            0x3C090000 | dst_hi,       # lui $t1, dst_hi
            0x25290000 | (dst_lo & 0xFFFF),  # addiu $t1, $t1, dst_lo
            0x250A0000 | (size & 0xFFFF),    # addiu $t2, $t0, size
            0x8D0B0000,                # lw $t3, 0($t0)
            0x00000000,                # nop
            0xAD2B0000,                # sw $t3, 0($t1)
            0x25080004,                # addiu $t0, $t0, 4
            0x25290004,                # addiu $t1, $t1, 4
            0x150AFFFA,                # bne $t0, $t2, -6
            0x00000000,                # nop
            0x08000000 | (jump_to >> 2),  # j jump_to
            0x00000000,                # nop
        ]
        
        for i, insn in enumerate(instructions):
            self.write_mem(ram_addr + i * 4, insn, 4,
                          f"copy_routine[{i}] at {hex(ram_addr)}")
```

### 5.2 C++ Patch Application

```cpp
// In instrumentation_bus.cpp
void InstrumentationBus::apply_patch(MIPS_CPU& cpu, PSX_Memory& mem) {
    PatchCommand cmd = pop_patch();
    
    switch (cmd.type) {
        case PatchCommand::WRITE_MEM: {
            // Save original for rollback
            uint32_t orig = 0;
            for (int i = 0; i < cmd.size; i++) {
                orig |= (mem.read8(cmd.addr + i) & 0xFF) << (i * 8);
            }
            m_patch_history.push_back({cmd.addr, orig, cmd.size});
            
            // Write new value
            for (int i = 0; i < cmd.size; i++) {
                mem.write8(cmd.addr + i, (cmd.value >> (i * 8)) & 0xFF);
            }
            tty_printf("[INJECT] WRITE_MEM 0x%08X = 0x%08X (%s)\n",
                       cmd.addr, cmd.value, cmd.description);
            break;
        }
        case PatchCommand::WRITE_REG: {
            int reg = (int)cmd.addr;
            uint32_t orig = cpu.get_reg(reg);
            m_reg_history.push_back({reg, orig});
            cpu.regs[reg] = cmd.value;
            tty_printf("[INJECT] WRITE_REG $%d = 0x%08X (%s)\n",
                       reg, cmd.value, cmd.description);
            break;
        }
        case PatchCommand::SET_PC: {
            tty_printf("[INJECT] SET_PC 0x%08X (%s)\n",
                       cmd.addr, cmd.description);
            cpu.pc = cmd.addr;
            cpu.pending_branch = false;
            cpu.in_branch_delay = false;
            break;
        }
        case PatchCommand::SET_BREAKPOINT: {
            cpu.add_breakpoint(cmd.addr);
            tty_printf("[INJECT] SET_BREAKPOINT 0x%08X (%s)\n",
                       cmd.addr, cmd.description);
            break;
        }
        case PatchCommand::DUMP_REGION: {
            // Emit memory contents back through the pipe
            for (uint32_t i = 0; i < cmd.size; i += 4) {
                uint32_t val = mem.read32(cmd.addr + i);
                emit_mem_access(cmd.addr + i, val, false, cpu.pc, cpu.instructions);
            }
            tty_printf("[INJECT] DUMP_REGION 0x%08X +%u bytes\n",
                       cmd.addr, cmd.size);
            break;
        }
        case PatchCommand::ROLLBACK: {
            // Restore all patched memory
            for (auto& h : m_patch_history) {
                for (int i = 0; i < h.size; i++) {
                    mem.write8(h.addr + i, (h.orig >> (i * 8)) & 0xFF);
                }
            }
            // Restore all patched registers
            for (auto& r : m_reg_history) {
                cpu.regs[r.reg_idx] = r.orig;
            }
            m_patch_history.clear();
            m_reg_history.clear();
            tty_printf("[INJECT] ROLLBACK: restored %zu mem patches, %zu reg patches\n",
                       m_patch_history.size(), m_reg_history.size());
            break;
        }
    }
}
```

### 5.3 Injection Workflow Example

```
1. Start emulator with breakpoints at critical addresses:
   psx_agent_runner.exe dq4_frankenstein_v40a.bin test_v40a 200000000 telemetry.json 0 \
     --bp 0x80073670 --bp 0x800BC700 --bp 0x8008E284 \
     --live-instrument 10000

2. Python sidecar starts, loads constraint_map.json + golden_state.json

3. Emulator hits breakpoint at 0x800BC700 (copy routine entry):
   → Emitter sends: {"type":"breakpoint_hit","pc":"0x800BC700","instr":1234,...}
   → Python queries Ghidra: "What function is at 0x800BC700?"
   → Ghidra returns: "copy_routine (injected code)"
   → Python checks golden state: t0 should be 0x800BC738, t1 should be 0x80100000
   → If mismatch → inject correction: write_reg(t0, 0x800BC738)
   → Resume execution

4. Emulator hits breakpoint at 0x80073670 (Huffman decompression):
   → Emitter sends register state
   → Python checks: $a2 (tree pointer) should be 0x80100000
   → If $a2 == 0x800EF1C8 (original, not patched) → DIVERGENCE!
   → Python injects: write_reg(a2, 0x80100000) — test if tree at high RAM works
   → Resume execution
   → If decompression succeeds → confirms tree placement is the issue
   → If decompression fails → tree data itself is corrupt

5. After testing, Python sends ROLLBACK to restore original state
   → Try different tree address (0x800F4980 instead of 0x80100000)
   → Re-test without rebuilding ROM
```

---

## 6. Divergence Log Structure

### 6.1 JSON Format

```json
{
    "session_id": "v40a_test_001",
    "profile": "frank_v40a",
    "disc": "dq4_frankenstein_v40a.bin",
    "start_time": "2026-07-19T20:15:00Z",
    "end_time": "2026-07-19T20:15:32Z",
    "duration_instrs": 123456789,
    "final_status": "FREEZE_TIGHT_LOOP",
    
    "divergences": [
        {
            "id": 1,
            "instr_count": 5000,
            "pc": "0x800BC700",
            "type": "REGISTER_MISMATCH",
            "register": "t1",
            "expected": "0x80100000",
            "actual": "0x00000000",
            "severity": "FATAL",
            "detail": "Copy routine $t1 (destination) is zero — tree will be copied to NULL",
            "ghidra_context": {
                "function": "copy_routine (injected)",
                "decompiled_line": "t1 = TREE_DST_ADDR;",
                "mips_instruction": "lui $t1, 0x8010",
                "source_line": 3
            }
        },
        {
            "id": 2,
            "instr_count": 5200,
            "pc": "0x800BC714",
            "type": "MEM_CONSTRAINT_VIOLATION",
            "address": "0x80100000",
            "expected": "0x2923BE84 (hybrid tree first word)",
            "actual": "0x00000000",
            "severity": "FATAL",
            "detail": "Tree not copied to 0x80100000 — destination is zero",
            "ghidra_context": {
                "function": "copy_routine (injected)",
                "decompiled_line": "dst[0] = src[0]; // sw $t3, 0($t1)",
                "mips_instruction": "sw $t3, 0($t1)",
                "source_line": 7
            }
        },
        {
            "id": 3,
            "instr_count": 5000000,
            "pc": "0x80073670",
            "type": "HUFFMAN_TREE_MISMATCH",
            "register": "a2",
            "expected": "0x80100000 (relocated hybrid tree)",
            "actual": "0x800EF1C8 (original DW7 tree, zeroed by BSS)",
            "severity": "FATAL",
            "detail": "Huffman decompression called with original tree address — tree refs not patched",
            "ghidra_context": {
                "function": "huffman_decompress_primary",
                "decompiled_line": "tree = (uint16_t*)a2;",
                "mips_instruction": "lw $v0, 0($a2)",
                "source_line": 1
            },
            "huffman_detail": {
                "tree_addr": "0x800EF1C8",
                "tree_first_word": "0x00000000",
                "expected_first_word": "0x2923BE84",
                "leaf_node_check": {
                    "expected": "0x0260 ('A')",
                    "actual": "decode failed — tree is zero"
                }
            }
        },
        {
            "id": 4,
            "instr_count": 5000100,
            "pc": "0x8007368C",
            "type": "HUFFMAN_DECODE_FAILURE",
            "address": "0x80120100",
            "expected": "0x0260 (leaf value for 'A')",
            "actual": "0x0000 (null — tree traversal hit zero node)",
            "severity": "WARN",
            "detail": "Huffman decode produced 0x0000 instead of expected leaf — tree is zeroed",
            "ghidra_context": {
                "function": "huffman_decompress_primary",
                "decompiled_line": "*dst++ = node & 0x7FFF; // leaf value",
                "mips_instruction": "sh $v1, 0($a1)",
                "source_line": 12
            }
        }
    ],
    
    "injection_log": [
        {
            "instr_count": 5000001,
            "type": "WRITE_REG",
            "target": "a2",
            "value": "0x80100000",
            "description": "Test: redirect tree pointer to high RAM",
            "result": "decompression succeeded — tree at 0x80100000 is valid"
        }
    ],
    
    "valid_paths": [
        {
            "pc": "0x800BC700",
            "instr_range": [1234, 5200],
            "description": "Copy routine executed successfully",
            "exit_pc": "0x8008E284",
            "regs_at_exit": {"t0": "0x800BCD00", "t1": "0x801005C8"}
        }
    ],
    
    "summary": {
        "total_divergences": 4,
        "fatal": 3,
        "warnings": 1,
        "first_fatal_instr": 5000,
        "first_fatal_pc": "0x800BC700",
        "root_cause": "Copy routine destination register zeroed — tree not copied to high RAM",
        "suggested_fix": "Verify copy routine instructions at EXE offset 0xA4800 — check lui/addiu for $t1",
        "injection_verified": true,
        "injection_result": "Redirecting $a2 to 0x80100000 fixes decompression"
    }
}
```

### 6.2 Human-Readable Divergence Report

```
╔══════════════════════════════════════════════════════════════════════╗
║           DIVERGENCE REPORT — frank_v40a_test_001                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ Profile: frank_v40a    Duration: 12.3M instructions                 ║
║ Status:  FREEZE_TIGHT_LOOP at 0x8009870C                            ║
║                                                                      ║
║ DIVERGENCES (4 total, 3 FATAL):                                     ║
║                                                                      ║
║ #1 [FATAL] instr=5,000  PC=0x800BC700                               ║
║   Type: REGISTER_MISMATCH                                            ║
║   $t1 expected 0x80100000, actual 0x00000000                        ║
║   → Copy routine destination is zero — tree copies to NULL          ║
║   Ghidra: copy_routine line 3: "t1 = TREE_DST_ADDR;"               ║
║   MIPS:   lui $t1, 0x8010                                           ║
║                                                                      ║
║ #2 [FATAL] instr=5,200  PC=0x800BC714                               ║
║   Type: MEM_CONSTRAINT_VIOLATION                                     ║
║   0x80100000 expected 0x2923BE84, actual 0x00000000                 ║
║   → Tree not copied to high RAM                                     ║
║   Ghidra: copy_routine line 7: "dst[0] = src[0];"                  ║
║   MIPS:   sw $t3, 0($t1)                                            ║
║                                                                      ║
║ #3 [FATAL] instr=5,000,000  PC=0x80073670                           ║
║   Type: HUFFMAN_TREE_MISMATCH                                        ║
║   $a2 expected 0x80100000, actual 0x800EF1C8                        ║
║   → Tree refs not patched — decompression uses zeroed original      ║
║   Ghidra: huffman_decompress_primary line 1                         ║
║   MIPS:   lw $v0, 0($a2)                                            ║
║   Tree at 0x800EF1C8: 00 00 00 00 (zeroed by BSS)                  ║
║   Expected:          84 BE 23 29 (hybrid tree root)                ║
║                                                                      ║
║ INJECTION TEST:                                                     ║
║   Injected WRITE_REG $a2 = 0x80100000 at instr 5,000,001            ║
║   Result: Decompression succeeded ✓                                 ║
║   → Confirms: tree at 0x80100000 is valid, refs need patching       ║
║                                                                      ║
║ ROOT CAUSE: Copy routine $t1 is zero → tree not copied              ║
║ SUGGESTED FIX: Check lui/addiu at EXE offset 0xA4808-0xA480C        ║
║   Expected: lui $t1, 0x8010; addiu $t1, $t1, 0x0000                 ║
║   Actual:   (verify on disc)                                        ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 7. Targeted Decompilation Guide (Ghidra)

### 7.1 Priority Targets for Ghidra Analysis

Based on our existing reverse engineering, these are the specific subroutines to decompile, in priority order:

| Priority | RAM Address | EXE Offset | Function | Why |
|----------|-------------|------------|----------|-----|
| P0 | `0x80073670` | `0x5B770` | Huffman decompress primary | 9 call sites, reads tree at `0x800EF1C8` via `$a2` |
| P0 | `0x80084BF0` | `0x6CCF0` | Huffman decompress secondary | 3 call sites, different code path |
| P0 | `0x8008E284` | `0x76384` | BSS clear entry (PC0) | Zeros `0x800BC668`–`0x800F4980`, wipes tree + thread entry |
| P0 | `0x800562C0` | `0x443C0` | HBD sub-block parser | Parses type/count/tree-index, DQ4 types differ from DW7 |
| P1 | `0x800D9E80` | `0xC1F80` | CD-ROM thread entry | Zeroed by BSS, must be preserved or injected |
| P1 | `0x80098898` | `0x80998` | CD-ROM command handler | Disc check intercept target |
| P1 | `0x80098670` | `0x80770` | CD-ROM after-branch | Post-disc-check continuation |
| P2 | `0x80041CE8` | `0x29CE8` | Huffman decompress inline (DQ4) | 4 inline copies in menu render funcs |
| P2 | `0x8009F790` | `0x87890` | GPU display setup | 6 callers, display init |
| P2 | `0x8009F828` | `0x87928` | GPU_cw JAL call site | Sole GP0 command dispatch |
| P2 | `0x800A5DA0` | `0x93EA0` | GPU_cw trampoline | BIOS A:0x49 wrapper |
| P3 | `0x80096FA0` | `0x850A0` | DW7 global tree location | 186 leaves, tree format reference |
| P3 | `0x4184` (EXE) | `0x4184` | ABS folder pointer table | 168 matched + 1755 delta entries |
| P3 | `0x511C` (EXE) | `0x511C` | REL folder pointer table | 167 matched entries |
| P3 | `0xA39AA` (EXE) | `0xA39AA` | Sequential sector table | 44 entries, LBA 354→355 |

### 7.2 Ghidra Project Setup

```
1. Import DW7 EXE (translation/SLUSP012.06) into Ghidra
   - Language: MIPS:LE:32:default
   - Base address: 0x80017F00 (EXE load address)
   
2. Set entry point: 0x8008E284 (PC0 from EXE header)

3. Define memory regions:
   - RAM:      0x80000000 - 0x80200000 (2MB)
   - Scratch:  0x1F800000 - 0x1F800400 (1KB)
   - HW regs:  0x1F801000 - 0x1F801080 (GTE, DMA, etc.)
   - BIOS:     0x1FC00000 - 0x1FC80000 (512KB)

4. Auto-analyze, then manually label known functions:
   - 0x80073670 → huffman_decompress_primary
   - 0x80084BF0 → huffman_decompress_secondary
   - 0x8008E284 → bss_clear_entry
   - 0x800D9E80 → cdrom_thread_entry
   - 0x800562C0 → hbd_subblock_parser
   - 0x80098898 → cdrom_cmd_handler

5. Export function map:
   - File → Export Program → JSON (function list with entry/end addresses)
   - Save as: dw7_function_map.json
```

### 7.3 Logic Extraction Methodology

For each priority target:

```
Step 1: Decompile in Ghidra → pseudo-C
Step 2: Map MIPS registers to C variables:
        $a0-$a3 → function arguments
        $s0-$s7 → preserved locals
        $t0-$t9 → temporaries
        $gp → global pointer (tree segment base)
        $sp → stack pointer
        
Step 3: Identify memory accesses:
        lw/sw through $gp → global variables
        lw/sw through $sp → stack locals
        lw/sw through $a0-$a3 → buffer pointers
        
Step 4: Convert to Python HLRM:
        - Preserve exact memory layout
        - Use same address arithmetic
        - Validate against emulator trace
        
Step 5: Validate:
        - Run emulator with --huffman-trace
        - Compare register state at function entry/exit
        - Verify memory writes match HLRM predictions
```

---

## 8. Constraint Identification: Critical MIPS State to Track

### 8.1 Register States

| Register | At PC | Expected | What It Tells Us |
|----------|-------|----------|-----------------|
| `$a0` | `0x80073670` | encoded text buffer ptr | Source data for decompression |
| `$a1` | `0x80073670` | decoded output buffer ptr | Where decoded text goes |
| `$a2` | `0x80073670` | `0x80100000` (Option A) | Tree pointer — MUST match placement |
| `$a3` | `0x80073670` | `0x5C8` (1480) | Tree size in bytes |
| `$ra` | `0x80073670` | caller return addr | Which function called decompression |
| `$t0` | `0x800BC700` | `0x800BC738` | Copy routine: source (tree in EXE) |
| `$t1` | `0x800BC700` | `0x80100000` (Option A) | Copy routine: destination (high RAM) |
| `$t2` | `0x800BC714` | `$t0 + 1480` | Copy routine: end marker |
| `$t3` | `0x800BC718` | tree data word | Copy routine: current word being copied |
| `$v0` | `0x8008E284` | `0x800BC668` | BSS clear start address |
| `$v1` | `0x8008E290` | `0x800F4980` | BSS clear end address |
| `$sp` | any | `0x801FFFF0` ± offset | Stack must stay in safe range |
| `$gp` | any | `0x800B5xxx` | Global pointer — if corrupted, all globals break |
| `$k0` | IRQ handler | saved PC | Interrupt context |

### 8.2 Memory Addresses

| Address | Region | What to Watch |
|---------|--------|--------------|
| `0x800BC668` | BSS start | Must be zeroed after BSS clear, unless tree placed here |
| `0x800BC700` | Tree (Option 3) / Copy routine (Options A/B) | If copy routine: verify 14 instructions intact |
| `0x800BC738` | Tree data (Options A/B) | First word should be `0x2923BE84` (hybrid tree root) |
| `0x800D9E80` | Thread entry | Must be non-zero after BSS clear — zeroed = game hangs |
| `0x800EF1C8` | Original DW7 tree | Should be zeroed (we use relocated tree) |
| `0x80100000` | Relocated tree (Option A) | Must contain hybrid tree after copy routine |
| `0x800F4980` | BSS end / Game data start | Boundary — tree must not cross this |
| `0x80120000` | Text cache buffer | Decompression output — should contain ASCII leaf values |
| `0x800B52A0` | Spinlock A | If stuck here → threading issue |
| `0x800B63D8` | VBlank counter | Must increment — if stuck, IRQ delivery failure |
| `0x1F8010A0` | DMA2 BASE (GPU) | GPU DMA during rendering |
| `0x1F8010B0` | DMA3 BASE (CD) | CD-ROM DMA during HBD load |

### 8.3 Huffman-Specific Checks

```python
# At breakpoint 0x80073670 (decompression entry):
def validate_huffman_state(regs, mem_dump, constraint_map):
    """Validate Huffman decompression entry state."""
    checks = []
    
    # Check 1: Tree pointer matches placement
    tree_addr = regs['a2']
    expected_tree = int(constraint_map['tree_placement']['addr'], 16)
    if tree_addr != expected_tree:
        checks.append({
            'name': 'TREE_POINTER_MISMATCH',
            'severity': 'FATAL',
            'expected': hex(expected_tree),
            'actual': hex(tree_addr),
            'detail': f"Tree pointer ${'$a2'} = {hex(tree_addr)}, expected {hex(expected_tree)}"
        })
    
    # Check 2: Tree data is present at pointer
    tree_first_word = mem_dump.get(tree_addr, 0)
    expected_first = 0x2923BE84  # Hybrid tree root marker
    if tree_first_word == 0:
        checks.append({
            'name': 'TREE_DATA_ZERO',
            'severity': 'FATAL',
            'expected': hex(expected_first),
            'actual': '0x00000000',
            'detail': f"Tree at {hex(tree_addr)} is zeroed — BSS clear or copy failure"
        })
    
    # Check 3: Tree size matches
    tree_size = regs['a3']
    expected_size = constraint_map['huffman_tree']['hybrid_tree_size']
    if tree_size != expected_size:
        checks.append({
            'name': 'TREE_SIZE_MISMATCH',
            'severity': 'WARN',
            'expected': str(expected_size),
            'actual': str(tree_size),
            'detail': f"Tree size ${'$a3'} = {tree_size}, expected {expected_size}"
        })
    
    # Check 4: Source buffer is valid
    src_addr = regs['a0']
    if src_addr < 0x80000000 or src_addr >= 0x80200000:
        checks.append({
            'name': 'SRC_BUFFER_INVALID',
            'severity': 'FATAL',
            'expected': '0x80xxxxxx',
            'actual': hex(src_addr),
            'detail': f"Source buffer {hex(src_addr)} outside valid RAM range"
        })
    
    return checks
```

---

## 9. Implementation Priority

### Sprint 1 (Immediate: Unblocks Black Screen)

1. **Instrumentation Bus (C++)** — Named pipe emitter + patch injector in `psx_emulator_core.cpp`. ~200 lines C++.

2. **Live Analysis Sidecar (Python)** — Reads pipe, compares against constraint map, displays context. ~300 lines Python.

3. **Divergence Logger** — Structured JSON output for all divergences. ~100 lines Python.

### Sprint 2 (Next: Ghidra Integration)

4. **Ghidra Function Map Export** — One-time Ghidra headless run to generate `dw7_function_map.json`.

5. **Ghidra Bridge** — Python module that queries Ghidra for decompiled C at current PC. ~200 lines Python.

6. **Live Context Display** — Terminal UI showing decompiled C + MIPS + registers at breakpoint. ~200 lines Python.

### Sprint 3 (Future: Automated Patching)

7. **Patch Injector** — Python module that sends WRITE_MEM/WRITE_REG commands to emulator. ~150 lines Python.

8. **Rollback Engine** — C++ side that tracks all injected patches and can restore original state. ~100 lines C++.

9. **Automated Iteration** — Script that tries multiple tree placements without rebuilding ROM. ~200 lines Python.

---

## 10. CLI Usage

### 10.1 Running with Live Instrumentation

```bash
# Terminal 1: Start emulator with instrumentation
psx_agent_runner.exe dq4_frankenstein_v40a.bin test_v40a 200000000 telemetry.json 0 \
  --bp 0x80073670 --bp 0x800BC700 --bp 0x8008E284 --bp 0x800D9E80 \
  --huffman-trace \
  --live-instrument 10000

# Terminal 2: Start Python live analysis sidecar
python cybergrime/live_analysis.py \
  --pipe \\.\pipe\cybergrime_live \
  --constraints constraint_map.json \
  --golden golden_state.json \
  --ghidra-map dw7_function_map.json \
  --output divergence_log.json
```

### 10.2 Running with Patch Injection

```bash
# Terminal 1: Emulator (same as above)

# Terminal 2: Live analysis + injection
python cybergrime/live_analysis.py \
  --pipe \\.\pipe\cybergrime_live \
  --constraints constraint_map.json \
  --golden golden_state.json \
  --inject \
  --output divergence_log.json

# Terminal 3: Manual patch commands
python cybergrime/patch_injector.py --pipe \\.\pipe\cybergrime_live
> write_mem 0x80073670 0x3C0A8010 "patch lui $t2, 0x8010"
> write_reg 6 0x80100000 "set $a2 to relocated tree"
> dump_region 0x80100000 1480 "verify tree data"
> rollback "undo all patches"
```

---

## 11. Key MIPS Addresses Quick Reference

All addresses are RAM (KSEG0). EXE offset = RAM - 0x80017F00 + 0x800.

| Address | EXE Offset | Symbol | Role |
|---------|-----------|--------|------|
| `0x80017F00` | `0x800` | EXE_LOAD | EXE load address (base) |
| `0x80041CE8` | `0x29CE8` | huffman_decompress_inline | DQ4 inline decompression |
| `0x80073670` | `0x5B770` | huffman_decompress_primary | Primary decompression (9 callers) |
| `0x80084BF0` | `0x6CCF0` | huffman_decompress_secondary | Secondary decompression (3 callers) |
| `0x8008E284` | `0x76384` | bss_clear_entry | Original PC0, BSS clear start |
| `0x80096FA0` | `0x850A0` | dw7_global_tree | DW7 Huffman tree (186 leaves) |
| `0x80098898` | `0x80998` | cdrom_cmd_handler | CD-ROM command handler |
| `0x80098670` | `0x80770` | cdrom_after_branch | Post-disc-check continuation |
| `0x8009F790` | `0x87890` | gpu_display_setup | GPU display init (6 callers) |
| `0x8009F828` | `0x87928` | gpu_cw_call_site | GP0 command dispatch |
| `0x800A5DA0` | `0x93EA0` | gpu_cw_trampoline | BIOS A:0x49 wrapper |
| `0x800B52A0` | — | spinlock_a | Threading spinlock |
| `0x800B63D8` | — | vblank_count | VBlank counter |
| `0x800BC668` | `0x74768` | bss_clear_start | BSS clear loop start |
| `0x800BC700` | `0xA4800` | tree_or_copy_routine | Tree (mode 3) or copy routine (A/B) |
| `0x800BCCF0` | `0xA4DF0` | copy_routine_v34 | v34 copy routine (PC0) |
| `0x800D9E80` | `0xC1F80` | cdrom_thread_entry | CD-ROM thread (zeroed by BSS) |
| `0x800EF1C8` | — | dw7_tree_runtime | DW7 tree runtime copy address |
| `0x800F4980` | — | bss_clear_end | BSS clear end / game data start |
| `0x80100000` | — | tree_relocated | Hybrid tree (Option A, safe high RAM) |
| `0x801005C8` | — | trampoline_relocated | CD-ROM stop trampoline (v34) |
| `0x80120000` | — | text_cache_buffer | Decompression output buffer |
| `0x80138000` | — | dynamic_heap | Map geometry heap |
| `0x801FFFF0` | — | stack_top | Stack top |
