# dq4frankenstein Progress ~99%
<img width="1024" height="1024" alt="dq4frank" src="https://github.com/user-attachments/assets/3ac9afc3-0cec-4a48-aab4-8b5e086aed73" />
DQ4/DW7 Frankenstein Project V.9,V.95,V.96,V.97,V.98,V.99

Dragon Quest IV PSX: "Lost Translation"
Created by Lux Aura with the help of previous work by Markus Projects, Mandy Wilkens, and ChickenKnife. http://markus-projects.net/dragon-hackst-iv/ https://github.com/mwilkens/dq4psxtrans https://www.romhacking.net/hacks/4275/ https://luxaura.bandcamp.com https://www.facebook.com/LuxAuraOfficial https://www.youtube.com/LuxAuraOfficial RadMage

Created by **Lux Aura** with the help of previous work by Markus Projects, Mandy Wilkens, and RadMage.

- http://markus-projects.net/dragon-hackst-iv/
- https://github.com/mwilkens/dq4psxtrans
- https://www.romhacking.net/hacks/4275/
- https://luxaura.bandcamp.com
- https://www.youtube.com/LuxAuraOfficial

Dragon Quest IV: The Zenithian Chronicles (Multi-Generational Localization Suite)
Platform: PSX
 
Platform: SNES
 
Architecture: Multi-Generational Study
 
Status: Playability-First Sprint

A monolithic reverse-engineering suite, asset bridge, and dual-platform localization pipeline for Dragon Quest IV: Michibikareshi Mono Tachi (ドラゴンクエストIV 導かれし者たち).

This project studies and bridges the data architecture of Dragon Quest IV across four console generations—Famicom/NES (1990), Super Famicom (1996 DQ3 Engine), PlayStation 1 (2001 HeartBeat Engine), and Nintendo DS (2007 Remake)—to deliver hardware-accurate, complete English editions.

🧭 Project Architecture Overview

                          ┌────────────────────────┐
                          │   Multi-Generational   │
                          │     Corpus Bridge      │
                          └───────────┬────────────┘
                                      │
               ┌──────────────────────┴──────────────────────┐
               ▼                                             ▼
  ┌─────────────────────────┐                   ┌─────────────────────────┐
  │   PlayStation 1 (PSX)   │                   │  Super Nintendo (SNES)  │
  │    Native RC2 Engine    │                   │ "Zenithian Forge" Engine│
  ├─────────────────────────┤                   ├─────────────────────────┤
  │ • Pristine SLPM_869.16  │                   │ • DQ3 SFC ExHiROM Host  │
  │ • HBD1PS1D.Q41 Rebuild  │                   │ • Mode 3 Title & Sprites│
  │ • Java TextPatcher      │                   │ • 15 Campaign Maps ($E0)│
  │ • Dual-Font Architecture│                   │ • Custom Huffman Tree   │
  │ • 64-byte WRAM Safe Box │                   │ • LOCN Collision Decode │
  └─────────────────────────┘                   └─────────────────────────┘
🎮 Track A: PlayStation 1 (Native RC2 — Primary Track)
The PlayStation 1 target (SLPM_869.16 / HBD1PS1D.Q41) focuses on native in-place re-authoring of the original Japanese retail disc.

Why Native Over Frankenstein?
Previous attempts to graft the North American Dragon Warrior VII executable onto the DQ4 disc (the "DW7 Frankenstein" approach) introduced irreconcilable thread collisions, MIPS sign-extension bugs, and CP stalls. The project has permanently retired the Frankenstein graft in favor of the Native RC2 pipeline.

Core PSX Subsystems:
Dual-Font System:
Font 1 (Menus & Battle UI): Registered at 0x800B2A3C. Fixed 
8
×
14
8×14 cell, resident 4bpp texture atlas. Injects into TextBlock 0x048C for field commands (TALK, ITEM, SPELL, TACTICS, etc.) and combat commands.
Font 2 (Dialogue & Narrative): Registered at 0x800B3600. Variable-width font rendering 2bpp RLE glyphs across 1,103 narrative sub-blocks (0x0022–0x0487).
Buffer-Safe Dialogue Pagination:
Natural English text exceeds the Japanese byte budget and crashes the engine's fixed WRAM decompression scratchpad (
∼
64
∼64 bytes per box at 0x8002A5E8).
The pipeline pre-paginates all dialogue into discrete 2-line 
×
≤
24
×≤24 visible character visual boxes, inserting <7F04> line advance and <7F0B> page-clear prompts to ensure 100% buffer compliance without lossy vowel stripping.
Type-39 Cutscene Referrer Remap:
When text blocks are re-encoded, string bit-offsets move. Unlike simple in-block pointer patchers, the Java TextPatcher remaps all external Type-39 cutscene SCRIPT triggers and global index tables.
Mastering & Checksums:
Generates ISO 9660 Mode 2 Form 1 CD-ROM sectors (2,048 data / 2,352 raw bytes) and recalculates Reed-Solomon EDC/ECC across all modified sectors using edcre.
🕹️ Track B: Super Nintendo / SFC ("Zenithian Forge")
The Super Nintendo target is a total conversion that backports Dragon Quest IV into the Dragon Quest III (SFC) engine, utilizing a 6 MB ExHiROM architecture.

Core SNES Subsystems:
65816 Host Engine: Deploys a custom coroutine scheduler and cold-boot hook ($C3:F30A) to bypass personality tests and boot directly into Zenithian chapter orchestration.
Graphic & Map Injections:
Mode 3 8bpp title graphics in Banks $5E and $5F.
DQ4 party sprites and monster graphics in Bank $D0.
15 campaign map tilemaps and collision tables mapped into Bank $E0.
Dialogue & Token Expansion:
Huffman-encoded script bank in $FC:C258 referencing a dynamic 8x8 VWF font.
Ongoing development focuses on resolving the stock DQ3 11-bit/char Huffman limit to house the complete English script.
🌉 The Multi-Generational Data Bridge
Rather than writing a standalone translation from scratch, the toolchain uses a synchronized content bridge cross-referencing multiple historical corpora:

Source Generation	Asset / Lineage	Role in Project
NES (1990)	Dragon Warrior IV (SUROM)	Original 8-bit script baseline, monster naming tables, and spell indices.
SFC (1996)	Dragon Quest III (ExHiROM)	Host assembly framework, tile engine, and audio sequencer for Track B.
PSX (2001)	Dragon Quest IV (HeartBeat HBD)	Target native binary, event scripts, camera coordinates, and 3D scenes.
NDS (2007)	Dragon Quest IV: Chapters of the Chosen	High-quality localized dialogue script (mpt0_dump.py 
→
→ string_library.json).
Dynamic Variable Preservation:
The translation normalizer strictly preserves live engine memory tokens across all platforms:

{7F1F}: Hero / Protagonist Name
{7F24}: Item Dynamic Reference
{7F2D}: Town / Location Dynamic Reference
{7eXX}: Native PSX Block Phrase Dictionary Tokens
📁 Repository Structure

DQLOSTTRANSLATION/
├── build/                 # Compiled disc images (.bin/.cue) and ROMs (.sfc)
├── study/                 # Forensic documentation, blueprints, and regression audits
│   ├── ROADMAP_PLAYABILITY_FIRST_Sep3_2026.md  # Active North Star
│   ├── SANITY_CHECK_HONEST_PIPELINE_AUDIT_Sep2_2026.md
│   └── DQHBE REFERENCE DOC/ # HeartBeat container and MIPS reverse-engineering specs
├── translation/           # Master corpora, JSON dictionaries, and term maps
│   ├── full_translation_clean_master.json # Canonical narrative corpus
│   └── csv/               # Markus Schroeder's PSX string extraction & referrer database
├── translation-tools/     # Native PSX build toolchain
│   ├── classes/TextPatcher.class  # Java external-referrer & text injection engine
│   ├── dq4_hbd_patcher.py         # Sub-block scanner, compressor & paginator
│   └── patch_font1_ourway.py      # Font 1 menu array injector
├── snes/                  # SNES Zenithian Forge pipeline
│   ├── tools/build_master_dq4.py  # Master 6 MB ExHiROM ROM compiler
│   └── data/                      # Map grids, sprite tiles, and character tables
├── tools/                 # Diagnostics, string paginators, and token normalizers
├── edcre/                 # Mode 2 Form 1 Reed-Solomon EDC/ECC recalculation utility
└── dragon-hackst-4-src/   # Source code for the core Java patcher
⚙️ Prerequisites & Environment Setup
Java Development Kit: OpenJDK 25 (Required for TextPatcher.class / compiled class version 69).
Python: Python 3.10+ (64-bit).
Sector Checksum Utility: edcre.exe (included in /edcre).
Recommended Emulators:
PSX: DuckStation (Strict hardware timings and VRAM inspection).
SNES: Mesen 2 or bsnes-plus (Cycle-accurate PPU and event debugging).
🔨 Build Instructions
Building the PSX Native English Disc:
powershell

# 1. Paginate the master English dialogue corpus (enforcing 64-byte WRAM box limits)
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
# 3. Recalculate Mode 2 Form 1 Reed-Solomon EDC/ECC (mandatory before boot)
.\edcre\edcre.exe build\dq4_paginated.bin
# 4. Boot in DuckStation
& "duckstation/duckstation-qt-x64-ReleaseLTCG.exe" "build/dq4_paginated.cue"
Building the SNES ExHiROM ROM:
powershell

# Compile the complete SNES DQ4-on-DQ3 master ROM
python snes/tools/build_master_dq4.py
# Verify cart header and dual-ExHiROM checksums
python snes/audit/run_all.py
🎯 Project Status & Roadmap
The project follows the Playability-First Discipline: gameplay stability from the Prologue through Chapter 1 must be proven on real hardware/DuckStation before secondary features are integrated.

Phase	Milestone	Focus Area	Status
Phase 0	Toolchain Hardening	Rebuild Java patcher from source; lock single source of truth.	✅ DONE
Phase 1	Playability Keystone	Stable English narrative from Prologue 
→
→ Chapter 1 (Font 2).	🔄 ACTIVE
Phase 2	Font 1 Menus & UI	Field & battle commands (0x800B2A3C / Block 0x048C).	⏳ QUEUED
Phase 3	Battle Dialogue	In-battle text, status notifications, and spell actions.	⏳ PLANNED
Phase 4	World & NPC Dialogue	Non-dialogue HBD strings, dictionary expansion ({7eXX}).	⏳ PLANNED
Phase 5	Gold Master Release	E2E playthrough (Chapters 1–5), XDelta3 patch creation.	⏳ PLANNED
🛡️ Ground Truth & Engineering Standards
Zero-Trust Validation: Success is defined solely by unclipped dialogue renders, DuckStation RAM/VRAM dumps, and stable gameplay—never by automated markdown reports.
Non-Destructive Patching: All original executable sections (SLPM_869.16) remain byte-pristine during dialogue testing to ensure MIPS CPU execution flow is never corrupted by misaligned pointer offsets.
Legal Notice: This repository contains reverse-engineering tools, patches, and documentation. It does not distribute copyrighted ISOs or ROMs. Users must supply their own legitimate copies of the retail software.
Maintained as part of the VoidWalkers Research Project.


CONTACT
---------------------------------------------------------------

  Bandcamp:  https://luxaura.bandcamp.com
  Facebook:  https://www.facebook.com/LuxAuraOfficial
  YouTube:   https://www.youtube.com/LuxAuraOfficial

  (cc) Lux Aura. For educational and preservation purposes.
===============================================================

