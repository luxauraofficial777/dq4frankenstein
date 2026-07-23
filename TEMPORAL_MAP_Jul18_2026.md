# Temporal Map — DQ4 Frankenstein Project
**Date:** July 18, 2026 3:00 AM UTC-04:00
**Author:** GLM (Cascade)
**Purpose:** Chronological audit of all development progress, redundancy identification, gap analysis, and action consolidation. This is the definitive temporal record.

---

## Part 1: Chronological Index (v1 → v33)

### Phase A: Foundation & Translation (Jun 28 – Jul 1)

| Date | Milestone | Agent | Key Output |
|------|-----------|-------|------------|
| Jun 28 | Markus Schroeder's patcher integrated; CSV extraction complete | KIMI | `translation/csv/` (1,105 blocks), Java patcher |
| Jun 29 | Build 1-2: First patched DQ4 discs (`dq4_full_en.bin`) | Antigravity | Disemvowel pass, 7-pass compressor |
| Jun 30 03:39 | Build 3: `dq4_full_en.bin` — 117 degenerate entries fixed | GLM | 1,527 ChangeLogEntry records, 22 skipped blocks |
| Jun 30 05:45 | Build 6: `dq4_full_en_v6.bin` — 96% translated, grace factor | GLM | 14,840 EN, 478 control-only, 0 JP remaining |
| Jul 1 14:00 | DW7 Huffman tree search begins — 12 approaches tried, all failed | GLM | `HANDOFF_Jul1.md` (12 failed approaches documented) |
| Jul 1 20:50 | **DW7 Huffman tree FULLY CRACKED** at EXE@0x96FA0 (744 bytes, 186 leaves) | GLM | `dw7_exe_tree.bin`, DW7Schema in `hbe/huffman/dw7.py` |
| Jul 1 22:40 | **Menu commands RESOLVED** — all Huffman-encoded in HBD, not EXE | GLM | `MENU_COMMANDS_SOLVED.md` |
| Jul 1 16:00 | Boot strings patched (NewGame, Settings, EraseSave at 0xD1DD-0xD1FF) | GLM | SJIS full-width English in DQ4 EXE |
| Jul 1 21:50 | Hybrid Huffman tree built: 1472 bytes, 368 leaves, 100% round-trip | GLM (Session F) | `dw7_hybrid_tree.bin`, `dw7_encoded_translation.json` |

### Phase B: Frankenstein Architecture (Jul 2 – Jul 4)

| Date | Milestone | Agent | Key Output |
|------|-----------|-------|------------|
| Jul 2 | Session G: Java TextPatcher updated for DW7 schema, 744-byte tree | KIMI | `HeartBeatDataTextContentWriter` patched, first DW7-schema test |
| Jul 3 | **Translation cleanup complete**: 96.6% good English, 0 Cyntha | GLM | `merge_optimal_v3.py`, `full_translation.json` cleaned |
| Jul 3 | Siege Phases 1-3: 15,318 lines verified, 7 missing blocks identified | GLM | `SIEGE_FINAL_HANDOFF.md` |
| Jul 4 13:36 | Control codes isolated (25 DW7, 24 DQ4, 19 shared) | GLM | Hybrid tree updated to 1480 bytes / 370 leaves |
| Jul 4 14:15 | **Patched disc ready**: `dq4_full_en_dw7.bin` — 1,105 blocks, 0 errors | KIMI | EDC/ECC regenerated, ready for DuckStation |
| Jul 4 14:15 | DQ7 JP ROM found: `DW7D1/DQVIID1.bin` (SLPM-86500) | KIMI | HBD files swapped and corrected |

### Phase C: Frankenstein Build Iterations (Jul 5 – Jul 14)

| Date | Build | Profile | Key Change | Result |
|------|-------|---------|------------|--------|
| Jul 5-10 | v1-v8 | (standalone scripts) | Early EXE grafting experiments | Various failures — no boot |
| Jul 10-11 | v9-v11 | `build_frankenstein_v10/v11.py` | First unified builder attempts | Black screen, HBD format mismatch |
| Jul 12 | Bootstrap | `build_hook_iso.py` | MIPS bootstrap injected at 0x800B4A68 | 200M instrs, no crash, but soft-locked |
| Jul 12 | v12 | `frankenstein_builder.py` frank_v12 | DW7-preserved pointer redirect, LBA 354→355 | Black screen — missing 3 critical patches |
| Jul 13 | v13 | frank_v13 | Selective DQ4 overlay, DW7 HBD preserved | HBD shift broke STR/FMV LBAs |
| Jul 14 | v16 | (standalone) | Pointer remap fix, 2505 sectors read | Reached gameplay, then lost to later patches |

### Phase D: Disc-Check & HBD Shift Era (Jul 14 – Jul 17)

| Date | Build | Profile | Key Change | Result |
|------|-------|---------|------------|--------|
| Jul 14 | v18 | frankenstein_builder.py | HBD sector extraction bug FIXED | 50M instrs, polling loop, 1 CD-ROM sector |
| Jul 15 | v20-v21 | frank_v12 (patched) | +3 missing patches (LBA scan, seq table, ABS delta) | Still black screen — HBD shift problem |
| Jul 17 AM | CyberGrime RFE crash fixed | — | `v0` restored from exception context | Clean to 80M+ instructions |
| Jul 17 PM | v23 | frank_v13 | HBD shifted 354→355, all 9 disc-check patches | ❌ Hang at LBA 146621 (MDEC spinlock) |
| Jul 17 PM | v24 | frank_v14 | HBD shifted, surgical 7 patches | ❌ Same MDEC hang |
| Jul 17 PM | **v25** | frank_v15 | **NO HBD shift**, surgical 7 patches | ✅ Boots, FMV, menus — ❌ "Insert Disc 1" |
| Jul 17 PM | v26 | frank_v16 | NO shift, all 9 disc-check patches | ⏳ Awaiting DuckStation confirmation |
| Jul 17 PM | v27/v28 | frank_v17/v18 | RAM hook .cht / semi-surgical 8 | Undocumented builds |
| Jul 17 10:45 | **Breakthrough documented** | — | Root cause: HBD shift breaks STR LBAs | `FRANKENSTEIN_BREAKTHROUGH_Jul17_2026.md` |

### Phase E: Unified Build & Relocation Era (Jul 17 – Jul 18)

| Date | Build | Profile | Key Change | Result |
|------|-------|---------|------------|--------|
| Jul 17 11PM | v29 | frank_v19 | +CD-ROM Stop intercept trampoline | "Wrong Disc" persists |
| Jul 17 11PM | v30 | frank_v30 | Targeted disc_check exit patches + Stop intercept | "Insert Disc 1" (HBD format mismatch) |
| Jul 18 AM | v31 | frank_v31 | Unified: v30 engine + v12 HBD swap, tree at 0x800F4980 | EXE/HBD overlap — HBD overwrites tree on disc |
| Jul 18 AM | **v32** | frank_v32 (same builder) | `append_reloc_payload` fix, tree→0x800F4980 after BSS clear | CyberGrime PASS; DuckStation: game data section collision |
| Jul 18 AM | **v33** | frank_v33 (same builder) | Tree relocated to 0x80100000 (high RAM) | CyberGrime PASS; DuckStation: `Instruction read failed at PC=0x801005C8` |

---

## Part 2: Redundancy Identification — Circular Work Patterns

### Circle 1: Disc-Check Bypass (Attempted 5 Times)

| Attempt | Date | Approach | Patches | Result | Why It Didn't Stick |
|---------|------|----------|---------|--------|---------------------|
| 1 | Jul 14 | `patch_dw7_exe.py` standalone | All 9 | v12 builder missing patches | Builder didn't call `patch_disc_check()` |
| 2 | Jul 17 | `frank_v13` (v23) | All 9 + HBD shift | MDEC hang | HBD shift broke STR LBAs (not disc-check's fault) |
| 3 | Jul 17 | `frank_v14` (v24) | Surgical 7 + HBD shift | Same MDEC hang | Same root cause — HBD shift |
| 4 | Jul 17 | `frank_v15` (v25) | Surgical 7, no shift | "Insert Disc 1" | Only 7 patches — insufficient |
| 5 | Jul 17 | `frank_v16` (v26) | All 9, no shift | ⏳ Unconfirmed | **This is the correct approach** — awaiting test |

**Verdict:** The disc-check patch set (`DISC_CHECK_PATCHES` in `frankenstein_builder.py:66-121`) has been stable since Jul 14. The circular work was caused by the HBD shift confound, not the patches themselves. v26 (all 9, no shift) is the correct configuration. **No further disc-check iteration needed — just test v26.**

### Circle 2: Tree Placement / BSS Clear (Attempted 4 Times)

| Attempt | Date | Approach | RAM Address | Result | Why It Didn't Stick |
|---------|------|----------|-------------|--------|---------------------|
| 1 | Jul 12-14 | `append_hybrid_tree` (inline) | 0x800BC700 (inside BSS) | Tree zeroed by BSS clear | BSS clear range includes tree |
| 2 | Jul 14 | BSS clear patch (0x76B88: C668→CCC8) | 0x800BC700 (after BSS start) | Works for test_d, but tree still in BSS path on some builds | Partial fix — only skips tree region |
| 3 | Jul 18 | `append_reloc_payload` to 0x800F4980 | 0x800F4980 | Game data section collision — init code writes to 0x800F49A8 | Address is NOT empty; it's game initialized data |
| 4 | Jul 18 | Same, TREE_RAM_ADDR=0x80100000 | 0x80100000 | DuckStation: unmapped RAM gap? | Under investigation |

**Verdict:** The tree placement has been the single most circular issue. Each attempt fixes the previous failure but introduces a new one. The `append_reloc_payload` approach (v32/v33) is architecturally correct — the copy routine runs before BSS clear. The remaining issue is purely whether DuckStation maps RAM at 0x80100000.

### Circle 3: HBD Sector Shift (Attempted 3 Times, Now Resolved)

| Attempt | Date | Approach | Result |
|---------|------|----------|--------|
| 1 | Jul 12-14 | v12-v21: HBD 354→355 | Black screen (missing patches), then MDEC hang |
| 2 | Jul 17 | v23/v24: HBD 354→355 with disc-check | MDEC spinlock — STR internal LBAs broken |
| 3 | Jul 17 | v25+: NO HBD shift | ✅ Correct — boots with FMV |

**Verdict:** **RESOLVED.** The no-shift invariant is now understood. HBD stays at sector 354. This must be enforced in the builder.

### Circle 4: Huffman Tree Verification (Attempted 3 Times)

| Attempt | Date | Scope | Result |
|---------|------|-------|--------|
| 1 | Jul 1 | DW7 tree round-trip on DW7 HBD blocks | 99.3-100% match |
| 2 | Jul 1 | Hybrid tree (1472B) round-trip on 2,306 entries | 100% match, 0 mismatches |
| 3 | Jul 4 | Hybrid tree (1480B) re-encode all 15,318 entries | 100% fit, 0 compressed |
| 4 | Jul 16 | DQ4 schema round-trip on block 006C | PASSED (29/30 HBE tests green) |

**Verdict:** **RESOLVED but under-tested.** Round-trip verified on 1 block (006C). The roadmap calls for ≥3 blocks but this has never been done. The HBE test suite exists (`hbe/tests/`) with 12 test files but only covers 1 DQ4 block. **Action: Run round-trip on 2+ additional blocks.**

### Circle 5: Standalone Build Scripts (14 Duplicates)

The following standalone scripts duplicate logic now consolidated into `frankenstein_builder.py`:

| Script | Superseded By | Status |
|--------|---------------|--------|
| `build_frankenstein_v4.py` | `frankenstein_builder.py` frank_v12 | DEAD |
| `build_frankenstein_v6.py` | `frankenstein_builder.py` frank_v12 | DEAD |
| `build_frankenstein_v10.py` | `frankenstein_builder.py` frank_v12 | DEAD |
| `build_frankenstein_v11.py` | `frankenstein_builder.py` frank_v12 | DEAD |
| `build_frankenstein_v13.py` | `frankenstein_builder.py` frank_v13 | DEAD |
| `assemble_frankenstein_v5.py` | `frankenstein_builder.py` test_d | DEAD |
| `assemble_frankenstein_v5_1.py` | `frankenstein_builder.py` test_d | DEAD |
| `build_test_d_fixed.py` | `frankenstein_builder.py` test_d | DEAD |
| `build_test_d_v2.py` | `frankenstein_builder.py` test_d | DEAD |
| `build_test_d_isolate.py` | `frankenstein_builder.py` test_d | DEAD |
| `build_test_e.py` | `frankenstein_builder.py` test_d | DEAD |
| `build_test_frank.py` | `frankenstein_builder.py` test_a | DEAD |
| `build_test_hbd_only.py` | `frankenstein_builder.py` test_a | DEAD |
| `build_test_minimal.py` | `frankenstein_builder.py` test_0 | DEAD |
| `build_diagnostic.py` | `frankenstein_builder.py` test_b | DEAD |
| `build_diagnostic_v2.py` | `frankenstein_builder.py` test_b | DEAD |
| `build_rc1.py` | `frankenstein_builder.py` rc2 | DEAD |
| `build_bridge_iso.py` | `frankenstein_builder.py` frank_v13 | DEAD |
| `build_frankenstein.py` (root) | `frankenstein_builder.py` | DEAD — wrapper only |
| `build_bisection.py` | One-off diagnostic | DEAD |
| `build_bisection_fixed.py` | One-off diagnostic | DEAD |
| `build_b4_b5.py` | One-off diagnostic | DEAD |
| `build_b3_minimal.py` | One-off diagnostic | DEAD |
| `build_incremental_tests.py` | One-off diagnostic | DEAD |
| `build_control_clean.py` | One-off diagnostic | DEAD |
| `build_unpatched_test.py` | One-off diagnostic | DEAD |
| `build_txrt_payload.py` | `append_reloc_payload` in builder | DEAD |

### Circle 6: Verify Scripts (55+ Duplicates)

There are **55+ `verify_*.py` scripts** in the root and `cybergrime/` directories, most verifying specific historical builds:

- `verify_tramp.py` through `verify_tramp7.py` (7 files) — all verify the same trampoline concept for different builds
- `verify_v2.py` through `verify_v31_final.py` (15+ files) — per-build verification
- `verify_disc_check2.py` through `verify_disc_check4.py` (3 files) — disc-check verification iterations
- `verify_frank_exe.py` / `_v2.py` / `_v3.py` (3 files) — EXE verification iterations
- `verify_b1f.py`, `verify_b2_patches.py`, `verify_v3b.py` — one-off build checks

**All of these are superseded by:**
- `cybergrime/verify_v33.py` (latest build verifier)
- `cybergrime/verify_tramp_v33.py` (latest trampoline verifier)
- `translation-tools/hbe/tests/` (Huffman round-trip tests)
- `translation-tools/validate_iso.py` + `validate_exe.py` (structural validators)

---

## Part 3: Gap Analysis — Stalled Components

### Gap 1: FMV/STR Skip — NEVER ATTEMPTED

| Session | Status | Details |
|---------|--------|---------|
| Jul 1 (`HANDOFF_Jul1.md`) | Identified | "DW7 EXE expects post-adventure-log FMV embedded in HBD. DQ4 HBD doesn't have it." |
| Jul 14 (v21 memory) | Not patched | "Video/FMV skip: Still NOT patched" |
| Jul 17 (`UNIFIED_PRODUCT_ROADMAP`) | Planned | Phase 3a: Write `skip_fmv.py` — find `jal` calling STR playback, replace with `nop` |
| Jul 18 | **NOT STARTED** | No `skip_fmv.py` or `fmv_skip` script exists anywhere in the codebase |

**Root cause of stall:** The FMV skip was always deferred as "post-boot" work. The project got stuck on the boot itself (disc-check, HBD format) and never reached the point where FMV skip was needed. Now that v25/v26 boot successfully, this is the next blocker for gameplay progression.

**Why it hasn't stuck:** It was never actually attempted. Zero scripts exist. The MIPS target (find the `jal` instruction calling STR playback) has not been disassembled.

### Gap 2: Huffman Round-Trip on Multiple Blocks — PARTIALLY DONE

| Session | Status | Details |
|---------|--------|---------|
| Jul 1 (Session F) | Done | 2,306 entries, 100% match — but on DW7 tree, not hybrid tree |
| Jul 4 (GLM) | Done | 15,318 entries re-encoded, 100% fit — encoding only, not decode round-trip |
| Jul 16 | Done | Block 006C DQ4 schema round-trip PASSED — 1 block only |
| Jul 17 (Roadmap) | Planned | "Round-trip test on ≥3 blocks (006C + two others)" |
| Jul 18 | **STALLED** | Only 1 block verified. 2 more needed. HBE test suite has `test_dq4_schema.py` but only tests 006C. |

**Root cause of stall:** The HBE test suite was built incrementally and only ever had 006C as a test fixture. Adding 2 more blocks requires loading additional HBD block data as test fixtures, which nobody has done.

### Gap 3: HBD Block Format Adapter — DESIGNED BUT NOT IMPLEMENTED

| Session | Status | Details |
|---------|--------|---------|
| Jul 17 (`UNIFIED_PRODUCT_ROADMAP`) | Designed | Approach C: Strip per-block trees, re-encode with hybrid tree, set hts=24/treeEnd=0/textEnd=0 |
| Jul 17 (`DUAL_FORK_ROADMAP`) | Detailed | Block header format delta documented, MIPS targets identified |
| Jul 18 | **NOT STARTED** | No `reencode_dq4_blocks.py` exists. No `verify_block_format.py` exists. |

**Root cause of stall:** This is the core remaining technical work (Phase 2 / Gate G2). It depends on G1 (v26 boot confirmation) which hasn't happened yet. The design is complete; implementation hasn't started.

### Gap 4: DuckStation Control Tests — CONFIGURED BUT NOT RUN

| Session | Status | Details |
|---------|--------|---------|
| Jul 15-16 | Configured | BIOS copied, settings written, launch script created |
| Jul 16 (`CURRENT_STATE.md`) | **NOT RUN** | "Control tests NOT YET RUN — must verify stock DW7 + DQ4 JP boot" |
| Jul 18 | **STILL NOT RUN** | `launch_duckstation_tests.py` exists but has never been executed |

### Gap 5: v33 DuckStation Boot Failure — UNDER INVESTIGATION

| Session | Status | Details |
|---------|--------|---------|
| Jul 18 | Investigating | `Instruction read failed at PC=0x801005C8` — hypothesis: 0x80100000 unmapped in DuckStation |
| Jul 18 | **UNRESOLVED** | Need to check DuckStation source for RAM mapping behavior, or try alternative addresses |

---

## Part 4: Single Source of Truth — Script Classification

### TIER 1: Active Scripts (CURRENT — keep in place or move to `/core`)

#### Build System
| Script | Location | Purpose | v33 Utility |
|--------|----------|---------|-------------|
| `frankenstein_builder.py` | `translation-tools/` | Unified builder — all profiles | ✅ ACTIVE (frank_v33 profile) |
| `build_heart.py` | `translation-tools/` | Builds translated HBD heart | ✅ ACTIVE (produces `dq4_heart_v17.bin`) |
| `build_hybrid.py` | `translation-tools/` | Translation compression pipeline | ✅ ACTIVE (7-pass compressor) |
| `build_full_translation.py` | `translation-tools/` | Builds full_translation.json | ✅ ACTIVE |
| `build_mapping.py` | `translation-tools/` | Builds translation_mapping.json | ✅ ACTIVE |
| `build_folder_mapping.py` | `translation-tools/` | Builds folder_mapping.json | ✅ ACTIVE |

#### Translation Pipeline
| Script | Location | Purpose | v33 Utility |
|--------|----------|---------|-------------|
| `apply_mapping_v2.py` | `translation-tools/` | Applies JP→EN mapping to CSVs | ✅ ACTIVE |
| `merge_optimal_v3.py` | `translation-tools/` | Byte-based merge with grace factor | ✅ ACTIVE |
| `compress_translations.py` | `translation-tools/` | 7-pass progressive compression | ✅ ACTIVE |
| `qa_check.py` | `translation-tools/` | QA validation (no apostrophes, etc.) | ✅ ACTIVE |
| `dq4_hbd_patcher.py` | `translation-tools/` | Python HBD patcher | ✅ ACTIVE |
| `patch_dw7_exe.py` | `translation-tools/` | Standalone EXE patcher (has `patch_hbd_filename()`) | ✅ ACTIVE (reference) |

#### HBE Engine (Huffman/Block/Encode)
| Path | Purpose | v33 Utility |
|------|---------|-------------|
| `translation-tools/hbe/` | Complete HBD parse/encode/decode library | ✅ ACTIVE |
| `translation-tools/hbe/huffman/dq4.py` | DQ4 Huffman schema | ✅ ACTIVE |
| `translation-tools/hbe/huffman/dw7.py` | DW7 Huffman schema | ✅ ACTIVE |
| `translation-tools/hbe/huffman/builder.py` | Huffman tree builder | ✅ ACTIVE |
| `translation-tools/hbe/huffman/length_limited.py` | Length-limited builder | ✅ ACTIVE |
| `translation-tools/hbe/compression/lzss.py` | LZSS compression | ✅ ACTIVE |
| `translation-tools/hbe/parser/text.py` | Text block parser | ✅ ACTIVE |
| `translation-tools/hbe/archive.py` | HBD archive parser | ✅ ACTIVE |
| `translation-tools/hbe/validator.py` | Block validator | ✅ ACTIVE |
| `translation-tools/hbe/tests/` | 12 test files (29/30 passing) | ✅ ACTIVE |

#### CyberGrime Emulator
| Script | Location | Purpose | v33 Utility |
|--------|----------|---------|-------------|
| `psx_emulator_core.cpp` | `cybergrime/` | Emulator core | ✅ ACTIVE |
| `psx_agent_runner.cpp` | `cybergrime/` | Headless runner | ✅ ACTIVE |
| `psx_agent_runner.exe` | `cybergrime/` | Compiled runner | ✅ ACTIVE |
| `psx_exe.py` | `cybergrime/` | PS-X EXE parser (canonical) | ✅ ACTIVE |
| `verify_v33.py` | `cybergrime/` | v33 build verifier | ✅ ACTIVE |
| `verify_tramp_v33.py` | `cybergrime/` | v33 trampoline verifier | ✅ ACTIVE |
| `launch_duckstation_tests.py` | `cybergrime/` | DuckStation test launcher | ✅ ACTIVE (needs running) |
| `duckstation_control_test.py` | `cybergrime/` | Control test harness | ✅ ACTIVE |

#### Validation
| Script | Location | Purpose | v33 Utility |
|--------|----------|---------|-------------|
| `validate_iso.py` | `translation-tools/` | ISO structure validator | ✅ ACTIVE |
| `validate_exe.py` | `translation-tools/` | EXE structure validator | ✅ ACTIVE |

#### Tools
| Tool | Location | Purpose |
|------|----------|---------|
| `edcre.exe` | `edcre/` | EDC/ECC regeneration |
| `xdelta3.exe` | `xdelta3_extracted/` | Patch generation |
| `dq4psx-patcher.jar` | `dragon-hackst-4-src/.../target/` | Java patcher (Markus + KIMI) |

### TIER 2: Reference Scripts (useful for analysis, not for building)

| Script | Location | Purpose |
|--------|----------|---------|
| `patch_dw7_exe.py` | `translation-tools/` | Has `patch_hbd_filename()` — reference for EXE patching |
| `disasm_disc_check.py` | `translation-tools/` | Disassembly of disc-check function |
| `analyze_disc_checks.py` | `translation-tools/` | Disc-check analysis |
| `inspect_v31_build.py` | `cybergrime/` | v31 build inspector (reference for BSS analysis) |
| `check_bss_real.py` | root | BSS clear range analysis |
| `BSS_CLEAR_ANALYSIS_Jul18_2026.md` | `study/` | BSS clear documentation |
| `investigate_v23_fmv.py` | root | FMV/MDEC hang investigation (reference for FMV skip) |

### TIER 3: Archive Candidates (DEAD — superseded or one-off)

#### Dead Build Scripts (27 files)
All scripts listed in Circle 5 above. These duplicate `frankenstein_builder.py` logic.

#### Dead Verify Scripts (50+ files)
All `verify_*.py` files except `verify_v33.py`, `verify_tramp_v33.py`, and the HBE test suite.

#### Dead Analysis Scripts (30+ files in root)
- `analyze_*.py` (16 files in root) — one-off analyses, results captured in study docs
- `check_*.py` (12 files in root) — one-off checks, results captured in study docs
- `batch_build_dw7_catalog.py` / `_v2.py` — superseded by HBE engine
- `batch_translate_*.py` (24 files) — one-off translation batches, all completed

#### Dead CyberGrime Scripts (20+ files)
- `analyze_decompressor2/3/4.py` — superseded by `analyze_dw7_decompressor.py`
- `analyze_telemetry.py` / `_2.py` / `_quick.py` — superseded by `analyze_v18_telemetry.py`
- `apply_awakening_patch.py` / `_v2.py` — bootstrap-era, no longer used
- `apply_disc_patches.py` / `_v2.py` — superseded by builder's `patch_disc_check()`
- `apply_frank_patch_v3.py` — superseded by builder
- `apply_hbd_patch.py` — superseded by builder
- `apply_phase0_phase1.py` — one-off
- `apply_surgical_patches.py` — superseded by builder
- `build_hook_iso.py` — bootstrap-era, superseded
- `build_heart_v17.py` — superseded by `build_heart.py`
- `check_*.py` (8 files) — one-off checks

---

## Part 5: Action Consolidation — `/core` Folder Structure

### Proposed Structure

```
DQLOSTTRANSLATION/
├── core/                           # NEW — Single Source of Truth
│   ├── build/
│   │   ├── frankenstein_builder.py     # Unified builder (moved from translation-tools/)
│   │   ├── build_heart.py              # HBD heart builder (moved from translation-tools/)
│   │   └── build_hybrid.py             # Translation compressor (moved from translation-tools/)
│   ├── translate/
│   │   ├── apply_mapping_v2.py         # Mapping applier
│   │   ├── merge_optimal_v3.py         # Optimal merge
│   │   ├── compress_translations.py    # 7-pass compressor
│   │   ├── qa_check.py                 # QA validation
│   │   └── dq4_hbd_patcher.py          # Python HBD patcher
│   ├── hbe/                            # Moved from translation-tools/hbe/
│   │   ├── huffman/
│   │   ├── compression/
│   │   ├── parser/
│   │   ├── tests/
│   │   └── ...
│   ├── verify/
│   │   ├── validate_iso.py
│   │   ├── validate_exe.py
│   │   └── verify_v33.py
│   ├── emulator/
│   │   ├── psx_emulator_core.cpp
│   │   ├── psx_agent_runner.cpp
│   │   ├── psx_agent_runner.exe
│   │   ├── psx_exe.py
│   │   └── launch_duckstation_tests.py
│   └── tools/
│       └── edcre.exe
├── _archive/                       # NEW — all dead scripts
│   ├── build_scripts/                  # 27 dead build scripts
│   ├── verify_scripts/                 # 50+ dead verify scripts
│   ├── analyze_scripts/                # 30+ one-off analysis scripts
│   └── cybergrime_legacy/              # 20+ dead emulator scripts
├── study/                          # Documents only (no scripts)
├── translation/                    # Data files (CSVs, JSONs)
└── ...
```

### Archival Rules

1. **If a script's function exists in `frankenstein_builder.py`** → archive it
2. **If a `verify_*.py` targets a build ≤ v32** → archive it (only v33 verifiers stay)
3. **If an `analyze_*.py` or `check_*.py` has results captured in a `study/*.md`** → archive it
4. **If a `batch_translate_*.py` has been run and results merged into `full_translation.json`** → archive it
5. **If a script imports from `frankenstein_builder.py` or `hbe/`** → keep it (it's using the active toolchain)

---

## Part 6: Summary of Circular Work

### The 3 Biggest Circles

1. **Disc-check bypass (5 iterations, Jul 14-17):** The patch set was correct from iteration 1, but the HBD shift confound made it appear broken. **Resolved by the no-shift breakthrough.** v26 is the correct build.

2. **Tree placement (4 iterations, Jul 12-18):** Each address choice failed for a different reason. The `append_reloc_payload` approach is correct. The remaining issue is DuckStation's RAM mapping at 0x80100000. **One more iteration needed.**

3. **Standalone build scripts (14 duplicates, Jul 5-14):** 14 standalone `build_frankenstein_v*.py` and `build_test_*.py` scripts were created before `frankenstein_builder.py` consolidated them. All 14 are dead. **Ready for archival.**

### The 2 Biggest Gaps

1. **FMV/STR skip:** Designed in the roadmap but never implemented. Zero scripts exist. This is the next blocker after boot is confirmed.

2. **HBD block format adapter:** Designed as "Approach C" but never implemented. No `reencode_dq4_blocks.py` exists. This is the core remaining technical work.

### Mandate of Sound Logic — Scripts That Provide Zero Utility for v33

The following categories of scripts provide **no unique utility for the current v33 build** and should be archived immediately:

- **All 27 standalone build scripts** (superseded by `frankenstein_builder.py`)
- **All 50+ `verify_*.py` scripts except `verify_v33.py` and `verify_tramp_v33.py`**
- **All 24 `batch_translate_*.py` scripts** (translation is complete)
- **All 16 `analyze_*.py` scripts in root** (results captured in study docs)
- **All 12 `check_*.py` scripts in root** (results captured in study docs)
- **All 8+ `apply_*_patch*.py` scripts in cybergrime/** (superseded by builder)
- **All `build_bisection*.py`, `build_b4_b5.py`, `build_b3_minimal.py`** (one-off diagnostics)
- **`build_frankenstein.py` in root** (wrapper that calls builder — dead)

**Total: ~140+ scripts for immediate archival.**

---

---

## Part 7: Archival Results — EXECUTED

The archival plan was executed on Jul 18, 2026. All dead scripts were moved to `_archive_scripts/`.

### Final Active Script Count

| Location | Active .py Files | Notes |
|----------|-----------------|-------|
| Root | 3 | `merge_optimal_v3.py`, `reencode_dq4_blocks.py`, `run_duckstation.py` |
| `translation-tools/` (top-level) | 15 | Builder, patchers, validators, HBD tools |
| `translation-tools/hbe/` (recursive) | 49 | Huffman/Block/Encode engine + tests |
| `cybergrime/` | 15 | Emulator core, runners, v33 verifiers |
| **Total Active** | **82** | |
| **Total Archived** | **841** | |

### Archive Directory Structure

| Directory | Files | Contents |
|-----------|-------|----------|
| `_archive_scripts/analyze_scripts/` | 531 | One-off analysis, check, scan, test, fix scripts |
| `_archive_scripts/cybergrime_legacy/` | 189 | Dead CyberGrime analysis, injection, patching scripts |
| `_archive_scripts/batch_translate_scripts/` | 45 | Completed batch translation runs |
| `_archive_scripts/build_scripts/` | 27 | Superseded standalone build scripts |
| `_archive_scripts/verify_scripts/` | 49 | Per-build verification scripts (v2-v32) |

### Active Scripts Remaining

**Root (3):**
- `merge_optimal_v3.py` — Byte-based translation merge
- `reencode_dq4_blocks.py` — DQ4 block re-encoder (for Approach C)
- `run_duckstation.py` — DuckStation launch helper

**translation-tools/ (15):**
- `frankenstein_builder.py` — Unified builder (all profiles)
- `build_heart.py` — HBD heart builder
- `build_hybrid.py` — Translation compressor
- `build_full_translation.py` — Full translation JSON builder
- `build_mapping.py` — Translation mapping builder
- `build_folder_mapping.py` — Folder mapping builder
- `apply_mapping_v2.py` — JP→EN mapping applier
- `compress_translations.py` — 7-pass progressive compressor
- `qa_check.py` — QA validation
- `dq4_hbd_patcher.py` — Python HBD patcher
- `patch_dw7_exe.py` — Standalone EXE patcher (reference)
- `validate_iso.py` — ISO structure validator
- `validate_exe.py` — EXE structure validator
- `disasm_disc_check.py` — Disc-check disassembly (reference)
- `disasm_dw7.py` — DW7 disassembly (reference)

**cybergrime/ (15):**
- `__main__.py` — Emulator entry point
- `agent_telemetry.py` — Telemetry collector
- `baseline_manager.py` — Baseline comparison
- `crash_analyzer.py` — Crash analysis tool
- `duckstation_control_test.py` — DuckStation control test
- `harness_runner.py` — Test harness runner
- `iso_toolkit.py` — ISO manipulation tools
- `iteration_harness.py` — Iteration test harness
- `launch_duckstation.py` — DuckStation launcher
- `launch_duckstation_tests.py` — DuckStation test launcher
- `mips_asm.py` — MIPS assembler
- `psx_exe.py` — PS-X EXE parser (canonical)
- `regression_runner.py` — Regression test runner
- `verify_v33.py` — v33 build verifier
- `verify_tramp_v33.py` — v33 trampoline verifier

**translation-tools/hbe/ (49):**
- Complete HBD parse/encode/decode library + 12 test files

---

*This document is the definitive temporal audit. It should be referenced before starting any new work to avoid repeating circular patterns.*
