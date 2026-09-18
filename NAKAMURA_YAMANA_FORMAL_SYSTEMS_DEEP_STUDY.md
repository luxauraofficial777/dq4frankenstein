# FROM 65816 TO MIPS
## The Architecture and Technical Lineage of Nakamura & Yamana's JRPG Engines

**Doc ID:** VW-DEEP-WP-001 · Rev A · 2026-09-18
**Series:** Formal Systems & White Paper · Enix Engine Specification Suite companion
**Scope:** Dragon Quest I–VII (FC/NES 1986–1992, SFC 1992–1996, PS1 2000–2001), the five mandated
topics: (1) host workstations & toolchains; (2) proprietary scripting VMs / zero-copy execution;
(3) memory pagination & bank-switching; (4) compression; (5) the 65816→MIPS architectural lineage.
**Consumers:** the Sovereign (DQ4 PSX) re-authoring project, future cross-era asset pipelines,
and the public record of Chunsoft/HeartBeat engineering practice.

---

## 0. Evidence tiers and sources

Every claim below is labelled by its strength. Reproducibility matters: nothing in this paper
is asserted from memory.

| Tier | Meaning | Instrument |
|---|---|---|
| **[PRESS]** | recorded in first-party/press interviews translated from Japanese | 1UP Nakamura (2021); Famitsu DQ4 (1989); Nemoto "serious game" interviews (1987); shmuplations Chunsoft 30th, DQ3, DQ6, Horii 2000; DQ6 children's book (Feb 1995); Heartbeat/atwiki genealogy |
| **[HARDWARE]** | platform datasheet / documented memory map | MOS 6502/Ricoh 2A03; WDC 65C816; MIPS R3000A + GPU/CD-RD/DMA registers; Sony PlayStation docs |
| **[MEASURED]** | byte-level measurement reproducible from in-tree artifacts | PG1–PG5 forensic scripts, `hbd_forensics.py`, `gapfill_forensics.py`, three Enix Engine monographs, disc audit `DISC_AUDIT_DATA.json` |
| **[DECODED]** | data decoded through a verified codec (Huffman/LZSS parsers) | `translation-tools\hbe\`, dq4psxtrans decoder stack |
| **[INFERRED]** | engineering extrapolation from the above, flagged | — |

**Working title** selected from the client's candidate list: *"From 65816 to MIPS"*. The
remaining candidates — *The Black Box Open (1990–2001), Deterministic Bytecode & Bare Metal,
Systems Archaeology & Research, Turning On the Lights, Silicon Sockets and Script, Ghost in
the ROM, Punchy & Publication-Ready, The Nakamura-Yamana Pipeline, Architecture Over
Abstraction, Zero-Copy Zen* — are retained as chapter/epigraph options for derivative papers.

---

## 1. Host workstations, toolchains, and the absent ICE

### 1.1 The computational autobiography [PRESS]

Nakamura's origin is a *cross-compilation origin* — a teenager who taught himself the hardware
through a photocopied English manual:

- PC-8001 (NEC, Z80) first home machine, chosen over the Sharp MZ-80 because the NEC ran color
  and hosted the admired "Geimu Kyoujin" Space-Invaders clone. Programming began in BASIC,
  then assembly, from a **copied foreign-language (English) manual** — the machine's 40×20-char
  CRT forced monochrome, low-density development.
- No printer at Chunsoft: **line-by-line debugging by eye**; no debugger, no ICE, on any target.
- PC-8001 → PC-8801 → PC-9801 (the NEC V30/i86 line). The PC-98 "just released" cost
  ~¥300,000 in the Door Door contest year — out of reach for a student, hence the Enix
  contest gambit (¥1,000,000 prize, 1983).
- DQ1 (1986): Nakamura programmed **everything except music** into **64 KB**. His verbatim
  constraint story — "we had to get rid of more than half of the katakana, then re-write the
  names of spells and towns with the remaining characters" — is the canonical datum for the
  era's symbol-budget physics.
- DQ2 (1987): first team, 3–4 programmers; 50% Nakamura. DQ3 (1988): ~10%. DQ4/DQ5: no
  hands-on programming — designer/director role only, team composition problem documented
  ("no one knew whose bug it was; unfriendly atmosphere").

Conclusion **[INFERRED]**: the Famicom code was **cross-assembled on NEC-family machines**
(6502 gas for the 2A03), mapped to PRG banks, and shipped to mask ROM via EPROM stages.
No hardware ICE/emulator ever existed in the Chunsoft toolchain; the "debugger" was a human
reading a hex/listing against a spec — which is why the engine punishes nothing more than
layout surprises, because the engineers optimized for zip-verifiable byte layouts.

### 1.2 From floppy to cartridge — the pipeline [PRESS + MEASURED]

Map-art for DQ6 was delivered on **floppy disks** from ArtePiazza (Majima) to HeartBeat
[PRESS]; Sugiyama's orchestral scores were converted to **computer notation** (data tables,
not PCM) [PRESS]; the scenario text was written **on paper by Horii and keyed in by the
programmers**, every generation including DQ7 ("scenario manuscript 16,000 pages") [PRESS].
The DQ6 children's book (Feb 1995) calls the 29-year-old Yamana the programming
"conductor", working in assembly checked against a "computer language dictionary" —
the assembler reference. This is the industrial pattern: **paper → key-entry → cross-assemble →
patch → EPROM/ROM**, with the text pipeline fully owned by the same team that owned the codecs.

### 1.3 Yamana and the team-conductor model [PRESS]

Manabu Yamana entered Chunsoft during university; DQ3 (program), DQ4 **chief programmer of
the AI battle system**, DQ5 director; founded HeartBeat (Oct 1992, rep. Yamana) which
developed DQ6, SFC DQ3, PS1 DQ4, and DQ7 (dissolved Feb 1 2002; the lineage surveys into
Genius Sonority, Aug 2002 — Pokémon Colosseum — a *different* branch, claimed only as
continuator of personnel, not engine). DQ7 moved from N64DD to PS on **Jan 15, 1997**;
Horio directly attests the PS constraint: **"it can only read about 2MB worth of data at a
time"** — the exact quote the streaming architecture in §3.3 implements.

### 1.4 The PS1 toolchain surface [MEASURED + HARDWARE]

The DQ4 PSX EXE carries standard PlayStation toolchain vectors: fixed entry `0x8008DAC0`,
a BSS-cleanse initializer at `0x8008E284`, DMA-channel write sequence `0x1F8010xx`, and a
sound-thread core mapped at `0x8002FA10–0x80036FCC` (registrations at `0x80025300–0x800258E8`).
Our rebuild pipeline validates the binary against the SDK-era library surface
(`psyq 11/13 pass: 2 warn, 0 fail`) — i.e. the linkage fingerprint of the SN Systems
PsyQ-era SDK, though the historical toolchain's exact product name is **not** recoverable
from the binary **[INFERRED, honest gap]**.

The hardware the toolchain had to feed — **[HARDWARE]**: R3000A @ 33.8688 MHz, **2 MB DRAM**
(first 64 KB OS-resident; ~1.9 MB user), 1 MB VRAM, 512 KB SPU, 512 KB BIOS, 1 KB scratchpad
at `0x1F800000`, Kseg0 (cacheable `0x8000_0000`) / Kseg1 (uncached `0xA000_0000`) /
Kuseg; 7 DMA channels (GPU 0/2, MDEC 1, SPU 3? per PSX map: CD ch3, SPU ch4, OTC ch6 —
see §3.3), double/double-speed CD at 150/300 KB/s with 2,048-byte sectors.

---

## 2. Proprietary scripting VMs: zero-copy execution under RAM starvation

### 2.1 G1 (FC) — no VM: addresses-as-data [MEASURED]

The 8-bit engine's "scripting" is direct addressing. A 5-bit MMC1 write selects a bank; a
16-bit offset walks it; **bank-up-on-wrap is a protocol, not a pointer**. Dialogue, maps, and
monsters are raw tables: 2,724 lines across 86 blocks × 32 lines, monster records as fixed
22-byte strides with 10-bit packed stats; **instruction-built maps** (turtle-graphics
push/pop streams, shared across DW2/3/4: 73 maps / 35 tilesets). AI is not bytecode — it is
action-slot *ordering* plus an opening-state byte. Zero-copy is achieved by *having no copy*:
the decode path is a single pass from ROM bitstream to the VRAM text buffer.

### 2.2 G3 (SFC) — 65816 inline-operand dispatch with return-address rewriting [MEASURED]

The HeartBeat event VM at `$C4:28FE` is the lineage's pivot: **code calls `JSL` into an
event table whose handler smashes the 65816 return address to jump into the *next* inline
operand stream**, forming a dispatch loop with no intermediate bytecode buffer. Six-entry
handler table `$C4:2970`; a WRAM parameter FIFO at `$7E40B5–A`; catalogue `$C42AD2`
(show-dialogue) → `$C439C0` (scripted battle). This is **zero-copy by construction**: the
event stream executes in place, in ROM, with the machine's own return stack doubling as the
VM's dispatch counter.

The dialogue primitive is equally in-place: a **65816 `BRK` co-opted with a 2-byte operand =
message ID** (tier-2, documented + live) — "every literal BRK #$XXXX in the ROM is an
message-ID pointer". The engine walks a global Huffman tree for travel dialogue and a raw
small-font page for battle text (base `$F6DEBD`); decoded glyphs stage through a WRAM font
build at `$7E9585` (0x700 bytes) then DMA to VRAM `$4900` — the only "copy" in the whole
render path, and it is a DMA path, not a CPU copy.

### 2.3 G4 (PS1) — `C0 21 A0` byte-aligned dialog VM + message-window lifecycle [MEASURED]

Primary program controller = byte-aligned **3-byte command**:

```
C021A0 <offset:2> <dialogId:2>   load dialogue stream at offset/dialogId
C021A0 <FFF0> <key>              evaluate event flags / system variables
```

`%A{ID}%X…%Z %B{ID}%X…%Z` leader-ID branches (Alena ID 120 discount duplicate assets);
`%H{var}%X%Ys%Z` pluralization. Controls `0000` (close), `7F01/7F02` (newline EXE vs scene),
`7F04` name decorator, `7F0A/0B` cursor-wait, `7F15` dynamic number (gold), `7Exx` per-block
dictionary refs (158; block-local). Run-time = the three monographs' measured mechanics:
registered-task dispatcher table `0x800AE5F8` (8×`{active,handler}`, indirect dispatch, zero
static callers), message directory ring base `0x800F5108` (894 × 4-byte self-chaining nodes,
"empty" = `next=(addr−4)&0xFFFFFF`), dispatch-context counter `[0x800FE578]` (+25 per
transition), unblank latch `0x800F84E4`, mirror tables `0x800C22B8`/`0x800C2448`.
**Zero-copy:** the CD drains a 2,048-byte sector into the heap (DMA ch3, bypassing the CPU
bus), the interpreter walks the bitstream in place in that heap — a copy exists only at the
bus level, never at the application level.

### 2.4 Why zero-copy is the invariant

All three eras share one physics: **RAM is the absolute constraint** (2 KB FC WRAM + 8 KB
battery, 128 KB/256 KB SFC, 2 MB PS). An engine that buffered script bytecode or dialogue
streams before interpretation would spend its only currency — bytes — on staging. The
through-line: *decode-into-place, dispatch in place, DMA out*. The three monographs document
this as the DQ4 PSX tenet and the reason the Sovereign re-authoring must accept a
no-clip, decode-in-place contract (224-unit message box, proportional font 2, **no width
check**, 15-unit prefix, 8-char name budget).

---

## 3. Memory pagination & bank-switching

### 3.1 FC — MMC1, 512 KB PRG, 5-bit banks, zero CHR-ROM [MEASURED]

DW4 NES: MMC1 mapper per iNES header; **512 KB PRG (32×16 KB banks)**, CHR-RAM only (no
CHR-ROM bank present) — the tile data is streamed from PRG into CHR-RAM at runtime.
2 KB WRAM + 8 KB battery-backed SRAM (save blocks *and* hot working sets). PG1's block
pointer table at `0x58961` (bank-22 `$8951`) with 16-bit `offset + $8000` bank-up-on-wrap
resolution **is** the paging primitive: the loader serializes a 5-bit mapper write and
increments the bank on address wrap, all without any indirection layer.

### 3.2 SFC — LoROM vs HiROM, windowed 24-bit tables [MEASURED]

G2 (DQ1+2, Chunsoft 65C816 SlowROM/LoROM) keeps the direct-table model at 16-bit cells
(item-price table `0x4EA5F` × 87; shared 122-record monster table `0x5DA0E`; zone tables
`0x5B52D`/`0x5BEC5`) — **no compression, no indirection**. G3 (HeartBeat DQ6/DQ3,
FastROM/HiROM and ExHiROM variants) replaces wrap-protocol with **windowed 24-bit pointers**:
all trees live in bank `$C1` (DQ6 node arrays at `$C1:67BE`/`$C1:700E`), script bases in
`$FC:C258`, per-resource 3-byte bit-addressed tables (`$C1:5331`, 506 × u24 = 21-bit byte
offset | 3-bit bit offset). The bank is no longer a protocol — it is a *field of the
address*, and tables know their home bank. Map geometry spreads across `$DD` (raw 1 KB blocks,
Shannon H = 4.06–4.65 bits/byte) with one compressed `$E8` isolate (H = 7.18); monster
frames live in `$F0–$F5` far-pointer tables.

### 3.3 PS1 — Kseg0/Kseg1, 2 MB DRAM, scratchpad, DMA, CD streaming [MEASURED + HARDWARE]

- **[HARDWARE]** 2 MB DRAM mirrored at Kuseg (`0x0000_0000`), Kseg0 cacheable
  (`0x8000_0000`), Kseg1 uncached (`0xA000_0000`); 1 KB **scratchpad** at `0x1F800000` —
  no TLB games needed (PS1 has a 64-entry TLB, but the engine never needs it: everything
  fits in the linear 2 MB).
- **[MEASURED]** DQ4 PSX main loop = *poll input → execute script (type-39) → update actor
  state machines → camera rotation (GTE) → sort OTs → push DMA buffer to VRAM*. Streams:
  per-sector 2 KB loads via `open_file 0x80076040` / `mopen_file 0x80082598` over **DMA Ch3
  (CD-ROM, `0x1F8010A0/A4/A8`)** into a 2 MB heap at `0x80138000`; the monographs' DMA spec
  verifies ch2 GPU (0xA0–A8… `0x1F801080` base), ch3 CD bootstrap with
  `BCR=0x00040000 CHCR=0xA1000010` (poll bit 24), ch4 SPU (`0xC0/C4/C8`), ch6 OTC
  (`0xF0/F4/F8`). The event-flag matrix occupies `0x8000F800–0x80010000`.
- **[HARDWARE + PRESS]** Horii's "only ~2MB at a time" is exactly this paging discipline:
  the archive (319,436,800 B / 155,975 sectors Q41) is served as a **sector-chain of typed
  entries** — the CD *is* the bank-switched memory, and a seek costs 300 ms single/1.3 s long.
  Consecutive-sector layout + typed sub-blocks (`HBD1PS1D.Q41`: 16-B header, 16-B file
  headers `size/size_uncompressed/unknown/flags/type`) is the optimizer's answer.

---

## 4. Compression systems

### 4.1 The Huffman bloodline [MEASURED, DECODED]

| Generation | Game | Tree | Bit order | Node test | Strings/ptr | Verified |
|---|---|---|---|---|---|---|
| G1 | DW4 NES (US) | per-block bitstreams, **3–18-bit prefix codes**, pair-dict byte mode (bank 22 `$B3A4`, `CMP #$80 BCS`) | **MSB-first** | prefix-greedy | 32 lines/block (86 blocks) | decoded |
| G1 | DW1/DW2 FC | 5-bit stream escapes (`$6D+`, `$6D+` families) | — | — | — | documented table |
| G2 | DQ1+2 SFC | **none** (plain bytes + `0xD0–0xD2` 768-entry kanji escape) | — | — | — | negative result, confirmed |
| G3 | DQ3 | 2×1002-entry u16-LE arrays `$C1:59D3/$C1:61A7`, root `0x3E9` | **MSB-first**; bit=1→`$161A7` | MSB-set=inner | **8/ptr** (SID=TID×8+slot) | byte-verified worked example |
| G3 | DQ6 | 2×1064-entry arrays `$C1:67BE/$C1:700E`, root `0x427` | **LSB-first**; polarity inverted vs DQ5 | MSB-set=inner | 8/ptr | tree measured (1064 nodes) |
| G4 | DQ4 PSX | dual-base per-block tree (`pair[NN]`/`pair[m+NN]`), `tree_len=4N+6` | per-block LSB-first | `(hByte&0xF0)==0x80` inner; `0x7F/7E` control | per-block referrer words `(id<<20)\|(bit+hts*8)` | hardware-proven |
| G4 | DQ7 | same family, global tree | per-block | `>=0x8000` inner | — | same engine family |

Invariants carried across the whole bloodline **[MEASURED]**:

1. Leaf ≥ `0x200` = glyph; < `0x200` = control (G3/G4 identical threshold).
2. Terminator `0x00AC/00AE` → `0x0000`; 8-strings-per-pointer + 3-byte `offset|bitoffset`
   pointer word survive G3→G4 until PS generalizes to 6-I headers + packed referrers.
3. Bit order is a **local property, not a family property** (DQ3 MSB vs DQ6 LSB) — the
   single most dangerous cross-engine hazard; same tree paradigm, re-tuned per game.
4. Per-block dictionary references (`7Exx`) are G1→G4 continuous: DW2 5-bit escapes
   (G1) → 158 per-block refs (G4).

### 4.2 DQ3 map loader — adaptive RAM dictionary [MEASURED]

Map decompression runs through an **adaptive 0x400-entry RAM dictionary at `$7F0000`**
(flag byte = 8×2-bit commands) at runtime — not in-stream LZSS framing (the `0x9385`
table's 15 targets census as raw 1 KB blocks + one compressed isolate; gap-fill Sep 14,
2026). This is a **stateful codec**: the dictionary grows during play, which is why a
saved session must carry dictionary state — a byte-packing behavior present again in G4's
per-script referrers and the reason the Sovereign rebuild must keep dictionary-reset
semantics ([MEASURED] flag-0x0500 LZSS files reset to zero ring each load).

### 4.3 LZSS → LZSSO [MEASURED]

G4's codec: **LZSSO** — LZSS family, 4,096-B zero-initialized ring buffer, max match 18,
threshold 3 (8/40 sampled files round-trip exactly to `size_uncompressed` this pass).
`flags==0x0500` → LZSS; `0` → raw. The direct 16-bit ancestor is the DQ3 `$C0:4923` stream
codec (same LZSS family, smaller window, no container header) [MEASURED] — this is the
unambiguous **65816-era codec that survived into the 32-bit engine unchanged in algorithm**.
Type-46 overlay modules (`LZS overlays`) package ≥20.2% of script pool strings; 612 modules
on Q41.

### 4.4 Dynamic font tile caching [MEASURED]

- NES: fonts are per-byte glyph streams; the entire font budget is a "curse-expensive"
  katakana economy ([PRESS] — half the katakana cut in DQ1).
- SFC DQ6: grouped variable-width font (5-byte metadata) built to WRAM `$7E9585` then DMA'd
  to VRAM `$4900`; furigana (`00C3`) is a DQ6 novelty, shipped through the same builder.
- PS1 DQ4: in-EXE dual-glyph atlases (Font 1/Font 2, fullwidth wall via `ori 0x8000`),
  type-1 font sheets loaded directly into VRAM to populate the text engine's character
  cache — glyphs are **tiles, and tiles are banked**; a text render is a tile-cache hit or a
  VRAM write, never a CPU blit of glyph pixels.

---

## 5. The 65816 → MIPS lineage

### 5.1 Five generations, five mechanisms **[MEASURED, PRESS]**

| # | Engine | CPU/RAM | Defining mechanism |
|---|---|---|---|
| G1 | Chunsoft 8-bit "table era" (DW1–4) | 6502-class, 2 KB + 8 KB battery, MMC1 | direct pointer tables, per-block Huffman, instruction-built maps |
| G2 | Chunsoft 16-bit transition (DQ5, DQ1+2 SFC) | 65C816 SlowROM, LoROM | raw byte text, flat pattern-dispatch AI, no compression |
| G3 | HeartBeat 16-bit "Huffman era" (DQ6, DQ3) | 65C816 FastROM, HiROM/ExHiROM | global 13-bit Huffman + per-resource streams + inline-operand event VM |
| G4 | HeartBeat 32-bit "archive era" (DQ7→DQ4 PSX) | MIPS R3000, 2 MB, CD | HBD1PS1D container, LZSSO, per-block Huffman + referrer words, `C0 21 A0` VM |
| G5 | DS sidestep (DQ4 DS) | Nitro (ARM7/ARM9) | NFTR fonts, .mpt message packs, full `%A/%B/%C/%H/%M/%O/%L/%D` grammar |

### 5.2 Patterns that survived the arch jump

- **The 16→32-bit dual-base tree**: two parallel arrays (SFC) → `pair[NN]/pair[m+NN]`
  (PSX) — the *same* node-table paradigm re-encoded in bigger cells; 8-strings-per-pointer +
  3-byte bit-addressed pointers (SFC) ⟶ per-block 6-I headers + `(id<<20)|bit` referrers
  (PSX) preserve the *share physics*.
- **AI weights as bit-fields**: FC-action-slot ordering → SFC pattern basis (`$0B:BD8C`
  jump table) → PSX `0x8000F800–0x80010000` event-flag matrix — the same weights at 8/16/32
  bits; the DQ4 PSX roster tables (type-44/type-26) still carry the FC-era 10-bit packed
  stat/resistance nibbles **[MEASURED]**.
- **Inline-operand dispatch**: 65816 `JSL` + inline bytes + return-address rewrite (§2.2)
  generalizes directly into the byte-aligned 3-byte `C0 21 A0 <offset> <dialogId>` dialog
  VM on MIPS (§2.3). Same trick, wider machine — the MIPS `lui/ori`/`jalr`
  parameterization is exactly the 65816 `JSL`-with-inline-operands idiom.
- **Streaming as memory**: bank-protocol on MMC1; windowed tables on SFC; sector-chain +
  DMA on PSX. All three are the *same* answer to "RAM is too small": page the data, never
  copy it, DMA the render output.

### 5.3 What the DS side-step proves

The G5 Nitro engine (NFTR, .mpt, full conditional grammar) outsources what G1–G4 built
in-house **the moment platform containers become fast enough** (cart, dual ARM). The
`%A/%B/%H` PSX conditional family appears *complete* on DS — proving the PSX `%A/%B` pair
was a deliberate subset, not a limitation. The decade of in-house codecs ended not because
the team lost the skill, but because the constraint that justified them (2 KB → 2 MB, slow
media) finally lifted. HeartBeat dissolved Feb 1, 2002; the engine's last artifact is the
DQ4 PSX build of Nov 22, 2001 that this project is restoring.

---

## 6. Implications for restoration — and the honest gaps

For the **Sovereign (DQ4 PSX)** work, the lineage demands:

1. **No-clip decode-in-place** consumer contract (224-unit box, 8-char name, no width check)
   — violates only are re-authoring violations, not engine bugs ([MEASURED], monograph-spec).
2. **Dictionary-reset semantics** for every LZSSO/type-46 load — the codec state must be
   per-file, zero-ring, exactly as the pristine engine does ([MEASURED]).
3. Per-block trees and referrer physics are load-bearing: re-encoding touches *both* the
   tree (itself 4-B/symbol) and the owning file's `tree_end/hte/end` window — the rebuilt
   archives must not drift the block-index math ([MEASURED]).
4. The message-directory ring (894 nodes) and registered-task table are the single runtime
   point of failure for any injected text: a wipe-as-repair PID (`0x80031E64`) exists and
   the A1–A11 watchdogs in the Enix Engine Spec Suite gate the transition.

**Honest gaps [INFERRED, unverified]:** the specific commercial assemblers/compilers for the
FC/SFC eras are not recorded (only the NEC-PC line, assembly, and "computer language
dictionary" are attested); the PS1 toolchain's exact SDK edition is not recoverable from the
binary (link-surface validation only, `psyq 11/13`); Yamana's personal AI source (DQ4's
italic-era) is not preserved. All link to the same historical fact that *motivates* this
whole study: **the machines were the manuals** — a private toolchain, no ICE recorded, no
source released. The byte-level record in PG1–PG5 + monographs + this paper is the closest
thing to primary evidence the lineage has, and it is reproducible.

---

## Appendix A — artifact index

- `study/NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG1.md` — DW4 NES byte map, MMC1, Huffman 0x58961.
- `study/NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG2.md` — DQ1+2 SFC (raw text, 18-B monsters, save loci).
- `study/NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG3.md` — the five-generation lineage + control-code table + monster-table thread.
- `study/NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG4.md` (VW-SNES-WP-004) — DQ6 SFC deep dive (global Huffman, BRK-message ID, font builder).
- `study/NAKAMURA_YAMANA_WHITEPAPER_ENIX_ENGINE_LINEAGE_PG5.md` — DQ7/DQ4 PSX container, LZSSO, `C0 21 A0`, memory remap.
- `study/DISC_WIDE_COMPARATIVE_AUDIT_Sep16_2026.md` + `DISC_AUDIT_DATA.json` — 4-image forensic concordance.
- `docs/TASK_DISPATCHER_AND_SCHEDULER_REGISTRY_SPEC.md`, `docs/DMA_CHANNEL_TRANSFER_SPEC.md`,
  `docs/MESSAGE_DIRECTORY_LIFETIME_SPEC.md` — the three Enix Engine Specification monographs (run-time measured).
- `study/WHITEPAPER_FORENSICS_DATA.json`, `study/HBD_FORENSICS_DATA.json`, `study/GAPFILL_FORENSICS_DATA.json` — machine-readable probes.
- Discs measured: `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin`, `DW7D1\DW7D1.bin`; cartridges `famicom\`, `snes\`.

*Regenerate every measurement: `python hbd_forensics.py`, `python gapfill_forensics.py`,
`python whitepaper_forensics.py`, `cd DQLOSTTRANSLATION && python disc_audit.py`.
Suite complete — PG1–PG5, three monographs, one deep-study; all claims tier-labelled here.*