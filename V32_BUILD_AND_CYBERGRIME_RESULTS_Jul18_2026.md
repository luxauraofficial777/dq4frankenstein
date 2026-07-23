# v32 Frankenstein Build — EXE/HBD Overlap Fix + CyberGrime Pass

**Date:** Jul 18, 2026  
**Build:** dq4_frankenstein_v32.bin  
**Status:** CyberGrime COMPLETE (no crash), DuckStation test pending

---

## Root Cause of v31 Crash

v31 used `append_hybrid_tree` with `target_ram_addr=0x800F4980`, padding the EXE to 909,312 bytes (444 sectors, LBA 24–467) to place the hybrid tree directly at RAM `0x800F4980`. But HBD starts at LBA 355, and the HBD write overwrites sectors 355–467, destroying the tree and trampoline data.

**Evidence:**
- Tree signature (`02800480...`) not found anywhere in the EXE on disc
- Trampoline area at `0x800F4F48` contained random data, not MIPS instructions
- Crash at PC=0x800F4F68 (inside trampoline area) — branch to unmapped `0x8FB6203C`
- 189,421 byte differences between expected EXE and disc EXE
- First difference at offset 0x1D (within the first EXE sector, likely EDC/ECC related)

## Fix Applied

Switched `build_frank_v31` in `frankenstein_builder.py` to use `append_reloc_payload` instead of `append_hybrid_tree` + `append_cdrom_trampoline`.

**How it works:**
1. Tree (1480 bytes) + trampoline (40 bytes) stored at end of EXE (file offset 0xA5000, RAM 0x800BC700 — inside BSS on disc)
2. 13-instruction copy routine appended after trampoline (RAM 0x800BCCF0)
3. PC0 patched from `0x8008E284` (original) to `0x800BCCF0` (copy routine)
4. Copy routine copies 1520 bytes from `0x800BC700` → `0x800F4980`, then jumps to original PC0
5. BSS clear runs after copy, wiping the on-disc copies at `0x800BC700` but the relocated copies at `0x800F4980` survive (outside BSS end)
6. CD-ROM intercept patched to `j 0x800F4F48` (trampoline at relocated address)

**Also fixed:** Off-by-one buffer checks in `append_reloc_payload`:
- Trampoline: `if tramp_file_off > len(exe)` → `if tramp_file_off + 48 > len(exe)`
- Copy routine: `if copy_file_off > len(exe)` → `if copy_file_off + 56 > len(exe)`

## v32 Disc Layout

| Component | LBA | Size | RAM Address |
|-----------|-----|------|-------------|
| EXE | 24 | 677,888 bytes | 0x80017F00 (load) |
| HBD | 355 | 319,436,800 bytes | — |
| Tree (on disc) | within EXE | 1,480 bytes | 0x800BC700 |
| Trampoline (on disc) | within EXE | 40 bytes | 0x800BCCC8 |
| Copy routine | within EXE | 52 bytes | 0x800BCCF0 |
| Tree (relocated) | — | 1,480 bytes | 0x800F4980 |
| Trampoline (relocated) | — | 40 bytes | 0x800F4F48 |

**Patches applied:**
- LBA references: 1 (354→355)
- Sequential sector table: 44
- ABS pointer remapping: 1923 (168 matched + 1755 delta)
- REL pointer remapping: 167
- Disc-check bypass: 10 (semi-surgical, no cd_init)
- Tree ref patches: 24 (lui/addiu → 0x800F4980)
- Reloc payload: 1 (copy routine + tree + trampoline)
- Stop intercept: 1 (j 0x800F4F48)
- EDC/ECC: SUCCESS
- Disc size: 711,567,024 bytes (matches DW7 disc)

## CyberGrime Smoke Test Result

```
Profile: v32_smoke
Disc: dq4_frankenstein_v32.bin
Pass_Status: COMPLETE
Instructions: 5,000,000
Wall_Clock: 7,641ms
VBlanks: 8
Crashed: false
Errors: 0
Last_PC: 0x80096650 (event poll loop)
```

**No crash. No divergence. No errors.** The game boots, initializes GPU, sets up CD-ROM, and enters the event poll loop.

**BIOS call sequence (successful init):**
1. B:fn25 — Event init
2. B:fn91, C:fn10, A:fn114 — Thread/event setup
3. B:fn23 — GPU interrupt enable
4. B:fn53 — DMA config to 0x800F4930 (adjacent to relocated tree at 0x800F4980)
5. A:fn73 — Timer setup
6. B:fn86, A:fn68 — Pad init
7. B:fn8, B:fn12 — CD-ROM commands
8. B:fn23 repeating every ~565K instructions — VBlank polling

**Why it stops:** CyberGrime's GPU stub doesn't generate display interrupts. The game is waiting for a VBlank event that never fires. This is a known emulator limitation, not a disc issue.

## Verification

Disc content verified with `verify_v32.py`:
- Tree at file offset 0xA5000: **matches dw7_hybrid_tree.bin exactly**
- Trampoline at 0xA55C8: first instruction `0x24010008` (addiu at, zero, 8) — **correct**
- Copy routine at 0xA55F0: loads from 0x800BC700, copies to 0x800F4980, jumps to 0x8008E284 — **correct**
- CD-ROM intercept: `j 0x800F4F48` — **correct**
- PC0: `0x800BCCF0` (copy routine) — **correct**
- EXE size: 677,888 bytes — **exactly at max before HBD overlap**

## Key Files

- **Disc:** `dq4_frankenstein_v32.bin` + `dq4_frankenstein_v32.cue`
- **Builder:** `translation-tools/frankenstein_builder.py` (profile `frank_v32`)
- **Verification scripts:** `cybergrime/verify_v32.py`, `cybergrime/inspect_v31_build.py`
- **Telemetry:** `cybergrime/telemetry/telemetry_v32_smoke.json`

## Next Step

Test v32 in DuckStation (full GPU implementation) for DQ4 title screen. DuckStation launched with `dq4_frankenstein_v32.cue` at 1:26 AM Jul 18, 2026.
