# dq4frankenstein Progress ~99%
<img width="1024" height="1024" alt="dq4frank" src="https://github.com/user-attachments/assets/3ac9afc3-0cec-4a48-aab4-8b5e086aed73" />
# Dragon Quest IV: The Zenithian Chronicles
### Multi-Generational Localization Suite & Dual-Platform Re-Authoring Engine
**Created & Architected by Lux Aura**  
*Maintained under the VoidWalkers Research Project*

[![Platform: PSX](https://img.shields.io/badge/Platform-PlayStation%201%20(Native%20RC2)-003791?logo=playstation&logoColor=white)](#-track-a-playstation-1-native-rc2--primary-track)
[![Platform: SNES](https://img.shields.io/badge/Platform-Super%20Famicom%20(ExHiROM)-E60012?logo=nintendo&logoColor=white)](#-track-b-super-nintendo--sfc-zenithian-forge)
[![Architecture: Multi-Generational Study](https://img.shields.io/badge/Architecture-Multi--Generational%20Study-4CAF50)](#-the-multi-generational-data-bridge)
[![License: CC-BY-NC-SA 4.0](https://img.shields.io/badge/License-CC--BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

---

## 🧭 Executive Summary

**Dragon Quest IV: The Zenithian Chronicles** is an industrial-grade reverse-engineering suite, asset bridge, and dual-platform localization toolchain for *Dragon Quest IV: Michibikareshi Mono Tachi* (ドラゴンクエストIV 導かれし者たち). 

For over two decades, the 2001 PlayStation 1 remake of *Dragon Quest IV* remained the only mainline entry in the franchise without a complete, hardware-accurate English home console release. Previous historical attempts relied on "Frankenstein" cross-engine grafting (forcing the North American *Dragon Warrior VII* executable onto the DQ4 disc), leading to irreconcilable thread collisions, MIPS CPU stalls, and broken camera scripts.

This project delivers a **100% native in-place re-authoring engine** targeting the authentic Japanese HeartBeat binary (`SLPM_869.16` / `HBD1PS1D.Q41`), alongside a parallel total-conversion research track (**Zenithian Forge**) backporting the campaign into the Super Famicom *Dragon Quest III* 6 MB ExHiROM engine.

---

## 🏛️ Project Pedigree & Historical Lineage

This suite represents the synthesis of hundreds of hours of low-level MIPS decompilation, CD-ROM container research, and community reverse-engineering milestones:
* **Core Architecture & Toolchain:** Conceived, engineered, and finalized by **Lux Aura**.
* **Foundation Text Extraction & Java Engine:** Built upon seminal structural research and early patcher tooling by **Markus Schroeder** ([Markus Projects](http://markus-projects.net/dragon-hackst-iv/)) and early extraction tooling by **Mandy Wilkens** ([dq4psxtrans](https://github.com/mwilkens/dq4psxtrans)).
* **Low-Level Codec & Hardware Inspection:** Incorporates forensic insights, asset inspection, and round-trip verification tooling adapted from the [DQIV_PSX_TOOLS](https://github.com/RadMageIRL/DQIV_PSX_TOOLS/tree/dev) project by **RadMage**.

---

## 📐 Unified System Architecture
        ┌────────────────────────┐
                      │   Multi-Generational   │
                      │     Corpus Bridge      │
                      └───────────┬────────────┘
                                  │
           ┌──────────────────────┴──────────────────────┐
           ▼                                             ▼
┌─────────────────────────┐                   ┌─────────────────────────┐│   PlayStation 1 (PSX)   │                   │  Super Nintendo (SNES)  ││    Native RC2 Engine    │                   │ "Zenithian Forge" Engine│├─────────────────────────┤                   ├─────────────────────────┤│ • Pristine SLPM_869.16  │                   │ • DQ3 SFC ExHiROM Host  ││ • HBD1PS1D.Q41 Rebuild  │                   │ • Mode 3 Title & Sprites││ • Java/MIPS Referrer Map│                   │ • 15 Campaign Maps ($E0)││ • Dual-Font Architecture│                   │ • Custom Huffman Tree   ││ • Buffer-Safe Pagination│                   │ • LOCN Collision Decode │└─────────────────────────┘                   └─────────────────────────┘
---

## 🎮 Track A: PlayStation 1 (Native RC2 — Primary Track)

The PlayStation 1 release re-authors the native retail Japanese disc in-place. By abandoning executable grafting, the native engine preserves 100% of HeartBeat's original 3D camera sweeps, battle animations, and sound sequencing.

### Core Breakthroughs & Subsystems:
1. **Dual-Font In-Engine Architecture:**
   * **Font 1 (Menus & Battle UI):** Registered in RAM at `0x800B2A3C`. Driven by a resident 137-bucket hash table referencing an $8 \times 14$ 4bpp texture atlas. Injected via TextBlock `0x048C` and resolved through the engine's multi-group command table at `0x8001E8C8` (field actions, battle dispatches, Yes/No prompts).
   * **Font 2 (Narrative & Dialogue):** Registered at `0x800B3600`. Variable-width 2bpp run-length encoded glyph renderer covering all 1,103 campaign sub-blocks (`0x0022`–`0x0487`).
2. **Buffer-Safe Dynamic Pagination:**
   * Natural English text exceeds Japanese kanji byte density and crashes the fixed WRAM decompressor scratchpad ($\sim 64$ bytes per box at `0x8002A5E8`).
   * Our native paginator automatically partitions English dialogue into strict **2-line visual boxes** ($\le 18$ characters on Line 1 to accommodate native `＊「` leading quotes; $\le 20$ characters on Line 2), injecting `<7F04>` line advance and `<7F0B>` VRAM-clear tokens without destructive vowel stripping.
3. **Comprehensive Type-39 Script Referrer Remapping:**
   * Unlike standard hex patches that only rewrite local block headers, our toolchain cross-indexes and remaps external Type-39 cutscene SCRIPT triggers and global index tables so event scripts reference relocated bit offsets with zero execution faults.
4. **Mastering & CD-ROM Parity:**
   * Direct generation of ISO 9660 Mode 2 Form 1 raw CD sectors (2,048 user data / 2,352 raw bytes) with sector-by-sector Reed-Solomon EDC/ECC recalculation via `edcre`.

---

## 🕹️ Track B: Super Nintendo / SFC ("Zenithian Forge")

The secondary research track is a ground-up total conversion porting the campaign into the **Super Famicom *Dragon Quest III* engine** running on a 6 MB ExHiROM architecture.

* **65816 Engine Orchestration:** Custom cold-boot hooks at `$C3:F30A` and coroutine schedulers bypass DQ3 character creation, initializing directly into Chapter 1 (Burland).
* **Mode 3 Graphics Pipeline:** 8bpp title graphics in Banks `$5E`/`$5F`, complete custom DQ4 monster and party sprite sheets in Bank `$D0`.
* **Map & Collision Port:** 15 Zenithian campaign maps ported to Bank `$E0` with HeartBeat LZSS compression and DQ3 metatile index translation.
* **Expanded Script Bank:** Dedicated Huffman dialogue pipeline targeting Bank `$FC:C258` referencing a dynamic 8x8 proportional font.

---

## 🌉 The Multi-Generational Data Bridge

The translation does not synthesize text in a vacuum. It synchronizes text architecture across four console generations:

| Generation | Platform & Asset Lineage | Role in Project |
|---|---|---|
| **1990** | **NES** (*Dragon Warrior IV*, SUROM) | Original 8-bit nomenclature baseline, spell names, and equipment indices. |
| **1996** | **SFC** (*Dragon Quest III*, ExHiROM) | Host assembly framework, tile engine, and audio sequencer for Track B. |
| **2001** | **PSX** (*Dragon Quest IV*, HeartBeat HBD) | Native binary, event scripts, 3D world parameters, and primary release target. |
| **2007** | **NDS** (*DQIV: Chapters of the Chosen*) | Modern reference corpus cross-checked for localization continuity. |

### Dynamic Token Engine:
Engine-level control memory variables are fully preserved during compilation:
* `{7F1F}`: Hero / Protagonist Runtime Name
* `{7F24}`: Dynamic Item Reference
* `{7F2D}`: Dynamic Town / Location Reference
* `{7eXX}`: Native System Phrase Dictionary Tokens (Blocks `0x0021`–`0x0025`)

---

## 📁 Repository Structure

DQLOSTTRANSLATION/├── build/                 # Compiled binaries (.bin/.cue, .sfc)├── study/                 # Technical documentation, memory audits, and specs│   ├── ROADMAP_PLAYABILITY_FIRST_Sep3_2026.md│   └── DQHBE REFERENCE DOC/ # MIPS disassembly and HBD specifications├── translation/           # Master translation corpora and term dictionaries│   ├── full_translation_clean_master.final.json # Canonical narrative script│   ├── exe_block_048c_truncated.json             # Font 1 menu command strings│   └── csv/                                      # PSX string & referrer databases├── translation-tools/     # PSX build pipeline and binary patchers│   ├── classes/TextPatcher.class  # Java external referrer & script injector│   ├── dq4_hbd_patcher.py         # Huffman compressor & line paginator│   └── patch_font1_ourway.py      # Font 1 menu array injector├── snes/                  # SNES Zenithian Forge pipeline│   ├── tools/build_master_dq4.py  # ExHiROM ROM compiler│   └── data/                      # Map grids, sprite sheets, character banks├── tools/                 # Token normalizers and diagnostic utilities├── edcre/                 # Mode 2 Form 1 Reed-Solomon EDC/ECC engine└── dragon-hackst-4-src/   # Source tree for the core Java patcher
---

## ⚙️ Environment Setup & Prerequisites

* **Java Development Kit:** OpenJDK 25 (Required for `TextPatcher.class`).
* **Python:** Python 3.10+ (64-bit).
* **Parity Utility:** `edcre.exe` (included in `/edcre`).
* **Recommended Emulators:**
  * **PSX:** [DuckStation](https://www.duckstation.org/) (Strict timing, VRAM debugger, and MIPS memory inspector).
  * **SNES:** [Mesen 2](https://github.com/SourMesen/Mesen2) or [bsnes-plus](https://github.com/devinacker/bsnes-plus).

---

## 🔨 Build Instructions

### Building the PSX Native Disc:
```powershell
# 1. Paginate the master English dialogue corpus (enforces strict 18/20 line limits)
python tools/psx_string_paginator.py `
  --input translation/full_translation_clean_master.json `
  --output translation/full_translation_paginated.json

# 2. Inject dialogue and remap Type-39 cutscene referrers via Java TextPatcher
& "C:\Program Files\OpenJDK\jdk-25\bin\java.exe" `
  -cp "translation-tools/classes;dq4psx-patcher.jar" `
  TextPatcher --schema dq4 `
  "Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin" `
  translation/full_translation_paginated.json `
  build/dq4_paginated.bin

# 3. Recalculate Mode 2 Form 1 Reed-Solomon EDC/ECC (mandatory before execution)
.\edcre\edcre.exe build\dq4_paginated.bin

# 4. Boot in DuckStation
& "duckstation/duckstation-qt-x64-ReleaseLTCG.exe" "build/dq4_paginated.cue"
Building the SNES ExHiROM ROM:PowerShell# Compile the complete SNES DQ4-on-DQ3 master cartridge
python snes/tools/build_master_dq4.py

# Verify cart header, memory mapping, and dual checksums
python snes/audit/run_all.py
🎯 Production RoadmapPhaseFocus AreaCore MilestoneStatusPhase 0Toolchain HardeningRebuild Java patcher from source; establish byte-level baseline.✅ COMPLETEPhase 1Playability KeystoneStable English narrative: Prologue $\to$ Chapter 1 (Font 2).🔄 ACTIVEPhase 2Font 1 Menus & UICommand menus, combat, shop dialogues (0x8001E8C8 / 0x048C).⏳ QUEUEDPhase 3Battle Engine TextIn-battle notifications, status messages, monster attacks.⏳ PLANNEDPhase 4World NPCs & DictionariesNon-dialogue HBD blocks and {7eXX} phrase expansion.⏳ PLANNEDPhase 5Gold Master ReleaseEnd-to-end playthrough (Chapters 1–5), standalone XDelta3 patch.⏳ PLANNED🛡️ Ground Truth & Engineering PrinciplesEmpirical Validation First: Success is defined exclusively by correct, unclipped in-game renders, clean DuckStation RAM/VRAM inspection, and verified hardware execution—never by self-certifying build scripts.Non-Destructive Integrity: Core executable sectors (SLPM_869.16) remain byte-pristine during dialogue runs to guarantee that CPU control flow, interrupt handlers, and DMA timings remain unmodified.Legal Notice: This repository contains reverse-engineering tools, patches, and asset bridges. No copyrighted ISOs or ROMs are distributed. Users must supply an authentic, legally dumped Japanese retail disc image to compile patches.📬 Contact & Official ChannelsOfficial Music & Audio Production: Lux Aura on BandcampVideo Devlogs & Demonstrations: Lux Aura Official YouTubeCommunity Updates: Lux Aura on Facebook(cc) Lux Aura. Released under Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International.For digital preservation, research, and educational purposes.

CONTACT
---------------------------------------------------------------

  Bandcamp:  https://luxaura.bandcamp.com
  Facebook:  https://www.facebook.com/LuxAuraOfficial
  YouTube:   https://www.youtube.com/LuxAuraOfficial

  (cc) Lux Aura. For educational and preservation purposes.
===============================================================

