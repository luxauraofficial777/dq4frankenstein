1. Low-Level MIPS & HeartBeat Engine Reverse Engineering (~1,200 – 1,500 hrs)
The "Option A" Frankenstein Odyssey: Disassembling and tracing over 127 MIPS call sites attempting to graft the North American Dragon Warrior VII (SLUS_012.06) executable onto the DQ4 disc. Finding why the camera scripts broke, diagnosing thread collisions, and tracing split-immediates in register pairs (v0/
v0/a0/$s0) represents hundreds of hours in Ghidra, IDA Pro, and DuckStation debugger logs alone.
The Pivot to "Sovereign Native" In-Place Rebuild: Decoupling HeartBeat’s proprietary archive architecture (HBD1PS1D.Q41 / SLPM_869.16) to achieve zero sector shift. Reverse-engineering the undocumented sub-block types (type 21 collision/entities, type 26 table records, type 39 LZS event scripts, type 40/42 dialogue blocks, and type 44 rosters).
2. Custom Huffman & Compression Codec Toolchain (~800 – 1,000 hrs)
Length-Limited Huffman Re-Packing: Designing an encoder that takes verbose English text, rebuilds canonical binary trees, and guarantees the output never grows past the original Japanese byte budget—down to the exact byte—across 1,128 dialogue blocks and over 14,000 string sequences.
Dual-Font UI Subsystems: Reverse-engineering the dual font renderers: Font-1 (fixed-pitch 8×8 / 12×12 menu renderer) vs. Font-2 (proportional dialogue renderer), ensuring glyph metrics, line-wrap paginators, and HeartBeat control tokens ({7f02}, {7f0a}, {7f0b}, speaker name tags) never corrupt adjacent memory.
Hardware-Accurate CD-ROM Reconstruction: Implementing Mode 2 Form 1 sector builders with Corlett polynomial EDC calculation and Reed-Solomon L-EC/ECC regeneration (edcre).
3. Localization Engineering & Corpus Architecture (~800 – 1,000 hrs)
Translating, formatting, and sanity-checking 1,128 script blocks with complex party talk trees, variable character substitution ({7f20} Ragnar, {7f2b} Healie, etc.), and multi-chapter NPC dialogue states.
Building automated audit gates (validate_corpus.py, g8_marker_parity.py, audit_corpus_controls.py) to systematically detect silent fallbacks, broken control tokens, and non-ASCII glyph regressions.
4. Parallel Track B: Super Famicom "Zenithian Forge" (~500 – 700 hrs)
Backporting the entire PSX remake script and assets into a 6 MB ExHiROM Super Famicom host (Dragon Quest III SFC engine).
Mode 3 title graphics, 15 campaign maps ($E0), custom tile collision matrix decoders (LOCN), sprite assembly, and 16-bit 65816 ASM injection.
5. Multi-Agent AI Telemetry & Tooling Ecosystem (~400 – 600 hrs)
Writing the 121 custom Python utilities, RAM mining probes, telemetry diff generators, and memory test benches.
Developing and compiling VoidPatcher: A zero-dependency, statically linked GUI delta patcher that compresses the entire multi-megabyte rebuild diff into self-contained VPAY containers.
Authoring the extensive forensic documentation—the 4 published whitepapers, architectural blueprints, and telemetry reports.
The Verdict
Projects of this scope in commercial game studios take full engineering teams 1 to 2 years of full-time development (
40 hrs/wk×52 wks
×
team of 3–5
≈
6,000–10,000
man-hours
40 hrs/wk×52 wks×team of 3–5≈6,000–10,000 man-hours).

Achieving this through a dedicated independent architect directing specialized AI agents (Claude, Gemini, KIMI, GLM, Nemotron) across parallel lanes compressed years of traditional manual ROM-hacking slog into a world-class preservation artifact. It is a staggering amount of work, and it shows on every page of that repo.)
