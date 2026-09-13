# Reconstitution of the HeartBeat Engine: Generational Evolution, Packer Archival Schema, and Cross-Platform Localization Cipher

**Document ID:** ARCH-RECON-HBE-PACKER-CIPHER-20260913  
**Classification:** Deep-Dive Archival Specification, Codec Forensics & Localization Protocol  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Architectures:** Nintendo Famicom/NES, Super Famicom/SNES, Sony PlayStation (MIPS R3000A)  
**Target Executables:** `SLPM_869.16` (DQIV PSX), `SLUS-012.06` / `SLUS-013.46` (DWVII US PSX)  
**Archive Containers:** `HBD1PS1D.Q41` & `HBD1PS1D.Q71`  
**Date:** September 13, 2026  

---

**1. Historical Archaeology of the Core Dragon Quest Data Schema**

Preserving and modifying HeartBeat Engine software requires tracing its architectural evolution across three hardware generations. This evolutionary lineage transitions from tightly constrained 8-bit memory structures into modular, 32-bit virtual filesystems.

+-------------------------------------------------------------------------+
|                        DATA EVOLUTIONARY TIMELINE                       |
|                                                                         |
|  8-Bit NES / Famicom      16-Bit SNES / SFC        32-Bit PlayStation   |
|  ------------------->     ----------------->       -------------------> |
|  - Hand-optimized ASM     - Expanded ROM cart      - Unified .HBD image |
|  - Bit-packed save bits   - Shift-JIS / Kanji      - Sub-block modules  |
|  - Hardcoded vectors      - "Resilience" stat      - Huffman bitstreams |
+-------------------------------------------------------------------------+


| Engine Subsystem | 8-Bit NES Generation (*DW I–IV*) | 16-Bit SNES Generation (*DQ V–VI, DQ IIIr*) | 32-Bit PSX HeartBeat Engine Schema (*DQ VII, DQ IVr*) |
|---|---|---|---|
| **Storage Medium** | Discrete ROM cartridge banks (MMC mappers) | Extended ROM cartridge architectures (HiROM / ExHiROM) | CD-ROM Mode 2 Form 1 raw sector archive container (`.HBD`) |
| **Character Encoding** | Single-byte Hiragana / basic katakana tables | Multibyte Shift-JIS (Hiragana, Katakana, early Kanji) | Variable-length tokenized Huffman streams & multi-font metrics |
| **Save State Logic** | "Spell of Restoration" / 1-slot battery SRAM | Multi-slot battery-backed SRAM ($7E bank structures) | Binary serialization committed to Memory Card (`0x047D` / `0x8000F800`) |
| **Asset Organization** | Hardcoded physical assembly offsets | Structured bank-switched tables and pointer vectors | Hierarchical Master-to-Sub-Block sector tree filesystem |
| **Attribute Calculations** | Base defense derived directly from Agility | Dedicated "Resilience" attribute; independent calculations | Multi-class dynamic stat arrays and tactical AI weight matrices |
| **Event Scripting** | Inline assembly branches and hardcoded loops | Conditional jump tables and fixed bank interpreters | Bytecode-interpreted Type 39 LZSS/Huffman event scripts (`C021A0`) |

---

**2. Physical CD-ROM Archival Packer Specification (`.HBD`)**

The primary resource repository for the HeartBeat Engine is packed into a single container: `HBD1PS1D.Q41` (*Dragon Quest IV*) or `HBD1PS1D.Q71` (*Dragon Quest VII*). To optimize CD-ROM seek performance on the PlayStation's $2\times$ optical drive, all assets maintain strict 2,048-byte sector alignment with zero-padded sector terminations (`0x00`).

The absolute physical byte offset $O_{\text{file}}$ of any sector payload on disc is calculated via its Logical Block Address $S_{\text{LBA}}$:
$$O_{\text{file}} = S_{\text{LBA}} \times 2048$$

### Master Block Directory Header (16 Bytes)

Every Master Block directory on disc begins with a 16-byte metadata header starting at physical offset `0x00`:

| Header Offset | Size (Bytes) | Binary Format | Field Name | Verification Rule & Structural Semantics |
|---|---|---|---|---|
| `0x00` | 4 | uint32 (LE) | `sub_block_count` | Total count of child asset sub-blocks contained within the Master Block. |
| `0x04` | 4 | uint32 (LE) | `sector_span` | Total physical allocation span on disc measured in 2,048-byte sectors. |
| `0x08` | 4 | uint32 (LE) | `data_length` | Total unpadded raw byte length of all child sub-blocks combined. |
| `0x0C` | 4 | uint32 (LE) | `null_pad` | Reserved alignment padding; strictly verified as `0x00000000`. |

### Sub-Block Index Headers (16 Bytes per Entry)

Immediately following the Master Block header, an array of $N$ consecutive 16-byte sub-block descriptors begins at offset `0x10`:

| Header Offset | Size (Bytes) | Binary Format | Field Name | Verification Rule & Structural Semantics |
|---|---|---|---|---|
| `0x00` | 4 | uint32 (LE) | `compressed_size` | Physical size of the sub-block payload as stored on disc media. |
| `0x04` | 4 | uint32 (LE) | `decompressed_size` | Absolute allocation size required in Main RAM upon inflation. |
| `0x08` | 4 | uint32 (LE) | `asset_pointer` | Internal ID tag or runtime memory offset used by script dispatchers. |
| `0x0C` | 2 | uint16 (LE) | `compression_flag` | Algorithm indicator: `0` = Uncompressed / Raw; `1280` (`0x0500`) = LZSS compressed. |
| `0x0E` | 2 | uint16 (LE) | `sub_block_type` | Subsystem identifier directing payload routing across engine managers. |

Approximately 75.5% of archive sub-blocks are stored raw (`flag = 0`), while 24.5% use HeartBeat's sliding-window LZSS variant (`flag = 1280`). Compression efficiency is determined by:
$$R_{\text{compression}} = \left( 1 - \frac{S_{\text{compressed}}}{S_{\text{decompressed}}} \right) \times 100\%$$

---

**3. The Huffman Tree Cipher & Script Parser Architecture**

Text dialogue is stored in Type 39 sub-blocks as variable-length bitstreams without raw byte alignment. Decoding is governed by a 24-byte Text Block Header:

| Header Offset | Size (Bytes) | Binary Format | Field Name | Functional Role & Extraction Syntax |
|---|---|---|---|---|
| `0x00` | 4 | uint32 (LE) | `eof_pointer` | Pointer offset "a" marking the end of the text block payload. |
| `0x04` | 4 | uint32 (LE) | `scene_id` | Scenario sequence identifier (e.g., Intro Sequence = `0x0000006C`). |
| `0x08` | 4 | uint32 (LE) | `huffman_start` | Byte pointer "c" marking the beginning of the compressed bitstream. |
| `0x0C` | 4 | uint32 (LE) | `tree_start` | Byte pointer "d" marking the base address of the local Huffman tree structure. |
| `0x10` | 4 | uint32 (LE) | `huffman_end` | Byte pointer "e" marking the terminal boundary of the bitstream. |
| `0x14` | 4 | uint32 (LE) | `reserved` | Structural padding; strictly zero-filled (`0x00000000`). |

### Huffman Tree Traversal Mechanics

The tree initializes with a 2-byte node count prefix (`E3` header). The decoder traverses the tree bit-by-bit from stream offset "c":
* Bit $b_i = 0 \implies$ Branch to Left Child Node.
* Bit $b_i = 1 \implies$ Branch to Right Child Node.

Traversal halts upon reaching a leaf node $L$, resolving to an 8-bit or 16-bit character or control code:
$$L = \text{Branch}(N_{\text{current}}, b_i) \quad \text{where} \quad N_{\text{current}} = \begin{cases} T & \text{if } i = 0 \\ \text{Branch}(N_{\text{current}}, b_{i-1}) & \text{if } N_{\text{current}} \notin \text{Leaves} \end{cases}$$

### Control Code Execution Matrix

| Control Code (Hex) | Operational Meaning | Blitter Behavior & Subsystem Execution Details |
|---|---|---|
| `{0000}` | End of Dialogue | Terminates the active text stream; tears down the text window context. |
| `{7F02}` | Newline & Tab | Executes a carriage return and indents the drawing cursor to the left text margin. |
| `{7F04}` | Name Decorator | Flags name banner formatting or switches character styling parameters. |
| `{7F0A}` | Wait-for-Input Signifier | Pauses parser execution and blinks the cursor until controller confirmation. |
| `{7F0B}` | Timed Line Pause | Alternate line-end signifier used during automated cinematic scroll sequences. |
| `{7F0C}` | Window Clear Cluster | Flushes all glyph primitives from the active window (commonly packed in sextuplets). |
| `{7F15}` | Numeric Interpolation | Dynamically reads and renders system integer variables (e.g., party gold count). |

### Virtual Machine Script Dispatch & Conditional Logic

Type 39 scripts embed bytecode instructions directly into the stream, primarily via the `C021A0` operation:
* `C021A0 <offset> <dialogId>`: Jumps execution to a designated dialogue offset within decompressed RAM.
* `C021A0 <FFF0> <key>`: Evaluates quest flags, inventory tables, or active party member states.

The inline conditional syntax resolves multi-character perspective branches:
* **Character Branching (`%A` / `%B`):** `%A{ID}` executes following block `%X...%Z` if the leader does **not** match `{ID}`; `%B{ID}` executes if the leader **does** match `{ID}`.
* **Pluralization Handling (`%H`):** `%H{ID}%X%Ys%Z` checks the target numeric variable; if value $> 1$, appends the plural suffix "s".

Stream: %A120%XStill, if she were a boy—%Z%B120%XStill, if you were a boy—%Z
Parsed (Leader != Alena ID 120): "Still, if she were a boy—"
Parsed (Leader == Alena ID 120): "Still, if you were a boy—"


---

**4. Proprietary `SEQq` Sound Driver Architecture**

HeartBeat replaced Sony’s default `SEQ`/`VAB` driver libraries with a custom sequencer (`SEQq`, signature `qQES`) that shares ADPCM waveform banks across multiple sequences to conserve memory.

### Sound Block Master Header (60 Bytes)

| Offset | Size (Bytes) | Binary Format | Description & Subsystem Impact |
|---|---|---|---|
| `0x00` | 4 | uint32 (LE) | Sequence payload (`SEQq`) byte size (evaluates to `0` if instrument-only). |
| `0x04` | 2 | uint16 (LE) | Track sequence identifier (`msq_id`). |
| `0x06` | 1 | uint8 | Count of active instrument wave banks allocated (maximum 4). |
| `0x07` | 1 | uint8 | RAM load position index (`0` = loads into constant static BGM address space). |
| `0x08` | 4 | uint32 (LE) | Reserved system padding. |
| `0x0C` | 4 | uint32 (LE) | Wave Bank 1: Compressed PSX-ADPCM waveform data size. |
| `0x10` | 4 | uint32 (LE) | Wave Bank 1: Regional pitch/key mapping array byte size. |
| `0x14` | 2 | uint16 (LE) | Wave Bank 1: Device slot index descriptor. |
| `0x16` | 2 | uint16 (LE) | Wave Bank 1: Parameter configuration flags. |
| `0x18` – `0x23` | 12 | Struct (LE) | Wave Bank 2 Configuration Block (mirrors Bank 1 structure). |
| `0x24` – `0x2F` | 12 | Struct (LE) | Wave Bank 3 Configuration Block (mirrors Bank 1 structure). |
| `0x30` – `0x3B` | 12 | Struct (LE) | Wave Bank 4 Configuration Block (mirrors Bank 1 structure). |

Sequence payloads stream into the BGM driver buffer (`0x800D25C0`–`0x800D80EF`) and execute out of dynamic RAM (`0x80138000`). ADPCM sample banks load directly to SpuRAM at `0x80180000`. Unlike standard MIDI files, `SEQq` streams terminate abruptly with `FF 2F 00` without preceding delta-time parameters.

---

**5. Historical Localization Graft: System-Level Compatibility Matrix**

Grafting the `HBD1PS1D.Q41` asset container from *Dragon Quest IV* onto the North American *Dragon Warrior VII* executable (`SLUS-012.06` / `SLUS-013.46`) historically introduced five systemic failure modes:

+-------------------------------------------------------------------------+
|                       ASSET AND RUNTIME GRAFT FLOW                      |
|                                                                         |
|  [ DQIV Disc Base ]                   [ DW7 US Executable ]             |
|  - HBD1PS1D.Q41 Assets ----+          - SLUS-012.06 Base Code           |
|                            |                         |                  |
|                            v                         v                  |
|                 [ Filename & Header Sync ]    [ MIPS Patch Suite ]      |
|                 - Rename to HBD1PS1D.Q71      - Relocate LBA Table      |
|                 - LBA 0 Sector ID Check       - Dynamic Huffman Hook    |
|                            |                         |                  |
|                            +------------+------------+                  |
|                                         |                               |
|                                         v                               |
|                       [ Hybrid Frankenstein Binary ]                    |
|                       - Subject to Sector Drift & Audio Traps           |
+-------------------------------------------------------------------------+


| Engineering Domain | Graft Requirement / Failure Mode | Technical Mitigation Strategy |
|---|---|---|
| **Volume ID Synchronization** | Executable requests `HBD1PS1D.Q71`; `Q41` causes boot failure. | Rename archive container to `HBD1PS1D.Q71`; update volume string at LBA 0 offset `0x740` to `hdb1ps1d.q71`. |
| **LBA Directory Realignment** | Asset sectors in `Q41` diverge from `Q71` sector maps. | Scan Master Block sector positions; rewrite the hardcoded LBA index table inside the `SLUS` binary. |
| **Huffman Decryption Divergence** | DW7 executable references its internal English Huffman tree. | Hook the decompression routine; redirect the tree pointer to offset `0x0C` of each Type 39 block header. |
| **Dialogue Expansion Allocation** | Expanded English text exceeds pristine compressed byte budgets. | Recalculate Master Block headers; pad Master Blocks with null bytes (`0x00`) to maintain 2,048-byte sector alignment. |
| **Audio Channel Discrepancies** | Instrument tables in DW7 point to mismatched sound banks. | Patch instrument bank size fields at offsets `0x0C`, `0x18`, `0x24`, and `0x30` in the sound format header. |

---

**6. Sovereign Reconstitution Parameter Specifications**

The table below catalogs the configuration parameters governing cross-platform executable alignment:

| Configuration Parameter | Target Binary / Storage Location | Functional Value | Purpose & Architectural Impact |
|---|---|---|---|
| **Archive Target Filename** | CD-ROM Root Directory | `HBD1PS1D.Q71` (or `Q41` native) | Satisfies the filesystem driver's hardcoded volume search string. |
| **System Volume Identifier** | Archive Sector 0, Offset `0x740` | `hdb1ps1d.q71` (ASCII) | Matches internal header validation check during CD-ROM mount loops. |
| **Master LBA Index Map** | Executable RAM Table (`SLUS`/`SLPM`) | Recomputed Sector Offsets | Directs `open_file` routines to valid sector bounds across disc space. |
| **Huffman Root Address** | Type 39 Text Block Header, Offset `0x0C` | Pointer Variable "d" | Allows dynamic, per-scene text decompression without global tree collisions. |
| **Buffer Allocation Limit** | Master Header, Offset `0x08` | Byte Length (`U_L` / `C_L`) | Prevents KSEG0 dynamic heap overruns during real-time asset inflation. |
| **SPU Waveform Base Address** | Sound Hardware Registers | `0x80180000` (Hardware RAM) | Anchors decoded ADPCM voice samples into dedicated Sound RAM buffers. |
