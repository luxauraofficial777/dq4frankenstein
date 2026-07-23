# FMV Skip Investigation — Jul 19, 2026

## Goal
Identify MIPS `jal` instructions responsible for FMV/STR playback in DW7 EXE to create `fmv_skip.py` for Frankenstein v37 builds.

## Method
Disassembled DW7 EXE (`translation/SLUSP012.06`, base 0x80017F00) across 21 iterations (v1–v21) to trace the STR/XA/MDEC playback call chain.

## Key Findings

### STR/XA Playback Call Chain
```
Event handlers (0x8003DA64, 0x800419B8, 0x80041C30, etc.)
  └─> 0x8008AEF4 (XA/STR playback core) — 148 callers
        ├─> 0x8008B65C (setup)
        └─> 0x800A2188 (BIOS B fn 16 = CdRead)
  └─> 0x8008B32C (XA start) — 5 callers (0x8004F4xx)
        └─> 0x8008AEF4
  └─> 0x8008CAD0 (MDEC init + STR streamer)
        └─> 0x8008CA20 (MDEC init)
```

### Functions Identified
| Address | Role | Callers | Key Calls |
|---------|------|---------|-----------|
| 0x8008AEF4 | XA/STR playback core | 148 | 0x800A2188 (CdRead) |
| 0x8008CAD0 | MDEC init + STR streamer | 1 (0x8003DA64) | 0x8008CA20 (MDEC init) |
| 0x8008B32C | XA start | 5 (0x8004F4xx) | 0x8008AEF4 |
| 0x8008B144 | XA stop | 12 | 0x800A21C8 (CDinit), 0x800A21D8 (CDdone) |
| 0x8008CA20 | MDEC init | 3 | — |
| 0x8008B82C | MDEC init wrapper | 1 (0x8007AE5C) | 0x8008CA20 |
| 0x8008CC48 | STR streamer (dead code) | 0 | 0x8008CA20 |
| 0x80079B88 | STR playback candidate | 0 direct callers | — |

### What Was NOT Found
- No BIOS StVIDEO calls — game handles STR/MDEC internally
- No direct MDEC register (0x1F801020) access in EXE — likely via DMA table at 0x800BB6E8
- 0x8008CC48 and 0x80079B88 have zero callers — dead code or called via computed jumps
- Function table at 0x800BB400 (18 entries) is not the STR dispatch table

### BIOS CD-ROM Wrappers (from v10–v15)
| Address | BIOS Function |
|---------|--------------|
| 0x800A2108 | CdSeek |
| 0x800A2148 | CdRead |
| 0x800A2178 | CdSeekL |
| 0x800A2188 | CdReadL |
| 0x800A21C8 | CDinit |
| 0x800A21D8 | CDdone |

## Patch Strategy
Instead of nopping 148+ `jal` instructions, patch 3 function prologues to `jr ra; nop` (immediate return):

1. **0x8008AEF4** → `jr ra; nop` — skips all XA/STR playback (148 call sites)
2. **0x8008CAD0** → `jr ra; nop` — skips MDEC init + STR streamer
3. **0x8008B32C** → `jr ra; nop` — skips XA start (5 call sites)

This is safe because DQ4 HBD does not contain DW7's STR/XA data files.

## Files Created
- `fmv_skip.py` — Patch script (applies 3 patches to EXE)
- `find_fmv_jals_v1.py` through `find_fmv_jals_v21.py` — Investigation scripts
- `translation/SLUSP012.06.fmv_skip` — Patched EXE output

## Next Steps
- Integrate `fmv_skip.py` into Frankenstein builder (v37 profile)
- Test patched EXE in DuckStation to verify game boots past FMV hang
- If XA audio is needed later, selectively un-patch 0x8008B32C
