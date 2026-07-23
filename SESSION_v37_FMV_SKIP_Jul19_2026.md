# Session: v37 FMV Skip Investigation — Jul 19, 2026

## Objective
Resolve the v36 black screen hang at LBA 146621 by implementing FMV skip patches in v37.

## Key Findings

### v36 Hang Analysis
- **Hang point**: LBA 146621 (MSF 32:34:71) — game reads ~55 sectors then pauses
- **GPU: 0.00** throughout — no rendering activity
- **FPS: 444** (fast-boot idle loop, not real game loop)
- LBA 146621 is deep in the DW7 HBD data (compressed/encoded content, not STR/FMV headers)
- The game boots, reads initial HBD sectors (LBA 166-174), then seeks to LBA 146621

### v37 FMV Skip Attempt 1: Patch 3 Playback Functions (jr ra; nop)
- **Targets**: `0x8008AEF4` (XA/STR playback core), `0x8008CAD0` (MDEC init + STR streamer), `0x8008B32C` (XA start)
- **Result**: Game loop runs at 59.82 FPS (real VPS), no CDROM hang, but **GPU: 0.00** persists
- **Root cause**: The event handler at `0x8003DA64` has a busy-wait loop:
  ```
  0x8003DAA0: jal 0x8008AEF4    ; call XA/STR playback (now returns immediately)
  0x8003DAA8: lh $r2, 0x46($r17) ; load playback state
  0x8003DAB4: beq $r2, $r0, 0x8003DAA0  ; if state==0, loop back forever
  ```
  Patching the playback function to return immediately means the state never changes, so the event handler loops forever doing nothing productive.

### v37 FMV Skip Attempt 2: Patch Event Handler Itself (jr ra; nop)
- **Target**: `0x8003DA64` (FMV event handler — skip entire FMV state)
- **Result**: Same behavior — FPS 444, GPU 0.00, CDROM reads LBA 146621 then idle
- **Conclusion**: The FMV event handler skip didn't help because the hang at LBA 146621 is **not the FMV**

### User Correction: FMV is AFTER Enix Logo
- The Enix logo should appear **before** the FMV
- The DW7 EXE may be trying to preload DW7 Disc 1's FMV from the HBD
- This triggers a disc validation ("Insert Disc 1 / Wrong Disc") loop
- The 10 existing disc-check patches don't cover this specific check
- **FMV skip was the wrong approach** — we need to unblock the FMV, not skip it

### D3D11 vs Vulkan
- Switched DuckStation renderer from Vulkan to D3D11
- No change in behavior — `GPU: 0.00` persists with both renderers
- The issue is not renderer-related

## Boot Sequence (from full DuckStation log)
1. BIOS POST sequence (0F → 01 → 03 → 04 → 05 → 06 → 07 → 08 → 09)
2. Kernel initialized
3. CDROM Init, Setloc 00:02:16 (LBA 166), ReadN — reads HBD header sectors
4. Reads LBA 166, 168, 172, 173, 174 (initial HBD scan, submode 0x89/0x08)
5. BSS Clear at 0x800BC668 (confirmed in log)
6. CDROM Init, Setmode 0xA0 (double-speed), Setloc 32:34:71 (LBA 146621)
7. Reads ~55 sectors at LBA 146621-146675 (submode 0x08)
8. Pause, Getstat — then idle loop forever (FPS 444, GPU 0.00)

## Current State of frankenstein_builder.py
- FMV skip patches **DISABLED** in `build_frank_v36` (commented out, `fmv_patches = 0`)
- v37 profile still registered but now builds identical to v36 (no FMV skip)
- All other patches intact: disc-check (10), tree relocation, LBA refs, HBD overlay

## Next Steps
1. **Investigate the disc validation loop** — the game reads LBA 146621 then enters an idle loop. This may be a "wrong disc" check that our 10 patches don't cover.
2. **Find the disc check that triggers on FMV preload** — search for additional disc validation code paths beyond the 10 existing patches
3. **Consider HBD block format incompatibility** — DW7's HBD parser may misinterpret DQ4 HBD block headers at LBA 146621, causing a silent failure that prevents rendering
4. **Trace the code path from LBA 146621 read to the idle loop** — use CyberGrime or MIPS disassembly to find what function reads LBA 146621 and why it fails

## Files Modified
- `translation-tools/frankenstein_builder.py` — FMV skip patches added then disabled (lines 2782-2786)

## Build Artifacts
- `dq4_frankenstein_v37.bin` — 817,414,080 bytes (v36 + disabled FMV skip = identical to v36)
- `dq4_frankenstein_v37.cue`
- `ds_v37_stdout.txt` — full DuckStation console log (79 seconds of output)
