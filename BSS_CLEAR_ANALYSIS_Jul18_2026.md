# BSS Clear Analysis Progress — Jul 18, 2026

## Status: BSS clear range confirmed, fix design in progress

## Problem
DuckStation shows `Instruction read failed at PC=0x800F4F48` — the trampoline at 0x800F4F48 is empty at runtime despite the copy routine placing it there.

## Root Cause Analysis

### v32 Disc Layout (verified via `cybergrime/verify_v32_copy.py`)
- **EXE header:** PC0=0x800BCCF0, t_addr=0x80017F00, t_size=0xA5000 (675,840 bytes)
- **Tree on disc:** file offset 0xA5000, RAM 0x800BC700 (1480 bytes, matches `dw7_hybrid_tree.bin`)
- **Trampoline on disc:** file offset 0xA55C8, RAM 0x800BCCC8 (40 bytes, 10 instructions)
- **Copy routine on disc:** file offset 0xA55F0, RAM 0x800BCCF0 (52 bytes, 13 instructions)
- **Copy routine == PC0:** Confirmed

### Copy Routine Decoded
```
lui   t0, 0x800C          # src = 0x800BC700 (tree on disc)
addiu t0, t0, 0xC700
lui   t1, 0x800F          # dst = 0x800F4980 (tree target, outside BSS)
addiu t1, t1, 0x4980
addiu t2, t0, 0x5F0       # end = src + 1520 (tree + trampoline)
loop:
lw    t3, 0(t0)
sw    t3, 0(t1)
addiu t0, t0, 4
addiu t1, t1, 4
bne   t0, t2, loop
nop
j     0x8008E284          # original PC0 (BSS clear)
nop
```

### BSS Clear Routine (at 0x8008E284)
```
lui   v0, 0x800C          # v0 = 0x800BC668 (BSS start)
addiu v0, v0, 0xC668
lui   v1, 0x800F          # v1 = 0x800F4980 (BSS end, exclusive)
addiu v1, v1, 0x4980
loop:
sw    zero, 0(v0)
addiu v0, v0, 4
sltu  v0, v0, v1          # Wait: this is sltu $zero, v0, v1... no
bne   v0, v1, loop        # Actually: 0x0043082B = sltu $zero, v0, v1
                          # 0x1420FFFC = bne $zero, $zero, -4... that can't be right
```

**Correction needed:** The instruction at 0x8008E29C (0x0043082B) is `sltu $zero, v0, v1` which sets $zero=0 always, and 0x1420FFFC is `bne $zero, $zero, loop` which would never branch. This doesn't make sense as a BSS clear loop. Need to re-examine — the actual instruction encoding may be different.

**Re-examination:**
- 0x8008E294: `0xAC400000` = `sw $zero, 0(v0)` — stores zero
- 0x8008E298: `0x24420004` = `addiu v0, v0, 4` — advances pointer
- 0x8008E29C: `0x0043082B` = `sltu $zero, v0, v1` — this is actually `sltu` with rd=$zero... but wait, rd field is (0x0043082B >> 11) & 0x1F = (0x860 >> 11)... let me recalculate.

Actually: 0x0043082B = 0000 0000 0100 0011 0000 1000 0010 1011
- op=0 (SPECIAL), rs=2 (v0), rt=3 (v1), rd=1 (at), sa=0, fn=0x2B (sltu)
- So: `sltu $at, $v0, $v1` — sets $at = (v0 < v1) ? 1 : 0
- 0x1420FFFC: op=5 (bne), rs=1 (at), rt=0 (zero), imm=0xFFFC (-4)
- So: `bne $at, $zero, loop` — branches back if v0 < v1

**BSS clear confirmed:** Zeros 0x800BC668 to 0x800F4980 (exclusive), 4 bytes at a time.

### Key Finding
- BSS clear range: **0x800BC668 to 0x800F4980** (exclusive)
- Tree dest (0x800F4980): **OUTSIDE** BSS range ✓
- Trampoline dest (0x800F4F48): **OUTSIDE** BSS range ✓
- Copy routine (0x800BCCF0): **INSIDE** BSS range — but runs first, then BSS clears it (fine, it's done)
- Tree on disc (0x800BC700): **INSIDE** BSS range — BSS clears it after copy (fine, copy already happened)

### The Paradox
The copy routine runs as PC0, copies tree+trampoline to 0x800F4980+ (outside BSS), then jumps to BSS clear. BSS clear should NOT touch 0x800F4980+. Yet DuckStation reports the trampoline at 0x800F4F48 is empty.

### Possible Explanations
1. **DuckStation BSS clear uses EXE header BSS fields** (offset 0x30: BSS addr, 0x34: BSS size) rather than the code-level loop. If the EXE header's BSS range extends past 0x800F4980, DuckStation may zero a larger region.
2. **The copy routine itself is being wiped** before it can execute — if DuckStation performs a BSS clear based on EXE header before jumping to PC0.
3. **CD-ROM intercept jump target is wrong**: The `j` instruction at the CD-ROM command write site encodes only 26 bits. The decoded target was 0x000F4F48 instead of 0x800F4F48 — the high 4 bits come from the PC's upper nibble. If the PC at that point isn't 0x800xxxxx, the jump goes to wrong address.

### CD-ROM Intercept Bug Found
```
CD-ROM intercept at file 0x80F64: 0x0803D3D2
  Jump target: 0x000F4F48 (should be 0x800F4F48)
```
The `j` instruction encodes `(target >> 2) & 0x3FFFFFF`. The high 4 bits come from `PC & 0xF0000000`. If PC=0x800xxxxx, the target becomes 0x800F4F48. This should work correctly at runtime since the code runs in the 0x800xxxxx segment. **Not a bug** — the disassembler just doesn't know the runtime PC.

## Root Cause Found: Game Data Section Collision

### EXE Header BSS Fields
- BSS addr: 0x801FFFF0, BSS size: 0 (both v32 and DW7 original)
- DuckStation does NOT use header BSS fields for clearing — the BSS clear is purely code-level

### Game Data Section at 0x800F4980+
Static analysis of the original DW7 EXE found multiple references to addresses in the 0x800F4980+ range:
- 0x800F4930: DMA buffer (4 refs with `sw $zero` nearby — init code)
- 0x800F4950: Game variable (function call argument)
- 0x800F4954: Game variable (function pointer table)
- 0x800F49A8: Indexed buffer access (`lhu` with computed offset)
- 0x800F48A0: Large buffer (multiple refs, 0x80 byte stride)

**0x800F4980 is NOT "empty space after BSS" — it's the start of the game's initialized data section.** Game init code writes to these addresses during startup, overwriting our relocated tree and trampoline.

## Fix Applied: v33 — Tree Relocated to 0x80100000

### Change
- `TREE_RAM_ADDR` changed from `0x800F4980` to `0x80100000` in `build_frank_v31()`
- 0x80100000 is high RAM, ~1MB above the game's data section, ~1MB below the stack (0x801FFFF0)
- No game code references this address range

### v33 Disc Layout
- Tree target: 0x80100000 (1480 bytes)
- Trampoline target: 0x801005C8 (40 bytes)
- Tree+tramp end: 0x801005F0
- Safe margin to stack: 1,047,040 bytes
- CD-ROM intercept: `j 0x801005C8` (verified correct)
- Copy routine: src=0x800BC700, dst=0x80100000, payload=1520 bytes (verified)
- BSS clear (0x800BC668-0x800F4980): does NOT touch 0x80100000+
- Game data (0x800F4980-0x800F9E54+): does NOT touch 0x80100000+

### Verification
- `verify_v33.py`: All addresses verified correct
- CyberGrime smoke test: COMPLETE, no crash, 5M instructions, 8 VBlanks, PC=0x80096650

### Files
- Disc: `dq4_frankenstein_v33.bin` + `dq4_frankenstein_v33.cue` (711,567,024 bytes)
- Builder: `translation-tools/frankenstein_builder.py` (profile `frank_v33`)
- Verification: `cybergrime/verify_v33.py`
- Telemetry: `cybergrime/telemetry/telemetry_v33_smoke.json`

## Next Step
Test `dq4_frankenstein_v33.bin` in DuckStation (full GPU implementation) for DQ4 title screen / Enix logo.

## Key Files
- `cybergrime/verify_v32_copy.py` — v32 disc verification script
- `cybergrime/find_bss_clear.py` — BSS clear pattern search
- `cybergrime/find_bss_clear2.py` — Extended BSS clear search
- `cybergrime/check_bss_patch.py` — BSS patch corruption check
- `translation-tools/frankenstein_builder.py` — Builder with `append_reloc_payload` (lines 348-484)
- Disc: `dq4_frankenstein_v32.bin` + `dq4_frankenstein_v32.cue`
