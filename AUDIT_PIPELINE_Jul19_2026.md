# Pipeline Audit — Jul 19, 2026 (10:00 AM session)

**Constraint:** Frankenstein path only (DW7 EXE + DQ4 HBD). No Native DQ4, no RC2.

---

## 1. New Evidence (verified on-disc this session)

### The hang read at LBA 146621 is NOT corruption
Compared v37's data at the hang point against the DW7 original (`verify_hang_lba.py`):

```
v37[146471+k] == DW7[146470+k]  -> MATCH for all 55 sectors (shift-correct)
v37[146471+k] == DW7[146471+k]  -> MISMATCH (proves the +1 shift IS applied)
```

- DuckStation Setloc 32:34:71 = abs LBA 146621 = data LBA 146471 = HBD-relative sector **146,116**
- The bytes are **MIPS code** — a boot overlay loaded from the HBD, linked against EXE
  addresses (`jal 0x8007FBA4`). Likely loaded into the 0x800BC668+ region (matches v35's
  observed execution at 0x800BDE60).
- Scanned 128 sectors of the overlay (`scan_boot_overlay.py`): **zero** refs to LBA 354,
  zero old-tree addresses (0x800EF1C8), zero CdlStop writes, zero disc-check signatures.
- v37 confirmed built with fixed extension sectors (valid sync+header at LBA 347000).

**Conclusion: the disc is clean at the failure point. The game loads its boot overlay
correctly, executes it, then idles with GPU off. Runtime state divergence, not data
corruption.**

---

## 2. What Has Worked (proven, locked in — do not re-litigate)

| Component | Status | Proof |
|-----------|--------|-------|
| Translation layer | DONE (Jul 4) | 15,318 entries, hybrid tree 1480B/370 leaves, round-trip verified |
| v34 boot chain | PROVEN | Copy routine, tree @0x80100000, PC0 redirect — BIOS/kernel/BSS/CD all pass in DuckStation |
| HBD shift pipeline | PROVEN (today) | Boot-time read returns exactly the right shifted sector |
| Disc assembly | FIXED+PROVEN | EDC/ECC, ISO dir, extension sectors (fixed Jul 19, confirmed in v37) |
| Verification tooling | TRUSTWORTHY | verify_e2e_jul19*.py, verify_hang_lba.py, scan_boot_overlay.py read actual disc bytes |

## 3. What Has Failed (with real root causes)

- v12: broken pointer remap
- v15/v16: disc-check variants (Insert Disc 1 / cd_init destroyed)
- v19: BSS expansion zeroing data
- v31/v32: tree placed in game data section (0x800F4980)
- v33: copy routine bugs (missing NOP, wrong bne)
- v35: in-place tree in overlay-load region (BSS writes destroyed it)
- v23/v24/v36: structurally invalid extension sectors (found+fixed Jul 19)
- v37: FMV skip x2 — wrong target both times; hang is not the FMV

### The honest pattern (why session logs > ROM size)
At least FOUR declared root causes were FALSE, each costing a session:
1. "0x80100000 is unmapped RAM" — false (DuckStation maps 2MB)
2. "CD-ROM intercept not on disc at 0x80F64" — false (checker read wrong sector)
3. "LUI/ADDIU swapped at 0x76B84/0x76B88" — false (checker read bug)
4. "Hang at 146621 is the FMV" — false (it's a boot overlay, loading correctly)

**Rule now in effect: no root-cause claim without a direct on-disc or in-log byte check.**

---

## 4. THE Actual Gap: Missing Control Experiment

**No build has ever shown the Enix logo with our engine patches applied — even on pure
DW7 data.** v30 (DW7 HBD, unshifted, disc-check bypass + Stop intercept, zero DQ4
content) still got "Wrong Disc." Every build since added DQ4 variables on top of an
engine layer never proven to boot cleanly on its own. We have been debugging two
unknowns at once for weeks.

Since the overlay loads correctly and contains no disc-check code itself, the idle loop
comes from **EXE-side state after our patches** — the overlay calls back into EXE code,
and something in the 10-patch bypass / Stop intercept leaves drive-state variables in a
value the boot path polls forever.

---

## 5. The Plan (Frankenstein only)

### Step 1 — Bisect the engine layer (no DQ4 anywhere, 3 cheap builds)
- **v38a**: DW7 disc + engine patches only (disc-check 10, Stop intercept, tree reloc),
  HBD at 354 unshifted → Enix logo?
- **v38b**: v38a + HBD shift to 355 + delta pipeline, still zero DQ4 data → logo?
- Whichever build loses the logo identifies the exact breaking patch layer. Then bisect
  WITHIN the 10 disc-check patches the same way.

### Step 2 — Log-diff at divergence
Run test_0 (pure DW7) and the failing v38 in DuckStation; diff the CD command streams.
First divergent command pinpoints the guilty patch. Boot overlay location known if
disassembly is needed: HBD-rel 146,116 (~55+ sectors).

### Step 3 — Only after logo renders with shift
Re-add the DQ4 overlay (already byte-verified correct in v36/v37), then attack the last
real blocker: re-encoding DQ4 text blocks to DW7 format
(`reencode_dq4_blocks.py` zero-length-tree fix).

Everything above Step 3 is engine plumbing on proven-good data. The architecture is
sound; the remaining failure is localized to our own patch set — 11 patches, finite,
bisectable in an afternoon.

---

## 6. Remaining Blockers (unchanged)

| # | Blocker | Status |
|---|---------|--------|
| 1 | Engine patches never proven to boot cleanly (Sec. 4) | **NEXT — v38a/v38b bisection** |
| 2 | HBD block format: DQ4 per-block trees vs DW7 global tree | Open (Step 3) |
| 3 | `reencode_dq4_blocks.py` zero-length-tree failure | Open (Step 3) |
| 4 | 7 missing translation blocks (043f, 0474-0476, 0481, 0489, 048a) | Open |
| 5 | FMV-after-logo behavior | Unknown until logo renders |

## 7. Verification Scripts (repo root, all trustworthy)
- `verify_e2e_jul19.py` — EXE patch points on disc
- `verify_e2e_jul19b.py` — HBD layout / shift / append
- `verify_e2e_jul19c.py` — EDC spot-check
- `verify_hang_lba.py` — hang-point data vs DW7 source
- `scan_boot_overlay.py` — boot overlay content scan

## 8. Related Docs
- `study/E2E_VERIFICATION_Jul19_2026.md` — extension-sector corruption + fix
- `study/SESSION_v37_FMV_SKIP_Jul19_2026.md` — FMV skip failure analysis
- `study/DEFINITIVE_BLUEPRINT_Jul18_2026.md` — architecture + build history
