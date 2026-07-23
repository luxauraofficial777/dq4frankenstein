# Session: v41 Frankenstein Build — Jul 20, 2026

## Objective

Deliver a working Frankenstein binary build that addresses the CD-ROM stall (HF-005) and integrates DQ4 content on the DW7 engine, based on Gate 1 DuckStation test results.

## Gate 1 Test Results (Context)

Four discs tested in DuckStation (Jul 19):

| Disc | Patches | Result |
|------|---------|--------|
| Retail DW7 | None | PASS — boots to character creation |
| v38a (disc-check only, 10 patches) | Semi-surgical disc-check | FAIL but BEST — "Please insert disc 1" |
| v38b (full engine: tree reloc + copy + CD-ROM intercept) | All engine patches | FAIL — black screen, page fault 0x80200008 |
| v37 (full build) | All patches | FAIL — identical to v38b |

### Key Findings from Gate 1
- **v38a is the best Frankenstein build** — DW7 EXE boots and runs with disc-check bypass only
- **Engine patches break the boot** — tree relocation to 0x80100000, copy routine at PC0=0x800BCCF0, and CD-ROM intercept at 0x80F64 cause page faults and black screens
- **v38a's disc-check bypass was incomplete** — used semi-surgical (10 patches, no cd_init), still showed "Please insert disc 1"
- **HBD LBA issue** — all discs read from LBA 146621 (DW7 HBD region), not LBA 355 (DQ4 HBD)

## v41 Build Strategy

Based on Gate 1 analysis, v41 applies ONLY patches proven safe by v38a, plus the missing pieces:

### Applied (9 patch groups):
1. **FULL disc-check bypass** (11 patches including cd_init_force_success) — v38a used only 10 (no cd_init), still got "insert disc 1"
2. **LBA references** patched 354→355 (1 patch)
3. **Sequential sector table** patched (44 patches)
4. **ABS/REL folder pointer remapping** — 168 matched ABS, 1755 unmatched ABS delta, 3 matched REL
5. **Hybrid tree appended in-place** — 1480B tree at RAM 0x800BC700, within t_size
6. **MIPS tree refs** patched to in-place tree address (24 patches)
7. **BSS clear start** moved from 0x800BC668 → 0x800BCCC8 (skip tree region)
8. **BSS clear end** kept at original 0x800F4980 (full clear after tree)
9. **DQ4 HBD swap** at LBA 355 (319,436,800 bytes) + ISO directory update

### NOT Applied (proven to break boot in v38b):
- NO copy routine at PC0 (PC0 stays 0x8008E284)
- NO CD-ROM Stop intercept / trampoline
- NO tree relocation to 0x80100000
- NO PC0 redirect

## On-Disc Verification

| Check | Expected | Actual | Status |
|-------|----------|--------|--------|
| PC0 | 0x8008E284 | 0x8008E284 | OK |
| Load address | 0x80017F00 | 0x80017F00 | OK |
| t_size | 0xA5000 (675,840B) | 0xA5000 | OK |
| BSS start | 0x800BCCC8 | 0x800BCCC8 | OK |
| BSS end | 0x800F4980 (original) | 0x800F4980 | OK |
| cd_init patch | 0x24020001 | 0x24020001 | OK |
| Tree @ file 0xA5000 | Present | Present | OK |
| ISO dir HBD LBA | 355 | 355 | OK |
| ISO dir HBD size | 319,436,800 | 319,436,800 | OK |
| EDC/ECC | SUCCESS | SUCCESS | OK |
| Disc size | 711,567,024 | 711,567,024 | OK |

## Build Output

- **Binary**: `dq4_frankenstein_v41.bin` (711,567,024 bytes)
- **CUE sheet**: `dq4_frankenstein_v41.cue`
- **Builder**: `translation-tools/frankenstein_builder.py --profile frank_v41`

## Patch Summary

| Patch | Count | Details |
|-------|-------|---------|
| Disc-check bypass | 11 | Full set including cd_init_force_success |
| LBA references | 1 | 354→355 |
| Sequential sector table | 44 | Range 300-400 shifted +1 |
| ABS matched | 168 | DW7→DQ4 folder sector remap |
| ABS unmatched delta | 1755 | +1 LBA delta for HBD shift |
| REL matched | 3 | DW7→DQ4 relative offset remap |
| Tree refs | 24 | LUI/ADDIU pairs → 0x800BC700 |
| Hybrid tree | 1 | 1480B appended at file 0xA5000 |
| BSS clear start | 1 | 0x800BC668 → 0x800BCCC8 |

## Key Addresses

| Symbol | Value | Notes |
|--------|-------|-------|
| EXE LBA | 24 | Sector on disc |
| HBD LBA | 355 | DQ4 HBD data |
| EXE load address | 0x80017F00 | PSX EXE header |
| PC0 | 0x8008E284 | Original entry point (unchanged) |
| Tree RAM | 0x800BC700 | In-place, within t_size |
| BSS clear start | 0x800BCCC8 | After tree (was 0x800BC668) |
| BSS clear end | 0x800F4980 | Original (unchanged) |
| Thread entry | 0x800D9E80 | Inside BSS range (zeroed by clear) |
| DQ4 HBD size | 319,436,800 bytes | 304MB |

## Differences from v38a (Best Previous Build)

| Feature | v38a | v41 |
|---------|------|-----|
| Disc-check patches | 10 (semi-surgical, no cd_init) | 11 (full, including cd_init) |
| HBD data | DW7 (pure, LBA 354) | DQ4 (LBA 355) |
| ISO dir HBD LBA | 354 (original DW7) | 355 (DQ4) |
| LBA references | Not patched | 354→355 |
| Sequential sector table | Not patched | Patched (+1) |
| ABS/REL pointers | Not patched | 168 matched + 1755 delta |
| Hybrid tree | Not appended | Appended at 0x800BC700 |
| Tree MIPS refs | Not patched | 24 LUI/ADDIU pairs |
| BSS clear start | 0x800BC668 (original) | 0x800BCCC8 (skip tree) |
| BSS clear end | 0x800F4980 (original) | 0x800F4980 (unchanged) |
| Copy routine | None | None |
| CD-ROM intercept | None | None |
| PC0 | 0x8008E284 | 0x8008E284 |

## Differences from v38b (Failed Build)

| Feature | v38b | v41 |
|---------|------|-----|
| Copy routine | Yes (PC0=0x800BCCF0) | NO (PC0 unchanged) |
| CD-ROM Stop intercept | Yes (trampoline) | NO |
| Tree relocation | 0x80100000 (copy routine) | In-place at 0x800BC700 |
| PC0 redirect | Yes (→ copy routine) | NO (original 0x8008E284) |
| Disc-check | Semi-surgical (10) | Full (11, with cd_init) |
| HBD data | DW7 (pure, shifted to 355) | DQ4 (at 355) |

## Known Risks

1. **cd_init_force_success** — v38a deliberately skipped this patch because it "destroys CD-ROM hardware init." If the CD-ROM drive fails to initialize, the game may stall when trying to read HBD data. This is the primary risk of v41.

2. **DQ4 HBD format** — DQ4 HBD uses per-block Huffman trees (tree_end≠0, text_end≠0), while DW7 uses a global tree. The hybrid tree at 0x800BC700 may not be compatible with DQ4 block headers that expect per-block trees. If text decoding fails, the game may crash or display garbled text.

3. **ABS pointer delta** — 1755 unmatched ABS pointers received +1 LBA delta. Some of these may point to DW7-specific data that no longer exists in the DQ4 HBD, causing reads from wrong sectors.

4. **Thread entry zeroing** — BSS clear still zeros 0x800D9E80 (thread entry). v38a proved this doesn't prevent initial boot, but the CD-ROM thread may not start, potentially causing a stall later in the boot sequence.

## Next Steps

1. **Test v41 in DuckStation** — Load `dq4_frankenstein_v41.bin` with `dq4_frankenstein_v41.cue`, SCPH-5501 BIOS, 60s observation
2. If "Please insert disc 1" persists → disc-check bypass still incomplete, investigate remaining disc-check code paths
3. If black screen → cd_init_force_success broke CD-ROM init; revert to semi-surgical and find alternative disc-check bypass
4. If boots but garbled text → DQ4 HBD per-block tree format incompatible with global tree; need HBD re-encoding
5. If boots with readable text → SUCCESS; proceed to golden path QA

## Files Modified

- `translation-tools/frankenstein_builder.py` — Added `build_frank_v41` function and profile entry
- `dq4_frankenstein_v41.bin` — Built disc image (711,567,024 bytes)
- `dq4_frankenstein_v41.cue` — CUE sheet

## Build Command

```
python translation-tools\frankenstein_builder.py --profile frank_v41
```
