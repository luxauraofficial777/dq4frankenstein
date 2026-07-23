# Project Roadmap & Execution Strategy
**Initiative:** Heartbeat Data Integration via DW7 Executable (Frankenstein/Graft Build)
**Date:** July 20, 2026

This roadmap details the low-level execution milestones required to natively boot and execute Dragon Quest IV Heartbeat (HBD) assets using the Dragon Warrior VII (DW7) executable engine.

---

## Phase 1: Binary Interface Analysis & Symbol Mapping

### 1.1 Executable Structure & Import Tables
- **EXE Mapping:** Target DW7 executable (`SLUSP012.06`) loads at `0x80017F00`. The base data section extends to `0xA4800` (RAM `0x800BC700`).
- **BSS Clear Routine:** Identify and map the native BSS zeroing loop initiated at PC0 `0x8008E284`. The memory wipe spans from `0x800BC668` to `0x800F4980`.
- **Thread Initialization:** Map the core thread entry point (`0x800D9E80`) to ensure initialization routines are not stalled or zeroed out by improperly placed payloads.

### 1.2 Heartbeat Core Interfacing
- **HBD Origin Shift:** The DW7 EXE expects HBD at LBA 354. To accommodate our extended EXE (331 sectors), shift the HBD mount point to LBA 355.
- **Safe Zone Allocation:** Establish a high-RAM safe zone outside of the BSS clear range (`0x80100000`) for custom injection payloads and translated text data structures to prevent overlay boundary collisions.

---

## Phase 2: Asset Parsing & Serialization Refactoring

### 2.1 Format Translation Layer
- **Huffman Schema Conversion:** Develop the `HuffTree` encoding layer to translate between DQ4's per-block local trees (`hts=24`) and DW7's global tree expectations (`hts=1366`, 1338-byte gap, `treeEnd=0`, `textEnd=0`).
- **Hybrid Tree Integration:** Construct a 1480-byte hybrid tree (370 leaves) capable of parsing full-width ASCII and DQ4/DW7 shared control codes, appending it directly to the extended EXE footprint.
- **Text Block Re-Encoding:** Serialize the raw English translations using a strict byte-budget system, guaranteeing a 100% fit within native Japanese memory boundaries without runtime buffer overflows.

### 2.2 Pointer Table & Offset Alignment
- **LBA Reference Patching:** Scan the entire EXE for 32-bit `354` offsets and sequential 16-bit table entries (`0xA39AA`), applying a `+1` delta to point to the new LBA 355 mount.
- **ABS/REL Table Alignment:** Read the ABS (`0x4184`) and REL (`0x511C`) pointer tables. Apply surgical remapping to the 168 matched folder structures and inject a `+1` LBA delta into the 1755 unmatched legacy pointers, maintaining alignment and preventing memory access violations.

---

## Phase 3: Runtime Integration & Execution Sandbox

### 3.1 Custom Hook Injection
- **MIPS Trampoline & Copy Routine:** Inject a 14-instruction MIPS copy routine at `0x800BCCF0`. This intercepts the native load sequence, copying the appended hybrid tree to the `0x80100000` safe zone immediately prior to the BSS clear loop.
- **Instruction Redirection:** Locate and patch the 24 `lui/addiu` MIPS instruction pairs in the DW7 EXE that reference the native tree (originally at `0x800EF1C8`), redirecting them to the injected payload at `0x800BC700` / `0x80100000`.
- **CD-ROM Polling Intercept:** Inject a CD-ROM branch hook at `0x80F64` (`j 0x801005C8`) to bypass stalling thread routines if the disk read cycle desynchronizes.

### 3.2 State Verification & Loop Stability
- **Execution Sandbox:** Utilize CyberGrime (`psx_agent_runner.exe`) as a strict hardware sandbox. Execute headless cycles monitoring PC registers for loops (e.g., hanging at CD-ROM poll loops due to invalid offset computation).
- **VBlank & Exception Tracking:** Capture execution states beyond 50,000,000 instructions to ensure stability past initial VBlank 126 and POST initialization phases.

---

## Phase 4: Verification, Telemetry & Stress Testing

### 4.1 Regression & Performance Validation
- **Hardware-Accurate Profiling:** Mount the compiled ROM in DuckStation to verify strict cache and seek-time parameters. Monitor for silent exceptions such as "Invalid byte read" during the `0xFFFF8002` execution cycle.
- **Vector Analyst Triage:** Feed runtime crashes into the Vector Allocation Node (`vector_db.py`) to map PC violation chains against specific executable modules. Resolve cross-ROM collision addresses systematically.

### 4.2 Final Build Pipeline
- **Sector Mode Assembly:** Run `extend_disc_mode2()` to guarantee structural Mode 2 Form 1 sector headers for extended data blocks (Sync + BCD MSF + Mode + Subheader).
- **Checksum Verification:** Route the final binary through `edcre.exe` for full EDC/ECC regeneration across all ~287,272 sectors.
- **Artifact Packaging:** Finalize the reproducible binary build (`frankenstein_builder.py`) and compute `xdelta3` deltas for lightweight deployment patches.
