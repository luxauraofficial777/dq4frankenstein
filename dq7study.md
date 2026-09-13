# Reverse Engineering the Monolithic Engine of Dragon Warrior VII: Memory Partitioning, Event Flag Matrices, and Asset Compression Protocols

**Document ID:** ARCH-STUDY-PSX-DW7-ENGINE-20260913-V2  
**Classification:** Deep-Dive Engine, Memory Architecture & Archive Protocol Audit  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Executables:** Sony PlayStation `SLUS_012.06` (North America) / `SLPM_865.00` (Japan)  
**Archive Target:** `HBD1PS1D.Q71` (CD-ROM Mode 2 Form 1 Resource Archive)  
**Date:** September 13, 2026  

---

## 1. Memory Partitioning & Heap Allocation Mapping

The execution of a massive, multi-disc role-playing game on the fifth-generation PlayStation platform forced developer Heart Beat to bypass standard operating system APIs in favor of a custom, highly optimized system architecture. With only 2 MB (2,048 KB) of system RAM available, the main executable `SLUS_012.06` (and its Japanese counterpart `SLPM_865.00`) maps out a strict partitioning layout. This layout separates the static, resident "base engine" from volatile "expansion content" and high-bitrate data streamed dynamically from the optical disc.

The lower bounds of system memory host the resident base engine, which includes core MIPS R3000 CPU instructions, geometry pipeline interfaces, peripheral input maps, and the main gameplay execution loop. Immediately above these system bounds lies the game state tracking matrix, followed by dedicated sound processing software. Because the standard Sony PsyQ sound library was too generic and resource-intensive, Heart Beat implemented a custom sound driver, designated `SEQq`, which utilizes the signature `qQES`. This custom driver bypasses the standard `SEQ`/`VAB` protocols to optimize the utilization of both system RAM and Sound RAM (SPU WRAM).

### PlayStation System Memory Map Layout

The partition boundaries mapped within the PlayStation's 2 MB physical memory space are structured as follows:

| Physical Address Range | Segment Classification | Core Engine Function / Subsystem |
|---|---|---|
| `0x80000000` – `0x8000F7FF` | Kernel & System Core | PlayStation operating system kernel vectors, PsyQ base configurations. |
| `0x8000F800` – `0x800252FF` | Global Flag Space | Active progress registers, character attributes, event triggers, save state buffer. |
| `0x80025300` – `0x800258E8` | Base Sound Engine | Structural interpretation tables for the custom `SEQq` music driver. |
| `0x8002FA10` – `0x80036FCC` | Base Sound Engine | Execution thread code (interrupt-driven sound processor). |
| `0x80037000` – `0x8007603F` | Core Executable | Main game loop, rendering pipelines, dialogue engines, camera systems. |
| `0x80076040` – `0x8008247F` | Disk File System | Core optical disc filesystem routines, logical block addressing (`open_file`). |
| `0x80082480` – `0x800849D4` | Disk File System | Sound streaming program code, low-level `mopen_file` file pointer tracker. |
| `0x80090E54` – `0x80094D14` | Base Sound Engine | PsyQ SPU interface library calls and low-level voice assignment code. |
| `0x800B9D10` – `0x800BB6E7` | SPU Work RAM | SPU library parameter storage, hardware voice registers. |
| `0x800D25C0` – `0x800D80EF` | Volatile Sequence Heap | BGM Sequence streaming buffer (maximum size allocated: 23,344 bytes). |
| `0x800D80F0` – `0x800DA0F7` | Volatile Sequence Heap | Instrument attribute maps, panning indices, envelope registers. |
| `0x800F7798` – `0x800F7799` | Diagnostic Segment | Master debug mode registration address (16-bit switch). |
| `0x80138000` – `0x8017FFFF` | Dynamic Heap Segment | Executing sequence (`SEQq`) play buffer, active script parameters. |
| `0x80180000` – `0x801FFFFF` | Dynamic Heap Segment | SPU ADPCM soundbank (`VAB`) sample buffer (Waveforms). |

### Custom SEQq Sound Format Header Layout

The custom `SEQq` sound format starts with a 60-byte header, which maps the layout of incoming audio waveforms and sequence pointers directly into the dynamic heaps. Unlike the standard PsyQ sound format, sequence data parsed by the `SEQq` driver terminates abruptly with the byte sequence `FF 2F 00` without any trailing delta-time parameters, saving memory overhead in the dynamic heap:

| Byte Offset | Size (Bytes) | Field Name | Technical Description |
|---|---|---|---|
| `0x00` | 4 | `SEQ Size` | Size of the music sequence payload in bytes (evaluates to 0 if absent). |
| `0x04` | 2 | `SEQ ID` | Micro-sequence identifier (`msq_id`). |
| `0x06` | 1 | `Bank Count` | Total number of ADPCM wave banks allocated (typically 4). |
| `0x07` | 1 | `Load Position` | Memory position indicator (`0` = Normal BGM, loads to static address). |
| `0x08` | 4 | `Reserved` | Reserved space (unutilized by the driver). |
| `0x0C` | 4 | `Bank 1 VB Size` | Compressed ADPCM waveform size (0 if preloaded bank is active). |
| `0x10` | 4 | `Bank 1 Region Size` | Size of the regional mapping tables for Bank 1. |
| `0x14` | 2 | `Bank 1 ID` | Instrument index descriptor for voice mapping. |
| `0x16` | 2 | `Bank 1 Attribute` | Voice panning, pitch modulation, and resonance parameters. |
| `0x18` – `0x23` | 12 | `Bank 2 Header` | Configuration block for Bank 2 (identical structure to Bank 1). |
| `0x24` – `0x2F` | 12 | `Bank 3 Header` | Configuration block for Bank 3 (identical structure to Bank 1). |
| `0x30` – `0x3B` | 12 | `Bank 4 Header` | Configuration block for Bank 4 (identical structure to Bank 1). |

---

## 2. Dynamic Map Block Loading & Streaming Architecture

Movement between the world map and individual town zones requires real-time loading routines to prevent memory overflows. The central function that coordinates this operation is located at address `0x80076040` (`open_file`). This routine acts as an interface to the PlayStation's CD-ROM controller. When the player transitions between map sectors, `open_file` invokes a secondary, non-blocking sub-loader at address `0x80082598` (`mopen_file`).

The `mopen_file` routine operates as follows:
1. It accepts a single parameter passed through CPU register `$a0`, which points to a file descriptor struct containing the target Logical Block Number (LBN), sector length, and the destination address in the dynamic heap.
2. Rather than attempting to copy large data blobs in a single processing frame—which would stall the main game loop—the function reads the file from the disc incrementally, sector-by-sector, in 2,048-byte chunks.
3. For map instances, the sub-loader detects the resource type as `0x13`, prompting the engine to partition the dynamic loading process across multiple frames.
4. The system leverages DMA Channel 4 (the CD-ROM direct memory access pathway) to stream sectors directly into the dynamic map geometry heap starting at `0x80138000`, bypassing main CPU cycles.

### Framebuffer Snapshot & Upscaling Artifact Analysis

When the game menu is opened, the engine takes a snapshot of the current frame buffer from VRAM and copies it to a dedicated background buffer in System RAM. Because of strict memory constraints, this copy is captured at the game's native internal rendering resolution (typically $256 \times 224$ or $320 \times 240$ pixels). On original hardware, this transfer is seamless. However, on modern emulators running at upscaled resolutions, this process creates a noticeable rendering discrepancy: the high-resolution menu elements are overlaid on top of a low-resolution, blocky background snapshot, resulting in a distinct "blurry" menu effect.

---

## 3. Event Trigger & Flag Matrix Architecture

To maintain a persistent, non-linear narrative across dozens of islands existing in distinct past and present time periods, the Heart Beat engine utilizes a compact, highly dense game state tracking system managed by an event flag matrix residing in the system's static memory bounds.

### Tracing the Progression Flags in RAM

Live memory tracing on a PlayStation emulator locates the core game progress flags within the RAM range `0x8000F800` through `0x80010000`. This space acts as a bit-packed matrix that records every major progression state:
* Active system flags are modified by the engine's state-writing routines (e.g., "Save Anywhere" debug codes write directly to ranges `0x8000F83C` through `0x8000F846`).
* Equipment rules and character restrictions are managed by related addresses in this block; for instance, the "Equip Any Armor" bypass writes directly to static address `0x80069BCA` to override standard inventory validation checks.

To trace these state changes in a debugging environment (such as No$psx), developers can set a write breakpoint directly on the target flag address using `[0x8000F83C]!`. When an event occurs—such as solving a puzzle in the Estard Ruins or recruiting an immigrant—the engine executes a store byte instruction:

```assembly
sb v1, $0(a1)    ; Store updated state byte to the flag pointer held in$a1
  To test flag bypasses or manipulate state transitions, the conditional branch instruction that evaluates these flags:Code snippetbeq v0, a0, $8001129C    ; Conditional branch evaluating flag criteria
  can be patched in memory with a No-Operation (nop) instruction to prevent the jump, or replaced with an unconditional jump (j $8001129C) to force the game state to evaluate as true regardless of actual flag values.  Immigrant Town Flag-Driven Progression StatesThe engine uses this progress flag matrix to dynamically stream 3D models and construct towns in real-time. The most prominent example is the Immigrant Town, which dynamically alters its physical layout, loaded buildings, and active NPCs based on the current population flag[cite: 2]:  Active Population Flag ValueStructural PhaseLoaded 3D Model Assets and NPC Configurations$1 - 4$ Immigrants[cite: 2]Phase 1[cite: 2]Campsite terrain mesh, base tents, Sim NPC model[cite: 2].$5 - 9$ Immigrants[cite: 2]Phase 2[cite: 2]Small wooden shack model, dynamic naming system initialized[cite: 2].$10 - 14$ Immigrants[cite: 2]Phase 3[cite: 2]Multi-room building models, raw construction frame meshes[cite: 2].$15 - 19$ Immigrants[cite: 2]Phase 4[cite: 2]Basic storefront textures, initial merchant NPC assets[cite: 2].$20 - 24$ Immigrants[cite: 2]Phase 5[cite: 2]Expanded Inn model, town management interface subroutine active[cite: 2].$25 - 29$ Immigrants[cite: 2]Phase 6[cite: 2]Stone church model, secondary shop models, religious NPC assets[cite: 2].$30 - 34$ Immigrants[cite: 2]Phase 7[cite: 2]Bank vault structures, high-tier armor and weapon merchant models[cite: 2].$35 - 40$ Immigrants[cite: 2]Phase 8[cite: 2]Final town layout (varies dynamically based on immigrant classes recruited)[cite: 2].This progression system operates through a specialized flag-to-asset pointer translation function[cite: 2]:Upon boundary collision detection (transition from world map to town instance), the engine suspends the world map renderer[cite: 2].The asset selector function queries 0x8000F800 to retrieve current state values[cite: 2].The returned state byte acts as an offset modifier for asset lookup tables in SLUS_012.06 (e.g., population flag = 22 selects Phase 5)[cite: 2].Sector addresses for the target Inn models, textures, and NPC scripts are extracted from HBD1PS1D.Q71[cite: 2].These addresses are passed to mopen_file (0x80082598), streaming selected assets into the dynamic heap at 0x80138000[cite: 2]. Assets from other phases are never loaded, preserving the 2 MB RAM limit[cite: 2].4. Archive Layout & Compression ProtocolsThe entire asset library of Dragon Warrior VII is contained within the monolithic resource archive HBD1PS1D.Q71, spanning several hundred megabytes and divided into strict 2,048-byte blocks aligning with the CD-ROM physical sectors[cite: 2].Archive Structural LayoutHeader Block: The first 2,048-byte sector (LBA 0) functions as the primary volume descriptor[cite: 2]. The ASCII string hbd1ps1d.q71 is located at byte offset 0x400 (1024 bytes)[cite: 2].Primary Data Blocks (* 00 00 00 Blocks): Index directories for grouped assets[cite: 2]. Each begins with a 16-byte header[cite: 2]:Offset 0x00 (4 bytes): Sub-block Count (capped at ~18 sub-blocks)[cite: 2].Offset 0x04 (4 bytes): Sector Count (total 2,048-byte sectors occupied by block)[cite: 2].Offset 0x08 (4 bytes): Total Data Length (unpadded byte length of concatenated sub-blocks)[cite: 2].Offset 0x0C (4 bytes): Reserved (null dword 0x00000000)[cite: 2].Asset Sub-blocks: Defined by 16-byte sub-headers at offset 0x10 + (i * 16) within the primary block[cite: 2]:Offset 0x00 (4 bytes): Compressed Data Size on physical disc[cite: 2].Offset 0x04 (4 bytes): Uncompressed Data Size in system RAM[cite: 2].Offset 0x08 (4 bytes): System flag[cite: 2].Offset 0x0C (2 bytes): Compression Flag (1280 / 0x0500 = LZSS0; other values = uncompressed)[cite: 2].Offset 0x0E (2 bytes): Resource Type Identifier (fonts, menus, 3D meshes, tilemaps)[cite: 2].Control Blocks (0x60010108 Blocks)Interspersed 2,048-byte control blocks interface with the CD-ROM drive controller, identified by magic signature 0x60010108[cite: 2]:Header OffsetSize (Bytes)Field Value / MappingTechnical Description0x00[cite: 2]4[cite: 2]0x60010108[cite: 2]Control block magic signature[cite: 2].0x04[cite: 2]2[cite: 2]0x0000 – 0x0004[cite: 2]Block index register (increments sequentially per block)[cite: 2].0x06[cite: 2]2[cite: 2]0x0005[cite: 2]Count constant (always 5)[cite: 2].0x08[cite: 2]4[cite: 2]Variable Counter[cite: 2]Part counter (increments by 1 each time block index resets)[cite: 2].0x0C[cite: 2]4[cite: 2]Integer Value[cite: 2]Internal system tracking integer[cite: 2].0x10[cite: 2]2[cite: 2]0x8000[cite: 2]SPU interface configuration constant (128)[cite: 2].0x12[cite: 2]2[cite: 2]0x7800[cite: 2]Audio stream mapping constant (120)[cite: 2].0x14[cite: 2]4[cite: 2]0xXXXXXX38[cite: 2]Hardware register offset (fourth byte locked to 0x38)[cite: 2].0x18[cite: 2]2[cite: 2]0x0100 / 0x0200 / 0x0300[cite: 2]System execution mode flag[cite: 2].0x1A[cite: 2]4[cite: 2]0x03000000[cite: 2]Main interface driver constant[cite: 2].0x1E[cite: 2]2[cite: 2]0x0000[cite: 2]Static null padding[cite: 2].5. LZSS0 Decompression Protocol & Isometric Tile AssemblyWhen offset 0x0C reads 1280, the engine passes the payload to the software decompressor, which initializes a 4,096-byte sliding dictionary ring buffer pre-filled with zero (0x00) bytes[cite: 2].The input stream is read as a sequence of control blocks starting with 1-byte control word $C$, evaluated bit-by-bit from LSB ($b_0$) to MSB ($b_7$)[cite: 2]:$$\text{Bit } b_i \text{ of control byte } C \implies \begin{cases} 1 & \text{Literal Byte Copy} \\ 0 & \text{Dictionary Reference (16-bit Descriptor } W) \end{cases}$$[cite: 2]Literal Byte Copy ($b_i = 1$): Reads the next raw byte directly, writes it to RAM, and appends it to the history window[cite: 2]:
$$\text{Output} \leftarrow \text{Input}[I], \quad D \leftarrow D + 1, \quad I \leftarrow I + 1$$[cite: 2]Dictionary Reference ($b_i = 0$): Reads a 16-bit descriptor word $W$[cite: 2]:
$$W \leftarrow (\text{Input}[I] \ll 8) \mid \text{Input}[I+1], \quad I \leftarrow I + 2$$[cite: 2]
Unpacks ring-buffer offset $O$ and match length $L$[cite: 2]:
$$O \leftarrow W \gg 4$$[cite: 2]$$L \leftarrow (W \ \& \ \text{0x0F}) + 3$$[cite: 2]
Copies $L$ bytes from the history buffer at offset $O$ into destination RAM, updating the dictionary ring buffer[cite: 2].Input Stream:
+---+---+---+---+---+---+
| C |       W       |...|   (C = Control Byte, W = 16-bit Descriptor)
+---+---+---+---+---+---+
  |
  +--> b_i = 0 (Reference Flag)
         |
         +--> Decode W:
                Offset (O) = W >> 4
                Length (L) = (W & 0x0F) + 3
                |
                v
Sliding Ring Buffer (Initialized to 0x00, Size = 4096 Bytes):
+---------------------------------------+
|                [O]----->|             |
+--------------------+------------------+
                     |
                     +--> Append to Buffer End & Output
                     v
Destination Output RAM Buffer:
+--------------------+---+---+---+
|                    |   |   |   |
+--------------------+---+---+---+
[cite: 2]LZSS0 requires minimal CPU cycles for decompression, relying on simple bitwise shifts and memory copies[cite: 2]. This allows real-time asset decompression without causing frame-rate stutter or disrupting interrupt-driven sound threads[cite: 2].Isometric Dungeon Tile-Set Assembly Sequenceopen_file locates target dungeon blocks inside HBD1PS1D.Q71[cite: 2].Two sub-blocks load: compressed $16 \times 16$ texture tile-sets and the grid-based isometric layout map[cite: 2].LZSS0 inflates texture data into a linear 4-bit or 8-bit indexed bitmap array referencing local palette tables[cite: 2].The layout sub-block decompresses into a 2D grid of 16-bit cell descriptors[cite: 2]:Bits 0–9: Tile Index (points to decompressed $16 \times 16$ texture array)[cite: 2].Bit 10: Horizontal Texture Flip Flag[cite: 2].Bit 11: Vertical Texture Flip Flag[cite: 2].Bits 12–15: Elevation and Height Parameters (isometric layer coordinate)[cite: 2].Tiles with active elevation parameters have their vertical draw coordinates offset downward by 4 to 6 pixels, aligning characters and objects cleanly with floor tiles[cite: 2].6. Technical Synthesis of Core Systems+---------------------------------------------------------------------------------+
|                               HBD1PS1D.Q71 (CD-ROM Archive)                     |
|  +-------------------+  +-------------------------+  +-----------------------+  |
|  |  Primary Header   |  |   * 00 00 00 Data Block  |  |  0x60010108 Control   |  |
|  | (Volume Desc. LBN)|  | (Asset Pointers & Flags)|  | (CD Controller Sync)  |  |
|  +---------+---------+  +------------+------------+  +-----------------------+  |
+------------|-------------------------|------------------------------------------+
             |                         |
             v                         v
+---------------------------------------------------------------------------------+
|                             PLAYSTATION SYSTEM RAM                              |
|  +---------------------------------------------------------------------------+  |
|  |  Base Engine Segment (0x80000000 - 0x800849D4 Static Bounds)               |  |
|  |  +------------------------+  +-------------------+  +------------------+  |  |
|  |  | Gameplay Loop & Render |  | open_file Routine |  | SEQq Sound prog. |  |  |
|  |  |   (Geometry Pipeline)  |  |   (0x80076040)    |  |   (0x8002FA10)   |  |  |
|  |  +------------------------+  +---------+---------+  +--------+---------+  |  |
|  +----------------------------------------|---------------------|------------+  |
|                                           | [mopen_file]        |               |
|                                           v (0x80082598)        |               |
|  +--------------------------------------------------+           |               |
|  |  Progress Flag Matrix (0x8000F800 - 0x80010000)  |           |               |
|  |  - Tracks global event flags & population phases |           |               |
|  +------------------------+-------------------------+           |               |
|                           |                                     |               |
|                           v                                     v               |
|  +---------------------------------------------------------------------------+  |
|  |  Dynamic Heap Segment (0x80138000 - 0x801FFFFF Volatile Bounds)           |  |
|  |  +----------------------------------+  +-------------------------------+  |  |
|  |  |    Active Map & Geometry Space   |  | SPU Audio Waveform Buffer     |  |  |
|  |  |  (LZSS0 Decompressed Tile-Sets)  |  | (Dynamic ADPCM Instruments)   |  |  |
|  |  +----------------------------------+  +-------------------------------+  |  |
|  +---------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------+
[cite: 2]Sector-Aligned Physical Blocks: File operations maximize CD-ROM drive throughput and minimize seek latency[cite: 2].Non-Blocking Asynchronous Streaming: Controlled by mopen_file, allowing dynamic map streaming while frame loops and audio playback run smoothly[cite: 2].Bit-Packed Progress Matrix: Consolidated progression tracking minimizes RAM consumption while driving real-time 3D model asset selection[cite: 2].Low-Overhead LZSS0 Codecs: Decompresses data on-the-fly without requiring dedicated hardware coprocessors or causing frame drops[cite: 2].
