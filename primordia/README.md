# Primordia

A desktop **genetic code simulator**. Import a real DNA sequence, run the molecular machinery that reads it, and watch what it actually produces — at single-nucleotide, single-molecule resolution, with every number traceable to a source.

This directory currently holds planning documents. No code yet.

## Documents

| File | What's in it |
|---|---|
| [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) | The plan: what the app is, the accuracy contract, architecture, the simulator design, roadmap, budget. **Start here.** |
| [`ACCURACY_ROADMAP.md`](./ACCURACY_ROADMAP.md) | What we do and don't understand about the genetic code, layer by layer, and what would have to change for a complete simulation to become possible. |
| [`WHY_NOT_BOTTOM_UP.md`](./WHY_NOT_BOTTOM_UP.md) | Why the simulator doesn't start from atoms and compose upward — four walls, one of which is structural rather than computational. |
| [`L2_SEQUENCE_TO_RATE.md`](./L2_SEQUENCE_TO_RATE.md) | What data and infrastructure it would take to make sequence-to-rate prediction measurement-limited. A costed thought experiment. |
| [`SETUP_GUIDE.md`](./SETUP_GUIDE.md) | Setting up the machine, including running AI locally so it can drive Blender and the rest of the toolchain. |

## The short version

**What it does.** Give it a sequence, annotations, and a defined molecular environment. It simulates the physical events — every nucleotide added, every codon read — and renders them, with every rate constant carrying a citation and every result carrying an uncertainty band.

**Why it's built this way.** The genetic code splits cleanly into things we know exactly (which codon means which amino acid, how a ribosome works) and things we don't (what an arbitrary protein does, how fast an unmeasured promoter fires). The app's core feature is a provenance system that keeps those apart and refuses to blur them.

**Where it starts.** A reconstituted cell-free expression system: a defined mix of purified components in a tube, where every ingredient is known and published models are validated against measured time courses. It is the most accurately simulatable instance of "the genetic code running" that exists.

**Where it goes.** Encapsulated cell-free → the minimal cell JCVI-syn3A → populations and evolution → speculative scenarios, including the RNA world, which runs on the same engine as ordinary content rather than being an architectural commitment.

## Status

| | |
|---|---|
| Phase | Pre-0 (planning) |
| Next | Answer the three open questions in the plan, then scaffold the repo |
| Stack | Tauri 2 · Rust core · TypeScript/React + three.js UI · Python reference lane |
