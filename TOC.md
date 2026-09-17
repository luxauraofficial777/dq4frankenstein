# TABLE OF CONTENTS — DRAGON QUEST IV PSX REVERSE-ENGINEERING ARCHIVE

**Repository:** `luxauraofficial777/dq4frankenstein`  
**Target:** Sony PlayStation 1 (`SLPM_869.16`) / HeartBeat Engine (`HBD1PS1D.Q41`)  
**Maintained By:** Zane (Lux Aura / Void Walkers Project)  
**Last Updated:** September 16, 2026  

---

## 1. PROJECT METADATA & REPOSITORY ESSENTIALS

* **[`README.md`](README.md)** — Master overview of the project, quickstart compilation steps, VoidPatcher RB1 integration, and build harness instructions.
* **[`how_we_did_this.md`](how_we_did_this.md)** — Architectural methodology, reverse-engineering ethos, and public domain attribution principles.
* **[`VERSION_LOG.md`](VERSION_LOG.md)** — Chronological release ledger detailing patch iterations from Frankenstein V.90 up through the Sovereign Native Pipeline.
* **[`CREDITS.md`](CREDITS.md)** — Comprehensive credits ledger, historical preservation contributors, and research citations.
* **[`LICENSE`](LICENSE)** — Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0).
* **[`TOC.md`](TOC.md)** — This comprehensive structural index and document taxonomy.

---

## 2. NAKAMURA & YAMANA WHITE PAPER SUITE (ENIX ENGINE LINEAGE)

* **[`CASE_STUDY_NAKAMURA_YAMANA_HEARTBEAT_ENGINE_IEEE_2026.md`](CASE_STUDY_NAKAMURA_YAMANA_HEARTBEAT_ENGINE_IEEE_2026.md)** — IEEE-format academic case study analyzing Koichi Nakamura and Manabu Yamana's systems engineering from the Super Famicom era to the 32-bit PlayStation architecture.
* **[`YAMANA_NAKAMURA_HEARTBEAT_ENGINE_GENERATIONAL_ARCHITECTURE.md`](YAMANA_NAKAMURA_HEARTBEAT_ENGINE_GENERATIONAL_ARCHITECTURE.md)** — Generational evolution monograph tracing data structures, memory banking, and bytecode dispatchers across DQ3, DQ4 PSX, and DQ7.
* **[`NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG1.md`](NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG1.md)** — *Part 1:* Architectural foundation, R3000A systems programming, and disc sector topology.
* **[`NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG2.md`](NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG2.md)** — *Part 2:* SRAM battery-backed memory allocation, save-slot integrity routines, and checksum validation algorithms.
* **[`NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG3.md`](NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG3.md)** — *Part 3:* Super Famicom DQ3 ExHiROM map-archive framing status and memory banking resolution.
* **[`NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG4.md`](NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG4.md)** — *Part 4:* Engine provenance documentation, historical codebase evolution, and unresolved engine boundary gaps.
* **[`NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG5.md`](NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG5.md)** — *Part 5:* Final empirical metrics, dynamic memory footprints, and cross-platform timing measurements.

---

## 3. ENIX ENGINE HARDWARE SPECIFICATION SUITE & DMA SUBLAYERS

* **[`YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md`](YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md)** — Formal hardware specification detailing container framing, 24-byte sub-block headers, bitstream alignment, and HBD memory management.
* **[`YAMANA_HBE_HBD_DMA_SUBBLOC_ROUTINE.md`](YAMANA_HBE_HBD_DMA_SUBBLOC_ROUTINE.md)** — Detailed specification of Yamana's CD-ROM DMA Channel 3 (`0x1F8010B0`) streaming handler, MIPS L1 D-cache flushing (`0x80018C20`), and uncached KSEG1 mirror decompression.
* **[`DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md`](DMA_SUBBLOCK_COMPRESSION_DISPATCH_AUDIT_Sep14_2026.md)** — Analysis of the three disjoint container classes (Wide-Huffman, DQLZS, and RAW Passthrough), resolving font-page latching and replacing the flawed `flags == 0x0500` rule.
* **[`MASTER_DUNGEON_WORLD_TRIGGERS_AND_DMA_SUBLAYER_LIBRARY.md`](study/MASTER_DUNGEON_WORLD_TRIGGERS_AND_DMA_SUBLAYER_LIBRARY.md)** — Comprehensive master census covering all Chapters 1–6 3D dungeon collision meshes, DMA burst intervals, event locks, and Chapter 5 Olin (Slot 8) door breach mechanics.

---

## 4. MASTER LIBRARIES, DICTIONARIES & SCHEMAS

* **[`YAMANA_HBE_MASTER_TID_LIBRARY.md`](YAMANA_HBE_MASTER_TID_LIBRARY.md)** — Complete census of all Table Identifiers (TIDs), archive byte offsets, sub-block layouts, and chapter allocations (0x0020 through 0x006F).
* **[`YAMANA_HBE_MASTER_SID_LIBRARY.md`](YAMANA_HBE_MASTER_SID_LIBRARY.md)** — Master string identifier index mapping packed 32-bit references `(TID << 20) | BitOffset` across the entire dialogue and script corpus.
* **[`YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md`](YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md)** — Reference manual for all 8-byte VM control codes, formatting tokens, ASCII plane switches (`{7f0b}`), and voice/SFX triggers.
* **[`KING_MEDAL_SUMMARY.md`](KING_MEDAL_SUMMARY.md)** — Complete inventory, memory flags, and reward validation tables for the King Medal collection sub-system.
* **[`facility_marker_worksheet.json`](facility_marker_worksheet.json)** — Machine-readable coordinate mapping for all church, inn, weapon, armor, and item shops disc-wide.

---

## 5. FORENSIC AUDITS, MEMORY MINING & ENGINE INVARIANTS

* **[`FORENSIC_AuldWell_RingManagers_TaskTable_Sep16_2026.md`](FORENSIC_AuldWell_RingManagers_TaskTable_Sep16_2026.md)** — Static disassembly and live probe confirming the 8-entry task table (`0x800AE5F8`), identifying the `0x80031DEC` ring wipe routine and the `0x80031F0C` directory builder.
* **[`FORENSIC_0x800357D4_AuldWell_Gate_REFUTATION_Sep16_2026.md`](FORENSIC_0x800357D4_AuldWell_Gate_REFUTATION_Sep16_2026.md)** — Ground-truth byte refutation of the KSEG3 underflow hypothesis; proves `0xFFFFF000` is an immutable index-25 mirror-table constant.
* **[`DISC_WIDE_COMPARATIVE_AUDIT_Sep16_2026.md`](DISC_WIDE_COMPARATIVE_AUDIT_Sep16_2026.md)** — Sector-by-sector audit across Pristine JP, Ship B, Ship C, and Sovereign Master; pinpoints missing EXE patches and un-remapped overlays.
* **[`DISC_AUDIT_DATA.json`](study/DISC_AUDIT_DATA.json)** — Machine-readable LBA run maps, differential sector lists, and PVD descriptor verifications across all four builds.
* **[`CHAPTER_1_TO_6_INVARIANT_SPEC.md`](CHAPTER_1_TO_6_INVARIANT_SPEC.md)** — Global compiler and runtime invariant specification; establishes the Tier 3 MIPS trampoline guard at `0x80031DEC`.
* **[`CYBERGRIME_RAM_MINE_0x80031E34_RebuildC_Sep16_2026.md`](CYBERGRIME_RAM_MINE_0x80031E34_RebuildC_Sep16_2026.md)** — RAM differential analysis between Pre-Stairs (228/230) and Post-Stairs (229/231); documents the 894-node ring collapse and unblank latch clearing.

---

## 6. HISTORICAL REVERSE-ENGINEERING MONOGRAPHS & BLUEPRINTS

* **[`HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE.md`](HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE.md)** — Comprehensive historical survey of community translation efforts, technical dead ends, and engine breakthroughs across two decades.
* **[`HBE_GENERATIONAL_TRANSLATION_HISTORY.md`](HBE_GENERATIONAL_TRANSLATION_HISTORY.md)** — Internal lineage documentation chronicling tools, decompressors, and patch formats used during the project.
* **[`HBD_PROCESSING_BLUEPRINT_Jul19_2026.md`](HBD_PROCESSING_BLUEPRINT_Jul19_2026.md)** — Original processing pipeline architecture for HBD container extraction and rebuilding.
* **[`HLRM_INTEGRATION_BLUEPRINT.md`](HLRM_INTEGRATION_BLUEPRINT.md)** — High-Level Resource Management integration specification.
* **[`INSTRUMENTATION_BLUEPRINT.md`](INSTRUMENTATION_BLUEPRINT.md)** — Dynamic instrumentation, hardware breakpoint triggers, and logging specifications for emulator harnesses.
* **[`FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md`](FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md)** — Summer 2026 critical bug triage and pipeline stabilization audit.
* **[`TELEMETRY_DIFF_REPORT_Jul20_2026.md`](TELEMETRY_DIFF_REPORT_Jul20_2026.md)** — Comparative instruction count and VBlank execution telemetry report.
* **[`VECTOR_ANALYSIS_REPORT.md`](VECTOR_ANALYSIS_REPORT.md)** — Analysis of MIPS hardware exception vectors and interrupt prioritization routines.

---

## 7. COMPARATIVE DRAGON QUEST & DRAGON WARRIOR STUDIES

* **[`Dragon Quest IV PSX Engine Analysis.md`](Dragon%20Quest%20IV%20PSX%20Engine%20Analysis.md)** / **[`.pdf`](Dragon%20Quest%20IV%20PSX%20Engine%20Analysis.pdf)** — Core technical analysis of DQ4 PSX systems, memory architecture, and disc streaming.
* **[`Dragon Warrior VII Reverse Engineering Study.md`](Dragon%20Warrior%20VII%20Reverse%20Engineering%20Study.md)** / **[`.pdf`](Dragon%20Warrior%20VII%20Reverse%20Engineering%20Study.pdf)** — Deep reverse-engineering study of the companion HeartBeat Engine implementation in *Dragon Warrior VII*.
* **[`Dragon Quest Engine Analysis & Cross-Referencing.md`](Dragon%20Quest%20Engine%20Analysis%20%26%20Cross-Referencing.md)** / **[`.pdf`](Dragon%20Quest%20Engine%20Analysis%20%26%20Cross-Referencing.pdf)** — Cross-referenced engine study comparing byte dispatch tables, sound drivers, and script interpreters across PSX DQ releases.
* **[`Dragon_Quest_ROM_Hacking_Schema.md`](Dragon_Quest_ROM_Hacking_Schema.md)** / **[`.pdf`](Dragon%20Quest%20ROM%20Hacking%20Schema.pdf)** — Structural schema documentation for memory layouts, dialogue pointers, and font metrics.
* **[`DQ4_PSX_Engine_Analysis_extracted.md`](DQ4_PSX_Engine_Analysis_extracted.md)** — Extracted structural notes and raw disassembly references for DQ4 PSX.
* **[`DW7_RE_Study_extracted.md`](DW7_RE_Study_extracted.md)** — Extracted structural notes and raw disassembly references for DW7.
* **[`dq4study.md`](dq4study.md)** / **[`dq7study.md`](dq7study.md)** — Working scratchpads and technical field notes for DQ4 and DQ7 disassemblies.
* **[`lineage.md`](lineage.md)** / **[`generation.md`](generation.md)** — Comparative architectural notes on the lineage and generational shifts in Enix software development.
* **[`STUDY_LIBRARY_ANALYSIS.md`](STUDY_LIBRARY_ANALYSIS.md)** — Meta-analysis of the research study library, validating cross-references and metrics.

---

## 8. BUILD HARNESSES, EMULATOR SUITES & DISTRIBUTION PACKAGES

* **[`V99_ReBuildB_DOCUMENTATION.md`](V99_ReBuildB_DOCUMENTATION.md)** — Technical documentation and deployment guide for the V.99 Rebuild B candidate disc.
* **`DQ4_Patcher_RebuildB_QuickStart.zip` (`.z01`–`.z05`)** — Multi-volume binary distribution package containing the standalone Rebuild B quickstart patcher suite.
* **`dist_rebuild_b.zip` (`.z01`–`.z06`)** — Complete multi-volume distribution build for Rebuild B.
* **`shipB.zip` (`.z01`–`.z06`)** — Verified master build tree and binary assets for the Ship B milestone.
* **`cybergrime.zip` (`.z01`–`.z07`)** — Complete source and binary distribution of the CyberGrime PSX debugging harness, timeline tracer, and regression test runner.
* **`frankenstein_pipeline_v090.zip`–`V.99.zip`** — Archived pipeline milestones tracing the evolution from early Python extraction scripts to the unified C++ native build toolchain.
* **`study.zip`** — Complete archival pack containing original research notes, disassembly maps, raw binary dumps, and intermediate analysis tools.
