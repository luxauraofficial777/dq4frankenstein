MASTER SCRIPT IDENTIFIER (SID) & REFERRER MATRIX
Architecture: Sony PlayStation 1 (SLPM_869.16 / MIPS R3000A @ 33.8688 MHz)
Target Engine: HeartBeat Engine (HBD1PS1D.Q41 / Dual-Blitter Hybrid Architecture)
Classification: Core System Architecture & Decompiled String Dispatch Matrix

1. Architectural Overview: The HeartBeat Decompression Subsystem
The PSX HeartBeat Engine decodes script text dynamically through a dual-blitter architecture:

Font 1 (System / Fixed-Grid): Fixed 8×14 cell single-byte ASCII glyphs loaded into VRAM buffers. Used for status matrices, item/spell names, tactical registries, and combat menus.
Font 2 (Dialogue / Proportional): 2-byte Shift-JIS / DW7-page hybrid characters parsed via variable-length canonical Huffman trees into WRAM work buffer 0x800F4DF0.


                +----------------------------------------------+
                |  Huffman Compressed Bitstream (HBD Archive)  |
                +----------------------------------------------+
                                       |
                                       v
         +------------------------------------------------------------+
         | Bitstream Reader: MIPS R3000A Shift & Branch Tree Traversal |
         +------------------------------------------------------------+
                                       |
                      +----------------+----------------+
                      |                                 |
           [Bit 0: Left Child]                [Bit 1: Right Child]
                      \                                 /
                       v                               v
             +--------------------+          +--------------------+
             |  Internal Node     |          | Terminal Leaf      |
             |  (Next Bit Read)   |          | (Glyph / Control)  |
             +--------------------+          +--------------------+
                                                        |
                                       +----------------+----------------+
                                       |                                 |
                                [ASCII / SJIS Glyph]             [Control Token]
                                       |                                 |
                                       v                                 v
                             +-------------------+             +-------------------+
                             |  Work Buffer      |             | Engine Parser     |
                             |  (0x800F4DF0)     |             | Dynamic Sub / Nav |
                             +-------------------+             +-------------------+
Core Engine Vulnerability Rules
Bit-Phase Desynchronization: Huffman trees are variable-length bitstreams. A single-bit offset drift (
N
±
1
N±1) desynchronizes the entire decoding state machine, routing the decoder down invalid branches. Instead of string terminators ({0000}), the decoder interprets random bit patterns as display characters or runaway control tokens (producing the alien text syndrome, such as "peynriohre-seale").
Buffer Bleed & DMA Channel 3 Lockout: Strings that lack terminal leaves ({0000}) bleed across adjacent heap structures in WRAM. If an unbounded string overrun overwrites the CD-ROM status loop or preempts DMA Channel 3 (0x1F801088), the engine halts CD streaming, causing a hard freeze at sub-map transitions or memory card operations.
MIPS Sign-Carry Calculation: MIPS immediate loading uses split arithmetic: 
Target
=
(
HighHalf
≪
16
)
+
SignExtend
(
LowHalf
)
Target=(HighHalf≪16)+SignExtend(LowHalf) When Bit 15 of LowHalf is 
≥
0x8000
≥0x8000, addiu sign-extends with negative polarity. The compiler compensates with HighHalf = (val + 0x8000) >> 16. Any manual referrer patch that omits the + 0x8000 sign-carry shifts execution into memory 
64
 KB
64 KB below the target block.
2. Master SID Matrix
SID / ID	Source Block / Overlay File	Base Bit/Byte Offset	Original Japanese Function / Context	Target English String (Clean Room)	Mandatory Control Tokens & Terminators	Strict Allocation Limit (Bytes/Bits)	Downstream Risk Factor (State Collision, Bleed, DMA)
048B:0028	Block 0x048B (Battle Primary)	Sector 105540 + 0x0504F4 (Bit: 3349)	Combat Spell Cast: {7F3F}{7F41}はマホカンタをとなえた！	{7F3F}{7F41} casts Bounce!{0000}	{7F3F} Target Slot, {7F41} Action Subject, {0000} Leaf	34 Bytes (272 Bits)	Critical: Bit drift bleeds into spell animation VM loop; causes animation hang without casting effects.
048B:0102	Block 0x048B (Battle Primary)	Sector 105540 + 0x01E820 (Bit: 12496)	Single Target Damage Message: {7F3D}に {param}のダメージ！	{7F3D} takes {param} damage!!{0000}	{7F3D} Mob Name Sub, {0000} Terminator	28 Bytes (224 Bits)	Fatal: Stale pointer triggers {7F3D} runaway into control band; hangs battle state machine at 0 FPS.
048B:0114	Block 0x048B (Battle Primary)	Sector 105540 + 0x0214C0 (Bit: 14420)	Combat Victory Reward Header: まものを やっつけた！	Thou hast defeated the monsters!{7F0A}{0000}	{7F0A} Wait-For-Input, {0000} Leaf	42 Bytes (336 Bits)	High: "peynriohre-seale" alien text root; desync causes runaway decode past victory sequence into bestiary table.
048B:0115	Block 0x048B (Battle Primary)	Sector 105540 + 0x021700 (Bit: 14780)	Combat EXP & Gold Award: けいけんち {param} ゴールド {7F15}	Gained {param} EXP and {7F15} Gold!{7F0A}{0000}	{7F15} Gold Variable, {7F0A} Wait, {0000} Leaf	48 Bytes (384 Bits)	High: Register $s2 arithmetic mismatch. Overwriting {7F15} causes gold payout to evaluate as 0xFFFFFFFF.
048B:0240	Block 0x048B (Battle Primary)	Sector 105540 + 0x03C5D0 (Bit: 30752)	Ally Party-Chat Combat Invalidation: なかまが いない！	No allies to talk to.{7F0B}{0000}	{7F0B} Carriage Return, {0000} Leaf	26 Bytes (208 Bits)	Fatal: Pointer 0x48B0790D boundary. Decode overrun hits {7F0B} mid-sub, stalling battle controller.
048B:0584	Block 0x048B (Battle Primary)	Sector 105540 + 0x08F4C0 (Bit: 73236)	Base Monster Bestiary Entry #1: スライム	SLM{0000}	Font 1 3-Glyph Shortform, {0000} Leaf	4 Bytes (32 Bits)	Moderate: Bestiary window overflow; names 
>
3
>3 characters overwrite monster attribute bitmasks.
048B:0801	Block 0x048B (Battle Primary)	Sector 105540 + 0x098B20 (Bit: 78255)	Late-Index Monster Bestiary: Residual Entry	FOE{0000}	{0000} Leaf	4 Bytes (32 Bits)	Low: Pointer 0x48B1326F cosmetic swap. Wrong-but-terminated string renders without freezing engine.
048C:0031	Block 0x048C (SLPM_869.16)	0x99624 + 0x0188 (Byte Offset)	Field Command Search - Null Yield: 何も 見つからなかった	NOT FND{0000}	Font 1 Monospace ASCII, {0000} Leaf	8 Bytes	Moderate: Search window cursor corrupts if length 
≠
7

=7 characters plus null byte.
048C:0153	Block 0x048C (SLPM_869.16)	0x99624 + 0x0840 (Byte Offset)	Hero Character Nameplate: ライアン	RAG{0000}	Font 1 ASCII, {0000} Null Byte	4 Bytes	High: Status window layout collision. Longer names clobber Level/HP numerical fields in VRAM.
048C:0446	Block 0x048C (SLPM_869.16)	0x99624 + 0x1420 (Byte Offset)	Inventory Item Registry: どうのつるぎ	CPR SWD{0000}	Font 1 Item Abbreviation, {0000} Null	8 Bytes	Moderate: Inventory menu cursor desynchronization. Length drift misaligns price column blit coordinates.
048C:0625	Block 0x048C (SLPM_869.16)	0x99624 + 0x2110 (Byte Offset)	Field Signpost Interaction Announcement	{7F34} RED SGN.{7F0A}{0000}	{7F34} Actor Name Token, {7F0A} Wait, {0000} Null	16 Bytes	High: Dynamic Actor injection. Omitting {7F34} leaves uninitialized WRAM address as string subject.
048C:0651	Block 0x048C (SLPM_869.16)	0x99624 + 0x2280 (Byte Offset)	Church Service Choice: なおす (Cure Poison)	CUR{0000}	Font 1 ASCII Menu Label, {0000} Null	4 Bytes	High: Choice array misalignment. Overwriting boundary corrupts jump table vector table at $s1.
048C:0658	Block 0x048C (SLPM_869.16)	0x99624 + 0x22B8 (Byte Offset)	Church Service Choice: おいのりをする (Pray/Save)	PRY{0000}	Font 1 ASCII Menu Label, {0000} Null	4 Bytes	Critical: Misaligned choice pointer vector routes menu selection into invalid memory handler.
048C:0773	Block 0x048C (SLPM_869.16)	0x99624 + 0x2A10 (Bit Offset: Base)	Main Menu Command #1: じゅもん (Spells)	SPL{0000}	Font 1 Delta-Lock Base SID	4 Bytes (32 Bits)	Absolute Lock: Base reference for UI Delta-Lock array.
048C:0774	Block 0x048C (SLPM_869.16)	0x99624 + 0x2A10 (Bit: Base + 44)	Main Menu Command #2: どうぐ (Items)	ITM{0000}	Font 1 Delta-Lock (774 - 773 = 44)	4 Bytes (32 Bits)	Critical: Delta shift 
≠
44

=44 causes cursor blinking animation in VRAM to overwrite window borders.
048C:0775	Block 0x048C (SLPM_869.16)	0x99624 + 0x2A60 (Bit Offset: Base)	Main Menu Command #3: しらべる (Search)	SRC{0000}	Font 1 Delta-Lock Base Sub-Array	4 Bytes (32 Bits)	Absolute Lock: Base reference for Command Execution Sub-Array.
048C:0776	Block 0x048C (SLPM_869.16)	0x99624 + 0x2A60 (Bit: Base + 36)	Main Menu Command #4: つよさ (Status)	STS{0000}	Font 1 Delta-Lock (776 - 775 = 36)	4 Bytes (32 Bits)	Critical: Delta shift 
≠
36

=36 corrupts status window deployment parameters; causes UI stack collapse.
048C:0777	Block 0x048C (SLPM_869.16)	0x99624 + 0x2A60 (Bit: Base + 50)	Main Menu Command #5: さくせん (Tactics)	TCT{0000}	Font 1 Delta-Lock (777 - 775 = 50)	4 Bytes (32 Bits)	Critical: Delta shift 
≠
50

=50 shifts sub-command options off-screen; breaks AI tactics assignment.
048C:0778	Block 0x048C (SLPM_869.16)	0x99624 + 0x2A60 (Bit: Base + 66)	Main Menu Command #6: はなす (Talk)	TLK{0000}	Font 1 Delta-Lock (778 - 775 = 66)	4 Bytes (32 Bits)	Critical: Delta shift 
≠
66

=66 causes talk command to fall into invalid coordinate pointer; freezes party.
048F:0221	Block 0x048F (SLPM_869.16)	0x99624 + 0x05420 (Bit: 12480)	Priest Dialogue: Save Confirmation Hook	Dost thou wish to record thy deeds?{7F0A}{0000}	{7F0A} Wait, {0000} Terminator	48 Bytes (384 Bits)	High: Memory card controller handshake follows; text bleed corrupts $a0 packet payload to SIO.
048F:0223	Block 0x048F (SLPM_869.16)	0x99624 + 0x05610 (Bit: 13120)	Priest Opening Salutation: おまちしておりました	Welcome back! Chosen one, thy deed…{7F02}	{7F02} Scene Newline, {0000} Leaf	54 Bytes (432 Bits)	Moderate: Window overflow into WRAM; missing {7F02} drops pen carriage reset, shifting text off-screen.
048F:0228	Block 0x048F (SLPM_869.16)	0x99624 + 0x05980 (Bit: 14240)	Priest Closing Salutation: アーメン！	Go forth in peace. Amen!{7F0A}{0000}	{7F0A} Wait, {0000} Leaf	30 Bytes (240 Bits)	High: Dismissal script triggers map sprite unlocking. Missing {7F0A} fails to clear prefix latch.
0474:0012	Block 0x0474 (Save Menu)	Sector 104210 + 0x0120	Memory Card Selection Prompt	Select a Memory Card slot.{7F01}{0000}	{7F01} UI Newline, {0000} Leaf	32 Bytes	High: Text bleed into SIO transfer buffer triggers false "Unformatted Card" status flags.
0474:0027	Block 0x0474 (Save Menu)	Sector 104210 + 0x02B0	Save File Overwrite Confirmation Prompt	Existing log will be lost. Proceed?{7F01}{0000}	{7F01} UI Newline, {0000} Leaf	40 Bytes	Critical: Corrupting this sequence shifts choice selector logic, causing accidental file deletion.
0474:0054	Block 0x0474 (Save Menu)	Sector 104210 + 0x04E0	Hardware Memory Card Warning Message #54	Do not remove Memory Card or reset.{7F0B}{0000}	{7F0B} Carriage Return, {0000} Leaf	44 Bytes	Fatal: Displayed during active SPI bus transaction. Overruns clobber kernel event vectors.
047D:0042	Block 0x047D (HBD Church)	Sector 106120 + 0x0340	Divine Blessing Invocation	{7F4C} receive the grace of God!{7F0A}{0000}	{7F4C} Party Representative Token, {7F0A}, {0000}	46 Bytes	High: {7F4C} dynamically splices party leader name. Missing variable expands null pointer to screen.
047D:0118	Block 0x047D (HBD Church)	Sector 106120 + 0x08A0	Divination: Experience Required for Level-Up	{7F11} requires {param} EXP for next level.{7F0A}{0000}	{7F11} Character Name Slot, {7F0A}, {0000}	52 Bytes	Critical: Monster Gramps / Level-Up crash root. Stack misalignment during string expansion freezes game.
0069:0006	Block 0x0069 (Prologue)	Sector 12040 + 0x00C0	Prologue Title Crawl Narration	A legend sleeps within the earth...{7F0C}{0000}	{7F0C} Title Crawl Line End, {0000} Leaf	48 Bytes	High: {7F0C} synchronizes scroll clock to SPU voice streaming. Missing {7F0C} locks title sequence.
006C:0001	Block 0x006C (Type-39 VM)	Sector 12180 + 0x0010	Cutscene VM Event Bytecode: Burland Castle	[Bytecode Event Stream - Pure Binary Remap]	Native LZSS Stream (dqlzs), Exact Length	1588 Bytes Strict	Fatal: The Historical Prologue Freeze. Expanding from 1588 to 1592 bytes shifts every sector on disc.
3. Implementation Notes & Technical Constraints
A. The Font 1 System Menu Delta-Locks (0x048C)
The field command menu system in 0x048C (SIDs 773–778) does not use pointer arrays; instead, the engine blits menu items using hardcoded relative bit-offset deltas embedded in MIPS assembly routines (SLPM_869.16):



[SID 773: SPL] --- (+44 bits) ---> [SID 774: ITM]
[SID 775: SRC] --- (+36 bits) ---> [SID 776: STS] --- (+14 bits / +50 total) ---> [SID 777: TCT] --- (+16 bits / +66 total) ---> [SID 778: TLK]
mips


# MIPS R3000A Delta Processing Assembly (SLPM_869.16)
lw    $a0, 0x0010($sp)       # Load base SID address (SID 773: SPL)
jal   render_font1_glyph     # Blit "SPL"
nop
addiu $a0, $a0, 44           # Hardcoded Delta: Jump exactly 44 bits to SID 774
jal   render_font1_glyph     # Blit "ITM"
nop
Zero-Tolerance Invariant: If len("SPL") or its Huffman tree allocation changes the bit distance between SID 773 and SID 774 from exactly 44 bits, the blitter reads intermediate bits as glyph indices. The menu renders corrupted ASCII characters, and the cursor blinking animation logic overwrites adjacent VRAM.
Command Sequence 775–778: The deltas relative to SID 775 (SRC) must remain locked:
SID 776 - SID 775 == 36 bits
SID 777 - SID 775 == 50 bits (Delta from 776 is 
14
 bits
14 bits)
SID 778 - SID 775 == 66 bits (Delta from 777 is 
16
 bits
16 bits)
B. Dynamic Variable Interpolation & Delimiter Rules
Dynamic parameters are expanded at runtime by the text parsing engine using reserved token codes (0x7Fxx). Failure to isolate variable insertion points causes stack parameter bleed:

Party Roster Slots ({7F11}–{7F14}, {7F1F}):
{7F1F} inserts the custom player name (up to 8 Shift-JIS characters / 16 bytes).
{7F11}–{7F14} expand character names based on active party formation indices.
Delimiter Requirement: Dynamic name insertions must be followed immediately by {7F04} if functioning as a speaker identifier, or by a space/punctuation token. Never place {7F0A} directly against {7F11} without an intervening clause or space; this causes the name blitter to drop the final character glyph.
Gold & Numerical Values ({7F15}):
Register $s2 passes numerical quantities as binary integers converted on-the-fly to ASCII numerals.
Delimiter Requirement: Must be explicitly anchored by {7F15} Gold or {7F15}G. Terminate the enclosing sentence with {7F0A}{0000}. If {7F15} lacks an explicit following byte before {0000}, the numerical ASCII converter fails to release its stack frame, leaving raw registers visible in the dialogue window.
Monster Substitution Tokens ({7F3D} / {7F3F}):
Evaluated by reading MIPS word tables loaded into RAM at 0x152610 and 0x15280C.
The Battle Freeze Mechanism: If the target pointer is stale (pointing to pristine offset instead of English remapped offset), {7F3D} jumps into the middle of an unmapped sequence. It decodes arbitrary bits until encountering {7F3F} or {7F0B}, running indefinitely in a loop without clearing the VBlank flag, resulting in a zero-FPS hard lock.
Remedy: Every monster string in 0x048B must terminate with {0000} leaf nodes, and the referrer words across all 18 type-46 overlay copies must be remapped via patch_overlay_duplicates.py.
C. Memory Card Safety & DMA Synchronization (SID 54 & Block 0x0474)
During church saves and memory card operations:

The SPI/SIO Bus Protocol: The PlayStation 1 handles Memory Card I/O through serial communication via the SIO controller (0x1F801040). String decoding routines running concurrently must execute within strict CPU cycle budgets.
SID 54 Critical Protection:


Original (JP): メモリーカードを　ぬかないで　ください{7F0B}{0000}
Target (EN):   Do not remove Memory Card or reset.{7F0B}{0000}
Hazard: Length must not exceed 44 bytes. If the string expansion overruns the 44-byte boundary, it writes directly into the memory card packet transmission buffer (0x800F5200). This corrupts the 128-byte card sector checksum, triggering the hardware BIOS to declare the card corrupted and display the destructive format prompt.
Termination: Must conclude with {7F0B}{0000}. {7F0B} acts as the flush latch that returns execution from the dialogue engine back to the SIO polling loop.
D. Sub-Map Transition Freezes & CD-ROM DMA Channel 3
When transitioning between maps (e.g., stairs descending from B1F to B2F in the Burland well):

CD-ROM Transfer Pipeline: The engine issues an 0x15 (ReadN) command to the CD-ROM controller. Data streams from the disc sector into the CD buffer, then transfers via DMA Channel 3 (D3_CHCR @ 0x1F801088) into WRAM.
Bytecode Script Clamping (Block 0x006C):
The historical prologue freeze at Chapter 1 opening was caused by Markus's early Java patcher expanding 006C's type-39 cutscene script from 1588 bytes to 1592 bytes.
In the HeartBeat HBD architecture, file records are aligned to sector boundaries. Expanding a single block by even 4 bytes shifts every subsequent file in that archive folder forward by one or more sectors.
When the engine issues a DMA read for a sub-map transition, it reads from the hardcoded logical block address (LBA). Due to the 4-byte shift, the DMA buffer receives misaligned data instead of the map transition header (0xC021A0), causing the CPU to execute invalid opcodes (0x00000000 / break) and permanently locking the CD-ROM DMA channel.
Assembly-Level Verification Requirement:
Run verification script against master image:
bash


python translation-tools/patch_type39_scripts.py --verify-clamping
Ensure len(recompressed_type39) <= 1588 bytes exact. Any trailing padding must consist of zero-slack NOPs (0x00) without expansion.
Confirm DMA Channel 3 status register 0x1F801088 clears bit 24 (CHCR.Busy == 0) upon map load completion before executing subsequent script threads.
