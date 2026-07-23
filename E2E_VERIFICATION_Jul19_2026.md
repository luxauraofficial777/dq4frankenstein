# E2E Verification — Jul 19, 2026

**Session:** Toolchain gap analysis + end-to-end on-disc verification of v34/v35/v36 builds.
**Verdict:** Real corruption found and fixed (disc extension sectors); two Jul 18 "corruption" claims debunked as verification-script bugs.

---

## 1. REAL Corruption Found: v36/v23/v24 Extended Disc Region Unreadable

The disc extension beyond the original DW7 end (LBA 302,537) was written as **raw zeros**:
- No sync pattern (should be `00 FF×10 00`)
- No BCD MSF header, mode byte = 0 (Mode 0 = empty sector)
- No Form1 subheader
- EDC/ECC = 0x00000000

Evidence:

```
LBA 302536 (last original): 00ffffffffffffffffffff00 67156102  <- valid sync+header
LBA 302537+ (extension):    0000000000000000...                <- structurally invalid
LBA 347000: stored EDC=0x00000000, calc=0xC51D612D  <- EDC MISMATCH
```

Impact:
- v36 appends 883 DQ4 folders starting at LBA 302,388; only 302,388–302,536 (~149 sectors) landed in valid sectors. **~45,150 appended DQ4 folder sectors are unreadable** by emulators.
- `edcre.exe` silently skips mode-0 sectors, so EDC/ECC regeneration never touched the extension.
- **v23 and v24 have the exact same defect** — likely contributed to their black screens (not just the "STR internal LBAs" theory).

### Fix Applied
- New `extend_disc_mode2()` helper in `translation-tools/frankenstein_builder.py` (lines ~520–545): writes structurally valid MODE2 Form1 sectors (sync + BCD MSF header with +150 pregap + mode 2 + Form1 subheader) for every extension sector.
- Wired into all three affected build paths: `frank_v13`, `frank_v14`, `frank_v36`.
- edcre timeout raised 300s → 900s for the larger 817MB disc.
- Syntax verified (`py_compile` OK). **v36 must be REBUILT.**

---

## 2. Jul 18 Session Claims DEBUNKED (from SESSION_SUMMARY_v35_debug_Jul18.md)

Both claims disproved by direct on-disc reads:

1. **"CD-ROM intercept NOT patched on disc at 0x80F64" — FALSE.**
   v34/v36 have `j 0x801005C8` + `nop` at 0x80F64/0x80F68 on disc (v35: `j 0x800BCCC8`).
   The Jul 18 checker computed the wrong sector: it claimed sector 172; correct is
   `24 + 0x80F64 // 2048 = 281`.

2. **"LUI/ADDIU swapped at 0x76B84/0x76B88" — FALSE.**
   Original DW7 EXE: `0x76B84 = lui $v0, 0x800C`, `0x76B88 = addiu $v0, 0xC668`. Correct order.
   The Jul 18 verification script had a read bug.

Do NOT chase these — the EXE patches on disc are correct.

---

## 3. Verified GOOD on v34/v36 (direct on-disc reads)

- **EXE header:** PS-X EXE sig, t_size=0xA5000 (extended file = 677,888 bytes), PC0=0x800BCCF0, base=0x80017F00
- **No EXE/HBD overlap:** sector 354 = hybrid tree data (EXE sector 330), HBD at 355
- **Hybrid tree:** 1480 bytes at file 0xA5000 (RAM 0x800BC700), byte-identical to `dw7_hybrid_tree.bin`
- **Copy routine:** all 14 instructions byte-exact v34 pattern (NOP @idx6, `bne 0xFFFA`, `j 0x8008E284`)
- **v36 DW7 HBD shift:** `v36[355+k] == DW7[354+k]` for k = 0, 1, 100, 10000, 150000, 302032 — all MATCH
- **v36 DQ4 folders:** sampled appended folders match source `dq4_heart_v17.bin` byte-exact
- **ISO dir:** v36 HBD LBA=355 size=711,034,880; v34 HBD LBA=355 size=319,436,800
- **EDC valid** on every checked sector within the original disc size (24, 281, 354, 355, 356, 150000, 302388, 302400)
- **DW7 and DQ4 HBD headers share identical first 16 bytes** (`2923be84e16cd6ae...`) — same HBD format family
- v35 sector 355 differs at bytes 8–11 **intentionally** (its `headers=1` graft patch), not corruption

---

## 4. Toolchain Gap Summary

| # | Gap | Status |
|---|-----|--------|
| 1 | Zero-filled extension sectors (v36/v23/v24) | **FIXED — rebuild v36** |
| 2 | Prior verification scripts read wrong sectors → false corruption reports | Replaced by 3 new scripts |
| 3 | `reencode_dq4_blocks.py` fails on tree-length=0 blocks (re-encoding skipped in builds) | Open — blocker for text rendering |
| 4 | 7 missing translation blocks (043f, 0474, 0475, 0476, 0481, 0489, 048a) | Open |
| 5 | FMV/STR skip not implemented | Open |

---

## 5. Verification Tooling (kept in repo root)

- `verify_e2e_jul19.py` — EXE patch points: header, CD-ROM intercept, BSS pair, tree bytes, copy routine, HBD header, ISO dir
- `verify_e2e_jul19b.py` — HBD layout: DW7 shift check, DQ4 folder append check, header comparisons
- `verify_e2e_jul19c.py` — EDC spot-check on modified sectors (MODE2 Form1 CRC)

---

## 6. Next Steps

1. Rebuild: `python translation-tools/frankenstein_builder.py --profile frank_v36`
2. Re-run all three verify scripts — extension-region EDC should now pass
3. DuckStation test of rebuilt v36 (title screen check)
4. Then: fix re-encoding zero-length trees, FMV skip, 7 missing blocks (see DEFINITIVE_BLUEPRINT_Jul18_2026.md)
