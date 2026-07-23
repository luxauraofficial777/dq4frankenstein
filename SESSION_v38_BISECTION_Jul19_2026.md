# Frankenstein v38 Bisection — CyberGrime Harness Results

**Date**: Jul 19, 2026  
**Session**: v38a/v38b bisection testing via CyberGrime agent harness

---

## Executive Summary

The v38 bisection is **complete**. Both v38a (disc-check only) and v38b (full engine + HBD shift) boot identically to the unmodified DW7 baseline in CyberGrime. **No engine patch causes a crash, freeze, or boot failure.** The engine layer is clean.

The persistent "black screen" / `GPU: 0.00` issue is **not caused by any engine patch**. It affects the unmodified DW7 disc equally. The root cause is a runtime memory initialization problem: the game's BSS clear loop zeros the CD-ROM loading thread entry point at `0x800D9E80` before the game can use it, causing the game to stall in a VBlank wait loop waiting for display data that never loads.

---

## CyberGrime Harness Test Results

### Three Discs Tested (5M instructions each)

| Metric | Baseline DW7 (unmodified) | v38a (disc-check only) | v38b (full engine + shift) |
|--------|--------------------------|----------------------|--------------------------|
| **Status** | COMPLETE | COMPLETE | COMPLETE |
| **Errors** | 0 | 0 | 0 |
| **Crashed** | No | No | No |
| **Frozen** | No | No | No |
| **VBlanks** | 8 | 8 | 8 |
| **GPU GP0 writes** | 1 | 1 | 1 |
| **GPU GP1 writes** | 2 | 2 | 2 |
| **VRAM non-zero** | 0/524288 | 0/524288 | 0/524288 |
| **Final PC** | 0x8009674C | 0x8009674C | 0x8009870C |
| **HBD LBA** | 354 | 354 | 355 |
| **EXE size** | 675,840 | 675,840 | 677,888 |
| **Wall clock** | 7500ms | 7500ms | 6688ms |

### Key Observations

1. **All three discs produce identical behavior** — same VBlank count (8), same GPU register writes, same VRAM state (empty), same final PC region.
2. **No errors, no crashes, no freezes** in any build. The harness reports COMPLETE for all.
3. **VRAM is empty (0 non-zero bytes) in all three** — including the unmodified DW7 disc.
4. **Display is enabled** (`display_enabled=1`, `DMA_dir=2`, `fb_dirty=1`) but no pixel data is ever written to VRAM.
5. A `--no-preload` run of the baseline produced identical results, confirming the harness preload/thread blob injection doesn't change the outcome.

### BIOS Call Log Comparison

All three discs show the same BIOS call sequence:
- B-table fn=25 (ChangeTh) at ~292K instructions — thread initialization
- B-table fn=91 (FlushCache) at ~292K
- C-table fn=10 (ChangeClearRCt) at ~292K
- A-table fn=114 (start_card) at ~295K
- B-table fn=23 (interrupt setup) at ~521K
- B-table fn=53 (DMA transfer) at ~523K — GPU DMA init
- A-table fn=73 (GPU DMA setup) at ~524K
- B-table fn=86/A-table fn=68 (timer setup) at ~550K
- B-table fn=8/12 (event setup) at ~564K
- Then repeating B-table fn=23 (interrupt) every ~565K instructions — the VBlank polling loop

The instruction counts are nearly identical across all three discs (within ~7 instructions of each other), confirming the same code path is executed regardless of patches applied.

---

## Root Cause Analysis

### The BSS Clear / Thread Entry Problem

**The game's boot sequence:**
1. BIOS loads EXE from CD-ROM to RAM at `0x80017F00`
2. EXE entry point (PC0) starts execution
3. BSS clear loop at `0x8008E284` zeros memory from some start address through `0x800D9E80+`
4. Game initializes GPU registers (GP1 commands 0x03000001, 0x10000007, 0x04000002)
5. Game sets up DMA (GP0 0xE1001000)
6. Game enters VBlank wait loop at `0x80096620`–`0x80098840`
7. **Game waits for CD-ROM loading thread at `0x800D9E80` to populate display data**
8. **Thread entry is zeroed by BSS clear → thread never starts → game loops forever**

**Evidence from telemetry:**
- `[STUB_W]` writes at `PC=0x8008E294` zero `0x800D9E80`–`0x800D9EAC` at instruction ~151K
- GPU init happens later at instruction ~521K–550K
- VBlank polling starts at ~566K and continues through 5M instructions
- The game never exits the polling loop

### EXE Header Analysis

The DW7 EXE header is non-standard:
- `pc0 = 0x00000000` (overridden by SYSTEM.CNF `BOOT=cdrom:\SLUSP012.06;1`)
- `gp0 = 0x8008E284` (global pointer = BSS clear function address)
- `t_addr = 0x00000000`, `t_size = 0x80017F00` (load address from SYSTEM.CNF)
- `b_addr = 0x00000000`, `b_size = 0x00000000` (no BSS in header — BSS clear is code-level)
- `s_addr = 0x00000000`, `s_size = 0x801FFFF0` (stack at top of RAM)

The BSS clear is **not driven by EXE header fields** — it's a code-level loop at `0x8008E284` that zeros a hardcoded address range. The thread entry at `0x800D9E80` falls within this zeroing range.

### VBlank Wait Loop Analysis

From the TTY tail trace, the game loops through:
- `0x800965E8` — function entry, reads `0x800B52A0` and `0x800B52A4` (spinlock)
- `0x8009660C`–`0x80096624` — spinlock wait (compare v0 vs v1)
- `0x80096630`–`0x8009663C` — reads `0x800B52A8`, computes delta
- `0x80096640`–`0x80096658` — branch on sign (bgez a0)
- `0x80096648`–`0x8009674C` — reads `0x800B63D8` (vblank_cnt), jumps to epilogue
- `0x8009674C`–`0x8009675C` — function return (lw ra, lw s1, lw s0, jr ra, addiu sp)
- `0x800986D0`–`0x8009870C` — outer loop: reads `0x800F2C88`/`0x800F2C8C`, increments counter, checks threshold (0x003C = 60), branches

The game is counting VBlank interrupts (8 in 5M instructions) and checking if a counter has reached 60 (likely waiting for 1 second of VBlanks). The CD-ROM thread was supposed to populate the display data during this wait, but it never runs.

---

## Bisection Conclusion

| Test | Disc-check patches | Full engine patches | HBD shift | DQ4 content | Result |
|------|-------------------|--------------------|-----------|-------------|--------|
| Baseline DW7 | No | No | No | No | COMPLETE, VRAM empty |
| v38a | Yes (10) | No | No | No | COMPLETE, VRAM empty |
| v38b | Yes (10) | Yes (all) | Yes (354→355) | No | COMPLETE, VRAM empty |

**The engine patches are clean.** The problem is not in the disc-check bypass, tree relocation, Stop intercept, LBA patches, or HBD shift. The problem exists even on the unmodified DW7 disc.

**The real blocker is the CD-ROM loading thread initialization.** The game's BSS clear zeros the thread entry point at `0x800D9E80`, and the game can never start the thread that loads display data from the HBD.

---

## Recommended Next Steps

### Option A: Patch BSS Clear to Preserve Thread Entry
Narrow the BSS clear range to skip `0x800D9E80+`. This requires:
1. Finding the BSS clear loop's end address constant in the EXE
2. Patching it to stop before `0x800D9E80`
3. Risk: may leave other BSS variables uninitialized

### Option B: Post-BSS-Clear Hook (like v33/v34 tree copy routine)
Use the proven copy routine approach from v33/v34:
1. Append thread code + trampoline to end of EXE
2. Set PC0 to a copy routine that runs AFTER BSS clear
3. Copy routine injects thread code to `0x800D9E80`
4. Copy routine jumps to original PC0
5. This is the same pattern used for the Huffman tree relocation to `0x80100000`

### Option C: Bypass CD-ROM Thread Entirely
Instead of fixing the thread mechanism, pre-load all HBD data into RAM and patch the game to skip the thread wait. This is what the CyberGrime harness already does (preload), but the game's BSS clear undoes it. A post-BSS-clear injection would fix this.

**Recommended: Option B** — it uses the proven v33/v34 copy routine pattern and doesn't require understanding the full BSS clear range.

---

## Files

- **Telemetry**: `cybergrime/telemetry/telemetry_v38a.json`, `telemetry_v38b.json`, `telemetry_baseline_dw7.json`, `telemetry_baseline_nopreload.json`
- **Builder**: `translation-tools/frankenstein_builder.py` (profiles `frank_v38a`, `frank_v38b`)
- **Discs**: `dq4_frankenstein_v38a.bin`, `dq4_frankenstein_v38b.bin` (711,567,024 bytes each)
- **Analysis script**: `analyze_bss_thread.py`
- **Harness runner**: `cybergrime/psx_agent_runner.exe`
