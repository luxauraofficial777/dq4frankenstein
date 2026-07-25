# CLEAN-ROOM REVERSE FRANKENSTEIN BLUEPRINT + TOOLCHAIN AUDIT
**Date:** Jul 23, 2026 9:43pm UTC-04:00
**Scope:** Audit of `build_reverse_option_a()` pipeline (DW7 EXE → DQ4 ROM body). NO code was edited — this is the blueprint of required changes.
**Sources:** `cybergrime/psx_binary_ops.cpp/.h`, `study/DISASM_PATCH_TARGETS_Jul23_2026.md`, `study/REVERSE_FRANKENSTEIN_PROGRESS_Jul23_2026.md`, `study/FINAL_ARCHITECTURE_TRIAGE_Jul23_2026.md`, `study/BUILD_HANDOFF_Jul23_2026.md`, DQHBE REFERENCE DOC (ROM Hacking Schema, dq4study, dq7study).

---

## VERDICT

The DW7 EXE is **NOT yet properly modified to load DQ4 HBD**. The reverse build pipeline (`build_reverse_option_a`, `psx_binary_ops.cpp:2733`) runs to completion but contains **2 critical correctness bugs, 3 broken/harmful patches, and 3 missing patches**. The observed runtime state (boots, GPU/threads init, stalls in VBlank loop, zero CD-ROM commands) is fully explained by the findings below.

---

## PART 1 — CRITICAL BUGS (must fix before any further test)

### BUG-1: Tree pointer variable initialized at WRONG file offset (CRITICAL)
- `psx_binary_ops.h:111` — `TREE_PTR_VAR_OFF=0xA46F8` is **wrong**.
- Correct: RAM `0x800BC6F8` − load `0x80017F00` + header `0x800` = **`0xA4FF8`**.
- Consequences (build step 7, `psx_binary_ops.cpp:2818-2823`):
  1. The tree address `0x800BC700` is written to file `0xA46F8` = RAM `0x800BBDF8` — **corrupting legit EXE data there**.
  2. The real variable at `0x800BC6F8` (file `0xA4FF8`) was zeroed by `zero_pre_tree_bss()` (zeros file `0xA4F68–0xA5000`) and stays **0x00000000**.
  3. Runtime: Target-4 patched code does `lw $reg, 0($at)` from `0x800BC6F8` → **loads NULL tree base** → decompressor walks tree at address 0 → garbage text / crash on first decompression.
- **Fix:** `TREE_PTR_VAR_OFF = 0xA4FF8`. Keep write-after-zero ordering (already correct).

### BUG-2: `patch_decomp_tree_pointer()` clobbers a live instruction + load-delay hazard (CRITICAL)
- `psx_binary_ops.cpp:1521-1593`. The patch overwrites **3 consecutive slots** (lui / addiu / "delay").
- Its own comment (`:1511`) shows the third slot at `0x8006F104` is `0x00031840` = **`sll $v1,$v1,1`** — NOT a nop. Overwriting it destroys the ×2 node-index scaling that feeds `addu $s0,$v1,$v0` at `0x8006F108`.
- Additionally `lw $v0` placed in the slot directly before `addu ...,$v0` = **R3000 load-delay hazard** (consumer reads stale $v0; DuckStation models load delay).
- The code never verifies the third slot's content before overwriting (no guard).
- **Fix (2-instruction form, delay slot untouched, no hazard):**
  ```
  0x8006F0FC: lui   $reg,0x800F    →  lui $at, 0x800C          (0x3C01800C)
  0x8006F100: addiu $reg,$reg,imm  →  lw  $reg, -14600($at)    (lw $reg,0xC6F8($at))
  0x8006F104: (sll $v1,$v1,1)      →  UNTOUCHED — fills the load-delay slot naturally
  ```
  `lw` encoding: `0x8C000000 | (1<<21) | (reg<<16) | 0xC6F8`. Same for site 2 (`0x8006F144/0x8006F148`). This removes the third write entirely.

### BUG-3: No runtime writer for the per-block tree pointer (ARCHITECTURAL GAP)
- Target 4 as designed points the decompressor at variable `0x800BC6F8`, but **nothing ever updates that variable per block**. It is statically initialized to the hybrid tree `0x800BC700` — that is only "global tree via indirection", NOT "read DQ4 per-block inline trees".
- Per `DISASM_PATCH_TARGETS` open issue + DQHBE ROM Hacking Schema requirement #3: *"the text decoder must dynamically read the Huffman tree offset from offset 0x0C of each text block's header."*
- **Required new patch (R-TREEHOOK):** hook the block loader where the 24-byte text-block header is parsed (region ~`0x8006Fxxx`, near the tree-materialization sites). Insert stub (in trampoline area `0x800BD080+`):
  `tree_addr = block_base + tree_start(header@0x0C)` → `sw` to `0x800BC6F8` before decompression is invoked.
- Until R-TREEHOOK exists, the build only works if HBD text is re-encoded to the hybrid global tree (the abandoned 37%-failure graft path). **Decide: native per-block (needs R-TREEHOOK) is the doctrine per FINAL_TRIAGE.**

### BUG-4: root_id +1 applied inconsistently across tree-base sites
- `patch_decomp_root_id_plus1()` adds +2 only at site `0x8006F1A4` (file `0x572A4`).
- Sites 1&2 (now runtime-variable loads) deliver the **unadjusted** base; per DISASM doc two more pairs exist in the block loader (`0x80056AEC/0x80056B34`) patched by `patch_tree_references` — also unadjusted.
- If the DQ4 +1 convention applies to **all node indices** (root and children — cross-parse evidence suggests a uniform index offset), then the +2 must be uniform: either store `tree_base+2` in the runtime variable and drop the site-3 patch, or +2 at every base materialization.
- **Verify first:** read `HuffTree::parse_dq4` (`psx_binary_ops.cpp:473-511`) — confirm whether child IDs also get +1 or only root. Then choose ONE uniform mechanism.

---

## PART 2 — BROKEN / HARMFUL PATCHES (remove or rework)

### BAD-1: `patch_cdrom_cmd_intercept()` — multiply broken, remove from reverse build
`psx_binary_ops.cpp:2035-2190`. Defects:
1. **Destroys the instruction at dispatch+4** (`0x80F68` = a branch per `CDROM_BRANCH_OFF`); writes nop there, trampoline re-executes only the FIRST original instruction. Patch record at `:2064` even captures the already-nop'd value as "original" — unrecoverable via revert.
2. **Wrong hardware model:** reads `0x1F801803` expecting the command byte (write-only command register; reads return IE mirror) and reads `0x1F801802` expecting pushed params (reads pop the DATA FIFO). Intercept conditions can never match reliably; stray data-FIFO reads disturb CD state.
3. "FIFO clear" writes `0x40` to `0x1F801803` without setting the index register — hits Request/BFRD, not the param-FIFO clear (Interrupt Flag reg, index 1, bit 6). Appending 3 more params after the game's 3 = the classic "expecting 3-3 got 12" Setloc error.
4. Redirect target LBA 355 (MSF 0:06:05) is a DW7-pipeline sector — on the DQ4 disc the safe target would be ≥362. Minute-only BCD check is coarse.
- **Blueprint action:** OMIT from the reverse/DuckStation build. If FMV redirect is ever needed, hook AFTER param collection at a RAM-visible point, not the hardware FIFO.

### BAD-2: `patch_fmv_skip()` — wrong mechanism vs. the documented root-cause fix
- Implemented (`:1956-2001`) as Setmode param `0xA0→0x00` byte-search in `0x80098800-0x80098900`. This changes CD speed/ADPCM mode; the EXE **still seeks nonexistent FMV sectors** → the diagnosed black-screen hang remains possible.
- `BUILD_HANDOFF` prescribes function stubs: `0x8008AEF4` (XA/STR, file `0x737F4`) and `0x8008CAD0` (MDEC init, file `0x753D0`) each ← `li $v0,0; jr $ra; nop` (`24 02 00 00 / 03E00008 / 00000000` — note: use `addiu $v0,$zero,0` = `0x24020000`).
- `gen_fmv_skip()` exists in `frankenstein_driver.cpp:475` but is **not wired** into this pipeline.
- **Blueprint action:** replace param-patch with the two 12-byte stubs.

### BAD-3: Self-contradicting verification gate
- `verify_integrity()` (`:1739-1782`) **fails** on any `fmv_skip*` patch (forbidden list) and calls `verify_cdrom_flow()` which **fails** if the stall-bypass bne at `0x800986E0` was patched.
- `build_reverse_option_a` applies BOTH (steps 9 & 11) → the build **always ends with integrity FAIL warnings**. Gate and pipeline disagree about policy.
- **Blueprint action:** (a) stall bypass must be OMITTED for DuckStation (mirror `build_v39` `build_flags bit0`); (b) FMV stub patches get a distinct name (`fmv_stub`) not on the forbidden list, or the forbidden rule is updated to match current doctrine; (c) PC0 check must accept the thread-stub PC0 redirect (see MISS-1).

---

## PART 3 — MISSING PATCHES (gaps)

### MISS-1: Thread code injection at 0x800D9E80 (THE observed blocker)
- Reverse build never injects `thread_code_blob.bin` (2,048 bytes, exists in `cybergrime/` and `frankenstein_pipeline/data/`). OpenTh/ChangeTh land on zeros → VBlank-loop stall, zero CD-ROM commands (confirmed by `telemetry_reverse_v4` trace).
- **Direct embed is IMPOSSIBLE:** RAM `0x800D9E80` = file `0xC1B80` > max EXE slot `0xA9000` (692,224 B). The EXE image can only cover RAM up to `0x800C0700`.
- **Required mechanism (boot copy stub):**
  1. Append blob at EXE end (current end after trampolines ≈ RAM `0x800BD070`; headroom to `0x800C0700` ≈ 13.9 KB ≫ 2 KB blob + stub).
  2. Append ~10-instruction stub: copy 2048 B from blob RAM → `0x800D9E80`, then `j 0x8008E284` (original entry).
  3. Set header PC0 (+0x10) → stub address.
  4. **Works because** runtime BSS clear in this build is `[0x800BD100, 0x800D9E80)` — the copied blob at `0x800D9E80` is NOT re-zeroed. (Forward build needed a post-BSS hook; reverse does not.)
  5. Keep blob+stub below `0x800BD100`? NO — blob source must live BELOW BSS-start `0x800BD100`... **CONFLICT:** current BSS start `0x800BD100` would zero any payload appended above it before... (BSS clear runs inside game init, AFTER our boot stub copies — order is: BIOS load → PC0 stub copies blob → jumps to `0x8008E284` → game clears `[0x800BD100,0x800D9E80)` → blob source region (if ≥0x800BD100) gets zeroed but the copy at `0x800D9E80` SURVIVES). Source zeroing after copy is harmless. ✔ viable.
- Existing `append_reloc_payload_with_thread_fix()` (`:299`) implements a variant — evaluate reuse, but the simple boot-copy above is sufficient and smaller.

### MISS-2: HBD archive identity check (filename + internal volume string)
- DW7 EXE hardcodes `hbd1ps1d.w71` at RAM `0x80026D00` (file `0xF600`). Current build renames the ISO dir entry Q41→W71 (`update_iso_dir_reverse`, `:2655`).
- **Gap:** DQHBE Schema doc requires the archive's **internal Sector-0 volume string** (`hdb1ps1d.q41` at HBD offset 0x740; dq7study says 0x400 — verify both) to match what the engine verifies. Grep confirms NO code patches it (`hdb1ps1d`/`0x740` absent from pipeline).
- **Recommended (cleanest, keeps HBD + ISO dir 100% pristine):** patch the EXE string at `0x80026D00` from `hbd1ps1d.w71` → `hbd1ps1d.q41` (same length), and **drop the Q41→W71 ISO rename** entirely. One 3-byte EXE edit replaces two riskier edits.
- Alternative: keep W71 rename + write `hdb1ps1d.w71` into HBD sector 362 offset 0x400/0x740 (3-byte edit inside "unmodified" HBD — violates doctrine).

### MISS-3: EDC/ECC regeneration failing
- `REVERSE_FRANKENSTEIN_PROGRESS`: build log shows `EDC/ECC: FAILED` (`run_edcre`, `:1298`, shells out via `system()`).
- DuckStation tolerates missing EDC; real hardware and some cores will not. **Action:** capture edcre stderr, verify path quoting/exit code; if tool rejects the image, run manually: `edcre.exe -v dq4_reverse.bin` and record output. Must be green before release builds.

---

## PART 4 — LOWER-SEVERITY FINDINGS

| ID | Finding | Location | Action |
|---|---|---|---|
| W-1 | Tree append: `TREE_FILE_OFF=0xA4800` guard is dead (EXE is already 0xA5000 B); append lands at file 0xA5000 = RAM 0x800BC700 **by accident of size**. Log line claims 0xA4800 (false). | `:2769-2785` | Assert `exe.size()==0xA5000` before append; fix log/constant (correct file off = 0xA5000). |
| W-2 | `patch_lba_references(354,362)` scans EVERY aligned 32-bit word == 354 → false-positive risk. | `:92-103` | Log count + offsets; diff against known-good audit (`exe_lba_audit.txt`). |
| W-3 | Sequential table shift: ALL 16-bit entries in [300,400] get +8. Assumes DQ4 HBD's first sectors serve the same roles at same relative positions as DW7's. Unverified against DQ4 archive layout. | `:104-122` | Dump DQ4 HBD sectors 350-370 roles first; may need per-entry mapping instead of blanket +8. |
| W-4 | HBD type trampoline swaps only types 32/39/40/42/46; other DQ4 types (1,6,8,10,13,21,23,24,25,27…) fall through to DW7 field interpretation. Schema doc says both games share the same 16-byte sub-header layout — reconcile with observed swap need. | `:1440-1474` | Hex-dump 5 non-text DQ4 sub-headers; extend type list only if evidence demands. |
| W-5 | Uncleared RAM windows on real HW: `[0x800BD070,0x800BD100)` (not loaded, not cleared) and `[0x800D9E80+blob, 0x800F4980)` (BSS end narrowed). DuckStation zero-fills; real hardware won't. Event counter `0x800F2C8C` lives there. | design | Acceptable for DuckStation phase; pad EXE file to 0x800BD100 with zeros; revisit before hardware testing. |
| W-6 | `update_iso_dir_reverse` writes HBD size field = `DQ4_HBD_SIZE` and LBA 362 — values already native on the DQ4 disc; writes are redundant but harmless. EXE entry size correctly set to final patched size (verify ≥ actual: BIOS loads per dir-entry size — appended tree/trampolines MUST be covered; current code passes `r.exe_size` captured AFTER all appends at `:2888` ✔). | `:2693-2720` | Keep; add assert dir-entry size == exe.size(). |
| W-7 | `verify_integrity` PC0 check expects `ORIGINAL_PC0` — will fail once MISS-1 PC0 redirect is added. | `:1744` | Update gate to accept documented stub PC0. |
| W-8 | Header constants stale for reverse: `CDROM_INTERCEPT_TRAMP_RAM=0x80101100`, `THREAD_ENTRY_SAFE_RAM=0x80101000` unused/misleading in reverse context (actual bases come from `set_tramp_base(0x800BCF00)`). | `psx_binary_ops.h:50,82` | Comment or split constants per-profile. |

**Verified-correct elements (no action):** trampoline branch offsets (all beq targets land on idx 22/20 correctly); `j 0x80056320` encoding; JAL 26-bit masking (post-fix); BSS start/end signed-immediate math (`0x800BD100`/`0x800D9E80`); disc-check surgical 9+1 set matches audited offsets; SYSTEM.CNF 11-char in-place rename; EXE-slot fit check (final ≈678 KB < 692,224 B); trampoline/intercept padding math; DQ4 disc copy leaves HBD untouched at 362.

---

## PART 5 — CLEAN-ROOM BUILD ORDER (BUILD-R2 spec)

**Inputs:** DQ4 disc (translated variant for shipping; JP for boot-bring-up), `translation/SLUSP012.06`, `frozen/dw7_hybrid_tree.bin` (interim only), `cybergrime/thread_code_blob.bin` (2048 B), `edcre.exe`.

**Phase A — EXE image construction (order matters):**
1. Load DW7 EXE; assert size == 0xA5000.
2. Append hybrid tree at file 0xA5000 (RAM 0x800BC700); pad to 0xA5800. *(Interim until R-TREEHOOK makes it obsolete.)*
3. `patch_tree_references(0x800F,0xF1C8 → 0x800C,0xC700)`.
4. BSS narrow: start→`0x800BD100`, end→`0x800D9E80` (existing two-site patch + re-patch).
5. `zero_pre_tree_bss()` (file 0xA4F68–0xA5000).
6. Write tree-ptr variable at **file 0xA4FF8** (BUG-1 fix) = tree base (+2 if uniform-offset resolved per BUG-4).
7. Decomp Target 4 via **2-instruction form** (BUG-2 fix), sites 0x571FC/0x57200 and 0x57244/0x57248.
8. Decomp Target 5 per BUG-4 resolution; Target 6 (`nop` @ file 0x1AFB8) unchanged.
9. **R-TREEHOOK** (BUG-3) when implemented — supersedes static var init.
10. Disc-check surgical 9+1 (existing offsets/values).
11. **FMV stubs** at file 0x737F4 and 0x753D0: `0x24020000, 0x03E00008, 0x00000000` (BAD-2 fix). NO Setmode param patch. NO cd_init force.
12. HBD type trampoline @ tramp_base 0x800BCF00 (existing).
13. ~~CD-ROM cmd intercept~~ — OMITTED (BAD-1). ~~CD-ROM stall bypass~~ — OMITTED for DuckStation (BAD-3).
14. EXE-side HBD name string @ file 0xF600: `w71` → `q41` (MISS-2; then SKIP ISO Q41→W71 rename).
15. Thread blob + boot copy stub + PC0 redirect (MISS-1). Record new PC0 in build manifest.
16. `patch_lba_references(354→362)` + sequential table (audit counts, W-2/W-3).
17. Assert final size ≤ 692,224 and dir-entry size will equal `exe.size()`.

**Phase B — Disc assembly:** copy DQ4 disc → write EXE @ LBA 24 → SYSTEM.CNF `SLPM_869.16→SLUSP012.06` → ISO dir: rename EXE entry + size=exe.size() (NO HBD rename) → PVD optional.

**Phase C — Finalize:** CUE (MODE2/2352) → edcre EDC/ECC (must exit 0; MISS-3) → SHA256 manifest.

**Phase D — Verification gates:**
- G1 static: re-read EXE from disc sector-by-sector (24-byte skip per sector — per Jul-22 phantom-bug lesson); verify all patch words, tree signature at 0xA5000, var at 0xA4FF8, stubs, PC0.
- G2 DuckStation smoke (`smoke_test_gold.py`): expect ≥1 Setloc with NO param-count errors, sectors read >0, execution reaches beyond `0x800967xx` poll loop, no jump to zeroed 0x800D9E80.
- G3 CyberGrime triage (secondary): OpenTh(0x800D9E80) → stub signature present (first word ≠ 0x00000000), ChangeTh proceeds.
- G4 visual: boot past disc check → menu/prologue render; then text-decode sanity (garbage text ⇒ revisit BUG-3/BUG-4).

**Open verification items before/while coding:** (V1) `parse_dq4` child-index convention → BUG-4 policy; (V2) DW7 dispatch+4 original instruction documented for the record (BAD-1 damage assessment on existing v4 binaries — rebuild, don't reuse); (V3) DQ4 HBD sectors 350-370 content roles (W-3); (V4) HBD sector-0 volume string offset 0x400 vs 0x740 (docs disagree) — only relevant if EXE checks it beyond the 0x80026D00 filename; trace `open_file` (0x80076040) compare loop to confirm.

---

## ONE-LINE STATUS
Reverse build pipeline exists and assembles a disc, but the EXE it produces cannot decode DQ4 text (NULL tree pointer + clobbered scaling instruction), still contains a self-defeating CD-ROM intercept, skips FMV the wrong way, and never installs the CD-ROM thread code — fix BUG-1/2, delete BAD-1, swap in FMV stubs, add MISS-1 blob injection, then re-run gates G1→G4.
