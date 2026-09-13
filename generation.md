           PLAYSTATION SYSTEM RAM (2 MB)
0x80000000 +---------------------------------------+| Kernel Vectors & System Core          |0x8000F800 +---------------------------------------+| Global Flag Space / Save Buffer       |0x80025300 +---------------------------------------+| Base Sound Engine (SEQq Tables)       |0x8002FA10 +---------------------------------------+| Sound Processor Code & Threads        |0x80037000 +---------------------------------------+| Core Executable (Main Engine Loop)    |0x80076040 +---------------------------------------+| CD-ROM Filesystem Routines            |0x80082480 +---------------------------------------+| Sound Streaming Program Code          |0x80090E54 +---------------------------------------+| PsyQ SPU Interface Libraries          |0x800B9D10 +---------------------------------------+| SPU Work RAM Parameter Storage        |0x800D1500 +---------------------------------------+| BGM Sequence Queue (Dynamic Buffer)   |0x800D7600 +---------------------------------------+| Waveform Instrument Attributes        |0x800F7798 +---------------------------------------+| Diagnostic Segment (Debug Switch)     |0x80138000 +---------------------------------------+| Dynamic Heap Segment (Map Geometry)   |0x80180000 +---------------------------------------+| SPU ADPCM Soundbank (Waveforms)       |0x801FFFFF +---------------------------------------+
---

**4. Script Virtual Machine & Dialogue Control Code Mapping**

The low-level, table-driven text engines of Super Famicom *Dragon Quest III* and *VI* were adapted into the PlayStation's Type 39 script sub-blocks.

| Functionality | Super Famicom (SFC) Control Syntax | PlayStation (PSX) Control Syntax | Execution Semantics & Blitter Behavior |
|---|---|---|---|
| **String Termination** | `$00` or `$00AE` | `{0000}` | Closes active dialogue stream; tears down window thread context. |
| **Line Break / Reset** | Standard table-mapped line break controls | `{7F02}` | Newline + Tab cursor reset to left window margin anchor. |
| **Name Decorator Prefix** | Hardcoded RAM index lookup triggers | `{7F04}` followed by actor ID (`7Fxx`) | Reads subsequent actor ID byte and dynamically renders nameplate. |
| **Blinking Cursor Pause** | Pause wait timers | `{7F0A}` (Wait for input) / `{7F0B}` (Reverse wait) | Halts parsing, blinks cursor, and waits for pad input or frame timeout. |
| **Dynamic Gold Variable** | Low-level accumulator decimal conversions | `{7F15}` | Dynamically inserts current party gold count into text stream. |
| **Conditional VM Command** | Hardcoded branching assembly scripts | `C021A0 <FFF0> <key>` | Evaluates global event flag matrix to alter script branching. |
| **Text Translation Wrap** | Custom DTE/MTE compression buffers | `@aName@b... @c0@ / @c1@` markup envelopes | Encapsulates localized string arrays and dynamic font markup envelopes. |

### Assembly Routine Comparison: Target HP Evaluation

```assembly
; ==============================================================================
; Super Famicom Assembly (65C816): Target HP Evaluation
; ==============================================================================
LDA $7e3925+HP_OFFSET, X  ; Load active character's current HP
CMP #$0014                ; Compare HP against critical threshold (20 HP)
BCS SkipCriticalHealing   ; If HP is above threshold, branch to standard actions
JSR CastCriticalHeal      ; Otherwise, prioritize immediate healing spell

; ==============================================================================
; PlayStation MIPS R3000 Disassembly: Target HP Evaluation
; ==============================================================================
lw    $v0, 0x10($a0)      ; Load active character's HP from status block in heap
li    $v1, 20             ; Load critical HP threshold value (20)
slt   $v1, $v0,$v1       ; Test if current HP is less than 20
beqz  $v1, SkipCriticalHeal ; If false, branch to standard combat logic
nop                       ; Branch delay slot padding
jal   CastHealSpell       ; Otherwise, jump and link to critical healing routine
5. Archive Structures & LZSS0 Decompression ProtocolsNon-executable assets reside in HBD1PS1D.Q41 (DQIV) or HBD1PS1D.Q71 (DQVII), aligned to 2,048-byte CD-ROM physical sectors.+-------------------------------------------------------------+
|                  HBD1PS1D Primary Sector Layout             |
+-------------------------------------------------------------+
| Offset 0x00 (4 bytes): Sub-block Count                      |
| Offset 0x04 (4 bytes): Sector Count (physical sectors)      |
| Offset 0x08 (4 bytes): Total Data Length (unpadded bytes)   |
| Offset 0x0C (4 bytes): Reserved / zero padding              |
+-------------------------------------------------------------+
| 0x10 + (i * 16) Sub-Block Headers:                          |
| - Offset 0x00 (4 bytes): Compressed Data Size on disc       |
| - Offset 0x04 (4 bytes): Uncompressed Data Size in RAM      |
| - Offset 0x08 (4 bytes): System Flag                        |
| - Offset 0x0C (2 bytes): Compression Flag (1280 = LZSS0)    |
| - Offset 0x0E (2 bytes): Resource Type Identifier           |
+-------------------------------------------------------------+
LZSS0 Bitstream Parsing Engine$$\text{Bit } b_i \text{ of control byte } C \implies \begin{cases} 1 & \text{Literal Byte Copy} \\ 0 & \text{Dictionary Reference (16-bit Descriptor } W) \end{cases}$$Compressed Input Stream:
+--------------+-----------------------+--------------------+
| Control Byte | 16-bit Descriptor (W) | Literal Byte (0xXX)|...
+--------------+-----------------------+--------------------+
  |
  +-- Bit b_i = 0 (Dictionary Reference Flag)
  |   - Offset (O) = W >> 4
  |   - Length (L) = (W & 0x0F) + 3
  |
  +-- Bit b_i = 1 (Literal Byte Copy Flag)
      - Read next raw byte directly to Output Buffer
      - Append raw byte to 4096-byte Sliding Ring Buffer

Sliding Ring Buffer (4096 Bytes, Initialized to 0x00):
+-----------------------------------------------------------+
| ... [Offset O] --------(Retrieve L Bytes)--------->|  |
+--------------------------------------------------------|--+
                                                         |
                                                         v
Destination Output RAM Buffer:             +-------------+------+
                                           |... | Copied L Bytes |
                                           +--------------------+
6. Decoupled Battle Background Pipeline & Dynamic Town StreamingDecoupling View from Logic in CombatSuper Famicom Combat Initialization:[Core Combat Loop] ---> [Load Direct Display Registers (Static VRAM Frame)]
PlayStation Double-Buffered Render Pipeline:[Core Combat Loop (Logic)] ---> [Update Combat State Registers]
                                          |
                                          v
[3D Render Thread]       <--- [Perspective Projection via GTE]
                                          |
                                          v
                              [Push Primitives to GPU Ordering Table]
Dynamic Town Streaming Pipeline (Immigrant Town)Boundary Collision Trigger ---> Suspend World Map Renderer ---> Query Progress Flag: 0x8000F800
                                                                                |
                                                                                v
mopen_file (0x80082598) Dynamic Stream <-- Retrieve Sector Addresses <-- Offset Modifier Lookup
|
+--- 3D Mesh qQES   -> RAM Heap 0x80138000
+--- NPC Sheets TIM -> VRAM Page Cache (X >= 512)
By separating game logic from presentation pipelines, preserving 16-bit database indices within a 32-bit architecture, and streaming sector-aligned assets via low-overhead LZSS decompression, HeartBeat bridged two console eras while maintaining complete gameplay fidelity."""filename = "STUDY_MIGRATION_HBE_SFC_TO_PSX_Sep13_2026.md"with open(filename, "w", encoding="utf-8") as f:f.write(content)print(f"File written successfully: {filename}")
```text?code_stdout&code_event_index=1
File written successfully: STUDY_MIGRATION_HBE_SFC_TO_PSX_Sep13_2026.md

Your Markdown document is ready: STUDY_MIGRATION_HBE_SFC_TO_PSX_Sep13_2026  MD OpenMarkdown# Architectural Migration, Binary Reconstruction, and Generational Analysis of the HeartBeat Engine in Dragon Quest IV and VII

**Document ID:** ARCH-STUDY-MIGRATION-HBE-SFC-TO-PSX-20260913-V2  
**Classification:** Deep-Dive Cross-Generational Engine Architecture & Systems Forensics  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Architectures:** Super Famicom (Ricoh 5A22 / 65C816) & Sony PlayStation (MIPS R3000A)  
**Primary Binaries:** `SLPM_869.16` (DQIV PSX), `SLPM_865.00` / `SLUS_012.06` (DQVII PSX)  
**Primary Archives:** `HBD1PS1D.Q41` & `HBD1PS1D.Q71`  
**Date:** September 13, 2026  

---

**1. Architectural Divergence: Solid-State Bus vs. Optical Streaming**

The migration of HeartBeat's engine from a fixed-bank ROM cartridge architecture on the Super Famicom to a disc-based dynamic system on the Sony PlayStation required restructuring memory boundaries, registration widths, and asset-loading paradigms under the PlayStation's 2 MB system RAM constraints.

| Architecture / Subsystem Metric | Super Famicom (SFC) Implementation | PlayStation (PSX) Implementation | Subsystem Divergence & Impact |
|---|---|---|---|
| **CPU / Hardware Platform** | 16-bit Ricoh 5A22 (65C816 derivative) @ 3.58 MHz | 32-bit MIPS R3000 CPU with GTE and MDEC @ 33.8688 MHz | Transitions from 8/16-bit accumulator math to 32-bit RISC pipeline and fixed-point vector coprocessing. |
| **Primary Media / Interface** | Solid-state cartridge (Direct Bus Mapping via HiROM/LoROM) | CD-ROM (ISO 9660 Mode 2 Form 1 File System Streaming) | Instantaneous single-cycle memory reads replaced by rotational disc seek latencies and sector buffering. |
| **Memory Management** | Bank switching (HiROM/LoROM, $7E Work RAM) | Static partitioning with dynamic heap allocation (2 MB RAM) | Moving from tightly packed 60-byte structs to partitioned resident engines and dynamic allocation heaps. |
| **State Tracking Paradigm** | High-efficiency 16-bit/8-bit register bit manipulation | 32-bit structures, flag matrices (`0x8000F800`), and dynamic VM variables | Preserves Boolean flag density while adapting dispatch mechanics to 32-bit word alignments. |
| **Asset Location & Access** | Instantaneous access at fixed ROM offsets | Dynamic sector-based buffering via CD-ROM DMA Channel 4 | Replaces synchronous frame-bound transfers with asynchronous, multi-frame sector streaming loops. |
| **Execution Loop Boundary** | Interfaced via direct CPU interrupts and hardware timers | VSync callback registration and thread-driven scheduling | Replaces hardware interrupt polling with a software event loop coordinating input, scripts, and geometry. |

---

**2. The Hidden Commonality: Database Schema Parity & Game Loop Preservation**

Despite the transition from ROM cartridge banks to optical ISO disc files, the underlying data schemas and progression logic remained mirrored across generations.

### Core Database Schema Layout

| Data Schema Type | Record Count | Structural Array Size | Target System Integration & Parameter Maps |
|---|---|---|---|
| **Monsters** | 50 records | 8 bytes per record | Base HP, EXP yield, Gold reward, ATK, DEF, and AGI parameter maps. |
| **Items** | 128 records | 2 bytes per record | Purchase price, equipment eligibility bitmasks, and item type tables. |
| **Shops** | 180 records | 2 bytes per record | Merchant inventory catalogs and price index configurations. |
| **Spells** | 50 records | 6 bytes per record | MP cost, element type bitmasks, and target-range parameters. |
| **Encounters** | 7,005 entries | 6 bytes per record | Map sector-based monster group indexing and encounter probability rates. |
| **Characters** | 16 records | 2 bytes per record | Base statistics, growth curve tables, and class definition vectors. |

### Scripted Events and 2D Grid Preservation

While the PlayStation engine renders environments using a rotatable 3D isometric camera (90-degree increments), collision detection, event activation vectors, and NPC walking paths are evaluated on the original flat 2D tile grid. The engine's renderer translates these 2D logical positions into 3D camera-space coordinates, completely decoupling game logic from the 3D presentation layer.

---

**3. System Memory Map: PSX 2 MB Partitioning Topology**

To support hybrid 2D/3D presentation without memory fragmentation, HeartBeat partitioned the PlayStation's physical memory space into dedicated resident code blocks and dynamic streaming heaps.

              PLAYSTATION SYSTEM RAM (2 MB)
0x80000000 +---------------------------------------+| Kernel Vectors & System Core          |0x8000F800 +---------------------------------------+| Global Flag Space / Save Buffer       |0x80025300 +---------------------------------------+| Base Sound Engine (SEQq Tables)       |0x8002FA10 +---------------------------------------+| Sound Processor Code & Threads        |0x80037000 +---------------------------------------+| Core Executable (Main Engine Loop)    |0x80076040 +---------------------------------------+| CD-ROM Filesystem Routines            |0x80082480 +---------------------------------------+| Sound Streaming Program Code          |0x80090E54 +---------------------------------------+| PsyQ SPU Interface Libraries          |0x800B9D10 +---------------------------------------+| SPU Work RAM Parameter Storage        |0x800D1500 +---------------------------------------+| BGM Sequence Queue (Dynamic Buffer)   |0x800D7600 +---------------------------------------+| Waveform Instrument Attributes        |0x800F7798 +---------------------------------------+| Diagnostic Segment (Debug Switch)     |0x80138000 +---------------------------------------+| Dynamic Heap Segment (Map Geometry)   |0x80180000 +---------------------------------------+| SPU ADPCM Soundbank (Waveforms)       |0x801FFFFF +---------------------------------------+
---

**4. Script Virtual Machine & Dialogue Control Code Mapping**

The low-level, table-driven text engines of Super Famicom *Dragon Quest III* and *VI* were adapted into the PlayStation's Type 39 script sub-blocks.

| Functionality | Super Famicom (SFC) Control Syntax | PlayStation (PSX) Control Syntax | Execution Semantics & Blitter Behavior |
|---|---|---|---|
| **String Termination** | `$00` or `$00AE` | `{0000}` | Closes active dialogue stream; tears down window thread context. |
| **Line Break / Reset** | Standard table-mapped line break controls | `{7F02}` | Newline + Tab cursor reset to left window margin anchor. |
| **Name Decorator Prefix** | Hardcoded RAM index lookup triggers | `{7F04}` followed by actor ID (`7Fxx`) | Reads subsequent actor ID byte and dynamically renders nameplate. |
| **Blinking Cursor Pause** | Pause wait timers | `{7F0A}` (Wait for input) / `{7F0B}` (Reverse wait) | Halts parsing, blinks cursor, and waits for pad input or frame timeout. |
| **Dynamic Gold Variable** | Low-level accumulator decimal conversions | `{7F15}` | Dynamically inserts current party gold count into text stream. |
| **Conditional VM Command** | Hardcoded branching assembly scripts | `C021A0 <FFF0> <key>` | Evaluates global event flag matrix to alter script branching. |
| **Text Translation Wrap** | Custom DTE/MTE compression buffers | `@aName@b... @c0@ / @c1@` markup envelopes | Encapsulates localized string arrays and dynamic font markup envelopes. |

### Assembly Routine Comparison: Target HP Evaluation

```assembly
; ==============================================================================
; Super Famicom Assembly (65C816): Target HP Evaluation
; ==============================================================================
LDA $7e3925+HP_OFFSET, X  ; Load active character's current HP
CMP #$0014                ; Compare HP against critical threshold (20 HP)
BCS SkipCriticalHealing   ; If HP is above threshold, branch to standard actions
JSR CastCriticalHeal      ; Otherwise, prioritize immediate healing spell

; ==============================================================================
; PlayStation MIPS R3000 Disassembly: Target HP Evaluation
; ==============================================================================
lw    $v0, 0x10($a0)      ; Load active character's HP from status block in heap
li    $v1, 20             ; Load critical HP threshold value (20)
slt   $v1, $v0,$v1       ; Test if current HP is less than 20
beqz  $v1, SkipCriticalHeal ; If false, branch to standard combat logic
nop                       ; Branch delay slot padding
jal   CastHealSpell       ; Otherwise, jump and link to critical healing routine
5. Archive Structures & LZSS0 Decompression ProtocolsNon-executable assets reside in HBD1PS1D.Q41 (DQIV) or HBD1PS1D.Q71 (DQVII), aligned to 2,048-byte CD-ROM physical sectors.+-------------------------------------------------------------+
|                  HBD1PS1D Primary Sector Layout             |
+-------------------------------------------------------------+
| Offset 0x00 (4 bytes): Sub-block Count                      |
| Offset 0x04 (4 bytes): Sector Count (physical sectors)      |
| Offset 0x08 (4 bytes): Total Data Length (unpadded bytes)   |
| Offset 0x0C (4 bytes): Reserved / zero padding              |
+-------------------------------------------------------------+
| 0x10 + (i * 16) Sub-Block Headers:                          |
| - Offset 0x00 (4 bytes): Compressed Data Size on disc       |
| - Offset 0x04 (4 bytes): Uncompressed Data Size in RAM      |
| - Offset 0x08 (4 bytes): System Flag                        |
| - Offset 0x0C (2 bytes): Compression Flag (1280 = LZSS0)    |
| - Offset 0x0E (2 bytes): Resource Type Identifier           |
+-------------------------------------------------------------+
LZSS0 Bitstream Parsing Engine$$\text{Bit } b_i \text{ of control byte } C \implies \begin{cases} 1 & \text{Literal Byte Copy} \\ 0 & \text{Dictionary Reference (16-bit Descriptor } W) \end{cases}$$Compressed Input Stream:
+--------------+-----------------------+--------------------+
| Control Byte | 16-bit Descriptor (W) | Literal Byte (0xXX)|...
+--------------+-----------------------+--------------------+
  |
  +-- Bit b_i = 0 (Dictionary Reference Flag)
  |   - Offset (O) = W >> 4
  |   - Length (L) = (W & 0x0F) + 3
  |
  +-- Bit b_i = 1 (Literal Byte Copy Flag)
      - Read next raw byte directly to Output Buffer
      - Append raw byte to 4096-byte Sliding Ring Buffer

Sliding Ring Buffer (4096 Bytes, Initialized to 0x00):
+-----------------------------------------------------------+
| ... [Offset O] --------(Retrieve L Bytes)--------->|  |
+--------------------------------------------------------|--+
                                                         |
                                                         v
Destination Output RAM Buffer:             +-------------+------+
                                           |... | Copied L Bytes |
                                           +--------------------+
6. Decoupled Battle Background Pipeline & Dynamic Town StreamingDecoupling View from Logic in CombatSuper Famicom Combat Initialization:[Core Combat Loop] ---> [Load Direct Display Registers (Static VRAM Frame)]
PlayStation Double-Buffered Render Pipeline:[Core Combat Loop (Logic)] ---> [Update Combat State Registers]
                                          |
                                          v
[3D Render Thread]       <--- [Perspective Projection via GTE]
                                          |
                                          v
                              [Push Primitives to GPU Ordering Table]
Dynamic Town Streaming Pipeline (Immigrant Town)Boundary Collision Trigger ---> Suspend World Map Renderer ---> Query Progress Flag: 0x8000F800
                                                                                |
                                                                                v
mopen_file (0x80082598) Dynamic Stream <-- Retrieve Sector Addresses <-- Offset Modifier Lookup
|
+--- 3D Mesh qQES   -> RAM Heap 0x80138000
+--- NPC Sheets TIM -> VRAM Page Cache (X >= 512)
By separating game logic from presentation pipelines, preserving 16-bit database indices within a 32-bit architecture, and streaming sector-aligned assets via low-overhead LZSS decompression, HeartBeat bridged two console eras while maintaining complete gameplay fidelity.
