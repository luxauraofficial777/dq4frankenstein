# Black Screen Debug — Jul 18, 2026

## Status: In Progress — Root Cause Identified

## Summary

The Frankenstein v34 disc boots successfully (no instruction read errors, no invalid byte reads), but displays a black screen with **GPU: 0.00** in PerfMon logs. The DW7 EXE runs (CPU: ~7%, 511 FPS), reads CD-ROM sectors, but never issues GPU commands. The game enters a spin loop after reading zeros at LBA 146471.

## Key Findings

### 1. HBD Block Format is NOT the Issue

The heart v17 HBD blocks are **already in DW7 format** (`hts≈1385`, `treeEnd=0`, `textEnd=0`), not DQ4 format (`hts=24`, `treeEnd>0`, `textEnd>0`). Scanning 500 sectors of the heart v17 HBD found:
- **41 DW7-format blocks** (hts≈1385, treeEnd=0, textEnd=0)
- **12 OTHER blocks** (various hts values, treeEnd=0, textEnd=0)
- **1 DQ4-format block** (hts=24, treeEnd=1664, textEnd=1428) — likely an unpatched block

The `dq4_hbd_patcher.py` already uses `DW7ASCII` leaf mapping (0x0260 for 'A', etc.) and builds per-block trees, but the output blocks have DW7-style headers with `treeEnd=0, textEnd=0`.

### 2. MIPS Tree Refs are Correctly Patched

The v34 EXE has all 24 MIPS `lui/addiu` pairs correctly patched from `0x800EF1C8` → `0x80100000`. The copy routine copies the hybrid tree to `0x80100000` (outside BSS clear range `0x800BC668–0x800F4980`). No old refs remain.

### 3. HBD Header Format is Identical

Both DW7 and DQ4 HBDs share the same binary header (first sector is compressed/encrypted data, not a folder table). The second sector is a folder header in both:
- **DW7**: files=1, sectors=2
- **DQ4**: files=1, sectors=3

### 4. ROOT CAUSE: DQ4 HBD is Smaller Than DW7 HBD

The DW7 EXE seeks to HBD relative sector 146116 (LBA 146471), but DQ4's HBD content ends at sector 141196. The DQ4 HBD has **14,778 trailing zero sectors** (padding to fill the 304.6 MB HBD allocation).

- **DW7 HBD**: ~589 MB of content (302,016 sectors)
- **DQ4 HBD**: ~285 MB of content (141,196 non-zero sectors) + 14,778 zero sectors
- **Seek position**: Sector 146,116 — **4,920 sectors beyond DQ4's content**

The DW7 EXE reads zeros at this location, fails to parse a valid folder/block, and enters a spin loop without ever issuing GPU commands.

### 5. ABS Pointer Table Analysis (In Progress)

The ABS pointer table at EXE offset `0x4184` uses **8-byte entries** (not 4-byte). The `remap_pointers_matched_only` function:
- Patches 168 matched entries (DW7 sector → DQ4 sector + HBD_LBA)
- Applies +1 LBA delta to 1755 unmatched entries (val >= 354 and < 0x100000)

The unmatched entries that point to sectors beyond DQ4's content (but within DW7's content) are the likely cause. These need to be remapped to valid DQ4 folders or the game needs to be patched to skip the missing assets.

## CD-ROM Activity (from DuckStation log)

1. Sectors 166–179: ISO directory read
2. Sectors 355–504: HBD header/mount
3. LBA 146471 (MSF 32:34:71): Single sector read → all zeros
4. CPU spin loop, no further CD-ROM reads, no GPU commands

## Architecture

```
Frankenstein v34 Disc Layout:
  LBA 24:    DW7 US EXE (SLUSP012.06) — patched
  LBA 355:   DQ4 HBD (from heart_v17) — 319,436,800 bytes
             ├── Sector 0:     Binary header (compressed)
             ├── Sector 1:     Root folder (files=1, sectors=3)
             ├── Sectors 160+: Text blocks (DW7 format: hts≈1385, treeEnd=0)
             ├── ...
             ├── Sector 141196: Last non-zero content
             └── Sectors 141197–155974: Zero padding (14,778 sectors)

EXE Patches Applied:
  - LBA references: 1 patch (354→355)
  - Sequential sector table: 44 entries shifted +1
  - ABS pointer table: 168 matched + 1755 delta (+1)
  - REL pointer table: 167 matched
  - Disc-check bypass: 9 patches
  - MIPS tree refs: 24 patches (0x800EF1C8 → 0x80100000)
  - Hybrid tree: 1480 bytes at 0x80100000 (via copy routine)
  - Trampoline: CD-ROM Stop intercept
  - PC0: Redirected to copy routine → BSS clear → game init
```

## Next Steps

1. **Map unmatched ABS pointers**: Identify which ABS entries point beyond DQ4 HBD content (sector > 141196 relative). These need to be redirected to valid DQ4 folders or the game code needs to be patched to handle missing assets gracefully.

2. **Check ABS table with 8-byte entries**: Re-run analysis with correct entry size to get accurate sector values.

3. **Identify what the game expects at LBA 146471**: Disassemble the DW7 EXE code that computes this seek address to understand what asset it's trying to load (likely a font, image, or tilemap needed for the Enix logo).

4. **Consider asset injection**: If the missing asset is a font/tilemap, extract it from the DW7 HBD and inject it into the DQ4 HBD at the correct offset.

5. **Alternative: Patch the EXE to skip the missing asset**: If the asset is not critical for initial display (e.g., a late-game asset), patch the seek code to return success without reading.

## Files Created This Session

- `compare_hbd.py` — HBD header comparison between DW7 and DQ4
- `check_seek.py` — Seek location analysis
- `check_abs_ptrs.py` — ABS pointer table analysis (needs fix: 8-byte entries)

## Key Constants

| Constant | Value |
|----------|-------|
| EXE LBA | 24 |
| HBD LBA (Frankenstein) | 355 |
| HBD LBA (DW7 original) | 354 |
| HBD LBA (DQ4 heart_v17) | 362 |
| DQ4 HBD size | 319,436,800 bytes (155,975 sectors) |
| DQ4 last non-zero sector | 141,196 (relative) |
| DQ4 trailing zero sectors | 14,778 |
| DW7 HBD size | 618,563,584 bytes (~302,016 sectors) |
| Hybrid tree size | 1,480 bytes |
| Tree RAM (MIPS refs) | 0x80100000 |
| BSS clear range | 0x800BC668 – 0x800F4980 |
| ABS table offset | 0x4184 |
| REL table offset | 0x511C |
| Table entry size | 8 bytes |
| Max table entries | 6,000 |
| Last CD-ROM read | LBA 146471 (MSF 32:34:71) |
