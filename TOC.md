# Dragon Quest IV: The Zenithian Chronicles — Master Repository Table of Contents

**Repo:** `luxauraofficial777/dq4frankenstein` · **Branch:** `main` · **Current build:** V.99 Rebuild B (Sovereign Native / Live Playable)
**Compiled:** 2026-09-13 · **License:** CC BY-NC-SA 4.0

This document is a complete, annotated map of the repository: every deliverable, every
pipeline release, every documentation file, and the technical role each one plays in the
**Sovereign Native re-authoring of the Japanese HeartBeat Engine** (`SLPM_869.16` /
`HBD1PS1D.Q41`) and the historical DW7-EXE "Frankenstein" graft research that preceded it.

---

## Key Technical References (used throughout)

| Constant | Value |
|---|---|
| Source disc | `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin`, 368,057,424 B, Mode 2 Form 1, 156,487 LBA |
| Source CRC-32 | `3D67C858` |
| Source SHA-256 | `100D87DB9DEADF8F9FA4BB891D3A5D0BB112ACBF5ADBCBC93C637848ED9C7531` |
| Master output SHA-256 | `AC9F94A13A5627C30013FB0D13C88CD96F4BCEC878A2DA6E7B1978E1CD4E5703` |
| HBD archive | `HBD1PS1D.Q41`, 319,436,800 B, 155,975 × 2,048-B blocks, starts LBA 362 |
| EXE boot | `SLPM_869.16`, load `0x80017F00`, PC0 `0x8008E284` |
| Dialogue containers | 1,358 blocks re-encoded via length-limited Huffman (depth 14..9) |
| Cutscene scripts | 612 Type-39 blocks (LZSS decompress → remap → recompress) |
| Font 1 | Block `0x048C` @ file `0x97AC8`, 8×14 half-width, delta-locks 44/36/50/66 (SIDs 773–778) |
| Battle overlay | Block `0x048B`, 817 sequences |
| Stat anchor | LBA 40217 (18-row Monster-Gramps table) |
| Referrer formula | `ReferrerWord = (BlockID ≪ 20) \| (BitOffset + HeaderTreeSize × 8)` |
| HBE limits | 20-bit per-block ceiling (128 KB); `{0000}` mandatory terminator; zero-sector-shift invariant |

---

## 1. Distribution & End-User Deliverables

Buildable/applyable artifacts end-users actually consume. These are the "two ways to play"
referenced by the README (VoidPatcher fast path vs. BuildB / shipB pipeline).

| File(s) | Size | Role |
|---|---|---|
| `DQ4_Patcher_RebuildB_QuickStart.zip` (+ `.z01`–`.z05`) | ~29 MB | **Quick-Start patcher bundle** — VoidPatcher-style sealed payload for Rebuild B. Split into 5 MB chunks to honor GitHub's 100 MB/file cap; reassemble in order before use. |
| `shipB.zip` (+ `shipB.z01`–`.z06`) | ~33 MB | **Ship B build pipeline** — the full `shipB/` distribution (source image expected one level above; `edcre` runner, build scripts, gates). Output `shipB\build\dq4_shipB.bin` + `.cue`. |
| `dist_rebuild_b.zip` (+ `.z01`–`.z06`) | ~30 MB | **Rebuild B distribution tree** — mirror of the release payload used for the `dq4_rebuildB_sealed.bin` gate suite. |
| `cybergrime.zip` (+ `.z01`–`.z07`) | ~37 MB | **CyberGrime MIPS emulator harness** — the trace-verified emulation environment used for instrumentation (named-pipe bus `cybergrime_trace`, G0–G8 gate verification). |

---

## 2. Pipeline Release Archives (Version Lineage)

The full engine/toolchain history as snapshotted deliverables. `V.98`/`V.99` are the
Sovereign Native era; `v090`–`v097` are the exploratory DW7-EXE "Frankenstein" graft era.

| File | Size | Era / Meaning |
|---|---|---|
| `frankenstein_pipeline_V.99.zip` | 1.8 MB | **Sovereign Native Rebuild B toolchain** — current lineage: in-place patcher, Type-39 remapper, overlay referrer scanner, gate verifier (G0–G9). *(Registry master for this release is also posted in the "V.99 ReBuild B" GitHub Release.)* |
| `frankenstein_pipeline_V.98.zip` | 612 KB | **V.98** — late-graft architectural corrections (HBD block-header conversion A/B/C, `DQ4_HBD_END` fix at `psx_binary_ops.h:124`, PVD volume-space-size fix) before the sovereign pivot. |
| `frankenstein_pipeline_v097.zip` | 24.9 MB | **V.97** — last full Frankenstein (DW7 `SLUS_012.06` graft) deliverable; documents the graft dead-end and the LBA 354→362 relocation logic. |
| `frankenstein_pipeline_v096.zip` | 24.6 MB | **V.96** — restored XA/STR (`0x8008AEF4`) + MDEC (`0x8008CAD0`) stubs; stable 59.82 FPS but GPU 0.00 (async VBlank/DMA event not signaled). |
| `frankenstein_pipeline_v095.zip` | 24.6 MB | **V.95** — `split_addr()` signed-carry helper; sector-aligned text size `0xA6000`; left BIOS clean but MDEC spun at 569 FPS on missing FMV LBA 146621. |
| `frankenstein_pipeline_v090.zip` | 2 B *(stub)* | **V.90** — placeholder artifact; content was never populated (the 25-step graft plan itself is preserved in `VERSION_LOG.md` + `HBD_PROCESSING_BLUEPRINT`). |

---

## 3. Master Reference Libraries — HeartBeat Engine (HBE)

The technical core of the research: three machine-verified lookup libraries that map the
entire DQ4 PSX text/script dispatch system of **Manabu Yamana's HeartBeat Engine**.

| File | Size | Content |
|---|---|---|
| `YAMANA_HBE_MASTER_TID_LIBRARY.md` | 181 KB / 1,319 lines | **TID census (largest doc)** — "Exhaustive Decompiled Text Block (TID) & Sub-Block Census." 3,243 physical blocks / 23,828 functional sub-blocks across ~47 type rows; master catalog of **1,111 unique TIDs** in 11 narrative bands (Ch.1–5, party chat 8,052 SIDs, church/facilities, battle/menus); 40 multi-copy TIDs requiring lock-step patching; 24-byte sub-block header spec; critical TIDs incl. Endor mega-block `0x0021` (93,140 B / 1,087 strings), `0x048B` (817 SIDs, freeze hazard), `0x048C` delta-locks, `0x006C` 1,588-byte clamp. |
| `YAMANA_HBE_MASTER_SID_LIBRARY.md` | 36 KB / 271 lines | **SID matrix** — "Definitive Decompiled String Dispatch & Referrer Memory Matrix." **19,193 total SIDs** across 1,108 blocks + 4 overlays + 403 Type-39 scripts; packed 32-bit ReferrerWord formula; Font 1 / Font 2 dual-blitter model; 51-code `{7Fxx}` taxonomy with measured counts; "Alien Text Syndrome" (`peynriohre-seale`) & Runaway Decode Freeze; critical-SID registry with per-string budgets (`048B:0102`, `048C:0773–0778`, `0474:0054`, `006C:0001`). |
| `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md` | 21 KB / 259 lines | **Cross-engine control-code map** — function→code tables for Famicom (DW1/2/4), SFC (DQ1+2, DQ3 13-bit huff, DQ6), PSX DQ4 HBD, and DS DQ4; verified against RadMageIRL hardware measurements (RM), Wilkens tooling (MW), and on-disc census (MS); 51 codes on disc (43 archive + 8 EXE); `7Exx` block-local dictionary refs, `FExx` facility band; conflicts ledger + unresolved-code backlog. |
| *(supporting, referenced)* `translation/control_code_mapping.json` | — | Machine-readable 43-family PSX control-code mapping (referenced by the CONTROL_CODES library). |

---

## 4. Engine Studies & Analysis

In-depth reverse-engineering writeups of the two engines under study (DQ4 PSX hybrid and
DW7/DQ7 monolithic), plus the cross-generational ROM-hacking schema. Three content
"families" exist — polished study, working notebook (draft + generator + generated
artifact), and extracted clean version.

### 4A. DQ4 PSX (HeartBeat Hybrid, `SLPM_869.16` + `HBD1PS1D.Q41`)
| File | Size | Notes |
|---|---|---|
| `Dragon Quest IV PSX Engine Analysis.md` (+ `.pdf`) | 10.5 KB | Boot flow, executable bridge, BSS cleansing `0x8008E284`, DQVII→DQIV memory reallocation table, sprite-to-3D-grid init `0x8005C1F0`, Q41 sector layout, sub-block type catalog (1/6/8/10/13/21/32/39), dialogue parser. *Draft ends mid-section 4.* |
| `DQ4_PSX_Engine_Analysis_extracted.md` | 12.8 KB | Clean extracted/expanded version — completed through sections: architecture, executable bridge, asset table + LBA structure, dialogue parsing & conditionals (`%A/%B/%H`, Alena ID 120), sprite-grid sync & VRAM billboard math (Doc ID `ARCH-STUDY-PSX-DQ4-HYBRIDIZATION-20260913`). |
| `dq4study.md` | 38.6 KB | Working notebook: draft + Python generator producing `ARCH_STUDY_PSX_DQ4_HYBRIDIZATION_FINAL.md`, then the generated artifact inline. |
| `Dragon Quest Engine Analysis & Cross-Referencing.md` (+ `.pdf`) | 3 KB | Cross-generational divergence study (RICOH 5A22 SFC vs MIPS R3000A PSX): solid-state vs optical streaming, memory-partition topology. *Draft; truncated at section 2 intro.* |

### 4B. DW7 / DQ7 (Monolithic, `SLUS_012.06` / `SLPM_865.00`, `HBD1PS1D.Q71`)
| File | Size | Notes |
|---|---|---|
| `Dragon Warrior VII Reverse Engineering Study.md` (+ `.pdf`) | 27.9 KB | CD-ROM streaming chain (`open_file` `0x80076040` → `mopen_file` `0x80082598` → DMA4 → `0x80138000`), upscaling-blur discrepancy, event flag matrix `0x8000F800`, Immigrant Town 8-phase table, Q71 archive hierarchy, LZSS0 algorithm + 16-bit tile descriptors. Opens mid-analysis with generator script → generated study. |
| `DW7_RE_Study_extracted.md` | 16.7 KB | Clean extracted version of the DW7 study — complete through section 7 incl. "Save Anywhere" range `0x8000F83C–0x8000F846`, "Equip Any Armor" target `0x80069BCA`, `SEQq` sound driver 60-byte header, `0x60010108` control blocks. |
| `dq7study.md` | 20.8 KB | Clean writeup of the DW7 monolithic engine: full 2 MB memory partition map, custom `SEQq`/`qQES` sound driver spec, dynamic CD-ROM streaming, event flag matrix, Q71 archive layout, LZSS0 + isometric assembly. |

### 4C. Cross-Generational Schema & Migration
| File | Size | Notes |
|---|---|---|
| `Dragon_Quest_ROM_Hacking_Schema.md` (+ `.pdf`) | 15.1 KB | "Reconstitution of the HeartBeat Engine" — the master hacking schema: FC→SFC→PSX data-evolution timeline, `.HBD` physical packer spec (16-byte master + 16-byte sub-block headers, `0x0500` LZSS), Huffman tree cipher & `C021A0` VM parser, proprietary `SEQq`/`qQES` sound driver, historical localization graft matrix (Q41→Q71), sovereign reconstitution parameters. |
| `generation.md` | 20.8 KB | Notebook-style: Python generator → `STUDY_MIGRATION_HBE_SFC_TO_PSX_Sep13_2026.md` (Doc ID `ARCH-STUDY-MIGRATION-HBE-SFC-TO-PSX-20260913-V2`) — SFC→PSX divergence, hidden commonalities (Monsters 50×8 B, Items 128×2 B …), 2 MB memory map, SFC→PSX control-code mapping, HBD/LZSS0, decoupled battle pipeline & town streaming. |

---

## 5. Blueprints & Implementation Specifications

Engineering specs for the toolchain — the DW7-graft conversion pipeline, the comparison
simulation environment, the live instrumentation bus, and the final sovereign architecture.

| File | Size | Content |
|---|---|---|
| `FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md` | 15.7 KB | **Pivotal decision doc.** Binary-isomorphism proof that DQ4 per-block Huffman trees ≡ DW7 global hybrid tree (root_id `+1`, odd-vs-even `offset_b`, fix ≈ 3–5 MIPS instructions); the **Graft vs Rewrite decision (Option A: DQ4 disc + patched DW7 EXE)**; disc-layout feasibility (EXE LBA 24–361 slot, HBD @ 362); FMV black-screen root cause & XA/MDEC stubbing; full ~15–20 patch MIPS instruction plan; build Phases 1–4. |
| `HBD_PROCESSING_BLUEPRINT_Jul19_2026.md` | 18 KB | **DW7-EXE HBD conversion spec.** DQ4→DW7 block-header diff (hts 24 vs ~1366, inline vs global tree), Transformations A/B/C (block-header rewrite, sub-block field swap @ `0x800562E4`, LBA −7 relocation), `HbdReencoder::process_hbd()`, `build_v39()` integration, Gate 3 validation (DuckStation boot criteria). |
| `HLRM_INTEGRATION_BLUEPRINT.md` | 29.3 KB | **HLRM (High-Level Reference Model) sim environment.** CyberGrime-based comparative simulation feeding live traces to a Python sidecar vs a "golden" constraint map; three tree-placement options (A/B/C = v40a/v40b/v40c), constraints engine, Comparator levels, Huffman/LBA normalizer, telemetry pipeline, black-screen diagnosis (hang after LBA 146621). |
| `INSTRUMENTATION_BLUEPRINT.md` | 55.7 KB / 1,250 lines | **Live dynamic verification spec.** Named-pipe Instrumentation Bus (`cybergrime_live`) streaming register/memory state to a Python sidecar bridged to Ghidra; Golden Model checkpoints, deterministic patch injection (WRITE_MEM/REG, SET_PC, ROLLBACK…), divergence logs, targeted Ghidra decompilation map (`dw7_function_map.json`: huffman `0x80073670`, `0x80084BF0`, bss `0x8008E284`, cdrom `0x80098898` / `0x800D9E80` …), MIPS quick-reference table, Sprint 1–3 priority. |
| `V99_ReBuildB_DOCUMENTATION.md` | 7.1 KB | **Sovereign Native spec for the current build** — subsystem transformation matrix (1,358 Huffman blocks, 612 Type-39 scripts, `048C`/`048B`/`048F` injection), verification **Gate Suite G0–G9** (fingerprint → referrers → LBA anchor → split-immediates → Type-39 integrity → delta-locks → heap bounds → EDC/ECC → streaming bounds → live playtest), tooling inventory, Rebuild B→C convergence roadmap. |
| `TELEMETRY_DIFF_REPORT_Jul20_2026.md` | 25.8 KB | **Automated diff/telemetry report.** CyberGrime harness comparing `dq4_jp_ref / dw7_us_ref / v44_frank / v43_frank` across 11 sections (EXE header, memory map, ISO/LBA, 5M-instruction telemetry, sector reads + DMA4, BSS/stack, VRAM, AI kernel parity, offset deltas, root cause, binary diffs). Proves v44 stalls before display init (BSS zeroed thread entry `0x800D9E80`). |
| `VECTOR_ANALYSIS_REPORT.md` | 91 KB / 1,296 lines | **Crash-vector analysis.** Confirms BSS clear destroys the CD-ROM thread entry; post-BSS-clear injection fix (Option B / `0x76B88` alternative); **39 crash vectors / 5 collision zones / 11 patch sites**; BIOS-call relationship layer (4,773 calls, 3,237 mappings); 2,270-address index; hand-off deliverables. |

---

## 6. History, Method, Lineage & Provenance

The "why" — how this was done, the failed predecessors, and the studio/lineage history.

| File | Size | Content |
|---|---|---|
| `VERSION_LOG.md` | 7.3 KB | **Definitive lineage timeline (V90 → Rebuild B).** Per-version failure analysis of the graft era (V90 BIOS POST 03 stall; V95 MDEC 569 FPS; V96 GPU 0.00; V97 dead-end w/ `DQ4_HBD_END` fix) and the Sovereign Native breakthrough (Gates G0–G8 PASS, 59.94 FPS), plus the comprehensive version comparison matrix. |
| `HBE_GENERATIONAL_TRANSLATION_HISTORY.md` | 5.6 KB | **SFC vs PSX translation-lineage comparison** — DQ6 (NoPrgress/Pegazaru/DeJap), DQ3 SFC (Byuu/DQ Translations/DeJap), DQ4 PSX (Sovereign Native); the **"HeartBeat Vulnerability Triad"** (bit-phase desync, hardcoded literal-bit-offset referrers, scratchpad collisions) and "Generational Rosetta Stone" table mapping historic SFC bugs → sovereign fixes → gates. |
| `HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE.md` | 11.2 KB | **"Great Filter" historical study** — the three-phase cycle (teaser mirage → multi-year impasse → sovereign breakthrough) as proved by DQ6 (Murdaw's Keep), DQ3 (Shanpane Tower / RPGOne + Stealth), DQ1&2 (bank overflow → 16/24 Mbit expansion); argues DQ4 PSX **Chapter 2 is the functional watershed** (10-slot roster, Endor `0x0021`, multi-copy coherence G4). |
| `how_we_did_this.md` | 4.1 KB | **Methodology post-mortem** for the community — clean-room agentic toolchain (LIMINAL LORE): DW7-as-English-chassis "heart transplant", Font 1 anchor `0xD60`, MIPS pointer remapping (Sector 3191 redirections; 1202/1202 referrers resolved), sovereign refs overlap coefficient vs legacy tooling, EDC/ECC-automated Mode 2 Form 1 rebuilder. |
| `lineage.md` | 6.9 KB | **Forensic effort matrix + studio lineage** — 3,700–4,800 effort-HR across 5 tracks (127+ failed DW7 graft sites; 1,128-block Huffman; 121 Python tools; Track B "Zenithian Forge" SFC ExHiROM backport); studio genealogy: Chunsoft → **HeartBeat (Yamana/Eriguchi, 1992–2002)** → Tri-Ace; generational engine evolution; credit-disparity analysis. |
| `CREDITS.md` | 5.7 KB | **Credits & acknowledgments** — leadership (Lux Aura / Big Pickle / Omen Alpha), foundation research (Markus Schroeder, Mandy Wilkens, RadMageIRL), AI/neural cohort (KIMI, GLM, Claude Opus, Gemini, Nemotron, Cascade), tools (Corlett's `ecmtools`, DuckStation/Stenzek), heritage. |
| `KING_MEDAL_SUMMARY.md` | 8.9 KB | **Macro scale/cost audit** — measured repository scale (107,242 files / ~48.6 GB[^1]), what the RE produced (including 18,484 strings / 51-code control-token library), AI compute Fermi estimate, named human labor, traditional-studio pricing comparison, efficiency-ratio argument. |
| `STUDY_LIBRARY_ANALYSIS.md` | 5.8 KB | **Forensic "Library of Alexandria" inventory** — 859 MD studies / 137,551 lines; 191 Python files; 53 JSON (1,163,206 lines); 106 RAM dumps / 532 MB; compute vs Y2K equivalence (~15,800 yrs); human-labor equivalent (10.5k–14.4k hrs); corporate valuation matrix. |

[^1]: Scale figures in KING_MEDAL_SUMMARY/STUDY_LIBRARY_ANALYSIS count extended pre-release worktrees and differ from the compressed `main` tree (238.7 MB working tree as of this audit) plus full history.

---

## 7. Repository Metadata & Supporting Assets

| File | Size | Role |
|---|---|---|
| `README.md` | 16.8 KB | Repo front page: project overview, architecture diagram, pedigree, in-game showcase, VoidPatcher + BuildB instructions, bug-reporting protocol, legal/credits/links. |
| `LICENSE` | 1.1 KB | CC BY-NC-SA 4.0 (custom text; attribution → Lux Aura & VoidWalkers Research Project). |
| `facility_marker_worksheet.json` | 264 KB | Machine-readable facility/facility-variable marker parity worksheet (church/save/divination string markers) used by the `g8_marker_parity` validation lane. |
| `dq4.png` | 241 KB | Cover/banner artwork. |
| `study.zip` | 24.2 MB | Study-library snapshot export (the textual research library described in `STUDY_LIBRARY_ANALYSIS.md`). |
| *(compiled separately)* `V.99 ReBuild B` GitHub Release | — | Tag `dq4frankenstein` on `main`; release notes carry the source/dest hashes, technical highlights, and Rebuild C roadmap. |

---

## Appendix A — Repository Shape (compressed `main` tree)

- **70 blob entries** at root (git tree), 238.7 MB total working tree (measured at audit time).
- **35 archive/pipeline binaries** — 4 split bundles (`DQ4_Patcher_RebuildB_QuickStart`, `shipB`, `dist_rebuild_b`, `cybergrime`; 24 `.zNN` parts + 4 `.zip`), 6 `frankenstein_pipeline` zips (V.98/V.99/v095/v096/v097 + the 2 B `v090` stub), and the `study.zip` snapshot.
- **28 Markdown documentation files** covering every layer of the research (3 master HBE libraries, 11 engine-study docs, 5 blueprints, 3 reports, 6 history/method docs).
- **4 PDF renderings** of the analysis docs, plus `dq4.png` banner and `facility_marker_worksheet.json`.
- Languages (per topics): `python` / `python3`, `cpp`, `java` (jar-based early tooling), plus the emulation-side C++ harness.
- Topics: romhacking, psx, jrpg, huffman-compression-algorithm, hbd, hbe, translation, translation-tool, and 12 others.

## Appendix B — Known Internal Inconsistencies (discovered during TOC audit)

Documented here so future readers are not confused by cross-file drift:

1. **Block counts:** TID library = 1,111 unique TIDs / 3,243 blocks / 23,828 sub-blocks; SID library cites **1,108 blocks** on disc (spread across "1,108 archive blocks + 4 overlays"). Same engine, different census granularity.
2. **Control-code totals:** CONTROL_CODES library says **51 codes (43 archive + 8 EXE)**; SID library says **51 codes (43 archive + 7 EXE)** — a 7-vs-8 EXE split discrepancy, though both totals agree at 51.
3. **Per-code counts:** e.g., `{7F02}` newline measured at **166,235** (SID) vs **166,134** (CONTROL_CODES table).
4. **File names in docs:** `how_we_did_this.md` references `SLPS-031.70`, while the canonical analysis stack uses `SLPM_869.16` / `SLPM-869.16`. Minor, but the docs do not fully agree on the boot-file identifier string.
5. **Gate numbering:** the blueprint/report cluster references G0–G9 (V99 spec), single "Gate 3" (HBD blueprint), and gate letters G1–G11 in credits; the G-numbering was extended during the work and the docs were not uniformly renumbered.
6. **Truncated drafts in `main`:** `Dragon Quest IV PSX Engine Analysis.md` and `Dragon Quest Engine Analysis & Cross-Referencing.md` end mid-sentence in the repo; their full content lives only in the `_extracted` / notebook siblings (sections 4A/4C). Their `.pdf` companions are the completed renderings.
7. **`frankenstein_pipeline_v090.zip` is a 2-byte stub** — flagging for completeness; all real V.90 state is preserved in `VERSION_LOG.md` and `HBD_PROCESSING_BLUEPRINT.md`.

---

*ORDER. PRECISION. FIDELITY.*