# YAMANA MASTER TID (TEXT & TABLE BLOCK IDENTIFIER) LIBRARY
**Document ID:** HBE-ENG-PSX-MASTER-TID-20260912  
**Architecture:** Sony PlayStation 1 (`SLPM_869.16` / MIPS R3000A @ 33.8688 MHz)  
**Target Engine:** HeartBeat Engine (`HBD1PS1D.Q41` / Manabu Yamana Architecture)  
**Vanilla Disc Source:** `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` (156,487 Sectors / 368,057,424 Bytes)  
**Classification:** Exhaustive Decompiled Text Block (TID) & Sub-Block Subsystem Census  

---

## 1. ARCHITECTURAL ANATOMY OF A YAMANA TID

In the PlayStation 1 HeartBeat Engine (HBE), the **TID (Text Block Identifier / Table Identifier)** is the primary architectural coordinate system through which all 19,193 strings, dialogue trees, facility menus, and dynamic scripts are dispatched.

### 1.1 The 12-Bit TID Addressing Formulation
Every string reference in MIPS machine code or data tables is packed into a single 32-bit unsigned integer:

$$\mathbf{ReferrerWord} = (\mathbf{TID} \ll 20) \mid \Big(\mathbf{BitOffset} + (\mathbf{HTS} \times 8)\Big)$$

Where:
* **$\mathbf{TID}$ ($12\text{ bits}$, Range: `0x0020` to `0x048F`):** Identifies the target text container block.
* **$\mathbf{HTS}$ (Header & Tree Size, bytes):** Offset where the variable-length Huffman bitstream begins.
* **$\mathbf{BitOffset}$:** Bit-level coordinate of the string root node within the bitstream.
* **The 20-Bit Limit ($131,072\text{ bytes}$):** A single TID cannot exceed $2^{20} = 1,048,576\text{ bits}$, imposing an absolute ceiling of $128\text{ KB}$ per block.

### 1.2 The 24-Byte HBD Sub-Block Header
Each sub-block in the HBD archive begins with a strict 24-byte structural header:

```c
struct HbdSubBlockHeader {
    uint32_t flags_unk;       // Offset 0x00: Unknown / alignment flags
    uint16_t tid;             // Offset 0x04: Text Block Identifier (0x0020 - 0x048F)
    uint16_t sub_type;        // Offset 0x06: Sub-block type identifier
    uint32_t text_start_off;  // Offset 0x08 (c): Header + Huffman tree end offset (HTS)
    uint32_t tree_data_end;   // Offset 0x0C (d): Tree data boundary offset
    uint32_t tree_desc_off;   // Offset 0x10 (e): Huffman tree metadata descriptor offset
    uint32_t payload_len;     // Offset 0x14 (f6): Total payload length in bytes
};
```

## 2. DISC-WIDE SUB-BLOCK INVENTORY: 23,828 SUB-BLOCKS ACROSS 3,243 BLOCKS

The vanilla disc houses **3,243 physical blocks** encompassing **23,828 functional sub-blocks**:

| Sub-Block Type | Compression / Flags | Sub-Block Count | Functional Role in HeartBeat Engine |
|---|---|---|---|
| **Type  1** | Raw Binary (0x007E) | **    3 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type  1** | Raw Binary (0x0020) | **    3 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type  6** | LZSS Compressed (0x0500) | ** 1730 sub-blocks** | Sprite Blitter Frames & Character Animations |
| **Type  7** | Raw Binary (0x0000) | ** 1458 sub-blocks** | 16/256-Color Palette CLUT Data |
| **Type  8** | LZSS Compressed (0x0500) | **  309 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type  9** | LZSS Compressed (0x0500) | **  473 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 10** | LZSS Compressed (0x0500) | **  256 sub-blocks** | Raw 32-Bit Pointer Word Tables |
| **Type 11** | LZSS Compressed (0x0500) | **  309 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 12** | Raw Binary (0x0000) | **  309 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 13** | LZSS Compressed (0x0500) | **  970 sub-blocks** | Raw 32-Bit Pointer Word Tables |
| **Type 14** | LZSS Compressed (0x0500) | **   44 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 15** | LZSS Compressed (0x0500) | **    2 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 17** | LZSS Compressed (0x0500) | **    5 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 18** | LZSS Compressed (0x0500) | **    5 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 19** | LZSS Compressed (0x0500) | **  141 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 20** | Raw Binary (0x0000) | **    3 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 21** | Raw Binary (0x0000) | ** 3317 sub-blocks** | 3D Map Geometry, Elevation & Collision Meshes |
| **Type 22** | Raw Binary (0x0000) | **   72 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 23** | Raw Binary (0x0000) | **   44 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 24** | Raw Binary (0x0000) | ** 1062 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 25** | LZSS Compressed (0x0500) | **   27 sub-blocks** | Raw 32-Bit Pointer Word Tables |
| **Type 26** | Raw Binary (0x0000) | **  573 sub-blocks** | Sub-Map Record & Table Referrer Matrices |
| **Type 31** | Raw Binary (0x0000) | ** 1025 sub-blocks** | SPU Sound Sequences & VAG Audio Streams |
| **Type 32** | Raw Binary (0x0000) | **   32 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 34** | Raw Binary (0x0000) | **  975 sub-blocks** | SPU Sound Sequences & VAG Audio Streams |
| **Type 35** | Raw Binary (0x0000) | ** 1576 sub-blocks** | 3D Camera Path, Light Matrices & Entity Placements |
| **Type 36** | Raw Binary (0x0000) | ** 1506 sub-blocks** | 3D Camera Path, Light Matrices & Entity Placements |
| **Type 37** | Raw Binary (0x0000) | ** 1033 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 38** | Raw Binary (0x0000) | ** 1377 sub-blocks** | 3D Camera Path, Light Matrices & Entity Placements |
| **Type 39** | LZSS Compressed (0x0500) | **  922 sub-blocks** | Cutscene Event VM Bytecode Scripts (Type-39) |
| **Type 39** | Raw Binary (0x0000) | **   54 sub-blocks** | Cutscene Event VM Bytecode Scripts (Type-39) |
| **Type 40** | Raw Binary (0x0000) | ** 1315 sub-blocks** | Primary Huffman Compressed Dialogue / Scene Strings |
| **Type 41** | Raw Binary (0x0000) | ** 1730 sub-blocks** | Sprite Blitter Frames & Character Animations |
| **Type 42** | Raw Binary (0x0000) | **  213 sub-blocks** | Secondary Huffman Compressed System & Facility Text |
| **Type 43** | LZSS Compressed (0x0500) | **   24 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 44** | Raw Binary (0x0000) | **  148 sub-blocks** | Roster Parameter & Character Lookup Arrays |
| **Type 44** | LZSS Compressed (0x0500) | **    4 sub-blocks** | Roster Parameter & Character Lookup Arrays |
| **Type 45** | Raw Binary (0x0000) | **  140 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |
| **Type 46** | LZSS Compressed (0x0500) | **  600 sub-blocks** | MIPS R3000A Dynamic Executable Overlay Modules |
| **Type 46** | Raw Binary (0x0000) | **   12 sub-blocks** | MIPS R3000A Dynamic Executable Overlay Modules |
| **Type 47** | Raw Binary (0x0000) | **   27 sub-blocks** | Facility, Sub-Screen & Map Metadata Stream |

---

## 3. MASTER TID CATALOG: ALL 1,111 UNIQUE TIDS

The game defines **1,111 unique Text Block IDs (TIDs)** across 11 major narrative and operational bands:

### 3.1 Macro Functional Band Breakdown

| Band Range | Functional Domain | Unique TIDs | Sequence Capacity (SIDs) | Primary Operational Role |
|---|---|---|---|---|
| **`0x0020 - 0x0026`** | System Top-Level & Overworld Map Index | 7 | 2,298 | World map labels, town names, character naming master |
| **`0x0027 - 0x0099`** | Chapter 1: Ragnar McRyan Arc | 115 | 914 | Burland Castle, Strathross, Loch Tur, Izmit, Healie |
| **`0x009A - 0x0150`** | Chapter 2: Princess Alena Arc | 183 | 1,594 | Zamoksva, Taborov, Vrenor, Desert Bazaar, Endor Tourney |
| **`0x0151 - 0x0210`** | Chapter 3: Torneko Taloon Arc | 190 | 3,077 | Lakanaba, Ballymoral, Endor Shop, Goddess Cave, Tunnel |
| **`0x0211 - 0x02C0`** | Chapter 4: Meena & Maya Arc | 176 | 3,597 | Aubout du Monde, Laissez Faire, Aktemto, Palais de Leon |
| **`0x02C1 - 0x0380`** | Chapter 5: The Chosen Assembly | 192 | 3,898 | Hero Village, Casabranca, Mintos, Parthenia, Femiscyra |
| **`0x0381 - 0x0441`** | Chapter 5/6: Zenithia & Appraisal | 193 | 1,536 | World Tree, Zenithia, Nadiria, Torneko Appraisal 0x3A6-441 |
| **`0x0442 - 0x0473`** | Party Chat Context Banks | 49 | 8,052 variants | 8 Chosen heroes + companions regional dialogue |
| **`0x0474 - 0x0480`** | Church, Save Interface & Sub-Screens | 12 | 1,029 | Memory card I/O, vault, inn, casino, ending records |
| **`0x0481 - 0x048F`** | Resident Menus, Battle & Font 1 | 14 | 3,453 | Battle overlay (048B), Font 1 (048C), Priest EXE (048F) |
| **`Type-39 VM`** | Cutscene Event Script Modules | 403 scripts | 403 bytecode | Event VM scripts with strict LZS slot limits |

### 3.2 Master TID Detailed Registry

| TID (Hex) | TID (Dec) | LBA Sector | Type | HTS (Bytes) | Total Bytes | SIDs (Strings) | Copies | Functional Context / Location |
|---|---|---|---|---|---|---|---|---|
| **`0x0020`** | `  32` | `  4285` | `Type 40` | `  24` | `  1864` | ** 396** | `190` | World Map Coordinate Labels and Town/Dungeon Registry |
| **`0x0021`** | `  33` | `  4285` | `Type 40` | `1076` | ` 93140` | **1087** | `68` | Endor Mega-Block (Commercial Capital, Casino, Immigrant Hub) |
| **`0x0022`** | `  34` | ` 54540` | `Type 40` | ` 276` | `  1832` | **   8** | ` 2` | Endor Castle State Rooms and Royal Audience Chambers |
| **`0x0023`** | `  35` | ` 25775` | `Type 40` | ` 840` | ` 63056` | ** 577** | `79` | Immigrant Town Growth and Evolution (Stages 1 through 5) |
| **`0x0024`** | `  36` | ` 28537` | `Type 40` | ` 944` | ` 22728` | ** 148** | `39` | Special Event NPCs (King Leo, Queen, Eggula and Chikila) |
| **`0x0025`** | `  37` | ` 38513` | `Type 40` | ` 616` | ` 10848` | **  29** | ` 1` | Torneko's Journals, Travel Diaries and Store Ledger |
| **`0x0026`** | `  38` | ` 38513` | `Type 40` | `  24` | `   736` | **  60** | ` 1` | Core Chosen Heroes and Protagonist Naming Master Index |
| **`0x0027`** | `  39` | ` 12923` | `Type 40` | `  24` | `   832` | **   3** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0028`** | `  40` | ` 12887` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0029`** | `  41` | ` 11134` | `Type 40` | `  24` | `  1028` | **   5** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x002A`** | `  42` | ` 10960` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x002B`** | `  43` | ` 10565` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x002C`** | `  44` | ` 10249` | `Type 40` | `  24` | `   984` | **   6** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x002D`** | `  45` | ` 10421` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x002E`** | `  46` | ` 10039` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x002F`** | `  47` | ` 10079` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0030`** | `  48` | ` 10200` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0031`** | `  49` | ` 10163` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0032`** | `  50` | ` 10003` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0033`** | `  51` | `  9957` | `Type 40` | `  24` | `   448` | **   3** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0034`** | `  52` | `  9915` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0035`** | `  53` | ` 11520` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0036`** | `  54` | ` 11637` | `Type 40` | `  24` | `  1608` | **  11** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0037`** | `  55` | ` 11692` | `Type 40` | `  24` | `   256` | **   2** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0038`** | `  56` | ` 11376` | `Type 40` | `  24` | `   752` | **   3** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0039`** | `  57` | `  9628` | `Type 40` | `  24` | `  1120` | **   8** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x003A`** | `  58` | `  9586` | `Type 40` | `  24` | `   748` | **   3** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x003B`** | `  59` | `  9550` | `Type 40` | `  24` | `   224` | **   2** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x003C`** | `  60` | `  9729` | `Type 40` | `  24` | `   496` | **   4** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x003D`** | `  61` | `  9366` | `Type 40` | `  24` | `   576` | **   3** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x003E`** | `  62` | `  9078` | `Type 40` | `  24` | `  1080` | **   5** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x003F`** | `  63` | `  9117` | `Type 40` | `  24` | `   696` | **   3** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0040`** | `  64` | `  9462` | `Type 40` | `  24` | `  1308` | **  10** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0041`** | `  65` | `  9405` | `Type 40` | `  24` | `  2512` | **  33** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0042`** | `  66` | ` 17950` | `Type 40` | `  24` | `   940` | **   4** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0043`** | `  67` | ` 17908` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0044`** | `  68` | ` 18278` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0045`** | `  69` | ` 17871` | `Type 40` | `  24` | `   400` | **   2** | ` 1` | Ch.1: Burland Castle, King's Quest and Royal Barracks |
| **`0x0046`** | `  70` | ` 17480` | `Type 40` | `  24` | `   636` | **   4** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0047`** | `  71` | ` 17334` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0048`** | `  72` | ` 17606` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0049`** | `  73` | ` 17221` | `Type 40` | `  24` | `   576` | **   5** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x004A`** | `  74` | ` 17257` | `Type 40` | `  24` | `  1088` | **   5** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x004B`** | `  75` | ` 18742` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x004C`** | `  76` | ` 17110` | `Type 40` | `  24` | `   432` | **   4** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x004D`** | `  77` | `  8557` | `Type 40` | `  24` | `   220` | **   2** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x004E`** | `  78` | `  8721` | `Type 40` | `  24` | `   444` | **   2** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x004F`** | `  79` | `  8359` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0050`** | `  80` | `  6687` | `Type 40` | `  24` | `   896` | **   7** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0051`** | `  81` | `  6491` | `Type 40` | `  24` | `   500` | **   2** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0052`** | `  82` | `  6450` | `Type 40` | `  24` | `   512` | **   4** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0053`** | `  83` | `  6409` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0054`** | `  84` | `  7785` | `Type 40` | `  24` | `   612` | **   4** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0055`** | `  85` | `  7428` | `Type 40` | `  24` | `   532` | **   3** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0056`** | `  86` | `  7464` | `Type 40` | `  24` | `   224` | **   2** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0057`** | `  87` | `  7346` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0058`** | `  88` | `  7305` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0059`** | `  89` | `  7174` | `Type 40` | `  24` | `   960` | **   5** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x005A`** | `  90` | `  9021` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x005B`** | `  91` | `  8960` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x005C`** | `  92` | `  6786` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x005D`** | `  93` | ` 13704` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x005E`** | `  94` | ` 13584` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x005F`** | `  95` | ` 13546` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0060`** | `  96` | ` 13620` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0061`** | `  97` | ` 16041` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0062`** | `  98` | ` 15654` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0063`** | `  99` | ` 14999` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0064`** | ` 100` | ` 14963` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0065`** | ` 101` | `107247` | `Type 40` | `  24` | `   684` | **   2** | ` 1` | Ch.1: Strathross Town, Loch Tur and Izmit Village |
| **`0x0066`** | ` 102` | `107143` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0067`** | ` 103` | `107066` | `Type 40` | `  24` | `  1920` | **  14** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0068`** | ` 104` | `107026` | `Type 40` | `  24` | `   600` | **   3** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0069`** | ` 105` | `106980` | `Type 40` | `  24` | `  1524` | **  14** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x006A`** | ` 106` | `106943` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x006B`** | ` 107` | `106908` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x006C`** | ` 108` | `106864` | `Type 40` | `  24` | `  1680` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x006D`** | ` 109` | ` 59197` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x006E`** | ` 110` | ` 59801` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x006F`** | ` 111` | ` 59971` | `Type 40` | `  24` | `   776` | **   6** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0070`** | ` 112` | ` 59155` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0071`** | ` 113` | ` 58776` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0072`** | ` 114` | ` 58585` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0073`** | ` 115` | ` 58539` | `Type 40` | `  24` | `   452` | **   4** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0074`** | ` 116` | ` 58498` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0075`** | ` 117` | ` 59014` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0076`** | ` 118` | ` 60624` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0077`** | ` 119` | ` 61006` | `Type 40` | `  24` | `   256` | **   2** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0078`** | ` 120` | ` 60481` | `Type 40` | `  24` | `   772` | **   3** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0079`** | ` 121` | ` 21846` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x007A`** | ` 122` | ` 21806` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x007B`** | ` 123` | ` 21770` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x007C`** | ` 124` | ` 21465` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x007D`** | ` 125` | ` 21427` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x007E`** | ` 126` | ` 20903` | `Type 40` | `  24` | `   140` | **   2** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x007F`** | ` 127` | ` 21927` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0080`** | ` 128` | ` 21887` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Sarokhov Cave, Healie Recruitment and Flying Shoes |
| **`0x0081`** | ` 129` | ` 57065` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0082`** | ` 130` | ` 48188` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0083`** | ` 131` | ` 48051` | `Type 40` | `  24` | `  1108` | **  10** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0084`** | ` 132` | ` 56562` | `Type 40` | `  24` | `   744` | **   5** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0085`** | ` 133` | ` 56827` | `Type 40` | `  24` | `   100` | **  12** | ` 2` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0086`** | ` 134` | `106685` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0087`** | ` 135` | `106701` | `Type 40` | `  24` | `  1568` | **   5** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0088`** | ` 136` | `106718` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0089`** | ` 137` | `106734` | `Type 40` | `  24` | `  1568` | **   5** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x008A`** | ` 138` | `106751` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x008B`** | ` 139` | `106767` | `Type 40` | `  24` | `  1568` | **   5** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x008C`** | ` 140` | `106784` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x008D`** | ` 141` | `106800` | `Type 40` | `  24` | `  1568` | **   5** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x008E`** | ` 142` | `106817` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x008F`** | ` 143` | `106833` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0090`** | ` 144` | ` 46096` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0091`** | ` 145` | ` 46508` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0092`** | ` 146` | ` 46618` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0093`** | ` 147` | ` 43942` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0094`** | ` 148` | ` 44368` | `Type 40` | `  24` | `   220` | **   2** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0095`** | ` 149` | ` 43982` | `Type 40` | `  24` | `   444` | **   2** | ` 2` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0096`** | ` 150` | ` 43653` | `Type 40` | `  24` | `  1104` | **   8** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0097`** | ` 151` | ` 43491` | `Type 40` | `  24` | `   120` | **   2** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0098`** | ` 152` | ` 43699` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0099`** | ` 153` | ` 45520` | `Type 40` | `  24` | `   236` | **   3** | ` 1` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x009A`** | ` 154` | ` 45483` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x009B`** | ` 155` | ` 40054` | `Type 40` | `  24` | `   576` | **   4** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x009C`** | ` 156` | ` 39599` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x009D`** | ` 157` | ` 39408` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x009E`** | ` 158` | ` 39367` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x009F`** | ` 159` | ` 22532` | `Type 40` | `  24` | `   700` | **   4** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A0`** | ` 160` | ` 22925` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A1`** | ` 161` | ` 25395` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A2`** | ` 162` | ` 20332` | `Type 40` | `  24` | `   100` | **  12** | ` 2` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A3`** | ` 163` | ` 19959` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A4`** | ` 164` | ` 19397` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A5`** | ` 165` | ` 19260` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A6`** | ` 166` | ` 40090` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A7`** | ` 167` | ` 52340` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A8`** | ` 168` | ` 41569` | `Type 40` | `  24` | `   476` | **   3** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A9`** | ` 169` | ` 41655` | `Type 40` | `  24` | `   100` | **  12** | ` 2` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00AA`** | ` 170` | ` 46711` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00AB`** | ` 171` | ` 46774` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00AC`** | ` 172` | ` 23869` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00AD`** | ` 173` | ` 47834` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00AE`** | ` 174` | ` 47574` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00AF`** | ` 175` | `  6085` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B0`** | ` 176` | `  3535` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B1`** | ` 177` | `  3346` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B2`** | ` 178` | `  3425` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B3`** | ` 179` | `  3454` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B4`** | ` 180` | `  4285` | `Type 40` | `  24` | `   100` | **  12** | ` 2` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B5`** | ` 181` | `  4101` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B6`** | ` 182` | `  4067` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B7`** | ` 183` | `  1403` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B8`** | ` 184` | ` 40983` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B9`** | ` 185` | ` 44568` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00BA`** | ` 186` | `  1038` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00BB`** | ` 187` | `  1898` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00BC`** | ` 188` | `  2649` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00BD`** | ` 189` | `  2075` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00BE`** | ` 190` | `  1217` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00BF`** | ` 191` | `  1310` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00C0`** | ` 192` | `108626` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00C1`** | ` 193` | `108575` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C2`** | ` 194` | `108677` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C3`** | ` 195` | `108707` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C4`** | ` 196` | `  2551` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C5`** | ` 197` | `  2476` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C6`** | ` 198` | `  2354` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C7`** | ` 199` | ` 30747` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C8`** | ` 200` | ` 30971` | `Type 40` | `  24` | `   768` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00C9`** | ` 201` | ` 30864` | `Type 40` | `  24` | `   792` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00CA`** | ` 202` | ` 30628` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00CB`** | ` 203` | ` 31079` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00CC`** | ` 204` | ` 29849` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00CD`** | ` 205` | ` 29925` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00CE`** | ` 206` | ` 30342` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00CF`** | ` 207` | ` 30038` | `Type 40` | `  24` | `   300` | **   2** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D0`** | ` 208` | ` 30205` | `Type 40` | `  24` | `   812` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D1`** | ` 209` | ` 29680` | `Type 40` | `  24` | `   144` | **   2** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D2`** | ` 210` | ` 29780` | `Type 40` | `  24` | `   320` | **   2** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D3`** | ` 211` | ` 29263` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D4`** | ` 212` | ` 29485` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D5`** | ` 213` | ` 29378` | `Type 40` | `  24` | `  1072` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D6`** | ` 214` | ` 28987` | `Type 40` | `  24` | `   428` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D7`** | ` 215` | ` 28870` | `Type 40` | `  24` | `  1080` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D8`** | ` 216` | ` 28537` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00D9`** | ` 217` | ` 28613` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00DA`** | ` 218` | ` 28710` | `Type 40` | `  24` | `   312` | **   2** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00DB`** | ` 219` | ` 28422` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00DC`** | ` 220` | ` 28307` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00DD`** | ` 221` | ` 27914` | `Type 40` | `  24` | `   780` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00DE`** | ` 222` | ` 27807` | `Type 40` | `  24` | `  1072` | **   3** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00DF`** | ` 223` | ` 28022` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00E0`** | ` 224` | ` 27692` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Taborov Sacrifice, Master Borya and Priest Kiryl |
| **`0x00E1`** | ` 225` | ` 27577` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E2`** | ` 226` | ` 27297` | `Type 40` | `  24` | `   780` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E3`** | ` 227` | ` 27190` | `Type 40` | `  24` | `  1224` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E4`** | ` 228` | ` 27081` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E5`** | ` 229` | ` 26966` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E6`** | ` 230` | ` 26692` | `Type 40` | `  24` | `   512` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E7`** | ` 231` | ` 26585` | `Type 40` | `  24` | `  1224` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E8`** | ` 232` | ` 26442` | `Type 40` | `  24` | `  3204` | **  15** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00E9`** | ` 233` | ` 26334` | `Type 40` | `  24` | `   512` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00EA`** | ` 234` | ` 26228` | `Type 40` | `  24` | `   588` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00EB`** | ` 235` | ` 25986` | `Type 40` | `  24` | `   588` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00EC`** | ` 236` | ` 25775` | `Type 40` | `  24` | `  6780` | **  23** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00ED`** | ` 237` | ` 37736` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00EE`** | ` 238` | ` 37334` | `Type 40` | `  24` | `   764` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00EF`** | ` 239` | ` 37227` | `Type 40` | `  24` | `   920` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F0`** | ` 240` | ` 37442` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F1`** | ` 241` | ` 36891` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F2`** | ` 242` | ` 38379` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F3`** | ` 243` | ` 38267` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F4`** | ` 244` | ` 38155` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F5`** | ` 245` | ` 37989` | `Type 40` | `  24` | `   300` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F6`** | ` 246` | ` 37852` | `Type 40` | `  24` | `   740` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F7`** | ` 247` | ` 36971` | `Type 40` | `  24` | `   320` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F8`** | ` 248` | ` 37059` | `Type 40` | `  24` | `   144` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00F9`** | ` 249` | ` 37158` | `Type 40` | `  24` | `   320` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00FA`** | ` 250` | ` 32208` | `Type 40` | `  24` | `   144` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00FB`** | ` 251` | ` 32413` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00FC`** | ` 252` | ` 32483` | `Type 40` | `  24` | `  1292` | **   8** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00FD`** | ` 253` | ` 32620` | `Type 40` | `  24` | `   600` | **   5** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00FE`** | ` 254` | ` 32307` | `Type 40` | `  24` | `   500` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x00FF`** | ` 255` | ` 33203` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0100`** | ` 256` | ` 33133` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0101`** | ` 257` | ` 33026` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0102`** | ` 258` | ` 32870` | `Type 40` | `  24` | `   648` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0103`** | ` 259` | ` 31159` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0104`** | ` 260` | ` 31327` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0105`** | ` 261` | ` 31397` | `Type 40` | `  24` | `   252` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0106`** | ` 262` | ` 32008` | `Type 40` | `  24` | `   464` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0107`** | ` 263` | ` 31890` | `Type 40` | `  24` | `   288` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0108`** | ` 264` | ` 31780` | `Type 40` | `  24` | `   820` | **   3** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0109`** | ` 265` | ` 31676` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x010A`** | ` 266` | ` 33807` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x010B`** | ` 267` | ` 33702` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x010C`** | ` 268` | ` 33434` | `Type 40` | `  24` | `  1040` | **   4** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x010D`** | ` 269` | ` 34542` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x010E`** | ` 270` | ` 33967` | `Type 40` | `  24` | `   144` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x010F`** | ` 271` | ` 34066` | `Type 40` | `  24` | `   328` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0110`** | ` 272` | ` 34422` | `Type 40` | `  24` | `   412` | **   2** | ` 1` | Ch.2: Vrenor Town, Thief Armlet and Desert Bazaar |
| **`0x0111`** | ` 273` | ` 34288` | `Type 40` | `  24` | `  2004` | **   3** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0112`** | ` 274` | ` 34622` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0113`** | ` 275` | ` 34703` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0114`** | ` 276` | ` 34790` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0115`** | ` 277` | ` 34987` | `Type 40` | `  24` | `   284` | **   2** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0116`** | ` 278` | ` 34917` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0117`** | ` 279` | ` 36169` | `Type 40` | `  24` | `   412` | **   2** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0118`** | ` 280` | ` 36096` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0119`** | ` 281` | ` 35990` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x011A`** | ` 282` | ` 35886` | `Type 40` | `  24` | `   344` | **   2** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x011B`** | ` 283` | ` 35778` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x011C`** | ` 284` | ` 35635` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x011D`** | ` 285` | ` 35513` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x011E`** | ` 286` | ` 35392` | `Type 40` | `  24` | `   508` | **   3** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x011F`** | ` 287` | ` 35233` | `Type 40` | `  24` | `   652` | **   4** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0120`** | ` 288` | ` 36806` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0121`** | ` 289` | ` 36714` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0122`** | ` 290` | ` 36592` | `Type 40` | `  24` | `   376` | **   2** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0123`** | ` 291` | ` 36504` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0124`** | ` 292` | ` 63931` | `Type 40` | `  24` | `  6984` | ** 110** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0125`** | ` 293` | `107308` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0126`** | ` 294` | `107286` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0127`** | ` 295` | `108420` | `Type 40` | `  24` | `   480` | **   3** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0128`** | ` 296` | `108497` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0129`** | ` 297` | `108297` | `Type 40` | `  24` | `  1036` | **   8** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x012A`** | ` 298` | `108346` | `Type 40` | `  24` | `  1064` | **  13** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x012B`** | ` 299` | `108226` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x012C`** | ` 300` | `108182` | `Type 40` | `  24` | `  1004` | **  11** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x012D`** | ` 301` | `107983` | `Type 40` | `  24` | `  1024` | **   7** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x012E`** | ` 302` | `107582` | `Type 40` | `  24` | `   308` | **   3** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x012F`** | ` 303` | `107678` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0130`** | ` 304` | `107818` | `Type 40` | `  24` | `   216` | **   2** | ` 1` | Ch.2: Birdsong Tower, Elven Nectar and Desert Shrine |
| **`0x0131`** | ` 305` | `  1537` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0132`** | ` 306` | `108735` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0133`** | ` 307` | `114829` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0134`** | ` 308` | ` 65727` | `Type 40` | `  24` | `   408` | **   8** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0135`** | ` 309` | ` 65670` | `Type 40` | `  24` | `  1912` | **  22** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0136`** | ` 310` | ` 54589` | `Type 40` | `  24` | `  1232` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0137`** | ` 311` | `  5646` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0138`** | ` 312` | `  5546` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0139`** | ` 313` | `  5510` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x013A`** | ` 314` | `  5450` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x013B`** | ` 315` | `  5415` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x013C`** | ` 316` | `  5380` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x013D`** | ` 317` | `  5289` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x013E`** | ` 318` | `  5257` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x013F`** | ` 319` | `  5222` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0140`** | ` 320` | `  5186` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0141`** | ` 321` | `  5147` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0142`** | ` 322` | `  5004` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0143`** | ` 323` | ` 46938` | `Type 40` | `  24` | `   124` | **   2** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0144`** | ` 324` | ` 49275` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0145`** | ` 325` | ` 49353` | `Type 40` | `  24` | `   604` | **   3** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0146`** | ` 326` | ` 54418` | `Type 40` | `  24` | `   840` | **   5** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0147`** | ` 327` | ` 54457` | `Type 40` | `  24` | `   516` | **   4** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0148`** | ` 328` | ` 54002` | `Type 40` | `  24` | `   100` | **  12** | ` 2` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0149`** | ` 329` | ` 53885` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014A`** | ` 330` | ` 53571` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014B`** | ` 331` | ` 53056` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014C`** | ` 332` | ` 53092` | `Type 40` | `  24` | `   428` | **   4** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014D`** | ` 333` | ` 53128` | `Type 40` | `  24` | `  1032` | **   4** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014E`** | ` 334` | ` 42478` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014F`** | ` 335` | ` 48492` | `Type 40` | `  24` | `   100` | **  12** | ` 2` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0150`** | ` 336` | ` 55220` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0151`** | ` 337` | ` 54797` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0152`** | ` 338` | ` 54753` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0153`** | ` 339` | ` 54699` | `Type 40` | `  24` | `  1864` | **  13** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0154`** | ` 340` | ` 54646` | `Type 40` | `  24` | `   372` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0155`** | ` 341` | ` 55059` | `Type 40` | `  24` | `   684` | **   7** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0156`** | ` 342` | ` 55170` | `Type 40` | `  24` | `   620` | **   2** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0157`** | ` 343` | ` 54968` | `Type 40` | `  24` | `  3016` | **  17** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0158`** | ` 344` | ` 55112` | `Type 40` | `  24` | `  1656` | **  13** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x015A`** | ` 346` | `  1668` | `Type 40` | `  24` | `   872` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x015B`** | ` 347` | `107437` | `Type 40` | `  24` | `  1128` | **   4** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x015C`** | ` 348` | `107516` | `Type 40` | `  24` | `   976` | **   7** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x015D`** | ` 349` | `  1768` | `Type 40` | `  24` | `   908` | **   5** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x015E`** | ` 350` | `107881` | `Type 40` | `  24` | `  1712` | **  17** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x015F`** | ` 351` | `  1987` | `Type 40` | `  24` | `   908` | **   6** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0160`** | ` 352` | ` 10517` | `Type 40` | `  24` | `   652` | **   4** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0161`** | ` 353` | ` 18668` | `Type 40` | `  24` | `   288` | **   4** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0162`** | ` 354` | ` 17517` | `Type 40` | `  24` | `   496` | **  10** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0163`** | ` 355` | ` 53167` | `Type 40` | `  24` | `   496` | **  10** | ` 2` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0164`** | ` 356` | ` 10601` | `Type 40` | `  24` | `   576` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0165`** | ` 357` | ` 59233` | `Type 40` | `  24` | `   576` | **   3** | ` 2` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0166`** | ` 358` | ` 10698` | `Type 40` | `  24` | `   808` | **   5** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0167`** | ` 359` | ` 59539` | `Type 40` | `  24` | `   808` | **   5** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0168`** | ` 360` | ` 16077` | `Type 40` | `  24` | `  1348` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0169`** | ` 361` | ` 10997` | `Type 40` | `  24` | `  1348` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x016A`** | ` 362` | `  2229` | `Type 40` | `  24` | `   640` | **   6** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x016B`** | ` 363` | ` 13400` | `Type 40` | `  24` | `   492` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x016C`** | ` 364` | `  9502` | `Type 40` | `  24` | `  2864` | **  12** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x016D`** | ` 365` | `108052` | `Type 40` | `  24` | `  1164` | **   8** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x016E`** | ` 366` | `108093` | `Type 40` | `  24` | `   176` | **   3** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x016F`** | ` 367` | `  7382` | `Type 40` | `  24` | `  1176` | **   4** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0170`** | ` 368` | ` 39444` | `Type 40` | `  24` | `  1176` | **   4** | ` 1` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0171`** | ` 369` | ` 39799` | `Type 40` | `  24` | `   704` | **   3** | ` 2` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0172`** | ` 370` | `  8094` | `Type 40` | `  24` | `   680` | **   4** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0173`** | ` 371` | ` 46583` | `Type 40` | `  24` | `   244` | **   3** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0174`** | ` 372` | `  8130` | `Type 40` | `  24` | `   284` | **   3** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0175`** | ` 373` | ` 45758` | `Type 40` | `  24` | `   284` | **   3** | ` 2` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0176`** | ` 374` | `  8388` | `Type 40` | `  24` | `  1532` | **   4** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0177`** | ` 375` | ` 46223` | `Type 40` | `  24` | `  1532` | **   4** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0178`** | ` 376` | ` 46264` | `Type 40` | `  24` | `   624` | **   4** | ` 2` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0179`** | ` 377` | ` 48923` | `Type 40` | `  24` | `  2244` | **   4** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017A`** | ` 378` | ` 24211` | `Type 40` | `  24` | `  1516` | **   3** | ` 2` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017B`** | ` 379` | ` 17701` | `Type 40` | `  24` | `   520` | **   7** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017C`** | ` 380` | ` 12849` | `Type 40` | `  24` | `   264` | **   3** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017D`** | ` 381` | ` 61631` | `Type 40` | `  24` | `   264` | **   3** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017E`** | ` 382` | ` 52079` | `Type 40` | `  24` | `   820` | **   7** | ` 2` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017F`** | ` 383` | ` 50953` | `Type 40` | `  24` | `   284` | **   3** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0180`** | ` 384` | ` 38455` | `Type 40` | `  24` | `   136` | **   3** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0181`** | ` 385` | ` 29592` | `Type 40` | `  24` | `   440` | **   4** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0182`** | ` 386` | ` 32120` | `Type 40` | `  24` | `   260` | **   4** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0183`** | ` 387` | `113947` | `Type 40` | `  24` | `  1616` | **   9** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0184`** | ` 388` | ` 14136` | `Type 40` | `  24` | ` 10616` | **  89** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0185`** | ` 389` | ` 13111` | `Type 40` | `  24` | ` 11208` | ** 117** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0186`** | ` 390` | ` 13035` | `Type 40` | `  24` | ` 10560` | ** 109** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0187`** | ` 391` | ` 12958` | `Type 40` | `  24` | `  5140` | **  54** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0188`** | ` 392` | ` 13228` | `Type 40` | `  24` | `  2864` | **  24** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0189`** | ` 393` | ` 13440` | `Type 40` | `  24` | `  8344` | **  80** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x018A`** | ` 394` | ` 13662` | `Type 40` | `  24` | `  1868` | **  13** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x018B`** | ` 395` | ` 13330` | `Type 40` | `  24` | `   976` | **   9** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x018C`** | ` 396` | ` 13294` | `Type 40` | `  24` | `  1188` | **   9** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x018D`** | ` 397` | ` 13876` | `Type 40` | `  24` | `  5676` | **  48** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x018E`** | ` 398` | ` 13971` | `Type 40` | `  24` | `   744` | **   5** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x018F`** | ` 399` | ` 13746` | `Type 40` | `  24` | `  2460` | **  15** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0190`** | ` 400` | ` 13787` | `Type 40` | `  24` | `  2508` | **  11** | ` 1` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0191`** | ` 401` | ` 14011` | `Type 40` | `  24` | `  2636` | **  23** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0192`** | ` 402` | ` 13825` | `Type 40` | `  24` | `  3212` | **  25** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0193`** | ` 403` | ` 14071` | `Type 40` | `  24` | `  2296` | **  15** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0194`** | ` 404` | ` 14395` | `Type 40` | `  24` | ` 11320` | ** 113** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0195`** | ` 405` | ` 14282` | `Type 40` | `  24` | `   572` | **   3** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0196`** | ` 406` | ` 14323` | `Type 40` | `  24` | `  1092` | **   8** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0197`** | ` 407` | ` 14360` | `Type 40` | `  24` | `  1316` | **   7** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0198`** | ` 408` | ` 14525` | `Type 40` | `  24` | `  3100` | **  20** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x0199`** | ` 409` | ` 14561` | `Type 40` | `  24` | `  2616` | **  22** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x019A`** | ` 410` | ` 14607` | `Type 40` | `  24` | `  2680` | **  25** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x019B`** | ` 411` | ` 16822` | `Type 40` | `  24` | `  2520` | **  19** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x019C`** | ` 412` | ` 16868` | `Type 40` | `  24` | `  2456` | **  17** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x019E`** | ` 414` | ` 16302` | `Type 40` | `  24` | `  3112` | **  22** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x019F`** | ` 415` | ` 14696` | `Type 40` | `  24` | `  6980` | **  72** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A0`** | ` 416` | ` 14778` | `Type 40` | `  24` | `   520` | **   9** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A1`** | ` 417` | ` 14654` | `Type 40` | `  24` | `  1688` | **  12** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A2`** | ` 418` | ` 16507` | `Type 40` | `  24` | `  1620` | **  12** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A3`** | ` 419` | ` 16551` | `Type 40` | `  24` | `  1184` | **  11** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A4`** | ` 420` | ` 16597` | `Type 40` | `  24` | `  1248` | **  10** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A5`** | ` 421` | ` 16466` | `Type 40` | `  24` | `  1332` | **  10** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A6`** | ` 422` | ` 16652` | `Type 40` | `  24` | `  1108` | **   9** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A7`** | ` 423` | ` 16707` | `Type 40` | `  24` | `  1108` | **   9** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A8`** | ` 424` | ` 16757` | `Type 40` | `  24` | `  1648` | **  20** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01A9`** | ` 425` | ` 16351` | `Type 40` | `  24` | `   976` | **   6** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01AA`** | ` 426` | ` 16389` | `Type 40` | `  24` | `  1320` | **  12** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01AB`** | ` 427` | ` 16426` | `Type 40` | `  24` | `  2716` | **  22** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01AC`** | ` 428` | ` 15916` | `Type 40` | `  24` | `  8580` | **  85** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01AD`** | ` 429` | ` 15690` | `Type 40` | `  24` | `   628` | **   4** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01AE`** | ` 430` | ` 15786` | `Type 40` | `  24` | `  1672` | **  10** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01AF`** | ` 431` | ` 15741` | `Type 40` | `  24` | `  1760` | **  11** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01B0`** | ` 432` | ` 15874` | `Type 40` | `  24` | `   268` | **   3** | ` 1` | Ch.3: Endor Shop Purchase, Princess Flora and King Endor |
| **`0x01B1`** | ` 433` | ` 15834` | `Type 40` | `  24` | `  2048` | **   7** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B2`** | ` 434` | ` 16199` | `Type 40` | `  24` | `  2424` | **  24** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B3`** | ` 435` | ` 16253` | `Type 40` | `  24` | `  3088` | **  22** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B4`** | ` 436` | ` 16119` | `Type 40` | `  24` | `  3524` | **  24** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B5`** | ` 437` | ` 15330` | `Type 40` | `  24` | `  2632` | **  19** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B6`** | ` 438` | ` 15420` | `Type 40` | `  24` | `  7184` | **  57** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B7`** | ` 439` | ` 15503` | `Type 40` | `  24` | `   240` | **   3** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B8`** | ` 440` | ` 15541` | `Type 40` | `  24` | `  8124` | **  57** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01B9`** | ` 441` | ` 15606` | `Type 40` | `  24` | `  1408` | **   9** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01BA`** | ` 442` | ` 14820` | `Type 40` | `  24` | `  3760` | **  34** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01BB`** | ` 443` | ` 14874` | `Type 40` | `  24` | `  1052` | **   7** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01BC`** | ` 444` | ` 14916` | `Type 40` | `  24` | `  1880` | **  14** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01BD`** | ` 445` | ` 15174` | `Type 40` | `  24` | `  1880` | **  13** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01BE`** | ` 446` | ` 15039` | `Type 40` | `  24` | `  4292` | **  49** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01BF`** | ` 447` | ` 15233` | `Type 40` | `  24` | `  2436` | **  17** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C0`** | ` 448` | `  9862` | `Type 40` | `  24` | `  2136` | **  11** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C1`** | ` 449` | ` 10338` | `Type 40` | `  24` | `  5560` | **  28** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C2`** | ` 450` | ` 10459` | `Type 40` | `  24` | `  6196` | **  26** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C3`** | ` 451` | ` 11421` | `Type 40` | `  24` | `  3792` | **  25** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C4`** | ` 452` | ` 11556` | `Type 40` | `  24` | `  2432` | **  16** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C5`** | ` 453` | ` 11735` | `Type 40` | `  24` | `  1168` | **   5** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C6`** | ` 454` | ` 11778` | `Type 40` | `  24` | `   964` | **   6** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C7`** | ` 455` | ` 11819` | `Type 40` | `  24` | `   964` | **   6** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C8`** | ` 456` | ` 11848` | `Type 40` | `  24` | `  3016` | **  20** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01C9`** | ` 457` | ` 11961` | `Type 40` | `  24` | `  3008` | **  20** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01CA`** | ` 458` | ` 12006` | `Type 40` | `  24` | `  3008` | **  20** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01CB`** | ` 459` | ` 12081` | `Type 40` | `  24` | `  3476` | **  23** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01CC`** | ` 460` | ` 12115` | `Type 40` | `  24` | `  3576` | **  23** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01CD`** | ` 461` | ` 12178` | `Type 40` | `  24` | `  3356` | **  20** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01CE`** | ` 462` | ` 12272` | `Type 40` | `  24` | `  3472` | **  22** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01CF`** | ` 463` | ` 12400` | `Type 40` | `  24` | `  3356` | **  20** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01D0`** | ` 464` | ` 12448` | `Type 40` | `  24` | `  4380` | **  29** | ` 1` | Ch.3: Goddess Statue Cave, Silver Goddess and Fox Village |
| **`0x01D1`** | ` 465` | ` 12580` | `Type 40` | `  24` | `  3356` | **  20** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D2`** | ` 466` | ` 12645` | `Type 40` | `  24` | `  3356` | **  20** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D3`** | ` 467` | ` 12705` | `Type 40` | `  24` | `  3520` | **  18** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D4`** | ` 468` | ` 12748` | `Type 40` | `  24` | `  2768` | **  15** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D5`** | ` 469` | ` 11238` | `Type 40` | `  24` | `  2128` | **  18** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D6`** | ` 470` | ` 10653` | `Type 40` | `  24` | `  1984` | **  11** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D7`** | ` 471` | ` 10790` | `Type 40` | `  24` | `  3396` | **  20** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D8`** | ` 472` | ` 10746` | `Type 40` | `  24` | `  4380` | **  21** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01D9`** | ` 473` | ` 10837` | `Type 40` | `  24` | `  3548` | **  29** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01DA`** | ` 474` | ` 11042` | `Type 40` | `  24` | `  2156` | **  14** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01DB`** | ` 475` | ` 11188` | `Type 40` | `  24` | `  2540` | **  18** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01DC`** | ` 476` | `  9166` | `Type 40` | `  24` | `   824` | **   6** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01DD`** | ` 477` | `  9212` | `Type 40` | `  24` | `  2920` | **  19** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01DE`** | ` 478` | `  9257` | `Type 40` | `  24` | `  3772` | **  29** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01DF`** | ` 479` | `  9691` | `Type 40` | `  24` | `   172` | **   3** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E0`** | ` 480` | `  9799` | `Type 40` | `  24` | `  1188` | **   9** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E1`** | ` 481` | ` 12802` | `Type 40` | `  24` | `   788` | **   5** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E2`** | ` 482` | `  7607` | `Type 40` | `  24` | `  1852` | **  10** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E3`** | ` 483` | `  7500` | `Type 40` | `  24` | `  4512` | **  26** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E4`** | ` 484` | `  7125` | `Type 40` | `  24` | `  2456` | **  17** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E5`** | ` 485` | `  7219` | `Type 40` | `  24` | `  3832` | **  37** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E6`** | ` 486` | `  7736` | `Type 40` | `  24` | `  1804` | **   9** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E7`** | ` 487` | `  7650` | `Type 40` | `  24` | `  1568` | **   7** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E8`** | ` 488` | `  7692` | `Type 40` | `  24` | `   892` | **   7** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01E9`** | ` 489` | `  7820` | `Type 40` | `  24` | `  2148` | **  18** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01EA`** | ` 490` | `  6526` | `Type 40` | `  24` | `  3808` | **  31** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01EB`** | ` 491` | `  6613` | `Type 40` | `  24` | `   276` | **   4** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01EC`** | ` 492` | `  6651` | `Type 40` | `  24` | `   940` | **   5** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01ED`** | ` 493` | `  6343` | `Type 40` | `  24` | `  2864` | **  15** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01EE`** | ` 494` | `  6271` | `Type 40` | `  24` | `  1832` | **  12** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01EF`** | ` 495` | `  6227` | `Type 40` | `  24` | `  1684` | **  11** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01F0`** | ` 496` | `  6150` | `Type 40` | `  24` | `  3268` | **  19** | ` 1` | Ch.3: Trans-Continental Tunnel Excavation and Labor Gang |
| **`0x01F1`** | ` 497` | `  6112` | `Type 40` | `  24` | `  2416` | **   9** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F2`** | ` 498` | `  6937` | `Type 40` | `  24` | `  3384` | **  36** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F3`** | ` 499` | `  6823` | `Type 40` | `  24` | `  1152` | **  12** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F4`** | ` 500` | `  6727` | `Type 40` | `  24` | `  2248` | **  16** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F5`** | ` 501` | `  6901` | `Type 40` | `  24` | `   744` | **   4** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F6`** | ` 502` | `  8221` | `Type 40` | `  24` | `  2468` | **  21** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F7`** | ` 503` | `  8318` | `Type 40` | `  24` | `  1824` | **  15** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F8`** | ` 504` | `  8008` | `Type 40` | `  24` | `  1296` | **  11** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01F9`** | ` 505` | `  8428` | `Type 40` | `  24` | `  1256` | **   7** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01FA`** | ` 506` | `  8175` | `Type 40` | `  24` | `  2744` | **  15** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01FB`** | ` 507` | `  8473` | `Type 40` | `  24` | `  1636` | **   7** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01FC`** | ` 508` | `  8633` | `Type 40` | `  24` | `  2876` | **  24** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01FD`** | ` 509` | `  8594` | `Type 40` | `  24` | `  1056` | **   7** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01FE`** | ` 510` | `  8514` | `Type 40` | `  24` | `  1104` | **   7** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x01FF`** | ` 511` | `  8809` | `Type 40` | `  24` | `  1064` | **   8** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0200`** | ` 512` | `  8865` | `Type 40` | `  24` | `  1172` | **   6** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0201`** | ` 513` | `  8925` | `Type 40` | `  24` | `  1216` | **   7** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0202`** | ` 514` | `  7033` | `Type 40` | `  24` | `  2576` | **  13** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0203`** | ` 515` | `  7088` | `Type 40` | `  24` | `  1572` | **   8** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0204`** | ` 516` | `  7963` | `Type 40` | `  24` | `  1212` | **   7** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0205`** | ` 517` | `  8768` | `Type 40` | `  24` | `   684` | **   3** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0206`** | ` 518` | ` 52896` | `Type 40` | `  24` | `  9788` | **  65** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0207`** | ` 519` | ` 52788` | `Type 40` | `  24` | `  7232` | **  42** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0208`** | ` 520` | ` 52855` | `Type 40` | `  24` | `  2472` | **  12** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0209`** | ` 521` | ` 52985` | `Type 40` | `  24` | `  1480` | **   8** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x020A`** | ` 522` | ` 24704` | `Type 40` | `  24` | `  4032` | **  27** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x020B`** | ` 523` | ` 24600` | `Type 40` | `  24` | `  6808` | **  55** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x020C`** | ` 524` | ` 24526` | `Type 40` | `  24` | `  8288` | **  48** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x020D`** | ` 525` | ` 24455` | `Type 40` | `  24` | `  2460` | **  13** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x020E`** | ` 526` | ` 58834` | `Type 40` | `  24` | `  3040` | **  18** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x020F`** | ` 527` | ` 58924` | `Type 40` | `  24` | `  5400` | **  26** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0210`** | ` 528` | ` 59052` | `Type 40` | `  24` | `  4056` | **  13** | ` 1` | Ch.3: Endor Commercial Emporium and Chapter 3 Finale |
| **`0x0211`** | ` 529` | ` 59105` | `Type 40` | `  24` | `  2848` | **  22** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0212`** | ` 530` | ` 58444` | `Type 40` | `  24` | `  3600` | **  34** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0213`** | ` 531` | ` 58729` | `Type 40` | `  24` | `  1452` | **   7** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0214`** | ` 532` | ` 58621` | `Type 40` | `  24` | `  3648` | **  25** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0215`** | ` 533` | ` 60660` | `Type 40` | `  24` | `  5516` | **  31** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0216`** | ` 534` | ` 60531` | `Type 40` | `  24` | `  5056` | **  24** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0217`** | ` 535` | ` 61049` | `Type 40` | `  24` | `  1564` | **   5** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0218`** | ` 536` | ` 60739` | `Type 40` | `  24` | `  2984` | **  16** | ` 2` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0219`** | ` 537` | ` 45404` | `Type 40` | `  24` | `  5552` | **  41** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021A`** | ` 538` | ` 45227` | `Type 40` | `  24` | `  4820` | **  25** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021B`** | ` 539` | ` 45311` | `Type 40` | `  24` | `   976` | **   7** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021C`** | ` 540` | ` 45169` | `Type 40` | `  24` | `  3248` | **  24** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021D`** | ` 541` | ` 45347` | `Type 40` | `  24` | `  4948` | **  30** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021E`** | ` 542` | ` 57581` | `Type 40` | `  24` | `  8356` | **  61** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021F`** | ` 543` | ` 57741` | `Type 40` | `  24` | `  1124` | **   6** | ` 2` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0220`** | ` 544` | ` 57691` | `Type 40` | `  24` | `  4972` | **  23** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0221`** | ` 545` | ` 42360` | `Type 40` | `  24` | ` 10460` | **  85** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0222`** | ` 546` | ` 42309` | `Type 40` | `  24` | `  3536` | **  27** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0223`** | ` 547` | ` 23028` | `Type 40` | `  24` | `  1032` | **   8** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0224`** | ` 548` | ` 23076` | `Type 40` | `  24` | `  4292` | **  24** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0225`** | ` 549` | ` 23150` | `Type 40` | `  24` | `  3664` | **  25** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0226`** | ` 550` | ` 23216` | `Type 40` | `  24` | `  4852` | **  34** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0227`** | ` 551` | ` 22961` | `Type 40` | `  24` | `  1596` | **   9** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0228`** | ` 552` | ` 51168` | `Type 40` | `  24` | `  7800` | **  52** | ` 2` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0229`** | ` 553` | ` 51525` | `Type 40` | `  24` | ` 12304` | ** 103** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x022A`** | ` 554` | ` 51626` | `Type 40` | `  24` | ` 15796` | ** 102** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x022B`** | ` 555` | ` 51706` | `Type 40` | `  24` | ` 10468` | ** 113** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x022C`** | ` 556` | ` 51089` | `Type 40` | `  24` | `  3868` | **  18** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x022D`** | ` 557` | ` 51134` | `Type 40` | `  24` | `  2216` | **  10** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x022E`** | ` 558` | ` 51046` | `Type 40` | `  24` | `  2416` | **  10** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x022F`** | ` 559` | ` 40451` | `Type 40` | `  24` | `  9868` | **  88** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0230`** | ` 560` | ` 40523` | `Type 40` | `  24` | `  5060` | **  48** | ` 1` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0231`** | ` 561` | ` 40564` | `Type 40` | `  24` | `  8672` | **  71** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0232`** | ` 562` | ` 40651` | `Type 40` | `  24` | `  8136` | **  71** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0233`** | ` 563` | ` 40713` | `Type 40` | `  24` | `  6132` | **  51** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0234`** | ` 564` | ` 40160` | `Type 40` | `  24` | `  8516` | **  74** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0235`** | ` 565` | ` 40223` | `Type 40` | `  24` | `  3744` | **  34** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0236`** | ` 566` | ` 40124` | `Type 40` | `  24` | `  3744` | **  34** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0237`** | ` 567` | ` 40260` | `Type 40` | `  24` | `  3948` | **  35** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0238`** | ` 568` | ` 40302` | `Type 40` | `  24` | `  3744` | **  34** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0239`** | ` 569` | ` 40347` | `Type 40` | `  24` | `  5304` | **  48** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x023A`** | ` 570` | `  5764` | `Type 40` | `  24` | `  7052` | **  45** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x023B`** | ` 571` | `  1090` | `Type 40` | `  24` | `  6108` | **  38** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x023C`** | ` 572` | `  5871` | `Type 40` | `  24` | `  4500` | **  30** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x023D`** | ` 573` | `   744` | `Type 40` | `  24` | `  3476` | **  27** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x023E`** | ` 574` | `  5933` | `Type 40` | `  24` | `  2380` | **  15** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x023F`** | ` 575` | `   822` | `Type 40` | `  24` | `  1892` | **  12** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0240`** | ` 576` | `  5973` | `Type 40` | `  24` | ` 11180` | **  66** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0241`** | ` 577` | `   887` | `Type 40` | `  24` | ` 10432` | **  58** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0242`** | ` 578` | ` 64856` | `Type 40` | `  24` | `  2792` | **  21** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0243`** | ` 579` | ` 64932` | `Type 40` | `  24` | `  2736` | **  21** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0244`** | ` 580` | ` 64986` | `Type 40` | `  24` | `   728` | **   6** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0245`** | ` 581` | ` 65025` | `Type 40` | `  24` | `  3012` | **  17** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0246`** | ` 582` | ` 65099` | `Type 40` | `  24` | `  2992` | **  17** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0247`** | ` 583` | ` 65162` | `Type 40` | `  24` | `  1200` | **   8** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0248`** | ` 584` | ` 65196` | `Type 40` | `  24` | `  3024` | **  16** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0249`** | ` 585` | ` 65253` | `Type 40` | `  24` | `  1628` | **  13** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x024A`** | ` 586` | ` 65291` | `Type 40` | `  24` | `   684` | **   5** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x024B`** | ` 587` | ` 65324` | `Type 40` | `  24` | `   936` | **   8** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x024C`** | ` 588` | ` 65358` | `Type 40` | `  24` | `   936` | **   8** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x024D`** | ` 589` | ` 65410` | `Type 40` | `  24` | `  1516` | **  12** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x024E`** | ` 590` | ` 44448` | `Type 40` | `  24` | `  3552` | **  27** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x024F`** | ` 591` | ` 44511` | `Type 40` | `  24` | `  4008` | **  29** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0250`** | ` 592` | ` 44691` | `Type 40` | `  24` | `  3552` | **  27** | ` 1` | Ch.4: Laissez Faire, Alchemy Master Edgan and Secret |
| **`0x0251`** | ` 593` | ` 44724` | `Type 40` | `  24` | `  4316` | **  33** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0252`** | ` 594` | ` 44617` | `Type 40` | `  24` | `  3816` | **  26** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0253`** | ` 595` | ` 52729` | `Type 40` | `  24` | `  2384` | **  13** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0254`** | ` 596` | ` 55332` | `Type 40` | `  24` | `  1240` | **  11** | ` 2` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0255`** | ` 597` | ` 55575` | `Type 40` | `  24` | `  1168` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0256`** | ` 598` | ` 55605` | `Type 40` | `  24` | `  1168` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0257`** | ` 599` | ` 55686` | `Type 40` | `  24` | `  1452` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0258`** | ` 600` | ` 55646` | `Type 40` | `  24` | `  1452` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0259`** | ` 601` | ` 55741` | `Type 40` | `  24` | `  1452` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x025A`** | ` 602` | ` 55781` | `Type 40` | `  24` | `  1452` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x025B`** | ` 603` | ` 55832` | `Type 40` | `  24` | `  1452` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x025C`** | ` 604` | ` 55879` | `Type 40` | `  24` | `  1452` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x025D`** | ` 605` | ` 19351` | `Type 40` | `  24` | `  2480` | **  12** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x025E`** | ` 606` | ` 19301` | `Type 40` | `  24` | `  2788` | **  13** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x025F`** | ` 607` | ` 21955` | `Type 40` | `  24` | `  2616` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0260`** | ` 608` | ` 22048` | `Type 40` | `  24` | `  2616` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0261`** | ` 609` | ` 22093` | `Type 40` | `  24` | `  2616` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0262`** | ` 610` | ` 22146` | `Type 40` | `  24` | `  2616` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0263`** | ` 611` | ` 22179` | `Type 40` | `  24` | `  2616` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0264`** | ` 612` | ` 57981` | `Type 40` | `  24` | `  2776` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0265`** | ` 613` | ` 58055` | `Type 40` | `  24` | `  2776` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0266`** | ` 614` | ` 58161` | `Type 40` | `  24` | `  2776` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0267`** | ` 615` | ` 58209` | `Type 40` | `  24` | `  3396` | **  22** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0268`** | ` 616` | ` 58321` | `Type 40` | `  24` | `  2776` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0269`** | ` 617` | ` 58385` | `Type 40` | `  24` | `  2776` | **  16** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x026A`** | ` 618` | ` 62223` | `Type 40` | `  24` | `  3296` | **  23** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x026B`** | ` 619` | ` 43353` | `Type 40` | `  24` | `  1452` | **   8** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x026C`** | ` 620` | ` 43284` | `Type 40` | `  24` | `  1084` | **   7** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x026D`** | ` 621` | ` 43242` | `Type 40` | `  24` | `  1084` | **   7** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x026E`** | ` 622` | ` 43168` | `Type 40` | `  24` | `  1268` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x026F`** | ` 623` | ` 43130` | `Type 40` | `  24` | `  2416` | **   9** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0270`** | ` 624` | ` 47941` | `Type 40` | `  24` | `  1788` | **  10** | ` 1` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0271`** | ` 625` | ` 48007` | `Type 40` | `  24` | `  1428` | **  10** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0272`** | ` 626` | ` 48088` | `Type 40` | `  24` | `  1596` | **  13** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0273`** | ` 627` | ` 48125` | `Type 40` | `  24` | `  1220` | **   6** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0274`** | ` 628` | ` 42705` | `Type 40` | `  24` | `  4152` | **  24** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0275`** | ` 629` | ` 42756` | `Type 40` | `  24` | `  5464` | **  29** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0276`** | ` 630` | ` 42809` | `Type 40` | `  24` | `  3196` | **  13** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0277`** | ` 631` | ` 42844` | `Type 40` | `  24` | `  3100` | **  14** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0278`** | ` 632` | ` 52383` | `Type 40` | `  24` | `  5080` | **  28** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0279`** | ` 633` | ` 52455` | `Type 40` | `  24` | `  5080` | **  28** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x027A`** | ` 634` | ` 52529` | `Type 40` | `  24` | `  6800` | **  36** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x027B`** | ` 635` | ` 52570` | `Type 40` | `  24` | `  1384` | **  11** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x027C`** | ` 636` | ` 52631` | `Type 40` | `  24` | `  1008` | **   8** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x027D`** | ` 637` | ` 52680` | `Type 40` | `  24` | `   548` | **   5** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x027E`** | ` 638` | ` 47424` | `Type 40` | `  24` | `  2024` | **  11** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x027F`** | ` 639` | ` 47545` | `Type 40` | `  24` | `  1264` | **   7** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0280`** | ` 640` | ` 47606` | `Type 40` | `  24` | `  1156` | **   8** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0281`** | ` 641` | ` 47719` | `Type 40` | `  24` | `  1668` | **   9** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0282`** | ` 642` | ` 61340` | `Type 40` | `  24` | `  2124` | **  12** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0283`** | ` 643` | ` 61091` | `Type 40` | `  24` | `  2124` | **  12** | ` 2` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0284`** | ` 644` | ` 61398` | `Type 40` | `  24` | `  1424` | **  10** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0285`** | ` 645` | ` 61455` | `Type 40` | `  24` | `  1272` | **   9** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0286`** | ` 646` | ` 61500` | `Type 40` | `  24` | `  1424` | **  10** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0287`** | ` 647` | ` 61567` | `Type 40` | `  24` | `  1424` | **  10** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0288`** | ` 648` | `  3919` | `Type 40` | `  24` | `  3320` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0289`** | ` 649` | `  3951` | `Type 40` | `  24` | `  3320` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x028A`** | ` 650` | `  4026` | `Type 40` | `  24` | `  3320` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x028B`** | ` 651` | `  3983` | `Type 40` | `  24` | `  3320` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x028C`** | ` 652` | `  3876` | `Type 40` | `  24` | `  3320` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x028D`** | ` 653` | `  3789` | `Type 40` | `  24` | `  3320` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x028E`** | ` 654` | `  3829` | `Type 40` | `  24` | `  3648` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x028F`** | ` 655` | `  3722` | `Type 40` | `  24` | `  3648` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0290`** | ` 656` | `  3655` | `Type 40` | `  24` | `  3648` | **  22** | ` 1` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x0291`** | ` 657` | `  3617` | `Type 40` | `  24` | `  3088` | **  22** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0292`** | ` 658` | `  3567` | `Type 40` | `  24` | `  3116` | **  23** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0293`** | ` 659` | ` 44859` | `Type 40` | `  24` | `  2780` | **  24** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0294`** | ` 660` | ` 44916` | `Type 40` | `  24` | `  3256` | **  25** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0295`** | ` 661` | ` 44978` | `Type 40` | `  24` | `  2248` | **  18** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0296`** | ` 662` | ` 45013` | `Type 40` | `  24` | `  3208` | **  27** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0297`** | ` 663` | ` 45079` | `Type 40` | `  24` | `  3016` | **  26** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0298`** | ` 664` | ` 62275` | `Type 40` | `  24` | `  1152` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x0299`** | ` 665` | ` 62325` | `Type 40` | `  24` | `  1152` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x029A`** | ` 666` | ` 62428` | `Type 40` | `  24` | `  1152` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x029B`** | ` 667` | ` 62503` | `Type 40` | `  24` | `  1080` | **   9** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x029C`** | ` 668` | ` 62567` | `Type 40` | `  24` | `  1296` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x029D`** | ` 669` | ` 62615` | `Type 40` | `  24` | `  1296` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x029E`** | ` 670` | ` 62660` | `Type 40` | `  24` | `  1296` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x029F`** | ` 671` | ` 62714` | `Type 40` | `  24` | `  1296` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A0`** | ` 672` | ` 62758` | `Type 40` | `  24` | `  1636` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A1`** | ` 673` | ` 62818` | `Type 40` | `  24` | `  1636` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A2`** | ` 674` | ` 62873` | `Type 40` | `  24` | `  1356` | **   9** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A3`** | ` 675` | ` 62939` | `Type 40` | `  24` | `  1920` | **  12** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A4`** | ` 676` | ` 62978` | `Type 40` | `  24` | `   652` | **   5** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A5`** | ` 677` | ` 63095` | `Type 40` | `  24` | `  1228` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A6`** | ` 678` | ` 63158` | `Type 40` | `  24` | `  1228` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A7`** | ` 679` | ` 63214` | `Type 40` | `  24` | `  1228` | **  10** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A8`** | ` 680` | ` 63271` | `Type 40` | `  24` | `   264` | **   3** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02A9`** | ` 681` | ` 63318` | `Type 40` | `  24` | `  1036` | **   9** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02AA`** | ` 682` | ` 63422` | `Type 40` | `  24` | `  1004` | **   9** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02AB`** | ` 683` | ` 63479` | `Type 40` | `  24` | `   672` | **   4** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02AC`** | ` 684` | ` 63522` | `Type 40` | `  24` | `   900` | **   6** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02AD`** | ` 685` | ` 63641` | `Type 40` | `  24` | `   304` | **   3** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02AE`** | ` 686` | ` 63681` | `Type 40` | `  24` | `  2740` | **  17** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02AF`** | ` 687` | ` 63730` | `Type 40` | `  24` | `   888` | **   7** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02B0`** | ` 688` | ` 63795` | `Type 40` | `  24` | `   476` | **   4** | ` 1` | Ch.4: Balzack Showdown, Minister Keeleon and Secret Chamber |
| **`0x02B1`** | ` 689` | ` 63843` | `Type 40` | `  24` | `  1096` | **   7** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B2`** | ` 690` | ` 63890` | `Type 40` | `  24` | `  1632` | **  10** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B3`** | ` 691` | ` 53608` | `Type 40` | `  24` | `  3152` | **  25** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B4`** | ` 692` | ` 53646` | `Type 40` | `  24` | `  1280` | **   9** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B5`** | ` 693` | ` 53421` | `Type 40` | `  24` | `  6044` | **  41** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B6`** | ` 694` | ` 53529` | `Type 40` | `  24` | `  1036` | **   7** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B7`** | ` 695` | ` 59495` | `Type 40` | `  24` | `  1384` | **   7** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B8`** | ` 696` | ` 59629` | `Type 40` | `  24` | `  2200` | **  11** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02B9`** | ` 697` | ` 59587` | `Type 40` | `  24` | `  4408` | **  20** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02BA`** | ` 698` | ` 59837` | `Type 40` | `  24` | `  1992` | **   7** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02BB`** | ` 699` | ` 59674` | `Type 40` | `  24` | `  8836` | **  60** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02BC`** | ` 700` | ` 59888` | `Type 40` | `  24` | `  3324` | **  30** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02BD`** | ` 701` | ` 60023` | `Type 40` | `  24` | `  2344` | **  13** | ` 2` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02BE`** | ` 702` | ` 22423` | `Type 40` | `  24` | `  2268` | **  14** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02BF`** | ` 703` | ` 22478` | `Type 40` | `  24` | `  2620` | **  20** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02C0`** | ` 704` | ` 22571` | `Type 40` | `  24` | `  5196` | **  39** | ` 1` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02C1`** | ` 705` | ` 22684` | `Type 40` | `  24` | `  1144` | **   7** | ` 2` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C2`** | ` 706` | ` 53927` | `Type 40` | `  24` | `  1400` | **   7** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C3`** | ` 707` | ` 53965` | `Type 40` | `  24` | `  1488` | **  11** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C4`** | ` 708` | ` 54494` | `Type 40` | `  24` | `  6124` | **  34** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C5`** | ` 709` | ` 54247` | `Type 40` | `  24` | `  7216` | **  45** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C6`** | ` 710` | ` 54540` | `Type 40` | `  24` | `  3608` | **  21** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C7`** | ` 711` | ` 54347` | `Type 40` | `  24` | `  7300` | **  43** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C8`** | ` 712` | ` 25029` | `Type 40` | `  24` | `  1284` | **   6** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02C9`** | ` 713` | ` 25114` | `Type 40` | `  24` | `  1632` | **  11** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CA`** | ` 714` | ` 24973` | `Type 40` | `  24` | `  2360` | **  12** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CB`** | ` 715` | ` 25165` | `Type 40` | `  24` | `  7612` | **  61** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CC`** | ` 716` | ` 25452` | `Type 40` | `  24` | `  3544` | **  21** | ` 2` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CD`** | ` 717` | ` 25285` | `Type 40` | `  24` | `  1552` | **   6** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CE`** | ` 718` | ` 25338` | `Type 40` | `  24` | `   500` | **   4** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CF`** | ` 719` | ` 20001` | `Type 40` | `  24` | `   684` | **   5** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D0`** | ` 720` | ` 20057` | `Type 40` | `  24` | `   516` | **   4** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D1`** | ` 721` | ` 20095` | `Type 40` | `  24` | `  1752` | **   8** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D2`** | ` 722` | ` 20133` | `Type 40` | `  24` | `  3208` | **  22** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D3`** | ` 723` | ` 20184` | `Type 40` | `  24` | `  5896` | **  48** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D4`** | ` 724` | ` 20280` | `Type 40` | `  24` | `  1436` | **   9** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D5`** | ` 725` | ` 19438` | `Type 40` | `  24` | `  1028` | **   7** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D6`** | ` 726` | ` 19475` | `Type 40` | `  24` | `  1148` | **   6** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D7`** | ` 727` | ` 19510` | `Type 40` | `  24` | `  1360` | **   7** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D8`** | ` 728` | ` 19545` | `Type 40` | `  24` | `  4356` | **  39** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02D9`** | ` 729` | ` 19663` | `Type 40` | `  24` | `   820` | **   4** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DA`** | ` 730` | ` 19708` | `Type 40` | `  24` | `  4380` | **  21** | ` 2` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DB`** | ` 731` | ` 20811` | `Type 40` | `  24` | `  1168` | **   9** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DC`** | ` 732` | ` 20772` | `Type 40` | `  24` | `  1192` | **   6** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DD`** | ` 733` | ` 21047` | `Type 40` | `  24` | `  1628` | **   8** | ` 2` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DE`** | ` 734` | ` 20860` | `Type 40` | `  24` | `  2048` | **  11** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DF`** | ` 735` | ` 20941` | `Type 40` | `  24` | `  6204` | **  42** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02E0`** | ` 736` | ` 21295` | `Type 40` | `  24` | `  3300` | **  12** | ` 1` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02E1`** | ` 737` | ` 21344` | `Type 40` | `  24` | `  1128` | **   7** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E2`** | ` 738` | ` 21383` | `Type 40` | `  24` | `  1520` | **   7** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E3`** | ` 739` | ` 21524` | `Type 40` | `  24` | `  1696` | **  10** | ` 2` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E4`** | ` 740` | ` 38976` | `Type 40` | `  24` | `  4488` | **  31** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E5`** | ` 741` | ` 39023` | `Type 40` | `  24` | `  1252` | **   8** | ` 2` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E6`** | ` 742` | ` 39276` | `Type 40` | `  24` | `  8060` | ** 103** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E7`** | ` 743` | ` 39635` | `Type 40` | `  24` | `   888` | **   5** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E8`** | ` 744` | ` 39671` | `Type 40` | `  24` | `  2144` | **  13** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E9`** | ` 745` | ` 39710` | `Type 40` | `  24` | `  2304` | **  15** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02EA`** | ` 746` | ` 39753` | `Type 40` | `  24` | `  3288` | **  29** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02EB`** | ` 747` | ` 39490` | `Type 40` | `  24` | `  7364` | **  58** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02EC`** | ` 748` | ` 43526` | `Type 40` | `  24` | `   988` | **  10** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02ED`** | ` 749` | ` 43417` | `Type 40` | `  24` | `  1352` | **   8** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02EE`** | ` 750` | ` 43454` | `Type 40` | `  24` | `   240` | **   7** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02EF`** | ` 751` | ` 43567` | `Type 40` | `  24` | `  3220` | **  28** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F0`** | ` 752` | ` 46009` | `Type 40` | `  24` | `  2444` | **  14** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F1`** | ` 753` | ` 46125` | `Type 40` | `  24` | `  4436` | **  27** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F2`** | ` 754` | ` 46055` | `Type 40` | `  24` | `  1832` | **  13** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F3`** | ` 755` | ` 44329` | `Type 40` | `  24` | `   688` | **   4** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F4`** | ` 756` | ` 44405` | `Type 40` | `  24` | `  1220` | **   7** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F5`** | ` 757` | ` 44237` | `Type 40` | `  24` | `  6172` | **  43** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F6`** | ` 758` | ` 54837` | `Type 40` | `  24` | `  2412` | **  10** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F7`** | ` 759` | ` 56460` | `Type 40` | `  24` | `  3192` | **  14** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F8`** | ` 760` | ` 56521` | `Type 40` | `  24` | `  2124` | **  11** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02F9`** | ` 761` | ` 56615` | `Type 40` | `  24` | ` 10344` | **  77** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02FA`** | ` 762` | ` 56737` | `Type 40` | `  24` | `  2024` | **   7** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02FB`** | ` 763` | ` 56784` | `Type 40` | `  24` | `  3312` | **  14** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02FC`** | ` 764` | ` 48216` | `Type 40` | `  24` | `  5444` | **  34** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02FD`** | ` 765` | ` 48327` | `Type 40` | `  24` | `  1192` | **   5** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02FE`** | ` 766` | ` 48369` | `Type 40` | `  24` | `  1748` | **   5** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02FF`** | ` 767` | ` 48406` | `Type 40` | `  24` | `  1380` | **   6** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x0300`** | ` 768` | ` 48455` | `Type 40` | `  24` | `  2056` | **  10** | ` 1` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x0301`** | ` 769` | ` 48885` | `Type 40` | `  24` | `  2456` | **  12** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0302`** | ` 770` | ` 48737` | `Type 40` | `  24` | ` 10656` | **  84** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0303`** | ` 771` | ` 48964` | `Type 40` | `  24` | `  6924` | **  46** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0304`** | ` 772` | ` 49020` | `Type 40` | `  24` | `  4256` | **  28** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0305`** | ` 773` | ` 49311` | `Type 40` | `  24` | `  2780` | **  25** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0306`** | ` 774` | ` 49390` | `Type 40` | `  24` | ` 19556` | ** 176** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0307`** | ` 775` | ` 49515` | `Type 40` | `  24` | `  6092` | **  46** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0308`** | ` 776` | ` 49563` | `Type 40` | `  24` | `  7464` | **  63** | ` 2` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0309`** | ` 777` | ` 23913` | `Type 40` | `  24` | `  1436` | **   7** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x030A`** | ` 778` | ` 23949` | `Type 40` | `  24` | `  2964` | **  17** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x030B`** | ` 779` | ` 23996` | `Type 40` | `  24` | `  5692` | **  34** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x030C`** | ` 780` | ` 24099` | `Type 40` | `  24` | `  1912` | **  12** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x030D`** | ` 781` | ` 47044` | `Type 40` | `  24` | `  2060` | **  12** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x030E`** | ` 782` | ` 47097` | `Type 40` | `  24` | `  1720` | **   8** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x030F`** | ` 783` | ` 47139` | `Type 40` | `  24` | `  1488` | **   8** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0310`** | ` 784` | ` 46798` | `Type 40` | `  24` | `  8508` | **  88** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0311`** | ` 785` | ` 46735` | `Type 40` | `  24` | `  3960` | **  27** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0312`** | ` 786` | ` 46659` | `Type 40` | `  24` | `  6604` | **  47** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0313`** | ` 787` | ` 47178` | `Type 40` | `  24` | `  1308` | **   7** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0314`** | ` 788` | ` 41609` | `Type 40` | `  24` | `  7148` | **  49** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0315`** | ` 789` | ` 41894` | `Type 40` | `  24` | `  8096` | **  47** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0316`** | ` 790` | ` 42072` | `Type 40` | `  24` | `  1324` | **   7** | ` 2` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0317`** | ` 791` | ` 42028` | `Type 40` | `  24` | `  1744` | **   7** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0318`** | ` 792` | `  4132` | `Type 40` | `  24` | `  1848` | **   9** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0319`** | ` 793` | `  4158` | `Type 40` | `  24` | `  2476` | **  20** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031A`** | ` 794` | `  4189` | `Type 40` | `  24` | ` 17268` | ** 133** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031B`** | ` 795` | ` 61936` | `Type 40` | `  24` | `  1052` | **   8** | ` 2` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031C`** | ` 796` | ` 61706` | `Type 40` | `  24` | `  1036` | **   7** | ` 2` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031D`** | ` 797` | ` 61669` | `Type 40` | `  24` | `  1648` | **   8** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031E`** | ` 798` | ` 42885` | `Type 40` | `  24` | `  2876` | **  12** | ` 2` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031F`** | ` 799` | ` 55300` | `Type 40` | `  24` | `  2780` | **  23** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0320`** | ` 800` | ` 55267` | `Type 40` | `  24` | `  1656` | **  11** | ` 1` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0321`** | ` 801` | ` 57092` | `Type 40` | `  24` | `  2244` | **  20** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0322`** | ` 802` | ` 57132` | `Type 40` | `  24` | `  5428` | **  26** | ` 2` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0323`** | ` 803` | ` 49828` | `Type 40` | `  24` | `  2084` | **  11** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0324`** | ` 804` | ` 45138` | `Type 40` | `  24` | `  3140` | **  18** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0325`** | ` 805` | ` 51978` | `Type 40` | `  24` | `  1468` | **   8** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0326`** | ` 806` | ` 52017` | `Type 40` | `  24` | `  1420` | **   8** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0327`** | ` 807` | ` 40957` | `Type 40` | `  24` | `   912` | **   6** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0328`** | ` 808` | ` 41061` | `Type 40` | `  24` | `  2512` | **  18** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0329`** | ` 809` | ` 41016` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x032A`** | ` 810` | ` 41216` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x032B`** | ` 811` | ` 41272` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x032C`** | ` 812` | ` 41326` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x032D`** | ` 813` | ` 41371` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x032E`** | ` 814` | ` 41413` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x032F`** | ` 815` | ` 41466` | `Type 40` | `  24` | `  1644` | **   9** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0330`** | ` 816` | ` 41505` | `Type 40` | `  24` | `  1680` | **  10** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0331`** | ` 817` | `  3493` | `Type 40` | `  24` | `  1548` | **  11** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0332`** | ` 818` | `  3383` | `Type 40` | `  24` | `  1712` | **  10** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0333`** | ` 819` | ` 64192` | `Type 40` | `  24` | `   864` | **   8** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0334`** | ` 820` | ` 64149` | `Type 40` | `  24` | `   304` | **   4** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0335`** | ` 821` | ` 64116` | `Type 40` | `  24` | `   304` | **   4** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0336`** | ` 822` | ` 64083` | `Type 40` | `  24` | `   304` | **   4** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0337`** | ` 823` | ` 64643` | `Type 40` | `  24` | `  2996` | **  25** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0338`** | ` 824` | ` 64535` | `Type 40` | `  24` | `  2944` | **  21** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0339`** | ` 825` | ` 64573` | `Type 40` | `  24` | `  5008` | **  38** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x033A`** | ` 826` | ` 64614` | `Type 40` | `  24` | `  2948` | **  22** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x033B`** | ` 827` | ` 64453` | `Type 40` | `  24` | `  5600` | **  43** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x033C`** | ` 828` | ` 64499` | `Type 40` | `  24` | `  3312` | **  22** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x033D`** | ` 829` | ` 64409` | `Type 40` | `  24` | `  5068` | **  36** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x033E`** | ` 830` | ` 64372` | `Type 40` | `  24` | `  3400` | **  22** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x033F`** | ` 831` | ` 64336` | `Type 40` | `  24` | `  3208` | **  21** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0340`** | ` 832` | ` 64301` | `Type 40` | `  24` | `  3432` | **  21** | ` 1` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x0341`** | ` 833` | ` 64255` | `Type 40` | `  24` | `  5808` | **  37** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0342`** | ` 834` | ` 65447` | `Type 40` | `  24` | `  1896` | **  12** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0343`** | ` 835` | ` 65530` | `Type 40` | `  24` | `  5312` | **  46** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0344`** | ` 836` | ` 55909` | `Type 40` | `  24` | `  1864` | **  11** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0345`** | ` 837` | ` 55966` | `Type 40` | `  24` | `  1924` | **  11** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0346`** | ` 838` | ` 56016` | `Type 40` | `  24` | `  1728` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0347`** | ` 839` | ` 56069` | `Type 40` | `  24` | `  1728` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0348`** | ` 840` | ` 56116` | `Type 40` | `  24` | `  1728` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0349`** | ` 841` | ` 56154` | `Type 40` | `  24` | `  1808` | **  11** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x034A`** | ` 842` | ` 56209` | `Type 40` | `  24` | `  1992` | **  11** | ` 2` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x034B`** | ` 843` | ` 23738` | `Type 40` | `  24` | `  1784` | **   9** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x034C`** | ` 844` | ` 23782` | `Type 40` | `  24` | `  1784` | **   9** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x034D`** | ` 845` | ` 23683` | `Type 40` | `  24` | `  1784` | **   9** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x034E`** | ` 846` | ` 23627` | `Type 40` | `  24` | `  1784` | **   9** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x034F`** | ` 847` | ` 23576` | `Type 40` | `  24` | `  1784` | **   9** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0350`** | ` 848` | ` 23828` | `Type 40` | `  24` | `  1400` | **   8** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0351`** | ` 849` | ` 23285` | `Type 40` | `  24` | `  1644` | **  10** | ` 2` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0352`** | ` 850` | ` 50657` | `Type 40` | `  24` | `  4928` | **  25** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0353`** | ` 851` | ` 50712` | `Type 40` | `  24` | `  3760` | **  18** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0354`** | ` 852` | ` 50770` | `Type 40` | `  24` | `  4296` | **  25** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0355`** | ` 853` | ` 50832` | `Type 40` | `  24` | `  4164` | **  21** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0356`** | ` 854` | ` 50882` | `Type 40` | `  24` | `  4436` | **  25** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0357`** | ` 855` | ` 50369` | `Type 40` | `  24` | ` 12824` | ** 107** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0358`** | ` 856` | ` 50322` | `Type 40` | `  24` | `  4364` | **  28** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0359`** | ` 857` | ` 50109` | `Type 40` | `  24` | `  4208` | **  29** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x035A`** | ` 858` | ` 50014` | `Type 40` | `  24` | `  4208` | **  29** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x035B`** | ` 859` | ` 49893` | `Type 40` | `  24` | `  8892` | **  76** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x035C`** | ` 860` | `  4648` | `Type 40` | `  24` | `  1504` | **  11** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x035D`** | ` 861` | `  5092` | `Type 40` | `  24` | `  2008` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x035E`** | ` 862` | `  4708` | `Type 40` | `  24` | `  1500` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x035F`** | ` 863` | `  4777` | `Type 40` | `  24` | `  1500` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0360`** | ` 864` | `  4851` | `Type 40` | `  24` | `  1524` | **  10** | ` 1` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0361`** | ` 865` | `  5331` | `Type 40` | `  24` | `  1636` | **  10** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0362`** | ` 866` | `  4928` | `Type 40` | `  24` | `  1900` | **  10** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0363`** | ` 867` | `  5597` | `Type 40` | `  24` | `  1520` | **  10** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0364`** | ` 868` | ` 20570` | `Type 40` | `  24` | ` 26536` | ** 227** | `16` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0365`** | ` 869` | `  5689` | `Type 40` | `  24` | `  5696` | **  40** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0366`** | ` 870` | ` 64698` | `Type 40` | `  24` | `  5476` | **  34** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0367`** | ` 871` | `  4499` | `Type 40` | `  24` | `  9772` | **  66** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0368`** | ` 872` | ` 25727` | `Type 40` | `  24` | `  2968` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0369`** | ` 873` | ` 25854` | `Type 40` | `  24` | `   876` | **   5** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x036A`** | ` 874` | ` 26092` | `Type 40` | `  24` | `  4312` | **  21** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x036B`** | ` 875` | ` 26800` | `Type 40` | `  24` | `  4128` | **  19** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x036C`** | ` 876` | ` 27405` | `Type 40` | `  24` | `  4304` | **  19** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x036D`** | ` 877` | ` 28133` | `Type 40` | `  24` | `  4364` | **  22** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x036E`** | ` 878` | ` 29101` | `Type 40` | `  24` | `  1612` | **   7** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x036F`** | ` 879` | ` 30454` | `Type 40` | `  24` | `  1336` | **   7** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0370`** | ` 880` | ` 31523` | `Type 40` | `  24` | `  1620` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0371`** | ` 881` | ` 32751` | `Type 40` | `  24` | `   996` | **   6** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0372`** | ` 882` | ` 33565` | `Type 40` | `  24` | `  1204` | **   6** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0373`** | ` 883` | ` 34136` | `Type 40` | `  24` | `   948` | **   6** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0374`** | ` 884` | ` 35090` | `Type 40` | `  24` | `  1224` | **   6** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0375`** | ` 885` | ` 36298` | `Type 40` | `  24` | `   880` | **   5** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0376`** | ` 886` | ` 37561` | `Type 40` | `  24` | `  1484` | **   9** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0377`** | ` 887` | ` 38513` | `Type 40` | `  24` | `  1868` | **  14** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0378`** | ` 888` | ` 28779` | `Type 40` | `  24` | `   996` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0379`** | ` 889` | ` 30129` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x037A`** | ` 890` | ` 38079` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x037B`** | ` 891` | ` 33273` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x037C`** | ` 892` | ` 33879` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x037D`** | ` 893` | ` 35707` | `Type 40` | `  24` | `  2024` | **  10** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x037E`** | ` 894` | ` 36404` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x037F`** | ` 895` | ` 31239` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0380`** | ` 896` | ` 33354` | `Type 40` | `  24` | `   880` | **   8** | ` 1` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |
| **`0x0381`** | ` 897` | `106669` | `Type 40` | `  24` | `   100` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0382`** | ` 898` | `106849` | `Type 40` | `  24` | `  1752` | **   7** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0383`** | ` 899` | ` 18969` | `Type 40` | `  24` | `  1476` | **  11** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0384`** | ` 900` | ` 18822` | `Type 40` | `  24` | `  1440` | **  16** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0385`** | ` 901` | ` 18782` | `Type 40` | `  24` | `   896` | **   6** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0386`** | ` 902` | ` 18878` | `Type 40` | `  24` | `   896` | **   6** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0387`** | ` 903` | ` 18917` | `Type 40` | `  24` | `  1360` | **  14** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0388`** | ` 904` | ` 19015` | `Type 40` | `  24` | `   976` | **   7** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0389`** | ` 905` | ` 17642` | `Type 40` | `  24` | `  2056` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x038A`** | ` 906` | ` 17988` | `Type 40` | `  24` | `  5648` | **  41** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x038B`** | ` 907` | ` 18089` | `Type 40` | `  24` | `  3024` | **  22** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x038C`** | ` 908` | ` 18134` | `Type 40` | `  24` | `  2560` | **  16** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x038D`** | ` 909` | ` 18240` | `Type 40` | `  24` | `  1524` | **  18** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x038E`** | ` 910` | ` 17835` | `Type 40` | `  24` | `  1004` | **   5** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x038F`** | ` 911` | ` 18202` | `Type 40` | `  24` | `   980` | **   5** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0390`** | ` 912` | ` 17371` | `Type 40` | `  24` | `  7220` | **  54** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0391`** | ` 913` | ` 17564` | `Type 40` | `  24` | `  1196` | **   5** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0392`** | ` 914` | ` 16950` | `Type 40` | `  24` | `  7852` | **  57** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0393`** | ` 915` | ` 17045` | `Type 40` | `  24` | `  2544` | **  11** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0394`** | ` 916` | ` 17296` | `Type 40` | `  24` | `  1644` | **   8** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0395`** | ` 917` | ` 18623` | `Type 40` | `  24` | `  1388` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0396`** | ` 918` | ` 18483` | `Type 40` | `  24` | `  1368` | **  13** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0397`** | ` 919` | ` 18530` | `Type 40` | `  24` | `  1368` | **  13** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0398`** | ` 920` | ` 18568` | `Type 40` | `  24` | `  2564` | **  17** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x0399`** | ` 921` | ` 18429` | `Type 40` | `  24` | `  2448` | **  15** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x039A`** | ` 922` | ` 18320` | `Type 40` | `  24` | `   896` | **   7** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x039B`** | ` 923` | ` 18376` | `Type 40` | `  24` | `  2836` | **  24** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x039C`** | ` 924` | ` 17150` | `Type 40` | `  24` | `  1464` | **  10** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x039D`** | ` 925` | `  2789` | `Type 42` | `  24` | `   188` | **   2** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x039E`** | ` 926` | `  2790` | `Type 42` | `  24` | `   852` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x039F`** | ` 927` | `  2791` | `Type 42` | `  24` | `  1116` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A0`** | ` 928` | `  2792` | `Type 42` | `  24` | `  1396` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A1`** | ` 929` | `  2793` | `Type 42` | `  24` | `   908` | **  18** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A2`** | ` 930` | `  2794` | `Type 42` | `  24` | `   972` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A3`** | ` 931` | `  2795` | `Type 42` | `  24` | `   860` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A4`** | ` 932` | `  2796` | `Type 42` | `  24` | `   980` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A5`** | ` 933` | `  2797` | `Type 42` | `  24` | `   984` | **  12** | ` 1` | Ch.5: Yggdrasil World Tree, Zenithia Castle and Zenith Dragon |
| **`0x03A6`** | ` 934` | `  2798` | `Type 42` | `  24` | `  1128` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 0) |
| **`0x03A7`** | ` 935` | `  2799` | `Type 42` | `  24` | `  1132` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 1) |
| **`0x03A8`** | ` 936` | `  2800` | `Type 42` | `  24` | `   988` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 2) |
| **`0x03A9`** | ` 937` | `  2801` | `Type 42` | `  24` | `   964` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 3) |
| **`0x03AA`** | ` 938` | `  2802` | `Type 42` | `  24` | `   968` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 4) |
| **`0x03AB`** | ` 939` | `  2803` | `Type 42` | `  24` | `   876` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 5) |
| **`0x03AC`** | ` 940` | `  2804` | `Type 42` | `  24` | `  1384` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 6) |
| **`0x03AD`** | ` 941` | `  2805` | `Type 42` | `  24` | `  1276` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 7) |
| **`0x03AE`** | ` 942` | `  2806` | `Type 42` | `  24` | `  1080` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 8) |
| **`0x03AF`** | ` 943` | `  2807` | `Type 42` | `  24` | `   988` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 9) |
| **`0x03B0`** | ` 944` | `  2808` | `Type 42` | `  24` | `  1164` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 10) |
| **`0x03B1`** | ` 945` | `  2809` | `Type 42` | `  24` | `   876` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 11) |
| **`0x03B2`** | ` 946` | `  2810` | `Type 42` | `  24` | `  1244` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 12) |
| **`0x03B3`** | ` 947` | `  2811` | `Type 42` | `  24` | `  1208` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 13) |
| **`0x03B4`** | ` 948` | `  2812` | `Type 42` | `  24` | `   960` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 14) |
| **`0x03B5`** | ` 949` | `  2813` | `Type 42` | `  24` | `   844` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 15) |
| **`0x03B6`** | ` 950` | `  2814` | `Type 42` | `  24` | `   928` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 16) |
| **`0x03B7`** | ` 951` | `  2815` | `Type 42` | `  24` | `   920` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 17) |
| **`0x03B8`** | ` 952` | `  2816` | `Type 42` | `  24` | `  1192` | **  14** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 18) |
| **`0x03B9`** | ` 953` | `  2817` | `Type 42` | `  24` | `  1076` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 19) |
| **`0x03BA`** | ` 954` | `  2818` | `Type 42` | `  24` | `  1008` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 20) |
| **`0x03BB`** | ` 955` | `  2819` | `Type 42` | `  24` | `   968` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 21) |
| **`0x03BC`** | ` 956` | `  2820` | `Type 42` | `  24` | `   864` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 22) |
| **`0x03BD`** | ` 957` | `  2821` | `Type 42` | `  24` | `  1008` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 23) |
| **`0x03BE`** | ` 958` | `  2822` | `Type 42` | `  24` | `  1128` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 24) |
| **`0x03BF`** | ` 959` | `  2823` | `Type 42` | `  24` | `  1180` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 25) |
| **`0x03C0`** | ` 960` | `  2824` | `Type 42` | `  24` | `   804` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 26) |
| **`0x03C1`** | ` 961` | `  2825` | `Type 42` | `  24` | `   904` | **  16** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 27) |
| **`0x03C2`** | ` 962` | `  2826` | `Type 42` | `  24` | `   912` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 28) |
| **`0x03C3`** | ` 963` | `  2827` | `Type 42` | `  24` | `  1080` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 29) |
| **`0x03C4`** | ` 964` | `  2828` | `Type 42` | `  24` | `  1276` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 30) |
| **`0x03C5`** | ` 965` | `  2829` | `Type 42` | `  24` | `  1236` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 31) |
| **`0x03C6`** | ` 966` | `  2830` | `Type 42` | `  24` | `   828` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 32) |
| **`0x03C7`** | ` 967` | `  2831` | `Type 42` | `  24` | `   884` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 33) |
| **`0x03C8`** | ` 968` | `  2832` | `Type 42` | `  24` | `   972` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 34) |
| **`0x03C9`** | ` 969` | `  2833` | `Type 42` | `  24` | `   872` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 35) |
| **`0x03CA`** | ` 970` | `  2834` | `Type 42` | `  24` | `   980` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 36) |
| **`0x03CB`** | ` 971` | `  2835` | `Type 42` | `  24` | `  1116` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 37) |
| **`0x03CC`** | ` 972` | `  2836` | `Type 42` | `  24` | `  1360` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 38) |
| **`0x03CD`** | ` 973` | `  2837` | `Type 42` | `  24` | `   768` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 39) |
| **`0x03CE`** | ` 974` | `  2838` | `Type 42` | `  24` | `  1008` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 40) |
| **`0x03CF`** | ` 975` | `  2839` | `Type 42` | `  24` | `  1196` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 41) |
| **`0x03D0`** | ` 976` | `  2840` | `Type 42` | `  24` | `  1124` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 42) |
| **`0x03D1`** | ` 977` | `  2841` | `Type 42` | `  24` | `  1260` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 43) |
| **`0x03D2`** | ` 978` | `  2842` | `Type 42` | `  24` | `  1552` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 44) |
| **`0x03D3`** | ` 979` | `  2843` | `Type 42` | `  24` | `  1056` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 45) |
| **`0x03D4`** | ` 980` | `  2844` | `Type 42` | `  24` | `  1364` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 46) |
| **`0x03D5`** | ` 981` | `  2845` | `Type 42` | `  24` | `  1200` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 47) |
| **`0x03D6`** | ` 982` | `  2846` | `Type 42` | `  24` | `  1092` | **  15** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 48) |
| **`0x03D7`** | ` 983` | `  2847` | `Type 42` | `  24` | `  1052` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 49) |
| **`0x03D8`** | ` 984` | `  2848` | `Type 42` | `  24` | `  1256` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 50) |
| **`0x03D9`** | ` 985` | `  2849` | `Type 42` | `  24` | `  1292` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 51) |
| **`0x03DA`** | ` 986` | `  2850` | `Type 42` | `  24` | `  1172` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 52) |
| **`0x03DB`** | ` 987` | `  2851` | `Type 42` | `  24` | `  1160` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 53) |
| **`0x03DC`** | ` 988` | `  2852` | `Type 42` | `  24` | `  1052` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 54) |
| **`0x03DD`** | ` 989` | `  2853` | `Type 42` | `  24` | `  1032` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 55) |
| **`0x03DE`** | ` 990` | `  2854` | `Type 42` | `  24` | `  1156` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 56) |
| **`0x03DF`** | ` 991` | `  2855` | `Type 42` | `  24` | `  1160` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 57) |
| **`0x03E0`** | ` 992` | `  2856` | `Type 42` | `  24` | `   808` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 58) |
| **`0x03E1`** | ` 993` | `  2857` | `Type 42` | `  24` | `  1028` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 59) |
| **`0x03E2`** | ` 994` | `  2858` | `Type 42` | `  24` | `  1136` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 60) |
| **`0x03E3`** | ` 995` | `  2859` | `Type 42` | `  24` | `  1112` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 61) |
| **`0x03E4`** | ` 996` | `  2860` | `Type 42` | `  24` | `  1112` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 62) |
| **`0x03E5`** | ` 997` | `  2861` | `Type 42` | `  24` | `   696` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 63) |
| **`0x03E6`** | ` 998` | `  2862` | `Type 42` | `  24` | `   688` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 64) |
| **`0x03E7`** | ` 999` | `  2863` | `Type 42` | `  24` | `   956` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 65) |
| **`0x03E8`** | `1000` | `  2864` | `Type 42` | `  24` | `   788` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 66) |
| **`0x03E9`** | `1001` | `  2865` | `Type 42` | `  24` | `  1028` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 67) |
| **`0x03EA`** | `1002` | `  2866` | `Type 42` | `  24` | `  1032` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 68) |
| **`0x03EB`** | `1003` | `  2867` | `Type 42` | `  24` | `  1100` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 69) |
| **`0x03EC`** | `1004` | `  2868` | `Type 42` | `  24` | `  1092` | **  17** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 70) |
| **`0x03ED`** | `1005` | `  2869` | `Type 42` | `  24` | `   960` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 71) |
| **`0x03EE`** | `1006` | `  2870` | `Type 42` | `  24` | `  1384` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 72) |
| **`0x03EF`** | `1007` | `  2871` | `Type 42` | `  24` | `  1016` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 73) |
| **`0x03F0`** | `1008` | `  2872` | `Type 42` | `  24` | `   736` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 74) |
| **`0x03F1`** | `1009` | `  2873` | `Type 42` | `  24` | `   916` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 75) |
| **`0x03F2`** | `1010` | `  2874` | `Type 42` | `  24` | `  1156` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 76) |
| **`0x03F3`** | `1011` | `  2875` | `Type 42` | `  24` | `   768` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 77) |
| **`0x03F4`** | `1012` | `  2876` | `Type 42` | `  24` | `  1024` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 78) |
| **`0x03F5`** | `1013` | `  2877` | `Type 42` | `  24` | `   840` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 79) |
| **`0x03F6`** | `1014` | `  2878` | `Type 42` | `  24` | `  1128` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 80) |
| **`0x03F7`** | `1015` | `  2879` | `Type 42` | `  24` | `   936` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 81) |
| **`0x03F8`** | `1016` | `  2880` | `Type 42` | `  24` | `  1088` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 82) |
| **`0x03F9`** | `1017` | `  2881` | `Type 42` | `  24` | `   856` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 83) |
| **`0x03FA`** | `1018` | `  2882` | `Type 42` | `  24` | `  1104` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 84) |
| **`0x03FB`** | `1019` | `  2883` | `Type 42` | `  24` | `   796` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 85) |
| **`0x03FC`** | `1020` | `  2884` | `Type 42` | `  24` | `   860` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 86) |
| **`0x03FD`** | `1021` | `  2885` | `Type 42` | `  24` | `   860` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 87) |
| **`0x03FE`** | `1022` | `  2886` | `Type 42` | `  24` | `   952` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 88) |
| **`0x03FF`** | `1023` | `  2887` | `Type 42` | `  24` | `   912` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 89) |
| **`0x0400`** | `1024` | `  2888` | `Type 42` | `  24` | `   824` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 90) |
| **`0x0401`** | `1025` | `  2889` | `Type 42` | `  24` | `   976` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 91) |
| **`0x0402`** | `1026` | `  2890` | `Type 42` | `  24` | `   748` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 92) |
| **`0x0403`** | `1027` | `  2891` | `Type 42` | `  24` | `   968` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 93) |
| **`0x0404`** | `1028` | `  2892` | `Type 42` | `  24` | `  1020` | **  12** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 94) |
| **`0x0405`** | `1029` | `  2893` | `Type 42` | `  24` | `   724` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 95) |
| **`0x0406`** | `1030` | `  2894` | `Type 42` | `  24` | `   952` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 96) |
| **`0x0407`** | `1031` | `  2895` | `Type 42` | `  24` | `   920` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 97) |
| **`0x0408`** | `1032` | `  2896` | `Type 42` | `  24` | `   652` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 98) |
| **`0x0409`** | `1033` | `  2897` | `Type 42` | `  24` | `   836` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 99) |
| **`0x040A`** | `1034` | `  2898` | `Type 42` | `  24` | `   700` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 100) |
| **`0x040B`** | `1035` | `  2899` | `Type 42` | `  24` | `   692` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 101) |
| **`0x040C`** | `1036` | `  2900` | `Type 42` | `  24` | `   556` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 102) |
| **`0x040D`** | `1037` | `  2901` | `Type 42` | `  24` | `   544` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 103) |
| **`0x040E`** | `1038` | `  2902` | `Type 42` | `  24` | `  1052` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 104) |
| **`0x040F`** | `1039` | `  2903` | `Type 42` | `  24` | `   704` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 105) |
| **`0x0410`** | `1040` | `  2904` | `Type 42` | `  24` | `   712` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 106) |
| **`0x0411`** | `1041` | `  2905` | `Type 42` | `  24` | `   600` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 107) |
| **`0x0412`** | `1042` | `  2906` | `Type 42` | `  24` | `   692` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 108) |
| **`0x0413`** | `1043` | `  2907` | `Type 42` | `  24` | `   732` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 109) |
| **`0x0414`** | `1044` | `  2908` | `Type 42` | `  24` | `   820` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 110) |
| **`0x0415`** | `1045` | `  2909` | `Type 42` | `  24` | `   896` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 111) |
| **`0x0416`** | `1046` | `  2910` | `Type 42` | `  24` | `   740` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 112) |
| **`0x0417`** | `1047` | `  2911` | `Type 42` | `  24` | `   868` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 113) |
| **`0x0418`** | `1048` | `  2912` | `Type 42` | `  24` | `   480` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 114) |
| **`0x0419`** | `1049` | `  2913` | `Type 42` | `  24` | `   640` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 115) |
| **`0x041A`** | `1050` | `  2914` | `Type 42` | `  24` | `   484` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 116) |
| **`0x041B`** | `1051` | `  2915` | `Type 42` | `  24` | `   868` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 117) |
| **`0x041C`** | `1052` | `  2916` | `Type 42` | `  24` | `   956` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 118) |
| **`0x041D`** | `1053` | `  2917` | `Type 42` | `  24` | `   608` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 119) |
| **`0x041E`** | `1054` | `  2918` | `Type 42` | `  24` | `   428` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 120) |
| **`0x041F`** | `1055` | `  2919` | `Type 42` | `  24` | `   936` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 121) |
| **`0x0420`** | `1056` | `  2920` | `Type 42` | `  24` | `   792` | **   6** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 122) |
| **`0x0421`** | `1057` | `  2921` | `Type 42` | `  24` | `   508` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 123) |
| **`0x0422`** | `1058` | `  2922` | `Type 42` | `  24` | `   352` | **  10** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 124) |
| **`0x0423`** | `1059` | `  2923` | `Type 42` | `  24` | `   352` | **  10** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 125) |
| **`0x0424`** | `1060` | `  2924` | `Type 42` | `  24` | `   352` | **  10** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 126) |
| **`0x0425`** | `1061` | `  2925` | `Type 42` | `  24` | `   652` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 127) |
| **`0x0426`** | `1062` | `  2926` | `Type 42` | `  24` | `   712` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 128) |
| **`0x0427`** | `1063` | `  2927` | `Type 42` | `  24` | `   532` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 129) |
| **`0x0428`** | `1064` | `  2928` | `Type 42` | `  24` | `   500` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 130) |
| **`0x0429`** | `1065` | `  2929` | `Type 42` | `  24` | `   724` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 131) |
| **`0x042A`** | `1066` | `  2930` | `Type 42` | `  24` | `   896` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 132) |
| **`0x042B`** | `1067` | `  2931` | `Type 42` | `  24` | `   536` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 133) |
| **`0x042C`** | `1068` | `  2932` | `Type 42` | `  24` | `   716` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 134) |
| **`0x042D`** | `1069` | `  2933` | `Type 42` | `  24` | `   728` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 135) |
| **`0x042E`** | `1070` | `  2934` | `Type 42` | `  24` | `   688` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 136) |
| **`0x042F`** | `1071` | `  2935` | `Type 42` | `  24` | `   356` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 137) |
| **`0x0430`** | `1072` | `  2936` | `Type 42` | `  24` | `   668` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 138) |
| **`0x0431`** | `1073` | `  2937` | `Type 42` | `  24` | `   752` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 139) |
| **`0x0432`** | `1074` | `  2938` | `Type 42` | `  24` | `   712` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 140) |
| **`0x0433`** | `1075` | `  2939` | `Type 42` | `  24` | `   608` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 141) |
| **`0x0434`** | `1076` | `  2940` | `Type 42` | `  24` | `   620` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 142) |
| **`0x0435`** | `1077` | `  2941` | `Type 42` | `  24` | `   828` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 143) |
| **`0x0436`** | `1078` | `  2942` | `Type 42` | `  24` | `   728` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 144) |
| **`0x0437`** | `1079` | `  2943` | `Type 42` | `  24` | `   556` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 145) |
| **`0x0438`** | `1080` | `  2944` | `Type 42` | `  24` | `   944` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 146) |
| **`0x0439`** | `1081` | `  2945` | `Type 42` | `  24` | `   428` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 147) |
| **`0x043A`** | `1082` | `  2946` | `Type 42` | `  24` | `   720` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 148) |
| **`0x043B`** | `1083` | `  2947` | `Type 42` | `  24` | `   612` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 149) |
| **`0x043C`** | `1084` | `  2948` | `Type 42` | `  24` | `   252` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 150) |
| **`0x043D`** | `1085` | `  2949` | `Type 42` | `  24` | `   244` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 151) |
| **`0x043E`** | `1086` | `  2950` | `Type 42` | `  24` | `   248` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 152) |
| **`0x043F`** | `1087` | `  2951` | `Type 42` | `  24` | `   224` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 153) |
| **`0x0440`** | `1088` | `  2952` | `Type 42` | `  24` | `   228` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 154) |
| **`0x0441`** | `1089` | `  2953` | `Type 42` | `  24` | `   540` | **   3** | ` 1` | Ch.5/6: Torneko Item Appraisal Master System (Item Band 155) |
| **`0x0442`** | `1090` | `  2954` | `Type 42` | `  24` | `  1416` | **   7** | ` 1` | Party Chat Global Context Module (Bank 0) |
| **`0x0443`** | `1091` | `  2955` | `Type 42` | `  24` | `  1420` | **   7** | ` 1` | Party Chat Global Context Module (Bank 1) |
| **`0x0444`** | `1092` | `  2956` | `Type 42` | `  24` | `  1724` | **   7** | ` 1` | Party Chat Global Context Module (Bank 2) |
| **`0x0445`** | `1093` | `  2957` | `Type 42` | `  24` | `  1416` | **   7** | ` 1` | Party Chat Global Context Module (Bank 3) |
| **`0x0446`** | `1094` | `  2958` | `Type 42` | `  24` | `  1292` | **   7** | ` 1` | Party Chat Global Context Module (Bank 4) |
| **`0x0447`** | `1095` | `  2959` | `Type 42` | `  24` | `  1552` | **   7** | ` 1` | Party Chat Global Context Module (Bank 5) |
| **`0x0448`** | `1096` | `  2960` | `Type 42` | `  24` | `  1672` | **   7** | ` 1` | Party Chat Global Context Module (Bank 6) |
| **`0x0449`** | `1097` | `  2961` | `Type 42` | `  24` | `  1476` | **   7** | ` 1` | Party Chat Global Context Module (Bank 7) |
| **`0x044A`** | `1098` | `  2962` | `Type 42` | `  24` | `  1684` | **   7** | ` 1` | Party Chat Global Context Module (Bank 8) |
| **`0x044B`** | `1099` | `  2963` | `Type 42` | `  24` | `  1584` | **   7** | ` 1` | Party Chat Global Context Module (Bank 9) |
| **`0x044C`** | `1100` | `  2964` | `Type 42` | `  24` | `  1588` | **   7** | ` 1` | Party Chat Global Context Module (Bank 10) |
| **`0x044D`** | `1101` | `  2965` | `Type 42` | `  24` | `  1620` | **   7** | ` 1` | Party Chat Global Context Module (Bank 11) |
| **`0x044E`** | `1102` | `  2966` | `Type 42` | `  24` | `  1568` | **   7** | ` 1` | Party Chat Global Context Module (Bank 12) |
| **`0x044F`** | `1103` | `  2967` | `Type 42` | `  24` | `  1600` | **   7** | ` 1` | Party Chat Global Context Module (Bank 13) |
| **`0x0450`** | `1104` | `  2968` | `Type 42` | `  24` | `  1372` | **   7** | ` 1` | Party Chat Global Context Module (Bank 14) |
| **`0x0451`** | `1105` | `  2969` | `Type 42` | `  24` | `  1500` | **   7** | ` 1` | Party Chat Global Context Module (Bank 15) |
| **`0x0452`** | `1106` | `  2970` | `Type 42` | `  24` | `  1620` | **   7** | ` 1` | Party Chat Global Context Module (Bank 16) |
| **`0x0453`** | `1107` | `  2971` | `Type 42` | `  24` | `  1404` | **   7** | ` 1` | Party Chat Global Context Module (Bank 17) |
| **`0x0454`** | `1108` | `  2972` | `Type 42` | `  24` | `  1516` | **   7** | ` 1` | Party Chat Global Context Module (Bank 18) |
| **`0x0455`** | `1109` | `  2973` | `Type 42` | `  24` | `  1472` | **   7** | ` 1` | Party Chat Global Context Module (Bank 19) |
| **`0x0456`** | `1110` | `  2974` | `Type 42` | `  24` | `  1372` | **   7** | ` 1` | Party Chat Global Context Module (Bank 20) |
| **`0x0457`** | `1111` | `  2975` | `Type 42` | `  24` | `  1620` | **   7** | ` 1` | Party Chat Global Context Module (Bank 21) |
| **`0x0458`** | `1112` | `  2976` | `Type 42` | `  24` | `  1396` | **   7** | ` 1` | Party Chat Global Context Module (Bank 22) |
| **`0x0459`** | `1113` | `  2977` | `Type 42` | `  24` | `  1656` | **   7** | ` 1` | Party Chat Global Context Module (Bank 23) |
| **`0x045A`** | `1114` | `  2978` | `Type 42` | `  24` | `  1472` | **   7** | ` 1` | Party Chat Global Context Module (Bank 24) |
| **`0x045B`** | `1115` | `  2979` | `Type 42` | `  24` | `  1612` | **   7** | ` 1` | Party Chat Global Context Module (Bank 25) |
| **`0x045C`** | `1116` | `  2980` | `Type 42` | `  24` | `  1460` | **   7** | ` 1` | Party Chat Global Context Module (Bank 26) |
| **`0x045D`** | `1117` | `  2981` | `Type 42` | `  24` | `  1552` | **   7** | ` 1` | Party Chat Global Context Module (Bank 27) |
| **`0x045E`** | `1118` | `  2982` | `Type 42` | `  24` | `  1600` | **   7** | ` 1` | Party Chat Global Context Module (Bank 28) |
| **`0x045F`** | `1119` | `  2983` | `Type 42` | `  24` | `  1560` | **   7** | ` 1` | Party Chat Global Context Module (Bank 29) |
| **`0x0460`** | `1120` | `  2984` | `Type 42` | `  24` | `  1588` | **   7** | ` 1` | Party Chat Global Context Module (Bank 30) |
| **`0x0461`** | `1121` | `  2985` | `Type 42` | `  24` | `  1428` | **   7** | ` 1` | Party Chat Global Context Module (Bank 31) |
| **`0x0462`** | `1122` | `  2986` | `Type 42` | `  24` | `  1256` | **   7** | ` 1` | Party Chat Global Context Module (Bank 32) |
| **`0x0463`** | `1123` | `  2987` | `Type 42` | `  24` | `  1436` | **   7** | ` 1` | Party Chat Global Context Module (Bank 33) |
| **`0x0464`** | `1124` | `  2988` | `Type 42` | `  24` | `  1508` | **   7** | ` 1` | Party Chat Global Context Module (Bank 34) |
| **`0x0465`** | `1125` | `  2989` | `Type 42` | `  24` | `  1400` | **   7** | ` 1` | Party Chat Global Context Module (Bank 35) |
| **`0x0466`** | `1126` | `  2990` | `Type 42` | `  24` | `  1448` | **   7** | ` 1` | Party Chat Global Context Module (Bank 36) |
| **`0x0467`** | `1127` | `  2991` | `Type 42` | `  24` | `  1488` | **   7** | ` 1` | Party Chat Global Context Module (Bank 37) |
| **`0x0468`** | `1128` | `  2992` | `Type 42` | `  24` | `  1540` | **   7** | ` 1` | Party Chat Global Context Module (Bank 38) |
| **`0x0469`** | `1129` | `  2993` | `Type 42` | `  24` | `  1684` | **   7** | ` 1` | Party Chat Global Context Module (Bank 39) |
| **`0x046A`** | `1130` | `  2994` | `Type 42` | `  24` | `  1424` | **   7** | ` 1` | Party Chat Global Context Module (Bank 40) |
| **`0x046B`** | `1131` | `  2995` | `Type 42` | `  24` | `  1400` | **   7** | ` 1` | Party Chat Global Context Module (Bank 41) |
| **`0x046C`** | `1132` | `  2996` | `Type 42` | `  24` | `  1504` | **   7** | ` 1` | Party Chat Global Context Module (Bank 42) |
| **`0x046D`** | `1133` | `  2997` | `Type 42` | `  24` | `  1448` | **   7** | ` 1` | Party Chat Global Context Module (Bank 43) |
| **`0x046E`** | `1134` | `  2998` | `Type 42` | `  24` | `  1480` | **   7** | ` 1` | Party Chat Global Context Module (Bank 44) |
| **`0x046F`** | `1135` | `  2999` | `Type 42` | `  24` | `  1640` | **   7** | ` 1` | Party Chat Global Context Module (Bank 45) |
| **`0x0470`** | `1136` | `  3000` | `Type 42` | `  24` | `  1408` | **   7** | ` 1` | Party Chat Global Context Module (Bank 46) |
| **`0x0471`** | `1137` | `  3001` | `Type 42` | `  24` | `  1564` | **   7** | ` 1` | Party Chat Global Context Module (Bank 47) |
| **`0x0472`** | `1138` | ` 50205` | `Type 40` | `  24` | `  4412` | **  30** | ` 1` | Party Chat Global Context Module (Bank 48) |
| **`0x0474`** | `1140` | `104210` | `Overlay` | `  24` | `   974` | **  27** | ` 1` | Church Save and Memory Card Interface |
| **`0x0482`** | `1154` | ` 18709` | `Type 40` | `  24` | `  6172` | **   4** | ` 1` | System Status Panels, Mini-Medal Registry and Monster Book |
| **`0x048B`** | `1163` | `105540` | `Overlay` | `2028` | ` 11112` | ** 817** | `18` | Combat Engine Primary Overlay and Monster Bestiary |
| **`0x048C`** | `1164` | `   362` | `EXE` | `  24` | `  6762` | ** 779** | ` 1` | Font 1 System Menus, Tactics, Spells, Items and Delta-Locks |
| **`0x048D`** | `1165` | `   362` | `EXE` | `  24` | `   192` | **   0** | ` 1` | Font 1 Secondary System Parameter Arrays |
| **`0x048F`** | `1167` | `   362` | `EXE` | `  24` | `  7440` | ** 337** | ` 1` | Church Priest Service Text, Confession, Cures and Divination |

---

## 4. THE 40 MULTI-COPY TIDS: PHYSICAL REDUNDANCY ANALYSIS

In the PSX CD-ROM environment, seek latency during map streaming was a critical bottleneck. Yamana's engine addressed this by physically duplicating 40 high-frequency dialogue and facility TIDs across multiple disc sectors:

| TID (Hex) | Primary Sector | Copies | Physical Duplicate Sectors | Primary Operational Role |
|---|---|---|---|---|
| **`0x0020`** | `4285` | **190** | `4397, 19708, 19838, 20332, 20456, 21047, 21175, 21524, 21652, 22684, 22809, 23285, 23436, 24211, 24338, 25452, 25594, 25775, 25854, 25986, 26092, 26228, 26334, 26442, 26585, 26692, 26800, 26966, 27081, 27190, 27297, 27405, 27577, 27692, 27807, 27914, 28022, 28133, 28307, 28422, 28537, 28613, 28710, 28779, 28870, 28987, 29101, 29263, 29378, 29485, 29592, 29680, 29780, 29849, 29925, 30038, 30129, 30205, 30342, 30454, 30628, 30747, 30864, 30971, 31079, 31159, 31239, 31327, 31397, 31523, 31676, 31780, 31890, 32008, 32120, 32208, 32307, 32413, 32483, 32620, 32751, 32870, 33026, 33133, 33203, 33273, 33354, 33434, 33565, 33702, 33807, 33879, 33967, 34066, 34136, 34288, 34422, 34542, 34622, 34703, 34790, 34917, 34987, 35090, 35233, 35392, 35513, 35635, 35707, 35778, 35886, 35990, 36096, 36169, 36298, 36404, 36504, 36592, 36714, 36806, 36891, 36971, 37059, 37158, 37227, 37334, 37442, 37561, 37736, 37852, 37989, 38079, 38155, 38267, 38379, 38513, 39023, 39154, 39799, 39931, 41655, 41779, 42072, 42195, 42885, 43012, 43982, 44114, 45758, 45888, 46264, 46390, 48492, 48619, 49563, 49700, 51168, 51351, 52079, 52215, 53167, 53299, 54002, 54129, 54540, 55332, 55458, 56209, 56340, 56827, 56951, 57132, 57261, 57741, 57865, 59233, 59368, 60023, 60154, 60739, 60877, 61091, 61220, 61706, 61826, 61936, 62069, 63931, 106519` | World Map Coordinate Labels and Town/Dungeon Registry |
| **`0x0021`** | `4285` | **68** | `4397, 19708, 19838, 20332, 20456, 21047, 21175, 21524, 21652, 22684, 22809, 23285, 23436, 24211, 24338, 25452, 25594, 39023, 39154, 39799, 39931, 41655, 41779, 42072, 42195, 42885, 43012, 43982, 44114, 45758, 45888, 46264, 46390, 48492, 48619, 49563, 49700, 51168, 51351, 52079, 52215, 53167, 53299, 54002, 54129, 55332, 55458, 56209, 56340, 56827, 56951, 57132, 57261, 57741, 57865, 59233, 59368, 60023, 60154, 60739, 60877, 61091, 61220, 61706, 61826, 61936, 62069` | Endor Mega-Block (Commercial Capital, Casino, Immigrant Hub) |
| **`0x0022`** | `54540` | **2** | `63931` | Endor Castle State Rooms and Royal Audience Chambers |
| **`0x0023`** | `25775` | **79** | `25854, 25986, 26092, 26228, 26334, 26442, 26585, 26692, 26800, 26966, 27081, 27190, 27297, 27405, 27577, 27692, 27807, 27914, 28022, 28133, 28307, 28422, 28710, 28870, 28987, 29101, 29263, 29378, 29485, 29780, 29925, 30205, 30342, 30454, 30628, 30747, 30864, 30971, 31397, 31523, 31676, 31780, 31890, 32008, 32307, 32483, 32620, 32751, 32870, 33026, 33434, 33565, 33702, 34066, 34136, 34288, 34422, 34790, 34987, 35090, 35233, 35392, 35513, 35778, 35886, 35990, 36169, 36592, 36806, 37158, 37227, 37334, 37442, 37561, 37736, 37852, 38155, 38267` | Immigrant Town Growth and Evolution (Stages 1 through 5) |
| **`0x0024`** | `28537` | **39** | `28613, 28779, 29592, 29680, 29849, 30038, 30129, 31079, 31159, 31239, 31327, 32120, 32208, 32413, 33133, 33203, 33273, 33354, 33807, 33879, 33967, 34542, 34622, 34703, 34917, 35635, 35707, 36096, 36298, 36404, 36504, 36714, 36891, 36971, 37059, 37989, 38079, 38379` | Special Event NPCs (King Leo, Queen, Eggula and Chikila) |
| **`0x0085`** | `56827` | **2** | `56951` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x0095`** | `43982` | **2** | `44114` | Ch.1: Strathross Dungeon, Loch Tur Shrine and Master Healie |
| **`0x00A2`** | `20332` | **2** | `20456` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00A9`** | `41655` | **2** | `41779` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x00B4`** | `4285` | **2** | `4397` | Ch.2: Zamoksva Castle, Princess Alena Escape and Court |
| **`0x0148`** | `54002` | **2** | `54129` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x014F`** | `48492` | **2** | `48619` | Ch.2: Endor Tourney, Santeem Ruins and Chapter Finale |
| **`0x0163`** | `53167` | **2** | `53299` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0165`** | `59233` | **2** | `59368` | Ch.3: Lakanaba Weapon Shop, Torneko Taloon and Family |
| **`0x0171`** | `39799` | **2** | `39931` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0175`** | `45758` | **2** | `45888` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0178`** | `46264` | **2** | `46390` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017A`** | `24211` | **2** | `24338` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x017E`** | `52079` | **2** | `52215` | Ch.3: Ballymoral Castle, Prince Alex Diplomacy and Prison |
| **`0x0218`** | `60739` | **2** | `60877` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x021F`** | `57741` | **2** | `57865` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0228`** | `51168` | **2** | `51351` | Ch.4: Aubout du Monde Theater, Meena and Maya Opening |
| **`0x0254`** | `55332` | **2** | `55458` | Ch.4: Aktemto Gunpowder Mines, Night Travel and Ominous Ruins |
| **`0x0283`** | `61091` | **2** | `61220` | Ch.4: Palais de Leon Infiltration, Orlin and Dungeon Guards |
| **`0x02BD`** | `60023` | **2** | `60154` | Ch.4: Havrenence Port, Ship Charter and Sea Voyage |
| **`0x02C1`** | `22684` | **2** | `22809` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02CC`** | `25452` | **2** | `25594` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DA`** | `19708` | **2** | `19838` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02DD`** | `21047` | **2** | `21175` | Ch.5: Secret Hero Village Destruction and Cynthia Sacrifice |
| **`0x02E3`** | `21524` | **2** | `21652` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x02E5`** | `39023` | **2** | `39154` | Ch.5: Casabranca, Endor Reunion and Desert Wagon Escort |
| **`0x0308`** | `49563` | **2** | `49700` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0316`** | `42072` | **2** | `42195` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031B`** | `61936` | **2** | `62069` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031C`** | `61706` | **2** | `61826` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x031E`** | `42885` | **2** | `43012` | Ch.5: Mintos, Dying Kiryl, Fever Grass and Parthenia |
| **`0x0322`** | `57132` | **2** | `57261` | Ch.5: Porthtrunnel, Ship Construction and Torneko Voyage |
| **`0x034A`** | `56209` | **2** | `56340` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0351`** | `23285` | **2** | `23436` | Ch.5: Femiscyra Castle, Thief Trial and Zenithian Shield |
| **`0x0364`** | `20570` | **16** | `22221, 24771, 38572, 38774, 40755, 42503, 43740, 45556, 47222, 49073, 50455, 51776, 53683, 57379, 60279` | Ch.5: Rosaville, Flute of Revelation, Riverton and Colossus |

> [!WARNING]
> **Multi-Copy Synchronization Rule:** If a multi-copy TID is modified or re-encoded during localization, **all physical copies on disc must be updated simultaneously**. Modifying sector 61,707 while leaving duplicate sector 63,569 in pristine state causes regional dialogue regressions and pointer collisions when entering the duplicate sector's zone.

---

## 5. CRITICAL SYSTEM & OVERLAY TIDS (DEEP ARCHITECTURAL DIVE)

### 5.1 TID 0x0020: The Overworld Location Index
* **LBA Sector:** 106,157 | **Length:** 1,864 Bytes | **Strings:** 396
* **Function:** Houses all overworld town, dungeon, shrine, and sub-map location names displayed by the map blitter.
* **Referrer Constraint:** 97 direct EXE referrers point into this block. Any string length expansion must preserve offset alignment or remap all 97 call sites in `SLPM_869.16`.

### 5.2 TID 0x0021: The Endor Mega-Block
* **LBA Sector:** 61,707 | **Length:** 93,140 Bytes | **Strings:** 1,087
* **Function:** The largest dialogue block in the game. Contains all Chapter 3 commercial dialogues, Chapter 2 tournament spectator interactions, casino games, and Immigrant Town base sequences.
* **Safety Boundary:** With 93,140 bytes, TID 0x0021 operates at **71.1% of the absolute 128 KB 20-bit address limit**. Recompressed English text must not exceed 131,072 bytes.

### 5.3 TID 0x048B: The Battle Engine Overlay
* **LBA Sector:** 105,540 + 0x07EC | **Length:** 11,112 Bytes | **Strings:** 817
* **Function:** Core battle action messages, spell casting notifications, damage output, and monster bestiary strings.
* **The Freeze Hazard:** Houses token `{7F3D}` (monster substitution). Over 68 stale referrer words in the 18 Type-46 overlay copies point to pristine offsets. Must be remapped using `patch_overlay_duplicates.py` to prevent runaway zero-FPS battle freezes.

### 5.4 TID 0x048C: The Font 1 Fixed-Grid Matrix
* **Location:** Inside `SLPM_869.16` @ File Offset `0x99624` | **Length:** 6,762 Bytes | **Strings:** 779
* **Function:** Houses all Font 1 status menus, tactical commands, items, spells, and equipment.
* **Delta-Lock Invariants:** Command array SIDs 773–778 must preserve relative bit deltas (`44, 36, 50, 66`) to prevent VRAM blit corruption.

### 5.5 TID 0x048F: The Church Priest System Block
* **Location:** Inside `SLPM_869.16` @ File Offset `0x99624` | **Length:** 7,440 Bytes | **Strings:** 337
* **Function:** Church confession, saving, resurrection, poison detox, and divination.
* **Wild Tokens:** Contains 7 previously undocumented control codes (`7F06`, `7F10`, `7F19`, `7F1B`, `7F1C`, `7F1E`, `7F35`) essential for Memory Card slot formatting and gold shortfall evaluation.

### 5.6 TID 0x006C: The Burland Throne Room Event Script (Type-39)
* **LBA Sector:** 12,180 | **Allocated Slot:** Exactly 1,588 Bytes
* **The Historical Prologue Freeze:** Markus Schroeder's early Java patcher expanded this script from 1,588 to 1,592 bytes (+4 bytes). This 4-byte overflow shifted every subsequent sector on disc forward, causing CD-ROM DMA Channel 3 to read misaligned data and permanently locking the console upon leaving the throne room. Must remain clamped to $\le 1588\text{ bytes}$.

---

## 6. IN-PLACE MODIFICATION & RELOCATION RULES

1. **Zero-Sector-Shift Invariant:** The CD-ROM file allocation table and sub-map jump routines rely on exact LBA sector addresses. No file or sub-block may expand beyond its allocated sector budget.
2. **20-Bit Addressing Boundary:** No text block payload may exceed $131,072\text{ bytes}$ ($2^{20}\text{ bits}$).
3. **Multi-Copy Coherence:** All 40 redundant physical TID copies must receive identical re-encoded payloads.
4. **Terminal Leaf Isolation:** Every string sequence must terminate with a valid `{0000}` leaf before the next string offset begins.
