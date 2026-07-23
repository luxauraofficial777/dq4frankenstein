# DuckStation Bridge & C++ ROM Graft Verification Progress
## Jul 19, 2026 11:15 PM — Session 2

---

## Major Milestones This Session

### 1. C++ ROM Graft Verification Framework (NEW)
**Files:** `cybergrime/rom_graft_verifier.h`, `cybergrime/rom_graft_test.cpp`

Four verification pillars implemented and passing:

- **Pillar 1 — Deterministic Binary Packaging**: Sector alignment (302537 sectors, 0 remainder), LBA shift (+1), EXE signature at LBA 24, BSS range integrity, tree placement, address translation round-trip for 7 critical addresses
- **Pillar 2 — Surgical Memory & ROM Patching**: Post-patch verification confirms 0x2462CCC8 at EXE offset 0x076B90 → disc offset 0x096198
- **Pillar 3 — Integrity Checksum Validation**: CRC32 of EXE region (0x69A3A641), patch overlap detection
- **Pillar 4 — Closed-Loop Assertions**: Boundary collision check against protected regions (BIOS vectors, thread entry, tree dest, Huffman decompress) — no collisions

**Result: 23/23 checks PASS**

### 2. BSS Narrow Patch Applied at Build Time
**File:** `cybergrime/apply_bss_patch.py`

- Patched `dq4_frankenstein_v38a.bin` at disc offset 0x096198
- Original: `0x24634980` (addiu $v1, $v1, 0x4980) → BSS end 0x800F4980
- Patched:  `0x2462CCC8` (addiu $v1, $v1, 0xCCC8) → BSS end 0x800BCCC8
- Backup created: `dq4_frankenstein_v38a.bin.bak`
- Full disc SHA-256: `3d1f614feca0fd3c9e5a8167d71a4c45f25a0f08a760f1af2b32d1f313e33578`

### 3. DuckStation Bridge RAM Detection Fixed
**File:** `cybergrime/duckstation_bridge.py`

Root cause of RAM detection failure: **DuckStation MMap fastmem mode** splits the 2MB PSX RAM into pages with different protections, and the EXE header at offset 0x17F00 ends up in a `PAGE_EXECUTE_READ` page that isn't scanned.

**Fixes applied:**
- Switched DuckStation to **Interpreter mode** + **FastmemMode=Disabled** in settings.ini
- Expanded `_scan_regions` to include `PAGE_READONLY`, `PAGE_EXECUTE_READ`, `PAGE_WRITECOPY`
- Strengthened `_validate_ram_candidate`: now verifies PS-X EXE header fields (PC/SP must be in 0x80000000-0x80200000 range), checks MIPS opcodes at entry point, rejects UTF-16 text false positives, verifies full 2MB range is accessible
- Added fallback MIPS pattern search (BSS clear loop signature)
- Added `_search_region_for_pattern` for scanning large regions

### 4. Closed-Loop Controller — Full Pipeline Working
**File:** `cybergrime/closed_loop_controller.py`

End-to-end pipeline confirmed working:
1. DuckStation launches → RAM found (Interpreter mode)
2. CPU::Core state struct found
3. Telemetry captured (PC, registers, thread_entry, BSS instr, tree value)
4. Orchestrator correctly diagnoses **HF-001 (BSS_ZEROS_THREAD_ENTRY)**
5. Simulation PASSES (BSS narrow preserves thread entry + tree)
6. `write_mem` injection **SUCCEEDS** (VirtualProtectEx fix)
7. BSS patch confirmed at 0x8008E290 in live RAM
8. Stall persists (expected — BSS clear already executed before detection)

### 5. DuckStation Settings Fixed
**File:** `C:\Users\XFO777\AppData\Local\DuckStation\settings.ini`

- `SaveStateOnExit = false` (was causing stale state loads)
- `ExecutionMode = Interpreter` (was Recompiler — MMap fastmem prevented RAM scanning)
- `FastmemMode = Disabled` (was MMap)
- Save states deleted from `%LOCALAPPDATA%\DuckStation\savestates\`

---

## Critical Address Map (C++ Verified)

| Name | RAM Address | EXE File Offset | Disc Offset | Value (v38a patched) |
|------|------------|----------------|-------------|---------------------|
| BSS clear entry | 0x8008E284 | 0x076B84 | 0x09618C | 0x3C02800C |
| **BSS end instr** | **0x8008E290** | **0x076B90** | **0x096198** | **0x2462CCC8** ✅ PATCHED |
| BSS clear loop | 0x8008E294 | 0x076B94 | 0x09619C | 0xAC400000 |
| Thread entry | 0x800D9E80 | 0x0C2780 | 0x0F1D98 | (runtime: should be non-zero now) |
| Copy routine | 0x800BC700 | 0x0A5000 | 0x0D4598 | (BSS cleared — harmless) |
| Huffman decompress | 0x80041CE8 | 0x02A5E8 | 0x055A98 | 0x2404008D |
| Tree dest | 0x80100000 | 0x0E8900 | 0x117D98 | 0x240261AA |

**Disc offset formula:** `disc_offset = (EXE_LBA + exe_offset / 2048) * 2352 + 24 + (exe_offset % 2048)`

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VERIFICATION PIPELINE                             │
│                                                                      │
│  ┌──────────────┐  23/23 PASS   ┌──────────────┐  inject  ┌───────┐ │
│  │ C++ Verifier  │◄─────────────│  Patched Disc │─────────│DuckSta│ │
│  │ (4 pillars)   │              │  v38a+BSSpatch│         │tion   │ │
│  └──────────────┘              └──────────────┘         │Bridge  │ │
│       │                                               └───┬────┘ │
│       │ Pass                                              │      │
│       ▼                                                   ▼      │
│  ┌──────────────┐  telemetry  ┌──────────────┐  patch   ┌───────┐ │
│  │ Build Time   │             │ Orchestrator  │─────────│Closed │ │
│  │ Patch Applied│             │ (HF-001 match)│         │Loop   │ │
│  └──────────────┘             └──────────────┘         │Ctrl   │ │
│                                                        └───────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Files Created/Modified This Session

### New Files
- `cybergrime/rom_graft_verifier.h` — C++ verification framework (4 pillars, 600+ lines)
- `cybergrime/rom_graft_test.cpp` — Test harness
- `cybergrime/apply_bss_patch.py` — Build-time BSS narrow patcher with backup + checksums
- `cybergrime/rom_graft_test.exe` — Compiled verifier
- `dq4_frankenstein_v38a.bin.bak` — Backup of pre-patch disc

### Modified Files
- `cybergrime/duckstation_bridge.py` — RAM detection fixes (scan filter, validation, pattern fallback)
- `cybergrime/closed_loop_controller.py` — Retry logic for DuckStation launch
- `C:\Users\XFO777\AppData\Local\DuckStation\settings.ini` — Interpreter mode, save state disabled
- `dq4_frankenstein_v38a.bin` — BSS narrow patch applied at disc offset 0x096198

---

## Pending Work

1. **Test patched disc in DuckStation** — run closed-loop controller on patched v38a to verify thread entry is preserved after BSS clear
2. **Verify golden state** — thread_entry != 0, tree != 0, no stall, all boot phases hit
3. **If stall resolved** — declare BUILD READY after 3 clean cycles
4. **If new stall emerges** — feed new telemetry to orchestrator for next HF diagnosis
5. **Restore Recompiler mode** — after confirming the patch works, switch back to Recompiler for performance (need to add Recompiler-compatible RAM detection)
6. **Complete pipeline_audit.py** — run full 7-phase audit on patched disc
