# Yamana HeartBeat Engine — Audio (SEQq) & Media (XA/FMV/MDEC) Pipeline Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V9 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `dq7study.md` (custom SEQq/qQES sound driver 60-byte header), `DW7_RE_Study_extracted.md`, Enix Suite PG5, `YAMANA_HBE_FRAME_LOOP_SCHEDULER_SPEC.md` (audio spindle entry 6).
**License:** CC BY-NC-SA 4.0
**Status:** Consolidated reference for the DQ4-side audio driver + XA/MDEC media pipeline; FMV line is largely INFERRED (stub era evidence) until DuckStation frames are captured.

---

## 1. Executive Abstract

HeartBeat's audio is a proprietary **`SEQq`/`qQES` sound driver** (carried into the Q41
build from the DW7 lineage; 60-byte header format measured on the DW7 `SLUS_012.06` side)
and its video is XA/FMV through the PSX MDEC. In the Sovereign native build, FMV/XA playback
is **stubbed at stable FPS** (V96: 59.82 FPS, GPU 0.00 — async VBlank/DMA event not
signaled), restoring to full XA/MDEC only `0x8008AEF4` / `0x8008CAD0` when the event lane is
fed (V98-era restore).

---

## 2. The SEQq Sound Driver (MEASURED on DW7; DQ4-side INFERRED-compatible)

| Property | Value |
|---|---|
| Driver header | 60 bytes (SEqq/qQES family) |
| Control blocks | `0x60010108` class WORDs |
| Dispatch | sound spindles via frame-loop registered task entry 6 (`0x8004ED10` Audio Spindle Stream Sync) |
| Channel | SPU DMA ch4 (`0x1F8010C0 / C4 / C8`) |

The driver streams sequencing commands to the SPU; timbre/sequence banks live inside the HBD
archive (Q41 class). Event-driven trigger: BIOS event consumption same `0xF2000002`
class as the messaging/frame bridge.

---

## 3. XA / FMV / MDEC Media Pipeline (V96-era MEASURED; modern lanes INFERRED)

```
XA/FMV seek (HBD/STR) → CD ROM ch3 → MDEC-in ch0 (0x1F801080/84/88)
   → MDEC decode → MDEC-out ch1 (0x1F801090/94/98) → VRAM
   → display at 59.82 FPS (meas.) with async VBlank/DMA signaling
```

| Item | Address / Value | Status |
|---|---|---|
| XA/STR playback entry | `0x8008AEF4` | stubbed → restored V98 |
| MDEC entry | `0x8008CAD0` | stubbed → restored V98 |
| Stable-FPS evidence | F59.82, GPU 0.00 (V96) | async event unsignaled |
| Event bridge | EvMdINTR `0xF2000002` | frame-loop §4 |

**Vergent rule:** keep FMV stubs byte-stable unless the event lane is proven wired
(C-5 live capture); un-signaled async events are the recorded GPU-0.00 failure.

---

## 4. Integration & Arbitration

- Audio spindles must not run during card DMA (memory-card bus safety, messaging-VM §7).
- Media DMA (MDEC-in/out) contends on the same bus as dialogue DMA3; the deferred-VRAM
  arbitration `0x80025D10` defers blits while any CD DMA is active.
- SPU ch4 is the only audio DMA channel; SPU is mastered relative to event flag 0xF2000002.

## 5. Watchdogs

| # | Assertion |
|---|---|
| S-1 | SEQq header present and 60-byte intact per bank |
| S-2 | FPS ≥ 55 and GPU > 0 during any media-capable build |
| S-3 | no card/SPU DMA overlap (card-bus safe window) |
| S-4 | XA/MDEC entry integrity: `0x8008AEF4` / `0x8008CAD0` verifiable |

## 6. Open Items

| # | Item |
|---|---|
| M-1 | DQ4-side SEQq header + bank census (mirror the DW7 measurement) |
| M-2 | Live DuckStation frame capture of a media transition (FMV lane) |
| M-3 | C-5 event-flag capture proving the restored MDEC signaling |

---

*Rev 1.0 end. Companion: `dq7study.md` / `DW7_RE_Study_extracted.md` (SEQq spec heritage) ·
Frame-loop spec (audio spindle entry 6).*