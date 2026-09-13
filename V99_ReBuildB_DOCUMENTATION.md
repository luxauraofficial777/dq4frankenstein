# Sovereign Native Pipeline V.99 Rebuild B — Technical Specification

**Target:** Dragon Quest IV: Michibikareshi Mono Tachi (PSX) — Sovereign Native Rebuild B  
**Target Architecture:** Sony PlayStation (`SLPM_869.16` MIPS R3000A + `HBD1PS1D.Q41`)[cite: 1]  
**Author:** VoidWalkers Project / Lux Aura  
**Date:** September 13, 2026  

---

**1. System Specifications & Input Constraints**

| Parameter | Specification | Verification / Source Method |
|---|---|---|
| **Pipeline Model** | Sovereign Native In-Place Re-authoring[cite: 1] | Direct patch of native Japanese binary; replaces DW7 executable graft[cite: 1] |
| **Target Executable** | `SLPM_869.16` (PS1 MIPS R3000A)[cite: 1] | Pristine retail Japanese executable[cite: 1] |
| **Archive Target** | `HBD1PS1D.Q41`[cite: 1] | Native HeartBeat Engine compressed archive container[cite: 1] |
| **Source Disc Image** | `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` | Raw retail CD-ROM dump |
| **Disc Size** | 368,057,424 bytes | Exact 2352 bytes/sector Mode 2 Form 1 layout[cite: 1] |
| **Source CRC-32** | `3D67C858` | Pre-build input verification check |
| **Source SHA-256** | `100D87DB9DEADF8F9FA4BB891D3A5D0BB112ACBF5ADBCBC93C637848ED9C7531` | Pre-build input verification check |
| **Disc Sector Layout** | Mode 2 Form 1 (2,048 data / 2,352 raw)[cite: 1] | Zero sector drift maintained across all 156,487 LBAs[cite: 1] |
| **Master Build Hash** | `AC9F94A13A5627C30013FB0D13C88CD96F4BCEC878A2DA6E7B1978E1CD4E5703` | Full SHA-256 of verified Rebuild B master image |

---

**2. Subsystem Transformation Matrix**

| Subsystem | Target Files / RAM Vectors | Transformation Scope | Technical Resolution |
|---|---|---|---|
| **Dialogue Stream** | `HBD1PS1D.Q41` (1,358 blocks)[cite: 1] | Length-limited Huffman re-encoding | Re-encoded using depth 14..9 trees with controlled bit-boundary padding[cite: 1]. |
| **Cutscene Scripts** | 612 Type-39 script blocks[cite: 1] | LZSS decompress $\rightarrow$ remap $\rightarrow$ recompress | Decodes LZSS stream, realigns dialogue SIDs, prevents throne-room freeze[cite: 1]. |
| **Font Subsystem** | Block `0x048C` @ file `0x97AC8`[cite: 1] | 8×14 half-width Font 1 injection[cite: 1] | Resolves 57/57 resident referrers; injects half-width tile metrics[cite: 1]. |
| **NPC Ambient Text** | Block `0x048F` @ file `0x99624`[cite: 1] | Resident panic dialogue translation[cite: 1] | Resolves 8/8 resident referrers without clobbering priest descriptors[cite: 1]. |
| **Combat Subsystem (FIX B)** | Block `0x048B` (817 sequences)[cite: 1] | Command HUD & encounter string injection[cite: 1] | `ATK ITM / SPL EQP / DFD RUN` overlay mapped cleanly[cite: 1]. |
| **Stat Progression** | Bare-page table @ LBA 40217[cite: 1] | Level-up & EXP calculation anchors[cite: 1] | Anchored 18-row Monster-Gramps table; prevents WRAM corruption[cite: 1]. |
| **Overlay Referrers** | 67 Type-46 modules across 267 sites | Dynamic overlay runtime pointer updates | Single-owner formula: $\text{decoded\_offset} = \text{imm} - (\text{hts} \times 8)$[cite: 1]. |
| **Sector Integrity** | Mode 2 Form 1 sectors[cite: 1] | Full CD-ROM EDC/ECC regeneration[cite: 1] | Two-phase `edcre` pass recalculating EDC, L-EC, and P/Q parity[cite: 1]. |

---

**3. Verification Gate Suite**

| Gate | Validation Target | Pass Criterion | Operational Status |
|---|---|---|---|
| **G0** | Image Fingerprint | Output matches target full SHA-256[cite: 1] | Enforced on every build pass[cite: 1] |
| **G1** | Resident EXE Referrers | `048C` (57/57) and `048F` (8/8) resolve cleanly[cite: 1] | Verified PASS[cite: 1] |
| **G2** | Bare-Page Table Anchor | LBA 40217 18-row table locked to stat dispatcher[cite: 1] | Verified PASS[cite: 1] |
| **G3** | Split-Immediate Logic | `lui`/`addiu` split-immediates land on bitstream boundaries[cite: 1] | Verified PASS[cite: 1] |
| **G4** | Type-39 Script Integrity | 612 cutscene scripts decompress and resolve target SIDs | Verified PASS |
| **G5** | Delta-Lock Font Metrics | Block `0x048C` SIDs 773–778 match delta `44/36/50/66`[cite: 1] | Verified PASS[cite: 1] |
| **G6** | WRAM Heap Allocation | Dynamic malloc clamp bounds party loops to `0x800F83BF`[cite: 1] | Verified PASS[cite: 1] |
| **G7** | CD-ROM EDC/ECC | Mode 2 Form 1 sector integrity clean (`badEDC={}`)[cite: 1] | Verified PASS[cite: 1] |
| **G8** | LBA Streaming Bounds | Zero sector drift across all 156,487 sectors[cite: 1] | Verified PASS[cite: 1] |
| **G9** | Live Matrix Playtest | Clean execution across critical-path surfaces[cite: 1] | Ch1 combat/level-up clean; church menu pending C convergence[cite: 1] |

---

**4. Pipeline Tooling & Component Inventory**

| Component Path | Size / Type | Function & Purpose |
|---|---|---|
| `patcher/VoidPatcher_RB1.exe` | Standalone Win-x64 | In-place end-user disc patcher; verifies SHA-256 in ~2 minutes |
| `shipB/build_pipeline_B.py` | Python Script | Master automated build pipeline; coordinates extraction, patches, and rebuild |
| `shipB/patch_slpm_native.py` | Python Script | MIPS R3000A binary patcher for resident executable structures |
| `shipB/reencode_hbd_blocks.py` | Python Script | Length-limited Huffman encoder for `HBD1PS1D.Q41` dialogue blocks |
| `shipB/remap_type39_scripts.py` | Python Script | LZSS script decompressor, SID remapper, and recompression packer |
| `shipB/patch_overlay_refs.py` | Python Script | Single-owner Type-46 runtime overlay referrer update engine |
| `shipB/patch_overlay_duplicates.py` | Python Script | Multi-copy sector synchronizer across duplicate disc zones[cite: 1] |
| `shipB/verify_gates_rebuildB.py` | Python Script | Automated test harness executing Gates G0 through G8[cite: 1] |
| `edcre/edcre.exe` | 53 KB Binary | Mode 2 Form 1 CD-ROM error detection and correction recalculator[cite: 1] |
| `translation-tools/` | 192 Files, 23 Folders | Master schemas, SID/TID dictionaries, and Font 1 atlas assets |

---

**5. Active Convergence Roadmap (Rebuild B $\rightarrow$ Rebuild C)**

| Subsystem Task | Owning Worker | Immediate Blocker / Scope | Target Resolution |
|---|---|---|---|
| **Task A1 (Gate G2)** | Worker A | Church facility overlay menu text renders garbled[cite: 1] | Remap stale `0x48C0` SIDs 655–658 within decompressed Type-46 overlay[cite: 1] |
| **Task A3 (Gate G4)** | Worker A | Duplicate sector divergence causes room transition halts[cite: 1] | Sync byte parity across pairs `0x00A2`, `0x00A9`, `0x00B4`[cite: 1] |
| **Task A4 (Gate G3)** | Worker A | Endor mega-block entry freeze risk in Chapter 3[cite: 1] | Remap all 1,086 Type-44 referrers targeting TID `0x0021`[cite: 1] |
| **Task B1** | Worker B | Chapter 4 Palais de Léon & refuge event triggers[cite: 1] | Dialogue & pointer remaps for `0x027E` and `0x0280`[cite: 1] |
| **Task B2** | Worker B | Chapter 5 shared TID collision between Yggdrasil & Sky Tower[cite: 1] | Realignment pass for shared container `0x035C`; slot-8 guest handling[cite: 1] |
| **Task B4** | Worker B | Healie well staircase DMA streaming freeze risk[cite: 1] | Verify DMA3 `$I_STAT` Bit 2 wait-loop and cache flushes at TID `0x006C`[cite: 1] |
