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
        |     |   Update Camera Coordinates (Rot) |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Sort Primitives (Ordering      |
        |     |        Table & GTE Math)          |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Push DMA Buffer to GPU VRAM    |
        |     +-----------------------------------+
        |                       |
        +-----------------------+

### BSS Segment Cleansing & Memory Mapping Variations

During initialization, the executable runs a critical memory-clearing routine at `0x8008E284`. This function cleanses the BSS (Block Started by Symbol) segment, zeroing out uninitialized global and static variables to prevent garbage data from corrupting the engine state.

Comparing this routine and the resulting memory configuration against *Dragon Quest VII* (`SLPM_865.00`) reveals how the developers modified memory boundaries to support a hybrid 2D/3D rendering pipeline:

| System Subsystem or Memory Pool | DQVII Memory Allocations (`SLPM_865.00`) | DQIV Remapped Allocations (`SLPM_869.16`) | Functional Modification and Engineering Purpose |
|---|---|---|---|
| **System Boot Entry Point (`start`)** | `0x8008DAC0` | `0x8008DAC0` | Unchanged standard compiler entry point for hardware initialization. |
| **BSS Clearing Routine** | `0x8008DAC0` – `0x8008E1F0` | `0x8008E284` (Entry Target) | Shifted offset to accommodate customized engine initializers. |
| **BGM Sequence Queue (Dynamic Buffer)** | `0x800D25C0` – `0x800D80EF` | `0x800D1500` – `0x800D7500` | Reduced footprint to expand main RAM variable storage. |
| **Waveform Instrument Attributes** | `0x800D80F0` – `0x800DA0F7` | `0x800D7600` – `0x800D9600` | Relocated to align with the customized sound driver footprint. |
| **VRAM Draw/Display Double Buffers** | `0x800B9D10` – `0x800BB6E7` | `0x800BA800` – `0x800BC200` | Expanded page buffers to prevent frame dropping during map rotation. |
| **Sound Interrupt Thread** | `0x8002FA10` – `0x80036FCC` | `0x8002FA10` – `0x80036FCC` | Preserved core driver architecture for sound playback. |
| **Sequence Interpretation Tables** | `0x80025300` – `0x800258E8` | `0x80025300` – `0x800258E8` | Preserved driver mapping configurations for MIDI sequences. |

By shifting the BSS boundaries, Heart Beat expanded the maximum allocation size for sprite lookup tables. Unlike *Dragon Quest VII*, which used simple 3D models for many secondary characters, *Dragon Quest IV* retained classic 2D character designs, requiring constant buffering of multi-directional, highly detailed sprite sheets for players and NPCs.

### Sprite-to-3D-Grid Mapping Routine Initialization

The specific routine responsible for initializing the mapping of 2D sprites onto the 3D isometric grid is located at offset `0x8005C1F0` in program memory. Running immediately after the engine's rendering context is established, it registers a structured array of actor state blocks in memory:
* 3D world position vector ($X, Y, Z$)
* 2D viewport anchor point
* Active rotation angle
* Pointer to the character's active sprite sheet

This initialization routine configures transformation registers on the PlayStation's GTE, defining how the 3D coordinate frame of the terrain grid translates to the screen-space coordinates of the billboard quads. This prevents visual alignment issues and model drifting during scene rendering.

---

## 3. Asset Table Mapping & LBA File Structure

### Archive Format & Sector-Level Layout

All non-executable assets are packed into a single resource archive named `HBD1PS1D.Q41`, structured identically to *Dragon Quest VII*'s `HBD1PS1D.Q71`. The archive spans 319,436,800 bytes, dividing into 155,975 blocks of 2,048 bytes (matching physical CD-ROM Mode 2 Form 1 sectors).

LBA 0 (offsets `0x00` through `0x800`) holds the volume identification header, containing the ASCII identifier `hbd1ps1d.q41` at position `0x400` (1024). The remaining sectors follow a strictly aligned block system:

+-------------------------------------------------------------+
|                  HBD1PS1D.Q41 Sector Layout                 |
+-------------------------------------------------------------+
| 0x00 Main Block Header (16 Bytes)                          |
|      - Sub-block count (uint32)                             |
|      - Sector size of block (uint32)                        |
|      - Total decompressed raw data length (uint32)          |
|      - Reserved / zero padding (uint32)                     |
+-------------------------------------------------------------+
| 0x10 Sub-Block Headers (16 Bytes per Sub-block)             |
|      - Compressed data size (uint32)                        |
|      - Decompressed data size (uint32)                      |
|      - Offset within sector payload (uint32)                |
|      - Compression Flag (uint16) [e.g., 1280]               |
|      - Sub-Block Type (uint16)                              |
+-------------------------------------------------------------+
| Raw Payload Segment                                         |
|      - Compressed or raw data blocks                        |
|      - Padded with 0x00 to complete 2048-byte boundary      |
+-------------------------------------------------------------+


The compression flag at offset `0x0C` contains decimal `1280` (`0x0500`) when the payload is compressed using Heart Beat's variant of the LZS algorithm. If uncompressed, this flag is set to `0`, and the compressed size equals the decompressed size.

### Sub-Block Type Catalog

| Sub-Block Type Code | Compression Status | Primary Resource Contents | Role and System Integration |
|---|---|---|---|
| **Type 1** | Uncompressed | Font Glyph Sheets | Loaded directly into VRAM to populate the text engine's character cache. |
| **Type 6** | Compressed | Map Chipset Images | Holds textures and 2D tilesets used to construct world and town maps. |
| **Type 8** | Compressed | Monster & Battle Sprites | Standard PlayStation TIM images detailing battle animations and opponent sprites. |
| **Type 10** | Compressed | Multi-TIM Sprite Packages | Multi-directional frames for active combat graphics and effects. |
| **Type 13** | Compressed | NPC & Player Sprites | Multi-directional sheets containing player character and town NPC animation cycles. |
| **Type 21** | Uncompressed | 3D Map Geometry (`qQES`) | Holds the underlying 3D world meshes, polygon definitions, and collision fields. |
| **Type 32** | Uncompressed | Scene Dialogue Index Maps | Translates active script commands to physical text offsets in dialogue files. |
| **Type 39** | Compressed | Primary Dialogue Script Engine | Contains the script instructions and text data. |

---

## 4. Dialogue Script Parsing & Conditional Control Structures

### Text Block Architecture & Parser State

Dialogue files (Type 39 sub-blocks) are compressed using Huffman coding. When dialogue triggers, the engine locates the associated script segment via an offset index from Type 32 blocks. The script block starts with a 24-byte header:

+-----------------------------------------------------------------+
|                        Text Block Header                        |
+-----------------------------------------------------------------+
| Offset 0x00: Pointer offset "a" to end of block payload (uint32)|
| Offset 0x04: Unique Scenario Scene ID (uint32)                  |
| Offset 0x08: Offset pointer "c" to Huffman-encoded bitstream    |
| Offset 0x0C: Offset pointer "d" to Huffman Tree structures      |
| Offset 0x10: Offset pointer "e" to end of Huffman bitstream     |
| Offset 0x14: Reserved / padding                                 |
+-----------------------------------------------------------------+


The script execution path is driven by the virtual machine command `C021A0`:
* `C021A0 <offset> <dialogId>`: Directs the interpreter to load a specific dialogue stream, jumping to the designated offset inside the decompressed text buffer.
* `C021A0 <FFF0> <key>`: Evaluates event flags or registers system variables (such as item counts or active party members) to alter the dialogue flow.

### Formatting & Parser Control Codes

| Control Code | Operational Role | Parsing & Blitter Behavior |
|---|---|---|
| `{0000}` | End of Dialogue | Closes the current dialogue stream and terminates the window thread. |
| `{7F02}` | New Line & Tab | Resets the drawing cursor to the left margin of the next dialogue line. |
| `{7F04} {7Fxx}` | Name Decorator | Instructs interpreter to read the next byte (`7Fxx`) and render the matching actor's name. |
| `{7F0A}` / `{7F0B}` | End-of-Line Signifiers | Displays a blinking cursor and pauses script execution until player input is confirmed. |
| `{7F15}` | Dynamic Variable Token | Injects dynamic numeric values (such as party gold counts) directly into the text stream. |

### Conditional Scripting Syntax & Multi-Chapter Execution

To manage the game's chapter-based progression without duplicating script assets, Heart Beat implemented an inline conditional branching syntax:

$$\%A\{\text{ID}\}\%X\{\text{TEXT if not ID}\}\%Z\%B\{\text{ID}\}\%X\{\text{TEXT if ID}\}\%Z$$

When the interpreter encounters the `%A` control statement, it evaluates the subsequent character ID against the active party leader's ID:
* If the active leader does **not** match the specified ID, the engine executes the `%X` text block until it reaches `%Z`.
* If the active leader **matches**, the engine skips the `%A` block and executes the `%B` branch.

              +----------------------------------+
              |     Parse dialogue statement     |
              +----------------------------------+
                                |
                                v
              +----------------------------------+
              |    Encounter %A {ID} Operator    |
              +----------------------------------+
                                |
                /-----------------\-----------------\
               /                                     \
              v                                       v
  +-----------------------+               +-----------------------+
  |   Execute %X Branch   |               |   Skip %A %X Block    |
  |  ("TEXT if not ID")   |               +-----------------------+
  +-----------------------+                           |
              |                                       v
              v                           +-----------------------+
  +-----------------------+               |    Evaluate %B {ID}   |
  |   Skip %B %X Block    |               +-----------------------+
  +-----------------------+                           |
              |                                       v
              |                           +-----------------------+
              |                           |   Execute %X Branch   |
              |                           |    ("TEXT if ID")     |
              |                           +-----------------------+
              \                                       /
               \-------------------------------------/
                                |
                                v
                    +---------------------------+
                    |   Resume parsing script   |
                    +---------------------------+

* **Branching Example:**
  * Script Stream: `%A120%XStill, if she were a boy—%Z%B120%XStill, if you were a boy—%Z`
  * Result (Active leader is Alena / ID 120): `"Still, if you were a boy—"`
  * Result (Active leader is Ragnar or Torneko): `"Still, if she were a boy—"`
* **Pluralization Logic (`%H`):** Evaluates whether a designated numeric variable exceeds 1 to conditionally render plural suffixes:
  $$\%H\{\text{Variable ID}\}\%X\%Y\text{s}\%Z$$

---

## 5. Sprite-Grid Synchronization & VRAM Buffer Processing

### Billboard Coordinate Transformation Mathematics

To construct the world map, the engine merges a rotatable 3D polygonal landscape with flat 2D character sprites. The player can rotate the camera in 90-degree increments (with smooth interpolation transitions). To avoid distortion or flattening as the angle approaches 90 degrees, the engine uses two distinct transformation pathways.

Let the coordinate of a 3D grid vertex in world space be:
$$\mathbf{P}_w = \begin{pmatrix} X_w \\ Y_w \\ Z_w \end{pmatrix}$$

Camera orientation is defined by rotation matrix $\mathbf{R}_y(\theta)$ around the vertical Y-axis:
$$\mathbf{R}_y(\theta) = \begin{pmatrix} \cos\theta & 0 & \sin\theta \\ 0 & 1 & 0 \\ -\sin\theta & 0 & \cos\theta \end{pmatrix}$$

Applying camera translation vector $\mathbf{T}$, the 3D grid vertex is transformed into camera-space coordinates $\mathbf{P}_c$:
$$\mathbf{P}_c = \mathbf{R}_y(\theta) \mathbf{P}_w + \mathbf{T}$$

For 2D billboard character sprites, internal quad rotation is bypassed:
1. Extract the sprite's 3D anchor position in world space: $\mathbf{A}_w = [A_x, A_y, A_z]^T$.
2. Transform the anchor into camera space to determine depth and center position:
   $$\mathbf{A}_c = \mathbf{R}_y(\theta) \mathbf{A}_w + \mathbf{T}$$
3. Generate the four quad vertices $\mathbf{V}_c^{(i)}$ ($i \in \{1, 2, 3, 4\}$) by adding local width ($w$) and height ($h$) offsets directly to $\mathbf{A}_c$, keeping $\Delta z = 0$:
   $$\mathbf{V}_c^{(i)} = \mathbf{A}_c + \begin{pmatrix} \Delta x^{(i)} \\ \Delta y^{(i)} \\ 0 \end{pmatrix}$$
   where $\Delta x^{(i)} \in \{-w/2, w/2\}$ and $\Delta y^{(i)} \in \{0, h\}$.
4. The GTE projects camera-space coordinates to 2D screen coordinates $(X_s, Y_s)$ via perspective projection:
   $$X_s = \frac{V_{c,x} \cdot f}{V_{c,z}} + X_0, \quad Y_s = \frac{V_{c,y} \cdot f}{V_{c,z}} + Y_0$$
   where $f$ is focal length and $(X_0, Y_0)$ represents the framebuffer projection center offset.

### VRAM Layout & Page Swapping

To prevent screen tearing and rendering artifacts during active camera rotation, Heart Beat utilized double-buffered page swapping across the $1024 \times 512$ pixel VRAM space:

0,0           320,0                     512,0                     1024,0
+---------------+-------------------------+-------------------------+
|               |                         |                         |
| Framebuffer 0 |                         | Offscreen Textures      |
| (Draw Buffer) |                         | - Character Sheets      |
|  (320 x 240)  |                         | - Monster TIM Files     |
|               |                         |                         |
+---------------+                         |                         |
| 320,256       |                         |                         |
| Framebuffer 1 |                         |                         |
| (Disp Buffer) |                         |                         |
|  (320 x 240)  |                         |                         |
|               |                         |                         |
+---------------+-------------------------+-------------------------+
0,512                                                               1024,512


* **Framebuffer 0:** Located at `(0, 0)` to `(320, 240)`.
* **Framebuffer 1:** Located at `(0, 256)` to `(320, 496)`.
* **Offscreen Textures:** Sprite sheets for players and NPCs (sub-block types 13, 14, 15) are cached in offscreen VRAM above coordinate $X = 512$.
* **Color Lookup Tables (CLUTs):** Stored in dedicated palette memory, translating 4-bit or 8-bit indices into 16-bit high-color values.

During map rotation, transformed 3D terrain tiles and billboard quad primitives are written to the GPU's Ordering Table (OT) in back-to-front order. This ensures correct depth sorting, rendering character sprites over background terrain tiles without clipping artifacts.
"""

filename = "ARCH_STUDY_PSX_DQ4_HYBRIDIZATION_FINAL.md"
with open(filename, "w", encoding="utf-8") as f:
    f.write(content)
print(f"File written successfully: {filename}")

Your Markdown document has been formatted and generated:


ARCH_STUDY_PSX_DQ4_HYBRIDIZATION_FINAL
 MD 
Markdown
# Architectural Analysis of the PlayStation Engine Hybridization in Dragon Quest IV

**Document ID:** ARCH-STUDY-PSX-DQ4-HYBRIDIZATION  
**Classification:** Deep-Dive Graphics & Systems Engineering Audit  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Architecture:** Sony PlayStation (`SLPM_869.16` MIPS R3000A + GTE)[cite: 1]  
**Comparative Architecture:** `SLPM_865.00` (*Dragon Quest VII: Eden no Senshitachi*)  
**Primary Container:** `HBD1PS1D.Q41` (CD-ROM Mode 2 Form 1 Archive)[cite: 1]  

---

## 1. Architectural Overview & Design Paradigms

The development of the PlayStation remake of *Dragon Quest IV: Chapters of the Chosen* (`SLPM_869.16`) represents a sophisticated exercise in software engineering and game engine adaptation[cite: 1]. Developed by Heart Beat Inc., the title was constructed by refactoring and adapting the pre-existing, highly complex 3D engine created for *Dragon Quest VII: Eden no Senshitachi* (`SLPM_865.00`). Heart Beat chose to repurpose this 32-bit codebase to reconstruct a game originally built within the constraints of a 16-bit, tile-based architecture.

The technical challenge lay in bridging these distinct design paradigms:
* The engine had to render a fully rotatable, polygon-based isometric world.
* It simultaneously maintained the aesthetic utility of flat, 2D character sprites (billboards) synchronized with a 3D coordinate grid.

To accomplish this, Heart Beat injected customized memory limits, optimized asset tables, and engineered a specialized billboard rendering pipeline directly into the core foundation of the *Dragon Quest VII* architecture.

---

## 2. The Executable Bridge: Disassembly & Memory Adaptation

### Boot-Stage Execution Flow & Event Loop Registration

Static disassembly of the primary executable (`SLPM_869.16`) reveals a structured boot-up sequence that establishes the runtime environment before registering the core execution loops[cite: 1]:
1. When the PlayStation boot-ROM transfers execution to the executable's system entry point, the system runs through standard PsyQ compiler initialization libraries.
2. This process configures the coprocessor registers (specifically the Geometry Transfer Engine, or GTE), initializes the stack pointer, and mounts the CD-ROM subsystem.
3. Control routes immediately to the engine's main initialization function, which starts the primary execution loop.

The main event loop coordinates game state, processes controller inputs, executes the script interpreter, and populates the GPU's primitive drawing lists. It maintains frame synchronization by registering a callback routine with the vertical blank (VSync) interrupt handler, maintaining a strict frame budget of 30 or 60 fields per second.

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
        |     |   Update Camera Coordinates (Rot) |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Sort Primitives (Ordering      |
        |     |        Table & GTE Math)          |
        |     +-----------------------------------+
        |                       |
        |                       v
        |     +-----------------------------------+
        |     |    Push DMA Buffer to GPU VRAM    |
        |     +-----------------------------------+
        |                       |
        +-----------------------+

### BSS Segment Cleansing & Memory Mapping Variations

During initialization, the executable runs a critical memory-clearing routine at `0x8008E284`. This function cleanses the BSS (Block Started by Symbol) segment, zeroing out uninitialized global and static variables to prevent garbage data from corrupting the engine state.

Comparing this routine and the resulting memory configuration against *Dragon Quest VII* (`SLPM_865.00`) reveals how the developers modified memory boundaries to support a hybrid 2D/3D rendering pipeline:

| System Subsystem or Memory Pool | DQVII Memory Allocations (`SLPM_865.00`) | DQIV Remapped Allocations (`SLPM_869.16`) | Functional Modification and Engineering Purpose |
|---|---|---|---|
| **System Boot Entry Point (`start`)** | `0x8008DAC0` | `0x8008DAC0` | Unchanged standard compiler entry point for hardware initialization. |
| **BSS Clearing Routine** | `0x8008DAC0` – `0x8008E1F0` | `0x8008E284` (Entry Target) | Shifted offset to accommodate customized engine initializers. |
| **BGM Sequence Queue (Dynamic Buffer)** | `0x800D25C0` – `0x800D80EF` | `0x800D1500` – `0x800D7500` | Reduced footprint to expand main RAM variable storage. |
| **Waveform Instrument Attributes** | `0x800D80F0` – `0x800DA0F7` | `0x800D7600` – `0x800D9600` | Relocated to align with the customized sound driver footprint. |
| **VRAM Draw/Display Double Buffers** | `0x800B9D10` – `0x800BB6E7` | `0x800BA800` – `0x800BC200` | Expanded page buffers to prevent frame dropping during map rotation. |
| **Sound Interrupt Thread** | `0x8002FA10` – `0x80036FCC` | `0x8002FA10` – `0x80036FCC` | Preserved core driver architecture for sound playback. |
| **Sequence Interpretation Tables** | `0x80025300` – `0x800258E8` | `0x80025300` – `0x800258E8` | Preserved driver mapping configurations for MIDI sequences. |

By shifting the BSS boundaries, Heart Beat expanded the maximum allocation size for sprite lookup tables. Unlike *Dragon Quest VII*, which used simple 3D models for many secondary characters, *Dragon Quest IV* retained classic 2D character designs, requiring constant buffering of multi-directional, highly detailed sprite sheets for players and NPCs.

### Sprite-to-3D-Grid Mapping Routine Initialization

The specific routine responsible for initializing the mapping of 2D sprites onto the 3D isometric grid is located at offset `0x8005C1F0` in program memory. Running immediately after the engine's rendering context is established, it registers a structured array of actor state blocks in memory:
* 3D world position vector ($X, Y, Z$)
* 2D viewport anchor point
* Active rotation angle
* Pointer to the character's active sprite sheet

This initialization routine configures transformation registers on the PlayStation's GTE, defining how the 3D coordinate frame of the terrain grid translates to the screen-space coordinates of the billboard quads. This prevents visual alignment issues and model drifting during scene rendering.

---

## 3. Asset Table Mapping & LBA File Structure

### Archive Format & Sector-Level Layout

All non-executable assets are packed into a single resource archive named `HBD1PS1D.Q41`, structured identically to *Dragon Quest VII*'s `HBD1PS1D.Q71`[cite: 1]. The archive spans 319,436,800 bytes, dividing into 155,975 blocks of 2,048 bytes (matching physical CD-ROM Mode 2 Form 1 sectors).

LBA 0 (offsets `0x00` through `0x800`) holds the volume identification header, containing the ASCII identifier `hbd1ps1d.q41` at position `0x400` (1024). The remaining sectors follow a strictly aligned block system:

+-------------------------------------------------------------+
|                  HBD1PS1D.Q41 Sector Layout                 |
+-------------------------------------------------------------+
| 0x00 Main Block Header (16 Bytes)                          |
|      - Sub-block count (uint32)                             |
|      - Sector size of block (uint32)                        |
|      - Total decompressed raw data length (uint32)          |
|      - Reserved / zero padding (uint32)                     |
+-------------------------------------------------------------+
| 0x10 Sub-Block Headers (16 Bytes per Sub-block)             |
|      - Compressed data size (uint32)                        |
|      - Decompressed data size (uint32)                      |
|      - Offset within sector payload (uint32)                |
|      - Compression Flag (uint16) [e.g., 1280]               |
|      - Sub-Block Type (uint16)                              |
+-------------------------------------------------------------+
| Raw Payload Segment                                         |
|      - Compressed or raw data blocks                        |
|      - Padded with 0x00 to complete 2048-byte boundary      |
+-------------------------------------------------------------+


The compression flag at offset `0x0C` contains decimal `1280` (`0x0500`) when the payload is compressed using Heart Beat's variant of the LZS algorithm. If uncompressed, this flag is set to `0`, and the compressed size equals the decompressed size.

### Sub-Block Type Catalog

| Sub-Block Type Code | Compression Status | Primary Resource Contents | Role and System Integration |
|---|---|---|---|
| **Type 1** | Uncompressed | Font Glyph Sheets | Loaded directly into VRAM to populate the text engine's character cache. |
| **Type 6** | Compressed | Map Chipset Images | Holds textures and 2D tilesets used to construct world and town maps. |
| **Type 8** | Compressed | Monster & Battle Sprites | Standard PlayStation TIM images detailing battle animations and opponent sprites. |
| **Type 10** | Compressed | Multi-TIM Sprite Packages | Multi-directional frames for active combat graphics and effects. |
| **Type 13** | Compressed | NPC & Player Sprites | Multi-directional sheets containing player character and town NPC animation cycles. |
| **Type 21** | Uncompressed | 3D Map Geometry (`qQES`) | Holds the underlying 3D world meshes, polygon definitions, and collision fields. |
| **Type 32** | Uncompressed | Scene Dialogue Index Maps | Translates active script commands to physical text offsets in dialogue files. |
| **Type 39** | Compressed | Primary Dialogue Script Engine | Contains the script instructions and text data[cite: 1]. |

---

## 4. Dialogue Script Parsing & Conditional Control Structures

### Text Block Architecture & Parser State

Dialogue files (Type 39 sub-blocks) are compressed using Huffman coding[cite: 1]. When dialogue triggers, the engine locates the associated script segment via an offset index from Type 32 blocks. The script block starts with a 24-byte header:

+-----------------------------------------------------------------+
|                        Text Block Header                        |
+-----------------------------------------------------------------+
| Offset 0x00: Pointer offset "a" to end of block payload (uint32)|
| Offset 0x04: Unique Scenario Scene ID (uint32)                  |
| Offset 0x08: Offset pointer "c" to Huffman-encoded bitstream    |
| Offset 0x0C: Offset pointer "d" to Huffman Tree structures      |
| Offset 0x10: Offset pointer "e" to end of Huffman bitstream     |
| Offset 0x14: Reserved / padding                                 |
+-----------------------------------------------------------------+


The script execution path is driven by the virtual machine command `C021A0`:
* `C021A0 <offset> <dialogId>`: Directs the interpreter to load a specific dialogue stream, jumping to the designated offset inside the decompressed text buffer.
* `C021A0 <FFF0> <key>`: Evaluates event flags or registers system variables (such as item counts or active party members) to alter the dialogue flow.

### Formatting & Parser Control Codes

| Control Code | Operational Role | Parsing & Blitter Behavior |
|---|---|---|
| `{0000}` | End of Dialogue | Closes the current dialogue stream and terminates the window thread[cite: 1]. |
| `{7F02}` | New Line & Tab | Resets the drawing cursor to the left margin of the next dialogue line. |
| `{7F04} {7Fxx}` | Name Decorator | Instructs interpreter to read the next byte (`7Fxx`) and render the matching actor's name. |
| `{7F0A}` / `{7F0B}` | End-of-Line Signifiers | Displays a blinking cursor and pauses script execution until player input is confirmed[cite: 1]. |
| `{7F15}` | Dynamic Variable Token | Injects dynamic numeric values (such as party gold counts) directly into the text stream. |

### Conditional Scripting Syntax & Multi-Chapter Execution

To manage the game's chapter-based progression without duplicating script assets, Heart Beat implemented an inline conditional branching syntax:

$$\%A\{\text{ID}\}\%X\{\text{TEXT if not ID}\}\%Z\%B\{\text{ID}\}\%X\{\text{TEXT if ID}\}\%Z$$

When the interpreter encounters the `%A` control statement, it evaluates the subsequent character ID against the active party leader's ID:
* If the active leader does **not** match the specified ID, the engine executes the `%X` text block until it reaches `%Z`.
* If the active leader **matches**, the engine skips the `%A` block and executes the `%B` branch.

              +----------------------------------+
              |     Parse dialogue statement     |
              +----------------------------------+
                                |
                                v
              +----------------------------------+
              |    Encounter %A {ID} Operator    |
              +----------------------------------+
                                |
                /-----------------\-----------------\
               /                                     \
              v                                       v
  +-----------------------+               +-----------------------+
  |   Execute %X Branch   |               |   Skip %A %X Block    |
  |  ("TEXT if not ID")   |               +-----------------------+
  +-----------------------+                           |
              |                                       v
              v                           +-----------------------+
  +-----------------------+               |    Evaluate %B {ID}   |
  |   Skip %B %X Block    |               +-----------------------+
  +-----------------------+                           |
              |                                       v
              |                           +-----------------------+
              |                           |   Execute %X Branch   |
              |                           |    ("TEXT if ID")     |
              |                           +-----------------------+
              \                                       /
               \-------------------------------------/
                                |
                                v
                    +---------------------------+
                    |   Resume parsing script   |
                    +---------------------------+

* **Branching Example:**
  * Script Stream: `%A120%XStill, if she were a boy—%Z%B120%XStill, if you were a boy—%Z`
  * Result (Active leader is Alena / ID 120): `"Still, if you were a boy—"`
  * Result (Active leader is Ragnar or Torneko): `"Still, if she were a boy—"`
* **Pluralization Logic (`%H`):** Evaluates whether a designated numeric variable exceeds 1 to conditionally render plural suffixes:
  $$\%H\{\text{Variable ID}\}\%X\%Y\text{s}\%Z$$

---

## 5. Sprite-Grid Synchronization & VRAM Buffer Processing

### Billboard Coordinate Transformation Mathematics

To construct the world map, the engine merges a rotatable 3D polygonal landscape with flat 2D character sprites. The player can rotate the camera in 90-degree increments (with smooth interpolation transitions). To avoid distortion or flattening as the angle approaches 90 degrees, the engine uses two distinct transformation pathways.

Let the coordinate of a 3D grid vertex in world space be:
$$\mathbf{P}_w = \begin{pmatrix} X_w \\ Y_w \\ Z_w \end{pmatrix}$$

Camera orientation is defined by rotation matrix $\mathbf{R}_y(\theta)$ around the vertical Y-axis:
$$\mathbf{R}_y(\theta) = \begin{pmatrix} \cos\theta & 0 & \sin\theta \\ 0 & 1 & 0 \\ -\sin\theta & 0 & \cos\theta \end{pmatrix}$$

Applying camera translation vector $\mathbf{T}$, the 3D grid vertex is transformed into camera-space coordinates $\mathbf{P}_c$:
$$\mathbf{P}_c = \mathbf{R}_y(\theta) \mathbf{P}_w + \mathbf{T}$$

For 2D billboard character sprites, internal quad rotation is bypassed:
1. Extract the sprite's 3D anchor position in world space: $\mathbf{A}_w = [A_x, A_y, A_z]^T$.
2. Transform the anchor into camera space to determine depth and center position:
   $$\mathbf{A}_c = \mathbf{R}_y(\theta) \mathbf{A}_w + \mathbf{T}$$
3. Generate the four quad vertices $\mathbf{V}_c^{(i)}$ ($i \in \{1, 2, 3, 4\}$) by adding local width ($w$) and height ($h$) offsets directly to $\mathbf{A}_c$, keeping $\Delta z = 0$:
   $$\mathbf{V}_c^{(i)} = \mathbf{A}_c + \begin{pmatrix} \Delta x^{(i)} \\ \Delta y^{(i)} \\ 0 \end{pmatrix}$$
   where $\Delta x^{(i)} \in \{-w/2, w/2\}$ and $\Delta y^{(i)} \in \{0, h\}$.
4. The GTE projects camera-space coordinates to 2D screen coordinates $(X_s, Y_s)$ via perspective projection:
   $$X_s = \frac{V_{c,x} \cdot f}{V_{c,z}} + X_0, \quad Y_s = \frac{V_{c,y} \cdot f}{V_{c,z}} + Y_0$$
   where $f$ is focal length and $(X_0, Y_0)$ represents the framebuffer projection center offset.

### VRAM Layout & Page Swapping

To prevent screen tearing and rendering artifacts during active camera rotation, Heart Beat utilized double-buffered page swapping across the $1024 \times 512$ pixel VRAM space:

0,0           320,0                     512,0                     1024,0
+---------------+-------------------------+-------------------------+
|               |                         |                         |
| Framebuffer 0 |                         | Offscreen Textures      |
| (Draw Buffer) |                         | - Character Sheets      |
|  (320 x 240)  |                         | - Monster TIM Files     |
|               |                         |                         |
+---------------+                         |                         |
| 320,256       |                         |                         |
| Framebuffer 1 |                         |                         |
| (Disp Buffer) |                         |                         |
|  (320 x 240)  |                         |                         |
|               |                         |                         |
+---------------+-------------------------+-------------------------+
0,512                                                               1024,512


* **Framebuffer 0:** Located at `(0, 0)` to `(320, 240)`.
* **Framebuffer 1:** Located at `(0, 256)` to `(320, 496)`.
* **Offscreen Textures:** Sprite sheets for players and NPCs (sub-block types 13, 14, 15) are cached in offscreen VRAM above coordinate $X = 512$.
* **Color Lookup Tables (CLUTs):** Stored in dedicated palette memory, translating 4-bit or 8-bit indices into 16-bit high-color values.

During map rotation, transformed 3D terrain tiles and billboard quad primitives are written to the GPU's Ordering Table (OT) in back-to-front order. This ensures correct depth sorting, rendering character sprites over background terrain tiles without clipping artifacts.
