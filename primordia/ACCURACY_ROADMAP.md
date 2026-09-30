# Accuracy Roadmap

> **The question this document answers:** how accurately can the genetic code be simulated today, what exactly stands in the way of doing it perfectly, and what would have to happen — in the world, not just in this codebase — for each obstacle to fall?

**Status:** v1, 2026-09-30. Companion to [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md).

This document is deliberately pessimistic where the science is weak and precise where it is strong. Its purpose is to prevent the app from ever claiming more than it can deliver, and to give a concrete list of what would need to change for it to claim more.

---

## Table of contents

- [Part 1: Three different things "accurate" can mean](#part-1-three-different-things-accurate-can-mean)
- [Part 2: The knowledge audit, layer by layer](#part-2-the-knowledge-audit-layer-by-layer)
- [Part 3: Fidelity tiers for the app](#part-3-fidelity-tiers-for-the-app)
- [Part 4: The gap list — what blocks complete accuracy](#part-4-the-gap-list--what-blocks-complete-accuracy)
- [Part 5: What Primordia can do to move the needle](#part-5-what-primordia-can-do-to-move-the-needle)
- [Part 6: Outlook](#part-6-outlook)
- [Part 7: Decision rules](#part-7-decision-rules)

---

## Part 1: Three different things "accurate" can mean

Most arguments about whether biology "can be simulated" are people using one word for three things. Separating them makes the whole picture tractable.

### A. Mechanistic accuracy
*Does the simulation contain the right entities doing the right things in the right order?*

For bacterial transcription and translation, **the answer is essentially yes.** We know the molecules, the steps, the order, the structural basis. A simulator can be mechanistically correct today — not approximately, but correctly.

### B. Quantitative accuracy
*Are the rates and amounts right, with known error?*

**Partly, and it varies enormously by system.** In a reconstituted cell-free system with known component concentrations: good, within experimental error. In a living *E. coli* cell: order-of-magnitude for most genes, better for the few hundred that have been measured carefully.

### C. Predictive accuracy
*Given a sequence nobody has ever tested, can we say what it will do?*

**Mostly no**, and this is the real frontier. We can predict protein *structure* well. We cannot predict protein *function*, enzymatic rate constants, or the quantitative behavior of a novel regulatory sequence.

**The key insight for this project:** A ≫ B ≫ C in difficulty, and **A is enough to build a genuinely valuable instrument.** A simulator that is mechanistically exact, quantitatively honest, and explicit about where prediction fails is not a compromised version of the real thing — it is the correct thing to build, and it is what the professional whole-cell modeling field builds too.

A fourth thing people sometimes mean:

### D. Sufficiency
*Do we know **everything** that's going on, such that nothing is missing?*

**No, and we can't currently know whether we do.** Even in the 493-gene minimal cell — the simplest self-replicating organism ever constructed — a substantial fraction of genes still lack confident functional assignment, with published estimates ranging from roughly 15% to a third depending on how strictly "known" is defined. That gap is not a modeling problem; it is an unknown-unknowns problem, and it puts a hard ceiling on whole-cell claims.

---

## Part 2: The knowledge audit, layer by layer

For each layer: what we know, what the best method is, how wrong it is, and what limits it.

---

### L0 — Code semantics: sequence → polypeptide

**Status: SOLVED. Error: zero.**

The genetic code is a lookup table that has been experimentally determined and independently confirmed for six decades. Given a coding sequence and the correct translation table, the amino acid sequence is exactly determined.

What makes this non-trivial, and why "solved" still requires care — **the exceptions are known, catalogued, and each has an identifiable signal**, so a careful simulator handles them rather than ignoring them:

| Exception | What happens | Signal |
|---|---|---|
| Variant codon tables | ~30 known variants (e.g. UGA = Trp in *Mycoplasma*, several mitochondrial codes) | Organism/organelle |
| Selenocysteine (21st aa) | UGA recoded to Sec | SECIS element in the mRNA |
| Pyrrolysine (22nd aa) | UAG recoded to Pyl | PYLIS element, specific organisms |
| Programmed −1/+1 frameshifting | Ribosome shifts reading frame mid-gene | Slippery heptamer + downstream structure |
| Stop-codon readthrough | Stop read as an amino acid at some rate | Sequence context around the stop |
| Alternative start codons | GUG, UUG initiate in bacteria | Context-dependent |
| N-terminal Met excision | Initiator Met removed | Second-residue identity |
| RNA editing | The transcript differs from the gene | Editing-site context |

**Limit on accuracy:** none, for annotated sequences. For unannotated sequences the uncertainty is in *finding* the coding regions, not in translating them — which is an L2 problem.

**What Primordia does:** implements all of the above explicitly. V0 in the validation suite requires 100% agreement with deposited protein sequences across whole genomes, with every discrepancy explained by a documented recoding event. This is a test the app can simply pass.

---

### L1 — Machine mechanism: what physically happens

**Status: SOLVED mechanistically. Error: none in structure or step order; rates covered under L2.**

We have high-resolution structures of RNA polymerase and the ribosome in most functional states, single-molecule measurements of their stepping, and detailed kinetic schemes for the elongation cycles. The mechanism of transcription and translation in bacteria is among the best-understood processes in all of biology.

Known and modelable: promoter binding and open complex formation, abortive initiation, promoter escape, elongation, sequence-dependent pausing, backtracking, proofreading, intrinsic and factor-dependent termination; ribosome initiation, the full elongation cycle with tRNA selection and proofreading, translocation, frameshifting, termination, recycling; ribosome occlusion and polysome queueing.

**Residual uncertainty**, which is real but bounded: the precise kinetics of some intermediate steps, the mechanistic details of pausing at some sequences, and how much of in-vivo behavior is modified by factors not present in reconstituted systems.

**What Primordia does:** models these at step level rather than lumping them (see the implementation plan §5.2–5.3). Step-level modeling is what makes sequence-dependence emerge instead of being fitted.

---

### L2 — Sequence → rate: the first real ceiling

**Status: PARTLY SOLVED. Error: typically 2–10×, sometimes worse.**

This is where "the genetic code" stops being a lookup table and starts being physics. Everything here is a quantitative mapping from a stretch of sequence to a rate constant.

| Quantity | Best public method | Accuracy | Why it's hard |
|---|---|---|---|
| **Translation initiation rate** | Thermodynamic models of ribosome–mRNA interaction | ~53% within 2×, ~91% within 10× | Depends on mRNA secondary structure (itself uncertain, L3), standby sites, and long-range interactions |
| **Promoter strength** | Position weight matrices; increasingly ML | Poor; useful mainly for rank-ordering | Depends on sigma factor, supercoiling, spacer geometry, UP elements, and local chromosome context |
| **Elongation rate per codon** | tRNA-abundance-weighted models | Contested — different datasets disagree on magnitude and even on which codons are slow | Confounded by mRNA structure, charged-tRNA levels, ribosome interactions, and measurement artifacts |
| **mRNA half-life** | Sequence-feature models | Order-of-magnitude | Depends on structure, ribosome protection, RNase availability, and growth rate |
| **Transcription-factor binding affinity** | PWMs; biophysical models | Order-of-magnitude for unmeasured sites | Cooperativity, competition, chromosome accessibility |

**Why this layer is the *most tractable* of the unsolved ones:** it is fundamentally a **measurement and data problem**, not a conceptual one. There is no mystery about what determines promoter strength — there's simply not enough measured data across enough sequence space to fit a good model. Massively parallel reporter assays can measure tens of thousands of variants in one experiment, and each such dataset directly improves the models.

**What would close it:**
1. Order-of-magnitude more MPRA data, across more hosts and conditions, deposited in standard form.
2. Biophysical models with more of the real physics (supercoiling, competition for shared polymerase pools) rather than purely statistical fits.
3. Sequence models trained on those datasets — this is a good fit for machine learning because the input is sequence and the output is a scalar.

**Plausible horizon:** getting typical error inside 2× for well-studied hosts looks achievable on a several-year timescale. It is the gap most likely to close first.

**What Primordia does until then:** uses measured values where they exist; otherwise uses named predictors, tags the result `Predicted`, and attaches the predictor's *published* error distribution so it propagates into the uncertainty bands. The Capability Report shows you exactly how much of your genome falls into each bucket.

---

### L3 — RNA structure

**Status: PARTLY SOLVED. Error: ~30% of base pairs typically wrong; worse for novel families.**

Secondary structure prediction by nearest-neighbour thermodynamics has been the standard for four decades. Recent benchmarking is sobering: deep-learning methods generally **do not** reliably outperform thermodynamic folding on RNA families absent from their training data, with the best thermodynamics-constrained deep method reaching F1 ≈ 0.74 and classical methods statistically indistinguishable from it. The bottleneck is explicitly the limited volume and diversity of high-resolution RNA structures available for training.

Harder still, and relevant here:
- **Pseudoknots** are excluded from most efficient algorithms, and they matter (frameshifting signals, ribozymes, riboswitches).
- **Tertiary structure** is much less predictable than protein structure.
- **Ion dependence:** RNA folding depends strongly on magnesium, which most models treat crudely.
- **Cotranscriptional folding:** RNA folds as it is made, so the *kinetically trapped* structure often matters more than the thermodynamically optimal one. This is arguably the most under-modeled thing in the entire pipeline, and it directly affects termination, riboswitches, and initiation rates.

**What would close it:** far more structure-probing data (chemical probing at transcriptome scale), more experimental 3D structures, better treatment of ions, and kinetic folding models validated against cotranscriptional probing experiments.

**Plausible horizon:** incremental. Secondary structure may reach F1 ≈ 0.85 for common families within years; general tertiary structure prediction for RNA is a decade-scale problem.

**What Primordia does:** exposes the **ensemble**, not just the minimum-free-energy structure — base-pair probabilities rendered as confidence, so a structure the model is unsure about *looks* unsure. Cotranscriptional folding is modeled as a staged process (fold the emerging 5′ region as it appears) rather than folding the finished molecule, which is both more correct and more visually honest.

---

### L4 — Protein function: the hard wall

**Status: STRUCTURE largely solved; FUNCTION not solved. This is the deepest gap.**

Structure prediction changed decisively with AlphaFold-class methods: for natural proteins with evolutionary relatives, predicted structures are frequently competitive with experimental ones. **Primordia gets this for free** by using precomputed structures rather than predicting at runtime.

Function is a different matter entirely, and the gap is worth stating plainly:

| We can... | We cannot... |
|---|---|
| Predict the fold of a natural protein | Predict kcat or Km for an enzyme from sequence |
| Recognize homology to characterized families | Assign function to a protein with no characterized relatives |
| Rank-order variants as likely damaging or tolerated | Convert that rank into a quantitative change in activity |
| Predict binding interfaces reasonably | Predict allosteric regulation and conformational dynamics quantitatively |
| Design some proteins with intended folds | Reliably design an enzyme with a specified rate |

Two consequences bite directly:

1. **A mutation's quantitative effect is not predictable.** Variant-effect predictors — including recent genomic language models, which are state of the art especially for noncoding variants — produce scores, not rate constants. There is no general function from `(protein, mutation)` to `Δkcat`.
2. **Novel sequences are functionally opaque.** A designed or randomly generated coding sequence produces a protein whose function is genuinely unknown. This is not a limitation of the app; it is the state of the field.

**What would close it:** enzyme kinetics measured at a scale comparable to sequencing (currently they are measured one enzyme at a time, and database coverage is a small fraction of known enzymes); high-throughput functional assays across sequence space; physics-based simulation of catalysis becoming cheap enough for routine use (currently it is not, by orders of magnitude); and models that learn the mapping from those datasets.

**Plausible horizon:** decades for generality. Progress will likely come family by family — enzyme classes with lots of data first — rather than as a single breakthrough.

**What Primordia does:** it does not pretend. A protein with a measured kcat gets a green `Measured` badge. A protein with a homolog gets `Derived` with the homology noted. A protein with neither is reported as **function unknown** and, in strict mode, blocks any simulation that depends on its activity. The Capability Report counts these up front so you know before you run.

**This is the honest answer to "will it build an organism?"** — the genetic code specifies which molecules get made, and that part we know exactly. What those molecules *do*, we know for a minority of them.

---

### L5 — Networks and the whole cell

**Status: PARTLY SOLVED for minimal cells. Error: qualitative-to-~10% on headline quantities.**

Whole-cell models exist and work, within limits. The progression:

- The first whole-cell model (*M. genitalium*, 2012) integrated 28 sub-models of different mathematical types and reproduced a broad range of phenotypes.
- The *E. coli* whole-cell model was used in 2020 to **cross-evaluate heterogeneous published datasets** — using the model to find contradictions between thousands of independent measurements. Notably, it surfaced cases where reported data could not jointly account for observed doubling times. This is a genuinely novel use of simulation: *the model as an auditor of the literature.*
- The minimal cell JCVI-syn3A was modeled with a hybrid stochastic/deterministic approach in 2022, and in 2026 a full 4D model simulated the entire ~105-minute cell cycle with all 493 genes, metabolism, ribosome biogenesis, DNA replication and division, reproducing the measured doubling time to within about two minutes across 50 replicate runs.

**What they still require:** substantial parameter fitting, lumping of poorly-characterized processes, assumptions to fill gaps, and high-performance computing. And they inherit every L2–L4 uncertainty underneath them.

**The hard ceiling here is L4 plus unknown function.** You cannot build a complete mechanistic cell model when a meaningful fraction of the genes have no confident function. Everything those genes do is, necessarily, either omitted or absorbed into a fitted parameter.

**What would close it:** complete functional annotation of at least one organism (syn3A is the most likely candidate and the most actively worked on); comprehensive quantitative proteomics and metabolomics; and enough kinetic parameters to stop fitting.

**Plausible horizon:** a *mechanistically complete* minimal cell model — one where every gene has a known function and a measured parameter — is plausibly a 10–20 year goal, and syn3A is the most likely first. For *E. coli*, longer. For human cells, not foreseeable.

---

### L6 — Genome → organism

**Status: NOT SOLVED beyond minimal cells. Not a Primordia goal.**

Predicting morphology, behavior or development from genome sequence requires everything above plus multicellular coordination, mechanics and environment. Out of scope, and honestly out of scope for the field.

---

## Part 3: Fidelity tiers for the app

A formal ladder so that every feature can be labeled, and so "how accurate is this?" always has a specific answer.

| Tier | Name | Definition | Where it applies |
|---|---|---|---|
| **F5** | **Exact** | Deterministic consequence of established rules; no free parameters | Translation of an annotated CDS; reverse complement; codon tables |
| **F4** | **Measured** | Every parameter measured in the modeled system, with cited uncertainty | Cell-free expression with a fully characterized component set |
| **F3** | **Predicted** | Some parameters from named predictors with published error distributions | Expression of an arbitrary gene with a predicted initiation rate |
| **F2** | **Fitted** | Parameters tuned so the model reproduces a named dataset; interpolates, doesn't extrapolate | Whole-cell growth models |
| **F1** | **Structural** | Mechanism is right, numbers are order-of-magnitude or arbitrary | Novel-sequence expression; RNA-world scenarios |
| **F0** | **Illustrative** | Visual or conceptual only; not a simulation result | Speculative cinematics; schematic 3D folds |

**Rules the app enforces:**
- Every scenario, view and exported result declares its tier.
- A composite result takes the **minimum** tier of its inputs. One F1 parameter makes the whole answer F1.
- **Strict mode** refuses to run below F3.
- Tier is shown in the UI at all times and stamped into every export, screenshot and manifest.

---

## Part 4: The gap list — what blocks complete accuracy

Nine gaps stand between today and "complete accuracy at the genetic level." For each: what it is, why it's hard, what would close it, and how Primordia behaves in the meantime.

---

### Gap 1 — Sequence-to-rate mapping (L2)
**Blocks:** quantitative prediction for any unmeasured promoter, ribosome binding site or terminator.
**Why hard:** the models are statistical fits to sparse data; the underlying physics involves competition, supercoiling and structure.
**Closure path:** massively parallel reporter assays at much larger scale and diversity → biophysical models incorporating the missing physics → sequence models trained on the resulting data. Standardized deposition formats matter as much as the experiments.
**Difficulty:** **Moderate.** This is a data problem with a clear experimental route.
**Primordia meanwhile:** predictors behind a plugin trait with declared errors; measured values always preferred; uncertainty propagated to outputs; sensitivity analysis shows when this gap is what's limiting your result.

---

### Gap 2 — RNA folding, especially kinetic and cotranscriptional (L3)
**Blocks:** accurate initiation rates, termination efficiency, riboswitches, ribozymes, frameshifting signals.
**Why hard:** limited training and validation structures; pseudoknots break efficient algorithms; ion effects are poorly modeled; the kinetic pathway often matters more than the thermodynamic optimum.
**Closure path:** transcriptome-scale chemical probing (including cotranscriptional probing) → better energy models and explicit ion treatment → kinetic folding models validated against that data → ML trained on the enlarged corpus.
**Difficulty:** **Moderate to hard.**
**Primordia meanwhile:** predict ensembles rather than single structures; render base-pair probability as visible confidence; model folding cotranscriptionally rather than post hoc; mark all 3D RNA folds without experimental structures as `schematic`.

---

### Gap 3 — Protein function and enzyme kinetics (L4)
**Blocks:** essentially everything downstream — metabolism, regulation, and any claim about what a genome "builds."
**Why hard:** function is a property of a dynamic molecule in a specific context; catalysis involves electronic structure that cheap methods can't capture; measurements are laborious and one enzyme at a time; database coverage is a small fraction of known enzymes.
**Closure path:** high-throughput enzyme kinetics (the single highest-value experimental investment for this whole field) → ML for kcat/Km trained on it → eventually, cheap enough physics-based simulation of catalysis. Progress will be family by family.
**Difficulty:** **Hard. This is the deepest gap and the main reason "complete" accuracy is decades away.**
**Primordia meanwhile:** curated parameters for characterized proteins; explicit "function unknown" for everything else; strict mode blocks on unknowns; the Capability Report quantifies the gap per genome before you run.

---

### Gap 4 — Genes of unknown function
**Blocks:** any claim of mechanistic completeness for a whole cell.
**Why hard:** these are, by construction, the genes that resisted the easy methods. Some are small, fast-evolving, or lack characterized homologs anywhere.
**Closure path:** systematic experimental characterization of one organism end to end (syn3A is the realistic target — it's the smallest and has an active community); structure-based function prediction; genetic interaction mapping.
**Difficulty:** **Moderate for syn3A specifically, hard in general.**
**Primordia meanwhile:** annotate every gene's functional-knowledge status; count them in the Capability Report; never silently assign a function.

---

### Gap 5 — Quantitative inventories
**What's missing:** absolute counts of every molecular species, its localization and modification state, in a defined condition.
**Why hard:** proteomics is incomplete and quantification is difficult; post-translational modifications are poorly quantified; metabolite pools are hard to measure without perturbing them.
**Closure path:** absolute-quantification proteomics, improved metabolomics, single-cell measurements to capture heterogeneity rather than population averages.
**Difficulty:** **Moderate.** Steadily improving.
**Primordia meanwhile:** initial conditions come from cited measurements where available and are flagged loudly otherwise; sensitivity analysis reveals when initial conditions dominate the result.

---

### Gap 6 — Spatial organization and crowding
**What's missing:** the cytoplasm is not a well-stirred beaker. It's crowded to the point of altering reaction rates, diffusion is anomalous, and organization (nucleoid, membrane association, condensates) matters.
**Why hard:** measuring in vivo diffusion and local concentrations is difficult; spatial simulation is expensive — it is where whole-cell models spend most of their compute.
**Closure path:** better in vivo single-molecule imaging → validated coarse-grained crowding models → cheaper spatial simulation via GPU methods and machine-learned surrogates.
**Difficulty:** **Moderate.** Methods exist (lattice-based RDME approaches for speed; particle-based methods for detail); the limits are compute and validation data.
**Primordia meanwhile:** well-mixed compartments through Tier 2, with the well-mixed assumption stated explicitly as a model choice; spatial simulation deferred to an optional later phase.

---

### Gap 7 — Parameter identifiability
**What's missing:** even with a perfect model structure and perfect data, many different parameter sets can reproduce the same observations. A model that fits is not thereby correct.
**Why hard:** it's a mathematical property of the systems, not a deficiency of effort.
**Closure path:** experimental design aimed specifically at identifiability; more orthogonal data types; formal identifiability analysis as standard practice; reporting parameter *distributions* rather than point estimates.
**Difficulty:** **Moderate, and largely a matter of discipline rather than discovery.**
**Primordia meanwhile:** this is precisely why the provenance system distinguishes `Fitted` from `Measured`, records what each fit targeted, and propagates distributions rather than point values.

---

### Gap 8 — Compute
**What's missing:** a fully mechanistic, spatially resolved cell cycle currently needs high-performance computing, not a desktop.
**Why hard:** the state space is enormous; the timescales span nanoseconds to hours.
**Closure path:** GPU acceleration; better multiscale algorithms; machine-learned surrogates for expensive sub-models; and the ordinary march of hardware.
**Difficulty:** **Easy to moderate** — the most reliably improving item on this list.
**Primordia meanwhile:** hybrid scheduling; full detail only for focal molecules; the option to export a scenario for offline computation with an external simulator when you want a gold-standard run.

---

### Gap 9 — Validation data at the right resolution
**What's missing:** single-molecule, single-cell, time-resolved measurements to test simulations against. Population averages hide exactly the stochastic behavior a single-molecule simulator predicts.
**Why hard:** the experiments are demanding and low-throughput.
**Closure path:** more single-molecule imaging, ribosome profiling, direct RNA sequencing, and single-cell time-lapse — with data deposited in reusable form.
**Difficulty:** **Moderate.**
**Primordia meanwhile:** validate against the best available published datasets (V6–V9 in the plan's validation suite); make it easy to add a new dataset as a new test.

---

### Summary table

| Gap | Blocks | Difficulty | Plausible horizon |
|---|---|---|---|
| 1. Sequence → rate | Quantitative prediction for unmeasured elements | Moderate | Years |
| 2. RNA folding (kinetic) | Initiation, termination, regulatory RNA | Moderate–hard | Years to a decade |
| 3. Protein function / kinetics | Everything downstream | **Hard** | **Decades for generality** |
| 4. Unknown-function genes | Whole-cell completeness | Moderate (syn3A) | ~A decade for one organism |
| 5. Quantitative inventories | Initial conditions | Moderate | Years |
| 6. Spatial organization | In-cell realism | Moderate | Years |
| 7. Parameter identifiability | Trusting fitted models | Moderate | Discipline, not discovery |
| 8. Compute | Scale and speed | Easy–moderate | Continuous |
| 9. Validation data | Knowing if any of it is right | Moderate | Years |

**Reading the table:** eight of nine gaps are on years-to-a-decade trajectories with clear experimental routes. **Gap 3 is the wall.** A genuinely complete, predictive genetic-code simulator — one that takes an arbitrary novel sequence and tells you what it does — requires solving protein function, and that is a decades-scale problem with no shortcut currently visible.

**But note what is *not* blocked:** a simulator that is exact at L0–L1, quantitative at L2–L3 within declared error, and honest about L4 can be built **now**, and is useful now. That is the plan.

---

## Part 5: What Primordia can do to move the needle

Most of this document is about waiting for the field. These are things the project can actually do — modest, but real.

### 5.1 A curated parameter database with provenance
Rate constants are scattered across tens of thousands of papers, in inconsistent units, under inconsistent conditions. A versioned, unit-checked, citation-carrying database — even covering only the PURE system and syn3A — is a genuinely reusable artifact. Exporting it in standard formats (SBML with annotations) makes it useful outside this app.

### 5.2 Uncertainty propagation as a default
Most published models report point predictions. Routinely propagating parameter uncertainty into output bands, and ranking parameters by their contribution to output variance, turns the model into a **prioritized list of which measurements would matter most**. That list is useful to anyone doing the measuring.

### 5.3 Cross-evaluation of datasets
The *E. coli* whole-cell model demonstrated that a mechanistic simulation can be used to detect contradictions among independently published datasets — including finding that some reported values could not jointly account for observed growth. Any sufficiently complete model can do this. If Primordia's parameter database grows, running consistency checks across it is a natural and valuable feature.

### 5.4 A benchmark harness for predictors
Since predictors plug in behind a common trait, the app can benchmark them against held-out data and publish the comparison. Standardized, reproducible comparison of sequence-to-rate predictors is currently harder than it should be.

### 5.5 Reproducibility as a built-in
Bit-identical reproduction from a manifest `(engine version, parameter version, scenario hash, seed)` is unusual in computational biology and genuinely valuable. It costs little if designed in from the start and is very hard to retrofit.

---

## Part 6: Outlook

Hedged, because forecasting research is unreliable. These are trajectories, not predictions.

### Near term (1–3 years)
- Sequence-to-rate prediction improves as reporter-assay datasets grow; 2× accuracy for translation initiation in well-studied hosts looks reachable.
- Genomic language models continue improving, particularly for noncoding variant effects where classical methods are weakest.
- Spatial whole-cell simulation becomes more accessible as GPU methods mature.
- **For Primordia:** everything in Phases 0–9 of the implementation plan is achievable with today's methods. Nothing in the core plan waits on research.

### Mid term (3–10 years)
- Plausibly a complete functional annotation of the minimal cell, making a *mechanistically complete* syn3A model conceivable.
- Enzyme-kinetics prediction improves substantially within well-sampled enzyme families, while remaining poor outside them.
- RNA structure prediction improves with more probing data; kinetic folding becomes routine.
- **For Primordia:** fidelity tiers rise without architectural change — the plugin and provenance design means a better predictor is a data update, not a rewrite.

### Long term (10+ years)
- A fully mechanistic minimal cell, where every gene has a known function and a measured parameter, is a plausible goal.
- General de novo function prediction remains the open problem. Progress will likely be family-by-family rather than a single breakthrough.
- **For Primordia:** the architecture is designed so that if these land, they arrive as new predictors and new parameter data — not as a new app.

### The honest bottom line
**Complete, predictive accuracy at the genetic level is not achievable today and will not be for a long time, and the binding constraint is protein function, not computing power or software design.** What *is* achievable today is a simulator that is exact where the science is exact, quantitative where measurements exist, explicit about its error where it must predict, and refuses to run where it would have to guess. That tool does not exist in convenient form, and building it is worth doing.

---

## Part 7: Decision rules

Standing rules for how the project responds to new science, so the decisions are made once rather than argued each time.

1. **A new predictor is adopted only with a published benchmark on data it wasn't trained on.** Self-reported accuracy on a training-adjacent test set does not count. (The RNA-structure literature is a cautionary example: methods that looked superior in-family failed to beat classical thermodynamics out-of-family.)
2. **A predictor never upgrades provenance.** No matter how good, its output is `Predicted`.
3. **A measured value always beats a predicted one**, even a worse-looking measured value, unless the measurement conditions are clearly inapplicable — and that judgment is recorded in the parameter's notes.
4. **Fitting is declared.** Any parameter tuned to make the simulation match something is `Fitted`, records what it was fitted to, and can never be cited as evidence about that thing.
5. **Adding a parameter requires a citation** or an `Assumed` tag. There is no third option, and CI counts the `Assumed` tags.
6. **When a new capability crosses a fidelity tier, the tier label changes in the UI** before the feature ships. Labels are not aspirational.
7. **When in doubt, refuse.** An honest "I don't have the data for this" is a better output than a plausible number. This rule overrides all others.
