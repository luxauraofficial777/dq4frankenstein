# SHIPPING ROADMAP — The Final Road (Jul 19, 2026, 11:40 AM)

**Mission:** Boot Frankenstein (DW7 EXE + DQ4 HBD) in DuckStation, render DQ4, ship XDelta.
**This document supersedes all prior "next steps" lists.** Agents execute the gates IN ORDER.
**Rule in force:** No root-cause claim without a DuckStation log or on-disc byte check.

---

## ⚠ CRITICAL CORRECTION BEFORE ANY AGENT ACTS

The v38 bisection (SESSION_v38_BISECTION) and the VA/CB "thread-entry fix" conclusion have a
**fatal logic flaw** that must not drive shipping decisions:

- The bisection ran ONLY in CyberGrime, where **the unmodified retail DW7 disc ALSO shows
  empty VRAM and stalls in the VBlank loop**. Retail DW7 boots fine on real hardware and in
  DuckStation. Therefore "BSS clear zeros the thread entry at 0x800D9E80 → black screen"
  **cannot be the root cause** — retail DW7 needs no fix.
- What actually happens (proven Jul 19 AM, `verify_hang_lba.py`): after BSS clear, the game
  **loads a code overlay from the HBD** (HBD-rel sector 146,116, ~55 sectors of MIPS linked
  against EXE addresses) into RAM — this is how 0x800D9E80 gets populated at runtime.
  CyberGrime's stub CD/GPU interrupt chain can't complete that overlay hand-off on ANY disc.
  This is documented emulator limitation #1 (memory: "CyberGrime freeze, ALL discs").
- **DO NOT integrate `append_reloc_payload_with_thread_fix` (Option A) or
  `patch_bss_clear_narrow` (Option B) into shipping builds** until Gate 1 proves the problem
  exists in DuckStation. Narrowing the BSS clear on a retail-working engine risks a fifth
  false root cause. The C++ library itself (psx_ops.dll) is good work — keep it — but the
  thread-entry patches are quarantined pending Gate 1.

**What IS proven and locked (do not re-litigate):**
- v38a/v38b engine patches cause no crash/divergence (bisection's valid conclusion)
- v37 disc is byte-perfect at the hang LBA; shift pipeline correct; extension sectors valid
- v34 boots in DuckStation with GPU rendering (FPS ~511)
- Translation layer 100% encoded with hybrid tree

---

## GATE 1 — DuckStation Oracle Run (the single missing data point)
**CyberGrime cannot answer the logo question. DuckStation is the only oracle.**

Run these four discs in DuckStation, 60 seconds each, record: Enix logo Y/N, title Y/N, log tail.

| # | Disc | Question answered |
|---|------|-------------------|
| 1 | `DW7D1\DW7D1.bin` (retail) | Sanity: logo MUST appear. If not, fix DuckStation setup first. |
| 2 | `dq4_frankenstein_v38a.bin` | Do the 10 disc-check patches kill the logo? |
| 3 | `dq4_frankenstein_v38b.bin` | Does full engine + HBD shift kill the logo? |
| 4 | `dq4_frankenstein_v37.bin` | Current best full build — reference failure signature |

### Decision tree (branch immediately, no analysis sessions)
- **v38b shows logo** → engine + shift PROVEN. Failure is DQ4 content only → GATE 2.
- **v38a logo, v38b black** → bisect shift layer: build v38b minus one patch class at a time
  (order: ABS delta → seq table → LBA scan → tree reloc → Stop intercept). 5 builds max.
- **v38a black, retail logo** → bisect the 10 disc-check patches (binary split: 5+5, then 2-3 builds). 4 builds max.
- **Retail black** → DuckStation config problem, not our discs. Fix and rerun.

---

## GATE 2 — DQ4 Content Layer (only after v38b logo confirmed)
The v36/v37 machinery is byte-verified. If v38b boots but v37 doesn't reach the DQ4 title:

1. The difference is exactly: 168 matched ABS/REL pointers redirected to appended DQ4 folders
   (LBA 302,388+) + ISO dir size. Build **v39 = v38b + appended DQ4 folders but ZERO pointer
   redirects** → must boot identically to v38b (appended data inert). Confirms disc mechanics.
2. Then redirect ONE folder (the first boot-path folder from `folder_mapping.json`) → test.
   The first redirected folder that kills boot identifies the parser-facing block.
3. Expected outcome: DW7 parser chokes on DQ4 block format (hts=24, treeEnd/textEnd non-zero
   vs DW7's 1366-byte gap format). That is GATE 3's job — do not hack around it here.

---

## GATE 3 — HBD Block Re-encoding (the last real engineering)
Convert redirected DQ4 blocks to DW7 block format so the DW7 parser can read them:

- **Tool**: `cybergrime/psx_ops.py` → `HbdReencoder` (zero-length-tree handling now explicit
  — closes the old `reencode_dq4_blocks.py` blocker)
- **Text**: already encoded with hybrid tree (`dw7_encoded_translation.json`, 15,318 entries,
  100% fit). Tree ships at 0x80100000 via proven v34 copy routine.
- **Format transform per block**: DQ4 (hts=24, tree-after-text, treeEnd/textEnd set) →
  DW7 (1366-byte gap layout, treeEnd=0, textEnd=0, global tree). Validate each converted
  block by round-trip decode with DW7Schema before writing to disc.
- Re-encode ONLY the folders that get pointer-redirected (168 matched), not all 1,105 blocks.

---

## GATE 4 — Ship
1. `/golden-path` workflow: boot → first battle → save → victory checkpoint
2. Translate 7 missing blocks (043f, 0474, 0475, 0476, 0481, 0489, 048a)
3. FMV behavior check (only after logo/title работают — FMV is AFTER logo, per user)
4. XDelta3 patch vs original DQ4 JP disc + README. Done.

---

## Agent Standing Orders
1. **DuckStation is the oracle.** CyberGrime is for instruction-level tracing only — its
   VRAM/VBlank results are NOT evidence of disc problems (all discs stall in it).
2. **One variable per build.** Every build differs from a proven-good build by exactly one
   patch class. Name builds sequentially, log the diff in the build output.
3. **Verify on disc before testing.** Run `verify_e2e_jul19*.py` equivalents on every new
   build (30 s) before spending a DuckStation run on it.
4. **No new root-cause theories without evidence.** Five false root causes so far
   (unmapped RAM, intercept-not-patched, LUI/ADDIU swap, FMV hang, BSS-zeroes-thread-entry*).
   *pending Gate 1 disproof. Each cost a session. Check bytes first.
5. **Do not touch what is locked**: v34 copy routine, hybrid tree, shift pipeline,
   extension sectors, translation JSON.

## Budget
- Gate 1: 4 DuckStation runs, zero builds — ~30 min, answers the question that has blocked
  the project for 3 weeks.
- Worst case to logo: +5 bisection builds.
- Gate 2: 2-3 builds. Gate 3: the only open engineering (~1-2 sessions with HbdReencoder).
- Everything else is packaging.

*The combination on the LBAs is already broken — the disc reads the right sectors (proven
byte-for-byte). What remains is finding which of OUR patches DuckStation objects to (Gate 1),
then teaching DW7's parser to eat DQ4 blocks (Gate 3). Two problems, both bounded.*
