# Gate 1: DuckStation Boot Test Results — Jul 19, 2026

## Summary

Four disc images tested in DuckStation (0.1-11580-ge7f2f101c) with SCPH-5501 BIOS, 60s each.

| # | Disc | Patches | Result | Key Detail |
|---|------|---------|--------|------------|
| 1 | Retail DW7 (DW7D1.bin) | None | ✅ PASS | Enix logo, character creation works |
| 2 | dq4_frankenstein_v38a | Disc-check bypass only | ❌ FAIL | EXE runs, "Please insert disc 1" |
| 3 | dq4_frankenstein_v38b | Full engine (tree reloc + copy + CD-ROM intercept) | ❌ FAIL | Black screen, hung at 444 FPS |
| 4 | dq4_frankenstein_v37 | Full build | ❌ FAIL | Identical to v38b — black screen hang |

## Detailed Analysis

### Disc 1: Retail DW7 — PASS
- Enix logo displayed
- Reached character creation screen
- Confirms DuckStation + BIOS + DW7 disc image are all valid
- This is the baseline reference

### Disc 2: v38a (disc-check bypass only) — BEST FRANKENSTEIN
- EXE loads and executes normally
- Game runs through BIOS → kernel → EXE load → BSS clear → HBD read
- Reads HBD from LBA 146621 (DW7 HBD region)
- **Reaches disc check logic** — displays "Please insert disc 1"
- This means: DW7 EXE boots, game logic runs, but disc check catches the hybrid disc
- The disc-check bypass patches applied are insufficient

### Disc 3: v38b (full engine patches) — FAIL
- No Enix logo, black screen
- Boot sequence: BIOS → kernel → EXE load → BSS clear (0x800BC668) → CDROM init
- Reads EXE sectors LBA 166-503 normally
- Seeks to LBA 146621 (DW7 HBD), reads ~55 sectors
- Page fault at 0x80200008 (PC 800761E0) — accessing memory above stack top
- After ~15s, stuck in rendering loop at 444 FPS, zero CD-ROM activity
- **Engine patches (tree relocation, copy routine, CD-ROM intercept) break the boot**

### Disc 4: v37 (full build) — FAIL
- Identical failure pattern to v38b
- Same boot sequence, same LBA reads, same page fault at 0x80200008
- Same 444 FPS hang with no CD-ROM activity
- Confirms v37 and v38b have the same engine patch issue

## Root Cause Analysis

### Why v38a works but v37/v38b don't:
- **v38a**: Only disc-check bypass patches applied. DW7 EXE runs unmodified engine code. Game boots, loads HBD, reaches disc check. Fails at disc check because bypass is incomplete.
- **v37/v38b**: Engine patches (tree relocation to 0x80100000, copy routine at PC0=0x800BCCF0, CD-ROM intercept at 0x80F64) actively break the boot sequence. Game hangs during HBD loading, never reaches disc check.

### The page fault at 0x80200008:
- PC 800761E0 is in the DW7 EXE code section
- Address 0x80200008 is ~2MB above the stack top (0x801FFFF0)
- Indicates game code is dereferencing a corrupted pointer, likely caused by the engine patches

### HBD loading issue:
- All Frankenstein discs read from LBA 146621 (DW7 HBD region at 32:34:71)
- None read from LBA 355 (where DQ4 HBD is placed)
- The ISO directory still points to DW7 HBD location, not DQ4 HBD

## Conclusion

**The engine patch approach (tree relocation + copy routine + CD-ROM intercept) is a dead end.** It breaks the boot sequence before the game can even display a logo.

**The viable path forward is v38a-based:**
1. Fix the disc-check bypass (current bypass is insufficient)
2. Fix the ISO directory to point HBD LBA to 355 (DQ4 HBD) instead of 146621 (DW7 HBD)
3. Do NOT apply tree relocation, copy routine, or CD-ROM intercept patches

## Test Environment
- Emulator: DuckStation 0.1-11580-ge7f2f101c [dev]
- BIOS: SCPH-5501 (v3.0 11-18-96 A)
- Console region: NTSC-U/C
- GPU: D3D11, Intel HD Graphics 620
- All discs: MODE2/2352, 711,567,024 bytes
