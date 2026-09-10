<p align="center">
  <img width="800" alt="Dragon Quest IV: The Zenithian Chronicles" src="https://github.com/user-attachments/assets/3ac9afc3-0cec-4a48-aab4-8b5e086aed73" />
</p>

# Dragon Quest IV: The Zenithian Chronicles Current Build V.99 Rebuild B (Live/Playable)
### Multi-Generational Localization Suite & Dual-Platform Re-Authoring Engine
Deliverables: DQ4_Patcher_RebuildB_QuickStart.z01-z05 .zip dist_rebuild_b.z01-z05 .zip ShipB.zip Native Build Pipeline
**Created & Architected by Lux Aura**  
*Maintained under the VoidWalkers Research Project*  
🎮 **Steam:** [Void Walkers X64](https://store.steampowered.com/app/4979730/Void_Walkers_X64/)
[![Romhacking.net: #7711](https://img.shields.io/badge/Romhacking.net-Translation%20%237711-blue)](https://www.romhacking.net/translations/7711/)
[![YouTube: Devlook](https://img.shields.io/badge/YouTube-Devlook%20Video-red?logo=youtube&logoColor=white)](https://www.youtube.com/watch?v=UE0rsMtogGw)
<div align="center">
  <a href="https://www.youtube.com/watch?v=UE0rsMtogGw">
    <img src="https://img.youtube.com/vi/UE0rsMtogGw/maxresdefault.jpg" alt="DQ4 Frankenstein English Translation Devlook" width="750" style="border-radius: 8px;" />
  </a>
  <p><em>🎬 <b>Watch:</b> DQ4 Frankenstein English Translation Devlook (Rebuild B / Ship B)</em></p>
</div>

---

[![Platform: PSX](https://img.shields.io/badge/Platform-PlayStation%201%20(Native%20RC2%2FRC3)-003791?logo=playstation&logoColor=white)](#-track-a-playstation-1-native-sovereign-engine)
[![Platform: SNES](https://img.shields.io/badge/Platform-Super%20Famicom%20(ExHiROM)-E60012?logo=nintendo&logoColor=white)](#-track-b-super-famicom-zenithian-forge)
[![Architecture: Multi-Generational Study](https://img.shields.io/badge/Architecture-Multi--Generational%20Study-4CAF50)](#-unified-system-architecture)
[![Progress: ~99%](https://img.shields.io/badge/Progress-~99%25%20(V1.00%20Release%20Candidate)-orange)](#-build--release-pipeline)
[![License: CC-BY-NC-SA 4.0](https://img.shields.io/badge/License-CC--BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

---

## 🧭 Executive Summary

**Dragon Quest IV: The Zenithian Chronicles** is an industrial-grade reverse-engineering suite, asset bridge, and dual-platform localization toolchain for *Dragon Quest IV: Michibikareshi Mono Tachi* (ドラゴンクエストIV 導かれし者たち).

For over two decades, the 2001 PlayStation 1 remake of *Dragon Quest IV* remained the only mainline entry in the franchise without a complete, hardware-accurate English home console release. Previous historical attempts relied on "Frankenstein" cross-engine grafting (forcing the North American *Dragon Warrior VII* executable onto the DQ4 disc), leading to irreconcilable thread collisions, MIPS CPU stalls, and broken camera scripts.

This project delivers a **100% native in-place re-authoring engine** targeting the authentic Japanese HeartBeat binary (`SLPM_869.16` / `HBD1PS1D.Q41`) with **zero sector shift**, alongside a parallel total-conversion research track (**Zenithian Forge**) backporting the campaign into the Super Famicom *Dragon Quest III* 6 MB ExHiROM engine.

---

## 🏛️ Project Pedigree & Historical Lineage

This suite represents the synthesis of hundreds of hours of low-level MIPS decompilation, CD-ROM container research, and community reverse-engineering milestones:
* **Core Architecture & Sovereign Native Engine:** Conceived, engineered, and finalized by **Lux Aura**.
* **Foundation Text Extraction & Early Tooling:** Seminal structural research and early patcher tooling by **Markus Schroeder** ([Markus Projects](http://markus-projects.net/dragon-hackst-iv/)) and early extraction tooling by **Mandy Wilkens** ([dq4psxtrans](https://github.com/mwilkens/dq4psxtrans)).
* **SNES Authoring & ROM Mining:** Big Pickle & Omen Alpha (SNES data mining, DQ1+2+3 corpora, and RAM dump forensics).
* **AI Research & Pair Programming Team:** KIMI (Memory dumps & disassembly), GLM (MIPS dataflow analysis & split-immediates), Claude (Opus) (Pipeline architecture & Huffman packing), Gemini (Font 2 text mining & Class A/B remappers), Nemotron (Monster DB & battle overlay layers). Omen Alpha DQ4 SNES content authoring. Debugging RAM dump mining Omen Alpha & Big Pickle.

---

## 📸 Development & In-Game Showcase

<div align="center">
  <table>
    <tr>
      <td width="50%"><img src="https://github.com/user-attachments/assets/2ae20665-16ac-4f92-95f3-bda8b451ddf8" alt="Dialogue & Font 2 Rendering" /></td>
      <td width="50%"><img src="https://github.com/user-attachments/assets/594c17d6-6b24-4107-b54b-4b1cf9ab82d5" alt="Title Crawl & Prologue" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/2d9dd3da-7209-40d4-9385-cf6e24177e51" alt="Church Save System" /></td>
      <td><img src="https://github.com/user-attachments/assets/b605bb91-13a0-4a4c-97c6-ce67dca63306" alt="Font 1 Fixed Menu" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/eed68c3c-35b5-46c9-b719-75fdc652f311" alt="Battle Overlay & Bestiary" /></td>
      <td><img src="https://github.com/user-attachments/assets/2a00621e-2a0e-4681-bb69-82d898b9dc83" alt="Memory Card Management" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/caed6469-e4f4-445d-a070-5cecc5379762" alt="Party Chat Verification" /></td>
      <td><img src="https://github.com/user-attachments/assets/7447c3b9-590d-4948-bd9d-b59be2dca651" alt="Town NPC Interactions" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/a579e1a3-1eca-4f3a-8456-b40bb9e436e4" alt="Item & Inventory Windows" /></td>
      <td><img src="https://github.com/user-attachments/assets/31467a3d-9108-482d-9dda-f50c228c38f4" alt="Tactics & Status Screens" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/f39d29e4-e91d-4d04-b04e-4627f0248fb0" alt="Map Exploration" /></td>
      <td><img src="https://github.com/user-attachments/assets/650c8dcb-cbac-4174-afef-12b6f3caa3d5" alt="Combat Dialogue" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/a88d8bc4-21a3-472c-826e-fc242947fc9a" alt="MIPS Disassembly Verification" /></td>
      <td><img src="https://github.com/user-attachments/assets/2eb2e85a-1405-42e3-887d-f64e55b6de74" alt="RAM Mining Forensics" /></td>
    </tr>
    <tr>
      <td><img src="https://github.com/user-attachments/assets/825e9342-4aa0-4261-9404-7bde62591931" alt="Cutscene Script Inspection" /></td>
      <td><img src="https://github.com/user-attachments/assets/e4852791-0b1d-4494-9299-46bdfb671e74" alt="Dual-Platform Verification" /></td>
    </tr>
  </table>
</div>
<img width="662" height="667" alt="image" src="https://github.com/user-attachments/assets/bf30b7e2-d494-4b4c-9723-0f0ad0560f20" />

<img width="376" height="193" alt="image" src="https://github.com/user-attachments/assets/718c10f0-8031-4da8-869c-6119725cc55f" />


---

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
  │ • In-Place HBD Patcher  │                   │ • Mode 3 Title & Sprites│
  │ • Class A Overlay Remap │                   │ • 15 Campaign Maps ($E0)│
  │ • Class B LZS Duplicates│                   │ • Custom Huffman Tree   │
  │ • Dual-Font UI Engine   │                   │ • LOCN Collision Decode │
  │ • Mode 2 Form 1 EDC/ECC │                   │ • Native ASM Expansion  │
  └─────────────────────────┘                   └─────────────────────────┘

---

### File 2: `BUILD.md` (Separate Build & Track Guide)
*Contains the complete PlayStation 1 Native Sovereign Engine documentation, E2E build pipeline steps, Super Famicom Zenithian Forge track, and emulator compatibility table.*

```
# 🚀 Track A: PlayStation 1 Native Sovereign Engine

The PlayStation 1 release operates directly on the Japanese CD-ROM image in **Mode 2, Form 1** (2,352 bytes/sector, 2,048 bytes user data). Every modified dialogue block, menu, and overlay module is compressed to fit inside or below its pristine byte budget.

---

## 🔨 Build Instructions (E2E Pipeline)

### Prerequisites
* **Python 3.8+** with `numpy` installed (`pip install numpy`).
* Clean Japanese PS1 CD-ROM image:
  * **File:** `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin`
  * **Size:** `368,057,424` bytes
  * **CRC32:** `3D67C858`
  * **SHA-256:** `100D87DB9DEADF8F9FA4BB891D3A5D0BB112ACBF5ADBCBC93C637848ED9C7531`

 # DQ4 Frankenstein — Dragon Quest IV (PSX) English Localization
**Lux Aura & The VoidWalkers Research Project**

Two ways to play the English localization on your own Japanese disc:

| Path | Time | Needs |
|---|---|---|
| **VoidPatcher** (recommended) | ~2 min | Just the patcher exe — nothing to install |
| **BuildB pipeline** | ~80 min | Python 3.8+ (+numpy), edcre.exe |

> We distribute **no copyrighted content**. You supply your own pristine Japanese disc
> image; our tools transform **your** copy.

---

## Your input image

`Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` (PSX, SLPM_869.16)

| Hash | Value |
|---|---|
| SHA-1 | `85064625AFA12219880FC8D07047A3CC1C595CB9` |
| SHA-256 | `100D87DB9DEADF8F9FA4BB891D3A5D0BB112ACBF5ADBCBC93C637848ED9C7531` |

---

## Option A — VoidPatcher (fast path)

### Install
1. Download `VoidPatcher_RB1.exe` — a single self-contained executable.
2. Done. No runtime, no Python, no dependencies.

### Usage
1. Run `VoidPatcher_RB1.exe`.
2. **TARGET DISC IMAGE** → browse to (or type) your pristine image's path.
   It's validated on selection (size + SLPM_869.16 boot reference).
3. Leave **SEALED PAYLOAD (DEFAULT)** selected.
4. Press **[ >>> APPLY PATCH <<< ]** and watch the stage panel (~2 min).
5. Output appears next to the exe:
   - `dq4_zenithian_english_RB1.bin` + `.cue`
   - The final SHA-256 gate guarantees your output equals the verified master
     (`AC9F94A13A5627C30013FB0D13C88CD96F4BCEC878A2DA6E7B1978E1CD4E5703`).

     Exit 0 = every verification gate passed.

The patcher refuses non-pristine sources (already-patched masters, known-bad images)
with a specific message, and never writes to your source image.

---

## Option B — BuildB pipeline (build from source)

### Install
1. Clone / download this repo.
2. Python 3.8+ and `pip install numpy`.
3. Download `edcre.exe` (PSX EDC/ECC recalculator) and place it at `shipB\edcre\edcre.exe`
   — without it the build completes but EDC/ECC steps are skipped (output not hardware-safe).
4. Place your pristine image in the repo root (one level above `shipB\`).

### Usage
- ~80 minutes. Output: `shipB\build\dq4_shipB.bin` + `.cue` (SHA-256 printed at the end).
- Resume: `--from-step 4g` · Reuse dialogue: `--skip-dialogue` ·
  Custom corpus: `--corpus your_translation.json` ·
  Finish the seal: `--g11-dumps dump1.bin dump2.bin`
- The build runs hard verification gates (G2 dispatch integrity, G10 corpus census,
  G5 EDC/ECC) and the same-tree 048C rebuild with delta-lock assertions.

---

## Playing

1. DuckStation (retail PSX BIOS, e.g. SCPH-1001 — no custom BIOS needed).
2. File → Open Disc → the generated `.cue`.
3. Verify: title → prologue → Chapter 1; dialogue, battles, sound, saving.
4. Real hardware: burn the `.bin/.cue` to a CD-R at 4x–8x (modchipped/softmodded consoles).

## Reporting bugs

Include: in-game location + description; for freezes, the DuckStation log (timestamp,
failing LBA range, CD-ROM buffer status) plus paired RAM dumps; and your master's SHA-256.

## Credits & links

Full credits in [`shipB/CREDITS.md`](shipB/CREDITS.md).

**Follow Lux Aura** — the record label / publisher:
[Bandcamp](https://luxaura.bandcamp.com) ·
[Facebook](https://www.facebook.com/LuxAuraOfficial/) ·
[YouTube](https://www.youtube.com/LuxAuraOfficial) ·
[Steam (via SteamDB)](https://steamdb.info/publisher/Lux+Aura/)

## Legal

DQ4: Michibikareshi Mono Tachi © 1998 HeartBeat / Enix. All trademarks belong to their
respective owners. This project ships tools only — no game assets, no BIOS, no copyrighted
binary. It requires and transforms the user's own legally obtained disc image.

*ORDER. PRECISION. FIDELITY.*

Headless / scriptable:

### Quick Start
Place your pristine `.bin` file in the repository root or `ship/` folder, then run:

```bash
# Windows
build.bat

# Linux / macOS
python ship/build.py

[STEP -1] Ship Parity Gate (verify_ship_parity.py)
└── Validates 156-file mirror integrity across root and ship/ distributions.

[STEP 0] Pre-Flight Corpus & Marker Validation (validate_corpus.py + g8_marker_parity.py)
└── Validates strict hex tokens {xxxx}, sequence parity, and zero apostrophe rule.
└── Asserts facility variable marker parity against the pristine skeleton.

[STEP 1] Native In-Place HBD Dialogue Injection (dq4_hbd_patcher.py)
└── Re-encodes 1,358 blocks using length-limited Huffman (ml=14..9).
└── Enforces numNodes > 0 fix with zero padding isolated after tree end.

[STEP 1c] Type-39 Cutscene Script LZSS Remapper (patch_type39_scripts.py)
└── Decompresses 612 scripts, remaps bytecode dialogue words, and re-compresses.
└── Eliminates cutscene freezes (e.g. Burland Castle throne room).

[STEP 2] Font 1 Menus, Items, Spells & Delta-Locks (patch_font1_ourway.py)
└── Injects 8x14 half-width font into SLPM_869.16 Block 0x048C.
└── Locks cursor blink / advance bit deltas (44, 36, 50, 66) across SIDs 773–778.

[STEP 3] Church Save Menu & Memory Card (patch_save_menu.py)
└── Injects English text for Block 0x0474 (save confirmations and memory card management).

[STEP 4] Battle Overlay Primary Block & Monster DB (patch_battle_overlay.py)
└── Injects 817 sequences into Block 0x048B (combat actions, messages, and bestiary).

[STEP 4b] Class A Stale Referrer Remapping (patch_overlay_refs.py)
└── 32-instruction dataflow scanner resolving shifted pointers across EXE & raw sectors.

[STEP 4c] Class B Type-46 Duplicate Modules (patch_overlay_duplicates.py)
└── Decompresses and updates all 67 distinct Type-46 overlay modules across 267 sites.

[STEP 4e / 4d / 4h] Structural & Table Alignments
└── 4e: Same-tree delta-lock confirmation.
└── 4d: Stale word-table fixed-point sweep.
└── 4h: Remaps LBA 40217 bare page table rows, eliminating the "Monster Gramps" bug.

[STEP 1e / 4f / 4g / 4g-2] Late Structural Patches
└── 1e: Injects Block 0x048F priest save text directly into the EXE.
└── 4f: Remaps D4 split-immediates and cutscene call sites.
└── 4g-2: Injects D2 (church offering) and D3 (victory window) string-start identities.

[STEP 4h-2 / 4h-3 / 4h-4] Runtime Execution Guards
└── 4h-2: Bypasses VSync timeout panic (0x80099F60 -> exit(3)), preventing stream stalls.
└── 4h-3: Redirects overlay residency collisions during dialogue staging.
└── 4h-4: Injects 17-instruction KSEG0 allocator guard protecting message module memory.

[STEP 5] Disc Finalization & EDC/ECC Recalculation (edcre.exe)
└── Generates sovereign master CUE sheet.
└── Recalculates Mode 2 Form 1 EDC checksums and ECC P/Q parity across all modified sectors.

[STEP 6 & 7] Post-Build Gates & Release Sealing
└── Runs G2 dispatch gate, G10 corpus census, and G5 full-disc parity sweeps.
└── Asserts G11 post-battle RAM residency before labeling a tagged release.
