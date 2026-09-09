# dq4frankenstein Progress ~99% (Update to V1.00 In Progress)
<p align="center">
  <img width="1024" height="1024" alt="dq4frank" src="https://github.com/user-attachments/assets/3ac9afc3-0cec-4a48-aab4-8b5e086aed73" />
</p>

# Dragon Quest IV: The Zenithian Chronicles
### Multi-Generational Localization Suite & Dual-Platform Re-Authoring Engine
**Created & Architected by Lux Aura**  
*Maintained under the VoidWalkers Research Project*
https://store.steampowered.com/app/4979730/Void_Walkers_X64/

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
<img width="662" height="667" alt="image" src="https://github.com/user-attachments/assets/c401be6c-188d-4912-a381-03d142379dfa" />





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

Phase A: Pre-Flight Integrity & Dialogue Injection
STEP -1: Ship Parity Gate
Script: study/verify_ship_parity.py
Function: Verifies byte-level parity between the workspace root and the ship/ distribution tree across all 156 pipeline files (translation-tools/**, tools/native_build.py, canonical JSON databases).
Behavior: Fails fast with exit code 1 if a drift or missing file is detected.
STEP 0: Pre-Flight Corpus & Marker Validation
Scripts: 
ship/translation-tools/validate_corpus.py
 & study/g8_marker_parity.py
Function:
Validates every {xxxx} control code in 
full_translation_boot_with_0069.json
 (enforcing 4-digit hex, sequence index parity, and the Zero Apostrophe Rule to eliminate bracket corruption).
Runs the G8 Gate to ensure the variable marker sequence across town facility blocks matches the pristine skeleton, preventing dialogue renderer desyncs.
STEP 1: Native In-Place HBD Dialogue Injection
Script: 
ship/translation-tools/dq4_hbd_patcher.py
Input: Clean Japanese binary 
→
→ generates intermediate 
build/dq4_dialogue_en.bin
.
Mechanism:
Patches all linear text blocks in HBD1PS1D.Q41 starting at LBA 362.
Employs length-limited Huffman encoding (
m
l
=
14..9
ml=14..9) using hbe.huffman.
The numNodes = 0 Fix: Strictly writes numNodes > 0 at real_text_end + 8 with alignment padding placed after the tree, preventing decompression crashes.
Block 006C Handling: In-place patching eliminates the 4-byte overflow present in old tooling, permanently preventing the opening cutscene lockup.
Yield: 1,358 blocks patched in place with 0 sector drift.
(STEP 1b: External Referrer Remap — DISABLED)
Reason: The original raw HBD scan lacked container discrimination and clobbered non-dialogue file bodies and compressed scripts (producing false positives). Its functionality was replaced and superseded by STEP 1c.
STEP 1c: Type-39 Cutscene Script LZSS Remapping
Script: 
ship/translation-tools/patch_type39_scripts.py
Function: Cutscene VM scripts reference dialogue via packed words (BlockID << 20) | (BitOffset + HTS * 8). When English text changes bit positions, unremapped scripts freeze.
Mechanism: Decompresses all Type-39 scripts via lzss_uncompress(), validates VM bytecode context (c0 21 a0), remaps shifted bit addresses, and recompresses with native dqlzs_compress(), clamping with zeros. Resolves freezes such as Burland Castle throne room and prologue sequence cutscenes.
Phase B: UI, Menus, & Primary Overlays
STEP 2: Font 1 UI Menus, Items, Spells & Delta-Locks
Script: 
ship/translation-tools/patch_font1_ourway.py
Target: Executable SLPM_869.16, Block 0x048C at file offset 0x097AC8 (VA 0x800AF1C8). Creates master image 
build/dq4_sovereign_master.bin
.
Font Constraints: Fixed 8×14 half-width font with 3-character command truncation (TLK, SPL, ITM, SRC, STS, TCT).
Delta-Lock Preservation: The game assembly indexes menu cursors and window steps by hardcoded bit deltas between SIDs 773–778: 
Δ
(
774
−
773
)
=
44
(
Cursor Blink
)
Δ(774−773)=44(Cursor Blink) 
Δ
(
776
−
775
)
=
36
(
Command Advance
)
Δ(776−775)=36(Command Advance) 
Δ
(
777
−
775
)
=
50
(
Sub-Command Advance
)
Δ(777−775)=50(Sub-Command Advance) 
Δ
(
778
−
775
)
=
66
(
Status Window Advance
)
Δ(778−775)=66(Status Window Advance) patch_font1_ourway.py enforces these exact deltas during tree synthesis.
Remapping: Rewrites 1,202 direct words and 57 split-immediates in the executable.
STEP 3: Church Save Menu & Memory Card
Script: 
ship/translation-tools/patch_save_menu.py
Target: Block 0x0474 at Sector 105972.
Function: Injects English strings for adventure log selection, save confirmation, and memory card formatting with halfwidth encoding enabled.
STEP 4: Battle Overlay Primary Block & Monster DB
Script: 
ship/translation-tools/patch_battle_overlay.py
Target: Primary Block 0x048B at Sector 105540 (+2028 bytes).
Function: Injects 817 sequences (combat actions, battle messages, and the 237-monster bestiary). Remaps 1,625 direct words and 222 split-immediates across sectors 104000–106000.
Phase C: Deep Referrer Resolution & Overlay Duplicates
STEP 4b: Class A Stale Referrer Remapping
Script: 
ship/translation-tools/patch_overlay_refs.py
Problem Solved: Prevents garbled save confirmation boxes and menu crashes caused by unshifted MIPS instructions across the EXE and raw overlay sectors.
Algorithm:
Evaluates a 32-instruction dataflow window (WINDOW = 32) tracking register clobbers.
Accounts for signed ADDIU arithmetic using sign-carry high-immediate calculation: 
HighHalf
=
(
Address
+
0x8000
)
≫
16
HighHalf=(Address+0x8000)≫16
Features a cross-sector straddle scanner (detecting lui in sector 
N
N and addiu in sector 
N
+
1
N+1) and a pristine-value filter to prevent double-remapping.
STEP 4c: Class B Type-46 Overlay Duplicate Modules
Script: 
ship/translation-tools/patch_overlay_duplicates.py
Problem Solved: Eradicates the "stubborn Japanese" in combat and shops. In DQ4, the engine loads facilities and battles from compressed Type-46 MIPS overlay modules rather than the raw disc track.
Mechanism:
Extracts all Type-46 modules across the archive containing target blocks (0x048B, 0x047B, 0x047C, 0x047D, etc.).
Groups them into 67 distinct module images across 267 occurrence sites.
Decompresses each module, substitutes the English text block, updates internal word/split-immediate pointers, recompresses with native LZSS, and pads with 0x00 to the exact original length.
STEP 4e: 048C Same-Tree Rebuild
Script: tools/patch_block048c_verified.py
Function: Informational validation stage running a secondary tree rebuild against vp_native_exe.extract to confirm zero-drift compliance on the Font 1 tree header.
STEP 4d: Stale Word-Table Fixed-Point Sweep
Script: study/sweep_word_tables.py
Function: Consumes build/ram_scan_2_cache.pkl and performs a two-pass write plus verify pass, zeroing out stale pointers across internal lookup tables until reaching fixed point.
STEP 4h: 048B Bare Page-Table Remap
Script: study/remap_bare_pages.py
Target: LBA 40217, user offset 0x4D8.
Problem Solved: Eliminates the "Monster Gramps" level-up defect. The battle engine indexes level-up and combat status text through a 94-row bare bit-offset page table containing no 0x048B prefix.
Mechanism: Remaps the 18 leaf-boundary rows (e.g. Row 48, SID 105) to the new English bit offsets while preserving non-render metadata rows.
Phase D: Late Structural Fixes & Execution Guards
STEP 1e: EXE Block 0x048F Church Text
Script: 
ship/translation-tools/patch_exe_blocks.py
Function: Injects English church/priest dialogue directly into EXE block 0x048F. Bounded by strict byte-budget checks, zero drift outside touched regions, and deferred EDC/ECC recalculation.
STEP 4f: D4 Split-Immediate Remap
Script: 
ship/translation-tools/patch_d4_splitimm.py
Function: Remaps residual split-immediate pairs across Blocks 0x048C and 0x048F and fixes Type-39 cutscene call instructions.
STEP 4g: Table & Roster Ref Remap
Script: 
ship/translation-tools/patch_table_refs.py
Function: Performs a dual-pass (write + verify) sweep over table, roster, and character nameplate references.
STEP 4g-2: D2/D3 SID Identity Injection
Script: 
ship/tools/patch_d2_d3_4g2.py
Function:
Class E (D2): Fixes the church offering branch by remapping 14 disc word sites pointing to mid-stream offsets (0x48B10018 / 0x48B10141) to the true sequence starts of SID 575 and SID 578.
Class D (D3): Fixes the victory window letter-salad ("Peynriohre seale") by remapping 3 cloned module sites at LBA 102423, 102434, 102445 from 0x48B07187 to SID 253 START (0x48B07FA5).
STEP 4h-2: F6 VSync Timeout Panic Bypass
Script: 
ship/translation-tools/patch_f6_vsync_timeout.py
Target: EXE offset 0x82860 (RAM 0x80099F60).
Function: Eliminates the crash at second 30.4. Replaces the 10-instruction panic block (which calls BIOS printf and exit(3)) with j 0x80099FA0; nop (E8 67 02 08 00 00 00 00). When CD-ROM streaming masks interrupts, the function returns gracefully to the caller instead of terminating the console thread.
STEP 4h-3: F6 Overlay Collision Redirect
Script: 
ship/translation-tools/patch_f6_overlay_collision.py
Function: Prevents loader-vs-overlay memory collisions during speech staging.
STEP 4h-4: KSEG0 Allocator Guard
Script: 
ship/translation-tools/patch_alloc_guard.py
Target: Injects a 17-instruction MIPS guard into unused executable space at 0x800B8924.
Function: Intercepts allocator calls (0x8009B140). If an allocation targets [0x80011F00, 0x80012F00) (the active resident message module), it remaps the destination to safe scratch memory at 0x80014000, preventing message queue corruption and 
0
xFFFFFFF6
0xFFFFFFF6 bus faults.
Phase E: Verification, Parity, & Sealing
STEP 5: CUE Generation & Mode 2 Form 1 EDC/ECC Recalculation
Tool: 
ship/edcre/edcre-v1.1.0-windows-x86_64-static/edcre.exe
Function: Generates dq4_sovereign_master.cue and performs whole-disc recalculation of 4-byte EDC checksums and 276-byte Reed-Solomon P/Q ECC parity vectors across all modified sectors.
Requirement: Must complete with 0 bad sectors.
STEP 6: Multi-Tree Post-Build Quality Gates
Gate G2: study/g2_dispatch_gate.py sweeps the disc for unmapped or missed-remap referrers.
Gate G10: study/g10_corpus_census.py runs a full decode sweep across every SID to catch silent Japanese fallbacks and ensure all 7 wild freeze tokens (7F06, 7F10, 7F19, 7F1B, 7F1C, 7F1E, 7F35) remain preserved.
Gate G5: study/check_edc3.py runs an independent CRC/EDC sweep.
STEP 7: G11 Seal Gate & Release Labeling
Tool: study/g11_menu_residency_gate.py
Function: Validates paired post-battle RAM dumps (confirming message module survival across battle transitions without memory clobber).
Output:
When verified: Copies and tags the disc into ship/release/DQ4_RCx_<SHA8>.bin, generates matching .cue, and outputs .sha256.
If unverified: Writes build/G11_PENDING and preserves the candidate in build/ without tagging a release.
3. Reference Encoding & MIPS Pointer Mathematics
Strings in DQ4 PSX are packed into 32-bit words using block-relative bit coordinates:

W
o
r
d
=
(
BlockID
≪
20
)
∣
(
BitOffset
+
HTS
×
8
)
Word=(BlockID≪20)∣(BitOffset+HTS×8)
BlockID
BlockID: 12-bit ID (e.g., 0x048C for Font 1 UI, 0x048B for Battle, 0x0474 for Save Menu).
HTS
HTS: Header-to-Text Start offset in bytes (
24
 bytes
24 bytes for 048C/048B; 
936
 bytes
936 bytes for 047B; 
576
 bytes
576 bytes for 047C).
BitOffset
BitOffset: 0-indexed bit position where the Huffman leaf stream for that specific string begins: 
BitOffset
=
(
W
o
r
d
 
&
 0xFFFFF
)
−
(
HTS
×
8
)
BitOffset=(Word & 0xFFFFF)−(HTS×8)
Split-Immediate Representation in Assembly
When loading word pointers into registers, the compiler utilizes two forms:

Unsigned Logical OR (LUI + ORI):
mips


lui   $a0, (Word >> 16)
ori   $a0, $a0, (Word & 0xFFFF)
Signed Immediate Add (LUI + ADDIU): Because addiu sign-extends its 16-bit immediate, whenever Bit 15 is set (
LowHalf
≥
0x8000
LowHalf≥0x8000), it subtracts 
0
x10000
0x10000 from the result. The pipeline compensates using sign-carry arithmetic: 
HighHalf
=
(
W
o
r
d
+
0x8000
)
≫
16
HighHalf=(Word+0x8000)≫16 
LowHalf
=
W
o
r
d
 
&
 0xFFFF
LowHalf=Word & 0xFFFF
mips


lui   $a0, HighHalf
addiu $a0, $a0, LowHalf
4. Pipeline Execution Options & Flow Control
ship/tools/native_build.py provides targeted CLI flags for rapid debugging and partial execution:

Flag	Action
(default)	Runs full E2E pipeline from Step -1 through Step 7.
--skip-dialogue	Reuses existing build/dq4_dialogue_en.bin from Step 1, starting directly at Step 1c/2. Saves ~20–25 minutes of Huffman re-encoding.
--dialogue-only	Stops after Step 1 and Step 1c (used for testing dialogue slot budgets and Huffman tree packing).
--from-step <NAME>	Resumes from a specific named step (e.g. --from-step 4h-2 to run the F6 VSync bypass through Step 7 on an existing master).
--g11-dumps <D1> <D2>	Supplies post-battle RAM dumps to satisfy the G11 residency gate and release the final tagged candidate.

  (cc) Lux Aura. For educational and preservation purposes.
===============================================================

