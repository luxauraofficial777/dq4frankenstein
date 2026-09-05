# dq4frankenstein Progress ~99% (Update to V1.00 In Progress)
<p align="center">
  <img width="1024" height="1024" alt="dq4frank" src="https://github.com/user-attachments/assets/3ac9afc3-0cec-4a48-aab4-8b5e086aed73" />
</p>

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

This project delivers a **100% native in-place re-authoring engine** targeting the authentic Japanese HeartBeat binary (`SLPM_869.16` / `HBD1PS1D.Q41`)[cite: 2, 3], alongside a parallel total-conversion research track (**Zenithian Forge**) backporting the campaign into the Super Famicom *Dragon Quest III* 6 MB ExHiROM engine.

---

## 🏛️ Project Pedigree & Historical Lineage

This suite represents the synthesis of hundreds of hours of low-level MIPS decompilation, CD-ROM container research, and community reverse-engineering milestones:
* **Core Architecture & Toolchain:** Conceived, engineered, and finalized by **Lux Aura**.
* **Foundation Text Extraction & Java Engine:** Built upon seminal structural research and early patcher tooling by **Markus Schroeder** ([Markus Projects](http://markus-projects.net/dragon-hackst-iv/))[cite: 4, 14] and early extraction tooling by **Mandy Wilkens** ([dq4psxtrans](https://github.com/mwilkens/dq4psxtrans)).
* **Low-Level Codec & Hardware Inspection:** Incorporates forensic insights, asset inspection, and round-trip verification tooling adapted from the [DQIV_PSX_TOOLS](https://github.com/RadMageIRL/DQIV_PSX_TOOLS/tree/dev) project by **RadMage**[cite: 6, 14].
* <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2ae20665-16ac-4f92-95f3-bda8b451ddf8" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/594c17d6-6b24-4107-b54b-4b1cf9ab82d5" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2d9dd3da-7209-40d4-9385-cf6e24177e51" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b605bb91-13a0-4a4c-97c6-ce67dca63306" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/eed68c3c-35b5-46c9-b719-75fdc652f311" />
<img width="802" height="687" alt="image" src="https://github.com/user-attachments/assets/2a00621e-2a0e-4681-bb69-82d898b9dc83" />
<img width="802" height="687" alt="image" src="https://github.com/user-attachments/assets/caed6469-e4f4-445d-a070-5cecc5379762" />
<img width="802" height="687" alt="image" src="https://github.com/user-attachments/assets/7447c3b9-590d-4948-bd9d-b59be2dca651" />
<img width="802" height="687" alt="image" src="https://github.com/user-attachments/assets/a579e1a3-1eca-4f3a-8456-b40bb9e436e4" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/31467a3d-9108-482d-9dda-f50c228c38f4" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f39d29e4-e91d-4d04-b04e-4627f0248fb0" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/650c8dcb-cbac-4174-afef-12b6f3caa3d5" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2eb2e85a-1405-42e3-887d-f64e55b6de74" />






## 📐 Unified System Architecture

```text
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
  │ • Java/MIPS Referrer Map│                   │ • 15 Campaign Maps ($E0)│
  │ • Dual-Font Architecture│                   │ • Custom Huffman Tree   │
  │ • Buffer-Safe Pagination│                   │ • LOCN Collision Decode │
  └─────────────────────────┘                   └─────────────────────────┘

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

