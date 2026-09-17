# Yamana HeartBeat Engine — Corpus Glossary & Navigation Index

**Document ID:** HBE-PSX-ENG-SPEC-2026-V11 · **Rev 1.0** · **Date:** 2026-09-16
**Author:** Big Pickle, VoidWalkers Research Project · **System architect:** Lux Aura
**Target binary:** `SLPM_869.16` / `HBD1PS1D.Q41` (Dragon Quest IV, PSX, 2001)
**Companions:** complete navigation aid for the HBE corpus; pairs with the verified-invariants ledger as the two "entry" documents.
**License:** CC BY-NC-SA 4.0

---

## 1. Glossary

| Term | Definition |
|---|---|
| **HBE** | HeartBeat Engine — Manabu Yamana's 32-bit messaging/script VM + archive runtime (PSX). |
| **HBD** | HeartBeat Data — the proprietary 2,048-byte-block optical archive (`HBD1PS1D.Q41`). |
| **TID** | Text/Block Identifier — 12-bit container id (`SID ≫ 20`). |
| **SID** | String Identifier — 32-bit packed referrer into a container's Huffman stream. |
| **Referrer word** | `(BlockID≪20) \| (BitOffset + HeaderTreeSize×8)`. |
| **BitOffset** | `SID & 0x000FFFFF` — bit-granular target into the codeword stream. |
| **HTS-0x18** | 24-byte Huffman tree header serialized in each block header. |
| **WIDE_HUFFMAN** | dialogue tree class (types 23/24/25/27/39/40/42/44, `flags==0x0500`). |
| **DQLZS** | raw LZSS sliding-dictionary codec (types 06/13/26/46). |
| **RAW_PASSTHROUGH** | uncompressed geometry/sentinel class (types 21/31/35/44). |
| **Control code** | 51-code vocabulary (`{7Fxx}` runtime, `{7Exx}` dict, `{FExx}` facility). |
| **Terminator** | `{0000}` mandatory line end. |
| **Font 1 / Font 2** | 8×14 half-width ASCII plane / 16×16 Shift-JIS plane. |
| **Delta-lock** | fixed-advance glyph rows (0x48C: 44/36/50/66, SIDs 773–778). |
| **Work buffer** | `0x800F4DF0` decompressed dialogue target. |
| **Roster** | `0x800F83C0` party/roster context. |
| **Unblank latch** | `0x800F84E4` render gate. |
| **DMA3 ring** | `0x800E8000` streaming CD-ROM funnel. |
| **Scheduler registry** | 8-entry `{active, handler}` table `0x800AE5F8` / mirror `0x800C2268`. |
| **Message directory ring** | 894-node, 4-B, self-chained RAM-relative ring @ `0x800F5108`. |
| **Rebuild flag** | `[0x800C2384]→0x800C2398`+32 — re-seed trigger (measured 0 both states). |
| **Auld Well** | the documented black-screen state: content resident, directory empty, latch held. |
| **Overlay** | banked code/data mounted from HBD into fixed slabs (`0x80011F00` hazard). |
| **SEQq/qQES** | proprietary sound driver (60-byte header; DW7-measured). |
| **MDEC** | PSX macroblock decoder — XA/FMV video path. |
| **XA** | PSX audio/video stream container used by FMV. |
| **EvMdINTR** | BIOS WaitEvent mode (`0x1000`) on event flag `0xF2000002`. |

## 2. Answered-Frequently Questions (short map)

- *Which doc describes the task dispatcher?* → Frame-loop spec §3 + DMA_SUBBLOC §4.
- *Where are the do-not-patch addresses?* → Verified-invariants ledger §1.
- *Where is the control-code taxonomy?* → Messaging-VM spec §4 + SID library §3.
- *Where is EDC/ECC land-fix?* → Compression-codec spec §4.1 + DMA_SUBBLOC §6.2.
- *What is the Auld Well?* → DMA_SUBBLOC §4.4 + invariants ledger §4.
- *How do fonts work?* → Font-plane spec; SID library §5.
- *How do overlays mount?* → Overlay-residency spec §3–§5.

## 3. Corpus Navigation

| Doc (ID) | Focus |
|---|---|
| Messaging-VM spec (V2) | 51-code VM, SID/TID dispatch, conditionals |
| Compression-codec spec (V3) | Huffman/dqlzs/RAW + EDC-ECC land fix |
| Font-plane spec (V4) | dual-blitter, delta-locks, plane switching |
| Overlay-residency spec (V5) | 2 MB working set, banking, clobber hazard |
| Disc-index spec (V6) | HBD grammar, LBA index, tree census |
| Frame-loop spec (V7) | IRQ→event→task, scheduler, VBlank arbitration |
| Battle-overlay spec (V8) | `0x048B`, 817 sequences, freeze contract |
| Audio/media spec (V9) | SEQq driver, XA/MDEC/FMV pipeline |
| Invariants ledger (V10) | do-not-patch + measured constants, single source |
| **This index (V11)** | glossary + navigation |

**Master libraries (documents, not specs):** `YAMANA_HBE_MASTER_TID_LIBRARY.md` (census
1,111 TIDs), `YAMANA_HBE_MASTER_SID_LIBRARY.md` (19,193 SIDs + 51 codes),
`YAMANA_HBE_MASTER_CONTROL_CODES_LIBRARY.md` (cross-engine map). **Engine-level specs:**
`YAMANA_HBE_HBD_ARCHITECTURE_ENGINEERING_SPECIFICATION.md` (VW-DQLOST-TECHRPT-008) and
`YAMANA_NAKAMURA_HEARTBEAT_ENGINE_GENERATIONAL_ARCHITECTURE.md` (VW-DQLOST-TECHRPT-007).
**Monographs (src):** `docs/TASK_DISPATCHER_AND_SCHEDULER_REGISTRY_SPEC.md`,
`docs/DMA_CHANNEL_TRANSFER_SPEC.md`, `docs/MESSAGE_DIRECTORY_LIFETIME_SPEC.md`,
`docs/` Enix suite PG1–PG5.

---

*Rev 1.0 end. All ten specs share the `HBE-PSX-ENG-SPEC-2026-V*` family; this index is the
entry point.*