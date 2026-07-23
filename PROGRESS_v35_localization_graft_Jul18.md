# v35 Localization Graft — Progress Notes (Jul 18, 2026)

## What Was Done

### New Build Profile: `frank_v35` (Localization Graft)
Added to `translation-tools/frankenstein_builder.py` (lines 2320–2693):

**Five system-level adjustments:**

1. **Archive Identity Sync**
   - `patch_hbd_filename_q71()`: Patches EXE `hbd1ps1d.w71`→`q71`, `w72`→`q72`
   - `update_volume_identifier()`: Patches HBD offset 0x740 with `hdb1ps1d.q71`
   - `update_iso_dir_hbd_name()`: Updates ISO 9660 dir entry to `HBD1PS1D.Q71;1`

2. **LBA Index Reconstruction** (reused from v31)
   - `patch_lba_references()`: Broad scan for all 32-bit values == 354 → 355
   - `patch_sequential_sector_table()`: 16-bit table at 0xA39AA–0xA3A02
   - `remap_pointers_matched_only()`: Matched ABS/REL + delta for unmatched

3. **Dynamic Huffman Decoding** (data-level approach)
   - `reencode_hbd_blocks()`: Imports `reencode_dq4_blocks.py`, runs round-trip verification, then full re-encoding of all DQ4 text blocks to use global hybrid tree
   - Sets `hts=24, treeEnd=0, textEnd=0` per block (forces DW7 global-tree path)
   - `patch_mips_tree_refs()`: Patches 24 LUI/ADDIU pairs to relocated tree address
   - `append_reloc_payload()`: Tree + trampoline + copy routine appended to EXE

4. **Memory Allocation Management**
   - `update_hbd_block_headers()`: Updates master block total data length at offset 0x08
   - Pads HBD to 2048-byte sector alignment

5. **Audio Mapping**
   - `patch_audio_bank_sizes()`: Scans for `SEDB`/`PBAS` sound header signature
   - Patches bank size fields at offsets 0x0C/0x18/0x24/0x30

### Profile Registration
- `frank_v35` registered in `PROFILES` dict at line 2796
- Default output: `dq4_frankenstein_v35.bin`
- Syntax verified, profile listing confirmed

## Critical Issue Carried Over from v33

**v35 still uses `TREE_RAM_ADDR = 0x80100000`** — the same unmapped gap that caused v33's boot failure (`Instruction read failed at PC=0x801005C8`).

Per the v33 debug notes:
- EXE load range: `0x80017F00`–`0x800BCF00` (t_size=0xA5000)
- `0x80100000` falls in an unmapped gap between EXE load range end and dynamic heap start (`0x80138000`)
- DuckStation may not map RAM outside the EXE's declared `t_addr`/`t_size` range

### Fix Options (Priority Order)
1. **Expand t_size to cover 0x80100000**: Set t_size = 0x80100000 + 1520 - 0x80017F00 = 0xE8700. This makes DuckStation map the full range. The extra space (0xA5000→0xE8700) is ~270KB of zeros (BSS region on disc).
2. **Relocate tree to within existing t_size**: Place tree at `0x800BC700` (end of current t_size). BSS clear wipes it, but the copy routine runs BEFORE BSS clear and copies to a safe location. Problem: where is "safe"? BSS clear range is `0x800BC668`–`0x800F4980`.
3. **Place tree after BSS clear range**: `0x800F4980` is after BSS clear but within game data section. Risk of collision with game variables.
4. **Hook post-BSS-clear**: Instead of PC0→copy→BSS_clear, do BSS_clear→copy→game_init. This requires finding the post-BSS-clear entry point.

### Recommended Fix: Option 1 (Expand t_size)
- Change `compute_tree_addr` call to use `target_ram_addr=None` (appends after text segment)
- OR explicitly set t_size in EXE header to cover tree+trampoline
- The copy routine then copies from on-disc BSS region to the expanded area
- DuckStation will map the full range because t_size covers it

## Files Modified
- `translation-tools/frankenstein_builder.py`: +375 lines (helpers + build_frank_v35 + profile registration)

## v35 Build Result (Jul 18, 2026)

### Fix Applied: In-Place Tree (no 0x80100000)
- **`append_tree_inplace()`**: Appends tree+trampoline directly to EXE text segment
- Tree at RAM `0x800BC700` (file `0xA5000`), trampoline at `0x800BCCC8`
- **`patch_bss_clear_start()`**: BSS clear start moved from `0x800BC668` to `0x800BCCF0`
- No copy routine, no `0x80100000` — everything within DuckStation's mapped RAM range
- PC0 unchanged (`0x8008E284`) — BSS clear runs normally, just skips tree region
- EXE: 677,888 bytes (exactly max before HBD overlap) ✓

### Build Output
- File: `dq4_frankenstein_v35.bin` (711,567,024 bytes, matches DW7) ✓
- CUE: `dq4_frankenstein_v35.cue`
- EDC/ECC: SUCCESS ✓
- Patches: name=2, lba=1, seq=44, abs=1923(168 matched+1755 delta), rel=167, disc=10, tree=24, headers=1, audio=4, tree_inplace=1, stop_intercept=1

### Issue: Re-encoding Skipped
Round-trip verification failed — blocks had "Invalid DQ4 tree length: 0". The `reencode_dq4_blocks.py` `verify_roundtrip()` function found 2 text blocks with per-block trees but both had tree length=0. HBD is unre-encoded, so text will be garbled. The disc should still boot and reach the title screen.

### Next Steps
1. **Test `dq4_frankenstein_v35.bin` in DuckStation** — verify boot reaches title screen
2. **Debug re-encoding**: Investigate why `verify_roundtrip()` finds blocks with tree length=0. The block map may have stale offsets or the HBD extraction may be reading wrong sectors.
3. **Fix re-encoding**: Once round-trip passes, re-run build to get translated text
4. **Verify audio bank patches**: The `SEDB` signature was found at offset `0x77325C` — bank values look like garbage (`0xBD955588` etc.), may need different signature search
