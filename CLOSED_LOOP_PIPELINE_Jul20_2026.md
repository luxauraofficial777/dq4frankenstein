# Closed-Loop Test Environment Integration & Verification Pipeline
**Date:** July 20, 2026
**Architecture Target:** Dragon Quest IV "Frankenstein" / DW7 Graft
**Integration Stack:** Frankenstein Builder -> CyberGrime Harness -> Vector DB Analyst -> DuckStation Telemetry

---

## 1. Unified Test Runner & Harness Integration

### 1.1 Automated Build-to-Test Pipeline
- **Orchestration Engine (`regression_runner.py`):** Serves as the master execution controller. It automates the entire loop without manual intervention:
  1. Calls `frankenstein_builder.py` (e.g., `--profile rc2`) to compile the workspace and inject the translation payload.
  2. Applies `extend_disc_mode2()` to write strict Mode 2 Form 1 sector headers.
  3. Executes `edcre.exe` to natively regenerate EDC/ECC checksums.
  4. Generates lightweight deployment patches natively via **xdelta3** (superseding legacy IPS/BPS limitations for ISO-scale data).
  5. Mounts the resulting `dq4_frankenstein_vXX.bin` into `psx_agent_runner.exe` for immediate execution validation.

### 1.2 State & Regression Hooks
- **Memory Boundary Checkpointing:** Integrated via `agent_harness.h`. The harness validates that the injected MIPS copy routine (`0x800BCCF0`) correctly copies the hybrid tree to the `0x80100000` safe zone *before* the native BSS clear loop (`0x800BC668` to `0x800F4980`) executes.
- **Pointer Table Integrity:** Automated validation of the 168 matched ABS/REL folder pointers and the `+1` LBA delta applied to the 1755 unmatched pointers, confirming memory maps do not point to compressed data sectors inadvertently.
- **Execution Traps:** Checkpoints established to catch illegal read faults during Huffman text-parsing loops (specifically checking the `0x8000` branch bit routines).

---

## 2. Emulator Telemetry & Headless Verification

### 2.1 Headless Execution & Log Parsing (CyberGrime & DuckStation)
- **CyberGrime Telemetry (`psx_agent_runner.exe`):** Runs the game at max CPU throughput. Generates strict JSON profiles (`telemetry_vXX.json`) recording Instruction Execution Count (50M+ baseline), VBlank Counts, Last Known PC, and full CPU register dumps.
- **DuckStation Tracing:** Automated headless boot triggers via CLI flags. The pipeline tails `duckstation.log` to intercept native warnings (e.g., "Invalid byte read at address 0xFFFF8002") and parses VRAM cache events to ensure CD-ROM seek time simulation matches Heartbeat parsing schedules.

### 2.2 Exception & Fault Catching
- **Vector Analyst Node (`vector_node.py`):** Automatically ingests the JSON telemetry streams post-execution.
- **Trap Identification:** 
  - Immediately flags PC stalls if the threshold is met (`HARNESS_FREEZE_THRESHOLD = 50,000,000` instructions without PC change).
  - Flags thread-initialization failures (e.g., a stall at `0x800D9E80`).
  - Detects out-of-bounds BIOS calls (e.g., a return to `ra=0x00000001` triggering an illegal exception jump to `0x80000000`).

---

## 3. Cross-System Stress Testing & Loop Closure

### 3.1 Concurrent Simulation Verification
- **Dual-Engine Execution:** Executes parallel audits across CyberGrime and DuckStation environments.
- **Timing & Performance:** Verifies that injected Heartbeat data streams inside the DW7 executable do not violate the `CD_READ_TIME = 33868 cycles` limits. Frame-rate stability is proven by validating VBlank polling loops are bypassed correctly (i.e., not stalled waiting for display interrupts the GPU stub cannot produce).
- **Leak Prevention:** Statically-sized buffers (`HARNESS_TTY_BUF_SIZE = 1MB`, `HARNESS_VFS_MAX = 512`) ensure test endurance runs process continuously without garbage collection stalls or dynamic memory bloat.

### 3.2 Guaranteed Verification Gate
- **Pass/Fail Criteria:**
  1. `Pass_Status: COMPLETE` logged in CyberGrime telemetry.
  2. `edcre` processes 100% of 287,272 sectors with 0 EDC validation failures.
  3. `duckstation.log` yields 0 core engine exceptions (illegal instruction / unaligned memory).
  4. Vector DB detects 0 new PC violations relative to the previous known-good state.
- **Automated Rollback:** If any parameter fails the verification gate, the pipeline instantly halts. It dumps the exact failure PC offset, the telemetry trace file, and issues an automated rollback of `frankenstein_builder.py` and `full_translation.json` to the last verified commit ID to prevent pipeline contamination.
