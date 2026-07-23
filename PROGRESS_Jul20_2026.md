# Frankenstein Build Progress — Jul 20, 2026

## Current Build: v42

### What Changed
- **CD-ROM stall bypass + FMV skip: DISABLED** in `frankenstein_builder.py` (step 7c)
  - With HBD truncation fix, LBA 146621 has valid DW7 data, so events should fire naturally
  - The bypass was corrupting CD-ROM command FIFO (12 params instead of 3), causing black screen
- **HBD truncation fix: ACTIVE** — only writes 141,197 non-zero DQ4 sectors, preserves DW7 data from sector 141,552+
- **Pre-tree BSS zero: ACTIVE** — zeros 0x800BC668-0x800BC700 (152 bytes)

### DuckStation Test Results (v42, no bypass)
1. **Enix logo appears** — game boots past BIOS
2. **Goes dark after Enix logo** — screen goes black
3. **CD-ROM command spam** (t=9.6-10.3s): Setloc commands with 12 params instead of 3, rapid command cancellations
4. **Invalid memory reads** (t=64.6s+): Reading from 0x80916AD8+ at PC 0x800761DC — corrupted pointer from failed CD-ROM data load

### Root Cause Analysis
- The CD-ROM command FIFO corruption (12 params instead of 3) is happening **even without the bypass patch**
- This means the game's CD-ROM driver is itself sending corrupted commands
- The invalid reads at 0x80916AD8 (PC 0x800761DC) suggest the game loaded garbage data from CD and is using it as a pointer table
- Address 0x80916AD8 is in the PSX expansion region (above 0x80800000) — likely an unmapped area being accessed via a corrupted base pointer

### Key Addresses
- PC 0x800761DC: Code reading from invalid addresses 0x80916AD8+ (incrementing by 0x10)
- EXE load address: 0x80017F00
- EXE file offset for PC 0x800761DC: 0x800761DC - 0x80017F00 + 0x800 = 0x5E8DC

### Next Steps
1. **Disassemble code at PC 0x800761DC** to understand what pointer table it's walking
2. **Use cybergrime supervisor** (`duckstation_supervisor.py`) to monitor PC and registers live:
   - `python cybergrime/duckstation_supervisor.py --cue dq4_frankenstein_v42.cue --monitor 120`
   - Set breakpoint at 0x800761DC to inspect registers when invalid reads start
3. **Investigate CD-ROM command corruption**: The 12-param Setloc suggests the game is writing params to wrong CD-ROM register offsets — possibly a side effect of the HBD data format mismatch (DQ4 per-block tree vs DW7 global tree)
4. **Consider HBD re-encoding** (tree-mode 3) to convert DQ4 format to DW7 global-tree format

### Files Modified This Session
- `translation-tools/frankenstein_builder.py` lines 3419-3424: Disabled CD-ROM stall bypass and FMV skip

### Pending Cleanup
- User is manually deleting old .bin build artifacts (v1-v41 corpses)
- v42 is the current active build
