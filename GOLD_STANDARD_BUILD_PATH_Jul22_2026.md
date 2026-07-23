# GOLD STANDARD BUILD PATH & RECURSIVE QA LOOP

**Date:** Jul 22, 2026  
**Status:** ACTIVE — Supersedes all legacy build profiles (v12-v43)  
**Authority:** System Directive — Full C++ Native Pipeline  

---

## 1. DEPRECATION NOTICE

All legacy scripts, Python-dependent pipeline wrappers, and fragmented build profiles (v12 through v43) are **deprecated**. We operate exclusively on the native C++ build and verification pipeline.

**Active toolchain:**
- `frankenstein_driver.cpp` / `frankenstein_driver.h` — Driver framework (MipsEmitter, FunctionExtractor, PayloadRemapper, DriverCodegen, ExeBuilder, HbdProcessor, DiscAssembler, FrankensteinDriverBuilder)
- `psx_binary_ops.cpp` / `psx_binary_ops.h` — Low-level binary surgery (ExePatcher, HbdReencoder, IsoImage, HuffTree)
- `frankenstein_driver_test.cpp` — 83-test verification suite

**Compile command:**
```bash
g++ -std=c++17 -o frankenstein_build.exe frankenstein_driver.cpp psx_binary_ops.cpp
g++ -std=c++17 -o frankenstein_driver_test.exe frankenstein_driver_test.cpp frankenstein_driver.cpp psx_binary_ops.cpp
```

---

## 2. THE DEFINITIVE GOLD STANDARD BUILD PATH

A gold-standard build must execute in this exact sequence:

### Phase 1: INPUT VALIDATION

| Input | Description | Required |
|-------|-------------|----------|
| `SLUSP012.06` | Clean DW7 US executable (extracted from DW7 disc at sector 24) | YES |
| `dq4_heart_v17.bin` | DQ4 translated HBD (at sector 362 on DQ4 disc) | YES |
| `dw7_hybrid_tree.bin` | 1480-byte hybrid Huffman tree for DW7 global format | YES |
| `folder_mapping.json` | ABS/REL folder pointer mapping table | YES |
| `edcre.exe` | EDC/ECC regeneration tool | YES |

Verification: All files must exist and be non-empty. HBD size must be ~319,436,800 bytes. Hybrid tree must be 1480 bytes.

### Phase 2: EXE PATCHING (C++ Direct)

1. **Disc-check bypass (10 surgical patches):**
   - Neutralize failure exit points at offsets: `0x6112C`, `0x611D8`, `0x613B8`, `0x614A4`, `0x614F8`, `0x7308C`, `0x61424`, `0x61520`, `0x61360`
   - Patch values: `0x24020001` (li v0,1), `0x00000000` (nop), `0x08022BE9` (j 0x800AFA4), `0x1000000B`/`0x1000000A` (b skip)
   - Neutralize disc-change handler at `0x387CC` (12 bytes: jr ra + nop + nop)
   - **DO NOT** use `cd_init_force_success` — it breaks CD-ROM hardware init

2. **LBA reference update:**
   - Scan EXE for 32-bit values matching old HBA LBA 354
   - Patch to new LBA 355 (delta = +1)
   - Update sequential sector table (16-bit entries at `0xA39AA`)

3. **ABS/REL pointer remapping:**
   - Match folder pointers against `folder_mapping.json`
   - Matched ABS pointers → DQ4 base sector (302115)
   - Unmatched ABS pointers → apply LBA delta
   - Matched REL pointers → remap to DQ4 relative offsets

4. **Hybrid Huffman tree injection:**
   - Append 1480-byte tree to EXE payload
   - Update `text_size` in PS-X EXE header
   - Patch MIPS `LUI`/`ADDIU` tree references to target `0x800BC700`
   - Adjust BSS clear boundaries to skip `0x800BC668`–`0x800BC700` (tree region)

5. **Block type trampoline (at `0x80101000`):**
   - 27 MIPS instructions: type dispatch for 32/39/40/42/46
   - Swaps `type_id` ↔ `count` fields for DQ4 sub-block compatibility
   - Patches JAL at `0x800562E4` → `0x0C040400` (JAL 0x80101000)

6. **CD-ROM stall bypass:**
   - Patch `bne` → `beq` at `0x800986E0` (always branch, skip event-flag timeout)

7. **FMV skip (if configured):**
   - Overwrite post-verify at `0x8008AEF4` with `li v0, 0; jr ra; nop`

### Phase 3: HBD RE-ENCODING & BLOCK CONSERVATION

1. **Parse DQ4 HBD blocks from sector 362**
   - Sector 0: binary header (copy unchanged)
   - Sectors 1+: folder headers (file_count, sector_count, folder_size) + file entries + file data

2. **Transformation A — Block header conversion:**
   - DQ4 format: `hts=24`, per-block Huffman trees, `tree_end`/`text_end` non-zero
   - DW7 format: `hts=1366`, global hybrid tree, `tree_end=0`, `text_end=0`
   - Decode text with DQ4 per-block tree → re-encode with hybrid global tree
   - Rebuild block: 24-byte header + 1342-byte gap + re-encoded text + at_end + dp_data

3. **Transformation B — Sub-block field swap:**
   - For file types 32, 39, 40, 42, 46:
   - `count = flags; flags = 0; type_id = count`
   - This converts DQ4's `flags=count, type_id=type` to DW7's `flags=0, type_id=count`

4. **CRITICAL GATE — Block types 32 and 46:**
   - These are the known blockers (script and message blocks)
   - Must parse correctly through the re-encoder without falling back or corrupting BSS
   - If re-encoding fails for a block, text is copied verbatim (fallback) and `blocks_reencode_failed` increments
   - Validation: `HbdProcessor::validate()` must report zero DQ4 types remaining

5. **Transformation C — LBA relocation:**
   - Delta = `target_lba - source_lba` = `355 - 362` = `-7`
   - Scan for 32-bit values in range `[362, 362 + 155975)` and adjust by delta
   - HBD archive uses relative offsets — LBA scan may find zero hits (this is OK)

### Phase 4: DISC ASSEMBLY

1. Copy base DW7 disc to output `dq4_gold.bin`
2. Extend disc image if HBD doesn't fit (MODE2 Form1 sectors)
3. Write patched EXE at sector 24 (331 sectors, 677,888 bytes)
4. Write re-encoded HBD at sector 355 (~156,075 sectors, ~319MB)
5. Update ISO 9660 directory entries (file sizes for EXE)
6. Update Primary Volume Descriptor (volume space size)
7. Write CUE sheet (`dq4_gold.cue`, MODE2/2352)
8. Regenerate EDC/ECC across entire image using `edcre.exe`

**Output targets:** `dq4_gold.bin` + `dq4_gold.cue`

---

## 3. THE RECURSIVE CIRCULAR QA & DEBUG LOOP

When a build is initiated, the loop does not stop at compilation. The following cycle executes continuously:

```
┌─────────────────────────────────────────────────────────┐
│                    RECURSIVE QA LOOP                     │
│                                                         │
│  1. BUILD                                               │
│     frankenstein_build.exe --profile gold               │
│     --output dq4_gold.bin                               │
│                                                         │
│  2. VERIFY                                              │
│     - EXE header magic ("PS-X EXE")                     │
│     - Patch byte-matches (10 disc-check + trampoline)   │
│     - MIPS tree reference integrity (LUI/ADDIU → 0xBC7) │
│     - Sector PVD limits (HBD doesn't overflow disc)     │
│     - EDC/ECC status (edcre exit code = 0)              │
│     - HBD validation (no DQ4 types remain)              │
│                                                         │
│  3. EMULATE / TEST                                      │
│     - DuckStation headless smoke test                   │
│     - Target: 60-120 second boot + execution window     │
│     - Check: black screen vs. game init                 │
│     - Check: BIOS call diversity (not just fn=23)       │
│     - Check: GPU commands sent                          │
│     - Check: pad initialization                         │
│                                                         │
│  4. DIAGNOSE                                            │
│     - Binary verification failure:                      │
│       → Isolate exact patch/re-encoder function         │
│       → Fix in psx_binary_ops.cpp or frankenstein_driver│
│                                                         │
│     - Emulation crash (e.g., UnknownReadHandler):       │
│       → Capture crash PC                                │
│       → Trace BSS pointer corruption                    │
│       → Identify unhandled block type                   │
│       → Map to source function in C++                   │
│       → Patch C++ source                                │
│                                                         │
│  5. REBUILD                                             │
│     - Apply code modifications                          │
│     - Recompile: g++ -std=c++17 ...                     │
│     - Restart loop immediately                          │
│                                                         │
└──────────────────┬──────────────────────────────────────┘
                   │
                   └──→ LOOP UNTIL ZERO FAULTS
```

### Diagnostic Quick Reference

| Symptom | Likely Cause | Fix Location |
|---------|-------------|--------------|
| Black screen, only fn=23 BIOS calls | HBD not parsed, game stuck in VBlank | `HbdProcessor` / block type trampoline |
| Crash at `0x800E23C4` (jr $zero) | BSS corruption from HBD DMA overwriting BIOS vectors | BSS clear boundaries in `DriverCodegen` |
| Crash at `0x800761DC` (table walk) | Unhandled block type in folder parsing | `HbdReencoder::process_hbd` in `psx_binary_ops.cpp` |
| CD-ROM stall at `0x8009885C` | Event-flag timeout loop | CD-ROM stall bypass patch |
| Huffman decode garbage | Tree incompatibility (DQ4 per-block vs DW7 global) | `HuffTree` parse/encode in `psx_binary_ops.cpp` |
| FMV hang | Disc-swap trigger from FMV playback | FMV skip stub at `0x8008AEF4` |

### Known Critical Addresses

| Address | Description |
|---------|-------------|
| `0x80017F00` | EXE load address (both DW7 and DQ4) |
| `0x8008E284` | DW7 PC0 entry point |
| `0x8002EC2C` | DW7 HBD mount routine |
| `0x80042A04` | DW7 block header reader |
| `0x800562E4` | Divergence point (JAL to trampoline) |
| `0x80073670` | Huffman decompressor |
| `0x80078FF4` | Disc check function |
| `0x800506CC` | Disc change handler |
| `0x800986E0` | CD-ROM stall loop |
| `0x8009885C` | Main event loop |
| `0x800BC668` | BSS clear start |
| `0x800BC700` | Hybrid tree target RAM |
| `0x800F4980` | BSS clear end |
| `0x800D9E80` | Thread entry point |
| `0x800E3D90` | Disc number variable |
| `0x800EF1C8` | Original DW7 Huffman tree |
| `0x80101000` | Trampoline base (safe high RAM) |

---

## 4. BUILD PROFILES

| Profile | EXE Source | HBD Source | Tree Mode | Disc Check | FMV |
|---------|-----------|------------|-----------|------------|-----|
| **Gold** | DW7 (patched) | DQ4 (re-encoded) | HYBRID_GLOBAL | Surgical bypass | Skip |
| DQ4 Native | DQ4 (patched) | DQ4 (native) | DQ4_PER_BLOCK | Minimal | Native |
| Cleanroom | Custom (extracted funcs) | DQ4 (re-encoded) | HYBRID_GLOBAL | None | Skip |

**Gold profile is the primary target.** All others are fallbacks.

---

## 5. ACKNOWLEDGMENT

This document serves as the single source of truth for the DQ4 Frankenstein build pipeline. All agents working on this project must:

1. Follow the Gold Standard Build Path exactly
2. Execute the Recursive QA Loop until zero faults remain
3. Make all modifications in C++ source (`frankenstein_driver.cpp`, `psx_binary_ops.cpp`)
4. Never revert to Python scripts or legacy build profiles
5. Log all diagnostic findings and fixes to the study folder

---

*End of Gold Standard Build Path Directive*
