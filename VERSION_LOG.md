# Sovereign Native Pipeline — Version & Lineage Log (V90–Rebuild B)

**Project:** Dragon Quest IV (PSX) Localization Architecture  
**Document ID:** LOG-VER-HBE-LINEAGE-V90-REBUILD-B-20260913  
**Authors:** Lux Aura, Omen Alpha, Big Pickle / VoidWalkers Research Project[cite: 1]  
**Target Architecture:** Sony PlayStation (`SLPM_869.16` / `HBD1PS1D.Q41`)[cite: 1]  
**Date:** September 13, 2026  

---

## 1. Lineage Overview & Paradigm Evolution

| Epoch / Milestone | Build Strategy | Primary Binaries | Core Failure Mode / Architectural Achievement |
|---|---|---|---|
| **V90 – V97** *(Jul 2026)* | Exploratory "Frankenstein" Graft | `SLUS_012.06` (DW7 US) + `HBD1PS1D.Q41`[cite: 1] | **Stalled at Runtime:** Async FMV callback locks, MDEC 569 FPS loop spins, black screen GPU stalls, LBA sector drift. |
| **V99 / Rebuild B** *(Aug–Sep 2026)* | Sovereign Native In-Place Rebuild[cite: 1] | Native `SLPM_869.16` + `HBD1PS1D.Q41`[cite: 1] | **Playable Master:** Zero sector drift, in-place length-limited Huffman re-encoding, full Mode 2 Form 1 EDC/ECC compliance, verified Chapter 1[cite: 1]. |

---

## 2. Iteration History: The Exploratory Frankenstein Era (V90–V97)

### V90 — Initial Architecture Commit (July 24, 2026)
* **Strategy:** 25-step binary graft swapping the DW7 US executable (`0x80017F00`, `PC0=0x8008E284`) onto the DQ4 disc base.
* **Core Interventions:** Appended hybrid Huffman tree (`0x800BC700`), narrowed BSS clear range (`0x800BD100`–`0x800D9E80`), patched disc checks, redirected LBA table (354 → 362), and injected MISS-1 boot copy stub.
* **Failure Analysis:**
  * **BIOS POST 03 Stall:** Final appended binary was not sector-aligned (680,100 bytes); CD-ROM BIOS halted execution before loading `PC0`.
  * **MIPS Sign-Extension Bug:** Raw bit-shifting for `lui`/`addiu` split-immediates corrupted target pointers due to negative sign-extension on lower 16-bit immediates[cite: 1].
  * **FMV Spin Trap:** Disabling XA/MDEC caused complete execution failure.

### V95 — Execution Past BIOS (July 24, 2026)
* **Corrections Implemented:**
  * Resolved sign-extension via mathematical `split_addr()` helper handling signed carry for `lui`/`addiu` pairs[cite: 1].
  * Enforced strict 2,048-byte sector alignment padding on `text_size` (`0xA58A4` → `0xA6000`, 681,984 bytes total).
  * Removed FMV stubs in an attempt to let the native engine advance.
* **Failure Analysis:** BIOS POST completed cleanly (0F→0E→01→02→03→04→05→06→07→kernel init), but the engine sought DW7 FMV sector LBA 146621 (empty on DQ4 media), sending the MDEC hardware decoder spinning at 569 FPS on garbage bytes.

### V96 — GPU Display Latches & State Machine Blocker (July 24, 2026)
* **Corrections Implemented:**
  * Restored stubbing for XA/STR (`0x8008AEF4`) and MDEC init (`0x8008CAD0`) using `li v0, 0; jr ra; nop`.
  * Expanded MISS-1 boot stub from 14 to 18 instructions to force GP1 display enable: `GP1(0x03) = 0x03000000` via register `0x1F801814`.
* **Failure Analysis:** Game ran at a stable 59.82 FPS, but GPU command rendering remained completely idle (0.00). Disassembly revealed the engine's main loop waits for an asynchronous completion event (VBlank callback, DMA interrupt, or FMV state transition bit) that was never signaled by dummy stub returns.

### V97 — Full Structural Pipeline Audit (July 25, 2026)
* **Corrections Implemented:**
  * Fixed `DQ4_HBD_END` constant calculation (`psx_binary_ops.h:124`): Realigned base from DW7 LBA 355 to DQ4 LBA 362 (`362 + 155975 = 156337`).
  * Updated Primary Volume Descriptor (PVD) at offset 80 to correct both-endian 32-bit Volume Space Size matching physical sector count.
  * Packaged verified build harness as `frankenstein_pipeline_v097.zip`.
* **Lineage Conclusion:** Proved that grafting a foreign executable (`SLUS_012.06`) introduces unresolvable runtime state machine divergence, forcing the transition to the **Sovereign Native** architecture[cite: 1].

---

## 3. The Sovereign Native Breakthrough (Rebuild B Baseline)

### V99 / Rebuild B — Playable Master Baseline (August–September 2026)
* **Core Paradigm Shift:** Completely abandoned foreign executable grafting[cite: 1]. Retained native `SLPM_869.16` and authored all MIPS R3000A patches and text streams directly in-place with zero sector drift across all 156,487 LBAs[cite: 1].
* **Core Technical Milestones Achieved:**
  * **Length-Limited Huffman Encoding:** Re-encoded 1,358 dialogue containers in `HBD1PS1D.Q41` within strict original Japanese byte boundaries[cite: 1].
  * **Type-39 LZSS Event Script Remapping:** Decompressed, remapped, and re-packed 612 cutscene scripts, resolving throne room and camera stalls[cite: 1].
  * **Font 1 Atlas & Metrics Integration:** Injected 8×14 fixed-pitch English glyphs into resident block `0x048C` at file offset `0x97AC8`[cite: 1].
  * **FIX B Combat & Stat Alignment:** Replaced battle strings in block `0x048B`[cite: 1]. Anchored the 18-row Monster-Gramps stat dispatcher table to LBA 40217 to eliminate post-battle memory corruption[cite: 1].
  * **Two-Phase EDC/ECC Parity:** Regenerated all Mode 2 Form 1 sector headers, L-EC, EDC, and Reed-Solomon P/Q parity using `edcre`, achieving hardware-verified boot stability[cite: 1].

---

## 4. Comprehensive Version Comparison Matrix

| Version | Executable Target | Disc Framing | Boot Status | Render Status | Audio / Video Subsystem | Primary Failure Mode / Resolution |
|---|---|---|---|---|---|---|
| **V90** | `SLUS_012.06` (DW7) | Grafted Mode 2 | POST 03 Stall | None | Uninitialized | Non-aligned `text_size`; negative MIPS sign-extension on addresses. |
| **V95** | `SLUS_012.06` (DW7) | Grafted Mode 2 | Kernel Init Pass | None (Black Screen) | 569 FPS MDEC Spin | Seeks missing FMV LBA 146621; decoder hangs on garbage bytes. |
| **V96** | `SLUS_012.06` (DW7) | Grafted Mode 2 | 59.82 FPS Idle | GPU 0.00 (Black Screen) | FMV Stubs Active | Main loop hangs awaiting async FMV VBlank / DMA callback signal. |
| **V97** | `SLUS_012.06` (DW7) | Grafted Mode 2 | 59.82 FPS Idle | GPU 0.00 (Black Screen) | Pipeline Verified | Fixed `DQ4_HBD_END` (LBA 362) and PVD size; proved graft dead-end. |
| **Rebuild B**[cite: 1] | `SLPM_869.16` (Native)[cite: 1] | Zero Sector Shift[cite: 1] | 59.94 FPS Pass[cite: 1] | Active 3D + 2D Billboards[cite: 1] | Fully Synced Native SEQq[cite: 1] | **SOLVED:** In-place re-authoring, Fix B stat anchors, Gate G0–G8 PASS[cite: 1]. |

---

## 5. Architectural Retrospective & Active Engineering Tasks

* **The Graft Fallacy:** Attempting to force an executable designed for one resource tree (`SLUS_012.06` expecting `HBD1PS1D.Q71`) onto a different file system layout inevitably triggers unresolvable bus arbitration stalls, memory leaks, and CD-ROM DMA timeouts.
* **The Sovereign Solution:** Treating the original Japanese binary as immutable hardware architecture and performing surgical, length-limited bitstream injections eliminates the need for foreign stubs, bypasses, or filesystem relocation[cite: 1].
* **Active Rebuild C Scope:** Resolving the Church facility menu overlay (Task A1 / Gate G2) and locking duplicate sector byte parity (`0x00A2`, `0x00A9`, `0x00B4`) for the Chapter 2 Zamoksva transition[cite: 1].

**ORDER. PRECISION. FIDELITY.**
