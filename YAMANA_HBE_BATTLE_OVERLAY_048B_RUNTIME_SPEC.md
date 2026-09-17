# Yamana HeartBeat Engine — Battle Overlay `0x048B` Runtime Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V8 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_HBE_MASTER_TID_LIBRARY.md` (§5.3), `YAMANA_HBE_MASTER_SID_LIBRARY.md` (critical-SID registry `048B:0102`…), `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (§3, algorithmic RAW cells), `YAMANA_NAKAMURA_HEARTBEAT_ENGINE_GENERATIONAL_ARCHITECTURE.md` (§7).
**License:** CC BY-NC-SA 4.0
**Status:** Run-time consolidation for the engine's largest single overlay; freeze-hazard contract.

---

## 1. Executive Abstract

Block `0x048B` is the **battle engine overlay** — 817 sequences of battle dialogue/flow
driving the DQ4 command loop, delivered as an algorithmic raw cell (dqlzs/RAW class) that
must mount and run without recompression. It is the highest-consequence overlay in the
engine: 81… of the battle strings are delta-locked / budget-critical, and it carries the
principal freeze hazard from the "Runaway Decode Freeze" class.

---

## 2. Identity (MEASURED)

| Property | Value |
|---|---|
| Block ID (TID) | `0x048B` |
| Sequences | 817 |
| Class | RAW cell / dqlzs algorithmic content |
| Spartan SID set | `048B:0102` family (critical-SID registry) |
| Dispatch | loaded via overlay mount → battle loop (registered task / frame lane) |

## 3. Mount & Lifecycle (MEASURED + INFERRED)

1. Battle start → HBD open `0x048B` → raw-cell unpack (`0x8008F9A0` family) into battle
   overlay residency.
2. Sequence machine walks 817 entries; each sequence addresses battle text SIDs in the
   `048B` SID space (Font-plane rendering applies: § font spec).
3. On victory/run/escape → overlay evicted; framing returns to the world lane. Evict must
   defer while DMA3 active (VBlank arbitration, frame-loop spec §5).

### 3.1 Freeze contract

`0x048B` carries the canonical **Runaway Decode Freeze** witness when a re-authored
sequence breaks the `≤+3` overrun contract (codec spec K3) or drops a `{0000}` terminator
(messaging-VM K4). Verdict: keep `0x048B` byte-budget verified; never re-encode its rows to
a different delta (battle tactics garble `EずンてるタELL` = split-shift desync class).

## 4. SID & String Budgets Within the Overlay (MEASURED)

- Critical SID registry pins `048B` string IDs such as `048B:0102` with explicit per-string
  budgets — a patcher must respect the uncompressed-length ceiling per sequence.
- 817 sequences × variable length; the census in `YAMANA_HBE_MASTER_TID_LIBRARY.md` §5.3
  lists the overlay's structural map.

## 5. Integration Points (INFERRED, consistent with frame-loop spec)

| Event | Task/Handler |
|---|---|
| Battle enqueued | entity/spatial dispatch (frame loop entries 1/2) |
| Battle text render | messaging VM → font latch → GPU draw-list |
| Battle stream need | DMA3 `0x8008F810` ring (if streamed) |
| Conflict | overlay vs ring base `0x80011F00` occupancy (overlay spec §3) |

## 6. Watchdogs

| # | Assertion |
|---|---|
| B-1 | `0x048B` overlay mount completes before first battle frame |
| B-2 | sequence 817 termination intact post-patch |
| B-3 | no `{0000}`-less decode session (freeze watchdog) |
| B-4 | EDC/ECC valid across every modified Form-1 battle sector |

## 7. Open Items

| # | Item |
|---|---|
| B-5 | Sequence-address table extraction (817 entries → SID) |
| B-6 | Battle-lane dispatch ordering capture (C-1 live lane) |

---

*Rev 1.0 end. Companion: `YAMANA_HBE_MASTER_TID_LIBRARY.md` §5.3 · `YAMANA_HBE_MASTER_SID_LIBRARY.md` **critical-SID registry**.*