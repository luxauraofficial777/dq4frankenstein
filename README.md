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
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a88d8bc4-21a3-472c-826e-fc242947fc9a" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2eb2e85a-1405-42e3-887d-f64e55b6de74" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/825e9342-4aa0-4261-9404-7bde62591931" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e4852791-0b1d-4494-9299-46bdfb671e74" />



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
  │  Sovereign Native RC2   │                   │ "Zenithian Forge" Engine│
  ├─────────────────────────┤                   ├─────────────────────────┤
  │ • Pristine SLPM_869.16  │                   │ • DQ3 SFC ExHiROM Host  │
  │ • 7-Stage Native Chain  │                   │ • Mode 3 Title & Sprites│
  │ • Class A Overlay Remap │                   │ • 15 Campaign Maps ($E0)│
  │ • Class B LZS Duplicates│                   │ • Custom Huffman Tree   │
  │ • Dual-Font UI Engine   │                   │ • LOCN Collision Decode │
  │ • Mode 2 Form 1 EDC/ECC │                   │ • Native ASM Expansion  │
  └─────────────────────────┘                   └─────────────────────────┘

## 🔨 Build Instructions

### Building the PSX Sovereign Master Disc (7-Stage Native Pipeline):
```powershell
# Execute the unified, self-contained native build pipeline
# Executes: Step 0 (Validation) -> Step 1 (Dialogue) -> Step 1c (Type-39 Scripts) -> 
#           Step 2 (Font 1 UI)   -> Step 3 (Save Menu) -> Step 4 (Battle 0x048B) -> 
#           Step 4b (Overlay Refs) -> Step 4c (LZS Duplicates) -> Step 5 (EDC/ECC)
python tools/native_build.py --corpus translation/full_translation_boot_with_0069.json

# (Optional) Verify RAM buffer alignment and overlay referrers
python translation-tools/patch_overlay_refs.py --report

# (Optional) Direct DuckStation Diagnostic Run (for MIPS/RAM inspection)
& "duckstation/duckstation-qt-x64-ReleaseLTCG.exe" "build/dq4_sovereign_master.cue"
  YouTube:   https://www.youtube.com/LuxAuraOfficial

  (cc) Lux Aura. For educational and preservation purposes.
===============================================================

