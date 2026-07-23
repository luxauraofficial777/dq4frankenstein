# v33 Boot Debug Progress — Jul 18, 2026

## Current Status
- **Disc**: `dq4_frankenstein_v33.bin` (+ .cue)
- **Problem**: DuckStation fails to boot — `Instruction read failed at PC=0x801005C8`
- **Root cause hypothesis**: DuckStation may not have RAM mapped at `0x80100000` because it falls outside the EXE's declared load range

## v33 Memory Layout (Verified On-Disc)
- EXE load range: `0x80017F00` - `0x800BCF00` (t_addr=0x80017F00, t_size=0xA5000)
- Tree: `0x80100000` (1480 bytes)
- Trampoline: `0x801005C8` (40 bytes)
- Copy routine: `0x800BCCF0` (PC0, 52 bytes) → copies 1520 bytes from `0x800BC700` to `0x80100000`
- Copy routine end: `j 0x8008E284` (original PC0, BSS clear)
- BSS clear range: `0x800BC668` - `0x800F4980` (code-level, not header-driven)
- Game data section: `0x800F4980` - `0x800F9E54+`
- Dynamic heap (per DW7 docs): `0x80138000` - `0x801FFFFF`

## Key Finding from PDF Analysis
The DW7 memory map shows `0x80100000` falls in an **unmapped gap** between:
- EXE load range end: `0x800BCF00`
- Dynamic heap start: `0x80138000`

No game code or engine subsystem references this range. DuckStation may only map RAM for the EXE's declared `t_addr`/`t_size` range, causing writes to `0x80100000` to be silently dropped.

## PSX Memory Map (from PDFs)
| Range | Classification |
|-------|---------------|
| `0x80000000-0x8000F7FF` | Kernel & system core |
| `0x8000F800-0x800252FF` | Global flag space |
| `0x80025300-0x800258E8` | SEQq sound tables |
| `0x8002FA10-0x80036FCC` | Sound interrupt thread |
| `0x80076040` | open_file (CD-ROM) |
| `0x80082598` | mopen_file (async loader) |
| `0x80017F00-0x800BCF00` | EXE load range (v33) |
| `0x800BC668-0x800F4980` | BSS clear range |
| `0x800F4980-0x800F9E54+` | Game data section |
| `0x80100000` | **UNMAPPED GAP** (v33 tree target) |
| `0x80138000-0x801FFFFF` | Dynamic heap (volatile) |

## Verification Completed
- `verify_tramp_v33.py`: Confirmed tree, trampoline, copy routine bytes on disc
- `check_exe_load.py`: Confirmed PC0=0x800BCCF0, SYSTEM.CNF, EXE header
- `disasm_pc0.py`: Disassembled copy routine and original PC0
- `verify_v33.py`: Confirmed addresses avoid BSS clear and game data regions

## Next Steps
1. Investigate DuckStation source (`study/duckstation-src/`) for RAM mapping behavior
2. Use CyberGrime emulator to trace copy routine execution and memory writes
3. If `0x80100000` is unmapped, relocate tree to within EXE load range or expand t_size
4. Alternative: place tree after BSS clear completes (hook into post-BSS-clear path)

## CyberGrime Debugging
- CyberGrime smoke test already passed (5M instrs, no crash, PC=0x80096650)
- Need to set watchpoints on `0x80100000` and `0x801005C8` to trace writes/reads
- Use `psx_agent_runner` with `--wp` flag for watchpoint on trampoline address
- Check if copy routine at `0x800BCCF0` executes and successfully writes to `0x80100000`
