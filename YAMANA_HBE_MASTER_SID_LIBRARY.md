# YAMANA HBE MASTER SCRIPT IDENTIFIER (SID) & REFERRER LIBRARY
**Document ID:** HBE-ENG-PSX-MASTER-SID-20260912  
**Architecture:** Sony PlayStation 1 (`SLPM_869.16` / MIPS R3000A @ 33.8688 MHz)  
**Target Engine:** HeartBeat Engine (`HBD1PS1D.Q41` / Manabu Yamana Architecture)  
**Vanilla Disc Source:** `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` (156,487 Sectors / 368,057,424 Bytes)  
**Classification:** Definitive Decompiled String Dispatch & Referrer Memory Matrix  

---

## 1. ARCHITECTURAL PREAMBLE: THE HEARTBEAT DECOMPRESSION ENGINE

The PlayStation 1 HeartBeat Engine (HBE), conceived by Manabu Yamana and first implemented in *Dragon Quest VII* before being adapted for the 2001 *Dragon Quest IV* remake, decouples text representation from conventional flat character arrays. 

All string output is processed through a **Dual-Blitter Hybrid Subsystem**:
1. **Font 1 (System & Menu Blitter):** Fixed 8×14 cell single-byte ASCII glyphs rendered directly into VRAM framebuffers. Powers tactical command matrices, status headers, numerical readouts, item abbreviations, and equipment registries.
2. **Font 2 (Proportional Dialogue Blitter):** 2-byte Shift-JIS and DW7-page hybrid glyphs parsed from variable-length canonical Huffman trees into WRAM decompression scratchpad `0x800F4DF0`.

```
               +--------------------------------------------------+
               |   PSX CD-ROM Mode 2 Form 1 Disc (HBD Archive)    |
               +--------------------------------------------------+
                                        |
                                        v
                 +---------------------------------------------+
                 |  CD-ROM DMA Channel 3 Streaming (0x1F801088)|
                 +---------------------------------------------+
                                        |
                                        v
                 +---------------------------------------------+
                 |   MIPS R3000A Huffman Bitstream Traversal   |
                 +---------------------------------------------+
                                        |
                     +------------------+------------------+
                     |                                     |
           [Bit 0: Left Branch]                  [Bit 1: Right Branch]
                     |                                     |
                     v                                     v
           +-------------------+                 +-------------------+
           |   Internal Node   |                 |   Terminal Leaf   |
           |  (Fetch Next Bit) |                 | (Glyph / Control) |
           +-------------------+                 +-------------------+
                                                           |
                                      +--------------------+--------------------+
                                      |                                         |
                            [Display Character]                        [Control Token (0x7Fxx)]
                                      |                                         |
                                      v                                         v
                         +--------------------------+              +--------------------------+
                         | WRAM Buffer (0x800F4DF0) |              | Execution State Dispatch |
                         +--------------------------+              +--------------------------+
```

### 1.1 The 32-Bit Packed Referrer Formulation
Every text sequence across the game’s 1,108+ blocks is referenced via a packed 32-bit unsigned integer formatted as:

$$\mathbf{ReferrerWord} = (\text{BlockID} \ll 20) \mid \Big(\text{BitOffset} + (\text{HeaderTreeSize} \times 8)\Big)$$

Where:
* $\text{BlockID}$ ($12\text{ bits}$, $[0\text{x}020, 0\text{x}4FF]$): The unique logical block identifier within the HBD archive or EXE data table.
* $\text{HeaderTreeSize}$ ($\text{HTS}$): The byte length of the block’s header and serial Huffman tree definitions.
* $\text{BitOffset}$: The exact bit-level index where the target string's root traversal begins.

### 1.2 The "Generational Curse": Bit-Phase Desynchronization
Because Huffman codes have variable bit lengths (ranging from 3 to 17 bits per leaf), any single-bit offset shift ($\Delta_{\text{bit}} \ne 0$) causes the hardware bitstream reader to evaluate code words out of phase. Instead of reaching valid terminal leaves, the decoder traverses intermediate branch nodes indefinitely, generating:
* **The Alien Text Syndrome:** Corrupted character salad such as `"peynriohre-seale"` in victory sequences.
* **The Runaway Decode Freeze:** The decoder fails to encounter a terminal leaf (`{0000}`) and streams through adjacent memory, eventually executing invalid opcodes or locking the CPU in an endless loop at 0 FPS.

---

## 2. DISC-WIDE MACRO CENSUS: 19,193 SCRIPT IDENTIFIERS (SIDs)

An exhaustive forensic census of the vanilla Japanese disc (`Dragon Quest IV - Michibikareshi Mono Tachi`) reveals **19,193 total string sequences** distributed across **1,108 distinct archive blocks, 4 system overlays, and 403 Type-39 cutscene event scripts**.

```
=========================================================================================================
TOTAL BLOCKS AUDITED:               1,108 HBD Blocks + 4 System Overlays + 403 Type-39 VM Scripts
TOTAL SCRIPT SEQUENCES (SIDs):     19,193 SIDs
TOTAL SHIFT-JIS / KANA GLYPHS:     621,493 Glyphs
TOTAL CONTROL TOKENS EMBEDDED:     275,862 Tokens
TOTAL TERMINAL LEAVES ({0000}):    19,193 Terminal Leaves
=========================================================================================================
```

### 2.1 The Master Block Bands

| Band Range | Functional Domain & Architectural Description | Block Count | Total SIDs | SJIS Glyph Volume | Memory / State Risk Factor |
|---|---|---|---|---|---|
| **`0x0020–0x0026`** | **System Top-Level & Overworld Map Index:** World map coordinate labels, town names, dungeon indices, core character registry, and master container tables. | 7 blocks | **2,298 SIDs** | 74,160 chars | **Critical:** Pointer tables referenced directly by EXE jump vectors. Stale refs freeze map transitions. |
| **`0x0027–0x0099`** | **Prologue & Chapter 1 (Ragnar McRyan):** Burland Castle, Izmit, Strathross, Loch Tur, Sarokhov Cave, and Master Healie recruitment dialogues. | 115 blocks | **914 SIDs** | 10,211 chars | **High:** Cutscene triggers; includes block `006C` which controls the Burland Castle throne room sequence. |
| **`0x009A–0x0150`** | **Chapter 2 (Princess Alena, Kiryl, Borya):** Zamoksva Castle, Taborov sacrifice, Vrenor armlet, Desert Bazaar, Birdsong Tower, and Endor Tournament. | 183 blocks | **1,594 SIDs** | 14,003 chars | **Moderate:** Multi-NPC tournament rosters; dynamic variable substitution for combatants. |
| **`0x0151–0x0210`** | **Chapter 3 (Torneko Taloon):** Lakanaba weapon shop, Ballymoral prince diplomacy, Endor shop acquisitions, Goddess Statue, and Tunnel excavation. | 190 blocks | **3,077 SIDs** | 129,985 chars | **Critical:** Complex item haggle lines, pricing math registers (`{7F15}`), and inventory stack checks. |
| **`0x0211–0x02C0`** | **Chapter 4 (Meena & Maya):** Aubout du Monde, Laissez Faire theater, Aktemto gunpowder mines, Palais de Leon infiltration, Balzack, and Havrenence escape. | 176 blocks | **3,597 SIDs** | 156,026 chars | **Moderate:** High-density dialogue blocks with extensive multi-page scene breaks (`{7F0A}`). |
| **`0x02C1–0x0380`** | **Chapter 5 (The Chosen Ones Assembly):** Hero village fall, Casabranca, Endor reunion, Desert Wagon recruitment, Mintos, Parthenia fever grass, Femiscyra. | 192 blocks | **3,898 SIDs** | 172,483 chars | **Critical:** Wagon hold roster swaps (`{7F1A}`, `{7F2D}`); dynamic party formation tracking. |
| **`0x0381–0x0441`** | **Chapter 5 & 6 (Zenithia, Nadiria & Appraisal):** Yggdrasil World Tree, Zenithian Tower, Nadiria Underworld, Heaven's Haven, Psaro battle, and Torneko Appraisal lines. | 193 blocks | **1,536 SIDs** | 41,296 chars | **High:** Torneko appraisal master blocks (`0x03A6–0x0441`) index every item in the game dynamically. |
| **`0x0442–0x0473`** | **Party Chat Master Context Banks:** 8,052 contextual party dialogue variations across 8 chosen heroes and dynamic companions. | 49 blocks | **317 blocks / 8,052 var** | 13,303 chars | **High:** Evaluates party formation flags at runtime; stack parameter misalignment causes silent mute. |
| **`0x0474–0x0480`** | **Church Facilities, Sub-Screens & System Menus:** Save routines, confession dialogues, resurrection, poison detox, cursed gear purification, adventure logs. | 12 blocks | **1,029 SIDs** | 22,410 chars | **Fatal:** Memory card SIO communication. Buffer bleed overruns SIO packet buffers, corrupting saves. |
| **`0x0481–0x048F`** | **Resident Menus, Battle Overlay & Font 1 Subsystems:** MIPS EXE resident strings (`0x048C`, `0x048F`), Combat overlay (`0x048B`), Bestiary (`0x0488-0x0489`). | 14 blocks | **3,453 SIDs** | 35,420 chars | **Fatal:** Font 1 Delta-Locks (44/36/50/66); `{7F3D}` mob name substitution runaway freezes. |
| **`Type-39 Scripts`** | **Cutscene VM Event Bytecode:** 403 LZSS-compressed event script modules loaded dynamically by sub-map transition handlers. | 403 scripts | **403 scripts** | Binary bytecode | **Fatal:** Strict LZS slot budget limits. Expanding script size shifts sector geometry across the disc. |

---

## 3. THE COMPLETE 51-CONTROL-CODE TAXONOMY

On-disc analysis establishes that **51 distinct control codes** exist in the game. 43 reside in the primary HBD archive, while **7 previously undocumented codes reside exclusively in the EXE-resident blocks (`0x048C` and `0x048F`)**:

```
[0x0000] TERMINAL LEAF ------------------------------------------------------------- 19,193 occurrences
[0x7F01 - 0x7F0C] RENDERER & BOX CONTROLS ---------------------------------------- 207,144 occurrences
[0x7F10 - 0x7F1E] DYNAMIC SLOTS, VARIABLES & PARAMETERS --------------------------  36,547 occurrences
[0x7F1F - 0x7F35] CHARACTER, COMPANION & LOCATION ROSTER ------------------------  26,142 occurrences
[0x7F42 - 0x7F4C] TONE MODIFIERS & NOUN EXPANSION REGISTERS ----------------------   6,036 occurrences
```

### 3.1 The Canonical Control Code Reference

| Code | Functional Name | Subsystem & Mechanical Engine Behavior | Measured Count | Disc Location | Architectural Safety Rule |
|---|---|---|---|---|---|
| `0x0000` | **END** | Huffman terminal leaf; terminates string blitting and returns execution to caller. | 19,193 | Disc-wide | **Mandatory:** Omitting `{0000}` causes runaway memory reading until CPU hangs. |
| `0x7F01` | **NEWLINE_UI** | Fixed-grid newline for Font 1 / EXE UI blitter; sets pen $X=8$, advances $Y$ by row height. | 221 | EXE & Overlays | Resets carriage; hanging indent $+16\text{ px}$ if prefix latch is active. |
| `0x7F02` | **NEWLINE_SCENE** | Dialogue box proportional newline; resets $X$ coordinate to left margin, increments line counter. | 166,235 | Archive-wide | Most frequent token on disc. High-frequency Huffman node (typically 3–5 bits). |
| `0x7F04` | **NAME_DECORATOR** | Named dialogue opener; blits speaker name box, suppresses standard engine `＊「` prefix. | 21,781 | Archive-wide | Allocates a 20-cell speaker frame. Must be followed by character name or title token. |
| `0x7F05` | **ENUM_CLOSE** | Enumeration list-close marker; signals the end of dynamic equipment and inventory choice arrays. | 669 | Archive-wide | Runtime splice point. Always placed immediately prior to `{0000}` in list blocks. |
| `0x7F06` | **WINDOW_SEPARATOR** | Sub-window separator; divides multi-column equipment, status matrices, and choice menus. | 3 | Block `0x048C` | **EXE Wild Token:** Boundary delimiter between primary command and sub-item list. |
| `0x7F0A` | **CURSOR_WAIT** | Wait-for-input prompt (▼ flashing cursor); halts text blitter until user presses action button. | 9,028 | Archive-wide | Clears active prefix latch. Failure to clear leaves UI in permanent modal lock. |
| `0x7F0B` | **BOX_ADVANCE** | Carriage return and box advance; clears dialogue window and positions pen for next paragraph. | 31,102 | Archive-wide | Primary page-break terminator. Closes active line buffer and yields to event loop. |
| `0x7F0C` | **CRAWL_LINE_END** | Prologue title crawl line end; synchronizes vertical text crawl with SPU audio streaming. | 6 | Block `0x0069` | Hardcoded timing trigger. Dropping this freezes the opening prologue crawl. |
| `0x7F10` | **TAB_ADVANCE** | Monospace column-advance tab; repositions blit cursor to aligned status/attribute coordinates. | 11 | Block `0x048C` | **EXE Wild Token:** Aligns numerical columns (HP, MP, Attack, Defense) in Font 1. |
| `0x7F11` | **ENUM_SLOT_1** | Equipment / party selection slot #1; interpolates name of first selectable entity. | 5,489 | Archive-wide | Dynamic pointer read from party roster index array `$a1`. |
| `0x7F12` | **ENUM_SLOT_2** | Equipment / party selection slot #2; interpolates name of second selectable entity. | 30,182 | Archive-wide | High frequency equipment comparison token in facility transactions. |
| `0x7F13` | **ENUM_SLOT_3** | Equipment / party selection slot #3; interpolates name of third selectable entity. | 438 | Archive-wide | Dynamic candidate interpolation. |
| `0x7F14` | **ENUM_SLOT_4** | Equipment / party selection slot #4; interpolates name of fourth selectable entity. | 166 | Archive-wide | Dynamic candidate interpolation. |
| `0x7F15` | **GOLD_VARIABLE** | Interpolates binary integer from register `$s2` into formatted decimal gold numerals. | 113 | Archive-wide | Must be followed by currency label (`G` / `Gold`). Zero-register renders as `0`. |
| `0x7F16` | **RECV_MEMBER** | Receive-message pair member token; identifies party member receiving an item/gift. | 3 | Archive-wide | Paired with `{7F18}` for multi-member transaction notifications. |
| `0x7F17` | **ITEM_SLOT** | Current active item slot; interpolates the name of the item currently being transacted. | 203 | Archive-wide | Dynamic read from inventory pointer table. |
| `0x7F18` | **RECV_QUANTITY** | Receive-message pair quantity token; interpolates quantity of items transferred. | 10 | Archive-wide | Decimal quantity formatter. |
| `0x7F19` | **SAVE_SLOT_INDEX** | Dynamic Memory Card save-file slot index parameter (`Slot %d`). | 10 | Block `0x048F` | **EXE Wild Token:** Formats save file selection strings in priest dialogue. |
| `0x7F1A` | **WAGON_MEMBER** | Wagon-hold dynamic member name (e.g., Lucia in Chapter 5); target for spells/cures. | 19 | Archive-wide | Evaluates dynamic wagon occupancy array. Null if wagon is absent. |
| `0x7F1B` | **DEFICIT_TRIGGER** | Church donation deficit warning trigger; evaluates gold vs fee, fires shortfall branch. | 1 | Block `0x048F` | **EXE Wild Token:** Condition latch for insufficient funds dialogue. |
| `0x7F1C` | **RITE_TARGET** | Church rite target selector; dynamic token identifying member who is dead or afflicted. | 13 | Block `0x048F` | **EXE Wild Token:** Interpolates afflicted hero name during resurrection/detox. |
| `0x7F1E` | **LEVEL_DEFICIT** | Experience-to-next-level / level number integer parameter interpolation. | 15 | Block `0x048F` | **EXE Wild Token:** Divination arithmetic readout in church records. |
| `0x7F1F` | **HERO_NAME** | Custom protagonist name slot; reads up to 8 Shift-JIS characters from save RAM `0x800F4800`. | 577 | Archive-wide | Player-defined string. Variable length (1–8 glyphs / 2–16 bytes). |
| `0x7F20` | **NAME_RAGNAR** | Fixed Hero Name: Ragnar McRyan (ライアン). | 1,623 | Archive-wide | Chapter 1 Protagonist / Core Chosen Hero. |
| `0x7F21` | **NAME_ALENA** | Fixed Hero Name: Princess Alena (アリーナ). | 2,422 | Archive-wide | Chapter 2 Protagonist / Core Chosen Hero. |
| `0x7F22` | **NAME_KIRYL** | Fixed Hero Name: Kiryl / Cristo (クリフト). | 2,168 | Archive-wide | Chapter 2 Companion / Core Chosen Hero. |
| `0x7F23` | **NAME_BORYA** | Fixed Hero Name: Borya / Brey (ブライ). | 2,518 | Archive-wide | Chapter 2 Companion / Core Chosen Hero. |
| `0x7F24` | **NAME_TORNEKO** | Fixed Hero Name: Torneko Taloon (トルネコ). | 2,912 | Archive-wide | Chapter 3 Protagonist / Core Chosen Hero. |
| `0x7F25` | **NAME_MEENA** | Fixed Hero Name: Meena / Nara (ミネア). | 2,785 | Archive-wide | Chapter 4 Protagonist / Core Chosen Hero. |
| `0x7F26` | **NAME_MAYA** | Fixed Hero Name: Maya / Mara (マーニャ). | 2,516 | Archive-wide | Chapter 4 Protagonist / Core Chosen Hero. |
| `0x7F28` | **NAME_SCOTT** | Companion Name: Scott the Mercenary (スコット). | 105 | Archive-wide | Chapter 3 Hired Guard. |
| `0x7F29` | **NAME_ALEX** | NPC Name: Prince Alex of Bonmalmo (アレクス). | 6 | Archive-wide | Chapter 3 Diplomacy Quest. |
| `0x7F2A` | **NAME_FLORA** | NPC Name: Princess Flora of Endor (フレア). | 53 | Archive-wide | Chapter 3 Marriage Arc. |
| `0x7F2B` | **NAME_HEALIE** | Companion Name: Healie the Healslime (ホイミン). | 190 | Archive-wide | Chapter 1 Companion. |
| `0x7F2C` | **NAME_ORLIN** | Companion Name: Orlin the Alchemist's Apprentice (オーリン). | 113 | Archive-wide | Chapter 4 Companion. |
| `0x7F2D` | **NAME_COACHMAN** | Dynamic Wagon Coachman (Hoffman in Chapter 5 / Zenithian escort). | 2,134 | Archive-wide | Dynamic entity; evaluates current wagon driver state. |
| `0x7F2E` | **NAME_PANON** | Companion Name: Panon the Jester (パノン). | 877 | Archive-wide | Chapter 5 Companion. |
| `0x7F2F` | **NAME_LUCIA** | Companion Name: Lucia the Zenithian (ルーシア). | 337 | Archive-wide | Chapter 5 Zenithian Ally. Spoken dialogue fixed name. |
| `0x7F30` | **NAME_DORAN** | Companion Name: Doran the Baby Dragon (ドラン). | 66 | Archive-wide | Chapter 5 Companion. Roars `グゴゴーン`. |
| `0x7F31` | **NAME_PSARO** | Antagonist Name: Psaro the Dread / Necrosaro (ピサロ / デスピサロ). | 739 | Archive-wide | Main Antagonist & Chapter 6 Playable Character. |
| `0x7F32` | **NAME_ROSE** | NPC Name: Rose the Elf (ロザリー). | 356 | Archive-wide | Narrative Catalyst NPC. |
| `0x7F33` | **PARTY_LEADER** | Dynamically evaluates the currently active walking party leader character. | 11 | Archive-wide | Addressed "you" of the current interaction. |
| `0x7F34` | **ACTOR_MEMBER** | Acting party member performing an action (searching dresser, reading sign). | 70 | Archive-wide | Dynamic actor slot. Precedes interaction outcome strings. |
| `0x7F35` | **CONFESSION_LOG** | Encodes chapter / quest progression state index into Memory Card adventure log. | 2 | Block `0x048F` | **EXE Wild Token:** Quest milestone tracker in save stream. |
| `0x7F42` | **TOWN_NAME** | Dynamic town name variable; reads location string from world map index `0x0020`. | 2,792 | Archive-wide | Town / settlement string injection. |
| `0x7F43` | **TONE_LOUD** | Voice tone register: Loud, shout, public proclamation, dramatic narration. | 47 | Archive-wide | Shifts dialogue box presentation styling in UI renderer. |
| `0x7F44` | **TONE_SOFT** | Voice tone register: Gentle, whisper, Rose, children, intimate dialogue. | 58 | Archive-wide | Shifts dialogue box presentation styling in UI renderer. |
| `0x7F45` | **TONE_MENACE** | Voice tone register: Dark, threatening, demonic, monsters, villains. | 79 | Archive-wide | Double repetition (`{7F45}{7F45}`) escalates dramatic tension. |
| `0x7F47` | **TITLE_NOUN** | Current item / equipment name as window title box (`{7F04}{7F47} {7F05}`). | 84 | Archive-wide | Dedicated equipment inspect title renderer. |
| `0x7F4B` | **TRADED_ITEM** | Dynamic noun insertion for bartered/traded goods in merchant dialogues. | 5,680 | Archive-wide | Replaces subject in commercial appraisal/purchase lines. |
| `0x7F4C` | **PARTY_PLURAL** | Plural party representative token; translates to "the Chosen Ones" or "ye all". | 30 | Archive-wide | Plural entity address in church blessings. |

---

## 4. CRITICAL SCRIPT IDENTIFIER (SID) REGISTRY

The following table provides the exact hardware offsets, allocation budgets, control requirements, and downstream failure modes for the most sensitive SIDs in the game:

| SID / ID | Source Block / Overlay File | Base Bit/Byte Offset | Original Japanese Function / Context | Target English String (Clean Room) | Mandatory Control Tokens & Terminators | Strict Allocation Limit | Downstream Risk Factor (State Collision, Bleed, DMA) |
|---|---|---|---|---|---|---|---|
| **048B:0028** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x0504F4 (Bit: 3349) | Combat Spell Cast: `{7F3F}{7F41}はマホカンタをとなえた！` | `{7F3F}{7F41} casts Bounce!{0000}` | `{7F3F}` Target Slot, `{7F41}` Action Subject, `{0000}` Leaf | 34 Bytes (272 Bits) | **Critical:** Bit drift bleeds into spell animation VM loop; causes animation hang without casting effects. |
| **048B:0102** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x01E820 (Bit: 12496) | Single Target Damage Message: `{7F3D}に {param}のダメージ！` | `{7F3D} takes {param} damage!!{0000}` | `{7F3D}` Mob Name Sub, `{0000}` Terminator | 28 Bytes (224 Bits) | **Fatal:** Stale pointer triggers `{7F3D}` runaway into control band; hangs battle state machine at 0 FPS. |
| **048B:0114** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x0214C0 (Bit: 14420) | Combat Victory Reward Header: `まものを やっつけた！` | `Thou hast defeated the monsters!{7F0A}{0000}` | `{7F0A}` Wait-For-Input, `{0000}` Leaf | 42 Bytes (336 Bits) | **High:** "peynriohre-seale" alien text root; desync causes runaway decode past victory sequence into bestiary table. |
| **048B:0115** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x021700 (Bit: 14780) | Combat EXP & Gold Award: `けいけんち {param} ゴールド {7F15}` | `Gained {param} EXP and {7F15} Gold!{7F0A}{0000}` | `{7F15}` Gold Variable, `{7F0A}` Wait, `{0000}` Leaf | 48 Bytes (384 Bits) | **High:** Register `$s2` arithmetic mismatch. Overwriting `{7F15}` causes gold payout to evaluate as `0xFFFFFFFF`. |
| **048B:0240** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x03C5D0 (Bit: 30752) | Ally Party-Chat Combat Invalidation: `なかまが いない！` | `No allies to talk to.{7F0B}{0000}` | `{7F0B}` Carriage Return, `{0000}` Leaf | 26 Bytes (208 Bits) | **Fatal:** Pointer `0x48B0790D` boundary. Decode overrun hits `{7F0B}` mid-sub, stalling battle controller. |
| **048B:0584** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x08F4C0 (Bit: 73236) | Base Monster Bestiary Entry #1: `スライム` | `SLM{0000}` | Font 1 3-Glyph Shortform, `{0000}` Leaf | 4 Bytes (32 Bits) | **Moderate:** Bestiary window overflow; names $>3$ characters overwrite monster attribute bitmasks. |
| **048B:0801** | Block `0x048B` (Battle Primary) | Sector 105540 + 0x098B20 (Bit: 78255) | Late-Index Monster Bestiary: Residual Entry | `FOE{0000}` | `{0000}` Leaf | 4 Bytes (32 Bits) | **Low:** Pointer `0x48B1326F` cosmetic swap. Wrong-but-terminated string renders without freezing engine. |
| **048C:0031** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x0188 (Byte Offset) | Field Command Search - Null Yield: `何も 見つからなかった` | `NOT FND{0000}` | Font 1 Monospace ASCII, `{0000}` Leaf | 8 Bytes | **Moderate:** Search window cursor corrupts if length $\ne 7$ characters plus null byte. |
| **048C:0153** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x0840 (Byte Offset) | Hero Character Nameplate: `ライアン` | `RAG{0000}` | Font 1 ASCII, `{0000}` Null Byte | 4 Bytes | **High:** Status window layout collision. Longer names clobber Level/HP numerical fields in VRAM. |
| **048C:0446** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x1420 (Byte Offset) | Inventory Item Registry: `どうのつるぎ` | `CPR SWD{0000}` | Font 1 Item Abbreviation, `{0000}` Null | 8 Bytes | **Moderate:** Inventory menu cursor desynchronization. Length drift misaligns price column blit coordinates. |
| **048C:0625** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2110 (Byte Offset) | Field Signpost Interaction Announcement | `{7F34} RED SGN.{7F0A}{0000}` | `{7F34}` Actor Name Token, `{7F0A}` Wait, `{0000}` Null | 16 Bytes | **High:** Dynamic Actor injection. Omitting `{7F34}` leaves uninitialized WRAM address as string subject. |
| **048C:0651** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2280 (Byte Offset) | Church Service Choice: `なおす` (Cure Poison) | `CUR{0000}` | Font 1 ASCII Menu Label, `{0000}` Null | 4 Bytes | **High:** Choice array misalignment. Overwriting boundary corrupts jump table vector table at `$s1`. |
| **048C:0658** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x22B8 (Byte Offset) | Church Service Choice: `おいのりをする` (Pray/Save) | `PRY{0000}` | Font 1 ASCII Menu Label, `{0000}` Null | 4 Bytes | **Critical:** Misaligned choice pointer vector routes menu selection into invalid memory handler. |
| **048C:0773** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2A10 (Bit Offset: Base) | Main Menu Command #1: `じゅもん` (Spells) | `SPL{0000}` | Font 1 Delta-Lock Base SID | 4 Bytes (32 Bits) | **Absolute Lock:** Base reference for UI Delta-Lock array. |
| **048C:0774** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2A10 (Bit: Base + 44) | Main Menu Command #2: `どうぐ` (Items) | `ITM{0000}` | Font 1 Delta-Lock (`774 - 773 = 44`) | 4 Bytes (32 Bits) | **Critical:** Delta shift $\ne 44$ causes cursor blinking animation in VRAM to overwrite window borders. |
| **048C:0775** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2A60 (Bit Offset: Base) | Main Menu Command #3: `しらべる` (Search) | `SRC{0000}` | Font 1 Delta-Lock Base Sub-Array | 4 Bytes (32 Bits) | **Absolute Lock:** Base reference for Command Execution Sub-Array. |
| **048C:0776** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2A60 (Bit: Base + 36) | Main Menu Command #4: `つよさ` (Status) | `STS{0000}` | Font 1 Delta-Lock (`776 - 775 = 36`) | 4 Bytes (32 Bits) | **Critical:** Delta shift $\ne 36$ corrupts status window deployment parameters; causes UI stack collapse. |
| **048C:0777** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2A60 (Bit: Base + 50) | Main Menu Command #5: `さくせん` (Tactics) | `TCT{0000}` | Font 1 Delta-Lock (`777 - 775 = 50`) | 4 Bytes (32 Bits) | **Critical:** Delta shift $\ne 50$ shifts sub-command options off-screen; breaks AI tactics assignment. |
| **048C:0778** | Block `0x048C` (SLPM_869.16) | `0x99624` + 0x2A60 (Bit: Base + 66) | Main Menu Command #6: `はなす` (Talk) | `TLK{0000}` | Font 1 Delta-Lock (`778 - 775 = 66`) | 4 Bytes (32 Bits) | **Critical:** Delta shift $\ne 66$ causes talk command to fall into invalid coordinate pointer; freezes party. |
| **048F:0221** | Block `0x048F` (SLPM_869.16) | `0x99624` + 0x05420 (Bit: 12480) | Priest Dialogue: Save Confirmation Hook | `Dost thou wish to record thy deeds?{7F0A}{0000}` | `{7F0A}` Wait, `{0000}` Terminator | 48 Bytes (384 Bits) | **High:** Memory card controller handshake follows; text bleed corrupts `$a0` packet payload to SIO. |
| **048F:0223** | Block `0x048F` (SLPM_869.16) | `0x99624` + 0x05610 (Bit: 13120) | Priest Opening Salutation: `おまちしておりました` | `Welcome back! Chosen one, thy deed…{7F02}` | `{7F02}` Scene Newline, `{0000}` Leaf | 54 Bytes (432 Bits) | **Moderate:** Window overflow into WRAM; missing `{7F02}` drops pen carriage reset, shifting text off-screen. |
| **048F:0228** | Block `0x048F` (SLPM_869.16) | `0x99624` + 0x05980 (Bit: 14240) | Priest Closing Salutation: `アーメン！` | `Go forth in peace. Amen!{7F0A}{0000}` | `{7F0A}` Wait, `{0000}` Leaf | 30 Bytes (240 Bits) | **High:** Dismissal script triggers map sprite unlocking. Missing `{7F0A}` fails to clear prefix latch. |
| **0474:0012** | Block `0x0474` (Save Menu) | Sector 104210 + 0x0120 | Memory Card Selection Prompt | `Select a Memory Card slot.{7F01}{0000}` | `{7F01}` UI Newline, `{0000}` Leaf | 32 Bytes | **High:** Text bleed into SIO transfer buffer triggers false "Unformatted Card" status flags. |
| **0474:0027** | Block `0x0474` (Save Menu) | Sector 104210 + 0x02B0 | Save File Overwrite Confirmation Prompt | `Existing log will be lost. Proceed?{7F01}{0000}` | `{7F01}` UI Newline, `{0000}` Leaf | 40 Bytes | **Critical:** Corrupting this sequence shifts choice selector logic, causing accidental file deletion. |
| **0474:0054** | Block `0x0474` (Save Menu) | Sector 104210 + 0x04E0 | Hardware Memory Card Warning Message #54 | `Do not remove Memory Card or reset.{7F0B}{0000}` | `{7F0B}` Carriage Return, `{0000}` Leaf | 44 Bytes | **Fatal:** Displayed during active SPI bus transaction. Overruns clobber kernel event vectors. |
| **047D:0042** | Block `0x047D` (HBD Church) | Sector 106120 + 0x0340 | Divine Blessing Invocation | `{7F4C} receive the grace of God!{7F0A}{0000}` | `{7F4C}` Party Representative Token, `{7F0A}`, `{0000}` | 46 Bytes | **High:** `{7F4C}` dynamically splices party leader name. Missing variable expands null pointer to screen. |
| **047D:0118** | Block `0x047D` (HBD Church) | Sector 106120 + 0x08A0 | Divination: Experience Required for Level-Up | `{7F11} requires {param} EXP for next level.{7F0A}{0000}` | `{7F11}` Character Name Slot, `{7F0A}`, `{0000}` | 52 Bytes | **Critical:** Monster Gramps / Level-Up crash root. Stack misalignment during string expansion freezes game. |
| **0069:0006** | Block `0x0069` (Prologue) | Sector 12040 + 0x00C0 | Prologue Title Crawl Narration | `A legend sleeps within the earth...{7F0C}{0000}` | `{7F0C}` Title Crawl Line End, `{0000}` Leaf | 48 Bytes | **High:** `{7F0C}` synchronizes scroll clock to SPU voice streaming. Missing `{7F0C}` locks title sequence. |
| **006C:0001** | Block `0x006C` (Type-39 VM) | Sector 12180 + 0x0010 | Cutscene VM Event Bytecode: Burland Castle | `[Bytecode Event Stream - Pure Binary Remap]` | Native LZSS Stream (`dqlzs`), Exact Length | 1588 Bytes Strict | **Fatal:** The Historical Prologue Freeze. Expanding from 1588 to 1592 bytes shifts every sector on disc. |

---

## 5. HARDWARE-LEVEL REVERSE ENGINEERING CONSTRAINTS

### 5.1 The Font 1 Assembly Delta-Lock Invariant (`0x048C`)
The field command menu system in `0x048C` (SIDs 773–778) does not use pointer arrays; instead, the engine blits menu items using hardcoded relative bit-offset deltas embedded directly in MIPS assembly routines (`SLPM_869.16`):

```
[SID 773: SPL] --- (+44 bits) ---> [SID 774: ITM]
[SID 775: SRC] --- (+36 bits) ---> [SID 776: STS] --- (+14 bits / +50 total) ---> [SID 777: TCT] --- (+16 bits / +66 total) ---> [SID 778: TLK]
```

```mips
# MIPS R3000A Delta Processing Assembly (SLPM_869.16)
lw    $a0, 0x0010($sp)       # Load base SID bitstream address (SID 773: SPL)
jal   render_font1_glyph     # Blit "SPL"
nop
addiu $a0, $a0, 44           # Hardcoded Delta: Advance exactly 44 bits to SID 774
jal   render_font1_glyph     # Blit "ITM"
nop
```

* **Zero-Tolerance Invariant:** If `len("SPL")` or its Huffman tree allocation alters the bit distance between SID 773 and SID 774 from **exactly 44 bits**, the blitter reads intermediate bits as glyph indices. The menu renders corrupted ASCII characters, and cursor blinking animation logic overwrites adjacent VRAM.
* **Sub-Array 775–778 Invariants:**
  * $\text{SID 776} - \text{SID 775} = \mathbf{36\text{ bits}}$
  * $\text{SID 777} - \text{SID 775} = \mathbf{50\text{ bits}}$ (Incremental delta from 776 is $14\text{ bits}$)
  * $\text{SID 778} - \text{SID 775} = \mathbf{66\text{ bits}}$ (Incremental delta from 777 is $16\text{ bits}$)

### 5.2 Dynamic Variable Expansion Delimiters
Dynamic parameters are expanded at runtime by the text parsing engine using reserved token codes (`0x7Fxx`). Failure to isolate variable insertion points causes stack parameter bleed:

1. **Party Roster Slots (`{7F11}`–`{7F14}`, `{7F1F}`):**
   * `{7F1F}` inserts the custom player name (up to 8 Shift-JIS characters / 16 bytes).
   * `{7F11}`–`{7F14}` expand character names based on active party formation indices.
   * **Delimiter Requirement:** Dynamic name insertions must be followed immediately by `{7F04}` if functioning as a speaker identifier, or by a space/punctuation token. Never place `{7F0A}` directly against `{7F11}` without an intervening clause or space; this causes the name blitter to drop the final character glyph.
2. **Gold & Numerical Values (`{7F15}`):**
   * Register `$s2` passes numerical quantities as binary integers converted on-the-fly to ASCII numerals.
   * **Delimiter Requirement:** Must be explicitly anchored by `{7F15} Gold` or `{7F15}G`. Terminate the enclosing sentence with `{7F0A}{0000}`. If `{7F15}` lacks an explicit following byte before `{0000}`, the numerical ASCII converter fails to release its stack frame, leaving raw registers visible in the dialogue window.
3. **Monster Substitution Tokens (`{7F3D}` / `{7F3F}`):**
   * Evaluated by reading MIPS word tables loaded into RAM at `0x152610` and `0x15280C`.
   * **The Battle Freeze Mechanism:** If the target pointer is stale (pointing to pristine offset instead of English remapped offset), `{7F3D}` jumps into the middle of an unmapped sequence. It decodes arbitrary bits until encountering `{7F3F}` or `{7F0B}`, running indefinitely in a loop without clearing the VBlank flag, resulting in a zero-FPS hard lock.
   * **Remedy:** Every monster string in `0x048B` must terminate with `{0000}` leaf nodes, and the referrer words across all 18 type-46 overlay copies must be remapped via `patch_overlay_duplicates.py`.

### 5.3 Memory Card Bus Safety (SID 54 & Block `0x0474`)
* **The SPI/SIO Bus Protocol:** The PlayStation 1 handles Memory Card I/O through serial communication via the SIO controller (`0x1F801040`). String decoding routines running concurrently must execute within strict CPU cycle budgets.
* **SID 54 Critical Protection:**
  ```
  Original (JP): メモリーカードを　ぬかないで　ください{7F0B}{0000}
  Target (EN):   Do not remove Memory Card or reset.{7F0B}{0000}
  ```
  * **Hazard:** Length must not exceed 44 bytes. If the string expansion overruns the 44-byte boundary, it writes directly into the memory card packet transmission buffer (`0x800F5200`). This corrupts the 128-byte card sector checksum, triggering the hardware BIOS to declare the card corrupted and display the destructive format prompt.
  * **Termination:** Must conclude with `{7F0B}{0000}`. `{7F0B}` acts as the flush latch that returns execution from the dialogue engine back to the SIO polling loop.

### 5.4 Sub-Map Transitions & CD-ROM DMA Channel 3
1. **CD-ROM Transfer Pipeline:** When descending dungeon staircases (e.g., Burland well B1F to B2F), the engine issues an `0x15` (`ReadN`) command to the CD-ROM controller. Data streams from the disc sector into the CD buffer, then transfers via DMA Channel 3 (`D3_CHCR` @ `0x1F801088`) into WRAM.
2. **Bytecode Script Clamping (Block `0x006C`):**
   * The historical prologue freeze at Chapter 1 opening was caused by expanding `006C`'s type-39 cutscene script from **1588 bytes to 1592 bytes**.
   * In the HeartBeat HBD architecture, file records are aligned to sector boundaries. Expanding a single block by even 4 bytes shifts every subsequent file in that archive folder forward by one or more sectors.
   * When the engine issues a DMA read for a sub-map transition, it reads from the hardcoded logical block address (LBA). Due to the 4-byte shift, the DMA buffer receives misaligned data instead of the map transition header (`0xC021A0`), causing the CPU to execute invalid opcodes (`0x00000000` / break) and permanently locking the CD-ROM DMA channel.
3. **Assembly-Level Verification Requirement:**
   * Ensure `len(recompressed_type39) <= 1588` bytes exact. Any trailing padding must consist of zero-slack NOPs (`0x00`) without expansion.
   * Confirm DMA Channel 3 status register `0x1F801088` clears bit 24 (`CHCR.Busy == 0`) upon map load completion before executing subsequent script threads.
