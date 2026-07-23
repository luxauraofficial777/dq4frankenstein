# Frankenstein Forked Development Pathways — Playing God

## Premise

We are Heart Beat North America. We have:
- Full decompilation of both DW7 and DQ4 executables (3278 functions, 1.25M MIPS instructions)
- A hybrid Huffman tree with 100% round-trip fidelity (1480 bytes, 368 leaves)
- An HBD re-encoder that handles 5 block types (23/24/25/27 + field swaps for 39/40/42)
- 30+ build profiles documenting every iteration from v12 to v44
- A custom PSX emulator (CyberGrime) with 200M instruction stability
- A DuckStation integration with log capture
- An HBE library with proper archive parsing, Huffman tree cracking, and ISO9660 writing
- Full translation data (15,318 entries across 1,105 blocks)
- EDC/ECC regeneration, ISO directory manipulation, sector-level disc writing

The game boots. It shows the Enix logo, FMV, title screen, and adventure log. Then it says
"Insert Disc 1" / "Wrong Disc" and loops. This is not one problem — it is two:

1. **HBD format incompatibility**: 162 blocks (types 32/46) fail re-encoding → BSS corruption → crash
2. **Secondary disc validation**: 10-11 patches cover the primary disc check, but a secondary
   check fires after character creation / FMV preload that our patches don't cover

This document defines **four forked development vectors**, each with phased milestones,
targeting a gold-standard playable build. All vectors converge on a C++-native toolchain.

---

## The C++ Imperative

**Python and Java are for telemetry, analysis, and translation management.
Binary surgery — MIPS patching, LBA/BSS allocation, HBD re-encoding — must be C++.**

We cannot guess at exact MIPS instructions or file table allocations through ctypes
wrappers and Python struct.unpack. The current `psx_ops.dll` is a step in the right
direction but it's a black box called from Python. The gold standard is a single C++
binary that:
- Parses EXE headers with typed structs (not `struct.unpack_from`)
- Patches MIPS instructions with an instruction encoder (not hex string literals)
- Re-encodes HBD blocks with proper C++ class hierarchy (not Python dicts)
- Writes disc sectors with direct file I/O (not Python file.seek)
- Validates every step with assertions that abort on failure (not print warnings)

**All four forks mandate C++ as the build language. Python survives only as:
- Translation data management (JSON processing, CSV handling)
- Telemetry analysis (JSON parsing, chart generation)
- DuckStation log parsing
- Test orchestration (calling the C++ builder, launching the emulator)**

---

## Fork Alpha: Fix What We Have (The Pragmatist's Path)

**Philosophy**: The pipeline works. 90% of blocks re-encode. The disc boots.
Fix the remaining 10% and the secondary disc check. Ship it.

**Success criteria**: Playable game past character creation with English text.

### Phase A1: HBD Re-encoder Type 32/46 Fix (Days 1-3)

**Blocker**: `HbdReencoder::process_hbd()` in `psx_binary_ops.cpp` fails on 162 blocks.

**Root cause**: Types 32 and 46 have different internal structures:
- Type 32: Script/event data with nested sub-blocks (152 instances in DQ4)
- Type 46: Message/event data with compression flags (612 instances in DQ4)
- DW7 equivalents: Type 22 (script) and Type 26 (message) — different header layout

**Actions**:
1. Extract 5 sample blocks each of type 32 and 46 from `dq4_heart_v17.bin`
2. Hex-dump their headers and compare with DW7 type 22/26 headers
3. Identify the structural difference (likely: tree_end/text_end semantics, or
   sub-block count location, or compression flag position)
4. Add type-specific conversion logic in `HbdReencoder::process_hbd()`
5. Add unit tests: `hbd_reencode_test.cpp` with fixtures for each type
6. Rebuild `psx_ops.dll`, rebuild `dq4_frankenstein_v44.bin`
7. **Gate**: Validation passes with 0 residual DQ4 types

**Deliverable**: `psx_binary_ops.cpp` with complete type 32/46 handling + test suite

### Phase A2: Secondary Disc-Check Eradication (Days 3-5)

**Blocker**: "Insert Disc 1" loop after character creation.

**Root cause**: The 10 surgical patches cover the primary disc check at `0x80078FF4`
and the disc change handler at `0x800506CC`. But the game has a **secondary validation**
that fires when it tries to load the first game-world asset (FMV at LBA 146621 or
the first HBD data block after the title screen).

**Known addresses**:
- `0x800E3D90`: Disc number variable (we force it to 1, but it may be re-read)
- `0x8007932C`: `lui $v0, 0x800E + lw $v0, 0x3D98($v0)` — reads disc number
- `0x80079428`: Second disc number read
- `0x80079460`: `jal 0x800506CC` — calls disc change handler
- `0x8007948C`: Second call to disc change handler
- `0x8008AEF4`: Post-disc-change verify (157 call sites!)

**Actions**:
1. Use decompiled DW7 EXE to trace the code path from character creation → disc check
2. Find ALL `jal 0x800506CC` (disc change handler) call sites — we neutralized the
   handler itself, but the 157 call sites at `0x8008AEF4` may have their own logic
3. Patch the disc number LOAD instructions:
   - Replace `lw $v0, 0x3D98($v0)` at 0x8007932C+4 with `addiu $v0, $zero, 1`
   - Replace `lw $v0, 0x3D98($v0)` at 0x80079428+4 with `addiu $v0, $zero, 1`
4. Patch the post-verify function at `0x8008AEF4`:
   - Either: `jr $ra; nop` at function entry (neutralize entirely)
   - Or: Patch the 157 call sites to NOP (safer, more surgical)
5. **Gate**: DuckStation smoke test passes 120 seconds past character creation

**Deliverable**: Extended disc-check patch set (15-20 patches) covering all validation paths

### Phase A3: C++ Builder Consolidation (Days 5-7)

**Actions**:
1. Port all Python patch logic from `frankenstein_builder.py` into `frankenstein_build.cpp`
2. Remove Python builder dependency entirely
3. Single command: `frankenstein_build.exe --profile gold --output dq4_gold.bin`
4. **Gate**: C++ builder produces byte-identical output to Python v44

**Deliverable**: `frankenstein_build.exe` — standalone C++ builder, no Python dependency

### Phase A4: Gold Standard Verification (Days 7-8)

**Actions**:
1. Automated build + verify + test pipeline:
   ```
   frankenstein_build.exe --profile gold --output dq4_gold.bin
   frankenstein_verify.exe dq4_gold.bin  (sector-by-sector patch check)
   duckstation --headless --run 120s dq4_gold.cue  (smoke test)
   ```
2. **Gate**: All 3 steps pass. Game runs past character creation.

**Risk**: Medium. Types 32/46 may have fundamentally different compression that
resists simple header conversion. If so, fall through to Fork Beta.

---

## Fork Beta: DQ4 Native (The Purist's Path)

**Philosophy**: Stop forcing DQ4 data through a DW7 engine that doesn't understand it.
Use the DQ4 EXE natively. It already knows how to parse every block type. Patch only
what's needed to make it run on a DW7 disc shell.

**Success criteria**: Playable game with DQ4 engine + English text + DW7 FMV/audio.

### Phase B1: DQ4 EXE Disc Adaptation (Days 1-4)

**Core insight**: DQ4 EXE (`SLPM_869.16`, 692,224 bytes) natively handles types
23/24/25/27/32/46 with per-block Huffman trees. No re-encoding needed. No hybrid
tree needed. No BSS patching needed. No tree ref patching needed.

**What DQ4 EXE needs to run on a DW7 disc**:
1. **Disc ID patch**: DQ4 checks for its own disc ID. Patch to accept DW7 disc ID
   (or patch the ISO volume descriptor to report DQ4's ID)
2. **SYSTEM.CNF update**: Point boot file to DQ4 EXE instead of DW7 EXE
3. **HBD LBA**: DQ4 expects HBD at sector 362. DW7 disc has HBD at 354.
   Either: patch DQ4 EXE's HBD LBA reference (362→354), or place HBD at 362
4. **FMV/audio**: DQ4 FMV is at different LBAs than DW7. Either:
   - Patch DQ4 EXE's FMV LBA references to point to DW7 FMV locations
   - Or: copy DW7 FMV data to DQ4-expected LBAs (if space allows)
5. **Font data**: DQ4 has its own font in HBD. DW7 has fonts in HBD.
   With DQ4 HBD, DQ4 fonts come along automatically.

**Actions**:
1. Extract DQ4 EXE from `decompile/decompile_output/dq4_jp/dq4_jp.exe`
2. Use decompiled DQ4 EXE to find:
   - Disc ID check (search for `SLPM_869` or disc ID string)
   - HBD LBA reference (search for sector 362)
   - FMV LBA references (search for STR file entries)
3. Patch DQ4 EXE:
   - Disc ID: `jr $ra; li $v0, 1` or accept DW7's `SLUS-012`
   - HBD LBA: 362 → wherever we place DQ4 HBD on the DW7 disc
4. Assemble disc:
   - SYSTEM.CNF → boot `SLPM_869.16` (DQ4 EXE)
   - DQ4 EXE at sector 24
   - DQ4 HBD (from `dq4_heart_v17.bin`) at sector 362 (original location)
   - DW7 FMV/audio at their original sectors (unchanged)
5. **Gate**: DQ4 EXE boots on the hybrid disc

**Deliverable**: `dq4_native_v1.bin` — DQ4 EXE on DW7 disc shell

### Phase B2: FMV/Audio Bridge (Days 4-6)

**Problem**: DQ4 EXE expects DQ4 FMV at specific LBAs. DW7 FMV is at different LBAs.

**Options**:
- **Option A**: Patch DQ4 EXE's FMV LBA table to point to DW7 FMV sectors
  - Requires finding the FMV LBA table in DQ4 EXE (use decompilation)
  - DW7 FMV may be in different format (different STR encoding)
- **Option B**: Skip FMV entirely — patch DQ4 EXE to NOP the FMV play calls
  - Simpler but loses intro video
- **Option C**: Copy DQ4 FMV from DQ4 JP disc to the hybrid disc
  - DQ4 JP disc has the original FMV. Just copy those sectors.
  - But DQ4 JP FMV is in Japanese (intro text)

**Recommended**: Option C (copy DQ4 FMV) for correctness, Option B (skip) as fallback.

**Actions**:
1. Find DQ4 FMV sectors on DQ4 JP disc (STR files in ISO directory)
2. Find DW7 FMV sectors on DW7 disc
3. If DQ4 FMV fits in available space: copy DQ4 FMV to hybrid disc
4. If not: patch DQ4 EXE to skip FMV playback
5. **Gate**: Game boots past FMV to title screen

**Deliverable**: `dq4_native_v2.bin` with working FMV or FMV skip

### Phase B3: Full Playthrough Test (Days 6-8)

**Actions**:
1. DuckStation smoke test: 300 seconds, past character creation, into game world
2. Verify English text displays correctly (DQ4 per-block trees, no hybrid needed)
3. Verify audio plays (DW7 audio layer or DQ4 audio)
4. Verify save/load works (memory card access)
5. **Gate**: 5 minutes of gameplay with no crash

**Risk**: Low for HBD compatibility (native engine), Medium for FMV/audio bridging.
The DQ4 EXE is 16KB larger than DW7 EXE — need to verify it fits in the sector
allocation (sectors 24-354 = 330 sectors × 2048 = 675,840 bytes; DQ4 EXE is 692,224
bytes = 338 sectors — overflows by 8 sectors into HBD space).

**Mitigation**: Shift HBD start from 354 to 362 (matching DQ4's original layout).
This gives 338 sectors for EXE (24-361) and HBD starts at 362.

---

## Fork Gamma: Cleanroom Engine ( The Architect's Path)

**Philosophy**: We have the decompiled source for both engines. We have the HBE
library. Build a purpose-built engine that takes the best of both — DQ4's block
parser, DW7's CD-ROM handling, our translation layer — and compile it as a custom
PSX EXE.

**Success criteria**: A custom EXE that boots on any PSX disc, parses DQ4 HBD
natively, displays English text, and plays DW7 FMV/audio.

### Phase G1: Engine Specification (Days 1-5)

**Actions**:
1. Extract DQ4's HBD parser from decompiled source (functions at 0x8002EC2C,
   0x80042A04, 0x8002F238, 0x80075430)
2. Extract DW7's CD-ROM handling from decompiled source
3. Extract DW7's GPU rendering pipeline
4. Specify the engine interface:
   ```c
   // Engine entry point
   void main() {
       cdrom_init();
       hbd_mount(362);  // DQ4 HBD at sector 362
       gpu_init();
       event_loop();
   }
   ```
5. Design the module graph:
   ```
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ CD-ROM   │  │ HBD      │  │ GPU      │
   │ (DW7)    │──│ (DQ4)    │──│ (DW7)    │
   └──────────┘  └──────────┘  └──────────┘
        │              │              │
        │         ┌──────────┐        │
        └─────────│ Huffman  │────────┘
                  │ (Hybrid) │
                  └──────────┘
                        │
                  ┌──────────┐
                  │ Text     │
                  │ (English)│
                  └──────────┘
   ```

**Deliverable**: Engine specification document + module interface headers

### Phase G2: Core Engine Implementation (Days 5-15)

**Actions**:
1. Implement HBD parser in C (from decompiled DQ4 functions)
2. Implement Huffman decoder (from decompiled DQ4 decompressor at 0x80073670)
3. Implement CD-ROM layer (from decompiled DW7 CD-ROM wrapper at 0x80097814)
4. Implement GPU layer (from decompiled DW7 GPU functions)
5. Implement text rendering (from decompiled DQ4 text renderer)
6. Cross-compile to MIPS using GCC MIPS toolchain or assemble by hand
7. Link as PS-X EXE with proper header (load address, PC0, GP, SP)

**Deliverable**: `frankenstein_engine.exe` — custom PSX EXE

### Phase G3: Disc Assembly (Days 15-17)

**Actions**:
1. Assemble disc with custom EXE + DQ4 HBD + DW7 FMV/audio
2. Generate ISO 9660 directory, PVD, CUE sheet
3. Regenerate EDC/ECC
4. **Gate**: EXE boots in DuckStation

### Phase G4: Iteration (Days 17-30)

**Actions**:
1. Debug rendering, input, audio, save/load
2. Fix text display issues
3. Optimize performance
4. **Gate**: Full playable game

**Risk**: HIGH. This is essentially writing a game engine from decompiled code.
The decompilation is 55.7% logic-verified and 52.9% cycle-accurate. Missing
functions (297) and parity failures (1543) mean significant manual work.

**Mitigation**: Use this as a long-term research vector. Don't block Forks Alpha/Beta
on this. If Alpha or Beta succeed, Gamma becomes a "perfect version" pursuit.

---

## Fork Delta: Runtime Shim ( The Surgeon's Path)

**Philosophy**: Don't re-encode HBD. Don't swap EXEs. Don't build a new engine.
Instead, patch the DW7 EXE at runtime to handle DQ4 block types natively. The
DW7 engine already has a block type dispatcher — just teach it about types 32/46.

**Success criteria**: DW7 EXE with runtime DQ4 block type support, no HBD re-encoding.

### Phase D1: Block Type Dispatcher Analysis (Days 1-3)

**Actions**:
1. Use decompiled DW7 EXE to find the block type switch/dispatch
2. The dispatcher is likely at or near `0x80042A04` (block header reader)
3. Identify the switch instruction pattern:
   ```mips
   # Expected pattern:
   sltiu $v0, $a2, 4     # if type < 4
   beq $v0, $zero, ...   # branch to error
   sll $v0, $a2, 2       # multiply by 4 (jump table entry size)
   addu $v0, $v0, $t1    # add jump table base
   lw $v0, 0($v0)        # load jump target
   jr $v0                # jump to handler
   ```
4. Find the jump table and the handler for each type
5. Identify where types 32/46 would need to be added

**Deliverable**: Documented block type dispatch flow + jump table location

### Phase D2: Code Cave Identification (Days 3-4)

**Actions**:
1. Find unused space in the DW7 EXE for new handler code:
   - End of EXE (after current text_size, before HBD)
   - BSS region (if we can steal a few hundred bytes)
   - NOP slides or dead code paths
2. Calculate available space: need ~200 bytes for two new handlers
3. **Gate**: Code cave identified with sufficient space

### Phase D3: DQ4 Type 32/46 Handler Injection (Days 4-7)

**Actions**:
1. Write MIPS handler for type 32 (script/event data):
   ```mips
   type_32_handler:
       # Read DQ4-format header
       # Extract per-block tree
       # Decompress with per-block tree (not global)
       # Return pointer to decompressed data
       jr $ra
   ```
2. Write MIPS handler for type 46 (message/event data):
   ```mips
   type_46_handler:
       # Similar to type 32 but with compression flag handling
       jr $ra
   ```
3. Patch the jump table to redirect types 32/46 to new handlers
4. Or: Patch the `sltiu` bound check to accept higher type numbers
5. **Gate**: DW7 EXE with extended type dispatch, no HBD re-encoding needed

**Deliverable**: Patched DW7 EXE with runtime DQ4 block type support

### Phase D4: Full Build + Test (Days 7-9)

**Actions**:
1. Build disc with:
   - Patched DW7 EXE (with type 32/46 handlers + disc-check bypass)
   - DQ4 HBD as-is (no re-encoding — native DQ4 format)
   - DW7 FMV/audio (unchanged)
2. No hybrid tree needed — DQ4 per-block trees used natively
3. No BSS patching needed — no tree appended to EXE
4. No LBA shifting needed — EXE stays original size
5. DuckStation smoke test: 300 seconds
6. **Gate**: Game runs past character creation

**Risk**: Medium. Requires precise MIPS patching with register preservation.
The handlers must not clobber registers that the caller expects to be preserved.
The jump table modification must handle all edge cases.

**Advantage**: Eliminates the entire HBD re-encoding pipeline. The most elegant
solution — teach the engine a new trick instead of transforming the data.

---

## Convergence Matrix

| Factor | Fork Alpha | Fork Beta | Fork Gamma | Fork Delta |
|--------|-----------|-----------|------------|------------|
| **Time to playable** | 7-8 days | 6-8 days | 30+ days | 7-9 days |
| **Risk** | Medium | Low-Medium | HIGH | Medium |
| **HBD re-encoding** | Required (fix 32/46) | NOT required | NOT required | NOT required |
| **Hybrid tree** | Required | NOT required | Optional | NOT required |
| **Disc-check patches** | 15-20 | 3-5 (DQ4's own) | 0 (custom engine) | 15-20 |
| **FMV/audio** | DW7 (native) | Bridge needed | Full control | DW7 (native) |
| **C++ migration** | Phase A3 | Full C++ | Full C++ | Full C++ |
| **Decompilation use** | Trace disc check | Find LBAs | Full rebuild | Find dispatch |
| **Lines of new code** | ~500 | ~300 | ~10,000 | ~800 (MIPS) |
| **Elegance** | Low | Medium | High | High |
| **Maintainability** | Low | High | Highest | Medium |

---

## Recommended Strategy

### Immediate (Days 1-3): Dual Track Alpha + Delta

**Run Alpha and Delta in parallel.**

- **Alpha Phase A1**: Fix the HBD re-encoder for types 32/46 (C++ work in `psx_binary_ops.cpp`)
- **Delta Phase D1**: Analyze the block type dispatcher in decompiled DW7 EXE

Both attack the same blocker (types 32/46) from different angles:
- Alpha converts the data to DW7 format (upstream fix)
- Delta teaches the engine to read DQ4 format (downstream fix)

**Whichever succeeds first gives us a working build.**

### Short-term (Days 3-8): Complete the Winner

If Alpha A1 succeeds (re-encoder fixed):
→ Proceed to Alpha A2 (secondary disc-check) → A3 (C++ builder) → A4 (gold standard)

If Delta D1-D3 succeeds (runtime shim):
→ Proceed to Delta D4 (full build) → then port disc-check fixes from Alpha A2

### Parallel (Days 1-8): Beta as Insurance

**Run Beta B1 in parallel as a low-risk fallback.**

DQ4 native EXE is the cleanest approach. If Alpha and Delta both stall on types 32/46,
Beta bypasses the entire problem by using the engine that already understands the data.
The only work is disc ID patching and FMV bridging.

### Long-term (Days 8-30): Gamma as Perfection

Once we have a playable build (Alpha, Beta, or Delta), pursue Gamma to build the
definitive version — a purpose-built engine with no compromises.

---

## C++ Toolchain Roadmap

### Current State
```
Python (frankenstein_builder.py, 4342 lines)
    │
    ├── calls ctypes → psx_ops.dll (C++ black box)
    ├── calls subprocess → edcre.exe (external tool)
    └── calls Python functions for all patch logic
```

### Gold Standard
```
frankenstein_build.exe (C++, single binary)
    │
    ├── exe_patch.cpp      — EXE loading, header parsing, MIPS patching
    ├── hbd_reencode.cpp   — HBD block parsing, Huffman re-encoding
    ├── disc_assemble.cpp  — ISO 9660, sector I/O, EDC/ECC
    ├── mips_encode.cpp    — MIPS instruction encoder (no hex literals)
    ├── verify.cpp         — Per-step validation with assertions
    └── main.cpp           — CLI entry point, profile selection
```

### Migration Plan

| Component | Current | Target | Priority |
|-----------|---------|--------|----------|
| EXE patching | Python + ctypes | C++ direct | P0 |
| HBD re-encoding | C++ via DLL | C++ direct | P0 |
| Disc assembly | Python file I/O | C++ direct | P1 |
| MIPS encoding | Hex string literals | Instruction encoder | P1 |
| EDC/ECC | subprocess edcre.exe | C++ implementation or keep subprocess | P2 |
| Verification | Python post-build | C++ per-step assertions | P1 |
| Profile management | Python dict | C++ enum/config | P2 |
| Telemetry | Python JSON | Python (stays) | P3 (no change) |
| Translation mgmt | Python JSON | Python (stays) | P3 (no change) |

### C++ MIPS Instruction Encoder

**Current approach** (fragile, error-prone):
```python
{'offset': 0x6112C, 'patch': struct.pack('<I', 0x24020001), 'desc': 'exit1_success'}
```

**Gold standard** (type-safe, verifiable):
```cpp
// In mips_encode.cpp
MipsInstr li_v0(int imm) {
    return MipsInstr::addiu(MipsReg::v0, MipsReg::zero, imm);
}

MipsInstr jr_ra() {
    return MipsInstr::jr(MipsReg::ra);
}

// In exe_patch.cpp
patcher.patch(0x6112C, {li_v0(1), jr_ra(), MipsInstr::nop()});
patcher.verify(0x6112C, {li_v0(1), jr_ra(), MipsInstr::nop()});
```

This eliminates an entire class of bugs: wrong opcodes, wrong register encodings,
wrong immediate values. Every patch is verified against the instruction encoder
before being written to the EXE.

---

## The Secondary Disc-Check: Deep Analysis

This is the true final boss. The HBD re-encoding (types 32/46) is a known
engineering problem with a known solution space. The disc-check loop is
a moving target that has persisted across 30+ build iterations.

### What We Know

1. **Primary disc check** (`0x80078FF4`): Patched. Game boots past it.
2. **Disc change handler** (`0x800506CC`): Neutralized (jr $ra). 8 call sites defused.
3. **Post-disc-change verify** (`0x8008AEF4`): 157 call sites. NOT patched.
   This function validates disc identity after any CD-ROM operation.
   It may be called after the first HBD read, after FMV load, or after
   any CD-ROM seek.
4. **Disc number variable** (`0x800E3D90`): We force it to 1 via RAM hook.
   But it may be re-read from the actual CD-ROM hardware (GetLocP/GetLocL
   commands read sector headers which contain the disc ID).
5. **FMV preload trigger**: The game reads LBA 146621 (DW7 FMV region)
   then enters an idle loop. This may be a disc validation triggered
   by the FMV load attempt.

### What We Suspect

The secondary disc check is NOT a single function we can patch. It's a
**distributed validation pattern** — the game checks disc identity at
multiple points using different mechanisms:

- **Mechanism 1**: Compare disc number variable (`0x800E3D90`) against expected
- **Mechanism 2**: Read CD-ROM sector headers (GetLocP) and compare disc ID
- **Mechanism 3**: Check CD-ROM status (Getstat) for shell-open bit
- **Mechanism 4**: Compare HBD filename string ("HBD1PS1D.W71" vs "HBD1PS1D.Q41")

Our 10-11 patches cover Mechanism 1 (partially) and Mechanism 3 (partially).
We do NOT cover Mechanism 2 (hardware-level disc ID) or Mechanism 4 (string compare).

### The Nuclear Option

If we cannot find and patch every disc check, we can **patch the CD-ROM BIOS
wrapper** at `0x80097814` (13 call sites) to intercept ALL CD-ROM commands
and return fabricated responses:

```mips
cdrom_wrapper_patched:
    # If command is GetLocP (0x0F):
    #   Return DQ4 disc ID bytes instead of actual disc ID
    # If command is Getstat (0x03):
    #   Return 0x02 (spinning, no shell open) always
    # If command is GetlocL (0x0E):
    #   Return expected sector header
    # Otherwise:
    #   Pass through to real CD-ROM handler
```

This is a **CD-ROM lie layer** — the game asks "what disc is this?" and we
always answer "it's the right disc." 13 call sites, one patch point.

**This is the most robust approach because it doesn't matter how many
validation points the game has — they all go through the same CD-ROM wrapper.**

### Fork Applicability

| Fork | Disc-check approach |
|------|-------------------|
| Alpha | Extended patches (15-20) + CD-ROM lie layer |
| Beta | Minimal (DQ4 EXE has its own disc ID — patch 1-2 points) |
| Gamma | None (custom engine, no disc check) |
| Delta | Extended patches (15-20) + CD-ROM lie layer |

---

## Build Profile Rationalization

### Kill List (Archive Immediately)

All profiles v12 through v43. They are historical artifacts. The lessons
are documented in `TEMPORAL_MAP_Jul18_2026.md` and study notes.

### Keep List

| Profile | Purpose | Status |
|---------|---------|--------|
| `test_0` | Baseline DW7 copy | Diagnostic |
| `test_a` | DQ4 HBD swap only | Diagnostic |
| `test_b` | DQ4 HBD + tree patches | Diagnostic |
| `test_d` | DW7 HBD + tree patches | Diagnostic |
| `test_f` | ISO dir update only | Diagnostic |
| `frank_v44` | Current best Frankenstein | Active |
| `rc2` | DQ4 native EXE | Fork Beta |
| `gold` | Final gold standard | Target (NEW) |

### New Profiles to Create

| Profile | Fork | Description |
|---------|------|-------------|
| `alpha_gold` | Alpha | v44 + fixed types 32/46 + extended disc-check |
| `beta_native` | Beta | DQ4 EXE on DW7 disc shell |
| `delta_shim` | Delta | DW7 EXE + runtime type 32/46 handlers |
| `gamma_engine` | Gamma | Custom engine EXE |
| `gold` | Converged | Final playable build (whichever fork wins) |

---

## Verification Protocol (Gold Standard)

Every build must pass ALL of these before being marked as playable:

### Tier 1: Binary Verification (Automated, < 1 second)
- [ ] EXE header valid (magic, load_addr, text_size, PC0)
- [ ] All disc-check patches applied (byte-for-byte match)
- [ ] Hybrid tree present at expected offset (if applicable)
- [ ] MIPS tree refs point to correct address (if applicable)
- [ ] BSS clear range correct (if applicable)
- [ ] LBA references updated (no residual 354s if shifted to 355)
- [ ] ISO directory entries valid (LBA + size match actual data)
- [ ] PVD volume size matches disc image size
- [ ] EDC/ECC regenerated (edcre.exe exit code 0)

### Tier 2: HBD Verification (Automated, < 10 seconds)
- [ ] All block headers in DW7 format (hts, treeEnd=0, textEnd=0) — if re-encoded
- [ ] No residual DQ4 types (32, 39, 40, 42, 46 in type_id field) — if re-encoded
- [ ] All per-block trees removed — if re-encoded
- [ ] Sub-block field swaps applied (flags=0, type_id=count)
- [ ] LBA relocation applied (internal sector refs adjusted)

### Tier 3: Boot Verification (Automated, 60 seconds)
- [ ] DuckStation recognizes disc (SLUS-01206 or SLPM-869.16)
- [ ] EXE loads to RAM at correct address
- [ ] BIOS POST completes
- [ ] CD-ROM init succeeds
- [ ] First HBD read succeeds (sector data non-zero)
- [ ] No "Invalid word read" exceptions in first 60 seconds
- [ ] VBlank count increments (game is running, not frozen)

### Tier 4: Gameplay Verification (Manual, 5 minutes)
- [ ] Enix logo displays
- [ ] FMV plays (or skips cleanly if patched)
- [ ] Title screen displays with English text
- [ ] "New Game" selectable
- [ ] Character creation completes
- [ ] No "Insert Disc 1" / "Wrong Disc" loop
- [ ] Game world loads
- [ ] Player can move character
- [ ] English dialogue displays correctly
- [ ] No crash in 5 minutes of gameplay

---

## Session Meta

- Date: Jul 22, 2026
- Author: GLM (Playing God mode)
- Active forks: Alpha (fix re-encoder) + Delta (runtime shim) — parallel
- Insurance fork: Beta (DQ4 native) — parallel, low effort
- Long-term fork: Gamma (cleanroom engine) — after playable build achieved
- C++ mandate: All binary surgery in C++. Python for telemetry/translation only.
- The lights in Frankenstein's eyes aren't rolling yet. But the body is built,
  the brain is in place, and we know exactly where the electrodes need to go.
