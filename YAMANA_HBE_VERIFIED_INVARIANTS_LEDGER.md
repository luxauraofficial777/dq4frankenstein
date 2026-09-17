# Yamana HeartBeat Engine — Verified Invariants & Do-Not-Patch Ledger

**Document ID:** HBE-PSX-ENG-SPEC-2026-V10 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** all HBE corpora; single-source ledger consolidating every hard do-not-patch address and measured constant currently strewn across refutations.
**License:** CC BY-NC-SA 4.0
**Status:** Authoritative ledger — supersedes inline lists.

---

## 1. Hard Do-Not-Patch Addresses (MEASURED — carry from refutations, supersede earlier lists)

| Address | Region | Why (evidence) |
|---|---|---|
| `0x800357D4` | transition gate delay slot | valid fixed-point data `0xFFFFF000` (signed −4096) in 4 regions; clamping corrupts geometry |
| `0x800357EC` | transition gate branch | taken in both GOod/BAD; room seeder gated elsewhere |
| `0x8009E920` | GTE call | mirror-compare path integrity |
| `0x80031E34` | directory reset inner read | the *inner read of the wipe loop*, not a consumer; patching as "consumer fix" is a category error |

### 1.1 Fixed-point sentinel regions (never clamp)

`0x800AA5BC` · `0x800C231C` · `0x800C24AC` · `0x800CCF04` — `0xFFFFF000` is index 25 of both
mirror tables A/B (`0x800C22B8` / `0x800C2448`); it is a permanent signed sentinel, not an
underflow.

## 2. Measured Constants (single source of truth)

| Constant | Value | Measured in |
|---|---|---|
| Disc LBA total | 156,487 | `hbd_forensics` / PVD |
| HBD start LBA | 362 | layout |
| HBD blocks × 2,048 B | 155,975 (319,436,800 B) | census |
| EXE `t_addr / d_addr / d_size` | `0x800918F4 / 0x80017F00 / 0xA8800` | header read |
| EXE PC0 | `0x8008E284` | boot trace |
| Public believed DMA ch3 | `0x1F8010B0/B4/B8` | register map |
| DMA3 ring | `0x800E8000` (16 KB) | layout |
| Unblank latch | `0x800F84E4` | RAM dumps |
| Message ring base | `0x800F5108` (894×4, span 0xE00) | dumps |
| Ring holder | `[0x800FE56C]` | dumps |
| Registry | `0x800AE5F8` / mirror `0x800C2268` (RAM-rel `0x00C2268`) | dumps |
| Mirror tables A/B | `0x800C22B8` / `0x800C2448` (55 words) | dumps |
| Rebuild flag | `[0x800C2384]→0x800C2398`, flag16 @ +32 | Auld Well (reads 0 both states) |
| cdrom flags a/b | `0x800E2038 = 0xA8C30C80` / `0x800E3D94 = 0` | identical GOOD↔BAD |
| CD read times | single 451,584 cyc / double 225,792 cyc | hardware timing |
| Int3 sync timeout | 120 frames | stream sync routine |
| Control codes | 51 (43 archive + 8 EXE) | three-way verification |
| Huffman class predicate | `flags==0x0500 ∧ type∈{23,24,25,27,39,40,42,44}` | dispatch census |
| Overrun contract | `≤ +3` decoded bytes | decompressor |
| Re-encoded dialogue blocks | 1,358 Huffman (depth 14..9) | census |
| Type-39 script blocks | 612 (LZSS→remap→recomp) | census |
| Font 1 | `0x048C` @ 0x97AC8, 8×14, delta-locks 44/36/50/66, SIDs 773–778 | census |
| Battle overlay | `0x048B`, 817 sequences | census |
| Disc-index herbage | LBA 483–504 class (22 sectors) | DW7/Frankenstein |
| Sentinel dir | `0x80100168` (12 slots) | RAM dumps |

## 3. Build & Encoding Invariants

| # | Invariant | Source |
|---|---|---|
| I1 | RAW passthrough for types 21/31/35/44 | compile-time contract |
| I2 | 156,487 sector parity + PVD match | gate |
| I3 | rest_archive modified LBAs ≥ 1,285 | gate |
| I4 | `[0x800F84E4]`==1 within 45 frames of transition | watchdog |
| K2 | `HTS-0x18` header consumed before payload | codec |
| D1 | index cells stable; zero sector shift | index spec |

## 4. Auld Well Evidence Register (measured totality)

| Phase | Evidence | Status |
|---|---|---|
| Spawn | holder written `0x8003183C`/`0x80032548` | MEASURED |
| Seed | producer inserts `0x0019xxxx` slab ptrs (canary=27) | producer PC PENDING |
| Consume | `0x80031F0C` walk | PARTIAL decode |
| Teardown | ring self-chained + mirrors idx0–24 zeroed + counter +25 + latch 0 | MEASURED |
| Re-seed | trigger `0x800C2398+32` never arms | MEASURED (flag 0) |
| Result | content resident, directory empty, display held | MEASURED |

## 5. Invariant Violation Procedure

Per agent-drift ruleset: a doc/source contradiction STOPS work on that topic until resolved
against SOURCE/COMMAND evidence; the losing document is marked `SUPERSEDED` (§0 Mandate of
Sound Logic).

---

*Rev 1.0 end. This ledger supersedes inline do-not-patch lists in earlier documents.*