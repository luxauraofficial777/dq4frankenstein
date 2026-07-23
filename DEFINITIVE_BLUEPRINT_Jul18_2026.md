# DQLOSTTRANSLATION — Definitive Blueprint to 100% Completion

**Date:** Jul 18, 2026 | **Status:** v34 boots in DuckStation. Remaining: black screen past boot, HBD re-encoding, FMV skip, 7 missing blocks, golden path QA.

---

## 1. Architecture

Frankenstein hybrid disc: **DW7 US EXE** (engine) + **DQ4 JP HBD** (assets, English text re-encoded).

```
Sector 24:   DW7 EXE (patched, 677,888 bytes / 331 sectors)
Sector 355:  DQ4 HBD (English-patched, 319,436,800 bytes / 155,975 sectors)
Hybrid Tree: 1,480 bytes, 370 leaves (186 DW7 + 180 DQ4 controls + 4 punct)
```

### Memory Map (Proven)

| Range | Segment |
|-------|---------|
| `0x80017F00` | EXE load address (BASE) |
| `0x8008E284` | DQ4 BSS clear entry (original PC0) |
| `0x800BC668–0x800F4980` | BSS clear range (code-level, not header-driven) |
| `0x800BC700` | Hybrid tree on-disc (within EXE t_size) |
| `0x800BCCF0` | Copy routine (new PC0) |
| `0x800EF1C8` | DW7 original global tree (OUTSIDE loaded EXE) |
| `0x800F4980+` | Game initialized data section (NOT empty space) |
| `0x80100000` | Hybrid tree relocated (v33-v34, safe high RAM) |
| `0x801005C8` | CD-ROM Stop intercept trampoline (relocated) |
| `0x80138000–0x8017FFFF` | Dynamic heap (map geometry) |
| `0x801FFFF0` | Stack top |

### Key Constants

- EXE LBA=24, HBD LBA=355 (shifted from 354 to avoid overlap)
- DQ4 HBD: 319MB/155,975 sectors. DW7 HBD: 618MB/302,033 sectors
- ABS table @ EXE 0x4184, REL table @ 0x511C, 8-byte entries, max 6000
- 168 matched folders, 1755 unmatched ABS (+1 delta), 24 MIPS tree ref pairs
- 10 disc-check patches (semi-surgical), BSS patch @ EXE 0x76B88
- 15,318 translation entries, 1,105 blocks, 7 missing

---

## 2. Proven Facts

1. **DW7 global tree** at RAM `0x800EF1C8`, EXE@0x96FA0 (744B, 186 leaves). 24 `lui/addiu` pairs reference it.
2. **DQ4 per-block trees**: treeEnd/textEnd non-zero, hts=24. DW7: treeEnd=0, textEnd=0, hts~1390.
3. **DW7Schema**: `root_id = val & 0x7FFF`. **DQ4Schema**: `root_id = (val & 0x7FFF) + 1`.
4. **BSS clear is code-level** at `0x8008E284`. EXE header fields 0x30/0x34 unused by DuckStation.
5. **`0x800F4980` is game data**, not empty space. Game init writes to `0x800F4930`, `0x800F4950`, `0x800F49A8`.
6. **`0x80100000` is safe** — DuckStation maps full 2MB RAM. No game code references it.
7. **Copy routine requires NOP** between `lw`/`sw` (load delay hazard) and **bne offset -6** (0xFFFA).
8. **HBD shift 354→355 required** — extended EXE (331 sectors, LBA 24-354) overlaps HBD at 354.
9. **Translation 100% complete** for found blocks. 96.6% good English quality.
10. **Re-encoding round-trip verified** on 5 blocks via `reencode_dq4_blocks.py`.

### Copy Routine (v34 — PROVEN WORKING)

```
idx 0:  lui $t0, 0x800C          # src = 0x800BC700
idx 1:  addiu $t0, $t0, 0xC700
idx 2:  lui $t1, 0x8010          # dst = 0x80100000
idx 3:  addiu $t1, $t1, 0x0000
idx 4:  addiu $t2, $t0, 1520     # end = src + payload
idx 5:  lw $t3, 0($t0)           # ← LOOP TARGET
idx 6:  nop                      # CRITICAL: load delay slot
idx 7:  sw $t3, 0($t1)
idx 8:  addiu $t0, $t0, 4
idx 9:  addiu $t1, $t1, 4
idx 10: bne $t0, $t2, -6         # CRITICAL: offset -6 = 0xFFFA
idx 11: nop
idx 12: j 0x8008E284             # → original PC0
idx 13: nop
```

---

## 3. Build History — Everything Tried

### Phase 1: Translation (v1-v6)
- v1-v5: Direct SJIS replacement, partial success
- v6: Full English, Huffman grace 1.3x, 96% English, 0 JP, size matches

### Phase 2: Frankenstein Architecture (v12-v20)
- v12: DQ4 HBD swap, broken pointer remap → black screen
- v13/v23: DW7 HBD shifted +1, DQ4 folders appended → STR internal LBAs broken
- v15/v25: Surgical disc-check, no HBD shift → "Insert Disc 1"
- v16/v26: Full disc-check, no HBD shift → cd_init destroys MODE2 init
- v18/v28: Semi-surgical (no cd_init) → "Wrong Disc" (missing Stop intercept)
- v19/v29: + Stop intercept → infinite CD-ROM streaming (BSS expansion zeroing data)
- v20: `remap_pointers_matched_only` fix → black screen (missing LBA scan, seq table, ABS delta)
- v21: All fixes applied → not tested

### Phase 3: Unified Build (v30-v35)
- v30: Targeted disc_check exits + Stop intercept, no HBD shift → "Wrong Disc" (no DQ4 HBD)
- v31: v30 engine + v12 HBD swap, tree at 0x800F4980 via padding → crash (EXE/HBD overlap)
- v32: `append_reloc_payload`, copy to 0x800F4980 → DuckStation fail (game data collision)
- v33: Tree to 0x80100000 → DuckStation fail (copy routine bugs: missing NOP, wrong bne)
- **v34: NOP + bne -6 → DUCKSTATION BOOT SUCCESS** → black screen (HBD size mismatch)
- v35: In-place tree at 0x800BC700, BSS skip → black screen (BSS variables overwrite tree)

### v34 DuckStation Evidence
```
BIOS POST ✓ | Kernel ✓ | BSS clear ✓ | CD-ROM reads ✓
FPS: 0→61→511 ✓ | No instruction read errors ✓
→ Black screen: DQ4 HBD (319MB) < DW7 HBD (618MB), unmatched ABS pointers hit zeros
```

---

## 4. What Worked

1. **v34 copy routine** — DuckStation confirmed boot
2. **Hybrid tree** — 1480B, 370 leaves, round-trip verified
3. **Translation** — 15,318 lines, 100%, 96.6% good quality
4. **Disc-check bypass** — 10 semi-surgical patches, preserves CD-ROM init
5. **CD-ROM Stop intercept** — 10-instruction trampoline
6. **LBA patching** — broad scan, seq table, ABS/REL remapping
7. **CyberGrime smoke** — v32/v33 pass 5M instruction tests
8. **EDC/ECC** — edcre.exe regenerates 287,272 sectors
9. **HBD re-encoder** — `reencode_dq4_blocks.py`, round-trip on 5 blocks
10. **DuckStation QA harness** — `tools/frankenstein_qa/`, 8 checkpoints

---

## 5. Current Blockers

| # | Blocker | Status | Impact |
|---|---------|--------|--------|
| 1 | Black screen after boot | DQ4 HBD < DW7 HBD, unmatched ABS hit zeros | Prevents title screen |
| 2 | HBD block format incompatibility | DQ4 per-block trees vs DW7 global tree | Text decode fails |
| 3 | FMV/STR skip | DW7 EXE expects FMV in HBD, DQ4 doesn't have it | Hang after "New Game" |
| 4 | 7 missing translation blocks | 043f, 0474, 0475, 0476, 0481, 0489, 048a | Incomplete translation |
| 5 | Re-encoding edge cases | Blocks with tree length=0 fail | Incomplete re-encoding |
| 6 | CyberGrime GPU limitation | No display interrupts | Can't verify visually |

---

## 6. Final Implementation Angles

### Angle A: v34 + Selective HBD Overlay (RECOMMENDED)

Combine v34's proven engine patches with v23's HBD strategy:
1. Copy ALL DW7 HBD from 354→355 (shift +1, data preserved)
2. Append 883 matched DQ4 folders at sector 302,388
3. Patch 168 matched ABS/REL to DQ4 locations
4. +1 delta on 1755 unmatched ABS (correct: same DW7 data, shifted)
5. Keep v34's copy routine, disc-check, Stop intercept, tree at 0x80100000
6. Re-encode DQ4 text blocks before appending

**New profile**: `frank_v36` in `frankenstein_builder.py`

**Alternative A1**: Zero-pad DQ4 HBD to 618MB. Simple but wasteful.
**Alternative A3**: Clamp unmatched ABS pointers > DQ4_HBD_END to last valid sector.

### Angle B: HBD Re-encoding (Approach C)

Fix `reencode_dq4_blocks.py`:
1. Handle zero-length trees (write minimal DW7-format block with control codes only)
2. Handle blocks that grow (shift subsequent blocks, rebuild HBD)
3. Run on all 1,105 blocks
4. Integrate with builder via `reencode_hbd_blocks()`

### Angle C: FMV/STR Skip

1. Find MDEC register writes (`0x1F801824`) in DW7 EXE
2. Patch `jal` to FMV player → `nop; nop`
3. Or patch FMV player function → `jr $ra; nop`

### Angle D: 7 Missing Blocks

1. Extract JP text using `hbe/huffman/dq4.py`
2. Translate using NES US reference or manual
3. Re-encode with hybrid tree, insert into HBD

### Angle E: Golden Path QA

CyberGrime: `psx_agent_runner.exe <disc.bin> 500000000 golden_path_script.json`
DuckStation: `python qa_harness.py dq4_frankenstein --checkpoints boot title new_game`

---

## 7. Testing Infrastructure

### CyberGrime (Custom Emulator)
- Location: `cybergrime/`
- Full MIPS R3000A, HLE BIOS, CD-ROM, agent harness with JSON telemetry
- GPU stub (no display interrupts — known limitation)
- Best for: boot verification, BSS clear, CD-ROM reads, crash detection
- Build: `cl psx_autoplay.cpp psx_emulator_core.cpp agent_harness.cpp /Fe:psx_agent_runner.exe`

### DuckStation (Full Emulator)
- Location: `duckstation/`
- Full PSX emulation with GPU rendering
- Best for: visual verification, title screen, gameplay, final release testing

### Frankenstein QA Harness
- Location: `tools/frankenstein_qa/`
- `qa_harness.py` orchestrator with 8 checkpoints
- Screenshot analysis, log scanning, memory watch, save state management

### Golden Path Workflow
- Location: `.devin/workflows/golden-path.md`
- Script: `cybergrime/golden_path_script.json`
- VBlanks 0-6300: boot → title → new game → prologue → battle → save
- `--victory-checkpoint` for full save verification cycle

---

## 8. Completion Checklist

### Phase 1: Fix Black Screen
- [ ] 1.1: Implement `frank_v36` = v34 engine + v23 HBD overlay + re-encoding
- [ ] 1.2: Fix `reencode_dq4_blocks.py` zero-length tree handling
- [ ] 1.3: Build `dq4_frankenstein_v36.bin`
- [ ] 1.4: CyberGrime smoke test (5M instrs, no crash)
- [ ] 1.5: DuckStation test — verify title screen
- [ ] 1.6: If still black, try zero-pad (A1) or ABS clamping (A3)

### Phase 2: FMV/STR Skip
- [ ] 2.1: Find FMV player entry (MDEC writes)
- [ ] 2.2: Patch jal → nop or function → jr $ra
- [ ] 2.3: Test "New Game" doesn't hang

### Phase 3: Translation Completion
- [ ] 3.1: Extract JP from 7 missing blocks
- [ ] 3.2: Translate and re-encode
- [ ] 3.3: Insert into HBD, verify all 1,105 blocks

### Phase 4: Golden Path QA
- [ ] 4.1: Build `psx_agent_runner.exe`
- [ ] 4.2: Run golden path on v36
- [ ] 4.3: Run DuckStation QA harness (8 checkpoints)
- [ ] 4.4: Victory checkpoint verification

### Phase 5: Release
- [ ] 5.1: Fix remaining issues
- [ ] 5.2: Generate .xdelta patch
- [ ] 5.3: Write README
- [ ] 5.4: Final playthrough
- [ ] 5.5: Release

---

## Appendix: Build Profiles

| Profile | Output | Strategy | DuckStation |
|---------|--------|----------|-------------|
| `test_0` | dq4_test_0.bin | Plain DW7 | Baseline ✓ |
| `test_d` | dq4_test_d.bin | DW7 HBD + tree | Enix logo ✓, "Wrong Disc" |
| `frank_v12` | dq4_frankenstein_v12.bin | Full swap, broken remap | Black screen |
| `frank_v13` | dq4_frankenstein_v23.bin | Selective overlay | Black screen (STR LBAs) |
| `frank_v31` | dq4_frankenstein_v31.bin | Unified, tree at 0x800F4980 | Crash (EXE/HBD overlap) |
| `frank_v32` | dq4_frankenstein_v32.bin | Reloc payload, 0x800F4980 | Fail (game data collision) |
| `frank_v33` | dq4_frankenstein_v33.bin | Tree at 0x80100000 | Fail (copy routine bugs) |
| **`frank_v34`** | **dq4_frankenstein_v34.bin** | **NOP + bne -6** | **BOOT ✓ → black screen** |
| `frank_v35` | dq4_frankenstein_v35.bin | In-place tree, BSS skip | Black screen (BSS overwrite) |
| `rc2` | dq4_en_rc2.bin | DQ4 EXE + patched HBD | Not tested |

**Build command**: `python translation-tools/frankenstein_builder.py --profile frank_v34`
