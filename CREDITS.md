# Credits — Dragon Quest IV: The Zenithian Chronicles (PSX)

**Project:** Sovereign Native English Localization Suite  
**Maintenance & Direction:** Lux Aura & The VoidWalkers Research Project  
**Target Target Binary:** `SLPM_869.16` (Sony PlayStation) + `HBD1PS1D.Q41`  
**Pipeline Standard:** Zero Sector Shift, Native In-Place Re-authoring, Full EDC/ECC Compliance  
**Date:** September 13, 2026  

---

## 1. Project Leadership & Engineering Architecture

| Contributor / Lead | Primary Role | Engineering Scope & Core Responsibilities |
|---|---|---|
| **Lux Aura** | Project Director & Lead Systems Architect | Overall project direction, native reverse-engineering leadership, Sovereign Engine architecture, zero-drift sector planning. |
| **Big Pickle** | Systems Analysis & Corpora Mining | Super Famicom content authoring, RAM dump forensics, schema mapping, and cross-generational corpora mining. |
| **Omen Alpha** | Infrastructure & Systems Engineering | DQ4 content authoring, build infrastructure, live RAM debugging, DQ1/2/3 ROM mining, RC3 final build, stale word-table sweep, block `0x048C` rebuild, VoidPatcher design & build. |

---

## 2. Foundation Research & Pioneer Lineage

| Pioneer / Researcher | Repository / Research Tooling | Core Historical Breakthrough |
|---|---|---|
| **Markus Schroeder** | Foundation HBD Tooling (Java Lineage) | Foundational HBD archive extraction research; engineered the first functional format decoders that made modern disc extraction possible. |
| **Mandy Wilkens** | `dq4psxtrans` | Early Python-based text extraction frameworks and dialogue tooling research. |
| **RadMageIRL** | `DQIV_PSX_TOOLS` | Forensic engine inspection tooling; primary discovery of the Font 1 atlas and metric mapping. |

---

## 3. Autonomous AI Pair Programming & Neural Research Cohort

| Neural Agent / Model | Primary Research Specialization | Forensic Milestones & Subsystem Deliverables |
|---|---|---|
| **KIMI** | Memory Forensics & Empirical Audits | Deep codebase indexing, 2MB RAM dump analysis, script disassembly, forensic audits, and verification gate engineering (G2, G8, G10, G11). |
| **GLM** | MIPS Static Analysis & Dataflow Tracking | MIPS R3000A dataflow tracing, split-immediate sign-carry mechanics, root-cause isolation for Type-46 overlay duplicate sectors, forensic patch verification. |
| **Claude (Opus)** | Pipeline Architecture & Algorithmic Codecs | Sovereign build-pipeline authoring, native length-limited Huffman engine design, Type-39 cutscene remapping, pre-flight corpus verification gates. |
| **Gemini** | Text Mining, Referrer Logic & Integration | Font 2 proportional text mining, gap audits, Class A/B referrer remapping, Class B Type-46 overlay duplicate patcher, deterministic build integration. |
| **Nemotron** | Binary Mining & Asset Injection | Cross-game ROM mining, combat overlay (`0x048B`) text injection, master monster database alignment, Font 1 character nameplate layering. |
| **Cascade** | Operational Orchestration | Cross-session continuity, technical lane coordination, and task synchronization. |

---

## 4. Technical Foundations & Tooling Frameworks

| Subsystem / Tool | Core Author / Maintainer | Implementation within Sovereign Pipeline |
|---|---|---|
| **EDC/ECC Parity Engine** | Neill Corlett (`ecmtools`) | Implementation of zeroed-MSF-address raw sector parity recalculation; validated 276/276 against `edcre`. |
| **HBD Container Architecture** | Markus Schroeder & VoidWalkers | Reverse-engineering the flat 2,048-byte sector container `HBD1PS1D.Q41` to achieve zero sector drift. |
| **Font 1 Atlas & Layout** | VoidWalkers Project | Empirical RAM mapping and VRAM coordinate analysis for 8×14 fixed-pitch UI menu rendering. |
| **Huffman Codec Infrastructure** | VoidWalkers Project | Length-limited binary tree generation guaranteeing exact in-place byte budgets across 1,358 dialogue containers. |
| **DuckStation Emulator** | Stenzek & DuckStation Contributors | Low-level execution environment, hardware register telemetry, and headless boot verification harness. |

---

## 5. Original Intellectual Property & Heritage

* **Original Work:** *Dragon Quest IV: Michibikareshi Mono Tachi* (2001, PlayStation)
* **Original Engine Development:** HeartBeat Inc. (Led by Manabu Yamana & Keisuke Eriguchi)
* **Publisher:** Enix Corporation
* **Design & Scenario:** Yuji Horii (Armor Project)
* **Character & Visual Art:** Akira Toriyama (Bird Studio)
* **Musical Composition:** Koichi Sugiyama (Sugiyama Kobo)
* **Executable ID:** `SLPM_869.16`

*Disclaimer: This project distributes no copyrighted binary assets. All transformations are applied deterministically in-place to the user's legally acquired Japanese disc image.*

---

## 6. Official Channels & Resources

* **Record Label & Publisher:** Lux Aura ([Bandcamp](https://luxaura.bandcamp.com) | [Facebook](https://www.facebook.com/LuxAuraOfficial/) | [YouTube](https://www.youtube.com/LuxAuraOfficial))
* **Steam Publisher Portal:** Coming Soon — [Track via SteamDB](https://steamdb.info/publisher/Lux+Aura/)
* **Engineering Source Repository:** [Lux Aura GitHub](https://github.com/luxauraofficial777)

---

## 7. Acknowledgments & Scene Heritage

* **The Dragon Quest Research Community:** For four decades of continuous ROM data mining, structural documentation, and preservation effort.
* **The PlayStation Emulation Community:** For open-source diagnostic utilities, GTE debugging frameworks, and bit-accurate hardware emulation.
* **Alpha & RC Playtest Vanguard:** Everyone who stress-tested the RC cycles and the Rebuild B baseline across DuckStation and real hardware.

**ORDER. PRECISION. FIDELITY.**
