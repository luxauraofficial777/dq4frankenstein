# Macro Audit: 48 GB, ~1,000 GPU-Hours, and What It Would Have Cost

**A measured accounting of a Dragon Quest reverse-engineering archive, the multi-agent AI compute that finished it, and the traditional engineering bill it replaced.**

> **On the numbers.** §1 is *measured* file-system fact (re-verified at audit time). §2 sizes the work delivered. §3 is a *Fermi estimate* with honest ranges — compute is inferred, not metered. §4–§5 price the effort on *scope* (what the work is), not on padded hour counts, and separate this project's real effort from the decades of community lineage it stands on. §6 is the actual point. Nothing here is dressed up as more certain than it is.

---

## 1. Repository Scale (measured)

| Metric | Value |
|---|---|
| **Total files** | **107,242** (measured Sep 2026; grows continuously) |
| **Total footprint** | **~48.6 GB** |
| `translation/` | 7.2 GB |
| `build/` | 7.9 GB |
| `snes/` | 1.5 GB |
| `study/` (forensic research library) | 1.3 GB (~1,340 files) |
| C / C++ / Assembly sources | ~12,000 |
| Python toolchain | ~4,500 |
| JSON databases | ~2,700 |
| Markdown technical docs | ~1,900 |

Re-measured directly at audit time and matching the repository's own prior counts to within normal growth. The bulk of the ~48 GB is **captured evidence** — disc images, RAM dumps, telemetry logs — not authored source; the authored intellectual layer (docs + tools) is a small fraction of the bytes.

---

## 2. What Was Actually Built (the scope being priced)

Not "a translation patch." From a 2001 Japan-only PlayStation binary with **zero official documentation**, the project delivered:

- **Ground-up reverse engineering** of Manabu Yamana's undocumented HeartBeat engine — MIPS R3000A disassembly, the HBD archive container, variable-bit Huffman codecs, dual-font blitters, the packed-referrer addressing scheme, and the CD-ROM sector/DMA pipeline.
- **A custom compiler toolchain** — a length-limited Huffman re-packer that fits verbose English inside the exact original Japanese byte budgets, a native referrer remapper, a type-39 script recompiler, and a hardware-accurate Mode-2/Form-1 disc mastering pass with real EDC/ECC regeneration.
- **A full localization** — 18,484 strings across 1,108+ blocks, with a documented 51-code control-token library and a master SID reference.
- **A shipped, hardware-accurate result** — a world-first playable English boot of DQ4 PSX on retail hardware, distributed CC0 as a tools-only patcher (no copyrighted assets), catalogued on RomHacking.net.

This is the kind of work a studio scopes as a specialist reverse-engineering + engine-tooling + localization program.

---

## 3. The AI Compute That Did It (Fermi estimate)

Five models — Claude Opus, Gemini, GLM-4, KIMI, Nemotron — driven by human directors across months. Producing the authored layer meant processing far more tokens than were kept (context, reasoning, and discarded drafts dominate):

| Quantity | Estimate | Basis |
|---|---|---|
| Tokens processed | **~500M – 800M** | kept output × context/reasoning/discard multiplier |
| Inference FLOPs | **~7×10²⁰ – 2×10²¹** | ≈ 2 × params × tokens |
| GPU-hours (A100/H100-eq) | **~1,000 – 1,500** | at typical utilization |
| Energy | **~hundreds of kWh to ~1 MWh** | ≈ a few weeks of one household's electricity |
| **Actual out-of-pocket compute** | **~a few hundred to a few thousand dollars** | subscription-rate inference, not enterprise API |

**Y2K reality check:** the same inference on a 2001 Pentium III (~1 GFLOP/s) would run to tens of thousands of single-core-years. The workload that is routine for a modern GPU cluster was *physically impossible* on the silicon of the era the game shipped in.

---

## 4. The Human Labor — named, and the part no compute could do

This is the half the efficiency numbers must not erase. The compute was the *commodity*; the people below were the *irreplaceable* input. Three carried the human load:

- **Markus Schroeder (Markus Projects) — the foundation.** From roughly 2020–2023, the first genuine reverse engineering of the `HBD1PS1D.Q41` archive structure, the early Huffman work, and the first working Java patcher proof-of-concept. Hundreds of hours of solitary hex-editor work before there was any momentum. Everyone who touched DQ4 PSX afterward started from the ground he dug.

- **Mandy Wilkens (dq4psxtrans) — the early bridge.** Seminal extraction tooling and early text work that turned a raw, undocumented archive into something a pipeline could actually consume.

- **Lux Aura (Zane O'Brien) — the architect and the director.** Conceived and built the Sovereign Native in-place engine, drove the concentrated 2026 sprint that produced the rebuild, toolchain, and forensic library — and supplied the one thing the whole project turned on: the human judgment that made the AI *productive instead of dangerous.* Every confident-but-wrong model output that never compiled into the disc was caught by a human reading the actual bytes. That is not supervisory overhead; it is the scarce, load-bearing input of the entire effort.

**Two honest measures of the labor:**
- *Direct effort on the modern rebuild:* low **thousands of hours** of focused human work by the people above.
- *The 25-year lineage it stands on:* many thousands more hours across the wider community — RadMage's Font-1 coordinate, decades of Dragon Quest reverse engineering, and **every prior DQ4 PSX translation attempt that tried and stalled against the HeartBeat engine's walls.** Those attempts were real labor, not failures of will: they hit genuine bit-phase desyncs, stale referrers, and buffer collisions, mapped the dead ends, and proved the wall was no myth. The project stands on their receipts as much as anyone's.

The story isn't that a machine did it. It's that three people — one who laid the foundation, one who built the early bridge, and one who architected the rebuild and refused to let the model's fluent errors reach the disc — steered AI at scale to finish, in months, what a quarter-century of skilled effort had repeatedly run at and couldn't cross. The AI accelerated their labor. It never originated it, it could not have replaced their judgment, and it does not diminish the many who tried before and dug the tunnel this far.

---

## 5. What a Traditional Studio Would Have Billed (scope-based)

Priced on the **work delivered** (§2) — a 3–5 person specialist team over 1–2 years, at real reverse-engineering / tooling / localization rates:

| Role | Rate band | Notes |
|---|---|---|
| Principal reverse-engineering architect | $185–$450 / hr | undocumented-engine RE, the rarest skill here |
| Systems / tooling engineers | $135–$275 / hr | codec, compiler, disc mastering |
| Cross-platform + localization | $70–$150 / hr | 18,484-string localization, dual-platform |
| Emulation / hardware QA | $55–$120 / hr | bare-metal verification |

**Scope-based equivalent:** roughly **$1M – $3M** at Y2K-era rates, and **~$3M – $7M** at modern premium reverse-engineering rates — for a full-time specialist team over one to two years. (Notably, this is a scope commercial studios *repeatedly declined to fund* for 25 years, judging the man-hours not worth it.)

---

## 6. The Point: The Efficiency Ratio

Line the two up:

```
Traditional engineering cost .......... ~$1M – $7M   (3–5 specialists, 1–2 years)
Actual AI compute cost ................ ~$hundreds – $few thousand
Actual energy ......................... ~hundreds of kWh (a few weeks of a house)
Actual wall-clock ..................... months, not years
```

A **multi-million-dollar-equivalent** specialist engineering program — reverse-engineering an undocumented 32-bit engine and building a compiler for it — was compressed into roughly **1,000 GPU-hours** and a small human team's months, at a cost **two to four orders of magnitude** below the traditional bill.

The honest caveat is the whole lesson: the compute was the *cheap, commodity* input, and a good fraction of what the models produced was confidently wrong. The scarce, un-billable ingredient was the human judgment that verified every claim against the actual bytes and threw out the hallucinations. AI didn't replace the engineers — it collapsed the *cost* of the labor while a human collapsed the *risk* of the machine. That pairing is what moved the price from millions to hundreds.

---

## 7. Conclusion

48.6 gigabytes, 107,242 files, ~1,000 GPU-hours, a few hundred kilowatt-hours, a small team, and a few months — against a traditional bill in the millions and a wall that stood for a quarter century. The measured scale is real, the compute was cheap, the human judgment was the expensive part, and the result boots on bare silicon. That is the whole receipt, and it is honest to the byte.

*ORDER. PRECISION. FIDELITY.*
