# Frankenstein Pipeline — Version Log

---

## V90 — First Commit (Jul 24, 2026)

**Initial release of the Frankenstein reverse build pipeline.**

### Pipeline Steps (25)
1. Load DW7 US EXE (0x80017F00 load addr, PC0=0x8008E284)
2. Load hybrid Huffman tree
3. Append hybrid tree to EXE (file 0xA5000, RAM 0x800BC700)
4. Patch tree pointer references (0x800EF1C8 → 0x800BC700)
5. Narrow BSS clear (START: 0x800BD100, END: 0x800D9E80)
6. Zero pre-tree BSS region
7. Decompressor patches (T4: tree ptr, T5: root_id+1, T6: offset_b mask)
8. Disc check bypass (9 NOPs + 1 variable patch)
9. CD-ROM stall bypass — SKIPPED (DuckStation)
10. HBD type trampoline (0x800BCF00)
11. FMV stubs (XA/STR + MDEC init)
12. CD-ROM command intercept — SKIPPED (defective)
12b. HBD string patch (w71 → q41)
12c. Thread blob injection + boot copy stub + PC0 redirect
12d. Text size update
13. LBA reference patch (354 → 362)
14. Sequential sector table patch
15-25. Disc assembly, directory, PVD, CUE, EDC/ECC, verification

### Known Issues
- **BIOS POST 03 stall** — EXE text_size not sector-aligned after appending thread blob + boot stub
- **MIPS sign-extension bug** — Boot copy stub used raw bit-shifting for LUI/ADDIU address splitting, causing incorrect addresses when ADDIU sign-extends the lower 16 bits
- **Black screen** — FMV stubs caused black screen; engine waits for FMV completion signal

### Result
Game stalls at BIOS POST 03. EXE file size (680,100 bytes) not a multiple of 2048. BIOS stops reading EXE prematurely, PC0 points to unloaded memory.

---

## V95 — Second Commit (Jul 24, 2026)

**Boot stall resolved. First successful boot past BIOS.**

### Changes from V90
- **Fixed MIPS sign-extension bug** in boot copy stub: Replaced manual bit-shifting with `split_addr()` helper function that correctly handles ADDIU sign-extension for LUI/ADDIU instruction pairs
- **Added sector-aligned padding** before `update_text_size()`: Ensures final EXE size is a multiple of 2048 bytes after all payloads (thread blob + boot stub) are appended
  - EXE size: 680,100 → 681,984 bytes
  - text_size: 0xA58A4 → 0xA6000 (sector-aligned)
- **Removed FMV stubs** (`patch_fmv_stubs` call completely removed from `build_reverse_option_a`): Suspected FMV stubs caused black screen by preventing engine from proceeding past FMV state

### Result
- BIOS POST sequence completes fully (0F→0E→01→02→03→04→05→06→07→kernel init)
- EXE loads successfully
- HBD data read from CD-ROM
- Game runs at 59.82 FPS
- **Black screen** — no Enix logo, no visual output
- DuckStation log shows game seeking to LBA 146621 (DW7 FMV position), which is all zeros on DQ4 disc
- Without FMV stubs, MDEC decoder spins on garbage data (569 FPS)

---

## V96 — Current (Jul 24, 2026)

**FMV stubs re-enabled + GPU display enable. Black screen persists — state machine blocker identified.**

### Changes from V95
- **Re-enabled FMV stubs** (`patch_fmv_stubs` restored): DQ4 disc has no FMV data at DW7's LBA 146621. Stubs prevent the 569 FPS MDEC spin.
  - XA/STR function stubbed at 0x8008AEF4 (file 0x737F4)
  - MDEC init function stubbed at 0x8008CAD0 (file 0x753D0)
  - Stub pattern: `li v0, 0; jr ra; nop` (return success)
- **Added GPU display enable to boot stub**: Writes GP1(0x03)=0x03000000 to 0x1F801814 before jumping to ORIG_PC0, enabling display output
  - Boot stub expanded from 14 to 18 instructions
- **Diagnostic scripts created**:
  - `find_fmv_lba.ps1` — Searches EXE for hardcoded FMV LBA (146621) and BCD MSF (32:34:71)
  - `scan_xa.ps1` — Scans DQ4 disc for XA/STR sectors (found only 10, all system area)

### Result
- BIOS POST completes
- EXE loads, HBD reads succeed
- No seek to LBA 146621 (stubs prevent FMV seek)
- Game runs at steady 59.82 FPS
- **GPU: 0.00 throughout** — no rendering activity
- Black screen persists

### Root Cause Analysis
FMV stubs return 0 (success) but the game's main loop waits for an **async completion signal** (VBLANK callback, DMA interrupt, or event flag) that the FMV system would normally set. The stubs skip FMV playback but never trigger the state transition the game expects to proceed to title screen / Enix logo rendering.

The GPU display enable turns on output, but the game never sets up a frame buffer or drawing environment — that initialization was supposed to happen during/after FMV playback.

### Key Values
- EXE size: 681,984 bytes (text_size=0xA6000, sector-aligned)
- PC0: 0x800BD76C (boot stub) → jumps to 0x8008E284 (ORIG_PC0)
- Boot stub: 18 instructions (blob copy + GPU enable + jump)
- FMV stubs: 2 sites (XA/STR + MDEC init)
- DQ4 disc XA/STR sectors: 10 (system area only, no FMV data)

### Next Steps
1. Full 25-step pipeline audit for bugs or gaps
2. Find FMV completion flag / state transition mechanism
3. Compare working DW7 boot log with hybrid boot log
4. Consider: only stub MDEC init (not XA/STR) to let CD-ROM seek complete
5. Consider: patch the wait loop that checks FMV status
