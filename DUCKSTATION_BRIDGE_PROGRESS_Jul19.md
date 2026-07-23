# DuckStation Bridge & Closed-Loop Controller Progress
## Jul 19, 2026 10:45 PM

---

## Session Summary

Built and tested the complete closed-loop pipeline: DuckStation bridge → orchestrator analysis → memory injection → verification. Two major deliverables created and a critical write-path bug fixed.

---

## Files Created/Modified

### New Files
1. **`cybergrime/closed_loop_controller.py`** — Automated closed-loop controller that:
   - Launches DuckStation, finds PSX RAM and CPU state
   - Monitors PC for stall detection (128-sample window, ≤4 unique PCs = stall)
   - Feeds telemetry (PC, registers, thread_entry, BSS instr, tree value) to orchestrator
   - Injects orchestrator-recommended patches via `write_mem` with `VirtualProtectEx` fallback
   - Tracks boot phases (copy_routine → tree_copy → bss_clear → thread_entry)
   - Verifies golden state (thread_entry nonzero, tree nonzero, no stall, all phases hit)
   - Requires 3 clean cycles to declare BUILD READY
   - Has retry logic (3 attempts) for DuckStation launch

2. **`cybergrime/pipeline_audit.py`** — End-to-end pipeline verification suite (7 phases):
   - Phase 1: Disc structure (sync pattern, mode byte, sector alignment)
   - Phase 2: LBA alignment (HBD LBA shift, sector bounds, EDC)
   - Phase 3: EXE patches (PS-X EXE signature, load address, patch values)
   - Phase 4: BSS clear range integrity (overlap checks, narrowed range)
   - Phase 5: Huffman tree placement (RAM bounds, BSS overlap, alignment)
   - Phase 6: Address translation consistency (RAM ↔ file offset round-trip)
   - Phase 7: Live runtime verification (DuckStation bridge, read/write tests)

### Modified Files
1. **`cybergrime/duckstation_bridge.py`**:
   - Added `VirtualProtectEx` API binding (kernel32 prototype)
   - Rewrote `_write_raw` to handle read-only pages: tries direct WriteProcessMemory first, falls back to VirtualProtectEx → write → restore protection
   - Added 3-second initial delay in `wait_for_ram` before first scan

2. **`cybergrime/build_orchestrator.py`**:
   - Added BIOS HLE stall range (0x80030000-0x80040000) classification
   - Added game EXE stall range (0x80080000-0x800C0000) classification
   - Added catch-all: any frozen state classified as STALL_LOOP
   - Added `thread_entry` and `bss_clear_end_instr` to telemetry observation
   - Updated root cause matching: HF-001 matches on `thread_entry_zeroed=True` regardless of stall PC
   - Added `gp_anomaly` detection for BIOS HLE range
   - Added `telemetry` parameter to `_observe()` method

---

## Key Results

### Pipeline Audit (v38a, static + live)
- **49/52 checks PASS** (2 expected failures: MODE2 disc has no ISO9660 volume descriptor)
- All address translations verified: RAM ↔ file offset round-trip for 12 critical addresses
- BSS narrow patch offset confirmed: RAM 0x8008E290 → file 0x76B90
- Live runtime: all 12 critical memory addresses read successfully
- **Write capability CONFIRMED**: 0xDEAD1234 written and read back at 0x800D9E80
- **BSS narrow patch injection CONFIRMED**: 0x2462CCC8 written and read back at 0x8008E290

### Closed-Loop Controller (v38a, successful runs)
- Orchestrator correctly identifies **HF-001 (BSS_ZEROS_THREAD_ENTRY)** from telemetry
- Simulation PASSES: BSS narrow preserves thread entry and tree
- Patch recommendation: WRITE_MEM 0x8008E290 = 0x2462CCC8
- **Critical finding**: Runtime injection of BSS narrow patch alone doesn't un-stall because BSS clear has already executed — the instruction is already past. The patch must be applied at EXE build time.
- Thread entry is zeroed (0x00000000) in all runs — confirming HF-001 is the active blocker
- Tree is present (0x240261AA at 0x80100000) — copy routine executed successfully
- BSS start (0x800BC668) is zeroed — BSS clear ran
- Stall PC varies: 0x80032D14 (BIOS HLE), 0x800327C4 (BIOS HLE), 0x8008E8C4 (EXE code)

### VirtualProtectEx Fix
- **Root cause**: DuckStation marks code pages as read-only after JIT compilation
- **Fix**: `_write_raw` now tries WriteProcessMemory, falls back to VirtualProtectEx(PAGE_READWRITE) → write → restore protection
- **Verified**: Both test write and BSS patch injection succeed after fix

---

## Current Blocker: RAM Detection Failure

### Symptoms
- DuckStation launches (PID assigned) but RAM scan finds 540+ RW regions without matching PS-X EXE signature
- Occurs consistently across 3 retry attempts
- Earlier runs found RAM successfully (2MB exact match at various host addresses)

### Hypotheses
1. **DuckStation save state conflict**: Previous runs may have left a save state that causes DuckStation to skip EXE loading on subsequent launches
2. **DuckStation settings cache**: The `-nogui -fastboot` flags may not fully reset state between runs
3. **EXE signature not at offset 0x17F00**: DuckStation may load the EXE at a different RAM offset in some configurations
4. **Timing**: The 3-second initial delay may not be enough — DuckStation may need longer to load the CD and parse the EXE

### Next Steps
1. Delete DuckStation save states/memcards for this game
2. Check DuckStation settings for any caching behavior
3. Add fallback RAM detection: scan for BSS clear pattern instead of PS-X EXE signature
4. Try launching DuckStation without `-fastboot` flag
5. Verify the EXE is actually being loaded by checking DuckStation log output

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLOSED-LOOP CONTROLLER                         │
│                                                                   │
│  ┌─────────────┐    telemetry    ┌──────────────┐   patch_cmd    │
│  │ DuckStation  │──PC/regs/mem──▶│  Orchestrator │──────────────┐ │
│  │ Bridge       │                │  (Circle of   │              │ │
│  │              │◀──inject───────│   Truth)      │              │ │
│  │  RAM scan    │   write_mem    │               │              │ │
│  │  CPU state   │                │  HF-001..006  │              │ │
│  │  read/write  │                │  simulate     │              │ │
│  └─────────────┘                └──────────────┘              │ │
│       │                                                         │ │
│       │  stall? ◀────────────────verdict?──────────────────────┘ │
│       │  resolved?                                                │
│       ▼                                                          │
│  ┌─────────────┐                                                │
│  │  Verdict     │  3 clean cycles → BUILD READY                  │
│  └─────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘

Pipeline Audit (7 phases):
  Disc structure → LBA alignment → EXE patches → BSS range →
  Tree placement → Address translation → Live runtime
```

---

## Critical Addresses (Verified)

| Name | RAM Address | File Offset | Value (v38a) |
|------|------------|-------------|--------------|
| BSS clear entry | 0x8008E284 | 0x76B84 | 0x3C02800C |
| BSS end instr | 0x8008E290 | 0x76B90 | 0x24634980 (original) |
| BSS clear loop | 0x8008E294 | 0x76B94 | 0xAC400000 |
| Thread entry | 0x800D9E80 | 0xC2780 | 0x00000000 ⚠ ZEROED |
| Copy routine | 0x800BC700 | 0xA5000 | 0x00000000 (BSS cleared) |
| Huffman decompress | 0x80041CE8 | 0x2A5E8 | 0x2404008D |
| Tree dest | 0x80100000 | 0xE8900 | 0x240261AA (present) |
| Stall (event flag) | 0x8009885C | 0x8115C | 0x90820000 |

---

## EXE Patch Required (Build-Time)

**HF-001 BSS Narrow Patch:**
- File: `SLPM_869.16` (or equivalent EXE within disc)
- File offset: `0x76B90`
- Original bytes: `80 49 63 24` (little-endian 0x24634980)
- Patched bytes: `C8 CC 62 24` (little-endian 0x2462CCC8)
- Effect: BSS clear end changes from 0x800F4980 to 0x800BCCC8
- Preserves: Thread entry (0x800D9E80), Tree dest (0x80100000)

**Note**: Runtime injection of this patch does NOT un-stall because BSS clear has already executed by the time the stall is detected. This MUST be applied at EXE build time.

---

## Pending Work

1. **Fix RAM detection reliability** — investigate DuckStation state caching, add fallback detection methods
2. **Apply BSS narrow patch at build time** — patch the EXE within the disc image, not at runtime
3. **Rebuild v39 disc** with BSS narrow patch applied, test in DuckStation
4. **Run closed-loop controller on patched v39** — should reach thread_entry and boot
5. **Complete Huffman tree audit** — verify tree structure, node alignment, pointer integrity
6. **Cross-engine symbol resolution audit** — verify DQ4 heart assets map cleanly to DW7EXE architecture
