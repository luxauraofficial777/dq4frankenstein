# Frankenstein Build QA — Session Diagnosis Jul 22, 2026

## Executive Summary

The Frankenstein build pipeline is **verified correct** — all patches persist to disc.
The runtime crash is caused by **162 HBD blocks that fail re-encoding**, leaving DQ4-format
data in the disc image that the DW7 engine cannot parse, producing garbage pointers in BSS
that crash the first table walk.

This is the **same root cause** across every build variant (v12 through v44). The specific
crash PC changes, but the failure mode is always: DW7 engine mounts HBD → populates BSS
file table from unconverted DQ4 blocks → walks table → hits invalid pointer → crash.

---

## Build Pipeline Status: VERIFIED

All patches confirmed present on disc via sector-by-sector reads:
- 11 disc-check patches (surgical, no cd_init bypass)
- BSS clear at 0x800BCCC8
- Hybrid Huffman tree (1480 bytes at file offset 0xA5000, signature 02800480)
- 23 MIPS LUI/ADDIU tree reference pairs → 0x800BC700
- LBA refs: 0 refs to 354, 2 refs to 355
- C++ and Python builders produce identical output (0 byte diff)
- EDC/ECC regeneration: SUCCESS

## v44 DuckStation Smoke Test Results

### What Works
- BIOS POST sequence completes (0F → 0E → 01-07 → 08 → 09)
- Kernel initialized
- CD-ROM Init succeeds (v44 removed the cd_init_force_success stub)
- CD-ROM reads disc sectors: LBA 166, 168, 172, 173, 174 (SYSTEM.CNF, EXE, HBD start)
- Setloc/SeekL/ReadN commands complete without FIFO errors
- No crash for 100+ seconds of emulation
- Game recognized as SLUS-01206 (Dragon Warrior VII Disc 1)

### What Fails
- At ~60 seconds, `E(UnknownReadHandler): Invalid word read` errors begin
- Crash PC: `0x800761DC` (ADDIU v1, v1, 1 — inside a table-walking loop)
- Invalid addresses increment by 0x10: 0x8084CDA8, 0x8084CDB8, 0x8084CDC8...
- These addresses are **outside PSX main RAM** (0x80000000-0x801FFFFF)
- The loop reads 16-byte entries from a pointer table at `s2`, which is garbage

## Root Cause Analysis

### The Crash Chain

```
HBD mount routine (0x8002EC2C)
  → reads DQ4-format HBD data from disc
  → calls block header reader (0x80042A04)
  → block header reader calls 0x8002F238 (file table lookup)
  → 0x8002F238 reads from BSS at 0x800F2228 (file table)
  → BSS populated with garbage because 162 blocks weren't converted
  → file table lookup returns pointer to 0x8084CD9C (invalid)
  → crash function at 0x80076050 walks the table
  → LW at 0(a2) reads from invalid address → UnknownReadHandler
```

### The HBD Re-encoder Failure

From the v44 build telemetry:

```
blocks_processed:   1810
blocks_converted:   553
blocks_reencoded:   391
blocks_reenc_fail:  162        ← THESE ARE THE PROBLEM
subblock_swaps:     1590
errors:             3
Validation: FAIL
```

Validation failures (DQ4 blocks still in native format):
- File 220, folder sector 8440: **type 32** (unconverted)
- File 269, folder sector 18745: **type 32**
- File 421, folder sector 18745: **type 32**
- File 111, folder sector 40098: **type 32**
- File 31, folder sector 42660: **type 46**
- File 167, folder sector 42660: **type 32**
- File 919, folder sector 42660: **type 46**

**Block types 32 and 46 are not handled by the C++ HbdReencoder.**
These types likely have different header structures or compression schemes
that the re-encoder's conversion logic doesn't recognize.

### Why v44's CD-ROM Fix Didn't Solve This

v44 fixed the CD-ROM init issue (removed cd_init_force_success stub, removed stall bypass).
This allowed the game to boot further than any previous build — past BIOS, past CD-ROM init,
past initial sector reads. But the HBD re-encoder failure is a **separate, upstream problem**
that was always there, just masked by the earlier CD-ROM crash.

---

## Key Disassembly Findings

### Crash Function (0x80076050)
- Function entry: SW ra, 44(sp) at 0x80076050
- Calls 0x80075430 (file table lookup) at 0x80076160
- Return value (v0) stored in s2 at 0x80076168
- s2 is the base pointer for the table walk
- If s2 is garbage (from unconverted DQ4 blocks), the walk crashes

### File Table Lookup (0x80075430)
- Reads from BSS at 0x800F2228 (LUI 0x800F + offset)
- BSS is populated at runtime by the HBD mount routine
- Searches file entries by comparing type/flag fields
- Returns pointer to matching entry, or NULL

### Block Header Reader (0x80042A04)
- Takes 4 parameters (a0-a3), shifted right by 16 (SRA shamt=16)
- Calls 0x8002F238 (checks 0x800E0550 for file system state)
- Calls 0x8010BCF8 (heap-allocated function, likely decompressor)
- This is where DQ4 vs DW7 format differences cause silent parse failures

### HBD Mount Routine (0x8002EC2C)
- Calls 0x80048724 (initial HBD scan)
- Reads from 0x800F76E4 (HBD metadata)
- Processes folder/file structures
- Populates BSS file table that downstream code walks

---

## The Decision Point

Two paths to fix the 162 failed blocks:

### Path A: Fix the Re-encoder (Recommended)
- The C++ `HbdReencoder` needs to handle block types 32 and 46
- These types have different header structures that the current converter skips
- If all 162 blocks convert successfully, validation passes
- BSS gets populated with valid DW7-format entries
- The table walk at 0x800761DC receives valid pointers
- **No runtime patches needed** — the engine gets data it can natively parse
- Risk: types 32/46 may have fundamentally different compression that's hard to convert

### Path B: Runtime Shim
- Patch the block header reader at 0x80042A04 to detect DQ4 types 32/46
- Add a branch: if type == 32 or 46, use DQ4 parse logic instead of DW7
- This means the DW7 engine natively handles DQ4 blocks at runtime
- More complex: requires MIPS patch code, register preservation, branch trampolines
- Handles ALL DQ4 block types, not just the known failures
- Risk: more moving parts, harder to debug, potential for new crash vectors

### Path C: Both
- Fix the re-encoder for types 32/46 (reduces 162 failures to ~0)
- Add runtime shim as fallback for any residual unconverted blocks
- Belt and suspenders, but doubles the work

---

## Huffman Tree Compatibility (DQ4 vs DW7)

| Aspect | DQ4 | DW7 |
|--------|-----|-----|
| Root ID calc | (value & 0x7FFF) + 1 | value & 0x7FFF (no +1) |
| Offset B | (len + 2) / 2 - 2 | ((len + 2) / 2 - 2) & ~1u (even-aligned) |
| Tree access | Per-block via parameter | Global hardcoded (LUI/ADDIU) |
| Block types | 23, 24, 25, 27, 32, 46 | 19, 40, 42 |
| Global tree | N/A | EXE@0x96FA0, 744 bytes, 186 leaves |

The hybrid Huffman tree (appended at 0xA5000, RAM 0x800BC700) bridges the root_id
and offset_b differences. The 23 MIPS LUI/ADDIU reference pairs redirect the DW7
decompressor to use this hybrid tree. **This part works.**

The remaining incompatibility is **block types 32 and 46** — the re-encoder doesn't
know how to convert their headers from DQ4 format to DW7 format.

---

## File Inventory

### Build Artifacts
- `dq4_frankenstein_v44.bin` / `.cue` — Current build target
- `translation/SLUSP012.06` — DW7 EXE (source for patching)
- `DW7D1/DW7D1.bin` — DW7 disc (base image)
- `translation-tools/frankenstein_builder.py` — Python build system (v44 profile)

### Diagnostic Tools
- `verify_build.py` — Sector-by-sector patch verification (FIXED)
- `audit_macro.py` — Full disc audit (FIXED)
- `test_write_all_sectors.py` — Raw sector write test
- `test_edcre_impact.py` — EDC/ECC preservation test

### Key Source Files
- `frankenstein_build.cpp` — C++ builder (write_sector_user, patching, HBD re-encode)
- `translation-tools/frankenstein_builder.py` — Python builder (build_frank_v44)
- `cybergrime/huffman_compat_shim.asm` — MIPS shim for dual-layer Huffman (designed, not integrated)
- `cybergrime/huffman_compat_hook.c` — C implementation of Huffman hook
- `cybergrime/reencode_pipeline.py` — Pre-processing pipeline for HBD blocks

### Study Documents
- `study/July Week 3/HBD_DECOMPRESSION_ANALYSIS_Jul17.md` — DQ4 vs DW7 block format comparison
- `study/July Week 3/DUAL_FORK_ROADMAP_Jul17_2026.md` — Key MIPS addresses and patch strategy
- `study/July Week 3/UNIFIED_PRODUCT_ROADMAP_Jul17_2026.md` — Approach A/B/C analysis
- `study/RENDER_ENGINE_COMPARISON.md` — Cross-title text renderer comparison

---

## Key Addresses

| Address | Description |
|---------|-------------|
| 0x8002EC2C | HBD mount routine entry |
| 0x80042A04 | Block header reader (real entry, not 0x800429F0) |
| 0x8002F238 | File table lookup (reads BSS at 0x800F2228) |
| 0x80075430 | File entry search (returns pointer into file table) |
| 0x80076050 | Crash function (walks file table, calls 0x80075430) |
| 0x800761DC | Crash PC (ADDIU v1, v1, 1 inside table walk loop) |
| 0x80073670 | Huffman decompressor |
| 0x800BC700 | Hybrid Huffman tree (RAM address) |
| 0x800BCCC8 | BSS clear start |
| 0x800F2228 | BSS file table (populated from HBD at runtime) |
| 0x800F2308 | BSS file table (zeroed during init) |

---

## Next Steps

1. **Examine the C++ HbdReencoder** to understand why types 32 and 46 fail
2. **Analyze DQ4 block types 32 and 46** header format vs DW7 expected format
3. **Implement conversion logic** for types 32/46 in the re-encoder
4. **Rebuild v44** and check if validation passes
5. **If validation passes**, run DuckStation smoke test
6. **If game boots past the table walk**, proceed to gameplay smoke test
7. **If gameplay is stable**, generate XDelta3 patch and prepare release

---

## Session Meta

- Date: Jul 22, 2026
- Build: v44 (surgical disc-check, no cd_init bypass)
- DuckStation: 0.1-11580-ge7f2f101c [dev]
- BIOS: SCPH-5501 (v3.0 11-18-96 A)
- Status: **Blocked on HBD re-encoder types 32/46**
- Pattern: Same root cause across all builds (v12-v44) — unconverted DQ4 blocks
