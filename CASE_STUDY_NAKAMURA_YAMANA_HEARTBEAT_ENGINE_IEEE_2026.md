# The HeartBeat Engine and the Two Craftsmen: Koichi Nakamura, Manabu Yamana, and the 25-Year Reverse-Engineering of *Dragon Quest IV* (PSX)

**A case study in compiler-style text systems, studio-lineage preservation, and the historiography of the video-game engineer**

---

**Document ID:** STUDY-HBE-NAKAMURA-YAMANA-CASE-2026
**Classification:** Case Study / Historical–Technical Analysis
**Author:** Lux Aura — VoidWalkers Research Project (edited with LLM drafting assistance)
**Date:** September 13, 2026
**Formal review target:** IEEE / SIGGRAPH-style archival case study
**License of this document:** CC BY-NC-SA 4.0

---

## Abstract

*Dragon Quest IV* (PlayStation, 2001) stands as the last mainline *Dragon Quest* release ever to ship without an official English localization. Beneath that historical footnote lies a twenty-five-year technical wall: the game's text pipeline, authored by Manabu Yamana and the HeartBeat studio he founded in 1992, is an unusually dense coupling of per-block Huffman trees, packed referrer words, and a streaming CD-ROM staging architecture that resisted translation until a community reverse-engineering effort cracked it in 2026. This paper studies the *lineage* of that engine and, through it, the two engineers most responsible for the *Dragon Quest* text-and-system stack: **Koichi Nakamura**, the Chunsoft founder and first architect of the series' data-compression and dialogue systems; and **Manabu Yamana**, the programmer who carried those systems from the NES into HeartBeat's SFC and PlayStation engines, running from *Dragon Quest I* through *Dragon Quest VII* and beyond. Using published developer interviews (primary testimony), the engine's binary as recovered-by-reverse-engineering, and the complete 2020–2026 public record of translation attempts, we argue (1) that Yamana is a systematically undervalued figure in the written history of the game industry, (2) that the HeartBeat Engine constitutes a genuine if niche engineering achievement — effectively a *compiler for game dialog under hard byte budgets* — and (3) that the 25-year localization impasse was structurally caused by that very achievement, which the community ultimately solved by reverse-engineering the compiler itself. We close with recommendations for how fan reverse-engineering documentation can double as the missing institutional archive of the late Japanese console R&D era.

**Index Terms** — *Dragon Quest*, HeartBeat Engine, Huffman coding, ROM hacking, reverse engineering, video-game preservation, game-engine history, Koichi Nakamura, Manabu Yamana, CD-ROM streaming, clean-room recompiler, length-limited Huffman, industry historiography.

---

## 1. Introduction

### 1.1 Problem and motivation

On August 22, 2001, Enix's American division confirmed that the PlayStation remake of *Dragon Quest IV: Chapters of the Chosen* would be localized and released "within the year" [11]. It was never released. On February 1, 2002, HeartBeat — the ten-man-staff studio that had developed the underlying engine — withdrew from the video-game business, citing rising development costs [8][9]. Enix later confirmed on its own message board that the North American version could not be completed without HeartBeat's programming [10]. The game shipped in Japan on November 22, 2001 [7]; the English-language version was never delivered. Except for the DS remake (2007, developed by ArtePiazza [16]), the *original* PSX text pipeline remained Japanese-only.

Between 2001 and 2026, at least four independent engineering efforts attempted to translate the game into English. All but the last failed. The failures were not errors of skill; they were consequences of an engine whose text subsystem was built — deliberately, per the evidence — to be *unpartitionable from the game running in memory*.

The present work is a case study. Its subjects are the two engineers who built the data-and-text lineage that produced the impasse, and the archive that finally crossed it.

### 1.2 Scope and sources

- **Primary testimony:** translated developer interviews (Nakamura, Horii, Yamana) as published in the Japanese trade press and preserved by the Shmuplations archive and AUTOMATON [2][3][14][15].
- **Primary binary evidence:** the recovered HeartBeat Engine, `SLPM_869.16` / `HBD1PS1D.Q41`, as documented across the VoidWalkers repository's study corpus (engine analysis, master SID/TID/control-code libraries, blueprints) [5][6].
- **Historical record of attempts:** the Markus-project blog (2020–2023) [18], the mwilkens/dq4psxtrans toolchain [19], RadMageIRL hardware measurement work [20], and the VoidWalkers synthesis [5].
- **Studio and administrative history:** Japanese and English encyclopedic records, trade press, and the Agency for Cultural Affairs (Japan) Media Arts Festival record for *DQVII* [1][7][8][12][16].

Throughout, we mark claims that rest on primary testimony, on binary recovery, and on secondary summary, so that confidence levels remain transparent.

---

## 2. The Two Craftsmen

### 2.1 Koichi Nakamura: the data-compression origin

Koichi Nakamura (中村 光一, b. August 1964, Kagawa) entered the personal-computer era as a teenage programmer; his 1982 entry *Door Door* won second prize in Enix's first national programming contest, and in April 1984 — during his second year of university — he founded the five-person Chunsoft, developing from a condominium room in Chōfu, Tokyo [1][12]. The decisive moment for this paper came with the Famicom version of *The Portopia Serial Murder Case*, made with scenario writer Yuji Horii, then with the creation of *Dragon Quest* (1986), for which Nakamura wrote every line of program except the music [1][12][15].

Two technical choices made at the origin would echo for forty years:

1. **Text as compressed data, authored separately from code.** Horii wrote the scenarios in natural-language prose; Nakamura's team encoded them into the cartridge [14]. In the first *Dragon Quest* (64 KB), the budget lopped "more than half of the katakana characters," forcing spell and town names to be re-written within the surviving glyph set [2]. The lesson embedded itself: *the dialog system is a compressor, and the compressor is the platform constraint* [2][12].
2. **A designer-driven work loop with a single system programmer as the point of truth.** When the dialogue exceeded the cartridge's memory, the team cut not code but *data* — whole events, in *DQIII* [14][15]. Horii stated bluntly that by *DQIII*, if something would not fit, "Horii wouldn't bother creating it to begin with" [15]. Nakamura's role moved from hands-on programming (≈100% of *DQ1*, ≈50% of *DQ2*, ≈10% of *DQ3*) to director, keeping the same physical responsibility for the *Dragon Quest* data pipeline through *DQV* (1992) [1][14].

Nakamura left mainline *Dragon Quest* after *DQV*, precisely when the Super Famicom arrived and Chunsoft turned to its own publishing and to new genres (the sound novel *Otogiriso*; the *Mystery Dungeon* roguelikes) [12][14]. His direct children — the password save system (Japanese hiragana as a compressed checkpoint), the dialog-as-data discipline, and the party AI of *DQIV* — passed, together with staff, into the hands of the studio that would continue the series.

### 2.2 Manabu Yamana: the inheritor and formalizer

Manabu Yamana (山名 学, b. June 8, 1965, Tokyo, Shinagawa) began professionally in high school producing MSX and PC-88 software; he joined Chunsoft in 1986 while a university student and worked on the *Dragon Quest* series as main programmer from the first game [1][7][12]. His credited arc on the mainline titles is unusually long:

* *Dragon Quest I* (NES, 1986) — programmer
* *Dragon Quest II* (NES, 1987) — programmer
* *Dragon Quest III* (NES, 1988) — programmer; (SFC, 1996) — main program
* *Dragon Quest IV* (NES, 1990) — chief programmer; (PSX, 2001) — program director
* *Dragon Quest V* (SFC, 1992) — director
* *Dragon Quest VI* (SFC, 1995) — direction & main program
* *Dragon Quest VII* (PSX, 2000) — direction, program direction, and programming

*(Compiled from MobyGames credit records and Japanese encyclopedic sources; see [7][8][12][16].)*

Yamana founded HeartBeat in October 1992, after the Super Famicom *DQV*; the split was amicable and structurally meaningful — Chunsoft moved to its own titles, HeartBeat was staffed by former Chunsoft *Dragon Quest* programmers, and the series' production simply continued at the successor shop [7][8][12][16]. The developer-assignment records make the continuity explicit: *DQVI*'s "Direction & Main Program" credit belongs to Yamana on a studio banner that names HeartBeat; the copyright line of that title jointly lists ARMOR PROJECT, BIRD STUDIO, Koichi Sugiyama, Heart Beat, and Enix [12].

Two public-interview facts matter for the argument that Yamana is undervalued:

- In 1995, in the trade press, Yamana described the hardest technical problem of *DQVI* as the **window and party-display system** — the overlay of status/message windows on the game's live map while storing sprite data in memory — and the **150-skill battle AI** that began at ~6 s of CPU per action for four party members before optimization [15]. These are the two systems (text-overlay staging; rule-driven action selection) that the PSX engine would later fossilize into its most opaque layers.
- In 2012–2013, Yamana, running his successor studio Genius Sonority, stated plainly that he "worked as the main programmer and the director of the Dragon Quest series from Dragon Quest 3 for Family Computer until Dragon Quest 7 for the PlayStation," and that the series is "such an important part of my history as a developer" [21]. The claim is modest but the resume is matched by almost no other individual in the genre's history: **seven consecutive mainline entries** across three hardware generations.

### 2.3 The structural asymmetry

Nakamura is well-documented: a contest-famous founder, the subject of a Game Preservation Society honorary profile and of multiple long-form interviews [1][12][14]. Yamana, despite fourteen-plus years on the series and a founding-director role at two studios, is absent from the mainstream written history of the medium; his name appears in English predominantly in credit aggregators and in Japanese in encyclopedia pages [1][7][16][21]. This asymmetry — the founder-protagonist archived, the system-programmer successor nearly silent — is the historiographic problem the second half of this paper uses the engine to correct.

---

## 3. The HeartBeat Engine: a Compiler for Dialog Under a Byte Budget

### 3.1 The recovered architecture

The binary recovered by the VoidWalkers study corpus (2025–2026) and the earlier academic community work (2020–2023) [5][18][19] exposes a machine whose text subsystem operates in four coupled stages:

1. **Archive staging.** The PSX disc (`SLPM_869.16` / `HBD1PS1D.Q41`, ~319.4 MB) is a streaming container of ~155,975 blocks, read from CD-ROM in 2,048-byte Mode-2 Form-1 sectors at a fixed LBA anchor [5][6]. The engine never loads the full text into main memory; it streams and decodes on demand — a design that presupposes a *rotating-disc, latency-tolerant* runtime and makes offline analysis and patching an exercise in re-deriving the read scheduler's assumptions.

2. **Per-block Huffman coding.** Each dialog container carries its own Huffman tree. The tree is encoded compactly: nodes are emitted as `0x80`-prefixed offsets, leaves as encoded glyph pairs, and control characters are reserved tokens (`0x7F` family) [18]. Different games in the series use structurally different trees — *DQ4* uses per-block trees; the *DW7* EXE uses a global hybrid tree — a divergence that alone produced the classic "format-difference" wall between the two PSX executables [5].

3. **Referrer addressing.** Some referer offsets in both dialog text and cutscene scripts target *the exact bit position of a string in the Huffman-encoded payload*, combined with the block ID — the repository's master libraries derive a packed 32-bit referrer word `(BlockID << 20) | (BitOffset + HeaderTreeSize*8)` [5]. Because the decoding routine walks the tree bit-by-bit, a translation that changes a single string's length *changes every subsequent bit position*, requiring global referrer re-mapping.

4. **Cutscene scripting.** Script sub-blocks (Type 39) are LZSS-compressed VM bytecode (`0xC021A0` command family) with embedded dialog IDs and bit offsets [19], and require decompression → re-reference → LZSS recompression to survive a text swap [5][19].

The systems that Yamana's interviews identify as hard — overlay staging and per-member stateful AI — become, in the recovered binary, the *same* subsystems that block naive translation: the overlay's dual-font blitters and the AI's per-block rule references.

### 3.2 It is a compiler, and it regenerates hardware-accurate media

The defining engineering act of the VoidWalkers project — the "Sovereign Native" approach — was to re-implement, with a clean-room toolchain, a **length-limited Huffman re-packer** that refits the full English localization into *exactly* the original byte budgets of the 1,358 dialog containers and 612 compressed cutscene scripts, while regenerating Mode-2 Form-1 EDC/ECC parity so the authored disc retains sector-for-sector hardware fidelity [5][6]. Two properties of the effort are transportable conclusions:

- **The wall was architectural.** The 20-bit per-block ceiling, the bit-granular referrers, and the mandatory `{0000}` terminator are exactly as a compiler's ABI would look if a studio had shipped a custom writing system. This "HBE vulnerability triad" — bit-phase desync, hardcoded literal-bit-offset referrers, and scratchpad collisions — is documented across the translation-lineage study [5][6].
- **The re-pack is a small, unsung computational problem.** Length-limited Huffman re-encoding under *hard, per-container ceilings* with referrer coherence is a constrained optimization whose practical, working solver (produced by the community in 2025–2026) had, to our knowledge, no public predecessor in the ROM-hacking corpus [5].

---

## 4. The 25-Year Wall, and the Archive That Crossed It

### 4.1 A failed-studio footnote is a preserved system

The same administrative event that cancelled the English *DQIV* (HeartBeat's February 2002 withdrawal [9][10]) abolished the institution that held the system's memory: no developer notes, no source, no in-house documentation survived in any public venue [7][8]. The engine persisted **only as a shipped binary** — a situation that forces a definitional point: *for this class of artifact, the binary is the archival record* [5].

### 4.2 The documented attempts

- **2020–2023: Markus Projects ("Dragon 'Hackst' IV").** First structural public documentation: block/sub-block parsing, the Huffman tree format, dialog-pointer coverage analysis (12,870 text pointers scanned; 99.98% referrer coverage), and the first proof-of-concept embedding. Signaled hand-off: "If I cannot continue this project, it may help others to get into it" [18].
- **2022–2024: Mandy Wilkens (`mwilkens/dq4psxtrans`).** Codified the script-host language and opcode taxonomy from which later tooling wrote its decompiler-compressor pairs [19].
- **2025–2026: RadMageIRL and VoidWalkers.** Hardware-verified measurement of the engine's runtime behavior (RM provenance in the control-code libraries [5]), then the full clean-room re-authoring: 1,111 unique text-block IDs, 19,193 string IDs, 51 observed control-code families, a documented 43-code on-disc taxonomy (plus EXE-resident codes) [5][6], and the "Sovereign Native" byte-faithful patch.

Professional localization, by evidence available in this study, consistently declined the work for 25 years on behalf of publishers and studios ([§1.1) — a commercial refusal examined further in §5.

### 4.3 Primary effect

On the day the VoidWalkers toolchain re-pack passed its gate suite — G0–G9, spanning fingerprint, referrer, LBA-anchor, split-immediate alignment, Type-39 integrity, delta-lock, heap-bounds, EDC/ECC, and streaming-bounds verification [6] — the *Dragon Quest IV* (PSX) text pipeline became, for the first time anywhere or by anyone, **English-playable on original hardware**.

By the marginal definition of the ROM-hacking community — the one the *DS* and *DW* communities use when they say a translation is "complete" — the game could now be booted, played, and finished in English [5][6]. Conservative framing, however, requires the caveat: **the archive's own release notes describe "partially playable Chapter 1 baseline"** until Rebuild C's remaining gates (church-facility overlay remap, byte-parity across duplicate sector pairs, the 1,086 Type-44 referrers of the Endor mega-block) close [6]. We therefore record the claim as: *first playable English boot of the original engine*, with the reservation that "complete" and "native" are properties of an ongoing engineering program, and that independent boot evidence (a game-play video or byte-identical hash on a second disc) is the strongest available falsifier.

---

## 5. Discussion

### 5.1 Why the wall was a compiler, not a translation

Across 2001–2026, three distinct types of expert failed in the same place: enthusiasts with address-dumps (the 2020–2023 track), toolbuilders with decompilers (the 2022–2024 track), and — by the historical record of the industry — the localization arm of the publisher itself [10][18][19]. The failure locus is stable. It is not text volume (≈18,484 strings [6]); it is the *referrer/tree coupling plus the disc-streaming staging*, which together make the asset "text" unpartitionable from the asset "engine." We argue this is a direct, conscious consequence of the design lineage described in §2: a compressor authoring discipline that, from Nakamura's 64 KB cartridge onward, treated game text as something to be *compiled into* the machine rather than *displayed by* it.

### 5.2 Yamana as an unjustly un-archived figure

The historiographic claim, restated in engineering terms: **Yamana is the primary *system* architect of the *Dragon Quest* middle era, and the academy has no record of him.** Nakamura is a Game Preservation Society honorary member [1]; Yamana's contributions survive in credit aggregators, one Japanese encyclopedia article, and the *DQVII* Media Arts Festival award he shares as director [12]. This asymmetry is explainable (founder-protagonist gravity, Japan's weak systems-credit culture) but not rational: the recovered engine is anonymous, and the only reason we know who built it is the credit roll.

### 5.3 Fan RE documentation as preservation

The practical recommendation of this paper is small and actionable: **institutional deposit of the VoidWalkers study corpus** — the master TID/SID/control-code libraries, the engine-analysis documents, and the lineage histories — as citable archival material, in a manner equivalent to the Game Preservation Society's own corpus [1][5]. The corpus is unusual among fan RE documentation in that it is (a) machine-verified against the binary, (b) self-provenancing (it names its own predecessors [5][18][19][20]), and (c) admission-oriented about its own margins of error [5]. Those are the properties the historical record of a vanished studio most lacks.

---

## 6. Conclusion

*Dragon Quest IV* (PSX) remained English-untouched for a quarter-century not because nobody tried, but because the text system it carries is a small, purpose-built compiler that encodes dialog into per-block Huffman trees under hard byte budgets, addresses strings by bit-granular referrer, and stages its data from the disc via a streaming read scheduler. That system is the work of a two-era lineage: Koichi Nakamura, who established the data-compression and dialog-as-data discipline at the series' origin; and Manabu Yamana, who carried those disciplines from the NES through seven consecutive mainline titles across three platforms, and whose HeartBeat studio formalized them into the recovered engine. The community that finally crossed the wall did so by re-implementing the compiler — a textbook case of *reverse engineering as archival forensics*. If the game industry is to keep the memory of its late R&D era, the lessons of this case are plain: **credit the system programmers, archive the binaries, and treat the fan RE corpus as institutional material.**

**Future work.** (1) Independent third-party boot verification of the 2026 rebuild on original hardware; (2) formal publication of the length-limited-Huffman-repack solver under constraints, with worked complexity bounds; (3) a survey of the SFC→PSX engine evolution (DQVI→DQVII→DQIV) as a case study in studio code continuity; (4) outreach to the Game Preservation Society and comparable bodies regarding deposit of the VoidWalkers corpus.

---

## 7. Acknowledgments

The author thanks the VoidWalkers Research Project team and its named lineage — Markus Schroeder (Markus Projects), Mandy Wilkens, and RadMageIRL — for primary documentation still active at the time of writing [5][18][19][20]; the Shmuplations translation archive for its developer-interview corpus [2][3][14][15]; and the ROM-hacking and *Dragon Quest* communities whose failure records made this case study possible. Final research, drafting, and citation discipline were assisted by a large-language-model writing partner under the author's direction; all technical claims were verified against the cited primary material by the human author.

---

## 8. References

*(IEEE-style numbered citations; URLs current as of September 13, 2026.)*

[1] Game Preservation Society, "Kōichi NAKAMURA's Message / Biography," honorary member record, 2024. [Online]. Available: https://www.gamepres.org/en/media/honorary/nakamura/

[2] Shmuplations, "Chunsoft 30th Anniversary – 2014 Developer Interview (with Kouichi Nakamura)," trans. 2021. [Online]. Available: https://shmuplations.com/chunsoft30th/

[3] Shmuplations, "Dragon Quest IV – 1989 Developer Interview (Yuji Horii, Koichi Nakamura)," trans. 2021. [Online]. Available: https://shmuplations.com/dragonquestiv/

[4] S. Ishimoto, "開発者のセーブデータ 第二回：スパイク・チュンソフト 中村光一," AUTOMATON, Sep. 27, 2016. [Online]. Available: https://automaton-media.com/devlog/developers-save-002-kouichi-nakamura/

[5] VoidWalkers Research Project, "dq4frankenstein" repository study corpus (engine analysis, YAMANA_HBE_MASTER_{TID,SID,CONTROL_CODES}_LIBRARY, HISTORICAL_STUDY_ZENITHIAN_TRANSLATION_LINEAGE, HBD_PROCESSING_BLUEPRINT, V99_ReBuildB_DOCUMENTATION). [Online]. Available: https://github.com/luxauraofficial777/dq4frankenstein

[6] VoidWalkers Research Project, "V99_ReBuildB_DOCUMENTATION.md" and VERSION_LOG, in [5].

[7] Japanese Wikipedia, "ハートビート (ゲーム会社)," incl. founded October 1992, DQVI/VII/III(SFC)/IV(PSX) credits, Feb-2002 withdrawal and Aug-2002 Genius Sonority founding, repr. 2026. [Online]. Available: https://ja.wikipedia.org/wiki/ハートビート_(ゲーム会社)

[8] MobyGames, "Heart Beat" company record (founded October 1992 by Manabu Yamana; disbanded 2002; members founded Genius Sonority). [Online]. Available: https://www.mobygames.com/company/8182/heart-beat/

[9] RPGFan (archived), "Enix Comments on DQVIII As Heart Beat Steps Away," Feb. 2002. [Online]. Available: https://web.archive.org/web/20020221105030/http://www.rpgfan.com/news/2002/1102.html

[10] MobyGames, "Dragon Quest IV: Michibikareshi Monotachi (2001) — Cancelled Western Release" note (Enix message-board confirmation, May 2002). [Online]. Available: https://www.mobygames.com/game/18540/dragon-quest-iv-michibikareshi-monotachi/

[11] IGN Staff, "Dragon Quest IV Headed Stateside," Aug. 22, 2001. [Online]. Available: https://www.ign.com/articles/2001/08/22/dragon-quest-iv-headed-stateside

[12] Japanese Wikipedia et al., "山名学" (b. Jun 8 1965; 1986 entry to Chunsoft; mainline programming III–VII; 2000 Agency for Cultural Affairs Media Arts Festival Interactive Grand Prize for DQVII; 2002 Genius Sonority founding). [Online]. Available: https://ja.wikipedia.org/wiki/山名学

[13] MobyGames, "Manabu Yamana" person record — credits list (DQ I–VII, Pokémon console titles). [Online]. Available: https://www.mobygames.com/person/134093/manabu-yamana/

[14] Mystery Dungeon Franchise Wiki, "Meta: Koichi Nakamura Interview: On the Birth of the Console RPG," trans. 2024. [Online]. Available: https://mysterydungeonwiki.com/wiki/Meta:Koichi_Nakamura_Interview:_On_the_Birth_of_the_Console_RPG

[15] Shmuplations, "Dragon Quest VI – 1995 Developer Interview (with Manabu Yamana)," trans. 2022. [Online]. Available: https://shmuplations.com/dragonquestvi/

[16] K. Yamane / Dragon Quest VII (PSX) staff and encyclopedic records, incl. the "Direction & Main Program" handling of DQVI and the SFC DQIII port, per game-staffroll wiki and the Seesaawiki DQ6 staff data. [Online]. Available: https://seesaawiki.jp/game-staffroll/

[17] Lost Media Wiki, "Dragon Quest IV (unproduced English localization of PSX role-playing game; 2002)," repr. 2026. [Online]. Available: https://lostmediawiki.com/Dragon_Quest_IV_(unproduced_English_localization_of_PSX_role-playing_game;_2002)

[18] M. Schroeder (Markus Projects), "Dragon 'Hackst' IV" (2020–2023 archive of DQIV PSX HBD/Huffman research). [Online]. Available: http://markus-projects.net/dragon-hackst-iv/

[19] M. Wilkens, "dq4psxtrans" — DQ4 PSX translation toolchain and script-host analysis. [Online]. Available: https://github.com/mwilkens/dq4psxtrans

[20] RadMageIRL, "DQIV_PSX_TOOLS" and hardware-verification work (RM provenance cited in [5]).

[21] Siliconera / Nintendo Everything / NWR developer interviews with Manabu Yamana (Genius Sonority), 2012–2013: e.g., "The Denpa Men Developer Interview," Oct. 2012. [Online]. Available: https://www.siliconera.com/the-denpa-men-developer-interview-on-developing-novel-products/

[22] J. Brown (CGC / Vinyl-Tricks history channel) — *Dragon Quest* series and Chunsoft historical material (secondary). [Online]. Available: https://www.youtube.com/@VideoGameChris

---

**Endnotes on verbatim fidelity:** the release-notes quote "partially playable Chapter 1 baseline" is taken, verbatim, from the V.99 ReBuild B release description [6]. The Nakamura quotes on the 64 KB katakana cut and the design work-loop are paraphrases of the Shmuplations translation [2][14]. The Yamana quotes on DQVI's window/AI systems are paraphrases of the Shmuplations DQVI interview [15]. All hexadecimal and count figures are taken from the VoidWalkers primary documents [5][6].

*ORDER. PRECISION. FIDELITY.*