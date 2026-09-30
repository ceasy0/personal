# Primordia: Implementation Plan (v2)

> A **genetic code simulator**: import a real DNA sequence, run the molecular machinery that reads it, and watch what it actually produces — at single-nucleotide, single-molecule resolution, with every number traceable to a source.

**Status:** v2, 2026-09-30. Rewritten after your answers. v1 was an education-first app; this is an accuracy-first scientific instrument.
**Companion documents:**
- [`ACCURACY_ROADMAP.md`](./ACCURACY_ROADMAP.md) — what humanity does and doesn't understand about the genetic code, layer by layer, and the long-term path to closing each gap.
- [`SETUP_GUIDE.md`](./SETUP_GUIDE.md) — setting up your PC, including letting AI drive Blender and the scientific toolchain.

---

## Table of contents

- [0. What changed in v2](#0-what-changed-in-v2)
- [1. Your three questions, answered](#1-your-three-questions-answered)
- [2. What the app is](#2-what-the-app-is)
- [3. The accuracy contract](#3-the-accuracy-contract)
- [4. The system ladder](#4-the-system-ladder)
- [5. The core: a sequence-level expression simulator](#5-the-core-a-sequence-level-expression-simulator)
- [6. The Capability Report](#6-the-capability-report)
- [7. Architecture](#7-architecture)
- [8. Data layer](#8-data-layer)
- [9. Rendering, assets and the Blender pipeline](#9-rendering-assets-and-the-blender-pipeline)
- [10. Sound design](#10-sound-design)
- [11. AI inside the app](#11-ai-inside-the-app)
- [12. Validation suite](#12-validation-suite)
- [13. Roadmap](#13-roadmap)
- [14. Budget options](#14-budget-options)
- [15. Risks](#15-risks)
- [16. Terms you asked about](#16-terms-you-asked-about)
- [17. Remaining open questions](#17-remaining-open-questions)
- [Appendix A: Reference numbers](#appendix-a-reference-numbers)
- [Appendix B: References](#appendix-b-references)

---

## 0. What changed in v2

Your answers, and what each one did to the plan:

| Your answer | Effect on the plan |
|---|---|
| Accuracy above everything; audience is you | The whole plan is reorganized around a **provenance and uncertainty system** (§3). Education features are demoted to "the instrument explains itself". |
| The genetic code is the focus; start simple and work up | The core is now a **sequence-level, mechanism-level expression simulator** (§5), not a scene graph with animations attached. |
| Desktop app first | **Tauri 2 + Rust simulation core + Python reference lane** (§7). No web target in v1, but the core stays portable. |
| Journey later | Agreed, with reasoning in §1.3. Narrative content moves to Phase 7. |
| Import real genomes | First-class. GenBank/FASTA/GFF import, plus the **Capability Report** (§6) that tells you honestly what the app can and can't do with a given genome. |
| AI in the app eventually | §11. The valuable use is **AI as a parameter oracle with declared error bars**, not a chatbot. |
| Sound and visual quality both matter | §9 (Blender-baked assets) and §10 (event-driven procedural audio). |
| No accessibility/i18n/analytics priority, no reviewer needed | Dropped as requirements. Replaced by an **automated validation suite** (§12) — for an accuracy-first tool, tests beat reviewers anyway. |
| Budget options wanted | §14, three tiers. The honest answer is that the $0 tier covers almost everything. |
| Recs 11 and 14 unclear | Explained in §16. |

**The one-sentence version of the change:** v1 was "a story about the genetic code with a simulation inside it." v2 is "a simulator of the genetic code that can tell a story later."

---

## 1. Your three questions, answered

### 1.1 Do we understand exactly how the genetic code works? Can we simulate it accurately yet?

**Partly — and the parts split cleanly.** This is the single most important thing to get straight, because it determines what the app should claim. [`ACCURACY_ROADMAP.md`](./ACCURACY_ROADMAP.md) does this in full; here is the summary.

| Layer | Question it answers | Status | Best error today |
|---|---|---|---|
| **L0. Code semantics** | Which amino acid does this codon specify? | **Solved.** The codon tables are exact and experimentally verified, and the ~30 known variant tables and recoding events (selenocysteine, frameshifting, readthrough) are catalogued with their signals. | Zero, for annotated cases |
| **L1. Machine mechanism** | What physically happens, step by step, when a polymerase or ribosome reads a strand? | **Solved mechanistically.** Structures, step order and rates are known in detail for bacterial systems. | Mechanism: none. Rates: see L2 |
| **L2. Sequence → rate** | How fast does *this* promoter fire? How strongly does *this* ribosome binding site initiate? | **Partly solved.** Best public translation-initiation predictor lands within 2× of measurement about half the time, within 10× about 91% of the time. Promoter strength is worse. mRNA half-life from sequence is worse still. | 2–10× typical |
| **L3. RNA structure** | What shape does this RNA fold into? | **Partly solved.** Thermodynamic folding sits around F1 ≈ 0.7 on mixed benchmarks; recent deep-learning methods do not reliably beat it on families they weren't trained on. Cotranscriptional and kinetic folding is less settled. | ~30% of base pairs wrong |
| **L4. Protein function** | What does this protein *do*, and how fast? | **Structure largely solved** for natural sequences (AlphaFold-class). **Function not solved.** No general method predicts kcat, Km, or specificity from sequence. Variant-effect models give rank-order scores, not rate constants. | Order-of-magnitude or worse |
| **L5. Whole cell** | Does this genome keep a cell alive and dividing? | **Partly solved for minimal cells.** Whole-cell models of *M. genitalium*, *E. coli* and JCVI-syn3A reproduce doubling times and many phenotypes — but with heavy lumping, fitted parameters, and gaps. Roughly 15–30% of even syn3A's 493 genes still have unclear function, depending on how you count. | Qualitative to ~10% on headline numbers |
| **L6. Genome → organism** | What does this organism look like and do? | **Not solved** beyond minimal cells. | n/a |

**So: yes, start with basic genetic code and work up — that is exactly right, and it's not a compromise.** L0 and L1 are genuinely exact, and they are the whole of "the genetic code running." A simulator that nails L0–L1, is quantitative-with-error-bars at L2–L3, and is honest about L4–L6 is not a toy. It is close to the research frontier, because the research frontier is also stuck at L4.

The design consequence is §3: **the app must represent its own uncertainty as data, not as a disclaimer.**

### 1.2 Is the RNA world hypothesis good enough to structure the app around?

**No — and I'd now argue it's the wrong spine, regardless of whether the hypothesis is true.** Four reasons:

1. **It's a hypothesis about history; you want a simulator of a mechanism.** The central dogma is observed daily in labs. The RNA world is an inference about events four billion years ago that left no direct record. Architecture built on an inference has to be rebuilt when the inference moves. Architecture built on mechanism does not.
2. **It's the worst-parameterized corner of the whole field.** Prebiotic chemistry has few measured rate constants, contested conditions, and no consensus environment. Building the engine there means calibrating against the weakest data available — the opposite of accuracy-first.
3. **It isn't actually what you asked for.** Your focus is "the genetic code and what can be built with it." The RNA world is a story about where the code *came from*, which is a different (and much less settled) question than how it *works*.
4. **You lose nothing by demoting it.** In a properly general engine, "an RNA that catalyzes RNA polymerization" is just an entity with a catalytic rule. Once the engine handles arbitrary polymers, templated synthesis and catalysis, RNA-world scenarios become *content you load*, not architecture you depend on. They arrive free at Tier 5 (§4).

**What to structure it around instead: a ladder of physical systems ordered by how well-determined they are.** Each rung is a real system someone has actually built and measured, so each rung has a ground truth to validate against:

> **sequence semantics → reconstituted cell-free expression → encapsulated cell-free (synthetic cell) → whole minimal cell → populations → origin-of-life scenarios**

This ordering happens to also run from simple to complex, so "start simple" and "start where the data is best" point the same way. §4 develops it.

**The key unlock in this reframing** is rung 1: the **PURE system** — a reconstituted cell-free transcription–translation mix assembled from purified components, where every ingredient and its concentration is known. It is, almost literally, the thing you described in your first message: genetic code in an otherwise empty space, building something. It's real, it's commercially available, published mechanistic models of it are validated against experimental time courses, and it has **no unknown genes, no unmeasured cell context and no hidden regulation**. It is the most accurately simulatable "genetic code running" system that exists. That's where the app should start.

### 1.3 Should the Journey come later?

**Yes, and I agree with you.** In v1 I put it early on the assumption of an outside audience; you've removed that assumption, and the argument flips:

- **Content authored against an unfinished engine gets rewritten every time the engine changes.** Narrative is the most expensive thing to redo and the last thing that should be written.
- **For an audience of one who is building the thing, guided narration has near-zero value.** You'll know more about each scene than the script does.
- **The engine is the risky part.** Effort should go where uncertainty is, and the uncertainty is all in the simulation core.

**But keep the explanatory surfaces**, because they serve accuracy rather than pedagogy: the inspector panel, the provenance badges, the glossary, the "why did this happen" trace on any event. Those exist so *you* can tell whether the simulator is lying to you. That is debugging, not teaching.

Journey content moves to **Phase 7**, optional, built on a frozen engine.

---

## 2. What the app is

**Primordia is a desktop instrument for executing genetic sequences.**

You give it: a DNA (or RNA) sequence, annotations, and a defined molecular environment — which polymerases, ribosomes, tRNAs, nucleotides, amino acids and energy sources are present, at what concentrations.

It gives you: a physically-grounded, stochastic, single-molecule, single-nucleotide simulation of what that machinery does to that sequence over time, rendered in 3D, with every event traceable to sequence coordinates and every rate constant traceable to a source.

### 2.1 The three things it must do well

1. **Execute.** Run the machinery over the sequence, correctly and at the right speeds, producing the right molecules in the right amounts.
2. **Account.** For every number it uses and every result it shows, say where it came from and how confident it is.
3. **Show.** Render it so that what you see corresponds to what was simulated, with every visual deviation from physical reality named and toggleable.

### 2.2 Non-goals

Unchanged from v1 unless noted:

- Atom-level physics (molecular dynamics). Atoms appear as **rendered geometry** from known structures; they are never simulated as dynamical objects.
- Predicting function for genuinely novel sequences. The app reports "unknown" and means it (§6).
- Eukaryotic cells, multicellularity, development.
- Wet-lab design output (primer design, synthesis-ready constructs). Nothing here is validated for physical experiments.
- **New in v2:** no web deployment in v1, no accounts, no analytics, no translation, no classroom features.

---

## 3. The accuracy contract

This section is the difference between a simulator and an animation. Everything else in the plan hangs off it.

### 3.1 Every parameter carries provenance

No number enters the simulation as a bare float. Every one is a typed record:

```rust
// crates/primordia-params/src/lib.rs
pub enum Provenance {
    /// Directly measured. Must carry citation, organism, conditions.
    Measured   { source: CitationId, organism: Taxon, conditions: Conditions },
    /// Computed from other parameters. Uncertainty is propagated, not invented.
    Derived    { from: Vec<ParamId>, method: &'static str },
    /// Output of a named predictive model with a published error distribution.
    Predicted  { model: ModelId, benchmark: BenchmarkId },
    /// Tuned so the simulation reproduces a named dataset. Records what was fitted to.
    Fitted     { target: DatasetId, method: &'static str, residual: f64 },
    /// A guess. Always flagged; blocks strict mode.
    Assumed    { rationale: &'static str },
}

pub struct Param {
    pub id: ParamId,
    pub value: f64,
    pub unit: Unit,                      // dimensional analysis is enforced at compile time
    pub uncertainty: Uncertainty,        // point / interval / distribution
    pub provenance: Provenance,
    pub valid_range: Option<(f64, f64)>, // outside this, the simulator warns
}
```

Consequences, all of them deliberate:

- **Strict mode** refuses to run any simulation that depends on an `Assumed` parameter. If you want a number, you have to go find it or admit you're guessing. This is the single most important feature in the app.
- **Provenance is visible in the UI.** Every rate shown in the inspector has a colored badge: green `Measured`, blue `Derived`, amber `Predicted`, orange `Fitted`, red `Assumed`. Clicking it opens the citation.
- **Provenance propagates to results.** A protein count computed from three `Measured` and one `Assumed` parameter is itself marked `Assumed`. The weakest link colors the output. You can see at a glance which parts of a result you can trust.
- **The parameter store is a queryable database**, not scattered constants. It ships as a versioned data file, diffable in git, so changing a rate constant is a reviewable change with a citation attached.

### 3.2 Uncertainty is propagated, not hidden

- Every simulation can run as an **ensemble**: N replicates, with parameters resampled from their uncertainty distributions and a different RNG seed each time.
- Time-series outputs are drawn as **bands** (median + interquantile range), never as a single confident line, unless the ensemble is size 1 and the UI says so.
- A **sensitivity view** ranks which parameters actually drive the variance in a given output. This tells you where better data would help most — it is, in effect, a research to-do list generated by your own model.

### 3.3 Honest visualization

Geometric realism and scientific accuracy are not the same thing, and conflating them is how visualizations lie. Real cytoplasm is opaquely crowded; real molecules move microns per second; real transcription of a gene takes tens of seconds while a single nucleotide addition takes milliseconds. Render all of that literally and you get an unreadable blur.

So: **every deviation from physical reality is a named, logged, toggleable transformation with a numeric value shown in the HUD.**

| Transformation | Shown as | Default |
|---|---|---|
| Time dilation | `×0.001 real-time` in the HUD, always | On, value varies |
| Crowding reduction | `crowders: 12% shown` | On |
| Diffusion damping | `D ×0.05` | On for legibility |
| Schematic geometry (e.g. a folded RNA placed by layout, not by structure prediction) | Hatched outline on the object + `schematic` tag in the inspector | On where no real structure exists |
| Size exaggeration | `scale ×N` per class | **Off by default** — sizes are physically correct unless you say otherwise |
| Stoichiometry sampling (showing 1 of every N identical molecules) | `sampled 1:200` | On at cell scale |

A **"physical mode"** button sets every one of these to 1.0. The result is unreadable, and that's the point: seeing it once tells you exactly how much the normal view is helping you.

### 3.4 Determinism and reproducibility

- All randomness comes from a seeded, explicit PRNG in the Rust core. Nothing calls a system RNG.
- Transcendental functions go through `libm` rather than platform intrinsics, so results are **bit-identical across machines and OSes**. A run is reproducible from `(engine version, parameter set version, scenario hash, seed)`.
- Every run writes a **manifest**: those four identifiers plus the full resolved parameter list. Two runs that differ have a diffable reason.
- The event log is complete and replayable. Any frame in the 3D view can be traced back to the events that produced it, and any event back to the sequence coordinates and the parameters that fired it.

### 3.5 What "as accurate as possible" means operationally

It means this, and nothing vaguer:

1. Exact where the science is exact (L0, L1).
2. Measured values wherever measurements exist, cited.
3. Named predictors with published error bars where measurements don't exist.
4. Loud, run-blocking flags where neither exists.
5. A validation suite (§12) that numerically compares the simulator's output against published experimental datasets on every commit.
6. A second, independent implementation (the Python reference lane, §7.3) that the fast implementation is continuously cross-checked against.

---

## 4. The system ladder

Replaces v1's four narrative acts. Each tier is a physical system with real measurements to validate against. Tiers are built in order, and each is genuinely useful on its own.

| Tier | System | Why this rung | Ground truth available | Fidelity ceiling |
|---|---|---|---|---|
| **T0** | **Sequence semantics** — no physics, just the code | Exact. Transcription, translation, ORF finding, recoding events, restriction/annotation. | Annotated reference genomes; UniProt protein sequences | **Exact** |
| **T1** | **Reconstituted cell-free expression** (PURE-type): defined mix of purified components in a tube | Every component and concentration is known. No unknown genes, no cell context, no hidden regulation. Published mechanistic models validated against measured time courses. | Published expression time courses, fluorescence assays | **Quantitative, ~within experimental error** |
| **T2** | **Encapsulated cell-free**: the same mix inside a lipid vesicle | Adds a membrane, finite volume, resource depletion, osmotic effects. Still fully defined chemically. | Synthetic-cell literature | **Quantitative with larger bars** |
| **T3** | **Whole minimal cell** (JCVI-syn3A → later *E. coli*) | A real, living, sequenced, extensively modeled organism with published whole-cell simulations to compare against. | Doubling time, proteomics, essentiality data, published model outputs | **Semi-quantitative** |
| **T4** | **Populations and evolution** | Mutation, selection, lineages. Uses T1–T3 as the fitness evaluator. | Long-term evolution experiments, directed evolution data | **Qualitative to semi-quantitative** |
| **T5** | **Speculative scenarios** (RNA world, alternative genetic codes, designed genomes) | Runs on the same engine, clearly labeled. Now optional content rather than foundation. | Little to none — labeled accordingly | **Illustrative only** |

**T1 is the flagship.** If Primordia does nothing else well, "load a plasmid, put it in a defined cell-free mix, watch the genetic code run at nucleotide resolution, and get a protein yield curve that matches published measurements" is a real and defensible product.

---

## 5. The core: a sequence-level expression simulator

This is the heart of the app and where most of the engineering goes. The design goal: **simulate the physical events, not their statistical summary.** Aggregate behavior (yields, rates, noise) should *emerge* from mechanism rather than being fitted.

### 5.1 What gets represented

```rust
/// A physical polymer molecule. Each instance is one real molecule.
pub struct Polymer {
    pub id: MoleculeId,
    pub chemistry: Chemistry,          // Dna { strands: 1|2 } | Rna | Protein
    pub residues: ResidueBuffer,       // packed 2-bit for nucleic acid, 5-bit for protein
    pub five_prime: EndChemistry,      // triphosphate, monophosphate, cap, ...
    pub modifications: Vec<Modification>, // methylation, pseudouridine, PTMs
    pub topology: Topology,            // linear | circular
}

/// A molecular machine mid-operation: a real, individually tracked ribosome or polymerase.
pub struct Machine {
    pub id: MachineId,
    pub kind: MachineKind,             // RnaPolymerase | Ribosome30S | Ribosome70S | Rnase | ...
    pub substrate: MoleculeId,
    pub position: u32,                 // nucleotide index — this is the "program counter"
    pub state: MachineState,           // which step of the catalytic cycle
    pub nascent: Option<MoleculeId>,   // the chain being built
}

/// Well-mixed small-molecule pools, tracked as exact integer counts.
pub struct Pools {
    pub counts: HashMap<SpeciesId, u64>, // ATP, GTP, each of 20 amino acids,
                                         // each tRNA isoacceptor charged/uncharged, Mg2+, Pi, ...
    pub volume: Volume,
}
```

Three things follow from tracking real individual molecules rather than concentrations:

- **Resource limitation emerges.** Run out of a specific charged tRNA and ribosomes stall at that codon — you don't model the stall, you observe it.
- **Noise emerges.** Expression variability comes from the stochastic events, not from an added noise term.
- **Queueing emerges.** Ribosomes physically occupy ~30 nucleotides of mRNA and cannot pass each other, so polysome traffic jams appear on their own.

### 5.2 Transcription, modeled at the step level

Each arrow is a rate constant with provenance. Steps in **bold** are commonly omitted by simpler models, and each one matters quantitatively:

```
free RNAP
  → promoter search / non-specific DNA binding      k_on  (sequence-scored or measured)
  → closed complex
  → open complex formation (DNA melting, ~12–14 bp) k_open
  → **abortive initiation cycles** (2–15 nt products, often many per productive escape)
  → promoter escape                                  k_escape
  → elongation complex
      per nucleotide: NTP binding → catalysis → translocation
      **sequence-dependent pausing** (from measured pause maps where available)
      **backtracking and cleavage-factor rescue**
      **misincorporation and proofreading** (sets the real error rate)
  → termination:
      intrinsic (GC-rich hairpin + U-tract) — hairpin folding evaluated by the folding module
      factor-dependent (Rho) where the environment contains it
  → release, RNAP recycles
```

### 5.3 Translation, modeled at the step level

```
free 30S + initiation factors
  → mRNA binding at the ribosome binding site   k_init (predictor or measured; see §8.2)
  → start codon selection, initiator tRNA
  → 50S joining → elongating 70S
  → per codon (the elongation cycle):
        ternary complex (EF-Tu·GTP·aa-tRNA) sampling — rate depends on the
          **current charged level of that specific isoacceptor**
        codon–anticodon proofreading → accommodation or rejection
        peptidyl transfer
        EF-G-driven translocation
        **occlusion**: the ribosome covers ~30 nt; trailing ribosomes queue (a TASEP process)
        **programmed frameshifting** at slippery sequences with downstream structure
  → termination at a stop codon via release factors
        **stop-codon readthrough** at a low, context-dependent rate
  → ribosome recycling
  → nascent chain: cotranslational folding (as a state machine, not physics),
        N-terminal methionine excision, signal peptide handling
```

**Why this level of detail is the right call:** these are precisely the steps where "sequence" turns into "rate." A model that lumps translation into one Michaelis–Menten term cannot tell you why a rare-codon cluster slows a gene down, because it has thrown away the codons. Since the genetic code is the whole subject of the app, the codons have to stay.

### 5.4 Degradation, because steady state needs a sink

- mRNA: endonucleolytic cleavage (sequence- and structure-dependent where data exists), then exonucleolytic decay. Ribosome occupancy protects mRNA — an emergent coupling between translation and stability that falls out for free.
- Protein: first-order decay plus, where relevant, energy-dependent proteolysis with degron recognition.

### 5.5 Scheduling

A hybrid engine, partitioned automatically by species count and timescale — the same strategy the published whole-cell models use:

| Method | Used for | Why |
|---|---|---|
| **Exact stochastic simulation** (Gillespie direct → next-reaction method) | Low-count species, machine state transitions | Correct when molecule numbers are small, which is when randomness actually matters |
| **Tau-leaping** | Mid-count species | ~10–100× faster, with an error bound |
| **Deterministic ODE** (adaptive Runge–Kutta) | Large pools (NTPs, amino acids) | Smooth, high-count, no meaningful noise |
| **Fixed-step machine stepping** | Ribosome/RNAP position updates | A machine is a state automaton on a lattice; stepping it is cheaper than a full event queue |
| **Spatial (optional, later tiers)** | Reaction–diffusion where geometry matters | Lattice-based (RDME) for speed, particle-based for detail |

The partitioning is automatic but **inspectable and overridable**, and the choice is recorded in the run manifest, because switching method can change results and that must never be silent.

### 5.6 The genome compiler

The path from a file to a runnable system:

```
GenBank / FASTA / GFF
  → parse, validate (IUPAC ambiguity codes, circularity, coordinate conventions)
  → resolve annotations: keep supplied features; optionally augment with detection
        promoters      (position weight matrices, scored; flagged Predicted)
        ribosome sites (thermodynamic initiation-rate model; flagged Predicted)
        ORFs           (six-frame, alternative starts, correct translation table)
        terminators    (hairpin + U-tract, scored by the folding module)
  → build the molecular parts list: every RNA and protein this sequence encodes
  → bind parameters: for each part, look up Measured → else Predicted → else flag Assumed
  → emit: a runnable system + a Capability Report (§6) + a source map
```

The **source map** is what makes the whole thing debuggable: every simulated event points back to exact sequence coordinates, so the 3D view, the event log and the sequence editor stay in lockstep. Click a ribosome in 3D, the editor highlights the codon it's reading; set a breakpoint on a codon, the simulation stops when a ribosome reaches it.

---

## 6. The Capability Report

**Directly answers your "or, hopefully, anything it actually can."**

When you import a genome, before running anything, the app produces a report:

```
Imported: Escherichia coli K-12 MG1655 (U00096.3)
  4,641,652 bp · 4,494 annotated features · translation table 11

WHAT THIS SEQUENCE ENCODES                    (Tier 0 — exact)
  4,298 protein-coding genes  →  protein sequences derived exactly
    179 RNA genes             →  22 rRNA, 86 tRNA, 71 other
      of which 3 use programmed frameshifting (annotated) — handled
      of which 1 uses selenocysteine recoding — handled

WHAT I CAN SIMULATE QUANTITATIVELY            (Tier 1 — measured parameters)
    412 genes with measured promoter strength              9%
    338 genes with measured initiation rate                8%
  1,051 genes with measured mRNA half-life                24%
    892 proteins with measured abundance                  21%

WHAT I MUST PREDICT                           (amber — error bars apply)
  3,886 promoters      via PWM scoring         typical error: large, poorly characterized
  4,160 init. rates    via thermodynamic model  ~53% within 2×, ~91% within 10×
  3,447 mRNA half-lives via sequence features   order-of-magnitude

WHAT I CANNOT DO                              (red — honest gaps)
  - Enzymatic rate constants for 3,905 of 4,298 proteins: no measured kcat/Km
  - Regulatory logic for 2,190 genes: transcription-factor binding incompletely mapped
  - 1,412 proteins have no experimentally verified function
  → Whole-cell growth simulation for this organism is NOT supported.
    Supported at Tier 3: JCVI-syn3A only.

RECOMMENDED USE
  ✓ Tier 0 analysis of any gene here
  ✓ Tier 1 cell-free expression of any single gene or small construct
  ✗ Tier 3 whole-cell simulation — use syn3A
```

Two reasons this matters more than it looks:

1. **It makes the app's honesty structural rather than rhetorical.** You can't accidentally over-trust a result, because the app told you its coverage before you pressed run.
2. **It's genuinely useful output on its own.** "Which parts of this genome do we actually have numbers for?" is a real question with no convenient tool to answer it.

---

## 7. Architecture

### 7.1 Platform decision: Tauri 2

Given desktop-first, accuracy-first, and AI doing most of the coding:

| Option | Verdict |
|---|---|
| **Tauri 2** (Rust backend + system WebView frontend) | **Recommended.** The Rust backend *is* the simulation core — no bridge needed. Reported bundles are single-digit to low-tens of MB against Electron's 100+ MB, with correspondingly lower memory and startup cost (published comparisons vary; the order of magnitude is consistent). Built-in **sidecar** support cleanly manages the bundled Python process. Keeps a future web build possible via WASM. |
| Electron | Same UI ecosystem, much heavier, and no natural home for a Rust core. |
| Native Rust + wgpu (egui/Bevy) | Fastest and cleanest in principle, but you lose the mature text/UI ecosystem — and this app is UI-heavy (sequence editor, tables, plots, citations). |
| Python + Qt | Best scientific library access, worst rendering and packaging. Use Python as a *sidecar*, not the shell. |

### 7.2 The two-lane design

This is the structural expression of §3.5, and I consider it the most important architectural decision in the plan.

```
┌─ FAST LANE ──────────────────┐        ┌─ REFERENCE LANE ─────────────────┐
│ Rust core                    │        │ Python sidecar                   │
│ • deterministic, bit-exact   │◄─ CI ─►│ • ViennaRNA, BioPython, COBRApy  │
│ • multithreaded, interactive │ cross- │ • scipy, numpy                   │
│ • ships in the app           │ check  │ • slow, authoritative            │
│ • 60 fps target              │        │ • the oracle we test against     │
└──────────────────────────────┘        └──────────────────────────────────┘
```

- The **fast lane** is what you interact with. Rust, deterministic, compiled into the Tauri backend.
- The **reference lane** is what proves the fast lane is right. It uses the established scientific Python stack, runs offline, and is the source of truth in disputes.
- **CI runs both on the same inputs and fails if they diverge beyond a declared tolerance.** That test is the app's definition of "accurate."

The reference lane also does the jobs Python is simply better at: genome import pipelines, structure preprocessing, Blender asset baking (§9), and running heavyweight external simulators when you want a gold-standard comparison.

### 7.3 Stack

| Concern | Choice | Notes |
|---|---|---|
| Shell | **Tauri 2** | Sidecar for Python; native menus; auto-update later |
| Core language | **Rust** (stable, `#![forbid(unsafe_code)]` in the sim crates) | Determinism, speed, and it's the Tauri backend anyway |
| Numerics | `libm` for transcendentals; `uom` or a hand-rolled newtype layer for **compile-time unit checking** | Dimensional errors become type errors — cheap insurance for a scientific tool |
| RNG | `rand_xoshiro`, explicitly seeded, one stream per subsystem | Reproducibility |
| UI | **TypeScript + React + Tailwind + Radix** | Mature, and the UI is text-heavy |
| 3D | **three.js** via **React Three Fiber** + drei | WebGL2 baseline; WebGPU as an enhancement once the WebView supports it broadly |
| Sequence editor | **CodeMirror 6** with a custom nucleic-acid mode | Handles megabase documents; decorations for features and the program counter |
| Plots | **uPlot** | Handles streaming series and ensemble bands efficiently |
| IPC | Tauri **Channels** with binary payloads | Streams simulation frames without JSON overhead |
| Python sidecar | **3.12+**, pinned with `uv`, bundled via PyInstaller or a relocatable venv | Optional at runtime: the app degrades gracefully if absent |
| Audio | **Web Audio API** (procedural) | §10 |
| Storage | **SQLite** (parameters, runs, citations) + flat files for genomes/structures | Queryable provenance database; trivially diffable exports |
| Testing | `cargo test` + `proptest`; `pytest`; **Vitest**; **Playwright** | §12 |
| CI | GitHub Actions: build all three platforms, run the validation suite | |

### 7.4 Repository layout

```
primordia/
  crates/
    primordia-seq/         # alphabets, translation tables, recoding, parsing, ORFs
    primordia-params/      # provenance-typed parameter store, units, uncertainty
    primordia-fold/        # RNA secondary structure (own impl + optional ViennaRNA FFI)
    primordia-sim/         # entities, machines, processes, schedulers, event log, snapshots
    primordia-genome/      # the genome compiler + Capability Report
    primordia-io/          # GenBank/FASTA/GFF/SBML/SBOL import-export
    primordia-app/         # the Tauri backend: commands, channels, run manager
  ui/                      # React + R3F frontend
    src/
      viewport/            # 3D scene, camera, picking, LOD
      sequence/            # CodeMirror editor, feature track, program counter
      inspector/           # entity cards, provenance badges, citations
      run/                 # transport controls, ensemble config, manifest viewer
      report/              # Capability Report, validation dashboard
      plots/
  reference/               # Python reference lane
    oracle/                # independent implementations for cross-checking
    ingest/                # genome, structure and parameter import pipelines
    validate/              # published-dataset comparisons
  assets/
    bake/                  # Blender + Molecular Nodes scripts (see SETUP_GUIDE.md)
    baked/                 # generated glTF/KTX2 output (git-lfs or regenerated)
  data/
    parameters.sqlite      # the provenance database
    citations.bib
    genomes/               # reference genomes with annotations
    structures/            # preprocessed PDB/AlphaFold entries
  docs/
    adr/                   # architecture decision records
    science/               # per-module parameter notes and derivations
```

---

## 8. Data layer

Accuracy is mostly a data problem, so this deserves more attention than it usually gets.

### 8.1 What we need, and where it comes from

| Data | Sources to evaluate | Licensing note |
|---|---|---|
| Genomes + annotations | NCBI/GenBank, RefSeq, Ensembl Bacteria | Generally unrestricted |
| Protein sequences and function | UniProt | CC BY 4.0 |
| 3D structures | RCSB PDB (CC0), AlphaFold DB (CC BY 4.0) | Attribution required for AlphaFold |
| Enzyme kinetics (kcat, Km) | BRENDA, SABIO-RK | **Verify terms before any non-academic use** |
| Regulatory networks | RegulonDB, EcoCyc | **EcoCyc has licensing conditions — check** |
| Quantitative cell biology numbers | BioNumbers | Cited per entry |
| RNA families and structures | Rfam, RNAcentral | Generally open |
| Minimal-cell model parameters | Published syn3A whole-cell model repositories | **Check repository license before reuse** |
| Cell-free expression time courses | Published PURE/TXTL modeling papers | Extract from figures/supplements; cite |

**Process:** every dataset gets an ingest script in `reference/ingest/`, a recorded source URL and access date, and a license note in `docs/science/`. Nothing enters `parameters.sqlite` by hand.

### 8.2 Predictors, and their declared error

Each predictor is a plugin behind a trait, and each ships with a benchmark result that becomes the `Predicted` provenance's error distribution:

| Quantity | Method | Declared accuracy |
|---|---|---|
| Translation initiation rate | Thermodynamic initiation model (open-source implementation) | ~53% within 2×, ~91% within 10× |
| RNA secondary structure | Nearest-neighbour thermodynamics; optionally ML | F1 ≈ 0.7 typical; worse on unseen families; pseudoknots poor |
| Promoter strength | PWM scoring; optionally ML | Poor — treat as rank-order only |
| Variant effect | Genomic language models (see §11) | Rank-order, not rate constants |
| Protein structure | Precomputed from AlphaFold DB | High for natural proteins; **no structure prediction at runtime** |

**Rule:** a predictor may never upgrade its own provenance. Predicted stays Predicted no matter how good the model claims to be.

---

## 9. Rendering, assets and the Blender pipeline

You asked whether AI can drive programs on your PC. **Yes** — Claude Code running locally can invoke Blender headlessly and script it in Python. [`SETUP_GUIDE.md`](./SETUP_GUIDE.md) is the walkthrough. The architectural point:

### 9.1 Bake offline, render cheap at runtime

Blender is used as an **offline asset compiler**, never at runtime:

```
PDB / mmCIF / AlphaFold entry
  → reference/ingest: clean, select chains, compute coarse bead model
  → Blender (headless) + Molecular Nodes: build surface/cartoon representations,
      decimate to LOD meshes, bake ambient occlusion and normal maps,
      bake matcap/impostor sprite atlases for distant instances
  → export glTF/GLB + KTX2 textures → assets/baked/
  → the app loads these; it never runs Blender
```

Molecular Nodes is a Blender add-on built on Geometry Nodes that imports molecular formats and ships a large library of molecular-specific nodes. It has a Python API, though that API is explicitly experimental and has been changing — so the bake scripts should be **pinned to a specific Blender + add-on version** and treated as a reproducible build step, with outputs committed or cached.

### 9.2 What gets rendered how

| Scale | Representation | Technique |
|---|---|---|
| Atomic | Ball-and-stick / space-filling | GPU **sphere impostors** (ray-cast in the fragment shader) — millions of atoms without millions of triangles |
| Molecular | Cartoon / surface | Baked LOD meshes from Blender |
| Machine | Ribosome, polymerase at work | Real structures, animated between conformational states by the event stream |
| Crowd | Cytoplasm | Instanced impostors with baked matcaps; procedural Brownian motion for non-focal molecules |
| Nucleic acid | Helices of any length | Instanced per-nucleotide, transforms computed in the vertex shader from the instance index |

### 9.3 Visual quality, deliberately

Since you want it to look good: soft ambient occlusion (a screen-space method such as GTAO/N8AO), physically-based materials, subtle depth of field on the focal object, selective bloom only on catalytic events, and filmic tone mapping (AgX). Dark, near-black environment with a faint volumetric suggestion of solvent. The reference aesthetic is David Goodsell's molecular illustration: flat-ish shading, strong silhouettes, semantic color, no gratuitous specularity.

---

## 10. Sound design

You said yes to quality audio. The design that fits an accuracy-first tool: **sonification, not soundtrack.**

- **Event-driven procedural audio via the Web Audio API.** Each simulation event class maps to a short synthesized sound: nucleotide addition (a soft tick, pitch-mapped to base identity), peptide bond formation, initiation, termination, misincorporation (a distinct, slightly dissonant cue — errors should be *audible*), machine stalling.
- **Density becomes texture.** At realistic event rates you get thousands of events per second; individual ticks merge into a continuous texture whose density and timbre encode activity. That's genuinely informative: you can *hear* a ribosome stall.
- **Rate-aware mixing.** Because event rates span orders of magnitude, sounds are grouped and voice-limited (a pool of ~64 voices with priority by event salience), and the mix is normalized against the current time-dilation factor.
- **Ambience.** A low, slowly-evolving drone bed keyed to the current tier — quiet and spatially thin at T1 (a tube), denser and warmer at T3 (inside a cell).
- **Toggleable sonification channels**, so you can solo, say, "misincorporation events only" and listen to the error rate.

Implementation: Web Audio in the frontend, driven by the same event channel that drives rendering. No external audio middleware needed.

---

## 11. AI inside the app

Ranked by actual value, which is the reverse of what's usually built first:

### 11.1 AI as a predictor with declared error bars (highest value)

Genomic language models are, right now, the best available tools for some of the L2/L4 gaps. The recent generation is trained across all domains of life at very large scale (the leading open one was trained on roughly 9.3 trillion nucleotides from over 128,000 species, at 40B parameters with a 1-megabase context window) and is state-of-the-art for **noncoding** variant effects in particular — which is exactly where classical methods are weakest.

Integration rule, which is non-negotiable: **these plug in behind the same predictor trait as any other model, produce `Predicted` provenance, carry their benchmarked error distribution, and never upgrade themselves to `Measured`.** They are one more fallible instrument in a rack of fallible instruments.

Uses: variant-effect scoring for edits, annotation of unannotated imported sequences, plausibility scoring for designed sequences, and filling the "unknown function" column in the Capability Report with clearly-labeled guesses.

Practical note: running a 40B-parameter model locally needs serious GPU memory; a smaller checkpoint or a hosted endpoint is the realistic near-term route (§14).

### 11.2 AI as a scenario builder

Natural language → a validated scenario file. "Set up PURE at standard concentrations with 5 nM of this plasmid and run 20 replicates for two hours." The model emits a scenario YAML; the app validates it against the schema and shows you the diff before running. Low risk, high convenience, because the output is checked by a schema rather than trusted.

### 11.3 AI as a lab assistant

A local model (via a local inference server) with read access to the current run's state, event log and parameter provenance, answering questions like "why did expression plateau at t=40 min?" — with the constraint that it must cite specific events or parameters from the run. Grounded in the run data, not in its own recollection.

### 11.4 What not to build

An AI that generates biological explanations without grounding. For an accuracy-first tool, a confident wrong explanation is worse than no explanation.

---

## 12. Validation suite

This replaces the human science reviewer from v1. It runs in CI on every commit, and it *is* the credibility of the project.

| ID | Test | Pass criterion |
|---|---|---|
| **V0** | Translate every annotated CDS in three reference genomes; compare to the deposited protein sequences | **100% identical**, with any mismatch explained by a documented recoding event |
| **V1** | Round-trip GenBank → internal model → GenBank | Semantically identical |
| **V2** | Reverse complement, transcription, and all codon tables against known vectors | Exact |
| **V3** | Stochastic engine against analytic solutions (birth–death mean and variance, first-passage times) | Within sampling tolerance over N seeds |
| **V4** | Fast lane vs. reference lane on identical inputs | Within declared tolerance per quantity |
| **V5** | RNA folding vs. ViennaRNA native on a fixture set | Identical structures, or documented differences |
| **V6** | **Cell-free expression time course** vs. published measured curves | Within published experimental error |
| **V7** | Resource depletion: predicted amino-acid consumption vs. protein yield | Stoichiometrically exact |
| **V8** | Codon-level: rare-codon clusters reduce elongation rate in the measured direction and rough magnitude | Qualitative + rank correlation |
| **V9** | syn3A doubling time and macromolecular composition | Within ~10% of published values |
| **V10** | Determinism: same manifest → bit-identical output, across all three OSes | Exact hash match |
| **V11** | Strict mode: no `Assumed` parameter reachable in any shipped scenario | Zero |

**V6 is the one that matters most.** It is the first test where the simulator is compared against physical reality rather than against itself.

---

## 13. Roadmap

Reordered for accuracy-first and desktop-first. Estimates assume AI-assisted development with you reviewing, and include the data curation work, which is usually underestimated.

| Phase | Deliverable | Tier | Estimate |
|---|---|---|---|
| **0. Foundations** | Repo, Tauri shell, Rust core skeleton, parameter store with units and provenance, CI, ADRs | — | 2–3 weeks |
| **1. Tier 0 complete** | Sequence semantics: import any genome, exact translation, ORFs, recoding, feature detection, sequence editor, Capability Report v1. **V0–V2 green.** | T0 | 4–6 weeks |
| **2. First light** | 3D viewport, instanced helices, LOD, structure loading, picking, sequence↔3D sync. Static but real. | T0 | 4–6 weeks |
| **3. The engine** | Stochastic core, machines, transcription and translation at step level, event log, snapshots, determinism. **V3, V10 green.** | T1 | 8–10 weeks |
| **4. It runs** | Full cell-free (PURE-type) scenario: defined mix, one gene, nucleotide-resolution execution, animated in 3D, ensemble runs with uncertainty bands. **V4–V7 green.** ← *the flagship milestone* | T1 | 6–8 weeks |
| **5. Reference lane** | Python sidecar, cross-checking in CI, ingest pipelines, validation dashboard. **V8 green.** | T1 | 4 weeks |
| **6. Editing and comparison** | Mutate/insert/delete with live consequence analysis, A/B compare, saved runs, manifest diffing, variant-effect predictors. | T1 | 4–6 weeks |
| **7. Polish pass** | Blender asset pipeline, baked LODs, sound design, visual quality pass. | T1 | 4–6 weeks |
| **8. Encapsulation** | Vesicle compartment, finite volume, resource depletion, membrane rendering. | T2 | 6–8 weeks |
| **9. Whole cell** | syn3A: full gene set, metabolism, replication, growth, division. **V9 green.** | T3 | 16–24 weeks |
| **10. Populations** | Mutation, selection, lineage trees, evolution runs. | T4 | 8–10 weeks |
| **11. Optional** | Journey/narrative content, RNA-world and designed-genome scenarios, web build. | T5 | open-ended |

**Phase 4 is the real milestone.** At that point you have a working, validated genetic code simulator. Everything after is extension.

Rough totals: **Phases 0–4 in about 6–8 months part-time, 3–4 months full-time.** Through Phase 9 (a living cell): **18–30 months part-time.**

---

## 14. Budget options

| Tier | Cost | What it buys | Verdict |
|---|---|---|---|
| **A. Free** | **$0** | Rust, Python, Node, Tauri, Blender, Molecular Nodes, ViennaRNA, PDB, AlphaFold DB, UniProt, NCBI, GitHub free tier. Runs on your existing PC. | **Covers Phases 0–10 completely.** Genuinely nothing essential is paywalled. |
| **B. Assisted** | **~$20–200/mo** | AI coding assistance (the main cost, and the one that actually moves the schedule); optional hosted inference for genomic language models when a local GPU can't hold them; a cheap VM if you ever want CI on beefier hardware. | **Recommended.** The coding assistance is the highest-leverage spend by a wide margin. |
| **C. Hardware** | **$1,500–4,000 one-time**, or ~$0.50–3/hr cloud GPU | A GPU with 24 GB+ VRAM: local genomic-language-model inference, fast Blender rendering, GPU-accelerated spatial simulation later. | **Optional, defer.** Rent by the hour first; buy only if you find yourself doing it constantly. |

**Licensing costs to watch** (these are the realistic surprises): ViennaRNA's license terms for non-academic use; BRENDA and EcoCyc terms for anything commercial; AlphaFold DB requires attribution. If Primordia stays open-source and non-commercial, all of this is straightforward — which is a mild argument for staying open-source.

---

## 15. Risks

| Risk | Why it's real here | Mitigation |
|---|---|---|
| **Accuracy theatre** — the provenance system exists but everything is quietly `Assumed` | The easiest failure mode for this design | Strict mode blocks it; CI counts `Assumed` parameters and fails on regression (V11) |
| **The parameter hunt is the actual project** | Curating cited rate constants is slow, unglamorous, and the dominant cost of accuracy | Start with the PURE system precisely because its parameter set is small and published; grow outward |
| **Two lanes diverge and nobody notices** | Cross-checks silently disabled when they get annoying | V4 is a hard CI failure; tolerances live in version control and changing one is a reviewable diff |
| **Step-level detail is too slow to watch** | A ribosome does ~15 codons/s; a cell has hundreds | Hybrid scheduling; only focal molecules run at full detail; benchmark early in Phase 3 |
| **Scope creep into Tier 3 too early** | syn3A is seductive and is 4× the work of Tier 1 | Phase gate: no Tier 3 work until V6 passes |
| **Blender pipeline rot** | Molecular Nodes' API is explicitly experimental and changing | Pin Blender + add-on versions; cache baked outputs; the app never depends on Blender at runtime |
| **Solo project stalls** | Long build, no external deadline | Every phase produces something usable on its own; Phase 4 is deliberately early |

---

## 16. Terms you asked about

**Recommendation 11, "start with a vertical slice."** Two ways to build: *horizontally*, finishing the whole data layer, then the whole simulation layer, then the whole UI — you have nothing that works until the very end, and integration problems all surface at once, late. Or *vertically*: pick one narrow capability and build it end to end, thin but complete. Here that means: **one gene, one promoter, one ribosome binding site, expressed in one defined cell-free mix, rendered in 3D, with real cited parameters and a passing validation test.** Narrow, but every layer is exercised and every interface is proven. Phase 4 is that slice. Everything afterwards widens it.

**Recommendation 14, "i18n."** Internationalization: structuring the app so UI text lives in a lookup table rather than hardcoded in components, making translation to other languages a data change instead of a code change. It's nearly free upfront and expensive to retrofit. **Given that the audience is you, I'd skip it** — I'm noting it only so the choice is deliberate. (A related practice worth keeping regardless: don't concatenate strings to build sentences.)

---

## 17. Remaining open questions

Only three left, and none of them block Phase 0:

1. **Which organism after syn3A?** *E. coli* has by far the best data and the most published models, but it's ~10× the genes. Default: *E. coli* K-12 MG1655, at Tier 1 only (single genes in cell-free), with Tier 3 reserved for syn3A.
2. **How far into spatial simulation do you want to go?** Well-mixed compartments get you through Tier 1–2 and much of Tier 3. Full spatial reaction–diffusion is a large additional effort and is where the published whole-cell models spend most of their compute. Default: well-mixed through Phase 9, spatial as an optional Phase 12.
3. **Open-source now or later?** Affects which datasets you can use without license review. Default: develop in a private repo, decide before Phase 5 (when data ingestion starts in earnest).

---

## Appendix A: Reference numbers

Values for tuning and sanity checks. **Every one must be replaced by a cited entry in `parameters.sqlite` before it enters the simulation** — these are for orientation only.

| Quantity | Approximate value |
|---|---|
| B-DNA geometry | ~10.5 bp/turn, ~3.4 Å rise, ~20 Å diameter |
| A-form RNA duplex | ~11 bp/turn, ~2.6–2.8 Å rise |
| Transcription bubble | ~12–14 bp |
| RNAP elongation (*E. coli*) | ~40–80 nt/s |
| Ribosome elongation (*E. coli*) | ~10–20 aa/s |
| Ribosome mRNA footprint | ~30 nt (sets the queueing limit) |
| Replicative DNA polymerase | ~1,000 nt/s |
| Error rates | replication ~10⁻⁹–10⁻¹⁰/bp; transcription ~10⁻⁵–10⁻⁴; translation ~10⁻⁴–10⁻³ |
| σ70 promoter consensus | −35 `TTGACA`, −10 `TATAAT`, spacer 15–19 bp |
| Shine–Dalgarno | ~`AGGAGG`, ~5–9 nt upstream of the start |
| Translation tables (NCBI) | 1 standard; 11 bacterial; 4 *Mycoplasma* (UGA = Trp) |
| JCVI-syn3A | 493 genes, ~543 kbp, ~105 min doubling, ~400 nm diameter |
| syn3A unknown function | roughly 15–30% of genes, depending on source and criterion |
| *E. coli* K-12 | ~4.64 Mbp, ~4,300 protein-coding genes |
| PURE-type system | reconstituted from tens of purified components at known concentrations |
| Initiation-rate prediction | ~53% within 2×, ~91% within 10× |
| RNA structure prediction | F1 ≈ 0.7 typical on mixed benchmarks |

## Appendix B: References

**Whole-cell and systems modeling**
- Karr, J. R. et al. (2012). A whole-cell computational model predicts phenotype from genotype. *Cell.*
- Macklin, D. N. et al. (2020). Simultaneous cross-evaluation of heterogeneous *E. coli* datasets via mechanistic simulation. *Science.*
- Thornburg, Z. R. et al. (2022). Fundamental behaviors emerge from simulations of a living minimal cell. *Cell.*
- Thornburg, Z. R., Maytin, A. et al. (2026). Bringing the genetically minimal cell to life on a computer in 4D. *Cell.* Code: [Luthey-Schulten-Lab/Minimal_Cell](https://github.com/Luthey-Schulten-Lab/Minimal_Cell)
- Agmon, E. et al. (2022). Vivarium: an interface and engine for integrative multiscale modeling in computational biology. *Bioinformatics.*
- Breuer, M. et al. (2019). Essential metabolism for a minimal cell. *eLife.*
- Hutchison, C. A. et al. (2016). Design and synthesis of a minimal bacterial genome. *Science.*

**Cell-free expression**
- Shimizu, Y. et al. (2001). Cell-free translation reconstituted with purified components. *Nature Biotechnology.*
- Mavelli, F. et al. (2015). A simple protein synthesis model for the PURE system operation. *Bulletin of Mathematical Biology.*
- Nucleotide-level chemical reaction network modeling of reconstituted cell-free expression systems. *ACS Synthetic Biology* (2026).

**Prediction methods**
- Salis, H. M. et al. (2009). Automated design of synthetic ribosome binding sites. *Nature Biotechnology.*
- OSTIR: open source translation initiation rate prediction. *JOSS* (2021).
- Lorenz, R. et al. (2011). ViennaRNA Package 2.0. *Algorithms for Molecular Biology.*
- Sato, K. et al. (2021). RNA secondary structure prediction using deep learning with thermodynamic integration. *Nature Communications.*
- Deep learning for RNA secondary structure determination: gauging generalizability. *RNA* (2026).
- Brixi, G. et al. Evo 2: genome modeling and design across all domains of life. *Nature* (2026). Code: [arcinstitute/evo2](https://github.com/arcinstitute/evo2)
- Jumper, J. et al. (2021). Highly accurate protein structure prediction with AlphaFold. *Nature.*

**Simulation methods**
- Gillespie, D. T. (1977). Exact stochastic simulation of coupled chemical reactions. *J. Phys. Chem.*
- Roberts, E. et al. (2013). Lattice Microbes: high-performance stochastic simulation for the reaction-diffusion master equation. *J. Comput. Chem.*
- Hoffmann, M. et al. (2019). ReaDDy 2: fast and flexible software framework for interacting-particle reaction dynamics. *PLOS Comput. Biol.*
- Andrews, S. S. Smoldyn: particle-based simulation of spatial reaction–diffusion.

**Tooling**
- Johnston, B. A. [Molecular Nodes](https://github.com/BradyAJohnston/MolecularNodes) — molecular import and animation in Blender.
