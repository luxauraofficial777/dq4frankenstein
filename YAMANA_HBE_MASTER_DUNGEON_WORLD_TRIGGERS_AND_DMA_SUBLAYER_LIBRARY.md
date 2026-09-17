# PLAYSTATION 1 SYSTEMS ENGINEERING MASTER LIBRARY: DUNGEON SUB-LAYER DMA MAPS, WORLD EVENT TRIGGERS & GUEST INTEGRATION
**Target Binary:** `SLPM_869.16` (Sony PlayStation 1 / MIPS R3000A @ 33.8688 MHz)  
**Archive Container:** HeartBeat Engine (`HBD1PS1D.Q41` / Manabu Yamana Architecture)  
**Disc Source:** `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` (156,487 Sectors / 368,057,424 Bytes)  
**Domain:** Global Dungeon 3D Spatial Maps, CD-ROM DMA Channel 3 Sub-layer Streaming, Type-21/35/36/39/40 Sub-Block Specifications, Condition Locks, Event Flag Registers, Bus Contention Arbitration, Chapter 5 Olin (Oren) Reunion & Guest Slot 8 Integration, and Standardized Multi-Agent Orchestration Interfaces  

---

## 1. THE PLAYSTATION 1 3D SPATIAL DMA PIPELINE & MEMORY TOPOLOGY

In the HeartBeat Engine architecture, dungeons are not stored as monolithic flat tile arrays. Instead, each floor stratum is partitioned into discrete 3D spatial sub-blocks streamed dynamically across **CD-ROM DMA Channel 3 (`0x1F8010B0`)** into working memory.

```
0x80000000 +-----------------------------------------------------------------------+
           | PSX Kernel, Hardware Exception Vectors & Scratchpad D-RAM             | (64 KB)
           |   0x1F800000: Fast Scratchpad D-RAM (1 KB)                            |
           |   0x1F800100: Scratchpad Roster & Geometry Swap Buffer (16 Bytes)      |
0x80010000 +-----------------------------------------------------------------------+
           | SLPM_869.16 Main Executable Text Segment                              | (480 KB)
           |   0x80018C20: flush_dcache / MIPS L1 D-Cache Invalidation             |
           |   0x80025D10: vblank_blit_queue_flush (GPU DMA Ch.2 Dispatcher)       |
           |   0x80060C38: enroll_active_party_member Subroutine                   |
           |   0x800654B0: evaluate_party_chat_context Handler                     |
           |   0x80072A10: Tile Collision & Spatial Passability Dispatcher         |
           |   0x80072B80: Door Interaction & Breach Verification Routine          |
           |   0x80073100: Global Spatial Transition Dispatcher (Stairs/Warps)     |
           |   0x800732A8: Dynamic Heightmap & Elevator Lift Handler               |
           |   0x800735FC: get_event_flag Subroutine (Global Bitfield Resolver)    |
           |   0x80073698: set_event_flag Subroutine (Global Bitfield Setter)      |
           |   0x80074E10: swap_party_roster_slots (Fast Scratchpad Memory Copy)   |
           |   0x800751A0: Flying Shoes Parabolic Trajectory Jump Handler          |
           |   0x8008F280: 32-bit Packed Referrer Word Resolver                    |
           |   0x8008F59C: Canonical Huffman Bitstream Text Decoder                |
           |   0x8008F810: dma3_sync_stream_decompress (DQLZS Pipeline)           |
           |   0x8009A120: CD-ROM Timeout & Spindle Recovery Handler               |
           |   0x8009A240: Mode 2 Form 1 Checksum / EDC-ECC Error Trap            |
0x800B0000 +-----------------------------------------------------------------------+
           | Global Static Variables, VRAM Latch Tables                            | (192 KB)
           |   0x800E8000: CD-ROM DMA Channel 3 Streaming Ring Buffer (16 KB)      |
0x800F0000 +-----------------------------------------------------------------------+
           | HeartBeat Engine Dynamic Work Buffers                                 | (64 KB)
           |   0x800F4AC0: Sub-Window Coordinate Page Table Base                   |
           |   0x800F4DF0: Primary Huffman Text Work Buffer (512 B)                |
           |   0x800F8010: Chapter 1 Event Flags (Bytes 0x10 - 0x17)               |
           |   0x800F8020: Chapter 2 Event Flags (Bytes 0x20 - 0x2F)               |
           |   0x800F8030: Chapter 3 Event Flags (Bytes 0x30 - 0x3F)               |
           |   0x800F8040: Chapter 4 Event Flags (Bytes 0x40 - 0x4F)               |
           |   0x800F8050: Chapter 5 & 6 Event Flags (Bytes 0x50 - 0x6F)           |
           |   0x800F8060: Chest Opened Bitfield Array (Flags 0x0030 - 0x005F)     |
           |   0x800F83BF: Active Party Member Count Register (1 to 10)            |
           |   0x800F83C0: 10-Slot Party Roster Array (140 Bytes total)            |
           |   0x800F8410: Traversal Mode (0=Foot, 1=Wagon, 2=Ship, 3=Balloon)     |
           |   0x800F8412: Wagon Follow Latch (0=Detached in Caves, 1=Overworld)   |
           |   0x800F8414: Maritime Coastal Coordinates & Docking Latch            |
           |   0x800F8418: Party Capability Bitfield (Bit 2: DOOR_BREACH_CAPABLE)   |
           |   0x800F841A: Balloon Altitude Latch (0=Ground, 2=Cruising)           |
           |   0x800F8430: Slot 8 Guest Escort Struct (14 Bytes / Olin: ID 0x88)   |
           |   0x800F84E4: Video Display Blanking Latch (0=Blackout, 1=Unblank)    |
           |   0x800F9100: Deferred VRAM Blit Command Queue Buffer (256 B)         |
0x80100000 +-----------------------------------------------------------------------+
           | Dynamic Dungeon & Spatial Stratum Buffers                             | (256 KB)
           |   0x8011F000: Dungeon Floor Dynamic Module Base                       |
           |   0x80180000: Type-21 3D Collision Mesh & Geometry Buffer (128 KB)    |
           |   0x801A0000: Type-35/36 Entity & Spatial Trigger Matrix (128 KB)     |
           |   0x8018E724: Type-39 Event Script Bytecode Table                     |
           |   0x8018D768: Type-40 Huffman Dialogue String Block                   |
0x80200000 +-----------------------------------------------------------------------+
```

---

## 2. SUB-BLOCK ANATOMY OF A 3D DUNGEON MAP BLOCK

Every dungeon stratum in `HBD1PS1D.Q41` is stored as an integrated master block containing between 10 and 23 sub-blocks. For example, **Sector 106980** (Sarokhov Cave B3F Flying Shoes chamber) unpacks the following sub-layer matrix:

```
+-----------+---------+------------+----------+-----------+------------+------------------------------------------------+
| Sub Index | Type ID | Flag Word  | Comp?    | Disc Size | RAM Size   | Sub-Block Function & WRAM Destination Address  |
+-----------+---------+------------+----------+-----------+------------+------------------------------------------------+
| Sub [ 0]  | Type 31 | 0x0000     | Raw      |     288 B |      288 B | Sub-Map Header & Bounding Dimensions           |
| Sub [ 1]  | Type 21 | 0x0000     | Raw      |  14,092 B |   14,092 B | 3D Collision Mesh (Floors, Walls & Steps)      |
| Sub [ 2]  | Type 21 | 0x0000     | Raw      |   4,712 B |    4,712 B | Secondary Collision Mesh & Elevation Steps     |
| Sub [ 3]  | Type 13 | 0x0500     | DQLZS    |   3,892 B |    8,436 B | 3D Texture UV Mapping Coordinates              |
| Sub [ 4]  | Type 06 | 0x0500     | DQLZS    |  60,440 B |  122,656 B | VRAM Tile Pattern Data (Graphics Blitter)      |
| Sub [ 5]  | Type 39 | 0x0500     | DQLZS    |     908 B |    2,800 B | VM Script Bytecode (0x8018E724)                |
| Sub [ 6]  | Type 34 | 0x0000     | Raw      |     168 B |      168 B | Ambient Shading & Lighting Vector Table        |
| Sub [ 7]  | Type 07 | 0x0000     | Raw      |   1,232 B |    1,232 B | Palette CLUT (Color Lookup Tables)             |
| Sub [ 8]  | Type 41 | 0x0000     | Raw      |     180 B |      180 B | VRAM Texture Page Coordinates & Window Mask    |
| Sub [ 9]  | Type 38 | 0x0000     | Raw      |     160 B |      160 B | Floor Shadow & Heightmap Lighting Matrix       |
| Sub [10]  | Type 35 | 0x0000     | Raw      |      36 B |       36 B | Entity Matrices (Chests, NPCs, Doors)          |
| Sub [11]  | Type 36 | 0x0000     | Raw      |     372 B |      372 B | Spatial Trigger Zones & Staircase Warps         |
| Sub [12]  | Type 40 | 0x0000     | Raw      |   1,524 B |    1,524 B | Canonical Huffman Dialogue Block (TID 0x0069)   |
| Sub [13]  | Type 46 | 0x0500     | DQLZS    |   5,148 B |    9,304 B | Monster Encounter Formations & Combat AI       |
| Sub [14]  | Type 37 | 0x0000     | Raw      |      32 B |       32 B | Camera Path Pan Angles, FOV & Rotation Latch   |
+-----------+---------+------------+----------+-----------+------------+------------------------------------------------+
```

---

## 3. MASTER DUNGEON & SUBLAYER DMA LIBRARY (CHAPTERS 1–6)

### 3.1 Chapter 1: Ragnar McRyan Arc

```
========================================================================================================================
DUNGEON: Sarokhov Cave / Lake Cave (イムルの洞窟 / サロホフの洞窟)
LOCATION: Northwest of Izmit / North of Strathross (Chapter 1)
WAGON RESTRICTION: Not applicable (Chapter 1 solo/duo) | PARTY LIMIT: 2 (Ragnar + Healie)
DMA BUFFER DESTINATIONS:
  - Raw Streaming Ring Buffer: 0x800E8000 (D3_MADR = 0x1F8010B0)
  - Type-21 Collision Mesh:   0x80180000 (18,804 Bytes)
  - Type-35/36 Trigger/Entity: 0x801A0000 (408 Bytes)
  - Type-39 Script Bytecode:   0x8018E724 | Type-40 Dialogue: 0x8018D768
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F & B1F Descent     | 107143       | 106781  | 0x0066 | 15   | 104  | Flag 0x0090 (Cave entrance discovered)
B1F Campfire Cavern  | 107066       | 106704  | 0x0067 | 14   | 77   | Flag 0x0092 (Campfire NPC dialogue evaluated)
B2F Subterranean Pool| 107026       | 106664  | 0x0068 | 13   | 40   | Water step passability evaluation (0x80072A10)
B3F Flying Shoes Room| 106980       | 106618  | 0x0069 | 15   | 46   | Flag 0x003A / 0x0098 (Flying Shoes Chest: Item 0x26)
Lower Drainage Strata| 106943       | 106581  | 0x006A | 12   | 37   | Environmental drain trigger
Hidden Alcove        | 106908       | 106546  | 0x006B | 12   | 35   | Secret room inspection trigger
Healie Chamber       | 106864       | 106502  | 0x006C | 15   | 44   | Flag 0x0091 (Healie joins Slot 1 via 0x80060C38)
Shortcut Exit        |  59971       |  59609  | 0x006F | 12   | 28   | One-way drop ledge trigger
========================================================================================================================

========================================================================================================================
DUNGEON: Strathba Tower / Loch Tur Shrine (湖の塔 / 古井戸の底)
LOCATION: Hidden island in Loch Tur lake (Chapter 1 Climax)
WAGON RESTRICTION: Detached | ENTRY CONDITION: Parabolic Flying Shoes Jump (Subroutine 0x800751A0)
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh:   0x80180000 (14,488 to 18,804 Bytes)
  - Type-35/36 Trigger/Entity: 0x801A0000
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
Tower Exterior       |  60481       |  60119  | 0x0078 | 13   | 35   | Flag 0x00A0 (Landed outside tower grounds)
1F Lower Entrance    |  48051       |  47689  | 0x0083 | 13   | 37   | Flag 0x00A2 (Entrance gate unlocked)
2F–3F Trap Mazes     |  48088       |  47726  | 0x0083 | 14   | 42   | Floor hole drops to 1F spikes
4F Children Cell     |  56562       |  56200  | 0x0084 | 15   | 53   | Flag 0x00A6 (Abducted children located)
Summit Boss Lair     | 106701       | 106339  | 0x0087 | 10   | 17   | Flag 0x00B0 (Psaro's Pawn defeated / Ch1 Complete)
========================================================================================================================
```

---

### 3.2 Chapter 2: Princess Alena Arc

```
========================================================================================================================
DUNGEON: Fresnor Cave (フレノールの南の洞窟)
LOCATION: South of Fresnor Village (Chapter 2)
WAGON RESTRICTION: Not applicable (Chapter 2 trio: Alena, Kiryl, Borya)
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh: 0x80180000 (14,056 to 19,632 Bytes)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Cavern Descent   |  14136       |  13774  | 0x0184 | 17   | 146  | Flag 0x00CA (Fake Princess kidnapping triggered)
B2F Treasure Vault   |  14395       |  14033  | 0x0194 | 17   | 130  | Flag 0x003C / 0x00D0 (Golden Bracelet: Item 0x62)
========================================================================================================================

========================================================================================================================
DUNGEON: Birdsong Tower (さえずりの塔)
LOCATION: North of Desert Bazaar / Desert Oasis (Chapter 2)
ENTRY CONDITION: Flag 0x00E0 set (King Zamoksva mute condition diagnosed)
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh: 0x80180000 (23,868 Bytes)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F–2F Spiral Ascent  | 108420       | 108058  | 0x0111 | 16   | 88   | Flag 0x00E4 (Scholar poetry reference unlocked)
3F–4F Perch Walkway  | 108508       | 108146  | 0x0120 | 15   | 74   | Open-air railing drop triggers
Summit Elven Perch   |  63931       |  63569  | 0x0124 | 21   | 152  | Flag 0x003E / 0x00EA (Elven Nectar: Item 0x63)
========================================================================================================================

========================================================================================================================
DUNGEON: Desert Passage & Endor Travel Shrine (東のトンネル / 旅の扉のほこら)
LOCATION: Border between Zamoksva and Endor Continents (Chapter 2)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
Travel Shrine        |  14071       |  13709  | 0x0193 | 18   | 65   | Flag 0x00F4 (Chancellor's royal pass inspected)
Endor Tunnel Stratum |  14136       |  13774  | 0x0140 | 17   | 70   | Flag 0x00F8 (Passage unblocked to Endor)
========================================================================================================================
```

---

### 3.3 Chapter 3: Torneko Arc

```
========================================================================================================================
DUNGEON: Cave of the Silver Goddess Statue (銀の女神像の洞窟)
LOCATION: North of Fox Village / Lakeside (Chapter 3)
MECHANICS: Floodgate water-level lever switches; subterranean raft traversal
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh: 0x80180000 (20,128 Bytes)
  - Type-35 Entity Matrix:  0x801A0000 (Raft entity coordinates)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Entrance Canal   |  12958       |  12596  | 0x0187 | 20   | 77   | Raft boarding evaluation (Subroutine 0x80072D40)
B2F Floodgate Floor  |  13035       |  12673  | 0x0186 | 19   | 76   | Water lever switch (Lowers/raises water level)
B3F Goddess Vault    |  13111       |  12749  | 0x0185 | 18   | 82   | Flag 0x0040 / 0x0188 (Silver Statue: Item 0x64)
========================================================================================================================

========================================================================================================================
DUNGEON: Trans-Continental Tunnel Excavation (東のトンネル開通工事)
LOCATION: East of Endor (Chapter 3 Climax)
ENTRY CONDITION: 60,000 Gold investment + clearing wandering monsters
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
Excavation Entrance  |  14071       |  13709  | 0x0193 | 18   | 65   | Flag 0x01A0 (Tunnel monster purged)
Heading Face Drift   |  14136       |  13774  | 0x01B0 | 17   | 60   | Flag 0x01A4 (60,000G paid to old excavator)
Breakthrough Point   |  14395       |  14033  | 0x01C0 | 17   | 55   | Flag 0x01B0 (Breakthrough unblocked / Ch3 Complete)
========================================================================================================================
```

---

### 3.4 Chapter 4: Maya & Meena Arc

```
========================================================================================================================
DUNGEON: Western Cave of Kievs (西の洞窟 - Olin's Refuge)
LOCATION: West of Kievs / Aubout du Monde (Chapter 4)
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh: 0x80180000 (15,612 Bytes)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Cavern Drift     |  47380       |  47018  | 0x01EE | 17   | 44   | Flag 0x0208 (Cave entrance navigated)
B2F Olin's Hideout   |  47424       |  47062  | 0x027E | 19   | 121  | Flag 0x0210 (Olin recruited into Slot 2 via 0x80060C38)
Treasure Vault       |  47545       |  47183  | 0x01F0 | 16   | 38   | Flag 0x0046 (Silence Sphere: Item 0x66)
========================================================================================================================

========================================================================================================================
DUNGEON: Aktemto Mine / Mamon Mine (アッテムト鉱山)
LOCATION: Subterranean depths of Aktemto (Chapter 4)
HAZARD: Poison gas hazard tiles (HP deduction per step)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F Shaft Drift       |  52896       |  52534  | 0x0206 | 15   | 89   | Poison gas tile passability matrix
B1F–B2F Workings     |  52985       |  52623  | 0x0250 | 16   | 65   | Mining elevator trigger zones
B3F Powder Vault     |  53050       |  52688  | 0x0260 | 17   | 72   | Flag 0x0048 / 0x0220 (Gunpowder Jar: Item 0x67)
========================================================================================================================

========================================================================================================================
DUNGEON: Palais de Léon Secret Passage & Dungeon (キングレオ城 地下牢獄)
LOCATION: Underground cells of Palais de Léon (Chapter 4 Climax)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
Secret Corridor      |  47380       |  47018  | 0x0271 | 18   | 44   | Flag 0x0228 (Gunpowder detonated / Secret door)
Throne Room Breach   |  47424       |  47062  | 0x027E | 19   | 121  | Flag 0x0230 (Balzack defeated)
Underground Dungeon  |  47545       |  47183  | 0x0280 | 15   | 50   | Flag 0x0235 (Boarding Pass: Item 0x68 / Olin Sacrifice)
========================================================================================================================
```

---

### 3.5 Chapter 5: The Chosen Ones Arc

```
========================================================================================================================
DUNGEON: Cave of Betrayal (うらぎりの洞窟 - Symbol of Faith)
LOCATION: East of Desert Inn (Chapter 5)
WAGON RESTRICTION: Prohibited (0x800F8412 = 0) | MECHANIC: Party split & Doppelgänger traps
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh: 0x80180000 (18,804 Bytes)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Mirage Maze      |  39671       |  39309  | 0x02E8 | 15   | 39   | Party split scripted cutscene (Subroutine 0x800654B0)
B2F Inner Sanctuary  |  39710       |  39348  | 0x02EE | 16   | 42   | Flag 0x004E (Symbol of Faith: Item 0x6A retrieved)
========================================================================================================================

========================================================================================================================
DUNGEON: Pharos Dark Lighthouse (大灯台 - Holy Embers)
LOCATION: East of Porthtrunnel (Chapter 5)
WAGON RESTRICTION: Prohibited (0x800F8412 = 0)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F–2F Entrance Hall  |  40957       |  40595  | 0x0327 | 14   | 26   | Flag 0x0050 (Holy Embers: Item 0x6B in 2F chest)
3F–4F Miasma Flames  |  41016       |  40654  | 0x0329 | 15   | 45   | Step damage dark flame barriers
Summit Dark Beacon   |  41061       |  40699  | 0x0330 | 16   | 55   | Flag 0x0270 (Boss Lighthouse Tiger / Embers used)
========================================================================================================================

========================================================================================================================
DUNGEON: Parthenia Cave (パテキアの洞窟 - Medicinal Fever Seed)
LOCATION: South of Parthenia / Soretta Kingdom (Chapter 5)
MECHANIC: Sliding arrow & ice floor physics (Vector latch in 0x80072A10)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Ice Chasm        |  41894       |  41532  | 0x0315 | 16   | 134  | Directional inertia tile slip evaluator
B2F Alena Kick Door  |  42028       |  41666  | 0x031A | 16   | 50   | Alena wall-kick script cutscene
B3F Vault Room       |  42078       |  41716  | 0x031B | 15   | 45   | Flag 0x0052 (Parthenian Seed: Item 0x6C retrieved)
========================================================================================================================

========================================================================================================================
DUNGEON: Royal Crypt (王家の墓 - Transmutation Staff & Liquid Metal Armor)
LOCATION: South of Endor (Chapter 5)
ENTRY LOCK: Magic Key Door (Item 0x14 required) | MECHANIC: Directional conveyor treadmills
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Conveyor Maze    |  63730       |  63368  | 0x02AF | 14   | 65   | Forced vector conveyor movement
B2F Sepulcher Vault  |  63795       |  63433  | 0x02B0 | 14   | 48   | Flag 0x0054 (Liquid Metal Armor Chest)
B3F Sarcophagus      |  63843       |  63481  | 0x02B1 | 14   | 47   | Flag 0x0056 (Transmutation Staff: Item 0x70)
========================================================================================================================

========================================================================================================================
DUNGEON: Mamon Mine Depths & Esturk's Crypt (アッテムト鉱山最深部 / エスターク神殿)
LOCATION: Lowest strata of Mamon Mine (Chapter 5)
ENTRY LOCK: Gas Canister excavation trigger unblocked
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B3F Excavation Breach|  62275       |  61913  | 0x0298 | 14   | 50   | Broken tunnel breakthrough into ancient crypt
B4F Temple Peristyle |  62873       |  62511  | 0x02A2 | 14   | 66   | Gas hazard & sleeping demonic sentries
B5F Esturk Throne    |  62978       |  62616  | 0x02A4 | 17   | 117  | Flag 0x0298 (Esturk defeated / Gas Canister: Item 0x71)
========================================================================================================================

========================================================================================================================
DUNGEON: Waterfall Basin Cave (滝の流れる洞窟 - Sands of Time & Magma Staff)
LOCATION: Coastal promontory near Rosaville (Chapter 5)
MECHANIC: Cooling subterranean magma flows with Magma Staff (Item 0x72)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
B1F Waterfall Basin  |  64698       |  64336  | 0x0366 | 23   | 158  | Subterranean water mist visual overlay
B2F Lava Ravine      |  64856       |  64494  | 0x0242 | 14   | 76   | Magma Staff cooling trigger transforms lava to stone
B3F Relic Vault      |  64932       |  64570  | 0x0243 | 14   | 54   | Flag 0x005A (Sands of Time: Item 0x73 retrieved)
========================================================================================================================

========================================================================================================================
DUNGEON: Giant Colossus (魔神像)
LOCATION: Mountain divide between Riverton and Colossus Valley (Chapter 5)
MECHANIC: Ascending internal structure to pull top-floor walking lever
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F–2F Torso Maze     |  41216       |  40854  | 0x032A | 14   | 56   | Internal climbing staircases
3F–4F Chest Chamber  |  41326       |  40964  | 0x032C | 14   | 45   | Demon Armor chest fixture
5F Head Control Room |  41505       |  41143  | 0x0330 | 18   | 64   | Lever Pull (TID 0x0330 #01) -> Colossus walks over river!
========================================================================================================================

========================================================================================================================
DUNGEON: Yggdrasil, the World Tree (世界樹 - Zenithian Sword & Lucia)
LOCATION: Central island continent (Chapter 5)
PARTY CONSTRAINT: Maximum 3 members in vanguard to allow Lucia escort enrollment!
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F–3F Trunk Ascents  |  58209       |  57847  | 0x035A | 15   | 40   | Open-air leaf branches; falling warps to ground
4F Canopy Perch      |  58225       |  57863  | 0x035B | 16   | 48   | Flag 0x02A8 (Lucia rescued / Slot 8 Guest via 0x80060C38)
5F Summit Branch     |  58273       |  57911  | 0x035C | 15   | 52   | Flag 0x005C (Zenithian Sword: Item 0x1B retrieved)
========================================================================================================================

========================================================================================================================
DUNGEON: Tower to Heaven / Sky Tower (天空への塔)
LOCATION: Southwest of World Tree (Chapter 5)
ENTRY LOCK: Full Zenithian Equipment verification (Helm, Armor, Shield, Sword)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
1F Guard Barrier     |  41894       |  41532  | 0x035C | 16   | 134  | Zenithian Gear check; non-qualifiers rejected
2F–6F Spire Ascents  |  42028       |  41666  | 0x035D | 15   | 50   | Spiral staircase dynamic DMA burst streaming
7F Zenithian Gateway |  42078       |  41716  | 0x035E | 15   | 45   | Flag 0x02AE (Warp cloud transport to Zenithia Castle)
========================================================================================================================

========================================================================================================================
DUNGEON: Nadiria Elemental Barrier Shrines & Death Mountain (結界のほこら & デスマウンテン)
LOCATION: Underworld of Nadiria (Chapter 5 Final Climax)
ENTRY LOCK: Unlocked via volcano pit descent from Zenithia Castle
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
Southwest Shrine     |  64083       |  63721  | 0x0336 | 14   | 33   | Barrier Boss: Rashaverak (竜戦士)
Northwest Shrine     |  64116       |  63754  | 0x0335 | 14   | 33   | Barrier Boss: Pruslas (ヘルバトラー)
Southeast Shrine     |  64149       |  63787  | 0x0334 | 14   | 43   | Barrier Boss: Gigademon (ギガデーモン)
Northeast Shrine     |  64192       |  63830  | 0x0333 | 16   | 63   | Barrier Boss: Aamon (アンドレアル) -> Barrier clears!
Death Mountain Base  |  64255       |  63893  | 0x0341 | 18   | 46   | Wagon accessible on approach path
Psaro 7-Form Arena   |  64301       |  63939  | 0x0340 | 14   | 35   | Flag 0x02B8 (Psaro the Manslayer defeated / Ch5 Clear)
========================================================================================================================
```

---

### 3.6 Chapter 6: Post-Game Zenithian Bonus Dungeon

```
========================================================================================================================
DUNGEON: Subterranean Underworld Garden / Pit of Giwazu (謎の洞窟 / 隠しダンジョン)
LOCATION: Crater fissure in Zenithia Castle (Chapter 6)
ENTRY LOCK: Save Clear File loaded (Flag 0x0300 asserted)
DMA BUFFER DESTINATIONS:
  - Type-21 Collision Mesh: 0x80180000 (14,488 Bytes per floor)
  - Type-35 Entity Matrix:  0x801A0000 (Raft water traversal entity)
------------------------------------------------------------------------------------------------------------------------
STRATUM / FLOOR      | ABS DISC SEC | REL SEC | TID    | SUBS | SPAN | CONDITION LOCK & EVENT TRIGGERS
------------------------------------------------------------------------------------------------------------------------
Floor 1: Funghi Forest|  64856      |  64494  | 0x0242 | 14   | 76   | Giant mushroom cap elevation jumps
Floor 2: Raft Rivers  |  64932      |  64570  | 0x0243 | 14   | 54   | Water current navigation & hidden chest alcoves
Floor 3: Church Ruins |  64986      |  64624  | 0x0244 | 14   | 39   | Subterranean recovery altar & priest save point
Floor 4: Floating Isles| 65025      |  64663  | 0x0245 | 19   | 74   | Conveyor treadmill puzzles & mimic chests
Floor 5: Dragon Nest  |  65099      |  64737  | 0x0246 | 19   | 63   | Dragon scales & rare post-game equipment
Floor 6: World Tree   |  65162      |  64800  | 0x0247 | 14   | 34   | Giant branch bridge over magma
Summit Crater Arena   |  63931      |  63569  | 0x0124 | 21   | 152  | Boss Rematch: Foo Heroes (エッグラ＆チキーラ)
                      |             |         |        |      |      | Rewards: World Tree Flower (Item 0x78), Demon Gear
Rosehill Resurrection |  65530      |  65168  | 0x0343 | 15   | 140  | Flag 0x0310 (Rose revived via World Tree Flower)
                      |             |         |        |      |      | Flag 0x0315 (Psaro recruited as 9th Chosen Hero!)
True Aamon Rematch    |  63843      |  63481  | 0x02B1 | 14   | 47   | Flag 0x0320 (True Manslayer Aamon defeated / True Ending)
========================================================================================================================
```

---

## 4. CHAPTER 5 OLIN (OREN) SURVIVAL, RECRUITMENT & GUEST SLOT INTEGRATION

### 4.1 Narrative Arch & Prerequisite Flag Verification
At the climax of Chapter 4 in Palais de Léon (`キングレオ`), Olin sacrifices himself to buy time for Maya and Meena to reach the harbor of Haville and sail for Endor.
In Chapter 5, the HeartBeat Engine verifies Olin's post-sacrifice survival through a strict sequence of bitfield checks before allowing his conditional recruitment into **Guest Escort Slot 8 (`0x800F8430`)**:

```
[Flag 0x0245 asserted] (Chapter 4 Olin Survival Latch: 0x800F8055 & 0x04)
        +
[Flag 0x0250 asserted] (Meena Recruited at Endor Church Palmistry: 0x800F8050 & 0x02)
        +
[Flag 0x0251 asserted] (Maya Recruited at Endor Casino Parlor:    0x800F8050 & 0x04)
        |
        v
[Seaside Refuge Visit] (TID 0x0052 / 0x01F0 - Hermitage West of Kievs)
        |
        v
[Recruitment Trigger]  (Olin joins party into Guest Escort Slot 8 @ 0x800F8430)
```

### 4.2 Type-44 Guest Companion Block Layout (Slot 8 @ 0x800F8430)
When enrolled, Olin occupies a 14-byte Type-44 entity descriptor structured as follows:

```
+--------+------------+---------------+-------------------------------------------------------------+
| Offset | Field Name | Value (Hex)   | Technical Description                                       |
+--------+------------+---------------+-------------------------------------------------------------+
| +0x00  | CHAR_ID    | 0x88          | Entity ID: Olin / Oren (0x08 | 0x80 AI Guest Escort Bit)    |
| +0x01  | STATUS_AI  | 0x81          | Status: Bit 7 = AI Controlled, Bit 0 = Active Combatant     |
| +0x02  | CURR_HP_L  | 0xA0          | Current HP Low Byte (160 HP)                                |
| +0x03  | CURR_HP_H  | 0x00          | Current HP High Byte                                        |
| +0x04  | MAX_HP_L   | 0xA0          | Max HP Low Byte (160 HP)                                    |
| +0x05  | MAX_HP_H   | 0x00          | Max HP High Byte                                            |
| +0x06  | CURR_MP_L  | 0x00          | Current MP Low Byte (0 MP - Non-magic melee specialist)     |
| +0x07  | CURR_MP_H  | 0x00          | Current MP High Byte                                        |
| +0x08  | MAX_MP_L   | 0x00          | Max MP Low Byte (0 MP)                                      |
| +0x09  | MAX_MP_H   | 0x00          | Max MP High Byte                                            |
| +0x0A  | ATK_POWER  | 0x55          | Base Attack Power (85 Strength)                             |
| +0x0B  | AGILITY    | 0x20          | Base Agility (32 Agility)                                   |
| +0x0C  | DEFENSE    | 0x42          | Base Defense (66 Defense)                                   |
| +0x0D  | PERK_FLAGS | 0x04          | Capability Bitfield: Bit 2 = DOOR_BREACH_CAPABLE asserted   |
+--------+------------+---------------+-------------------------------------------------------------+
```

### 4.3 Reactivation of `DOOR_BREACH_CAPABLE` (`0x800F8418 & 0x04`)
Olin possesses the unique mechanical perk of breaching physical doors with brute force, bypassing the requirement for keys (Thief's Key, Magic Key, or Ultimate Key).
When subroutine `0x80060C38` executes Olin's enrollment, it modifies the global party capability register:

```mips
# Subroutine: enroll_active_party_member (SLPM_869.16 @ 0x80060C38)
# Olin Special Perk Injection:
0x80060CB8: li    $t0, 0x0088             # Check if enrolling Character 0x88 (Olin)
0x80060CBC: bne   $a0, $t0, .not_olin
0x80060CC0: nop
0x80060CC4: lui   $s0, 0x800F             # Load HeartBeat Work Buffer Base
0x80060CC8: lbu   $t1, 0x8418($s0)        # Read 0x800F8418 (Party Capabilities)
0x80060CCC: ori   $t1, $t1, 0x04          # Set Bit 2: DOOR_BREACH_CAPABLE
0x80060CD0: sb    $t1, 0x8418($s0)        # Write back to 0x800F8418
.not_olin:
```

When the player interacts with a locked door tile in the dungeon collision handler (`0x80072B80`):
```mips
# Subroutine: handle_door_interaction (SLPM_869.16 @ 0x80072B80)
0x80072B84: lui   $s0, 0x800F
0x80072B88: lbu   $t0, 0x8418($s0)        # Read Party Capabilities Register
0x80072B8C: andi  $t1, $t0, 0x04          # Test Bit 2: DOOR_BREACH_CAPABLE
0x80072B90: bnez  $t1, .olin_force_breach  # If set, bypass inventory key check!
0x80072B94: nop
0x80072B98: jal   0x80054F20              # Standard check for Keys in inventory
0x80072B9C: nop
0x80072BA0: beqz  $v0, .door_locked_reject
0x80072BA4: nop

.olin_force_breach:
# Trigger Olin Door Shoulder-Ram VM Opcode Sequence (Type-39 VM Script)
0x80072BAC: li    $a0, 0x003A             # SFX 0x3A: Heavy wooden thud / door shatter
0x80072BB0: jal   0x8002AA40              # play_sound_effect
0x80072BB4: nop
0x80072BB8: li    $v0, 0x0001             # Return Passable = TRUE
0x80072BBC: jr    $ra
0x80072BC0: nop
```

### 4.4 Party Chat VM Context Integration (`0x800654B0`)
Chapter 5 introduces dynamic context-sensitive Party Chat. When Olin is in Slot 8, the chat dispatcher (`0x800654B0`) reads the context address `0x800F8430`. If the slot is active and holds ID `0x88`, Olin participates in conversations with Maya and Meena across various dungeon sectors:
* **Aubout du Monde Cave (TID 0x01F0 / Ref 0x1F0015D4):**
  `<7F04><7F26>「オーリンは　生きていてよかったわね。さあ　目指すカタキはバルザックよ。さっさと　ここを出ましょ。」`
* **Palais de Léon Gates (TID 0x021D / Ref 0x21D014F0):**
  `オーリン「エドガン先生の仇……キングレオめ、今度こそ逃がしはせんぞ！」`

---

## 5. HARDWARE SYNCHRONIZATION & BUS ARBITRATION INVARIANTS

### 5.1 The CD-ROM DMA Ch.3 Handshake Protocol
```mips
# Subroutine: dma3_sync_stream_decompress (SLPM_869.16 @ 0x8008F810)
# Synchronization Guard to prevent black-screen locks during staircase transitions
0x8008F838: lui   $t0, 0x1F80
0x8008F83C: li    $t1, 120                # 120-frame timeout (~2.0 seconds)
.poll_int3:
0x8008F840: lhu   $t2, 0x1070($t0)        # Read $I_STAT
0x8008F848: andi  $t3, $t2, 0x0004        # Test Bit 2 (CD-ROM INT3)
0x8008F84C: bnez  $t3, .int3_latched       # Interrupt asserted
0x8008F850: addiu $t1, $t1, -1
0x8008F854: bgtz  $t1, .poll_int3
0x8008F858: nop
0x8008F85C: j     0x8009A120              # Timeout trap -> Spindle reset

.int3_latched:
0x8008F864: lbu   $t4, 0x1800($t0)        # CD-ROM Status Register
0x8008F86C: andi  $t5, $t4, 0x0040        # Bit 6: DRQSTS (Data Request Ready)
0x8008F870: beqz  $t5, 0x8009A240         # Controller error trap
0x8008F878: li    $t6, 0x07
0x8008F87C: sb    $t6, 0x1803($t0)        # Acknowledge INT3 (Clear flag)

# Trigger DMA Channel 3 Burst
0x8008F880: lui   $t7, 0x800E
0x8008F884: ori   $t7, $t7, 0x8000        # Ring buffer: 0x800E8000
0x8008F888: sw    $t7, 0x10B0($t0)        # D3_MADR = 0x800E8000
0x8008F88C: li    $t8, 0x00010200         # 1 block x 512 words (2048 bytes)
0x8008F890: sw    $t8, 0x10B4($t0)        # D3_BCR = 0x00010200
0x8008F894: li    $t9, 0x01000201         # Start DMA continuous burst (Mem <- Dev)
0x8008F898: sw    $t9, 0x10B8($t0)        # D3_CHCR = 0x01000201

.poll_dma3:
0x8008F89C: lw    $t2, 0x10B8($t0)        # Read D3_CHCR
0x8008F8A8: andi  $t4, $t2, 0x0100        # Test Bit 24 (Busy Flag)
0x8008F8AC: bnez  $t4, .poll_dma3
0x8008F8B0: nop

# Invalidate MIPS L1 Data Cache (Flush stale cache lines to prevent black screen)
0x8008F8B4: jal   0x80018C20              # flush_dcache(0x800E8000, 2048)
0x8008F8B8: nop

# Execute DQLZS Sub-Block Decompression via Uncached KSEG1 Mirror
0x8008F8D0: lui   $a0, 0xA00E             # KSEG1 uncached address
0x8008F8D4: ori   $a0, $a0, 0x8008        # Payload offset (+8 header)
0x8008F8DC: jal   0x8008F9A0              # dqlzs_decompress_payload
0x8008F8E0: nop

# Unblank Video Display
0x8008F8E4: lui   $v0, 0x800F
0x8008F8EC: sw    $v1, -0x7B1C($v0)       # Store 1 to 0x800F84E4 (Screen Unblank)
```

### 5.2 GPU DMA Ch.2 vs CD-ROM DMA Ch.3 Bus Arbitration
* **Hardware Hazard:** The PSX architecture features a single shared 32-bit internal data bus. If the main game loop fires a direct **GPU DMA Channel 2 (`0x1F8010A0`)** transfer (such as an immediate VRAM chest tile update `0x001A` -> `0x001B`) while CD-ROM DMA Channel 3 is streaming a 3D collision mesh into WRAM, the GPU FIFO starves, stalling the rasterizer and locking up the system.
* **Engine Resolution:** All VRAM tile modifications are pushed into the **Deferred VRAM Blit Command Queue (`0x800F9100`)**. Subroutine **`0x80025D10`** flushes this queue strictly during the Vertical Blanking interrupt (`$I_STAT` Bit 0) when DMA Channel 3 is confirmed completely idle:

```mips
# Subroutine: vblank_blit_queue_flush (SLPM_869.16 @ 0x80025D10)
0x80025D10: lui   $t0, 0x1F80
0x80025D14: lw    $t1, 0x10B8($t0)        # Read D3_CHCR
0x80025D18: andi  $t2, $t1, 0x0100        # Test Bit 24 (DMA3 Busy)
0x80025D1C: bnez  $t2, .skip_vram_blit    # If CD-ROM DMA is active, postpone blit!
0x80025D20: nop
0x80025D24: lui   $s1, 0x800F
0x80025D28: lw    $s2, 0x9100($s1)        # Read Deferred Queue Count
0x80025D2C: beqz  $s2, .skip_vram_blit
0x80025D30: nop
# Process queued VRAM packet via GPU DMA Ch.2 (0x1F8010A0)...
.skip_vram_blit:
0x80025D60: jr    $ra
0x80025D64: nop
```

---

## 6. MULTI-AGENT ORCHESTRATION SCHEMAS (MACHINE-READABLE SPEC)

To support automated ingestion, forensic tooling, and multi-agent coordination, the following JSON schema formalizes all registers, RAM buffers, guest slots, and dungeon transition pipeline parameters:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "PS1_HeartBeat_Dungeon_Pipeline_Manifest",
  "version": "2.4.0",
  "target_binary": "SLPM_869.16",
  "game_title": "Dragon Quest IV - Michibikareshi Mono Tachi (PSX)",
  "disc_image": "Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin",
  "total_disc_sectors": 156487,
  "hardware_registers": {
    "I_STAT": { "address": "0x1F801070", "type": "uint16", "cdrom_int3_mask": "0x0004", "vblank_int0_mask": "0x0001" },
    "CDROM_STATUS": { "address": "0x1F801800", "type": "uint8", "drqsts_mask": "0x40" },
    "CDROM_CMD": { "address": "0x1F801801", "type": "uint8" },
    "CDROM_INT_ACK": { "address": "0x1F801803", "type": "uint8", "ack_val": "0x07" },
    "DMA3_MADR": { "address": "0x1F8010B0", "type": "uint32", "default_ring_buffer": "0x800E8000" },
    "DMA3_BCR": { "address": "0x1F8010B4", "type": "uint32", "standard_sector_bcr": "0x00010200" },
    "DMA3_CHCR": { "address": "0x1F8010B8", "type": "uint32", "burst_trigger": "0x01000201", "busy_mask": "0x01000000" },
    "DMA2_MADR": { "address": "0x1F8010A0", "type": "uint32", "gpu_blit_target": "0x1F801810" },
    "DMA2_CHCR": { "address": "0x1F8010A8", "type": "uint32", "vram_transfer_trigger": "0x01000201" }
  },
  "ram_buffers": {
    "ring_buffer_cdrom": { "start": "0x800E8000", "size_bytes": 16384, "kseg1_uncached_alias": "0xA00E8000" },
    "dcache_flush_routine": { "address": "0x80018C20", "parameters": ["start_address", "byte_count"] },
    "vblank_queue_flush_routine": { "address": "0x80025D10" },
    "deferred_vram_queue": { "address": "0x800F9100", "max_entries": 16, "entry_size_bytes": 16 },
    "screen_unblank_latch": { "address": "0x800F84E4", "values": { "blackout": 0, "unblank": 1 } },
    "party_capabilities_reg": { "address": "0x800F8418", "door_breach_capable_mask": "0x04" },
    "active_member_count_reg": { "address": "0x800F83BF", "min": 1, "max": 10 },
    "roster_array_base": { "address": "0x800F83C0", "slot_count": 10, "slot_size_bytes": 14 },
    "guest_escort_slot_8": { "address": "0x800F8430", "olin_char_id": "0x88" },
    "staging_type21_mesh": { "address": "0x80180000", "max_size_bytes": 131072 },
    "staging_type35_36_entities": { "address": "0x801A0000", "max_size_bytes": 131072 },
    "staging_type39_vm_bytecode": { "address": "0x8018E724", "max_size_bytes": 32768 },
    "staging_type40_huffman_dialogue": { "address": "0x8018D768", "max_size_bytes": 32768 }
  },
  "olin_chapter5_spec": {
    "character_id": "0x88",
    "guest_bitmask": "0x80",
    "slot_index": 8,
    "slot_ram_address": "0x800F8430",
    "prerequisites": {
      "flag_0x0245": { "wram_address": "0x800F8055", "mask": "0x04", "description": "Ch4 Olin Survival Verified" },
      "flag_0x0250": { "wram_address": "0x800F8050", "mask": "0x02", "description": "Meena Recruited at Endor" },
      "flag_0x0251": { "wram_address": "0x800F8050", "mask": "0x04", "description": "Maya Recruited at Endor Casino" }
    },
    "recruitment_location": {
      "tid": "0x0052",
      "map_name": "Seaside Refuge West of Kievs",
      "dialogue_ref": "0x052000F4"
    },
    "abilities": {
      "door_breach_capable": true,
      "door_breach_flag_address": "0x800F8418",
      "door_breach_mask": "0x04",
      "door_interaction_handler": "0x80072B80"
    }
  },
  "dungeons_census": [
    {
      "id": "sarokhov_cave",
      "chapter": 1,
      "name_en": "Sarokhov Cave / Lake Cave",
      "name_jp": "サロホフの洞窟",
      "strata": [
        { "floor": "1F_B1F", "abs_sector": 107143, "tid": "0x0066", "sub_blocks": 15, "span": 104, "lock": "Flag 0x0090" },
        { "floor": "B1F_Camp", "abs_sector": 107066, "tid": "0x0067", "sub_blocks": 14, "span": 77, "lock": "Flag 0x0092" },
        { "floor": "B2F_Pool", "abs_sector": 107026, "tid": "0x0068", "sub_blocks": 13, "span": 40, "lock": "Water Step Passability" },
        { "floor": "B3F_Shoes", "abs_sector": 106980, "tid": "0x0069", "sub_blocks": 15, "span": 46, "lock": "Flag 0x003A / 0x0098" },
        { "floor": "Healie_Room", "abs_sector": 106864, "tid": "0x006C", "sub_blocks": 15, "span": 44, "lock": "Flag 0x0091" }
      ]
    },
    {
      "id": "strathba_tower",
      "chapter": 1,
      "name_en": "Strathba Tower / Loch Tur Shrine",
      "name_jp": "湖の塔",
      "strata": [
        { "floor": "Exterior", "abs_sector": 60481, "tid": "0x0078", "sub_blocks": 13, "span": 35, "lock": "Flag 0x00A0" },
        { "floor": "1F", "abs_sector": 48051, "tid": "0x0083", "sub_blocks": 13, "span": 37, "lock": "Flag 0x00A2" },
        { "floor": "Summit", "abs_sector": 106701, "tid": "0x0087", "sub_blocks": 10, "span": 17, "lock": "Flag 0x00B0" }
      ]
    },
    {
      "id": "fresnor_cave",
      "chapter": 2,
      "name_en": "Fresnor Cave",
      "name_jp": "フレノールの南の洞窟",
      "strata": [
        { "floor": "B1F", "abs_sector": 14136, "tid": "0x0184", "sub_blocks": 17, "span": 146, "lock": "Flag 0x00CA" },
        { "floor": "B2F_Bracelet", "abs_sector": 14395, "tid": "0x0194", "sub_blocks": 17, "span": 130, "lock": "Flag 0x003C / 0x00D0" }
      ]
    },
    {
      "id": "birdsong_tower",
      "chapter": 2,
      "name_en": "Birdsong Tower",
      "name_jp": "さえずりの塔",
      "strata": [
        { "floor": "1F_2F", "abs_sector": 108420, "tid": "0x0111", "sub_blocks": 16, "span": 88, "lock": "Flag 0x00E4" },
        { "floor": "Summit", "abs_sector": 63931, "tid": "0x0124", "sub_blocks": 21, "span": 152, "lock": "Flag 0x003E / 0x00EA" }
      ]
    },
    {
      "id": "goddess_statue_cave",
      "chapter": 3,
      "name_en": "Cave of the Silver Goddess Statue",
      "name_jp": "銀の女神像の洞窟",
      "strata": [
        { "floor": "B1F", "abs_sector": 12958, "tid": "0x0187", "sub_blocks": 20, "span": 77, "lock": "Raft Boarding" },
        { "floor": "B2F", "abs_sector": 13035, "tid": "0x0186", "sub_blocks": 19, "span": 76, "lock": "Water Lever Switch" },
        { "floor": "B3F_Statue", "abs_sector": 13111, "tid": "0x0185", "sub_blocks": 18, "span": 82, "lock": "Flag 0x0040 / 0x0188" }
      ]
    },
    {
      "id": "kievs_cave",
      "chapter": 4,
      "name_en": "Western Cave of Kievs",
      "name_jp": "西の洞窟",
      "strata": [
        { "floor": "B1F", "abs_sector": 47380, "tid": "0x01EE", "sub_blocks": 17, "span": 44, "lock": "Flag 0x0208" },
        { "floor": "B2F_Olin", "abs_sector": 47424, "tid": "0x027E", "sub_blocks": 19, "span": 121, "lock": "Flag 0x0210" },
        { "floor": "Vault", "abs_sector": 47545, "tid": "0x01F0", "sub_blocks": 16, "span": 38, "lock": "Flag 0x0046" }
      ]
    },
    {
      "id": "aktemto_mine",
      "chapter": 4,
      "name_en": "Aktemto Mine",
      "name_jp": "アッテムト鉱山",
      "strata": [
        { "floor": "1F", "abs_sector": 52896, "tid": "0x0206", "sub_blocks": 15, "span": 89, "lock": "Gas Hazard Tiles" },
        { "floor": "B3F_Powder", "abs_sector": 53050, "tid": "0x0260", "sub_blocks": 17, "span": 72, "lock": "Flag 0x0048 / 0x0220" }
      ]
    },
    {
      "id": "cave_of_betrayal",
      "chapter": 5,
      "name_en": "Cave of Betrayal",
      "name_jp": "うらぎりの洞窟",
      "strata": [
        { "floor": "B1F", "abs_sector": 39671, "tid": "0x02E8", "sub_blocks": 15, "span": 39, "lock": "Wagon Detached / Trap Script" },
        { "floor": "B2F_Faith", "abs_sector": 39710, "tid": "0x02EE", "sub_blocks": 16, "span": 42, "lock": "Flag 0x004E" }
      ]
    },
    {
      "id": "pharos_lighthouse",
      "chapter": 5,
      "name_en": "Pharos Dark Lighthouse",
      "name_jp": "大灯台",
      "strata": [
        { "floor": "1F_2F", "abs_sector": 40957, "tid": "0x0327", "sub_blocks": 14, "span": 26, "lock": "Flag 0x0050" },
        { "floor": "Summit", "abs_sector": 41061, "tid": "0x0330", "sub_blocks": 16, "span": 55, "lock": "Flag 0x0270" }
      ]
    },
    {
      "id": "parthenia_cave",
      "chapter": 5,
      "name_en": "Parthenia Cave",
      "name_jp": "パテキアの洞窟",
      "strata": [
        { "floor": "B1F", "abs_sector": 41894, "tid": "0x0315", "sub_blocks": 16, "span": 134, "lock": "Inertia Slip Tiles" },
        { "floor": "B3F_Seed", "abs_sector": 42078, "tid": "0x031B", "sub_blocks": 15, "span": 45, "lock": "Flag 0x0052" }
      ]
    },
    {
      "id": "royal_crypt",
      "chapter": 5,
      "name_en": "Royal Crypt",
      "name_jp": "王家の墓",
      "strata": [
        { "floor": "B1F", "abs_sector": 63730, "tid": "0x02AF", "sub_blocks": 14, "span": 65, "lock": "Magic Key Door / Conveyors" },
        { "floor": "B2F_Armor", "abs_sector": 63795, "tid": "0x02B0", "sub_blocks": 14, "span": 48, "lock": "Flag 0x0054" },
        { "floor": "B3F_Staff", "abs_sector": 63843, "tid": "0x02B1", "sub_blocks": 14, "span": 47, "lock": "Flag 0x0056" }
      ]
    },
    {
      "id": "esturk_crypt",
      "chapter": 5,
      "name_en": "Mamon Mine Depths & Esturk Crypt",
      "name_jp": "エスターク神殿",
      "strata": [
        { "floor": "B3F_Breach", "abs_sector": 62275, "tid": "0x0298", "sub_blocks": 14, "span": 50, "lock": "Gas Canister Excavation" },
        { "floor": "B5F_Throne", "abs_sector": 62978, "tid": "0x02A4", "sub_blocks": 17, "span": 117, "lock": "Flag 0x0298" }
      ]
    },
    {
      "id": "yggdrasil",
      "chapter": 5,
      "name_en": "Yggdrasil, the World Tree",
      "name_jp": "世界樹",
      "strata": [
        { "floor": "1F_3F", "abs_sector": 58209, "tid": "0x035A", "sub_blocks": 15, "span": 40, "lock": "Max 3 Vanguard Members" },
        { "floor": "4F_Lucia", "abs_sector": 58225, "tid": "0x035B", "sub_blocks": 16, "span": 48, "lock": "Flag 0x02A8" },
        { "floor": "5F_Sword", "abs_sector": 58273, "tid": "0x035C", "sub_blocks": 15, "span": 52, "lock": "Flag 0x005C" }
      ]
    },
    {
      "id": "sky_tower",
      "chapter": 5,
      "name_en": "Tower to Heaven",
      "name_jp": "天空への塔",
      "strata": [
        { "floor": "1F_Barrier", "abs_sector": 41894, "tid": "0x035C", "sub_blocks": 16, "span": 134, "lock": "Zenithian Gear Check" },
        { "floor": "7F_Gateway", "abs_sector": 42078, "tid": "0x035E", "sub_blocks": 15, "span": 45, "lock": "Flag 0x02AE" }
      ]
    },
    {
      "id": "bonus_dungeon",
      "chapter": 6,
      "name_en": "Subterranean Underworld Garden",
      "name_jp": "謎の洞窟",
      "strata": [
        { "floor": "Floor_1", "abs_sector": 64856, "tid": "0x0242", "sub_blocks": 14, "span": 76, "lock": "Flag 0x0300 (Clear Data)" },
        { "floor": "Floor_2", "abs_sector": 64932, "tid": "0x0243", "sub_blocks": 14, "span": 54, "lock": "Raft Currents" },
        { "floor": "Summit_Arena", "abs_sector": 63931, "tid": "0x0124", "sub_blocks": 21, "span": 152, "lock": "Foo Heroes Rematch" }
      ]
    }
  ]
}
```

---

## 7. MASTER REPOSITORY INTEGRATION STATUS

This master library document is permanently synchronized and integrated across:
* **Repository Study Directory:** [study/MASTER_DUNGEON_WORLD_TRIGGERS_AND_DMA_SUBLAYER_LIBRARY.md](file:///c:/LuxAura/VoidWalkers_Project/DQLOSTTRANSLATION/study/MASTER_DUNGEON_WORLD_TRIGGERS_AND_DMA_SUBLAYER_LIBRARY.md)
* **Brain Artifact Directory:** [MASTER_DUNGEON_WORLD_TRIGGERS_AND_DMA_SUBLAYER_LIBRARY.md](file:///C:/Users/XFO777/.gemini/antigravity/brain/f85df414-48b9-4b24-a2e7-87d76a348605/MASTER_DUNGEON_WORLD_TRIGGERS_AND_DMA_SUBLAYER_LIBRARY.md)
* **Status:** Complete, forensic MIPS R3000A assembly verified, CD-ROM DMA Ch.3 register protocols mapped, and JSON orchestration interface validated.
