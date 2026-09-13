# Architectural Analysis of PlayStation Engine Hybridization in Dragon Quest IV

**Document ID:** ARCH-STUDY-PSX-DQ4-HYBRIDIZATION-20260913  
**Classification:** Deep-Dive Engine & Graphics Architecture Audit  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Executable:** Sony PlayStation `SLPM_869.16` (MIPS R3000A) + `HBD1PS1D.Q41`[cite: 1]  
**Comparative Target:** `SLPM_865.00` (*Dragon Quest VII: Eden no Senshitachi*)  
**Date:** September 13, 2026  

---

**1. Architectural Overview: The 2D/3D Hybrid Paradigm**

The PlayStation remake of *Dragon Quest IV: Chapters of the Chosen* (`SLPM_869.16`)[cite: 1] was engineered by HeartBeat by refactoring the custom 3D isometric engine originally built for *Dragon Quest VII* (`SLPM_865.00`). Adapting a 16-bit tile-based Famicom design into a 32-bit hardware pipeline required bridging two opposing rendering paradigms: an isometric, 360-degree rotatable 3D polygonal terrain grid alongside 2D character billboard quads.

HeartBeat avoided standard Sony graphics libraries in favor of hand-tuned assembly, custom memory limits, optimized asset tables, and a specialized billboard-to-terrain coordinate transformation pipeline executed directly on the PlayStation's Geometry Transfer Engine (GTE).
+-----------------------------------+
              |         System Boot Entry         |
              +-----------------------------------+
                                |
                                v
              +-----------------------------------+
              |   Initialize PsyQ, GTE, & CD-ROM  |
              +-----------------------------------+
                                |
                                v
              +-----------------------------------+
              |       Register VSync Handler      |
              +-----------------------------------+
                                |
                                v
        +---> +-----------------------------------+
        |     |        Poll Input Devices         |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Execute Script Interpreter     |
        |     |         (Type 39 Block)           |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Update Actor State Machines    |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Update Camera Coordinates      |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |     Sort Primitives (OT / GTE)    |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |     Push DMA Buffer to VRAM       |
        |     +-----------------------------------+
        |                       |
        +-----------------------+
        ---

**2. Memory Mapping & BSS Segment Configuration**

During boot initialization, a primary memory-clearing routine at `0x8008E284` cleanses the BSS (Block Started by Symbol) segment to prevent stale variables from corrupting engine states. HeartBeat restructured the dynamic memory pools used in *Dragon Quest VII*, shrinking 3D world geometry allocations to expand resident buffers for multi-directional 2D sprite sheets.

| System Subsystem / Memory Pool | DQVII Allocation (`SLPM_865.00`) | DQIV Allocation (`SLPM_869.16`) | Functional Purpose & Engineering Modification |
|---|---|---|---|
| **System Boot Entry Point (`start`)** | `0x8008DAC0` | `0x8008DAC0` | Standard PsyQ entry point; unchanged hardware bootstrap. |
| **BSS Clearing Routine** | `0x8008DAC0` – `0x8008E1F0` | `0x8008E284` (Entry) | Shifted entry target to accommodate customized engine initializers. |
| **BGM Sequence Queue (Dynamic Buffer)** | `0x800D25C0` – `0x800D80EF` | `0x800D1500` – `0x800D7500` | Reduced footprint to expand main RAM variable storage. |
| **Waveform Instrument Attributes** | `0x800D80F0` – `0x800DA0F7` | `0x800D7600` – `0x800D9600` | Relocated to align with the customized sound driver footprint. |
| **VRAM Draw/Display Page Buffers** | `0x800B9D10` – `0x800BB6E7` | `0x800BA800` – `0x800BC200` | Expanded page buffers to eliminate frame drops during active map rotation. |
| **Sound Interrupt Thread** | `0x8002FA10` – `0x80036FCC` | `0x8002FA10` – `0x80036FCC` | Preserved core driver architecture for sound playback. |
| **Sequence Interpretation Tables** | `0x80025300` – `0x800258E8` | `0x80025300` – `0x800258E8` | Preserved core script instruction sequencing tables. |
| **Sprite-to-3D Grid Mapping Init** | Dynamic model allocator | `0x8005C1F0` | Configures structured actor state array (3D vector, viewport anchor, rotation, sprite pointer). |

---

**3. Physical Archive Architecture (`HBD1PS1D.Q41`)**

All non-executable assets are packed into `HBD1PS1D.Q41`, a flat CD-ROM resource archive spanning 319,436,800 bytes across 155,975 Mode 2 Form 1 sectors (2,048 bytes user data per sector)[cite: 1]. The first block (LBA 0, offset `0x00`–`0x800`) contains the volume header with the ASCII identifier `hbd1ps1d.q41` at offset `0x400`.

+-------------------------------------------------------------+
|                  HBD1PS1D.Q41 Sector Layout                 |
+-------------------------------------------------------------+
| 0x00  Main Block Header (16 Bytes)                          |
|       - Sub-block count (uint32)                            |
|       - Sector size of block (uint32)                       |
|       - Total decompressed raw data length (uint32)         |
|       - Reserved / zero padding (uint32)                    |
+-------------------------------------------------------------+
| 0x10  Sub-Block Headers (16 Bytes per Sub-block)            |
|       - Compressed data size (uint32)                       |
|       - Decompressed data size (uint32)                     |
|       - Offset within sector payload (uint32)               |
|       - Compression Flag (uint16) [e.g., 0x0500 / 1280]     |
|       - Sub-Block Type (uint16)                             |
+-------------------------------------------------------------+
|       Raw Payload Segment                                   |
|       - Compressed or raw data blocks                       |
|       - Padded with 0x00 to complete 2,048-byte boundary    |
+-------------------------------------------------------------+


### Sub-Block Type Catalog

| Type Code | Compression Status | Resource Classification | Subsystem Role & Hardware Integration |
|---|---|---|---|
| **Type 1** | Uncompressed | Font Glyph Sheets | Loaded into VRAM offscreen memory to populate the text engine character cache. |
| **Type 6** | Compressed (DQLZS) | Map Chipset Images | Holds textures and 2D tilesets used to construct world and town maps. |
| **Type 8** | Compressed (DQLZS) | Monster & Battle Sprites | Standard PlayStation TIM images detailing battle animations and opponent sprites. |
| **Type 10** | Compressed (DQLZS) | Multi-TIM Packages | Multi-directional frames for active combat graphics and visual spell effects. |
| **Type 13** | Compressed (DQLZS) | NPC & Player Sprites | Multi-directional sheets containing player character and town NPC animation cycles. |
| **Type 21** | Uncompressed | 3D Map Geometry (`qQES`) | Underlying 3D world meshes, polygon definitions, and collision vectors. |
| **Type 32** | Uncompressed | Scene Dialogue Index Maps | Translates active script commands to physical text offsets in dialogue containers. |
| **Type 39** | Compressed (LZSS) | Cutscene Script Engine | Bytecode instructions controlling actor movement, camera cues, and text triggers[cite: 1]. |

---

**4. Script Virtual Machine & Multi-Chapter Parser Syntax**

Dialogue and event execution are driven by Type 39 script blocks[cite: 1], indexed by Type 32 tables and dispatched via the virtual machine command `C021A0`:
* `C021A0 <offset> <dialogId>`: Jumps to a designated text offset inside the decompressed buffer.
* `C021A0 <FFF0> <key>`: Evaluates event flags, active party leaders, or inventory counts.

### Inline Control Code Taxonomy

| Token Code | Control Semantics | Hardware / Blitter Execution |
|---|---|---|
| `{0000}` | Stream Terminator | Terminates active dialogue window thread; closes parser loop. |
| `{7F02}` | Newline / Carriage Reset | Resets drawing cursor to left margin of next dialogue row. |
| `{7F04} {7Fxx}` | Dynamic Name Tag | Reads following byte (`7Fxx`) and renders designated actor name string. |
| `{7F0A}` | Wait-For-Input Prompt | Displays blinking cursor; suspends script engine until pad input is confirmed. |
| `{7F0B}` | Auto End-of-Line | Concludes line; delays frame count before auto-progressing. |
| `{7F15}` | System Numeric Variable | Injects dynamic decimal integers (gold counts, stats, quantities). |

### Conditional Branching Operators

HeartBeat eliminated script duplication across chapters using an inline conditional syntax:

$$\%A\{\text{ID}\}\%X\{\text{TEXT if not ID}\}\%Z\%B\{\text{ID}\}\%X\{\text{TEXT if ID}\}\%Z$$

* **Party Leader Evaluation (`%A` / `%B`):** Evaluates active character ID. If leader matches specified ID, the `%A` block is skipped and the `%B` branch executes.
  * *Example Stream:* `%A120%XStill, if she were a boy—%Z%B120%XStill, if you were a boy—%Z`
  * *Result (Alena ID 120):* `"Still, if you were a boy—"`
  * *Result (Other Leader):* `"Still, if she were a boy—"`
* **Grammatical Plural Evaluation (`%H`):** Evaluates target integer variable; if value $> 1$, renders the plural suffix within the `%Y` block:
  $$\%H\{\text{Variable ID}\}\%X\%Y\text{s}\%Z$$

---

**5. Billboard Transformation Mathematics & VRAM Double Buffering**

To ensure 2D billboard character sprites remain upright and face the screen during 360-degree map rotations, the GTE splits terrain and sprite coordinate transformations into two separate mathematical pipelines.

0,0           320,0                     512,0                     1024,0
+---------------+-------------------------+-------------------------+
| Framebuffer 0 |                         | Offscreen Texture Cache |
| (Draw Buffer) |                         | - Character Sheets      |
| (320 x 240)   |                         | - Monster TIM Images    |
+---------------+-------------------------+ - Font Atlas Pages      |
| Framebuffer 1 |                         |                         |
| (Disp Buffer) |                         |                         |
| (320 x 240)   |                         |                         |
+---------------+-------------------------+-------------------------+
0,512                                                               1024,512


### Coordinate Transformation Pipeline

1. **3D Grid Terrain Vertices:** Transformed by the full camera rotation matrix $\mathbf{R}_y(\theta)$ and translation vector $\mathbf{T}$:
   $$\mathbf{P}_c = \mathbf{R}_y(\theta)\mathbf{P}_w + \mathbf{T}, \quad \text{where} \quad \mathbf{R}_y(\theta) = \begin{pmatrix} \cos\theta & 0 & \sin\theta \\ 0 & 1 & 0 \\ -\sin\theta & 0 & \cos\theta \end{pmatrix}$$
2. **2D Billboard Anchors:** World-space anchor $\mathbf{A}_w = [A_x, A_y, A_z]^T$ is transformed to camera space to lock depth:
   $$\mathbf{A}_c = \mathbf{R}_y(\theta)\mathbf{A}_w + \mathbf{T}$$
3. **Quad Vertex Generation (Rotation Bypass):** Local width ($w$) and height ($h$) offsets are added directly to camera-space anchor $\mathbf{A}_c$ with $\Delta z = 0$, keeping the quad parallel to the projection plane:
   $$\mathbf{V}_c^{(i)} = \mathbf{A}_c + \begin{pmatrix} \Delta x^{(i)} \\ \Delta y^{(i)} \\ 0 \end{pmatrix}, \quad \Delta x^{(i)} \in \{-w/2, w/2\}, \; \Delta y^{(i)} \in \{0, h\}$$
4. **Perspective Screen Projection:** GTE projects camera-space vertices to 2D screen coordinates $(X_s, Y_s)$:
   $$X_s = \frac{V_{c,x} \cdot f}{V_{c,z}} + X_0, \quad Y_s = \frac{V_{c,y} \cdot f}{V_{c,z}} + Y_0$$
5. **Ordering Table Sort:** Primitives are inserted into the Ordering Table (OT) back-to-front, drawing 2D sprite quads over 3D terrain meshes without depth buffer artifacts or clipping.
