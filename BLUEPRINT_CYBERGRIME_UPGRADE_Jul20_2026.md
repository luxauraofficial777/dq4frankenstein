# Blueprint: CyberGrime Upgrade for EXE/HBD Patching and FMV Resolution

**Date:** Jul 20, 2026  
**Goal:** Upgrade CyberGrime to reliably patch the DW7 EXE and DQ4 HBD, resolve CD-ROM/FMV communication failures, and achieve a working playable build.

---

## 1. Root Cause Analysis: Setloc Parameter Overflow

### The Error
```
W(ExecuteCommand): Incorrect parameters for command 0x02 (Setloc), expecting 3-3 got 12
```

### Mechanism
DuckStation's CD-ROM controller (`study/duckstation-src/src/core/cdrom.cpp:519`) defines:
```cpp
{"Setloc", 3, 3},  // min=3, max=3 parameters
```

The `EndCommand()` function (`cdrom.cpp:1884`) clears the parameter FIFO after each command completes:
```cpp
void CDROM::EndCommand() {
    s_state.param_fifo.Clear();
    ...
}
```

The DW7 CD-ROM driver follows this sequence for each command:
1. Write 3 BCD params (minute, second, frame) to param FIFO via reg 0x1F801802
2. Write command byte (0x02 = Setloc) to reg 0x1F801803
3. **Wait for INT3 interrupt** (acknowledge) — the interrupt handler calls `EndCommand()` which clears the FIFO
4. Read response from response FIFO
5. Proceed to next command

### The Bug
The `patch_cdrom_stall_bypass` at `0x800986E0` converts a conditional `bne` to an unconditional `beq`, skipping the event-flag wait loop. This means step 3 is bypassed — the driver never waits for the INT3 acknowledge. It immediately writes the next command's parameters **without the FIFO being cleared**.

After 4 commands (3 params each = 12 bytes), the next Setloc sees 12 parameters instead of 3, triggering the error. The game then enters a Getstat/Setloc spam loop trying to recover.

### Why FMV Skip Also Failed
The `patch_fmv_skip` searches for `addiu $reg, $zero, 0xA0` (Setmode with double-speed + ADPCM) in the region 0x80098800-0x80098900. The DW7 EXE's CD-ROM Setmode call is likely at a different address or uses a different register pattern. The patch reported "no patches needed" — it found nothing to patch.

### Available PSX Reference Sources in `/study`

| Repo | Path | CD-ROM Relevance |
|------|------|------------------|
| **DuckStation** | `study/duckstation-src/` | Full CD-ROM controller emulation (`src/core/cdrom.cpp`, 4451 lines). Authoritative for command timing, FIFO behavior, interrupt delivery. |
| **PSn00bSDK** | `study/PSn00bSDK/` | Real-hardware CD-ROM library (`libpsn00b/psxcd/`). Shows how games issue commands: `CdControl()`, `CdCommand()`, `CdSync()`. Event/callback model. |
| **psxe** | `study/psxe/` | Alternate emulator with clean CD-ROM implementation (`psx/dev/cdrom/impl.c`). Simpler than DuckStation, good for cross-referencing. |
| **pctation** | `study/pctation/` | Another emulator with CD-ROM drive (`src/io/cdrom_drive.cpp`). Minimal Setloc handler for comparison. |
| **PSX_MiSTer** | `study/PSX_MiSTer/` | FPGA PSX implementation in Verilog. Hardware-accurate CD-ROM timing in `rtl/`. |
| **PSoXide** | `study/PSoXide/` | Rust PSX emulator. CD-ROM in `emu/` crates. |
| **starpsx** | `study/starpsx/` | Another Rust PSX emulator. CD-ROM in `core/`. |
| **lrusso-playstation** | `study/lrusso-playstation/` | JavaScript PSX emulator. Single-file `PlayStation.js` (1.3MB). |
| **OpenBIOS** | `study/reference/openbios_*.c` | PCSX-Redux open BIOS. Event delivery (`openbios_events.c`), interrupt handlers (`openbios_handlers.c`). Shows how BIOS delivers CD-ROM interrupts to game. |
| **DQHBE REFERENCE** | `study/DQHBE REFERENCE DOC/` | DW7-specific RE study. Memory map, engine architecture, CD-ROM filesystem routines at 0x80076040-0x8008247F. |

---

## 2. Upgrade Plan: CyberGrime CD-ROM Diagnostics

### 2.1 CD-ROM Command Tracer (New)

Add a structured CD-ROM command log to CyberGrime's harness that captures every CD-ROM register write and command execution, similar to DuckStation's `DEV_COLOR_LOG` output.

**File:** `cybergrime/psx_cdrom_tracer.h` (new)

```cpp
struct CdromCommandLog {
    uint64_t instruction_count;  // CPU instruction count at time of command
    uint8_t command;             // CD-ROM command byte
    uint8_t params[16];          // Parameter FIFO contents
    int param_count;             // Number of params in FIFO
    uint32_t pc;                 // CPU program counter
    uint32_t ra;                 // Return address (caller)
    int result;                  // 0=success, nonzero=error code
    char command_name[16];       // Human-readable name
};
```

**Integration points:**
- Hook into `psx_emulator_core.cpp` at the CD-ROM command execution switch (line ~1813)
- Log every command with PC, RA, params, and result
- Export as JSON in telemetry output
- Detect patterns: command spam, parameter overflow, missing interrupts

**Key diagnostic outputs:**
1. **Command sequence timeline** — ordered list of commands with PC/RA
2. **Parameter FIFO state** — dump FIFO contents before each command
3. **Interrupt delivery log** — track INT1/INT2/INT3/INT5 delivery
4. **Caller trace** — RA register shows which function issued each command
5. **Spam detection** — flag when same command repeats >5 times without progress

### 2.2 EXE Disassembler and Symbol Mapper (New)

CyberGrime's MIPS disassembler exists (`cybergrime/mips/disassembler.py`) but needs C++ integration for real-time analysis during emulation.

**File:** `cybergrime/psx_disasm.h` (new)

```cpp
class ExeDisassembler {
    // Disassemble a single instruction
    std::string disasm_one(uint32_t addr, uint32_t insn);
    
    // Disassemble a function (follow branches until jr ra)
    std::vector<DisasmLine> disasm_function(uint32_t entry_addr);
    
    // Find all callers of a function (scan for jal target)
    std::vector<uint32_t> find_callers(uint32_t target);
    
    // Identify function boundaries (prologue: addiu $sp, $sp, -N)
    std::vector<FunctionRange> map_functions(uint32_t start, uint32_t end);
    
    // Annotate known addresses with names
    void register_symbol(uint32_t addr, const std::string& name);
    std::string lookup_symbol(uint32_t addr);
};
```

**Pre-registered DW7 symbols** (from FMV_SKIP_INVESTIGATION and DQHBE reference):

| Address | Symbol | Source |
|---------|--------|--------|
| 0x8008AEF4 | `xa_str_playback_core` | FMV_SKIP_INVESTIGATION |
| 0x8008CAD0 | `mdec_init_str_streamer` | FMV_SKIP_INVESTIGATION |
| 0x8008B32C | `xa_start` | FMV_SKIP_INVESTIGATION |
| 0x8008B144 | `xa_stop` | FMV_SKIP_INVESTIGATION |
| 0x8008CA20 | `mdec_init` | FMV_SKIP_INVESTIGATION |
| 0x800A2108 | `CdSeek` (BIOS wrapper) | FMV_SKIP_INVESTIGATION |
| 0x800A2148 | `CdRead` (BIOS wrapper) | FMV_SKIP_INVESTIGATION |
| 0x800A2188 | `CdReadL` (BIOS wrapper) | FMV_SKIP_INVESTIGATION |
| 0x800A21C8 | `CDinit` (BIOS wrapper) | FMV_SKIP_INVESTIGATION |
| 0x800986E0 | `cdrom_event_wait_loop` | Stall bypass analysis |
| 0x80098898 | `cdrom_cmd_write` | ExePatcher constants |
| 0x80076040 | `disk_fs_core` | DQHBE reference |
| 0x80082480 | `disk_fs_sound` | DQHBE reference |
| 0x8003DA64 | `fmv_event_handler` | FMV_SKIP_INVESTIGATION |

### 2.3 CD-ROM Event Flag Monitor (New)

The core issue is event flag synchronization. We need to monitor the BIOS event system.

**File:** `cybergrime/psx_event_monitor.h` (new)

```cpp
class EventMonitor {
    // Hook deliverEvent (BIOS A0:83 / B0:32)
    void on_deliver_event(uint32_t class_id, uint32_t spec);
    
    // Hook testEvent (BIOS B0:30)
    void on_test_event(uint32_t event_id, int result);
    
    // Hook waitEvent (BIOS B0:31)
    void on_wait_event(uint32_t event_id, int result);
    
    // Track CD-ROM event specifically
    // CD-ROM events: class=0xF0000003, spec=0x00000001 (data ready)
    struct EventState {
        uint32_t event_id;
        uint32_t class_id, spec;
        int flags;  // 0=free, 1=disabled, 2=enabled, 3=pending
        uint64_t last_delivered_instr;
        uint64_t last_tested_instr;
        int test_count;  // How many times testEvent was called
    };
    std::map<uint32_t, EventState> events;
};
```

This lets us see:
- Whether the CD-ROM event is being delivered (interrupt fires)
- Whether the game is polling it (testEvent) vs blocking (waitEvent)
- How many times the game tests the event before giving up

---

## 3. FMV Resolution Strategy

### 3.1 Why the Current Approach Fails

| Patch | Problem |
|-------|---------|
| `patch_cdrom_stall_bypass` | Skips event wait → FIFO not cleared → param overflow (12 vs 3) |
| `patch_fmv_skip` (Setmode 0xA0→0x00) | Pattern not found in target region → no patches applied |
| `jr ra; nop` on playback functions | State never changes → event handler loops forever (v37 finding) |
| `jr ra; nop` on event handler | Doesn't help — hang is at LBA 146621 read, not FMV (v37 finding) |

### 3.2 Correct Approach: Preserve DW7 FMV Data

The v43 build already implements this partially — it writes only non-zero DQ4 HBD sectors and preserves DW7 data from sector `dq4_hbd_end` onwards, which includes FMV at LBA 146621. However, the DW7 HBD at LBA 354 is being **replaced** by DQ4 HBD at LBA 355, which means the DW7 HBD data (including FMV references) is lost.

**The real issue:** The DW7 EXE expects to find its own HBD data at LBA 354-355. When we replace it with DQ4 HBD, the EXE reads DQ4 data where it expects DW7 data, including FMV sector pointers. The FMV data itself (STR files) may still be on the disc, but the HBD metadata that points to them is wrong.

### 3.3 Strategy A: Hybrid HBD (Recommended)

Instead of replacing DW7 HBD entirely with DQ4 HBD, **concatenate** them:

```
LBA 355:     DW7 HBD (original, 618,563,584 bytes = 301,760 sectors)
LBA 302,115: DQ4 HBD (re-encoded, ~319,436,800 bytes = 155,975 sectors)
```

This preserves all DW7 FMV pointers and data. The ABS/REL pointer remapping already maps matched folders to the DQ4 base sector (302,115). Unmatched ABS pointers that reference DW7 FMV/data sectors continue to point at the DW7 HBD region.

**Current v43 status:** The build already does this for the data (`DW7 data preserved from sector dq4_hbd_end onwards`), but the ISO directory only reports DQ4 HBD size. The ISO directory needs to report the **total** HBD size (DW7 + DQ4).

**Required changes:**
1. In `build_frank_v43`: Write DW7 HBD at LBA 355, then DQ4 HBD after it
2. Update ISO directory HBD size to `DW7_HBD_SIZE + DQ4_HBD_SIZE`
3. Remove the FMV skip and stall bypass patches entirely — they're not needed if DW7 data is preserved
4. The ABS/REL remapping already handles this (168 matched → DQ4, 1755 unmatched → DW7 + delta)

### 3.4 Strategy B: FMV Sector Redirect

If Strategy A doesn't work (e.g., DQ4 HBD is too large to fit alongside DW7 HBD):

1. **Identify FMV sector ranges** in DW7 HBD (STR/XA data)
2. **Preserve those sectors** in the output disc at their original LBAs
3. **Overwrite only non-FMV HBD sectors** with DQ4 data
4. This requires parsing the DW7 HBD to identify which sectors contain STR/XA data vs. text/game data

**Implementation:**
- Scan DW7 HBD for sectors with submode byte indicating STR (video) or XA-ADPCM (audio)
- Build a "preserve map" of sectors that must not be overwritten
- When writing DQ4 HBD, skip sectors in the preserve map

### 3.5 Strategy C: CD-ROM Command Interception (Last Resort)

If both A and B fail, intercept the CD-ROM command flow at the MIPS level:

1. **Hook the CD-ROM command dispatch function** (0x80098898)
2. **Inject a MIPS trampoline** that:
   - Checks if the command is `Setloc` (0x02)
   - If the target LBA is in the FMV range (146621+), redirects to a "safe" sector
   - If the command is `Setmode` with 0xA0 (FMV mode), changes it to 0x00
   - Otherwise, passes through to the original function
3. This is more surgical than the current blind stall bypass

**Trampoline code (MIPS):**
```asm
# At 0x80098898 (cdrom_cmd_write), inject:
lui   $t0, 0x1F80        # CD-ROM reg base
lbu   $t1, 0x1803($t0)   # Read command register
addiu $t2, $zero, 0x02   # Setloc command
bne   $t1, $t2, .pass    # If not Setloc, pass through
nop
# Check if target LBA is in FMV range
# ... (compare setloc_lba with FMV range)
# If in FMV range, redirect to safe sector
.pass:
# Original instruction
```

---

## 4. EXE Patcher Improvements

### 4.1 Remove Broken Patches

**Remove from v43 build:**
- `patch_cdrom_stall_bypass` — causes parameter FIFO overflow
- `patch_fmv_skip` — pattern not found, no effect

### 4.2 Add Validation to ExePatcher

**File:** `cybergrime/psx_binary_ops.cpp` — add to `ExePatcher` class:

```cpp
// Verify patch doesn't break CD-ROM command flow
bool verify_cdrom_flow() {
    // Check that event wait loop at 0x800986E0 is intact
    uint32_t off = CDROM_POLL_BNE_OFF + 0x800;
    uint32_t insn = *(uint32_t*)(data.data() + off);
    if (insn != 0x1460000C) {
        printf("  WARNING: CD-ROM event wait loop modified!\n");
        return false;
    }
    return true;
}

// Verify all tree references point to valid tree
bool verify_tree_refs(uint32_t tree_ram) {
    // Scan for lui/addiu pairs referencing tree_ram
    // Verify they match expected count (24)
    // Verify tree data exists at tree_ram in EXE
}

// Full EXE integrity check after all patches
bool verify_integrity() {
    // 1. PC0 unchanged
    // 2. Load address unchanged
    // 3. Text size correct
    // 4. No forbidden patches (from DEFINITIVE_BLUEPRINT Rule 3.2)
    // 5. CD-ROM event loop intact
    // 6. All tree refs valid
    // 7. ABS/REL tables parseable
}
```

### 4.3 Add Patch Revert Capability

```cpp
struct PatchRecord {
    uint32_t offset;
    uint32_t original_value;
    uint32_t patched_value;
    const char* patch_name;
};

std::vector<PatchRecord> patch_history;

// Revert a specific patch by name
bool revert_patch(const char* name);
```

This allows surgical patch testing — apply/revert individual patches without rebuilding.

---

## 5. CyberGrime Harness Upgrades

### 5.1 Enhanced Telemetry

Add to `agent_harness.cpp`:

```cpp
// CD-ROM command trace (new)
struct CdromTelemetry {
    std::vector<CdromCommandLog> commands;     // Every CD-ROM command
    std::vector<EventDeliveryLog> events;       // BIOS event deliveries
    int setloc_errors;                          // Parameter overflow count
    int getstat_spam_count;                     // Getstat repeat count
    uint32_t last_progress_lba;                 // Last LBA that advanced
    int instructions_at_last_progress;          // Instruction count at last progress
};
```

### 5.2 Breakpoint on CD-ROM Error

Add a harness mode that sets a breakpoint when the CD-ROM controller would return an error:

```cpp
// Break when CD-ROM error is about to be sent
void set_cdrom_error_breakpoint() {
    // Set breakpoint at the error response path in the emulator
    // When hit, dump: PC, RA, SP, all registers, param FIFO, last 10 commands
}
```

### 5.3 DuckStation Log Comparison

Add a tool that compares CyberGrime's CD-ROM command trace with DuckStation's log output:

**File:** `cybergrime/cdrom_log_compare.py` (new)

```python
def compare_cdrom_logs(cybergrime_json, duckstation_log):
    """Compare CD-ROM command sequences between CyberGrime and DuckStation."""
    # Parse both logs
    # Align by instruction count / timestamp
    # Report divergences:
    #   - Different command sequences
    #   - Different parameter counts
    #   - Missing interrupts in one but not the other
    #   - Different error responses
```

---

## 6. Implementation Priority

| Priority | Task | Effort | Files |
|----------|------|--------|-------|
| **P0** | Remove stall bypass + FMV skip from v43 | 0.5h | `frankenstein_builder.py:3784-3806` |
| **P0** | Implement Strategy A (hybrid HBD: DW7 + DQ4) | 2h | `frankenstein_builder.py` build_frank_v43 |
| **P0** | Update ISO dir for total HBD size | 0.5h | `frankenstein_builder.py` update_iso_dir call |
| **P0** | Build v44 and test in DuckStation | 1h | `build_v44.ps1` |
| **P1** | CD-ROM command tracer in CyberGrime | 4h | `psx_cdrom_tracer.h`, `psx_emulator_core.cpp` |
| **P1** | EXE disassembler with DW7 symbols | 4h | `psx_disasm.h`, `psx_binary_ops.cpp` |
| **P1** | Event flag monitor | 2h | `psx_event_monitor.h`, `agent_harness.cpp` |
| **P2** | Enhanced telemetry with CD-ROM trace | 2h | `agent_harness.cpp`, `agent_harness.h` |
| **P2** | DuckStation log comparison tool | 2h | `cdrom_log_compare.py` |
| **P2** | EXE patch validation + revert | 2h | `psx_binary_ops.cpp`, `psx_binary_ops.h` |
| **P3** | Strategy C: CD-ROM command interception | 4h | `psx_binary_ops.cpp` ExePatcher |

**Total P0: ~4 hours** (can be done immediately)  
**Total P1: ~10 hours** (diagnostic infrastructure)  
**Total P2: ~6 hours** (verification tools)  
**Total P3: ~4 hours** (fallback strategy)

---

## 7. Verification Protocol

### 7.1 Build Verification (v44)

```
1. EXE patches: disc-check (11), LBA (1), seq table (44), ABS/REL (168+1755+3),
   tree refs (24), hybrid tree (1), BSS narrow (1)
   NO stall bypass, NO FMV skip
   
2. HBD layout: DW7 HBD at LBA 355 (301,760 sectors) + DQ4 HBD at LBA 302,115 (155,975 sectors)
   
3. ISO directory: HBD LBA=355, HBD size=DW7_HBD_SIZE+DQ4_HBD_SIZE
   
4. EDC/ECC: edcre.exe 100% success
   
5. EXE integrity: PC0=0x8008E284, load=0x80017F00, text_size=0xA5000
```

### 7.2 DuckStation Test

```
1. Load dq4_frankenstein_v44.cue in DuckStation
2. Observe for 60 seconds
3. Check for:
   - Enix logo appears (expected)
   - FMV plays or is skipped gracefully (no hang)
   - No "Incorrect parameters for command 0x02" errors
   - No Getstat/Setloc spam
   - Game reaches title screen or main menu
4. If hang: capture DuckStation log, check CD-ROM command sequence
```

### 7.3 CyberGrime Harness Test

```
1. Run psx_agent_runner.exe with v44 disc
2. Check telemetry:
   - VBlank count > 126 (past initial boot)
   - No freeze detection (PC not stagnant)
   - CD-ROM reads progressing (LBA advancing)
   - No invalid memory reads
   - GPU rendering > 0.00
3. If freeze: check CD-ROM command trace for error patterns
```

---

## 8. Key Reference Code Paths

### DW7 EXE CD-ROM Flow (from study sources)

```
Boot sequence:
  BIOS → CdInit (0x800A21C8) → CdRead (0x800A2148) → HBD header read
  → BSS clear → CdSetmode(0xA0) → CdSetloc(LBA 146621) → CdReadN
  → STR/XA playback (0x8008AEF4) → MDEC init (0x8008CAD0)
  → Event handler (0x8003DA64) polls playback state

FMV playback chain:
  0x8003DA64 (event handler)
    → 0x8008AEF4 (XA/STR playback core, 148 callers)
      → 0x800A2188 (CdReadL BIOS wrapper)
    → 0x8008CAD0 (MDEC init + STR streamer)
      → 0x8008CA20 (MDEC init)

CD-ROM command dispatch:
  Game code → writes params to 0x1F801802
  → writes command to 0x1F801803
  → BIOS IRQ handler delivers INT3
  → game reads response from 0x1F801801
  → BIOS EndCommand clears param FIFO

Event flag mechanism (from openbios_events.c):
  openEvent(class, spec, mode, handler) → returns event_id
  enableEvent(event_id) → sets flags=ENABLED
  deliverEvent(class, spec) → sets flags=PENDING (or calls handler)
  testEvent(event_id) → if PENDING, sets ENABLED, returns 1
  waitEvent(event_id) → busy-waits until PENDING
```

### DuckStation CD-ROM Command Info (authoritative)

From `study/duckstation-src/src/core/cdrom.cpp:519`:
```cpp
{"Setloc", 3, 3}    // Exactly 3 params required
{"Setmode", 1, 1}   // Exactly 1 param required
{"Getstat", 0, 0}   // No params
{"ReadN", 0, 0}     // No params (uses pending setloc)
```

Parameter FIFO size: 16 bytes max (`PARAM_FIFO_SIZE = 16`)

Error path (`cdrom.cpp:1896`):
```cpp
if (param_fifo.GetSize() < min || > max) {
    WARNING_LOG("Incorrect parameters...");
    SendErrorResponse(STAT_ERROR, ERROR_REASON_INCORRECT_NUMBER_OF_PARAMETERS);
    EndCommand();  // Clears FIFO
}
```

---

## 9. File Impact Summary

| File | Change | Priority |
|------|--------|----------|
| `translation-tools/frankenstein_builder.py` | Remove stall/FMV patches, implement hybrid HBD (Strategy A) | P0 |
| `cybergrime/psx_binary_ops.h` | Add `verify_cdrom_flow()`, `verify_integrity()`, `PatchRecord` | P1 |
| `cybergrime/psx_binary_ops.cpp` | Implement verification methods, remove broken patches | P1 |
| `cybergrime/psx_cdrom_tracer.h` | New: CD-ROM command trace structures | P1 |
| `cybergrime/psx_disasm.h` | New: EXE disassembler with DW7 symbols | P1 |
| `cybergrime/psx_event_monitor.h` | New: BIOS event flag monitor | P1 |
| `cybergrime/psx_emulator_core.cpp` | Integrate tracer, event monitor | P1 |
| `cybergrime/agent_harness.cpp` | Enhanced telemetry with CD-ROM trace | P2 |
| `cybergrime/cdrom_log_compare.py` | New: CyberGrime vs DuckStation log comparison | P2 |
| `build_v44.ps1` | New: Build script for v44 (Strategy A) | P0 |

---

## 10. Decision Matrix

| Question | Answer |
|----------|--------|
| Should we keep the stall bypass? | **No** — it causes parameter FIFO overflow |
| Should we keep the FMV skip? | **No** — pattern not found, no effect |
| Should we preserve DW7 HBD? | **Yes** — Strategy A (hybrid HBD) |
| Do we need the CD-ROM tracer? | **Yes** — for diagnosing any remaining issues |
| Should we port everything to C++? | **Eventually** — P0 fixes can be done in Python first |
| Is the HBD re-encoding correct? | **Likely yes** — v43 verification passed, 0 per-block trees |
| Is the disc-check bypass correct? | **Likely yes** — v38a proved 10 patches work, v41 has 11 |
| Is the BSS narrow patch safe? | **Yes** — v41 proved it boots past Enix logo |
