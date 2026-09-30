# Why Not Start From Atoms?

> **The question:** we understand the properties of individual atoms completely. We don't understand what folds RNA. So why not start at the level we understand — atoms — and build upward by composing them into larger structures? Wouldn't that give us the higher layers for free?

**Status:** v1, 2026-09-30. Companion to [`ACCURACY_ROADMAP.md`](./ACCURACY_ROADMAP.md).

This is the right question to ask, and the reasoning behind it is sound. It's also the question that the entire field of computational biology has spent fifty years running into. The answer is not "that's naive" — it's that there are four specific walls, three of which are quantitative and one of which is structural. The fourth is the interesting one.

---

## 1. The premise, stated more precisely

"We understand the properties of individual atoms" is true in a specific sense that turns out not to be the useful one.

What we have is the **governing equation**. Non-relativistic quantum mechanics describes electrons and nuclei to a precision far beyond anything biology requires. Dirac noticed the implication immediately, in 1929:

> "The underlying physical laws necessary for the mathematical theory of a large part of physics and the whole of chemistry are thus completely known, and the difficulty is only that the exact application of these laws leads to equations much too complicated to be soluble."

That sentence is the whole problem, and it has aged extremely well. **Knowing the law is not the same as being able to compute its consequences.** The gap between the two does not shrink as you learn more physics — it grows with system size, and it grows faster than exponentially.

So the honest restatement of the premise is: *we know the rules that govern atoms exactly, and we cannot solve them for more than a few hundred atoms at the accuracy biology needs.*

Everything below follows from that.

---

## 2. Wall 1 — Time

This one is just arithmetic, and the arithmetic is brutal enough to settle the question on its own.

### The setup

Classical molecular dynamics (MD) — already a large approximation, not quantum mechanics — integrates Newton's equations with a timestep of about **2 femtoseconds** (2 × 10⁻¹⁵ s). The timestep is set by the fastest motion in the system, which is hydrogen bond vibration. You cannot make it much larger without the integration going unstable.

The fastest special-purpose hardware built for this problem, **Anton 3**, simulates systems of *millions of atoms at speeds of microseconds per day*. Take that as our budget.

### The arithmetic

Throughput, being generous:

```
5 × 10⁶ atoms  ×  (1 µs/day ÷ 2 fs/step)
= 5 × 10⁶ atoms  ×  5 × 10⁸ steps/day
≈ 2.5 × 10¹⁵ atom-steps per day
```

Now price a **minimal cell**, JCVI-syn3A — the simplest self-replicating organism ever built, ~400 nm across:

```
volume       = (4/3)π(200 nm)³ ≈ 3.4 × 10⁻¹⁷ L
water        = 55 mol/L × 3.4 × 10⁻¹⁷ L × 6.02 × 10²³ ≈ 1.1 × 10⁹ molecules
             → ~3.3 × 10⁹ atoms of water
+ macromolecules, ions, metabolites
TOTAL        ≈ 10¹⁰ atoms
```

One cell cycle is 105 minutes:

```
6,300 s ÷ 2 × 10⁻¹⁵ s/step ≈ 3.2 × 10¹⁸ steps

10¹⁰ atoms × 3.2 × 10¹⁸ steps ≈ 3 × 10²⁸ atom-steps
```

Divide:

```
3 × 10²⁸ ÷ 2.5 × 10¹⁵ ≈ 10¹³ days
```

The universe is about 5 × 10¹² days old.

> **Simulating one cell cycle of the simplest possible organism, atom by atom, on the fastest machine ever built for the purpose, takes on the order of the age of the universe.** My assumptions could be off by an order of magnitude in either direction and it wouldn't matter.

### Why you can't compute your way out

The instinct is "hardware gets faster." Run the numbers on that too:

| Speedup over Anton 3 | Time for one minimal cell cycle |
|---|---|
| 1× (today) | ~10¹³ days (~30 billion years) |
| 1,000× | ~10¹⁰ days (~30 million years) |
| 1,000,000× | ~10⁷ days (~30,000 years) |
| 1,000,000,000× | ~10⁴ days (~30 years) |

A **billion-fold** speedup — far beyond any credible projection — gets you to thirty years for a single cell cycle of a single cell, with no parameter scan, no replicates, and no ability to try a second condition.

And note what that billion-fold-faster machine is still running: **classical** MD with empirical force fields, not quantum mechanics. The thing the question actually proposes — computing from atomic properties — is many further orders of magnitude worse. Density functional theory scales as roughly O(N³) and is practically limited to hundreds to a few thousand atoms for a single energy evaluation. Coupled-cluster theory, the accuracy standard, scales as O(N⁷) and tops out around tens of atoms.

**Wall 1 alone ends the bottom-up route for whole systems.** But it is not the most interesting wall.

---

## 3. Wall 2 — Accuracy, and the cancellation problem

Suppose time were free. You still couldn't do it, because of how biological quantities are built.

### Everything that matters is a small difference between large numbers

Protein folding is the clean example. For a small protein:

```
ΔH (enthalpy of folding)   ≈ −100 kcal/mol
−TΔS (entropy term)        ≈ +90 kcal/mol
                             ─────────────
ΔG (what determines folding) ≈ −10 kcal/mol
```

The two terms are each ~100 and they cancel to ~10. **A 10% error in either term produces a 100% error in the answer** — potentially a sign error, meaning your protein folds when it shouldn't or doesn't when it should.

### The exchange rate between energy error and rate error

This is the number to internalize. At body temperature:

```
RT = (1.987 cal/mol·K)(310 K) = 0.616 kcal/mol
RT · ln(10) = 1.42 kcal/mol
```

> **Every 1.4 kcal/mol of error in a free energy is a factor of 10 error in the resulting rate or equilibrium constant.**

Now compare against what computation can actually deliver:

| Method | Typical error | Resulting error in rate |
|---|---|---|
| "Chemical accuracy" (the aspirational target) | 1 kcal/mol | ~5× |
| Good DFT on reaction barriers | 2–5 kcal/mol | 10²–10³× |
| Classical force field, binding free energy | 1–2 kcal/mol at best | 5–25× |
| Classical force field, difficult systems | 3–5+ kcal/mol | 10²–10³× |

So a bottom-up computed rate constant is routinely wrong by two to three orders of magnitude. A measured one is wrong by 10–20%.

**And this error does not average out as you compose.** A pathway of ten steps, each with a 10× rate error, does not give you a 10× error in the pathway — it gives you something unrecognizable, because the errors are systematic and they interact through the network.

---

## 4. Wall 3 — The bottom is already empirical

This is the wall that most undermines the premise, and it's the one people are usually surprised by.

**Classical force fields are not derived from atomic properties. They are fitted to experiments.**

A force field is a functional form — springs for bonds, cosines for torsions, Lennard-Jones for van der Waals, point charges for electrostatics — whose thousands of parameters are tuned to reproduce quantum calculations on small molecules *and* experimental measurements: densities, heats of vaporization, solvation free energies, NMR observables.

So "starting from atoms" in practice means "starting from parameters someone fitted to measurements of small molecules, and hoping they transfer to your system." Often they don't:

- **RNA force fields are notably less reliable than protein force fields.** Base stacking energies and backbone torsional preferences have been repeatedly re-parameterized, and different force fields give qualitatively different behavior for the same RNA.
- **Magnesium is a known weak point**, and RNA folding is strongly Mg²⁺-dependent. Divalent ions polarize their surroundings; most production force fields have fixed point charges and cannot represent that.
- **Polarization in general** is absent from most force fields. Polarizable versions exist and are substantially more expensive.

Here is the sharpest version of the point, and it's specific to the exact example in the question:

> **RNA folding prediction already tried the bottom-up route, and the empirical route won.** Production RNA folding software does not compute base-pair stacking energies from quantum mechanics. It uses **nearest-neighbor thermodynamic parameters measured experimentally**, by melting short synthetic duplexes in a spectrophotometer and fitting the curves. Those measured parameters remain the standard because computed ones are not accurate enough to replace them.

So the claim "we don't understand what folds RNA" is worth refining. We understand the *physics* of RNA folding qualitatively very well — base pairing, stacking, backbone entropy, electrostatic screening by ions. What we lack is *quantitative parameters accurate enough for specific predictions*. And when people went down a level to compute those parameters from atoms, the answer came back less accurate than just measuring them.

That is not a one-off. It is the general pattern, and Wall 4 explains why.

---

## 5. Wall 4 — Composition loses information, provably

The first three walls are quantitative: too slow, too imprecise, secretly empirical. This one is structural, and it's the real answer to "why not build composites upward."

When you coarse-grain — replace a group of atoms with a single effective particle, or replace a molecule with a rate constant — you **integrate out** degrees of freedom. The resulting effective interaction is not an energy. It is a **free energy**, and free energies depend on the conditions under which you integrated.

Two rigorous consequences follow, both well known in the coarse-graining literature:

### The transferability problem
A coarse-grained model derived at one temperature, concentration, or ionic condition **is not valid at another**, because the averaged-out degrees of freedom behaved differently there. You cannot derive it once and reuse it. Every state point needs re-derivation, or re-measurement.

### The representability problem
A coarse-grained model fitted to reproduce one observable will generally get others wrong. Match the structural distributions and you typically miss the thermodynamics; match the thermodynamics and you miss the pressure or the dynamics. **You cannot generally get a coarse-grained model that is simultaneously right about everything the underlying model was right about.** Information was destroyed, and which information you keep is a choice you make when you fit.

> **This is why "compose upward" does not work even in principle, independent of compute.** Each level of description has to be re-anchored to observations at that level. The composition operator is lossy, and what it loses is exactly what you'd need to get the next level right for free.

This is not a biology problem. It is the reason we have thermodynamics as well as statistical mechanics, why we compute bridge stress from measured Young's moduli rather than from electronic structure, and why nobody derives the properties of water from quantum chromodynamics even though water is, in the end, made of quarks.

---

## 6. The counter-example, and what it actually teaches

The strongest argument against bottom-up isn't any of the walls. It's what happened when someone solved one of these problems for real.

**Protein structure prediction was the flagship bottom-up problem for fifty years.** The physics-based program — compute the energy landscape, find the minimum — ran from the 1970s and never got there. The problem was solved, decisively, by **AlphaFold**, which does not simulate physics at all. It learns from evolutionary sequence data and from the corpus of experimentally determined structures.

The lesson is not "physics is useless." The lesson is sharper:

> **The win came from data at the level of the question being asked, not from computation at a level below it.**

AlphaFold works because the Protein Data Bank exists — because for fifty years people experimentally determined ~200,000 structures at the level of "what shape is this protein," which is the level the question was asked at. The bottleneck was never physical understanding. It was data at the right level.

This is precisely the argument for the rest of this project's approach, and it's why the companion document [`L2_SEQUENCE_TO_RATE.md`](./L2_SEQUENCE_TO_RATE.md) is about *measurement programs* rather than *simulation programs*. If you want to know how fast a promoter fires, the route is not to simulate the polymerase from atoms. It is to measure a million promoters.

---

## 7. Where bottom-up genuinely wins

Being fair to the idea, because it is the right tool for a real and important set of problems:

| Use | Why it works there |
|---|---|
| **Mechanism of a single reaction step** | Small system, short timescale, and you want the mechanism rather than a precise rate. QM/MM is the standard tool and it's excellent. |
| **Ion binding sites, protonation states** | Localized, quantum effects genuinely matter, and experiment is often ambiguous. |
| **Conformational change in one domain** | Microsecond–millisecond MD reaches this now. |
| **Drug binding poses** | Structure is more forgiving than rate; relative free energies within a congeneric series are much easier than absolutes. |
| **Generating training data for higher levels** | The best modern use. Run quantum calculations on many small systems, train a model on them, use the model where you couldn't afford the calculation. |
| **Where no experiment is possible** | Transition states, short-lived intermediates, conditions you can't reach in a lab. |

Note the pattern: **bottom-up works for small, well-posed, local questions.** It fails at composition, not at physics.

---

## 8. What would change this picture

Three developments are worth tracking honestly, because two of them are real and one is overrated for this purpose.

### Machine-learned interatomic potentials — genuinely a step change
Neural network potentials trained on quantum calculations now reach near-quantum accuracy at roughly **an order of magnitude slower than classical force fields, and many orders of magnitude faster than quantum methods**. Foundation-scale versions trained on tens of millions of quantum geometries across most of the periodic table are now routine.

This substantially attacks **Wall 2** (accuracy) and partly **Wall 3** (empiricism), because the parameters now come from quantum calculations rather than from fitting to experiments.

**But it does almost nothing to Wall 1.** A ten-times-slower-than-classical potential makes the age-of-the-universe calculation *worse*, not better. Better energy functions do not solve a sampling problem.

> **There are two separate gaps — accuracy and sampling — and machine-learned potentials close one of them.** The one they don't close is the binding constraint.

### Quantum computing — overrated for this specific problem
Quantum computers may eventually solve electronic structure efficiently, which would help with Wall 2 for small systems. **It does not touch Wall 1**, because the timescale problem is a classical sampling problem: you need 10¹⁸ sequential timesteps, and that sequence is not something quantum parallelism speeds up.

### Enhanced sampling and coarse-graining — helps, at a cost
Methods that bias the simulation to cross barriers faster are mature and useful. But every one of them introduces approximations and requires you to choose in advance *what* to accelerate — which means you have to already know what matters. And coarse-graining, which is the other route to longer timescales, walks straight into Wall 4.

---

## 9. The synthesis: this is how physics works, not a failure of biology

The framing worth adopting: **nature is organized into levels, and each level has its own effective theory with its own parameters, measured at that level.** This is not a workaround. It is the structure of successful science everywhere:

| Level | Effective theory | Parameters measured at that level |
|---|---|---|
| Quarks | QCD | — |
| Nuclei | Nuclear models | Binding energies |
| Atoms | Quantum chemistry | Orbital energies, spectra |
| Molecules | Force fields | Bond constants, partial charges |
| Macromolecules | Folding models | Nearest-neighbor parameters |
| Molecular machines | Kinetic schemes | Rate constants |
| Cells | Network models | Expression rates, fluxes |

Nobody computes the row below from the row above in production. Each row is anchored by measurements at its own scale, and the rows below explain *why* the parameters have the values they do, without being able to produce them to sufficient precision.

**The right question is therefore not "what is the lowest level we understand?" but "what is the highest level at which we can measure the things we need?"** For the genetic code, that level is remarkably high: rate constants for polymerases and ribosomes, initiation rates for promoters, half-lives for transcripts. These are directly measurable, and a measured value beats a computed one by two to three orders of magnitude in accuracy.

Which flips the original intuition:

> Start at the **highest** level where measurement is possible, not the **lowest** level where the physics is known.

---

## 10. What this means for Primordia

Concrete architectural consequences, all of which are already in the plan and now have their justification recorded:

1. **Atoms are geometry, never dynamics.** Structures from the PDB and AlphaFold DB are rendered as coordinates. No atom in this app has a force acting on it. Trying would be a category error costing orders of magnitude for a less accurate answer.

2. **Parameters are measured, not computed.** The provenance system ranks `Measured` above `Predicted` for exactly the reason in §3: the exchange rate between energy error and rate error makes computed rate constants nearly worthless at this scale.

3. **The layered architecture is correct, and this is why.** L0–L6 in the accuracy roadmap are not a convenience. They are effective theories, and Wall 4 says each must be anchored independently.

4. **Each tier gets its own validation.** Because composition is lossy, a model validated at one tier is not thereby validated at the next. Tier 1 passing V6 says nothing about whether Tier 3 is right.

5. **Better physics arrives as better parameters, not as a new architecture.** If machine-learned potentials eventually produce reliable nearest-neighbor RNA parameters or enzyme rate constants, they enter through the predictor plugin interface as one more `Predicted` source with a benchmarked error. Nothing about the app changes.

6. **The honest place to spend effort is data curation**, not simulation depth. That is what [`L2_SEQUENCE_TO_RATE.md`](./L2_SEQUENCE_TO_RATE.md) works out in detail.

---

## Summary

| Wall | What it says | Can compute fix it? |
|---|---|---|
| **1. Time** | One minimal cell cycle at atomic resolution ≈ age of the universe on the best hardware built | No — a billion-fold speedup still leaves 30 years for one run |
| **2. Accuracy** | Biological quantities are small differences between large numbers; 1.4 kcal/mol of error = 10× in rate | Partly — machine-learned potentials help here |
| **3. Circularity** | Force fields are fitted to experiments, so "bottom-up" is already empirical | Partly — quantum-trained potentials reduce this |
| **4. Composition** | Coarse-graining is provably lossy; effective interactions don't transfer across conditions or observables | **No — this is structural, not computational** |

**The answer to the question:** we do understand atoms completely, and it doesn't help, because understanding the law is not the same as computing its consequences, and because the operation of building composites destroys exactly the information that would make the next level free. The level to start at is the highest one where you can *measure* what you need — which, for the genetic code, is quite high up.
