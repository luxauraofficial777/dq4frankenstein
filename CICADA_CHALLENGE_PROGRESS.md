# Cicada Challenge — Progress Notes
## Jul 24, 2026

## Current State
- **Build:** `dq4_reverse.bin` / `dq4_reverse.cue` — DW7 EXE on DQ4 disc base
- **Boot:** PASS — BIOS POST sequence completes (0F→0E→01→02→03→04→05→06→07→kernel init)
- **EXE Loading:** PASS — Sector-aligned text_size (0xA6000 = 679,936 bytes, 681,984 byte file)
- **FMV Stubs:** ENABLED — XA/STR (0x8008AEF4) + MDEC init (0x8008CAD0) stubbed with `li v0,0; jr ra; nop`
- **GPU Display Enable:** Added GP1(0x03)=0x03000000 write to boot stub before jumping to ORIG_PC0
- **Result:** Game runs at solid 59.82 FPS but **black screen** — GPU: 0.00 throughout

## Log Analysis (duckstation.log)
- Lines 138-213: BIOS POST sequence completes normally
- Lines 266-290: CD-ROM reads SYSTEM.CNF + EXE header sectors (LBA 154-161)
- Lines 415-498: CD-ROM reads HBD sectors (LBA 166-168, submode 0x89 then 0x08)
- Lines 596-935: Long sequential read of data sectors (LBA 174-506, submode 0x08)
- Line 955: `Setmode 0xA0` — engine enters double-speed ADPCM mode (FMV setup path)
- Lines 963-986: Steady 59.82 FPS, **zero GPU activity**, no further CD-ROM reads
- Line 987: GPU frame queue starts (user closing window)
- **No seek to LBA 146621** (DW7 FMV position) — stubs successfully prevent FMV seek
- **No GPU rendering** — game stuck in idle loop waiting for FMV completion signal

## Root Cause of Black Screen
FMV stubs return 0 (success) but the game's main loop waits for an **async completion signal**
(VBLANK callback, DMA interrupt, or flag) that the FMV system would normally set.
The stubs skip FMV playback but never trigger the event/flag the game expects.

Additionally, GPU display enable (GP1 write) turns on display output, but the game never
sets up a frame buffer or drawing environment — that initialization was supposed to happen
during/after FMV playback.

## Patches Applied (build_reverse_option_a)
1. License sector: NTSC-J → NTSC-U (LBA 4)
2. PVD: Volume ID → "DRAGONWARRIOR7_1"
3. Tree patches: 24 sites (0x800F→0x800C LUI/ADDIU refs)
4. BSS clear: start/end adjusted for hybrid tree + thread blob
5. Zero pre-tree BSS region
6. Decompressor patches: tree ptr variable (0x800BC6F8 = tree_base+2), root_id+1, offset_b mask
7. Disc check bypass: 10 surgical patches (no cd_init_force_success)
8. CD-ROM stall bypass: SKIPPED (DuckStation mode)
9. HBD type trampoline: at 0x800BCF00
10. FMV stubs: 2 sites (XA/STR at 0x8008AEF4, MDEC init at 0x8008CAD0)
11. CD-ROM command intercept: SKIPPED (removed — BAD-1)
12. HBD string: 'w71'→'q41' at file 0xF600
13. Thread blob injection: thread_code_blob.bin appended + boot copy stub
14. Boot copy stub: 18 instructions at 0x800BD76C (new PC0)
    - Copies thread blob to 0x800D9E80
    - Writes GP1(0x03)=0x03000000 to 0x1F801814 (GPU display enable)
    - Jumps to ORIG_PC0 (0x8008E284)
15. Sector-aligned padding + text_size update
16. LBA reference: HBD sector 354→362
17. Sequential sector table: 44 entries patched

## Key Values
- EXE load addr: 0x80017F00
- ORIG_PC0: 0x8008E284 (DW7 entry point)
- New PC0: 0x800BD76C (boot copy stub)
- EXE size: 681,984 bytes (text_size=0xA6000, sector-aligned)
- Thread blob dest: 0x800D9E80
- BSS clear range: [0x800BD100, 0x800D9E80)
- FMV XA/STR func: 0x8008AEF4 (file 0x737F4)
- FMV MDEC init func: 0x8008CAD0 (file 0x753D0)
- DW7 FMV LBA: 146621 (MSF 32:34:71) — DQ4 disc has zeros here
- DQ4 HBD LBA: 362 (vs DW7 HBD at 354)
- DQ4 disc XA/STR sectors: only 10 (system area) — no FMV data

## Files Modified
- `cybergrime/psx_binary_ops.cpp`:
  - `build_reverse_option_a()`: FMV stubs re-enabled, GPU display enable in boot stub
  - `split_addr()`: Sign-extension-aware address splitting for MIPS LUI/ADDIU
  - Sector-aligned padding before `update_text_size()`
  - Boot stub: 14→18 instructions (added GP1 display enable)

## Next Steps for Cicada Challenge
1. **Find FMV completion flag** — Game checks a variable/callback to know FMV is done.
   Patch it to be pre-set so the game proceeds past the wait loop.
2. **Only stub MDEC init, not XA/STR** — Let CD-ROM seek happen (reads garbage) but
   skip decode. The seek completion might trigger the event the game waits for.
3. **Patch the wait loop** — Find the branch that checks FMV status and force it to
   proceed (bne→beq or nop the branch).
4. **Full GPU init in boot stub** — Set up frame buffer base, display range, drawing
   environment so the game has a valid display context before entry.
5. **Compare with working DW7 boot** — Run original DW7 in DuckStation and compare
   log to identify what GPU/drawing setup happens after FMV that we're missing.

## Alternative Paths (from previous diagnostics)
- Native DQ4 EXE approach (`dq4_full_en.bin` — DQ4 EXE + patched HBD)
- RC2 approach (`dq4_en_rc2.bin` — DQ4 EXE on DW7 body)
- Post-decompression RAM hook at 0x800B4A68 (needs boot-time payload loader)
