# V33 Debug Progress — Jul 18, 2026

## Current Status: Debugging "Instruction read failed at PC=0x801005C8"

### V33 Layout (Verified Correct)
- **EXE**: LBA 24, 677,888 bytes (331 sectors), load_addr=0x80017F00
- **PC0**: 0x800BCCF0 (copy routine) → copies 1520 bytes → j 0x8008E284 (original PC0)
- **Copy routine source**: 0x800BC700 (tree on disc, in loaded range ✓)
- **Copy routine dest**: 0x80100000 (high RAM, outside BSS clear ✓)
- **Tree**: 1480 bytes at 0x80100000 (after copy)
- **Trampoline**: 40 bytes at 0x801005C8 (after copy)
- **CD-ROM intercept**: j 0x801005C8 at EXE offset 0x80F64 (RAM 0x80098664)
- **BSS clear**: 0x800BC668 → 0x800F4980 (does NOT touch 0x80100000+ ✓)
- **memfill_start/size**: 0/0 (DuckStation does NOT use header BSS fields ✓)
- **RAM**: 0x801005C8 physical = 0x001005C8, within 2MB RAM (< 0x200000 ✓)

### Verification Results
1. **Copy routine instructions** — all 13 instructions verified correct on disc
   - Source: lui 0x800C + addiu 0xC700 = 0x800BC700 (sign-extension correct)
   - Dest: lui 0x8010 + addiu 0x0000 = 0x80100000
   - Payload: 1520 bytes (tree 1480 + trampoline 40)
   - Jump: j 0x8008E284 (original PC0)
2. **Trampoline instructions** — all 10 instructions verified correct on disc
3. **Tree data** — non-zero, starts with 0x0280 0x0480... (valid hybrid tree)
4. **BSS clear loop** — confirmed at 0x8008E284-0x8008E2A4, clears 0x800BC668 to 0x800F4980
5. **CD-ROM intercept** — j 0x801005C8 found at 0x80098664, delay slot nop'd
6. **Game init** — at 0x8004F344, calls many init functions, no heap init referencing 0x801F found

### Root Cause Hypothesis (Under Investigation)
The copy routine and all data are verified correct. The "Instruction read failed" error at 0x801005C8 means either:
1. **The copy routine never executes** — BIOS doesn't jump to PC0 correctly
2. **Something overwrites 0x80100000 after the copy** — PsyQ heap allocator or DMA
3. **The CD-ROM intercept fires before the copy completes** — timing issue
4. **DuckStation fast boot bypasses PC0** — BIOS patch may skip EXE entry point

### Next Steps
- Run DuckStation with `-earlyconsole -batch` to capture full boot log
- Check if copy routine at 0x800BCCF0 actually executes
- If copy executes but data is overwritten, move tree to safer address (e.g. 0x80180000)
- If copy doesn't execute, investigate BIOS fast boot behavior

### Key Files
- Disc: `dq4_frankenstein_v33.bin` / `.cue`
- Builder: `translation-tools/frankenstein_builder.py` (profile frank_v33)
- Verification scripts: `check_exe_header.py`, `verify_copy_routine.py`, `verify_bss_clear.py`, `check_heap_init.py`
- DuckStation: `duckstation\duckstation-qt-x64-ReleaseLTCG.exe`
