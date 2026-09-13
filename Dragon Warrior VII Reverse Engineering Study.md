[ Transition / Sector Boundary Collision ]
|
v
open_file (0x80076040)
- Resolves Target File Descriptor via LBN
|
v
mopen_file (0x80082598)
- Passed descriptor struct pointer via register $a0
- Splits transfer into sequential 2,048-byte Mode 2 Form 1 sector reads
|
v
[ DMA Channel 4 (CD-ROM Direct Memory Access) ]
- Direct streaming into Dynamic Heap (0x80138000)
- Bypasses Main MIPS CPU cycles across multi-frame loading intervals


### The Upscaling Menu Blur Discrepancy

When an in-game menu is invoked, the engine captures a snapshot of the active VRAM framebuffer and transfers it to an offscreen buffer in System RAM to serve as a static backdrop. To conserve memory bandwidth, this copy is captured strictly at the native internal rendering resolution ($256 \times 224$ or $320 \times 240$). On native CRT displays, this handoff is imperceptible; under modern hardware-accelerated emulation with upscaled 3D geometry, high-resolution 2D UI elements are overlaid on top of this low-resolution snapshot, producing a prominent pixelated/blurred background artifact.

---

**4. Event Flag Matrix & Dynamic World Streaming**

Global progression across multiple historical eras is tracked in a bit-packed matrix located at `0x8000F800` – `0x80010000`. Progress updates are committed via MIPS store-byte instructions (`sb v1, $0(a1)`), where `$a1` holds the destination flag pointer. Conditional checks (`beq v0, a0, Target`) govern script jumps, gate unsealing, and world geometry queries.

### Immigrant Town Flag-Driven Progression States

The engine dynamically loads town geometry by evaluating the population counter flag stored in the matrix against lookup tables in `SLUS_012.06`, ensuring only current-phase assets occupy the volatile dynamic heap at `0x80138000`.

| Active Population Flag | Structural Phase | Streamed 3D Model Assets & Resident NPC Configurations |
|---|---|---|
| **1 – 4 Immigrants** | Phase 1 | Base campsite terrain mesh, canvas tents, Sim NPC base model. |
| **5 – 9 Immigrants** | Phase 2 | Small wooden shack model, dynamic town naming interface initialized. |
| **10 – 14 Immigrants** | Phase 3 | Multi-room building models, timber construction frame meshes. |
| **15 – 19 Immigrants** | Phase 4 | Commercial storefront textures, primary merchant NPC actor scripts. |
| **20 – 24 Immigrants** | Phase 5 | Expanded Inn model, town management administrative subroutine active. |
| **25 – 29 Immigrants** | Phase 6 | Stone church geometry, secondary retail shop models, religious NPC assets. |
| **30 – 34 Immigrants** | Phase 7 | Bank vault structures, specialized high-tier weapon/armor merchant models. |
| **35 – 40 Immigrants** | Phase 8 | Final town layout (dynamically branched based on majority resident classes). |

---

**5. `HBD1PS1D.Q71` Archive Hierarchy & Control Block Protocol**

All assets are structured within the flat CD-ROM resource archive `HBD1PS1D.Q71`, formatted into contiguous 2,048-byte physical sectors to match CD-ROM Mode 2 Form 1 framing.

* **Header Block (LBA 0):** Primary volume descriptor; contains ASCII signature `hbd1ps1d.q71` at byte offset `0x400` (1024).
* **Primary Data Directory Blocks (`* 00 00 00`):** Directory sector indexing up to 18 sub-blocks:
  * Offset `0x00` (uint32): Sub-block Count.
  * Offset `0x04` (uint32): Sector Count (total 2,048-byte sectors occupied).
  * Offset `0x08` (uint32): Total unpadded raw decompressed payload size.
  * Offset `0x0C` (uint32): Null padding (`0x00000000`).
* **Sub-Block Headers (Offset `0x10 + i * 16`):**
  * Offset `0x00` (uint32): Compressed Data Size on disc.
  * Offset `0x04` (uint32): Decompressed Allocation Size required in RAM.
  * Offset `0x08` (uint32): System flag / alignment register.
  * Offset `0x0C` (uint16): Compression Flag (`1280` / `0x0500` = HeartBeat LZSS0; `0` = uncompressed).
  * Offset `0x0E` (uint16): Resource Type Identifier (TIM sprites, 3D meshes, scripts).

### CD-ROM Controller Synchronization Blocks (`0x60010108`)

Control blocks are interspersed across sectors to govern CD-ROM controller timing and audio streaming synchronization:

| Header Offset | Size (Bytes) | Field Value / Mapping | Technical Function & Driver Impact |
|---|---|---|---|
| `0x00` | 4 | `0x60010108` | Control block magic signature. |
| `0x04` | 2 | `0x0000` – `0x0004` | Block index register (increments sequentially per physical block). |
| `0x06` | 2 | `0x0005` | Constant count identifier (fixed at 5). |
| `0x08` | 4 | Variable Counter | Part counter (increments by 1 upon block index reset). |
| `0x0C` | 4 | Integer Tracking | Internal system state tracking integer. |
| `0x10` | 2 | `0x8000` | SPU interface configuration constant (decimal 128). |
| `0x12` | 2 | `0x7800` | Audio stream channel mapping constant (decimal 120). |
| `0x14` | 4 | `0xXXXXXX38` | Hardware register offset (fourth byte strictly locked to `0x38`). |
| `0x18` | 2 | `0x0100` / `0x0200` / `0x0300` | System execution mode flag. |
| `0x1A` | 4 | `0x03000000` | Primary interface driver baseline constant. |
| `0x1E` | 2 | `0x0000` | Static null alignment padding. |

---

**6. LZSS0 Decompression Algorithm & Isometric Tile Assembly**

When sub-block header offset `0x0C` evaluates to `1280` (`0x0500`), the payload is inflated via the LZSS0 algorithm. HeartBeat utilized this lightweight sliding-window scheme because the MIPS R3000A lacks dedicated decompression hardware; the minimal CPU overhead of bitwise shifts and memory copies preserves execution cycles for VSync loops and the sound engine.

The sliding dictionary initializes a 4,096-byte ring buffer filled with `0x00`. Parsing is governed by a 1-byte control word $C$, evaluated bit-by-bit from LSB ($b_0$) to MSB ($b_7$):

$$\text{Bit } b_i \text{ of } C \implies \begin{cases} 1 & \text{Literal Byte Copy: Read raw byte, append to output and dictionary.} \\ 0 & \text{Dictionary Reference: Read 16-bit descriptor } W. \end{cases}$$

For dictionary references ($b_i = 0$), the 16-bit word $W$ is unpacked:
* **Ring Buffer Offset ($O$):** $O \leftarrow W \gg 4$
* **Match Length ($L$):** $L \leftarrow (W \ \& \ \text{0x0F}) + 3$
* The decompressor copies $L$ bytes from ring buffer position $O$ directly into destination RAM, updating the history buffer sequentially.

Input Stream:
+---+---+---+---+---+---+
| C |       W       |...|  (C = Control Byte, W = 16-bit Descriptor)
+---+---+---+---+---+---+
|
+--> b_i = 0 (Reference Flag)
|
+--> Decode W:
Offset (O) = W >> 4
Length (L) = (W & 0x0F) + 3
|
v
Sliding Ring Buffer (Size = 4096 Bytes, Initialized to 0x00):
+---------------------------------------+
|                [O]----->|             |
+--------------------+------------------+
|
+--> Copy L Bytes to Destination Output RAM Buffer


### Isometric Dungeon Layout Reconstruction

1. **Asset Retrieval:** `open_file` extracts the target dungeon block containing the compressed $16 \times 16$ tile bitmaps and the layout map.
2. **Texture Expansion:** LZSS0 unpacks texture sheets into a linear 4-bit or 8-bit indexed array referencing local CLUTs.
3. **Descriptor Unpacking:** The layout sub-block provides a 2D matrix of 16-bit cell descriptors:
   * **Bits 0–9:** Tile Index (pointers to unpacked $16 \times 16$ texture array).
   * **Bit 10:** Horizontal Texture Mirror Flag.
   * **Bit 11:** Vertical Texture Mirror Flag.
   * **Bits 12–15:** Elevation / Height Parameter (isometric rendering layer).
4. **Elevation Coordinate Offsetting:** Cells with active elevation bits have their vertical drawing coordinates adjusted downward by a constant 4 to 6 pixels, ensuring correct visual alignment and depth sorting between elevated walls and floor tiles.

---

**7. Integrated Architecture & Data Flow Synthesis**

+---------------------------------------------------------------------------------+
|                       HBD1PS1D.Q71 (CD-ROM Mode 2 Form 1 Archive)               |
|  +--------------------+  +-------------------------+  +-----------------------+ |
|  | Primary Header     |  | * 00 00 00 Data Block   |  | 0x60010108 Control    | |
|  | (Volume ID LBA 0)  |  | (Asset Directories)     |  | (CD Controller Sync)  | |
|  +---------+----------+  +------------+------------+  +-----------------------+ |
+------------|--------------------------|-----------------------------------------+
|                          |
v                          v
+---------------------------------------------------------------------------------+
|                         PLAYSTATION SYSTEM RAM (2 MB)                           |
|  +---------------------------------------------------------------------------+  |
|  | Base Engine Segment (0x80000000 - 0x800849D4 Static Bounds)               |  |
|  |  +------------------------+  +-------------------+  +------------------+  |  |
|  |  | Gameplay Loop & Render |  | open_file Routine |  | SEQq Sound Prog. |  |  |
|  |  | (Geometry Pipeline)    |  | (0x80076040)      |  | (0x8002FA10)     |  |  |
|  |  +------------------------+  +---------+---------+  +------------------+  |  |
|  +----------------------------------------|----------------------------------+  |
|                                           | [mopen_file (0x80082598)]           |
|                                           v                                     |
|  +--------------------------------------------------+                           |
|  | Progress Flag Matrix (0x8000F800 - 0x80010000)   |                           |
|  | - Evaluates global quest flags & town population |                           |
|  +------------------------+-------------------------+                           |
|                           |                                                     |
|                           v [Sector-Aligned DMA Channel 4 Transfers]            |
|  +---------------------------------------------------------------------------+  |
|  | Dynamic Heap Segment (0x80138000 - 0x801FFFFF Volatile Bounds)            |  |
|  |  +----------------------------------+  +-------------------------------+  |  |
|  |  | Active Map & Geometry Space      |  | SPU Audio Waveform Buffer     |  |  |
|  |  | (LZSS0 Decompressed Tile-Sets)   |  | (Dynamic ADPCM Instruments)   |  |  |
|  |  +----------------------------------+  +-------------------------------+  |  |
|  +---------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------+


By decoupling disc transfers via asynchronous DMA streaming, compressing assets with a low-complexity sliding-window algorithm, and indexing map configurations through a consolidated global flag matrix, HeartBeat successfully deployed a massive, multi-disc RPG within the physical memory limits of the PlayStation console.
"""

filename = "STUDY_PSX_DW7_MONOLITHIC_ENGINE_Sep13_2026.md"
with open(filename, "w", encoding="utf-8") as f:
    f.write(content)
print(f"File written successfully: {filename}")

Generated the document:


STUDY_PSX_DW7_MONOLITHIC_ENGINE_Sep13_2026
 MD 
Markdown
# Reverse Engineering the Monolithic Engine of Dragon Warrior VII: Memory Partitioning, Event Flag Matrices, and Asset Compression Protocols

**Document ID:** ARCH-STUDY-PSX-DW7-ENGINE-20260913  
**Classification:** Deep-Dive Engine, Memory Architecture & Archive Protocol Audit  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Executables:** Sony PlayStation `SLUS_012.06` (North America) / `SLPM_865.00` (Japan)  
**Archive Target:** `HBD1PS1D.Q71` (CD-ROM Mode 2 Form 1 Resource Archive)  
**Date:** September 13, 2026  

---

**1. System Memory Architecture & Partitioning Map**

Operating within the PlayStation's strict 2 MB (2,048 KB) physical system RAM ceiling forced developer HeartBeat to bypass high-level Sony PsyQ operating system libraries in favor of a low-level, deterministic memory map. The memory space strictly partitions the resident "base engine" from volatile dynamic heaps and optical disc streaming buffers.

To eliminate the heavy RAM and CPU footprint of Sony's generic Sound Multi-Voice (SMV) and sequence/voice-bank (`SEQ`/`VAB`) drivers, HeartBeat implemented a custom assembly-level sound engine designated `SEQq` (signature `qQES`). This driver handles music sequences and ADPCM hardware voice allocations directly within dedicated work areas and volatile sequence buffers.

| Physical Address Range | Segment Classification | Engine Subsystem & Technical Allocation Scope |
|---|---|---|
| `0x80000000` – `0x8000F7FF` | Kernel & System Core | PlayStation operating system vectors, interrupt masks, PsyQ baseline hardware configuration. |
| `0x8000F800` – `0x800252FF` | Global Flag Space | Active progress registers, character attributes, event trigger matrices, and memory card save-state buffer. |
| `0x80025300` – `0x800258E8` | Base Sound Engine | Structural interpretation tables for the custom `SEQq` sound driver. |
| `0x8002FA10` – `0x80036FCC` | Base Sound Engine | Dedicated execution thread code (interrupt-driven sound processor). |
| `0x80037000` – `0x8007603F` | Core Executable | Primary game loop, rendering pipelines, dialogue engines, camera transformation matrices. |
| `0x80076040` – `0x8008247F` | Disc File System | CD-ROM optical disc file system routines, Logical Block Number (LBN) mapping, `open_file` dispatcher. |
| `0x80082480` – `0x800849D4` | Disc File System | Sound streaming program code, low-level non-blocking `mopen_file` file pointer tracker. |
| `0x80090E54` – `0x80094D14` | Base Sound Engine | SPU low-level interface routines, direct voice assignment, and audio channel panning registers. |
| `0x800B9D10` – `0x800BB6E7` | SPU Work RAM | SPU library parameter storage, hardware voice configuration registers, and echo buffer tracking. |
| `0x800D25C0` – `0x800D80EF` | Volatile Sequence Heap | Dynamic BGM sequence streaming buffer (maximum allocated footprint: 23,344 bytes). |
| `0x800D80F0` – `0x800DA0F7` | Volatile Sequence Heap | Instrument attribute maps, panning indices, ADSR envelope control registers. |
| `0x800F7798` – `0x800F7799` | Diagnostic Segment | Master debug mode registration address (16-bit hardware/software switch). |
| `0x80138000` – `0x8017FFFF` | Dynamic Heap Segment | Executing sequence (`SEQq`) playback buffers, active script parameters, streamed map geometry. |
| `0x80180000` – `0x801FFFFF` | Dynamic Heap Segment | SPU ADPCM soundbank (`VAB`) waveform sample buffer (compressed instrument data). |

---

**2. Custom `SEQq` Sound Driver Header Specification**

The custom `SEQq` audio format utilizes a compact 60-byte binary header that precedes all background music sequences. Unlike standard MIDI or PsyQ formats that require trailing delta-time termination events, `SEQq` streams terminate abruptly with the 3-byte token sequence `FF 2F 00`, stripping redundant framing overhead.

| Byte Offset | Size (Bytes) | Field Name | Technical Description & Subsystem Impact |
|---|---|---|---|
| `0x00` | 4 | `SEQ Size` | Size of the music sequence payload in bytes (evaluates to 0 if sequence is external or absent). |
| `0x04` | 2 | `SEQ ID` | Micro-sequence identifier (`msq_id`) passed to voice routing dispatchers. |
| `0x06` | 1 | `Bank Count` | Total number of ADPCM waveform banks allocated in SPU RAM (typically 4). |
| `0x07` | 1 | `Load Position` | Memory placement flag (`0x00` = Normal BGM, loads directly to static buffer address). |
| `0x08` | 4 | `Reserved` | Reserved system padding (zero-padded; unutilized by driver). |
| `0x0C` | 4 | `Bank 1 VB Size` | Byte size of Bank 1 compressed ADPCM waveforms (`0` if using resident preloaded cache). |
| `0x10` | 4 | `Bank 1 Region Size`| Size of the regional instrument mapping tables for Bank 1. |
| `0x14` | 2 | `Bank 1 ID` | Master instrument index descriptor used for SPU voice mapping. |
| `0x16` | 2 | `Bank 1 Attribute` | Default panning, pitch modulation registers, and resonance parameters. |
| `0x18` – `0x23` | 12 | `Bank 2 Header` | Configuration block for Bank 2 (mirrors Bank 1 structure: size, region, ID, attributes). |
| `0x24` – `0x2F` | 12 | `Bank 3 Header` | Configuration block for Bank 3 (mirrors Bank 1 structure: size, region, ID, attributes). |
| `0x30` – `0x3B` | 12 | `Bank 4 Header` | Configuration block for Bank 4 (mirrors Bank 1 structure: size, region, ID, attributes). |

---

**3. Dynamic CD-ROM Streaming Architecture & Menu VRAM Mechanics**

Traversing boundaries between the world map and dense town environments requires non-blocking data streaming to avoid frame drops. The file loading pipeline decouples logical sector requests from CPU execution:

[ Transition / Sector Boundary Collision ]
|
v
open_file (0x80076040)
- Resolves Target File Descriptor via LBN
|
v
mopen_file (0x80082598)
- Passed descriptor struct pointer via register $a0
- Splits transfer into sequential 2,048-byte Mode 2 Form 1 sector reads
|
v
[ DMA Channel 4 (CD-ROM Direct Memory Access) ]
- Direct streaming into Dynamic Heap (0x80138000)
- Bypasses Main MIPS CPU cycles across multi-frame loading intervals


### The Upscaling Menu Blur Discrepancy

When an in-game menu is invoked, the engine captures a snapshot of the active VRAM framebuffer and transfers it to an offscreen buffer in System RAM to serve as a static backdrop. To conserve memory bandwidth, this copy is captured strictly at the native internal rendering resolution ($256 \times 224$ or $320 \times 240$). On native CRT displays, this handoff is imperceptible; under modern hardware-accelerated emulation with upscaled 3D geometry, high-resolution 2D UI elements are overlaid on top of this low-resolution snapshot, producing a prominent pixelated/blurred background artifact.

---

**4. Event Flag Matrix & Dynamic World Streaming**

Global progression across multiple historical eras is tracked in a bit-packed matrix located at `0x8000F800` – `0x80010000`. Progress updates are committed via MIPS store-byte instructions (`sb v1, $0(a1)`), where `$a1` holds the destination flag pointer. Conditional checks (`beq v0, a0, Target`) govern script jumps, gate unsealing, and world geometry queries.

### Immigrant Town Flag-Driven Progression States

The engine dynamically loads town geometry by evaluating the population counter flag stored in the matrix against lookup tables in `SLUS_012.06`, ensuring only current-phase assets occupy the volatile dynamic heap at `0x80138000`.

| Active Population Flag | Structural Phase | Streamed 3D Model Assets & Resident NPC Configurations |
|---|---|---|
| **1 – 4 Immigrants** | Phase 1 | Base campsite terrain mesh, canvas tents, Sim NPC base model. |
| **5 – 9 Immigrants** | Phase 2 | Small wooden shack model, dynamic town naming interface initialized. |
| **10 – 14 Immigrants** | Phase 3 | Multi-room building models, timber construction frame meshes. |
| **15 – 19 Immigrants** | Phase 4 | Commercial storefront textures, primary merchant NPC actor scripts. |
| **20 – 24 Immigrants** | Phase 5 | Expanded Inn model, town management administrative subroutine active. |
| **25 – 29 Immigrants** | Phase 6 | Stone church geometry, secondary retail shop models, religious NPC assets. |
| **30 – 34 Immigrants** | Phase 7 | Bank vault structures, specialized high-tier weapon/armor merchant models. |
| **35 – 40 Immigrants** | Phase 8 | Final town layout (dynamically branched based on majority resident classes). |

---

**5. `HBD1PS1D.Q71` Archive Hierarchy & Control Block Protocol**

All assets are structured within the flat CD-ROM resource archive `HBD1PS1D.Q71`, formatted into contiguous 2,048-byte physical sectors to match CD-ROM Mode 2 Form 1 framing.

* **Header Block (LBA 0):** Primary volume descriptor; contains ASCII signature `hbd1ps1d.q71` at byte offset `0x400` (1024).
* **Primary Data Directory Blocks (`* 00 00 00`):** Directory sector indexing up to 18 sub-blocks:
  * Offset `0x00` (uint32): Sub-block Count.
  * Offset `0x04` (uint32): Sector Count (total 2,048-byte sectors occupied).
  * Offset `0x08` (uint32): Total unpadded raw decompressed payload size.
  * Offset `0x0C` (uint32): Null padding (`0x00000000`).
* **Sub-Block Headers (Offset `0x10 + i * 16`):**
  * Offset `0x00` (uint32): Compressed Data Size on disc.
  * Offset `0x04` (uint32): Decompressed Allocation Size required in RAM.
  * Offset `0x08` (uint32): System flag / alignment register.
  * Offset `0x0C` (uint16): Compression Flag (`1280` / `0x0500` = HeartBeat LZSS0; `0` = uncompressed).
  * Offset `0x0E` (uint16): Resource Type Identifier (TIM sprites, 3D meshes, scripts).

### CD-ROM Controller Synchronization Blocks (`0x60010108`)

Control blocks are interspersed across sectors to govern CD-ROM controller timing and audio streaming synchronization:

| Header Offset | Size (Bytes) | Field Value / Mapping | Technical Function & Driver Impact |
|---|---|---|---|
| `0x00` | 4 | `0x60010108` | Control block magic signature. |
| `0x04` | 2 | `0x0000` – `0x0004` | Block index register (increments sequentially per physical block). |
| `0x06` | 2 | `0x0005` | Constant count identifier (fixed at 5). |
| `0x08` | 4 | Variable Counter | Part counter (increments by 1 upon block index reset). |
| `0x0C` | 4 | Integer Tracking | Internal system state tracking integer. |
| `0x10` | 2 | `0x8000` | SPU interface configuration constant (decimal 128). |
| `0x12` | 2 | `0x7800` | Audio stream channel mapping constant (decimal 120). |
| `0x14` | 4 | `0xXXXXXX38` | Hardware register offset (fourth byte strictly locked to `0x38`). |
| `0x18` | 2 | `0x0100` / `0x0200` / `0x0300` | System execution mode flag. |
| `0x1A` | 4 | `0x03000000` | Primary interface driver baseline constant. |
| `0x1E` | 2 | `0x0000` | Static null alignment padding. |

---

**6. LZSS0 Decompression Algorithm & Isometric Tile Assembly**

When sub-block header offset `0x0C` evaluates to `1280` (`0x0500`), the payload is inflated via the LZSS0 algorithm. HeartBeat utilized this lightweight sliding-window scheme because the MIPS R3000A lacks dedicated decompression hardware; the minimal CPU overhead of bitwise shifts and memory copies preserves execution cycles for VSync loops and the sound engine.

The sliding dictionary initializes a 4,096-byte ring buffer filled with `0x00`. Parsing is governed by a 1-byte control word $C$, evaluated bit-by-bit from LSB ($b_0$) to MSB ($b_7$):

$$\text{Bit } b_i \text{ of } C \implies \begin{cases} 1 & \text{Literal Byte Copy: Read raw byte, append to output and dictionary.} \\ 0 & \text{Dictionary Reference: Read 16-bit descriptor } W. \end{cases}$$

For dictionary references ($b_i = 0$), the 16-bit word $W$ is unpacked:
* **Ring Buffer Offset ($O$):** $O \leftarrow W \gg 4$
* **Match Length ($L$):** $L \leftarrow (W \ \& \ \text{0x0F}) + 3$
* The decompressor copies $L$ bytes from ring buffer position $O$ directly into destination RAM, updating the history buffer sequentially.

Input Stream:
+---+---+---+---+---+---+
| C |       W       |...|  (C = Control Byte, W = 16-bit Descriptor)
+---+---+---+---+---+---+
|
+--> b_i = 0 (Reference Flag)
|
+--> Decode W:
Offset (O) = W >> 4
Length (L) = (W & 0x0F) + 3
|
v
Sliding Ring Buffer (Size = 4096 Bytes, Initialized to 0x00):
+---------------------------------------+
|                [O]----->|             |
+--------------------+------------------+
|
+--> Copy L Bytes to Destination Output RAM Buffer


### Isometric Dungeon Layout Reconstruction

1. **Asset Retrieval:** `open_file` extracts the target dungeon block containing the compressed $16 \times 16$ tile bitmaps and the layout map.
2. **Texture Expansion:** LZSS0 unpacks texture sheets into a linear 4-bit or 8-bit indexed array referencing local CLUTs.
3. **Descriptor Unpacking:** The layout sub-block provides a 2D matrix of 16-bit cell descriptors:
   * **Bits 0–9:** Tile Index (pointers to unpacked $16 \times 16$ texture array).
   * **Bit 10:** Horizontal Texture Mirror Flag.
   * **Bit 11:** Vertical Texture Mirror Flag.
   * **Bits 12–15:** Elevation / Height Parameter (isometric rendering layer).
4. **Elevation Coordinate Offsetting:** Cells with active elevation bits have their vertical drawing coordinates adjusted downward by a constant 4 to 6 pixels, ensuring correct visual alignment and depth sorting between elevated walls and floor tiles.

---

**7. Integrated Architecture & Data Flow Synthesis**

+---------------------------------------------------------------------------------+
|                       HBD1PS1D.Q71 (CD-ROM Mode 2 Form 1 Archive)               |
|  +--------------------+  +-------------------------+  +-----------------------+ |
|  | Primary Header     |  | * 00 00 00 Data Block   |  | 0x60010108 Control    | |
|  | (Volume ID LBA 0)  |  | (Asset Directories)     |  | (CD Controller Sync)  | |
|  +---------+----------+  +------------+------------+  +-----------------------+ |
+------------|--------------------------|-----------------------------------------+
|                          |
v                          v
+---------------------------------------------------------------------------------+
|                         PLAYSTATION SYSTEM RAM (2 MB)                           |
|  +---------------------------------------------------------------------------+  |
|  | Base Engine Segment (0x80000000 - 0x800849D4 Static Bounds)               |  |
|  |  +------------------------+  +-------------------+  +------------------+  |  |
|  |  | Gameplay Loop & Render |  | open_file Routine |  | SEQq Sound Prog. |  |  |
|  |  | (Geometry Pipeline)    |  | (0x80076040)      |  | (0x8002FA10)     |  |  |
|  |  +------------------------+  +---------+---------+  +------------------+  |  |
|  +----------------------------------------|----------------------------------+  |
|                                           | [mopen_file (0x80082598)]           |
|                                           v                                     |
|  +--------------------------------------------------+                           |
|  | Progress Flag Matrix (0x8000F800 - 0x80010000)   |                           |
|  | - Evaluates global quest flags & town population |                           |
|  +------------------------+-------------------------+                           |
|                           |                                                     |
|                           v [Sector-Aligned DMA Channel 4 Transfers]            |
|  +---------------------------------------------------------------------------+  |
|  | Dynamic Heap Segment (0x80138000 - 0x801FFFFF Volatile Bounds)            |  |
|  |  +----------------------------------+  +-------------------------------+  |  |
|  |  | Active Map & Geometry Space      |  | SPU Audio Waveform Buffer     |  |  |
|  |  | (LZSS0 Decompressed Tile-Sets)   |  | (Dynamic ADPCM Instruments)   |  |  |
|  |  +----------------------------------+  +-------------------------------+  |  |
|  +---------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------+


By decoupling disc transfers via asynchronous DMA streaming, compressing assets with a low-complexity sliding-window algorithm, and indexing map configurations through a consolidated global flag matrix, HeartBeat successfully deployed a massive, multi-disc RPG within the physical memory limits of the PlayStation console.
