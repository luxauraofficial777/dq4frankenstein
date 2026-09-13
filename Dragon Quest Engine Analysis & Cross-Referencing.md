# Architectural Migration, Binary Reconstruction, and Generational Analysis of the HeartBeat Engine in Dragon Quest IV and VII

**Document ID:** ARCH-STUDY-MIGRATION-HBE-SFC-TO-PSX-20260913  
**Classification:** Deep-Dive Cross-Generational Engine Architecture & Systems Forensics  
**Author:** Lux Aura / VoidWalkers Reverse Engineering Group  
**Target Architectures:** Super Famicom (Ricoh 5A22 / 65C816) & Sony PlayStation (MIPS R3000A)[cite: 1]  
**Primary Binaries:** `SLPM_869.16` (DQIV PSX), `SLPM_865.00` / `SLUS_012.06` (DQVII PSX)[cite: 1]  
**Primary Archives:** `HBD1PS1D.Q41` & `HBD1PS1D.Q71`[cite: 1]  
**Date:** September 13, 2026  

---

**1. Architectural Divergence: Solid-State Bus vs. Optical Streaming**

The migration of HeartBeat's engine from a fixed-bank ROM cartridge architecture on the Super Famicom to a disc-based CD-ROM system on the Sony PlayStation required fundamentally restructuring memory models, pipeline execution, and asset retrieval while managing the PlayStation's 2 MB system RAM constraints.

| Architecture / Subsystem Metric | Super Famicom (SFC) Implementation | PlayStation (PSX) Implementation | Subsystem Divergence & Impact |
|---|---|---|---|
| **CPU / Hardware Co-Processors** | 16-bit Ricoh 5A22 (65C816 core) @ 3.58 MHz | 32-bit MIPS R3000A @ 33.8688 MHz + GTE & MDEC | Transitions from 8/16-bit accumulator math to 32-bit RISC pipeline and fixed-point vector coprocessing. |
| **Media Interface & Access Latency** | Solid-state cartridge (Direct bus mapping via HiROM/ExHiROM) | CD-ROM (ISO 9660 Mode 2 Form 1 raw sector streaming)[cite: 1] | Instantaneous single-cycle memory reads replaced by rotational disc seek latencies and sector buffering[cite: 1]. |
| **Physical System RAM** | 128 KB Work RAM ($7E0000–$7FFFFF) | 2 MB Main RAM (`0x80000000`–`0x801FFFFF`) | Moving from tightly packed 60-byte structs to partitioned resident engines and dynamic allocation heaps. |
| **State Tracking Architecture** | Bit-packed arrays and direct memory-mapped registers | Bit-packed flag matrices (`0x8000F800`–`0x80010000`) and structured VM variables | Preserves Boolean flag density while adapting dispatch mechanics to 32-bit word alignments. |
| **Asset Transfer Engine** | Direct DMA block transfers to VRAM during V-Blank | CD-ROM controller DMA Channel 4 to dynamic heap (`0x80138000`) | Replaces synchronous frame-bound transfers with asynchronous, multi-frame sector streaming loops. |
| **Game Loop Scheduling** | Hardware timers and vertical blanking NMI interrupts ($4210) | Root VSync callback handlers and cooperative thread scheduling loops | Replaces hardware interrupt polling with a software event loop coordinating input, scripts, and geometry. |

---

**2. System Memory Map: PSX 2 MB Partitioning Topology**

To support hybrid 2D sprite blitting on a 3D isometric terrain grid without memory fragmentation, HeartBeat partitioned the PlayStation's physical memory space into dedicated resident code blocks and dynamic streaming heaps.
