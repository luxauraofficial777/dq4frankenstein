# Yamana HeartBeat Engine — Frame Loop & Task Scheduler Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V7 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_HBE_HBD_DMA_SUBBLOC_ROUTINE.md` (§4 task registry), `docs/TASK_DISPATCHER_AND_SCHEDULER_REGISTRY_SPEC.md`, `docs/DMA_CHANNEL_TRANSFER_SPEC.md` (§1.3 IRQ→event bridge), Enix Suite PG5.
**License:** CC BY-NC-SA 4.0
**Status:** Consolidated execution model — VBlank/IRQ→event→task→frame choreography. Hander *behavior* detail is INFERRED until C-1 (live capture) completes.

---

## 1. Executive Abstract

HBE runs a **VBlank-driven frame loop** with an IRQ→event→task bridge: hardware IRQs are
converted to software events (BIOS event flags) consumed by a **registered-task dispatcher**
over an 8-entry table, while DMA/CD streaming is synchronized against frame boundaries by
deferred VRAM-queue draining. The scheduler is the engine's heartbeat — the same cadence
that, in the modern project lineage, inspired the "HBE" streaming/pulse telemetry of the VW
tooling.

---

## 2. The Frame/Task Pipeline (MEASURED + INFERRED)

```
VBlank IRQ (I_STAT bit 0)
   │  # BIOS EmitEvent/SetEvent (event flag 0xF2000002, mode EvMdINTR 0x1000)
   ▼
WaitEvent returns → frame/task loop step
   │
   ▼
registered-task dispatcher (registry 0x800AE5F8 / mirror 0x800C2268)
   →  handler(entry) per enabled flag
   →  0x80031F0C builds/consumes the message ring (entry 4)
   →  0x80031DEC wipes/re-seeds the ring (entry 5)
   │
   ▼
VBlank deferred queue flush (0x80025D10): drains blit queue 0x800F9100
   while D3_CHCR bit 24 idle (GPU ch2 vs CD-ROM ch3 arbitration)
```

## 3. The Registered-Task Table (MEASURED)

`0x800AE5F8` — 8 × 8 B `{u32 active, u32 handler}`; mirror `0x800C2268` byte-identical.

| Entry | Handler | Duty |
|---|---|---|
| 0 | `0x80053AC0` | World Event State Evaluator |
| 1 | `0x8004F3C0` | Entity Collision Update |
| 2 | `0x8004F500` | Spatial Trigger Dispatcher |
| 3 | `0x80029900` | Camera Smoothing Interpolator |
| 4 | `0x80031F0C` | **Directory build/consume task** |
| 5 | `0x80031DEC` | **Directory reset/re-seed (wipe) task** |
| 6 | `0x8004ED10` | Audio Spindle Stream Sync |
| 7 | `0x80056B58` | VRAM Dynamic Buffer Commit |

Geometry: table window `0x800AE5F8..0x800AE638` (0x40 B); mirror window
`0x800C2268..0x800C22A8`. Mirror abuts the mirror-tables cluster `0x800C22B8` (A) /
`0x800C2448` (B).

**Entry semantics (INFERRED, consistent with zero static callers):**

```
if (flag(entry[i]) != 0)  handler(entry[i]);
```

The leading flag is an **enable gate, not completion state** — stays 1 across GOOD↔BAD.
The fault on the Auld Well warp is dispatch *selection*, not descheduling.

## 4. IRQ → Event → Task Bridge (MEASURED)

- CD/DMA completion IRQ (INT2/INT3 → `I_STAT` bit 2) → `EmitEvent/SetEvent` on event flag
  `0xF2000002` (EvMdINTR, mode `0x1000`) → `WaitEvent` → frame/task loop → dispatcher.
- `0x8008F810` (DMA3 streaming sync) polls `I_STAT` bit 2 with a 120-frame timeout, then
  asserts `[0x800F84E4]=1` on drain.
- **The bridge is software:** DMA hardware asserts only the IRQ; which task consumes the
  event is dispatch-gated (§3). Delivered-sectors flag + empty message ring = precisely the
  Auld Well state.

## 5. VBlank Queue Arbitration (MEASURED)

- Deferred VRAM blit queue at `0x800F9100`; `vblank_blit_queue_flush` (`0x80025D10`) runs
  during VBlank and **defers one frame** while `D3_CHCR` bit 24 (CD DMA active) is set —
  starving GPU FIFO is avoided.
- Camera interpolator `0x80029900` and VRAM commit `0x80056B58` are the frame-critical
  members; ordering among members is dispatch order (INFERRED until C-1).

## 6. Frame-Loop Invariants

| # | Invariant |
|---|---|
| L1 | A single dispatcher walk per frame (one entry per enabled task) |
| L2 | Blit drain only during VBlank or when D3 idle |
| L3 | Message-ring producer is dispatch-gated (entry 4), consumer is entry 5 |
| L4 | Unblank latch is render-gate: `0x800F84E4` must eval 1 within 45 frames post-transition (I4 watchdog) |

## 7. Priority / Preemption Status (UNVERIFIED)

Entry order in the registry is the only static signal; a priority-based patch must not be
authored before C-1 live capture (RAM write-tripwire over `0x800AE5F8..0x800AE638` +
`0x800C2268..0x800C22A8`; breakpoint at entries 4 & 5 during an Auld Well warp).

## 8. Open Items

| # | Item | Decides |
|---|---|---|
| C-1 | Reader PC + index + whether `0x80031F0C` executes on the Auld Well warp | dispatch/ordering |
| C-3 | `[0x800FE578]` +25 claim-log semantics | counter model |
| C-5 | `WaitEvent` returns / event-flag words around warp | handoff consumption |

---

*Rev 1.0 end. Companion: `YAMANA_HBE_HBD_DMA_SUBBLOC_ROUTINE.md` §4 · Task-Dispatcher monograph · DMA-Channel monograph §1.3.*