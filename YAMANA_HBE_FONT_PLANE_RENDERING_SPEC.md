# Yamana HeartBeat Engine — Font-Plane & Rendering Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V4 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_HBE_MASTER_SID_LIBRARY.md` (§5 hardware RE constraints, delta-locks), `REGRESSION_AUDIT_Font1_StartMenu_GlyphSubstitution_Sep8_2026.md`, `REGRESSION_REPORT_Font1_RAM_CORRUPTION_Sep8_2026.md`, `YAMANA_HBE_MESSAGING_VM_CONTROL_CODE_DISPATCH_SPEC.md` (§6), `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (§1.1 font lockstep, §6 delta-locks).
**License:** CC BY-NC-SA 4.0
**Status:** Consolidated rendering/typography reference.

---

## 1. Executive Abstract

HBE renders dialog with **two fixed VRAM font planes** under a single page latch — there is
no dynamic texture allocation:

| Plane | Grid | Content |
|---|---|---|
| **Font 1** | 8×14 half-width | Proportional ASCII/Latin glyphs |
| **Font 2** | 16×16 proportional | Shift-JIS kanji/kana |

Glyph-plane selection is driven by the character stream itself. The renderer
(`0x8008F3BC`) decodes Shift-JIS lead bytes onto the kanji/kana plane and single-byte ASCII
onto the Latin plane; control tokens in the `{7f..}` family (`{7f0b}` in particular) act as
**plane-switch commands**. Missing switch = the renderer decodes ASCII byte-pairs as
Shift-JIS leads → font corruption + texture-cache collisions.

---

## 2. Fixed Residency & Coordinates (MEASURED)

| Object | Address / Value | Notes |
|---|---|---|
| Font 1 block | `0x048C` @ file `0x97AC8` | 8×14 half-width, delta-locks 44/36/50/66 |
| Font 1 SIDs | 773–778 | delta-locked glyph rows |
| ASCII font table (primary) | `0x800A9FA0` | Latin plane |
| ASCII font table (secondary) | `0x80019CE4` | Latin plane (menu/alt) |
| Heading resolver | `0x8008F280` | TID→font-page latch |
| Glyph-plane renderer | `0x8008F3BC` | lead-byte decode + plane latch |
| Message work buffer | `0x800F4DF0` | streaming text target |

---

## 3. Plane-Latch Rules (§ `0x8008F3BC` — MEASURED)

```
Shift-JIS lead bytes (0x81–0x9F, 0xE0–0xFC, ≥0xFE)  → latch 16×16 kanji/kana plane
Single-byte ASCII (0x01–0x0F, 0x11–0xFD)            → Latin plane
Control tokens {7f..}, esp. {7f0b}                  → plane-switch command
```

The engine treats kana lead-ranges and the ASCII page as **mutually exclusive current
planes**; a line that interleaves ASCII and Shift-JIS must re-latch on every boundary.

---

## 4. Font 1 Delta-Lock Invariant (MEASURED)

`0x048C` carries four delta-lock rows (44/36/50/66) over SIDs 773–778. The renderer treats
these rows as **fixed-width invariant** — byte-equal glyph advancement. Re-authored text
must preserve the same glyph widths or the delta-lock breaks:

- **Delta-lock violations** are the measured cause of "Font 1 RAM corruption" and start-menu
  glyph substitution classes (Sep-8 regression reports).
- The lock rows are byte-pinned; patchers must never re-encode `0x048C` with a different
  width set (enforced by G4 delta-lock gate in the build ladder).

### 4.1 The rendering ground rule

Font 1 may only substitute **glyphs of identical advance delta**. English text is written to
fit within the 8×14 cell; proportional layout is handled by the engine's kerning/draw-list,
not by altering cell geometry.

---

## 5. Draw-List / GPU Path (MEASURED + INFERRED)

1. VM walker emits glyph IDs + control tokens (§ messaging-VM spec §4).
2. Resolver `0x8008F280` maps SID→block font page; renderer `0x8008F3BC` decodes the lead
   byte and latches the plane.
3. Glyph draws are queued into the GPU display list; the box budget is a **fixed 224-unit
   message box** (Enix Suite PG5); growth beyond budget is a compile-time error.
4. Streaming lines wrap through the DMA3 ring `0x800E8000` and unblank via `0x800F84E4`.

---

## 6. Failure Classes (MEASURED)

| Class | Mechanism | Witness |
|---|---|---|
| **Font corruption on missing plane switch** | ASCII decoded as Shift-JIS leads | Font 1 RAM corruption / start-menu glyph substitution (Sep-8 reports) |
| **Split-shift desync** | referrer resumes `+6/+8/+19` bits late after control/terminator | `ＮＴ　ＷＲＭ` fragments; `EずンてるタELL` battle tactics garble |
| **Delta-lock break** | non-invariant width substitution in `0x048C` rows | glyph misaligned columns |
| **Texture-cache collision** | plane latch not toggled → same cache line serves wrong plane | corrupted glyph artifacts |

---

## 7. Authoring Constraints (INVARIANTS)

1. `{7f0b}` (plane-switch token) must partition every ASCII ⇄ Shift-JIS boundary.
2. `{0000}` terminates every line; absent terminator → runaway decode.
3. Font 1 rows 44/36/50/66 (SIDs 773–778) must retain byte-equal advance — **never
   re-encoded with altered widths**.
4. Fixed box budget: no glyph insertion may grow the 224-unit box (branch-cut or split).

---

## 8. Open Items

| # | Item | Decides |
|---|---|---|
| F-1 | Full per-glyph advance table extraction for Font 1 rows | proportional-fit proof |
| F-2 | Kerning/tracking table address (INITIAL INFERRED in draw-list) | precise Latin layout |

---

*Rev 1.0 end. Companion: `YAMANA_HBE_MASTER_SID_LIBRARY.md` §5 · Sep-8 Font1 regression
reports · Enix Suite PG5.*