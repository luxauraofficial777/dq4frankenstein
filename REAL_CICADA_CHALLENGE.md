# The REAL Cicada Challenge
## DQ4 PSX Reverse Translation — Frankenstein Build Black Screen Audit

## The Challenge

The REAL Cicada Challenge: boot a hybrid PlayStation disc that pairs Dragon Warrior 7's 32-bit engine (EXE) with Dragon Quest IV's Japanese game data (HBD), producing an English-translated DQ4 that runs on real hardware and emulators.

The disc **boots successfully** — BIOS POST completes, the EXE loads, HBD data is read from CD-ROM, and the game runs at a steady 59.82 FPS. But the screen is **black**. No Enix logo, no title screen, no visual output.

## What Works

- **BIOS boot sequence:** POST 0F→0E→01→02→03→04→05→06→07→kernel init — all pass
- **EXE loading:** Sector-aligned text_size (0xA6000), 681,984 byte file, PC0 redirected to boot stub
- **Thread blob injection:** Boot stub copies thread code to 0x800D9E80 before BSS clear
- **CD-ROM reads:** SYSTEM.CNF, EXE header, HBD sectors all read successfully
- **Disc check bypass:** 10 surgical patches — game doesn't reject the hybrid disc
- **HBD access:** LBA references patched (354→362), HBD string patched (w71→q41)
- **Huffman tree patches:** 24 tree reference sites patched for DQ4 node layout
- **Decompressor patches:** Tree pointer, root_id+1, offset_b mask — all applied
- **Region compatibility:** License sector NTSC-J→NTSC-U, PVD volume ID matched
- **Frame rate:** Solid 59.82 FPS — no crash, no hang, no infinite loop

## Current Blockers

### Blocker 1: Black Screen — No GPU Activity

**Symptom:** DuckStation log shows `GPU: 0.00` for the entire session. The game runs at 60 FPS but never renders a single frame.

**Root Cause (suspected):** The FMV stubs (XA/STR at 0x8008AEF4, MDEC init at 0x8008CAD0) return 0 ("success") but the game's main loop waits for an async completion signal that the FMV system would normally provide. The stubs skip the FMV but never trigger the event/flag/callback the game expects to proceed to the next state (title screen / Enix logo rendering).

**Evidence:**
- Line 955 of duckstation.log: `Setmode 0xA0` — engine enters FMV mode (double-speed ADPCM)
- After Setmode, no further CD-ROM reads, no GPU activity, just steady 60 FPS idle
- No seek to LBA 146621 (DW7 FMV position) — stubs prevent the seek
- GPU display enable (GP1 write) was added to boot stub but didn't help — the game never sets up a frame buffer or drawing environment

### Blocker 2: DQ4 Disc Has No FMV Data

The DQ4 Japanese disc contains only 10 XA/STR sectors (all in the system area). The Enix logo on DQ4 is likely real-time rendered, not FMV. The DW7 engine, however, expects FMV data at LBA 146621 — that location on the DQ4 disc is all zeros.

This means:
- We cannot simply "let the FMV play" — there's no FMV to play
- We cannot redirect to DQ4's Enix logo — it's not FMV data
- We must either skip FMV entirely AND trigger the game's next state, or find what DQ4's native boot sequence does instead of FMV

### Blocker 3: Unknown Game State Machine

The DW7 engine has a state machine for boot: FMV → title screen → main menu → game. We're stubbing the FMV state, but the state transition mechanism is unknown. It could be:
- A VBLANK callback registered during FMV init
- A DMA completion interrupt
- A polling loop on a status variable
- A thread event flag (the game uses multithreading via the thread blob)

Without understanding the state transition, we can't patch it.

## The 25-Step Pipeline

The `build_reverse_option_a` function in `psx_binary_ops.cpp` performs approximately 25 distinct operations:

1. Load DW7 EXE from disc
2. License sector patch (NTSC-J → NTSC-U)
3. PVD patch (volume ID → DRAGONWARRIOR7_1)
4. Tree reference patches (24 sites: 0x800F → 0x800C LUI/ADDIU)
5. BSS clear start patch (tree append shifted BSS start)
6. BSS clear end patch (thread-safe end for thread blob)
7. Zero pre-tree BSS region
8. Tree pointer variable init (0x800BC6F8 = tree_base+2)
9. Decompressor tree pointer patch (sites 1&2: load from variable)
10. Decompressor root_id+1 patch (site 3: +2 node offset)
11. Decompressor offset_b mask patch (remove FFFE mask)
12. Disc check bypass (10 surgical patches)
13. CD-ROM stall bypass (SKIPPED — DuckStation mode)
14. HBD type trampoline (at 0x800BCF00)
15. FMV stubs (2 sites: XA/STR + MDEC init)
16. CD-ROM command intercept (SKIPPED — removed BAD-1)
17. HBD string patch (w71 → q41 at file 0xF600)
18. Thread blob loading (thread_code_blob.bin)
19. Thread blob append to EXE (4-byte aligned)
20. Boot copy stub (18 instructions: copy blob + GPU enable + jump to ORIG_PC0)
21. PC0 redirect (0x8008E284 → 0x800BD76C)
22. Sector-aligned padding
23. Text size update in EXE header
24. LBA reference patch (HBD 354 → 362)
25. Sequential sector table patch (44 entries)

With this many steps, a single bug in any step could cause the black screen. The next action is a full pipeline audit.

## What We Need to Find

1. **Which step has a bug?** — Is a patch writing to the wrong offset? Is a value off by one? Is a skipped step actually needed?
2. **What does the DW7 native boot do that we don't?** — Compare a working DW7 boot log with our hybrid boot log
3. **What state does the game expect after FMV?** — Disassemble the post-FMV code path to find the state transition
4. **Is the thread blob correct?** — The thread code at 0x800D9E80 drives CD-ROM and event handling. If it's wrong, events don't fire.

## Key Files

- `cybergrime/psx_binary_ops.cpp` — Build pipeline (build_reverse_option_a)
- `cybergrime/psx_binary_ops.h` — MIPS macros, constants, struct definitions
- `cybergrime/frankenstein_build_main.cpp` — Build driver / CLI
- `cybergrime/thread_code_blob.bin` — Thread code injected into BSS
- `DuckStation/duckstation.log` — Boot log (1021 lines)
- `dq4_reverse.bin` / `dq4_reverse.cue` — Output hybrid disc
- `Dragon Quest IV - Michibikareshi Mono Tachi (Japan).bin` — DQ4 JP source disc
- `Dragon Warrior VII (USA).bin` — DW7 US source disc (EXE donor)

## Community

This challenge has been opened to the community. The groundwork is laid — 25-step pipeline, boot verified, CD-ROM working. The remaining issue is a state machine / event signaling problem in the FMV skip path. Good hunting.
