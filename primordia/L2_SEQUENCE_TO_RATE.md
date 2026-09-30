# Closing L2: What It Would Take

> **The question:** L2 is the layer that maps a stretch of sequence to a rate — how fast this promoter fires, how strongly this ribosome binding site initiates, how long this transcript survives. Today it's wrong by 2–10×. What data, and what else, would be needed to make it error-free?

**Status:** v1, 2026-09-30. Companion to [`ACCURACY_ROADMAP.md`](./ACCURACY_ROADMAP.md) (which defines the layers) and [`WHY_NOT_BOTTOM_UP.md`](./WHY_NOT_BOTTOM_UP.md) (which explains why this has to be measured rather than computed).

This is a thought experiment, not a project proposal. But it's a useful one, because the answer turns out to be encouraging: **L2 is blocked by money and coordination, not by missing science.** That makes it the only major gap in the accuracy roadmap with a concrete price tag.

Numbers below marked *(est.)* are my own order-of-magnitude reasoning from stated assumptions, not literature values.

---

## Table of contents

- [1. "Error-free" needs redefining first](#1-error-free-needs-redefining-first)
- [2. What L2 actually decomposes into](#2-what-l2-actually-decomposes-into)
- [3. The uncomfortable question: is L2 even a function?](#3-the-uncomfortable-question-is-l2-even-a-function)
- [4. The data requirement](#4-the-data-requirement)
- [5. Why existing data mostly can't be reused](#5-why-existing-data-mostly-cant-be-reused)
- [6. What's needed besides data](#6-whats-needed-besides-data)
- [7. The program](#7-the-program)
- [8. Cost and time](#8-cost-and-time)
- [9. Closure criteria](#9-closure-criteria)
- [10. The cheap version: L2 in a test tube](#10-the-cheap-version-l2-in-a-test-tube)
- [11. What this would do for Primordia](#11-what-this-would-do-for-primordia)

---

## 1. "Error-free" needs redefining first

Strictly error-free is not achievable, and not because of any solvable deficiency. Three reasons, in increasing order of how fundamental they are:

**Measurement error sets a floor.** You cannot validate a model below the precision of the data validating it. The best absolute measurements of transcription initiation rates carry perhaps 10–20% error. A model predicting better than that is unfalsifiable, not accurate.

**Rates are not scalars.** A promoter does not have "a" firing rate. It has a rate that depends on temperature, ionic strength, supercoiling, polymerase and sigma factor availability, growth rate, and what else in the cell is competing for the same machinery. "The" rate is a projection of a high-dimensional function onto one number, and the projection is only meaningful once you fix everything else.

**Cell-to-cell variation is real.** Two genetically identical cells in the same flask have different instantaneous rates because the molecule counts are small. There is no single true value to converge on — only a distribution.

So the achievable target is:

> **Measurement-limited prediction.** For any sequence in the defined space, predict the rate to within the error of the best available measurement of that rate, in a specified condition, with correctly calibrated uncertainty — meaning the model's stated 90% interval contains the truth 90% of the time.

That's the definition used for the rest of this document. It is a demanding target and nobody is close to it, but it is reachable in principle without new physics.

---

## 2. What L2 actually decomposes into

"Sequence → rate" is five different problems wearing one label. They have different data requirements and different difficulty.

| | Sub-problem | Input window | Output | Hardest part |
|---|---|---|---|---|
| **L2a** | Transcription initiation | ~100 bp around the start site | initiation events/s per DNA copy | Epistasis between promoter elements; sigma factor competition |
| **L2b** | Translation initiation | ~80 nt around the start codon | initiation events/s per mRNA | Depends on RNA structure — i.e. it inherits L3's error |
| **L2c** | Elongation rate | whole transcript, plus tRNA pools | dwell time per position (a *profile*, not a scalar) | Charged-tRNA levels are rarely measured |
| **L2d** | Transcript and protein decay | whole molecule | half-life | Ribosome occupancy protects mRNA, so it couples to L2b/L2c |
| **L2e** | Regulatory binding | ~20–30 bp per site | affinity, and combinatorial logic | Cooperativity between sites is a second combinatorial layer |

Two couplings matter enormously and are usually ignored:

- **L2b depends on L3.** Translation initiation is governed largely by whether the ribosome binding site is sequestered in mRNA structure. So L2b can never be more accurate than RNA structure prediction allows — *unless you measure it directly rather than predicting it*. This is a strong argument for direct measurement over better modeling.
- **L2d depends on L2b and L2c.** Ribosomes physically shield mRNA from nucleases, so transcript stability depends on how heavily it's being translated. These cannot be measured independently and then composed; they have to be fitted jointly.

---

## 3. The uncomfortable question: is L2 even a function?

This section is the most important one, and it's the part that a naive "just measure more" program would get wrong.

**In a living cell, the output of a promoter depends on things that are not in the promoter.**

| Context variable | Effect |
|---|---|
| Position on the chromosome | Gene dosage changes during replication; supercoiling varies by domain; expression differs measurably between loci |
| Plasmid vs. chromosome | Copy number, supercoiling state, and regulation all differ |
| Neighboring genes | Read-through from upstream transcription; convergent transcription collisions; roadblocking |
| Global resource pool | Adding one strong promoter measurably *reduces* output from every other gene, because they share polymerases, ribosomes and nucleotides |
| Growth rate | Machinery allocation shifts dramatically with growth rate; every rate is growth-rate dependent |

The last two are the serious ones. **Resource competition means gene expression is not decomposable into independent parts** — a fact synthetic biologists have run into repeatedly, and the reason well-characterized parts fail to compose predictably.

So, strictly: `rate = f(local sequence)` **does not exist** as a well-defined function. Only `rate = f(local sequence, full cell state)` exists, and the second argument is enormous.

Three ways out, and any real program has to pick one:

1. **Fix the context.** Define L2 as `f(sequence | one specified context)`. Well-posed, measurable, and valid only in that context.
2. **Model the context.** Include the cell-state variables explicitly, which means L2 can only be solved jointly with a whole-cell allocation model. Much larger problem, much more useful answer.
3. **Quantify the irreducible variance.** Measure how much the same sequence varies across contexts and accept that as a floor.

**Nobody currently knows how big that floor is**, and finding out is a cheap, high-value experiment: take ~10³ characterized promoters, place each in ~10² chromosomal contexts, measure all of them. *(est.)* My expectation is the context-driven spread is somewhere around 2–5×, which would mean **context dependence — not the sequence model — is the binding constraint on in-vivo L2 accuracy.**

If that's right, it has a sharp implication: **the accuracy-maximizing move is to get rid of the context entirely by working in a test tube.** See §10.

---

## 4. The data requirement

Sizing each sub-problem. The method: estimate the effective parameter count, then multiply by the sampling density needed to capture interactions between parameters, then multiply by conditions.

### 4a. Transcription initiation

Sequence determinants: −35 hexamer, extended −10, −10 hexamer, spacer length (15–19 bp) and its sequence, discriminator, UP element, and the initially transcribed region (which controls abortive initiation and escape).

```
Naive sequence space over ~100 bp      4^100 ≈ 10^60      (irrelevant — never sampled)
Informative positions                  ~40
Additive model parameters              ~120
With pairwise epistasis                ~10^4
Sampling density for reliable fitting  ~10^2 per parameter
                                       ─────────────────────
Per host, per sigma factor             ~10^6 sequences     (est.)
× ~10 growth/media conditions          ~10^7               (est.)
```

Higher-order epistasis would push this further. Existing public data is perhaps 10⁵–10⁶ sequences *total*, fragmented across labs, assays and units.

### 4b. Translation initiation

Determinants: Shine–Dalgarno-like sequence and its spacing, start codon identity, secondary structure across the whole region, standby sites, and the first ~15 codons.

Structure is the dominant term and it is combinatorially entangled with the primary sequence, so the effective dimensionality is higher than the SD motif alone suggests.

```
Per host                               ~10^6 sequences     (est.)
× conditions (temperature matters —
  structure is temperature-sensitive)  ~5×
                                       ─────────────────────
                                       ~5 × 10^6           (est.)
```

### 4c. Elongation

Different in kind: the output is a profile along the transcript, not a scalar.

What's needed:
- Absolute-calibrated ribosome profiling across many conditions and organisms — and current profiling has known, contested artifacts, so method standardization has to come first.
- **Charged-tRNA quantification in every condition.** This is the missing piece. Elongation rate at a codon depends on the charged fraction of its isoacceptor, which is condition-dependent and almost never measured alongside the profiling.
- Single-molecule measurements to constrain dwell-time distributions rather than just averages.

```
High-quality profiling datasets        ~10^2 conditions    (est.)
  each paired with charged-tRNA quantification  ← the gap
Targeted variant libraries for
  context effects and pause sequences  ~10^5 sequences     (est.)
```

### 4d. Decay

```
Genome-wide decay measurements by
  metabolic labeling (not transcription
  shutoff, which perturbs the cell)    ~10^2 conditions    (est.)
Variant libraries probing cleavage-site
  and structural determinants          ~10^5 sequences     (est.)
```

### 4e. Regulatory binding

```
Per transcription factor               ~10^4–10^5 variants (est.)
× ~300 TFs in a model bacterium        ~10^7               (est.)
+ pairwise combinatorial logic         another layer entirely
```

### Total

```
~10^7 – 10^8 calibrated measurements, for ONE organism,
across a modest condition set.                              (est.)
```

**Is that achievable?** Massively parallel reporter assays currently measure 10⁴–10⁶ variants per experiment. So the total is **roughly 100–1,000 large experiments.** That's a consortium, not a miracle. This is the encouraging part.

---

## 5. Why existing data mostly can't be reused

This is the most actionable finding in the document, and it's the reason "just mine the literature" doesn't work.

### Most published data measures the wrong thing

The overwhelming majority of "promoter strength" measurements are **steady-state fluorescent protein levels**. That single number is:

```
fluorescence  ∝  transcription rate
                 × (1 / mRNA decay rate)
                 × translation initiation rate
                 × elongation completion
                 × (1 / protein decay rate)
                 × maturation efficiency
                 × copy number
```

**Seven processes convolved into one observable.** You cannot recover the transcription rate from it without independently knowing the other six — and if you knew those, you wouldn't need the experiment.

This means most existing data constrains a *product* of L2a, L2b and L2d rather than any of them. It is not useless, but it cannot be decomposed after the fact. Re-measurement with designs that separate the terms is unavoidable.

### The other four problems with existing data

| Problem | What it looks like |
|---|---|
| **Relative units** | "3.4× the reference promoter" — unusable in a mechanistic model that needs events per second |
| **Incomparable contexts** | Different strains, plasmids, copy numbers, media, temperatures, reporters; no way to place two datasets on one scale |
| **Missing metadata** | Growth rate frequently unreported, though it may be the single largest covariate |
| **Selection on what's published** | Characterized promoters are overwhelmingly strong, well-behaved ones — the sequence space is sampled non-randomly, in the region least useful for training a general model |

---

## 6. What's needed besides data

Data volume is necessary and nowhere near sufficient. Seven other things, roughly in order of how often they're neglected:

### 6.1 Absolute units and physical calibration standards
Every measurement in `transcripts · s⁻¹ · (DNA copy)⁻¹`, not in fold-change. That requires physical reference standards — a set of calibration sequences with certified absolute rates, distributed to every participating lab, included in every experiment. This is how other measurement fields achieve cross-lab comparability, and biology largely does not have it.

### 6.2 Direct rather than proxy measurement
Nascent transcript sequencing or single-molecule imaging for initiation; simultaneous mRNA and protein quantification to separate transcription from translation; metabolic labeling rather than transcription shutoff for decay.

### 6.3 Mandatory metadata
Strain, exact medium, measured growth rate, temperature, copy number, genomic position, assay, calibration standard, replicate structure. Machine-readable, deposited alongside the data, no exceptions.

### 6.4 Physics-constrained models rather than pure statistical fits
The RNA structure field provides the cautionary example: deep learning models that looked superior on random test splits turned out **not to reliably beat forty-year-old thermodynamic models** on sequence families absent from training. Models that carry the right functional form need less data and generalize better than models that learn everything from scratch.

### 6.5 Benchmarking held out by sequence distance, not randomly
A random train/test split on a library of related sequences enormously overstates generalization, because near-duplicates land on both sides. Held-out sets must be separated by sequence distance — and, ideally, by *design*: sequences drawn from a region of space the model never saw.

### 6.6 A resource-allocation model to embed the parameters in
Because of §3, in-vivo rates are coupled through shared machinery. L2 parameters are only meaningful inside a model that tracks the global pools. This means **L2 cannot be fully closed independently of L5** — a genuinely unwelcome result, and the main reason I'd expect a real program to take longer than the experiments alone suggest.

### 6.7 Prospective validation
Retrospective fitting proves nothing. The test is: design sequences nobody has measured, publish the predictions with uncertainty intervals, *then* measure. Anything else is a curve fit wearing a lab coat.

---

## 7. The program

What a serious attempt would look like, phased so each stage de-risks the next.

| Phase | What | Duration |
|---|---|---|
| **1. Standards** | Define absolute units; build and distribute physical calibration standards; publish metadata and deposition schemas; standardize the assay protocols. Unglamorous and entirely determinative of whether the rest is usable. | 1–2 yr |
| **2. Method development** | Scale direct-measurement assays (nascent transcript, simultaneous mRNA+protein, metabolic labeling) to 10⁶-variant throughput with absolute calibration. | 2–3 yr |
| **3. Context mapping** | The §3 experiment: ~10³ sequences × ~10² contexts. Determines the irreducible floor, and therefore whether the rest of the program has a worthwhile ceiling. **Run this early — it could invalidate the plan.** | 1–2 yr |
| **4. Bulk measurement** | The §4 campaign: ~10⁷–10⁸ measurements across five sub-problems, ~10 conditions, 2–3 hosts. Parallelizable across labs once phase 1 makes results comparable. | 4–6 yr |
| **5. Joint modeling** | Physics-constrained models fitted jointly with a resource-allocation model, per §6.6. Distance-held-out benchmarking throughout. | 3–4 yr, overlapping |
| **6. Prospective rounds** | Design → predict → measure → refit, iterated until closure criteria are met. Each round is a real falsification test. | 2–3 yr, overlapping |

Total: **8–12 years** with overlap.

---

## 8. Cost and time

*(est.)* Order-of-magnitude, from stated assumptions.

**Per large MPRA experiment (10⁶ variants):**

```
Oligo pool synthesis (10^6 × ~200-mers)      $10k – 50k
Library construction, cloning, transformation  $5k – 20k
Readout (sorting and/or deep sequencing)      $5k – 20k
Labor and overhead                            $30k – 100k
                                              ──────────────
                                              $50k – 200k
```

**Campaign:**

```
5 sub-problems × ~10 conditions × 2–3 hosts
  × replicates                     ≈ 150–300 experiments
                                   ≈ $10M – 60M direct
+ standards infrastructure         ≈ $5M – 15M
+ method development               ≈ $10M – 25M
+ context mapping                  ≈ $3M – 10M
+ modeling, compute, data infra    ≈ $10M – 30M
+ prospective validation rounds    ≈ $5M – 15M
                                   ──────────────────────
TOTAL                              ≈ $50M – 150M over 8–12 years
```

For scale, that is comparable to a mid-sized genomics consortium — large, but routinely spent on less tractable problems.

**The conclusion worth holding onto:**

> **L2 is a money-and-coordination problem, not a science problem.** Every technique it needs exists today. Nothing on the list requires a discovery.
>
> Contrast with **L4** (protein function, enzyme kinetics), where no amount of money buys the answer because the method doesn't exist. That is the real wall, and L2 is not it.

---

## 9. Closure criteria

"Done" needs to be falsifiable. Six criteria, all of which must hold:

1. **Prospective accuracy.** Design 10⁴ sequences never measured, spanning the defined space. Publish predictions *before* measuring. ≥90% fall within the measurement error of the observed value (~1.3×).
2. **Calibrated uncertainty.** Nominal 90% intervals contain the truth 90 ± 2% of the time, across the whole range — not just on average.
3. **Distance generalization.** Criteria 1–2 hold on sequences held out by distance, not randomly.
4. **Condition interpolation.** Accurate at conditions between those trained on, with uncertainty that correctly widens as you extrapolate.
5. **Host transfer.** Degrades by less than ~5× on a new host with ≤10⁴ retraining measurements.
6. **Mechanistic consistency.** The same parameters work inside a whole-cell model **without refitting**.

Criterion 6 is the one that separates a physical parameter from a fitted curve, and it is the hardest. A parameter that has to be retuned when you change the model it sits in was never a measurement of anything.

---

## 10. The cheap version: L2 in a test tube

Everything above is priced for a living cell, and most of the cost comes from the context problem in §3. **Remove the cell and the problem collapses.**

In a reconstituted cell-free system — a defined mix of purified components in a tube:

| In a cell | In a tube |
|---|---|
| Chromosomal position effects | None — DNA is free in solution |
| Supercoiling domains | Controlled or absent |
| Growth-rate coupling | No growth |
| Competing genome | Only what you pipette in |
| Unknown regulators | None — the component list is the component list |
| Machinery concentrations | Unknown and variable | Set by you, to a known value |
| Condition space | Enormous | Small and fully specified |

The consequences for the program are dramatic:

- **Context mapping (phase 3) becomes unnecessary.** There is no context.
- **The condition axis shrinks by ~10×.** You control every variable rather than sampling a physiological range.
- **The resource-allocation coupling (§6.6) dissolves.** Pools are known, finite, and directly measurable.
- **Measurement is direct.** You can sample the tube and quantify the actual molecular species rather than inferring from a downstream reporter.

*(est.)* Revised sizing:

```
~10^6 – 10^7 measurements, one condition family
≈ 20–40 experiments
≈ $2M – 8M over 2–4 years
```

> **L2 is roughly 20× cheaper and 3× faster to close in a test tube than in a cell**, and the result is measurement-limited within that system rather than context-limited.

This is not a consolation prize. A cell-free system is where the genetic code demonstrably runs, and rates measured there are real physical rate constants — they are the right inputs for any model, including cellular ones. **The in-vivo problem is harder largely because of the cell, not because of the code.**

---

## 11. What this would do for Primordia

Even partially closed, the effects on the app are structural rather than cosmetic:

| Today | With L2 closed in a tube | With L2 closed in vivo |
|---|---|---|
| Tier 1 runs at fidelity **F3** (predicted parameters) | Tier 1 runs at **F4** (measured) | — |
| Capability Report's amber column dominates | Amber column collapses to green for cell-free scenarios | Green for a whole genome |
| Uncertainty bands span ~10× | Bands narrow to measurement error | Same, in vivo |
| Strict mode blocks most arbitrary sequences | Strict mode runs any sequence in a defined mix | Strict mode runs whole genomes |
| "What will this edit do?" → rank-order only | → quantitative, with calibrated error | → quantitative in a living cell |

And two things that don't change, which is the point of the architecture:

- **No code changes.** Better L2 arrives as a new predictor behind the existing plugin interface, plus new rows in the parameter database. That is a data update.
- **L4 still blocks whole-cell claims.** Closing L2 does not get you a predictive cell, because you still won't know what most proteins do. L2 raises the ceiling on Tiers 1–2 dramatically and on Tier 3 only modestly.

**The practical takeaway for the project as it stands:** the app's Tier 1 target sits precisely in the regime where L2 is most nearly closed *already* — a defined tube with known component concentrations and a small condition space. That was chosen for other reasons, but this analysis independently confirms it. Building there isn't just the easiest starting point; it's the only rung where measurement-limited accuracy is achievable with what exists today.
