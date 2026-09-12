# How We Translated Dragon Quest IV PSX to English

A technical post-mortem and architecture overview for the ROM hacking and reverse-engineering community.

---

## 1. Project Overview & Sovereign Architecture

*Dragon Quest IV: Michibikareshi Monotachi* (PSX, Japan, SLPM-869.16 / SLPS-031.70) has stood for nearly 25 years as one of the console scene's most stubborn untranslated titles. Prior efforts frequently broke against the HeartBeat engine’s non-linear memory maps, hardcoded MIPS pointer branches, and delicate sector integrity constraints.

This release represents a bootable retail disc (`dq4_sovereign_master.bin`) built via **LIMINAL LORE**—a sovereign, clean-room agentic toolchain directed by **Lux Aura**. By combining human systems architecture with a coordinated multi-LLM engineering roster, the project achieved a native English text-rendering and font-injection pipeline under a public domain dedication (**CC0 1.0 Universal**).

---

## 2. The Multi-Agent Engineering Roster

Rather than relying on closed, legacy third-party binary patches, the toolchain was built, debugged, and audited across a heterogeneous agent platoon under strict human direction:

* **Human Pilot / Systems Architect (Lux Aura):** System state preservation, hardware constraints, binary auditing, and hallucination containment.
* **Lead Coder & Systems Architecture (Gemini 3.8 / Flash):** Authored the core Python/MIPS injection routines, sector repointers, native EDC/ECC automation, and sovereign ISO rebuilder modules.
* **Structural Decompilation & Logic Verification (Omen Alpha):** Deep analytical isolation of font atlas memory offsets, sector boundaries, and pointer tables.
* **Deep Analysis & Patcher Hardening (Kimi):** CLI driver wrappers (`TextPatcher`), cross-title HBD/EXE filename handling, sequence error diagnostics, and patcher stabilization.
* **High-Throughput Grunt Work (GLM, Big Pickle, Nemotron):** Regex parsing, translation mapping generation (`translation_mapping.json`), batch extraction, and QA rule enforcement.
* **Strategic Audits & High-Level Sanity Checks (Claude):** Edge-case logic review, architectural sanity checking, and disaster-recovery analysis.

---

## 3. Core Technical Breakthroughs

### The "Heart Transplant" Concept
The PlayStation *Dragon Quest* releases share HeartBeat’s proprietary platform (`HBD1PS1D` archives, Huffman compression, and uniform dialog control codes). By studying the US release of *Dragon Quest VII* (`HBD1PS1D.W71`), we treated the engine mechanics as an English-aware target chassis, transplanting the raw narrative, events, and assets of *DQ4* into a fully working execution environment.

### Font 1 Atlas Anchor (`0xD60`) & Sub-Block Mining
* Anchored the raw **Font 1 tile atlas** at offset `0xD60`.
* Decoded the proprietary character-spacing and tile-referencing system.
* Mapped surrounding variable-width sub-blocks to prevent English text expansions from colliding with adjacent runtime VRAM buffers.

### MIPS Pointer Remapping (Field Menu Hooks)
Hardcoded Japanese character assumptions broke English menu rendering. The pipeline hooks and remaps execution sectors directly on the disc binary:
* **Sector 3191:** Re-anchored `SPL` (Spells), `ITM` (Items), `STS` (Status), and `TLK` (Talk) draw routines.
* **Sector 10652:** Realignment of sub-window boundaries and inventory cursor positions for longer terminology.
* Resolved split-text pairs and dynamic pointer mismatches (`1202/1202` referrers resolved with 0 false positives).

### Clean-Room Sovereign Pipeline vs. Legacy Wrappers
To avoid broken dependencies and legacy pipeline lock-in, all shims were replaced with an in-house toolchain (`hbe/sovereign_refs.py`):
* **Containment Verification:** Mathematical audits verify an overlap coefficient of just **0.003 (0.3%)** against legacy research scripts (`hbd.py`), establishing a verified clean-room codebase.
* **Native Mode 2 Form 1 Rebuilder:** Direct sector rebuilding with automated EDC/ECC recalibration:
  ```text
  Font 1 (0xD60): INJECTION -- OUR PIPELINE (hbe, no psx_tools)
  ...
  EDC/ECC: PASS
  disc exit 0 in 163.4s
