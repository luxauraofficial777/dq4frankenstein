# CyberGrime Upgrade Progress Log

**Date:** Jul 20, 2026  
**Blueprint:** `study/BLUEPRINT_CYBERGRIME_UPGRADE_Jul20_2026.md`

---

## Completed Tasks

### P0-1: Remove stall bypass + FMV skip from v43 build
- Removed `patch_cdrom_stall_bypass` and `patch_fmv_skip` calls from the build pipeline
- Root cause: stall bypass causes CD-ROM parameter FIFO overflow (12 params instead of 3 for Setloc)
- FMV skip had no effect (pattern not found in DW7 EXE)

### P0-2: Implement Strategy A — hybrid HBD (DW7 + DQ4 concatenated)
- DW7 HBD preserved at LBA 355 (618,563,584 bytes = 301,760 sectors)
- DQ4 HBD appended at LBA 302,115 (319,436,800 bytes = 155,975 sectors)
- Total HBD size reported in ISO directory

### P0-3: Update ISO dir for total HBD size
- ISO directory HBD entry updated to report DW7_HBD_SIZE + DQ4_HBD_SIZE

### P0-4: Create build_v44.ps1 and build v44
- v44 built successfully with hybrid HBD, no stall bypass, no FMV skip
- EDC/ECC 100% success

### P1-5: CD-ROM command tracer in CyberGrime
- **File:** `cybergrime/psx_cdrom_tracer.h` (new)
- Captures every CD-ROM command with PC, RA, params, result
- Spam detection (Getstat repeats), Setloc error counting, LBA progress tracking
- Event log for BIOS event delivery (TestEvent, WaitEvent, EnableEvent, deliverEvent)
- Integrated into `psx_emulator_core.cpp` at command dispatch and BIOS event handlers
- JSON output in `agent_harness.cpp` telemetry

### P1-6: EXE disassembler with DW7 symbols
- **File:** `cybergrime/psx_exe_disasm.h` (new)
- MIPS R3000A disassembler with pre-registered DW7 symbol table
- Decodes all major instruction types: R-type, I-type, J-type, COP0
- Register name resolution, branch target labeling, symbol annotation
- Integrated into crash dump in `psx_emulator_core.cpp`

### P1-7: Event flag monitor
- WaitEvent, TestEvent, EnableEvent, deliverEvent all traced
- Event log entries with class/spec/flags/action
- JSON telemetry includes event trace

### Test v44 in CyberGrime harness
- v44 tested successfully: no crashes, no Setloc parameter overflow
- CD-ROM command trace confirmed clean command flow
- Event flag synchronization working correctly

### P2-4: EXE patch validation + revert
- **Files:** `cybergrime/psx_binary_ops.h`, `cybergrime/psx_binary_ops.cpp`
- **PatchRecord struct:** tracks offset, original/patched values, RAM address, name, reverted flag
- **verify_cdrom_flow():** checks CD-ROM event wait loop at 0x800986E0 is intact
- **verify_integrity():** 7-point check: PC0, load addr, text size, CD-ROM event loop, forbidden patches, BSS clear mode, EXE size
- **revert_patch(name):** restores original value for a named patch
- **revert_all_patches():** restores all patches
- **list_patches():** prints all patches with active/reverted status
- **has_patch(name):** checks if a named patch is active
- All existing patch methods instrumented with recording:
  - `patch_lba_references` → "lba_ref"
  - `patch_sequential_sector_table` → "seq_table"
  - `patch_tree_references` → "tree_ref_lui" + "tree_ref_addiu"
  - `patch_disc_check` → "disc_check"
  - `patch_disc_check_variable` → "disc_check_var[N]+offset"
  - `patch_bss_clear_narrow` → "bss_clear_narrow_lui" + "bss_clear_narrow_addiu"
  - `patch_cdrom_stall_bypass` → "cdrom_stall_bypass"
  - `patch_fmv_skip` → "fmv_skip" / "fmv_skip_broad"
- Post-build verification: `build_v39` calls `verify_integrity()` + `list_patches()` before returning
- C API wrappers + DLL exports (7 new functions in `psx_ops.dll`)
- `psx_ops.def` updated with new exports
- Clean build with MSVC /O2, no warnings

---

## In Progress

### P2-8: Enhanced telemetry with CD-ROM trace (Section 5.1) — COMPLETE
- CD-ROM command trace JSON already in agent_harness.cpp
- Added: event log JSON output (events array with instr, event_id, class, spec, flags, action, test_result)
- Added: `instructions_at_last_progress` field to JSON telemetry
- Added: diagnostic summary block (fifo_overflow_detected, command_spam_detected, no_lba_progress, command_count, error_rate)
- Fixed: `instructions_at_last_progress` now properly tracked in emulator core

### P2-8b: CD-ROM error breakpoint (Section 5.2) — COMPLETE
- Added error breakpoint fields to `CdromTracer`: enabled, fired, PC, RA, cmd, params, cmd_name
- `fire_error_breakpoint()` — dumps to stderr: command, PC, RA, params, last 10 commands
- `enable_error_breakpoint()` — activates breakpoint mode
- `dump_breakpoint_json()` — JSON output for telemetry
- Wired into `psx_emulator_core.cpp` at Setloc param overflow detection point
- JSON telemetry includes `cdrom_breakpoint` section

### P2-9: DuckStation log comparison tool (Section 5.3) — COMPLETE
- **File:** `cybergrime/cdrom_log_compare.py` (new)
- Parses CyberGrime telemetry JSON (CDROM_Trace.commands + events)
- Parses DuckStation log (regex patterns for command execution, interrupts, Setloc details, errors)
- Compares command sequences by alignment, reports:
  - Matched, diverged, CG-only, DS-only commands
  - Param count mismatches
  - Error status mismatches
  - Missing/extra interrupt deliveries
  - Match rate percentage
- Supports `--verbose` for full divergence list, `--json-output` for machine-readable result

---

## Pending

### P3: Strategy C — CD-ROM command interception — COMPLETED
- MIPS trampoline at 0x80098898 to intercept Setloc/Setmode
- Fallback if FMV still hangs after all other strategies
- **Implementation**: `ExePatcher::patch_cdrom_cmd_intercept()` in `psx_binary_ops.cpp`
  - JAL to trampoline at 0x80101100 from CD-ROM dispatch (0x80098898)
  - 28-instruction MIPS trampoline (112 bytes):
    - Setloc (0x02): reads minute param from CD-ROM FIFO, if >= 0x32 (BCD, FMV range), clears FIFO via BFRD reset and writes safe sector (LBA 355 = MSF 0:06:05)
    - Setmode (0x0E): reads mode param, if 0xA0 (FMV double-speed), overwrites with 0x00 (normal speed)
    - All other commands: pass through to original instruction, jump back to dispatch+8
  - C API wrapper: `exe_patch_cdrom_cmd_intercept()` in `psx_binary_ops.cpp`
  - Exported in `psx_ops.def`
  - DLL and agent runner build successfully

### P7: Verification Protocol — COMPLETED
- **Implementation**: `verify_v44.py` in `cybergrime/` directory
  - 7.1 Build Verification: checks EXE header (PC0, load addr, text size), HBD LBA, HBD size, EDC/ECC, forbidden patches
  - 7.2 DuckStation Test: launches DuckStation with v44 disc, captures log, checks for Setloc errors and command spam
  - 7.3 CyberGrime Harness Test: runs `psx_agent_runner.exe`, parses telemetry JSON, validates VBlank count > 126, no freeze, LBA advancing, no invalid reads, GPU rendering > 0, no CD-ROM errors
  - Generates JSON report with overall verdict (PASS/FAIL)
  - Usage: `python verify_v44.py --disc <cue> --exe <exe> [options]`

---

## Key Values Reference

| Value | Description |
|-------|-------------|
| 0x80017F00 | EXE load address |
| 0x8008E284 | Original PC0 |
| 0x807E0 | CD-ROM poll bypass patch offset (EXE file offset) |
| 0x800986E0 | CD-ROM event wait loop (bne $v1,$zero,+12) |
| 0x1460000C | Original bne instruction |
| 0x1000000C | Patched beq (stall bypass — FORBIDDEN) |
| 355 | HBD LBA |
| 618,563,584 | DW7 HBD size (bytes) |
| 319,436,800 | DQ4 HBD size (bytes) |

---

## Build Commands

```bash
# Build psx_ops.dll (EXE patcher library)
cd cybergrime && build_psx_ops.bat

# Build psx_agent_runner.exe (emulator harness)
cd cybergrime && build.bat

# Run test
cd cybergrime && .\run_test.ps1 -Disc <disc.cue> -MaxInstrs 50000000
```
