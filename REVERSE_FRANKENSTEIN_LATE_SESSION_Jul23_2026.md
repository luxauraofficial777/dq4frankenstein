# Reverse Frankenstein Session Progress — Jul 23, 2026 (Late Session)

## Session Summary

Implemented all fixes from the Cleanroom Reverse Frankenstein Blueprint, rebuilt the disc, and tested in both CyberGrime and DuckStation. The build is stable (no crash at 50M instructions) but does not boot past the BIOS screen in DuckStation.

## Fixes Applied This Session

### 1. EDC/ECC Regeneration (FIXED)
- **File**: `cybergrime/psx_binary_ops.cpp` lines 1297-1351
- **Problem**: `edcre.exe` was called with a relative path that broke after CWD changes. Exit code 1 (sectors updated) was treated as failure.
- **Fix**: Convert `edcre_path` to absolute path before `chdir` to CUE's directory. Treat exit codes 0 and 1 as success.
- **Result**: EDC/ECC regeneration now passes. `edcre.exe` exit code 1 = SUCCESS (sectors updated).

### 2. EXE text_size Not Updated After Blob Injection (FIXED — CRITICAL)
- **File**: `cybergrime/psx_binary_ops.cpp` line 3060-3063
- **Problem**: After appending the thread blob (2048 bytes) and boot copy stub (56 bytes) to the EXE, `exe.update_text_size()` was never called. The EXE header's `text_size` field at offset 0x1C remained `0x000A506C` (675,948 bytes) instead of `0x000A58A4` (678,052 bytes). The BIOS only loads `text_size` bytes into RAM, so PC0 at `0x800BD76C` (the boot stub) pointed to **unloaded memory**.
- **Fix**: Added `exe.update_text_size()` call after MISS-1 thread blob injection, before computing final `exe_size`.
- **Result**: EXE header now correctly shows `text_size=0xA58A4`. Boot stub is within loaded region. DuckStation log confirms boot stub executes (BSS clear runs from `0x800BD76C` through `0x801FF000`).

### 3. DuckStation BIOS Region (FIXED)
- **File**: `DuckStation/settings.ini` lines 156, 161
- **Problem**: BIOS search directory was `bios\JP` only. Console region was `Auto`, which detected the DQ4 base disc's Japanese volume ID (`DRAGONQUEST4_EN`) and loaded the Japanese BIOS (SCPH-5500). The DW7 US EXE expects the US BIOS.
- **Fix**: Changed `SearchDirectory` to `bios\US`. Changed `Region` to `NTSC-U`.
- **Result**: DuckStation now finds and uses `SCPH-5501` (US BIOS v3.0). Confirmed in log: `Using BIOS: SCPH-5501, 5503, 7003 (v3.0 11-18-96 A)`.

## Verification Results

### CyberGrime 50M Instruction Test
- **Status**: COMPLETE, no crash
- **Instructions**: 50,000,000
- **VBlanks**: 88
- **Last PC**: `0x800D9F3C` (thread blob + 0xBC)
- **GPU calls**: 0
- **CD-ROM commands**: 0
- **BIOS calls**: ~300+ A-table calls (fn=102 lseek, fn=104 read, fn=105 firstfile/nextfile)
- **HBD loading**: Game found `HBD1PS1D.Q41` at LBA 362 and started reading sequential sectors into RAM 0x8012F3C0-0x80139010
- **Stall reason**: Thread blob enters yield/wait loop at `0x800D9F34` after HBD load. CyberGrime HLE BIOS lacks thread scheduler — can't switch back to main thread. Emulator limitation, not a code bug.

### DuckStation Test (US BIOS, NTSC-U Region)
- **BIOS found**: SCPH-5501 (US v3.0) — confirmed
- **BIOS POST sequence**: 0F → 0E → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 — all passed
- **Kernel**: Initialized successfully
- **CD-ROM**: Read sectors 154-174 (SYSTEM.CNF + EXE header + beginning of EXE)
- **Boot stub**: EXECUTED — BSS clear runs from `0x800BD76C` through entire RAM to `0x801FF000`
- **Stall**: After BSS clear, `Instruction read failed at PC=0xBFC0E5E8` — falls back to uncached interpreter. FPS stays 0.00 indefinitely.
- **Disc region warning**: `disc region: NTSC-J, console region: NTSC-U`

## Current Disc State

### EXE Header (sector 24)
| Field | Value |
|-------|-------|
| Magic | PS-X EXE |
| PC0 | `0x800BD76C` (boot copy stub) |
| Load addr | `0x80017F00` |
| Text size | `0x000A58A4` (678,052 bytes) |
| Total EXE size | 680,100 bytes |

### SYSTEM.CNF (sector 23)
```
BOOT = cdrom:\SLUSP012.06;1
TCB = 4
EVENT = 16
STACK = 801ffffc
```

### ISO Directory (sector 22)
- `SLUSP012.06;1` — EXE at LBA 24
- `HBD1PS1D.Q41;1` — HBD at LBA 362
- `SYSTEM.CNF;1` — config at LBA 23

### PVD (sector 16)
- Volume ID: `DRAGONQUEST4_EN` (still Japanese — may need to change)
- System ID: `PLAYSTATION`

## Blueprint Implementation Status

| Fix | Status | Notes |
|-----|--------|-------|
| BUG-1: Tree pointer wrong file offset | DONE | `0xA4FF8` patched correctly |
| BUG-2: Target-4 clobbering sll + load-delay | DONE | Patch removed |
| BAD-1: CD-ROM intercept removed | DONE | |
| BAD-2: FMV skip via function stubs | DONE | 2 stubs at `0x8008AEF4` and `0x8008CAD0` |
| BAD-3: Self-contradicting integrity gate | DONE | Removed |
| MISS-1: Thread blob + boot copy stub + PC0 redirect | DONE | Blob at `0x800BCF6C`, stub at `0x800BD76C` |
| MISS-2: EXE-side HBD name string w71→q41 | DONE | No ISO rename, string patched in EXE |
| EDC/ECC regeneration | DONE | edcre exit code 1 = success |
| text_size update after blob injection | DONE | Critical fix this session |

## Remaining Issues

### 1. PVD Volume ID Still Japanese (PRIORITY: HIGH)
The disc's PVD still says `DRAGONQUEST4_EN`. DuckStation reports `disc region: NTSC-J, console region: NTSC-U`. The DW7 EXE's CD-ROM driver may check the disc region and refuse to proceed. 
**Potential fix**: Change the PVD volume ID to match a US disc (e.g., `SLUS_012.06` or `DRAGONWARRIOR7_1`).

### 2. BFC0E5E8 Instruction Read Failure (PRIORITY: HIGH)
After BSS clear completes, the CPU jumps to `0xBFC0E5E8` (BIOS ROM area) and fails to read an instruction. This is in the BIOS exception handler area. The US BIOS may be handling an exception that the DW7 EXE triggers during initialization. 
**Investigation needed**: Enable DuckStation TTY logging (`TTYLogging = true`) and check what exception is being raised. The boot stub copies the blob to `0x800D9E80` then jumps to `ORIG_PC0` (`0x8008E284`). The BSS clear that follows is part of the DW7 EXE's normal startup. The crash at `0xBFC0E5E8` happens after BSS clear, likely during the EXE's first real instruction after initialization.

### 3. HBD Format Incompatibility (PRIORITY: MEDIUM — Known Architectural Issue)
The DW7 EXE's HBD parser expects DW7-format HBD data (global Huffman tree at `0x800EF1C8`, `root_id = value & 0x7FFF`). The DQ4 disc has DQ4-format HBD (per-block trees, `root_id = (value & 0x7FFF) + 1`). The hybrid tree approach was implemented to bridge this, but the game may still fail to parse the HBD data after loading.

### 4. DuckStation FastBoot Option (PRIORITY: LOW)
Enable `PatchFastBoot = true` in DuckStation settings to skip the BIOS splash screen and go straight to the EXE. This may bypass region checks and get the game running faster.

## Key Files

- **Build tool**: `cybergrime/frankenstein_build.exe` (rebuilt this session)
- **Source**: `cybergrime/psx_binary_ops.cpp` (text_size fix at line 3060-3063, EDC/ECC fix at lines 1297-1351)
- **Disc output**: `dq4_reverse.bin` (368,057,424 bytes), `dq4_reverse.cue`
- **DuckStation settings**: `DuckStation/settings.ini` (BIOS path = `bios\US`, Region = `NTSC-U`)
- **DuckStation log**: `DuckStation/duckstation.log`
- **CyberGrime telemetry**: `cybergrime/telemetry/telemetry_reverse_cleanroom_50m.json`
- **Blueprint**: `study/CLEANROOM_REVERSE_FRANKENSTEIN_BLUEPRINT_Jul23_2026.md`

## Next Steps

1. **Change PVD volume ID** from `DRAGONQUEST4_EN` to a US-compatible identifier. This may fix the disc region mismatch that could be causing the CD-ROM driver to reject the disc.
2. **Investigate `0xBFC0E5E8` crash** — enable DuckStation's TTY logging and debug output to see what exception the game triggers after BSS clear.
3. **Test with FastBoot patch** — enable `PatchFastBoot = true` in DuckStation settings.
4. **Consider testing RC2 or Native DQ4 builds** in DuckStation — both use the DQ4 EXE (which can parse DQ4 HBD natively) and may boot successfully as a fallback path.
