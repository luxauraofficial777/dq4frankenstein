# Yamana HeartBeat Engine — Messaging VM & Control-Code Dispatch Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V2 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_HBE_MASTER_SID_LIBRARY.md` (§3 taxonomy), `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md` (cross-engine map), `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (§5 scripting subsystem), Enix Suite `PG5`.
**License:** CC BY-NC-SA 4.0
**Status:** Consolidated reference — decoding runtime behavior, not authoring patches.

---

## 1. Executive Abstract

The HBE messaging VM is a **compiler for game dialog under hard byte budgets** executed as
hand-written MIPS: a decoded character stream is rasterized directly into the GPU draw-list
without an interpretive language runtime. The user-visible unit of dispatch is the **SID**
(String Identifier) — a 32-bit packed referrer — and the character stream embeds a
**51-code control vocabulary** (`{7Fxx}` runtime family, `{7Exx}` block-local dictionary
family, `{FExx}` facility band) that the renderer expands to glyphs, pauses, input gates,
variable insertions, and engine actions.

This spec formalizes that VM: the packed-referrer addressing model, the dispatch cycle, the
control-code taxonomy with measured counts, the format-conditionals (`%A/%B/%H`), the font
plane-switch contract, and the runtime failure modes that produced the "Alien Text Syndrome".

---

## 2. The Packed Referrer Model (SID / TID)

### 2.1 Formulation (MEASURED)

```
ReferrerWord = (BlockID ≪ 20) | (BitOffset + HeaderTreeSize × 8)
TID          = SID ≫ 20
BitOffset    = SID & 0x000FFFFF
```

- **BlockID (TID)** — the 12-bit container identity (1,111 unique TIDs censused; 3,243
  physical blocks / 23,828 functional sub-blocks across ~47 type rows).
- **BitOffset** — bit granularity because the Huffman stream is *not* byte-aligned at the
  referrer level: `HeaderTreeSize × 8` skips the serialized tree so the offset lands on the
  first code bit of the target line.

### 2.2 Census (MEASURED)

| Space | Count | Notes |
|---|---|---|
| SIDs on disc (archive) | 19,193 | across 1,108 blocks + 4 overlays + 403 Type-39 scripts |
| TIDs | 1,111 | Ch.1–5 narrative, party chat (8,052 SIDs), church/facilities, battle/menus |
| Multi-copy TIDs | 40 | require lock-step patching (coherence G4) |
| Control codes | 51 | 43 archive + 8 EXE (verification: RadMageIRL + Wilkens + on-disc census) |

The dual-name 12-bit TID / 32-bit SID hierarchy is the *generation* bridge from the SFC
string VM: the SFC "character stream + control embedded in stream" becomes the PSX
"character stream addressed by a compound handle, decompressed through an engine-wide
Huffman tree, dispatched through a 51-opcode control VM."

---

## 3. Dispatch Cycle (MEASURED + INFERRED)

```
SID (packed referrer)  →  TID lookup → container open (HBD index)
                       →  BlockID fixes the Huffman tree (block-header tree or global meta-tree)
                       →  BitOffset + HTS×8 seeks into the serialized codeword stream
                       →  Huffman decode → character stream
                       →  VM walker (C021A0 family): consume codewords → glyph draw / control action
                       →  {0000} terminal → dialog closes, box advances
```

Fixed-residency working globals (no `malloc`):

| Object | Address | Role |
|---|---|---|
| Work buffer | `0x800F4DF0` | decoded line being rendered |
| Roster | `0x800F83C0` | roster / party-chat context |
| Message-directory headers | `0x800F51EC..0x800F5208` | live box pointers |
| Unblank latch | `0x800F84E4` | render gate (1 = unblank) |
| DMA3 ring | `0x800E8000` | streaming sector funnel |

The stream may arrive **mid-DMA**: `dma3_sync_stream_decompress` (@ `0x8008F810`)
synchronizes on INT3, invalidates the D-cache (`0x80018C20`), decompresses from the
uncached KSEG1 alias (`0xA00E8008`), and only then unblanks via `0x800F84E4`.

---

## 4. The 51-Code Control Vocabulary

### 4.1 Families (MEASURED)

| Family | Meaning | Class |
|---|---|---|
| `{7Fxx}` | runtime engine controls | 43 archive codes |
| `{7Exx}` | block-local dictionary references (lexical reuse within a container) | archive |
| `{FExx}` | facility/jump band (menu, saves, system services) | archive/EXE |
| `{00..}` / `{0000}` | stream terminator (mandatory per line) | universal |
| `%A` / `%B` / `%H` | format string conditionals (inline variables) | renderer |

### 4.2 The taxonomy (MEASURED — canonical ordering)

Per `YAMANA_HBE_MASTER_SID_LIBRARY.md` §3.1, the canonical 51-code reference enumerates each
code with its opcode, semantic, and measured occurrence count (per-code counts verified, e.g.
`{7F02}` newline ≈ 166k). Codes in the `7F` family control:

- text flow (newline, page advance, pause),
- face/color/glyph-plane selection,
- input gates (wait-for-tap, wait-for-flag),
- variable/name insertion,
- end-action (close box, advance script).

`7E` codes are **block-local dictionary refs**: they re-emit a string previously defined in
the same container, saving bytes in repeat-heavy dialogue (church/facility text, monster
roster). `FE` codes route into the facility band used by menus and system services.

### 4.3 Verification ledger (MEASURED)

- **RadMageIRL (hardware measurements)** — confirms code semantics on silicon.
- **Wilkens tooling** — cross-tool decode agreement on sample containers.
- **On-disc census** — 51 codes present (43 archive + 8 EXE family split: 7-vs-8 EXE split
  discrepancy between SID and CONTROL_CODES libraries is recorded in TOC Appendix B, total
  agreeing at 51).

Known internal do-not-rely items (see TOC Appendix B): per-code counts differ between
libraries for `{7F02}` (166,235 vs 166,134) — census-granularity drift, not semantic.

---

## 5. Format Conditionals (`%A/%B/%H` and friends)

The renderer parses `%`-introduced conditionals inline (documented in
`DQ4_PSX_Engine_Analysis_extracted.md`, conditionals with Alena party ID 120 as a reference
use case). These select **which literal substitutes** at draw time based on party state /
speaker context — the mechanism behind name and gendered-line insertion.

### 5.1 Runtime shaping contract (INFERRED — consistent with draw-list evidence)

1. Conditional groups are evaluated left-to-right at the time the line is rasterized.
2. Non-matching branches are skipped without emitting glyph draws (no phantom advance).
3. A conditional that cannot resolve must not change the box size budget (fixed box model:
   224-unit message-box model per Enix Suite PG5).

---

## 6. Font Plane-Switch Contract (MEASURED)

Referrer→renderer path: `0x8008F280` (heading resolver), `0x8008F3BC` (glyph-plane
decode/latch).

```
TID = SID ≫ 20 ; BitOffset = SID & 0x000FFFFF
0x8008F3BC decodes glyph planes:
  - Shift-JIS lead bytes 0x81–0x9F, 0xE0–0xFC, ≥0xFE   → latch 16×16 kanji/kana plane
  - single-byte ASCII 0x01–0x0F, 0x11–0xFD              → Latin plane
  - control tokens {7fxx} (esp. {7f0b})                  → plane-switch command
```

- ASCII font tables: Primary `0x800A9FA0`, Secondary `0x80019CE4`.
- **Omitting a plane switch** forces the renderer to decode ASCII byte-pairs as Shift-JIS
  leads → font corruption + texture-cache collisions. This is the measured root of the
  "Alien Text Syndrome" (`peynriohre-seale` fragments) and the Runaway Decode Freeze class.

### 6.1 Desync classes (MEASURED)

| Class | Mechanism | Witness |
|---|---|---|
| Split-shift kana/English desync | referrer lands mid-glyph; renderer resumes `+6/+8/+19` bits late after a control/terminator | `ＮＴ　ＷＲＭ`-class fragments; battle tactics garble `EずンてるタELL` |

The `+6/+8/+19` bit resumption offsets are the measured decoder behavior after
control-token emission — a hard constraint for any clean-room re-author.

---

## 7. Dynamic Variable Expansion Delimiters (MEASURED)

Per SID library §5.2: the VM supports engine variable insertion delimited in the stream.
These tokens are resolved against engine state at draw time (names, gold counts, party
composition). They are distinct from `%`-conditionals (branching) in that they are
**substitution-only** and never alter control flow.

Memory-card bus safety (SID 54 & block `0x0474`): save/load dialogue performs its card
accesses during the draw-safe window so the VM never renders while the card DMA slot is
busy (freeze-avoidance constraint inherited from the SFC generation).

---

## 8. Failure Modes & Guard Contracts

1. **Mandatory `{0000}` terminator** — every re-authored line must terminate; absence
   drives runaway decode (measured freeze class).
2. **20-bit per-block ceiling (128 KB)** — HBE limit; re-encodes must respect the per-block
   budget and the zero-sector-shift invariant.
3. **`≤ +3` decoded-byte overrun contract** — decompressor generators may overrun by at most
   3 bytes; enforced by every generator in the pipeline (Rebuild C standard).
4. **No blank-byte budget growth** — fixed 224-unit box; insertions must fit or be branch-cut.

---

## 9. Pointer Index

| Symbol | Address |
|---|---|
| Message work buffer | `0x800F4DF0` |
| Roster | `0x800F83C0` |
| Directory headers | `0x800F51EC..0x800F5208` |
| Unblank latch | `0x800F84E4` |
| Heading resolver | `0x8008F280` |
| Glyph-plane renderer | `0x8008F3BC` |
| ASCII font (primary) | `0x800A9FA0` |
| ASCII font (secondary) | `0x80019CE4` |
| Streaming decompress sync | `0x8008F810` |
| D-cache invalidate | `0x80018C20` |
| DMA3 ring | `0x800E8000` |
| VM walker family | `C021A0` (dialogue VM) |

---

## 10. Open Items (label: PENDING)

| # | Item | Decides |
|---|---|---|
| V-1 | Exact runtime dispatch stride for the 51-opcode table (INFERRED as `$Base+(ID×Stride)`) | byte-exact re-implementation |
| V-2 | Conditional-resolution precedence across `%A/%B/%H` stacks | branch correctness |
| V-3 | `7E` dictionary reference lifecycle across warp re-seeds | re-use safety |

---

*Rev 1.0 end. Companion monographs: `YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md` ·
`YAMANA_HBE_MASTER_SID_LIBRARY.md` · Enix Engine Specification Suite `PG5` (PSX HeartBeat archive era).*