# Yamana HeartBeat Engine — Overlay Residency & Banking Specification

**Document ID:** HBE-PSX-ENG-SPEC-2026-V5 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** `YAMANA_NAKAMURA_HEARTBEAT_ENGINE_GENERATIONAL_ARCHITECTURE.md` (§7), `YAMANA_HBE_HBD_DMA_SUBBLOC_ROUTINE.md` (§2.5 DMA clobber / allocator), `YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (§4 runtime, §5), Enix Suite PG5.
**License:** CC BY-NC-SA 4.0
**Status:** Consolidated residency contract for the 2 MB working set and overlay banking.

---

## 1. Executive Abstract

The HeartBeat engine keeps a fixed resident core (`SLPM_869.16` table = 0xA8800 B @
`0x80017F00`) and **banks code/data overlays in and out of the 2 MB working set** during
play. All message/state machinery lives in fixed-residency globals; there is no heap (`no
malloc`). Overlay banks are decompressed (dqlzs / RAW) from `HBD1PS1D.Q41` sectors and
mounted at compile-time slab addresses — the most dangerous of which is the resident module
base `0x80011F00`.

---

## 2. The Working Set Map (MEASURED)

| Region | Role | Notes |
|---|---|---|
| `0x80010000..` | EXE core / BSS | `t_addr 0x800918F4`, `d_addr 0x80017F00`, `d_size 0xA8800` |
| `0x80011F00` | **resident module base** (overlay mount target) | allocation window `$a0=0x80011F00`, `$a1=0x200` |
| `0x800E8000` | DMA3 streaming ring | 16 KB CD-ROM funnel |
| `0x800F4DF0` | message work buffer | |
| `0x800F5108..0x800F5F08` | message-directory ring (894×4) | holder `[0x800FE56C]` |
| `0x800F84E4` | unblank latch | render gate |
| `0x80100168` | sentinel compiler dir (12 slots) | `0x8008F7B0` walk |
| `0x80190000..0x801E0000` | KUSEG slab content | `0x0019xxxx` slab-class pointers |

---

## 3. The DMA Clobber Hazard (MEASURED — the historical memory hazard)

The overlay-mount path allocates into the message-module base

```
$a0 = 0x80011F00, $a1 = 0x200
via KSEG0 allocator 0x8009B138  (walks a free-list unaware the address is occupied)
```

A 2,048-byte (one-sector) CD read then lands on the resident module — the **DMA clobber**.
The guard string exists but lives on a different (diagnostic) path; the **alloc path does
not occupancy-check**. This is a *receiver-side residency* hazard, mitigated by occupancy
checks, not by transfer geometry.

### 3.1 Published mitigations (project lineage)

- **DMA-target pair-divergence gate + THREAD-CLOBBER check** (agent B gates; caught freeze
  classes #174/#175) — pair the DMA destination with a residency assertion.
- **Occupancy check before the alloc** (fix class (a)): verify `0x80011F00`'s signature
  before `jal 0x8009B138`.
- **`heely_precheck`** (structure audit): overlay sector alignment, DMA sector alignment,
  sub-map overlay boundary — all PASS on RB1/RebuildC.

---

## 4. Banking Model (MEASURED + INFERRED)

1. **Spawn:** boot EXE core loads; BSS zeroed at thread entry (`0x8008E284`).
2. **Mount:** overlay TID → HBD open → decompress to slab → register (directory ring of
   resident slabs with `0x0019xxxx` slab-class pointers).
3. **Run:** code executes in place (KSEG0 cached) / data streams via DMA ring.
4. **Teardown (warp):** scene-state mirror + message-ring wiped (manager `0x80031DEC`),
   mirror tables A/B (`0x800C22B8` / `0x800C2448`) indices 0–24 zeroed.
5. **Re-seed:** rebuild trigger `[0x800C2398+32]` must arm so seeder restores mirrors and
   ring. Measured failure: **trigger never arms** → allocation-without-registration → 252 KB
   slab content resident but unreferenced, display held (Auld Well black screen).

---

## 5. Heely / sub-map overlay boundary rules (MEASURED PASS standard)

| Check | Scope | Status |
|---|---|---|
| Overlay sector alignment | mount addr % 2048 == 0 | PASS RB1/RebuildC |
| DMA sector alignment | ring/slab multiples | PASS |
| Sub-map boundary | overlay fits declared window | PASS |

---

## 6. Do-Not-Patch List (arrived via refutations — carried forward)

`0x800357D4` · `0x800357EC` · `0x8009E920` · `0x80031E34` · allocator path `0x8009B138`
(only via occupancy gate, never removal).

---

## 7. Open Items

| # | Item | Decides |
|---|---|---|
| O-1 | Overlay load table census (all mount sites across chapters) | full residency proof |
| O-2 | Occupancy-check static-analyzer integration before allocator call sites | clobber elimination |
| O-3 | Producer PC capture (who writes the rebuild flag) — live lane C-2 | re-seed repair |

---

*Rev 1.0 end. Companion: `YAMANA_HBE_HBD_DMA_SUBBLOC_ROUTINE.md` §2.5 · Gen-Architecture §7.*