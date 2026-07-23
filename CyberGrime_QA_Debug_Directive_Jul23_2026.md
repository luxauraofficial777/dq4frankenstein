# CyberGrime — Test/QA/Debug Frankenstein DQ Directive
## The Golden Mile

**Date:** Jul 23, 2026 4:10am UTC-04:00
**Classification:** Build Directive — Final Build Phase

---

## TARGET VECTOR

The game does not begin at the title screen. The actual diagnostic window opens only **after character creation and the initial adventure save are completed**. That post-save heartbeat data load is where the runtime actually executes — and where the Frankenstein bridge must prove itself.

---

## DIRECTIVE

### 1. Telemetry Loop Must Simulate User-Driven Boot Sequence

The QA/telemetry loop must automate the **exact user-driven sequence**:

1. **Boot** — PSX BIOS loads EXE from sector 24
2. **Title screen** — DW7 EXE renders title (disc check bypass must pass)
3. **Character creation** — user input required (name entry, class selection if applicable)
4. **Adventure log creation** — EXE writes initial save state
5. **Initial adventure save** — save confirmed to memory card or internal state
6. **Post-save transition** — EXE attempts to load next game state from HBD
7. **THIS IS THE DIAGNOSTIC WINDOW** — the heartbeat data load immediately after the initial save

Steps 1-5 are prerequisites. Step 6 is where the Frankenstein bridge either works or fails. The telemetry loop must reach step 6 to produce meaningful diagnostics.

### 2. QA Debug Loop Must Isolate Post-Save HBD Load

The debug loop must:

- **Capture state at the moment of post-save transition** — register dump, RAM snapshot, CD-ROM LBA
- **Isolate the heartbeat data load** — identify which HBD sectors are being requested
- **Verify the requested sectors are valid DQ4 HBD data** — not FMV sectors, not DW7 sectors
- **Trace the Huffman decompression path** — confirm the per-block tree is being read correctly
- **Detect the FMV skip branch** — confirm the EXE skips FMV and proceeds to game state
- **Log any Setloc errors** — sector mismatch, parameter count errors
- **Capture first frame render** — confirm GPU is receiving draw commands

### 3. Automation Requirements

The telemetry loop must automate user input for steps 3-5:

- **Character creation:** Auto-enter a default name (e.g., "AAAA" or the first valid input)
- **Adventure log:** Auto-select "New Game" / first adventure log slot
- **Save confirmation:** Auto-confirm any save prompts
- **Input timing:** Use frame-accurate input injection (not wall-clock delays) to avoid desync
- **Timeout:** Allow up to 120 seconds for the full sequence (boot → save → post-save load)

### 4. Diagnostic Output Format

Each telemetry run must produce:

```
=== FRANKENSTEIN QA TELEMETRY REPORT ===
Date: [timestamp]
Disc: [filename]
EXE: SLUSP012.06 (patched)
HBD: DQ4 native (sector 362, unmodified)

[Phase 1: Boot]
  EXE loaded: YES/NO
  Load address: 0x80017F00
  PC0: [value]
  BSS clear: [start-end]
  Disc check: PASS/BYPASS/FAIL

[Phase 2: Title Screen]
  First frame: [timestamp] ms
  GPU activity: [vertex count, draw calls]
  Input required: YES (press start)

[Phase 3: Character Creation]
  Name entry: [automated]
  Class selection: [automated or N/A]
  Completion: [timestamp] ms

[Phase 4: Adventure Log]
  Save slot: [selected]
  Save written: YES/NO
  Completion: [timestamp] ms

[Phase 5: Post-Save Transition — DIAGNOSTIC WINDOW]
  CD-ROM LBA requested: [sector]
  Expected: DQ4 HBD sector (362+)
  Actual: [sector]
  Match: YES/NO
  Setloc errors: [count, details]
  FMV skip triggered: YES/NO
  FMV skip target: 0x8008AEF4 / 0x8008CAD0
  Heartbeat data load: STARTED/COMPLETE/FAILED
  Huffman decompression: [block_id, tree_size, success/fail]
  First game frame: [timestamp] ms or NOT RENDERED

[Phase 6: Game State]
  Rendering: YES/NO
  Black screen: YES/NO
  Crash: YES/NO (PC=[value] if crash)
  Duration: [seconds]
  Sectors read: [count]
  Last LBA: [value]

=== VERDICT: [PASS/FAIL/BLOCKED] ===
```

### 5. Failure Mode Classification

| Verdict | Condition |
|---|---|
| **PASS** | Game renders past post-save transition, no crash for 60+ seconds |
| **FAIL — FMV hang** | EXE hangs at CD-ROM seek after save, FMV skip not triggered |
| **FAIL — Setloc error** | CD-ROM seeks to wrong sector (LBA mismatch) |
| **FAIL — Decompression error** | Huffman tree parse fails, garbage text output |
| **FAIL — Cache overflow** | Text cache buffer overflows during decompression |
| **FAIL — BSS corruption** | CD-ROM thread entry zeroed, crash in BIOS dispatch |
| **FAIL — Disc check** | EXE rejects disc, shows "Insert Disc" screen |
| **BLOCKED** | Telemetry loop cannot reach post-save transition (input automation failure) |

### 6. Iteration Protocol

1. Build disc with all 10 patches applied
2. Run telemetry loop (automated boot → character creation → save → post-save)
3. Capture diagnostic output
4. Classify failure mode (if any)
5. If FAIL: identify root cause from diagnostic output, patch, rebuild
6. If BLOCKED: fix input automation, re-run
7. If PASS: extend test duration to 10 minutes, verify stability
8. Repeat until PASS + 10-minute stability

### 7. DuckStation-Specific Configuration

- **Fast boot:** Enabled (skip BIOS animation)
- **Fast forward:** Disabled during diagnostic window (need accurate timing)
- **Save state:** NOT allowed (must run full boot sequence)
- **Memory card:** Required (for adventure log save)
- **Controller input:** Automated via `duckstation_process.py` (frame-accurate injection)
- **Log capture:** DuckStation debug log enabled, filtered for CD-ROM + GPU events
- **Screenshot:** Capture at post-save transition + every 10 seconds after

---

## IMPLEMENTATION NOTES

### Input Automation Sequence
```
Frame 0:     Boot (no input)
Frame 60:    Press START (title screen)
Frame 120:   Press X (confirm "New Game")
Frame 180:   Press D-pad to select name characters
Frame 240:   Press X to confirm each character (4x for "AAAA")
Frame 300:   Press X to confirm name
Frame 360:   Press X to confirm adventure log slot 1
Frame 420:   Press X to confirm save
Frame 480+:  NO INPUT — observe post-save transition
```

*Note: Frame counts are approximate. The telemetry loop must detect screen transitions (via screenshot diff or GPU activity) and advance input accordingly, rather than using fixed frame counts.*

### DuckStation Log Filters
- `CDROM` — Setloc, read commands, interrupt status
- `GPU` — Draw commands, VRAM transfers
- `DMA` — CD-ROM to RAM transfers
- `CPU` — Exception/interrupt vectors
- `MDEC` — FMV decode (should show NO activity if FMV skip works)

### Memory Card Setup
DuckStation must have a formatted memory card in slot 1. The adventure log save requires it. If no memory card is present, the save step will fail and the diagnostic window never opens.

---

## RELATIONSHIP TO BUILD HANDOFF

This directive defines the **QA validation framework** for the build described in `BUILD_HANDOFF_Jul23_2026.md`. The build produces the disc; this directive defines how to test it.

The 10 EXE patches in the build handoff must all be verified by this telemetry loop:
- Patches 1-2 (LBA): verified by Setloc error count = 0
- Patch 3 (type trampoline): verified by successful sub-block field handling
- Patches 4-6 (tree): verified by successful Huffman decompression
- Patch 7 (BSS): verified by no crash in BIOS dispatch
- Patch 8 (disc check): verified by passing title screen
- Patch 9 (FMV skip): verified by no MDEC activity + game state reached
- Patch 10 (CD-ROM stall): N/A for DuckStation

---

## THE GOLDEN MILE DEFINITION

**The Golden Mile is the path from boot to first game frame render after adventure log save.**

If the telemetry loop captures a game frame render after the post-save transition, the Frankenstein bridge is proven. Everything after that is polish (text rendering quality, translation insertion, font patches).

The Golden Mile has these checkpoints:
1. ✅ EXE loads from disc
2. ✅ BSS clears without destroying thread entry
3. ✅ Disc check bypass passes
4. ✅ Title screen renders
5. ✅ Character creation completes (automated input)
6. ✅ Adventure log saves
7. ✅ Post-save HBD load requests correct DQ4 sectors
8. ✅ FMV skip triggers (no MDEC activity)
9. ✅ Huffman decompression succeeds (per-block tree read correctly)
10. ✅ **First game frame renders**

**The build is not done until checkpoint 10 is captured.**
