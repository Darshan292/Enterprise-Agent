# RESEARCH VALIDITY CHALLENGE

Status: **GATE 0.75 — RESEARCH VALIDITY. IMPLEMENTATION NOT AUTHORIZED.**
Document version: 0.1.0
Date: 2026-09-22

This document exists to **disprove** the project's thesis, not to defend it. Where it fails to
disprove, it records the narrowest surviving claim, not the most flattering one.

---

## A. Exact falsifiable thesis

### A.1 The thesis as stated

> "Failures in tool-using AI agents can emerge from interactions between agent decisions,
> execution history, environment state, tool semantics, context state, authorization state, and
> runtime conditions. A useful system should discover, characterize, reduce and regression-test
> these interaction-dependent failures."

### A.2 Why that statement is not testable as written

It bundles two claims of very different strength:

- **T1 (existence).** Interaction-dependent failures exist. **Almost certainly true and almost
  certainly trivial.** Every distributed system has them; every textbook documents
  timeout + retry + eventual consistency. Confirming T1 confirms nothing about this project.
- **T2 (utility).** A system doing discover/characterize/reduce/regression-test is *useful* and
  *not already available*. This is the real claim, and as written it is unfalsifiable because
  "useful" is undefined and "a system" does not say *better than what*.

A thesis that cannot fail is not a thesis. Restated:

### A.3 T* — the falsifiable form

> **T\*:** There exists a class **F** of tool-using-agent failures such that, for a non-trivial
> fraction of **F**:
>
> **(a) Multiplicity.** Each failure requires ≥ 2 simultaneously-active conditions drawn from
> distinct layers (agent decision / execution history / environment state / tool semantics /
> context state / authorization state / runtime condition).
>
> **(b) Not single-factor-reachable.** Not discovered by single-factor fault sweeps at equal
> trial budget.
>
> **(c) Not random-reachable.** Not discovered by **uniform random configuration sampling plus
> standard interaction regression** at equal trial budget. *(This is the null hypothesis that
> matters. §C.1 shows it is far stronger than Gate 0.5 assumed.)*
>
> **(d) Characterizable.** The minimized description names the responsible conditions, excludes
> decoys, and reproduces at a measured rate.
>
> **(e) Externally real.** **F** is non-empty for agents **we did not write**, under factor
> dimensions **we did not choose**.

**Each of (a)–(e) is independently falsifiable.** T\* fails if any one fails. (e) is the hardest
and the one the project has never tested.

### A.4 The three existential risks, restated as tests against T\*

| Risk | Falsifies | Test |
|---|---|---|
| 1. Rebuilding deterministic fault-testing infrastructure | (b), (c) | §C + benchmark arms A0–A3 |
| 2. The deterministic substrate tests our own harness | (e) | L-DET → L-STO transfer rate, `SUBSTRATE_VALIDITY.md` |
| 3. The factor model is circular | (e) | Absent-factor benchmark class, `FACTOR_MODEL_ADEQUACY.md` |

---

## B. What would prove the thesis weak or false?

Ordered by how cheaply each can be checked. **Three of the five are checkable with arithmetic or
a small benchmark, before any infrastructure exists.**

| # | Falsifier | Falsifies | Cost to check | Status |
|---|---|---|---|---|
| B1 | Uniform random sampling at equal budget covers substantially the same interaction space | (c) | **Arithmetic. Free.** | **CHECKED — see §C.1. Partially falsified.** |
| B2 | Interaction failures are found by existing fuzzing/chaos pipelines at comparable cost | (b),(c) | Benchmark arms A0–A3 | Designed, not run |
| B3 | Discoveries in the deterministic substrate do not reproduce in a stochastic agent | (e) | Transfer test, n = 30 | Designed, not run |
| B4 | The system returns confident explanations for failures whose mechanism is outside its factor model | (d),(e) | Benchmark absent-factor class | Designed, not run |
| B5 | Reduction offers no measurable advantage over ddmin at equal cost | (d) | `REDUCTION_VALIDITY.md` criteria | Designed, not run |

### B.1 — B1 is checked, and it substantially falsifies the Gate 0.5 framing

Gate 0.5 presented covering arrays as the mechanism that makes interaction discovery tractable,
citing a ~1.5 × 10⁵ reduction versus full factorial. **That comparison is against the wrong
baseline.** Nobody proposed running a full factorial. The real baseline is uniform random
sampling at the same trial budget.

Computed over the declared 14-factor space (level counts 6,3,3,5,2,2,5,4,3,3,3,2,3,2; 91 factor
pairs; 972 two-way level-combinations):

| N uniform-random configurations | Expected fraction of 2-way level-combinations covered |
|---|---|
| **45** (= the designed pairwise array size) | **95.4 %** |
| 77 | 99 % |
| 133 | 99.9 % |
| 237 | 100 % |

> **`FACT` (arithmetic, reproducible from the level counts alone): at the same budget as the
> designed pairwise array, uniform random sampling already covers 95.4 % of all two-way level
> combinations. The designed array's entire contribution is the remaining 4.6 %. Reaching 99 %
> coverage by random sampling costs 77 trials against the array's ~45 — an efficiency factor of
> 1.7 ×.**

At three-way strength it is worse. Random sampling reaches 99 % three-way coverage in ≈ 289
configurations; the assumed designed array is ≈ 350. **At t = 3, over this factor-space shape,
the designed array has no measurable advantage over random at all.**

**Consequences, stated without softening:**

1. "Covering-array testing for agents" **is not a thesis.** It is a 1.7 × constant-factor
   optimization over a free baseline. Any document presenting it as the core mechanism is
   overstating it by an order of magnitude.
2. Condition (c) of T\* is **not satisfied by the screening design**. Random sampling finds the
   same two-way failures at 1.7 × the budget, and trials are cheap (≈ 2 s, zero remote calls).
   1.7 × of cheap is cheap.
3. The covering array should be **retained** — 1.7 × free is worth having, and coverage
   *accounting* is exact rather than probabilistic with a designed array — but **demoted from
   thesis to default**.
4. Whatever differentiates this project, **it is not the sampling design. It must be the
   analysis, or something outside the level-combination space entirely.**

This finding was available at Gate 0.5 from arithmetic over numbers already in the documents. It
was not computed. That is a process failure worth recording: the project compared itself to the
baseline that flattered it.

---

## C. Existing systems that already solve substantially the same problem

`ASSERTED_UNVERIFIED` for all product capabilities — characterisations from general knowledge of
the space, not a current feature audit. `OQ-02` requires a documentation check before Gate 1.

### C.1 The null hypotheses, ranked by threat

| Arm | System | What it does | Does it satisfy T\*(b)(c)(d)? |
|---|---|---|---|
| **A2** | **Uniform random sampling + logistic interaction regression** | Standard DOE / statistics. Free. No tooling required beyond a sampler and a regression library. | **(b) yes. (c) by definition. (d) yes — under uniform assignment a decoy is independent of the true cause, so regression rejects it.** |
| A3 | ACTS/PICT (covering arrays) + ddmin | Existing CIT tooling composed with an existing reducer | (b) yes, (c) 1.7 × better than random, (d) partially — ddmin alone does not reject decoys |
| A1 | Fuzzing loop (ClusterFuzz-shaped): random inputs → minimize → bucket → regression | Mature, enormous scale | (b) yes, (c) yes, (d) partially — hands you a reproducer, not an attribution |
| A0 | Chaos engineering (Gremlin/Chaos Mesh) | Fault injection, blast radius | (b) **no** — single-factor oriented. The only arm T\* clearly beats. |
| — | Deterministic simulation (Antithesis, FDB lineage, VOPR) | Explore fault combinations, minimize, deterministically reproduce | (b)(c)(d) **yes**, with a stronger determinism story than ours |

### C.2 The finding that hurts most

**A2 is not a product. It is two standard techniques.** And A2 satisfies (b), (c) and (d).

Under uniform random assignment, every factor is independent of every other by construction.
A decoy — a factor correlated with failure but not necessary for it — is therefore *not*
correlated with the true cause in the sample, and a logistic model with interaction terms will
assign it a coefficient indistinguishable from zero. Decoy rejection does not require a designed
experiment. **It requires doing the analysis at all, which fuzzers and chaos tools do not — but
which costs a sampler and `statsmodels`.**

> **If you delete the covering array and substitute uniform random sampling plus interaction
> regression, you lose a 1.7 × efficiency factor and retain 100 % of the epistemic value.**
>
> That means the design is an optimization and the analysis is standard. Neither is a thesis.

### C.3 What is genuinely NOT covered by any arm

Three things survive C.2. They are the only candidates left.

| Survivor | Why no arm covers it |
|---|---|
| **S1 — Failures outside the level-combination space** (history/sequence-dependent, interleaving-dependent) | A0–A3 all sample *assignments of levels to factors*. A failure requiring a specific **ordering** or **interleaving** is not an assignment. Random sampling over levels never reaches it; neither does a covering array. Reaching it requires sequence/state-machine generation — which exists (stateful Hypothesis, model-based testing) but is a **different** baseline that must be added as an arm. |
| **S2 — Factor-model adequacy adjudication** | **No arm attempts this.** A2 fits a model *conditional on the declared factors*; it has no mechanism to say "your factor set is incomplete." DOE assumes you know your factors. Fuzzers have no factor model at all. Chaos tools have a fault list, not a model. **Nobody ships an instrument that returns `UNEXPLAINED` / `UNKNOWN_MODEL`.** |
| **S3 — Stochastic reduction validity and the stochastic regression artifact** | ddmin assumes a deterministic test. At a failure reproducing at p = 0.4, a single-sample accept/reject decision wrongly rejects a genuinely-failing candidate **60 %** of the time. Existing reducers do not quantify this. The artifact format for "reproduces at 0.4 ± CI, run r = 6 times" does not exist in any standard harness. |

**S2 is the only one of the three that is not an engineering refinement of an existing method.**

---

## D. The smallest defensible differentiation

Stated as narrowly as the evidence supports. Anything broader is contradicted by §C.

> **Given an observed agent failure and a declared model of what can vary, adjudicate whether
> that model explains the failure — returning `EXPLAINED`, `PARTIALLY_EXPLAINED`, `UNEXPLAINED`
> or `UNKNOWN_MODEL` — while rejecting decoys and refusing to attribute when the mechanism lies
> outside the model.**

Three components, in descending order of defensibility:

| # | Component | Defensible against | Vulnerable to |
|---|---|---|---|
| **D1** | **Factor-model adequacy verdict**, derived from residual diagnostics against **ambient observables that are recorded but not declared as factors**. | Every arm in §C.1. Nobody attempts it. | "Nobody attempts it because nobody wants it" (`CONTINUE_OR_KILL` K4). |
| **D2** | **Refusal to attribute.** On a failure whose mechanism is absent from the model, return `UNKNOWN_MODEL` rather than confidently naming a decoy. | A1/A3 (no attribution at all); A2 (will happily fit a decoy if the true cause is unobserved and the decoy is not independent of it under a **non-uniform** sampler). | Itself, if the system attributes anyway. This is the make-or-break test. |
| **D3** | **Stochastic reduction + stochastic regression artifact.** `r_confirm`, fingerprint-aware acceptance, measured reproduction rate with a derived repetition count. | Naive ddmin, and every harness that reports pass/fail on a flaky failure. | Being a two-week engineering addition to an existing reducer. |

**What is explicitly NOT differentiation, per §B.1 and §C.2:**
covering arrays · fault injection · minimization · fingerprinting · regression compilation ·
interaction detection · coverage accounting. All of these are either standard or 1.7 × better
than free.

### D.1 The reframe this forces

The product is **not** a failure finder. Finding is commodity.

The product is a **model-adequacy instrument**: it tells you when your understanding of what can
vary is incomplete, and it declines to guess when it is. That is a smaller, stranger, and more
defensible claim than Gate 0.5's, and it is the claim the benchmark is designed to test.

---

## E. What the deterministic substrate (L-DET) can legitimately prove

L-DET = stub planner, frozen fixture, seeded fault injection, byte-reproducible.

**Can prove:**
1. The **adapter and runtime** mishandle a named factor combination. (A claim about our code.)
2. An **oracle** fires or does not fire on a constructed trial. (A claim about the oracle.)
3. A failure is **reproducible** under a recorded assignment, seed and fixture digest.
4. A reduction is **stable** across repeated runs at fixed seed family.
5. **INV-9 (determinism)** holds or does not.
6. A candidate factor set **survives or fails** a confirmatory factorial — i.e. decoy rejection
   *within the substrate*.
7. The **residual** at a fixed cell is non-zero, i.e. something varies that we did not declare.

**Cannot prove — and this is risk #2 stated precisely:**
- That any **real agent** would ever enter the state the stub was scripted into.
- That the factor combination occurs at any rate in real operation.
- That the failure matters.

> **The sharpest formulation of risk #2: in L-DET, the agent's trajectory is authored by us. A
> failure found there is a statement about our runtime under a scenario we wrote. It becomes a
> statement about *agents* only if an unscripted agent reaches the same state.**
>
> That is not an argument for abandoning L-DET — it is the argument for the **transfer test**
> being a gate criterion rather than a nice-to-have.

---

## F. What requires stochastic-agent evidence (L-STO)

A claim needs L-STO when it asserts something about **agent behaviour** rather than about our
runtime.

| Claim | Why L-DET is insufficient |
|---|---|
| "A real agent enters state S under condition X" | L-DET's trajectory is scripted. |
| "This failure is reachable without adversarial scripting" | Same. |
| "The failure rate under factor combination X is r" | L-DET's rate is a property of the script. |
| "Discovery in L-DET transfers" | **This is the definition of the transfer test.** |
| "A history-dependent failure arises from planning, not from our fixture" | Sequence in L-DET is authored. |

### F.1 The transfer test

For each failure discovered in L-DET, attempt reproduction in L-STO with the same factor
assignment and an **unscripted** local agent, n independent attempts.

One-sided 95 % lower bounds on the transfer rate:

| n | all reproduce → lower bound | 80 % reproduce → lower bound |
|---|---|---|
| 10 | 0.79 | 0.54 |
| 20 | 0.88 | 0.62 |
| **30** | **0.92** | **0.66** |
| 50 | 0.95 | 0.69 |

`DESIGN_DECISION`: **n = 30** is the standard transfer test. It distinguishes "transfers well"
(≥ 0.92) from "transfers poorly" (≤ 0.66) — which is the only distinction that matters, and 30
L-STO runs is affordable against a local model.

---

## G. What requires external-model evidence (L-REM)

| Claim | Layer |
|---|---|
| "This failure occurs against a provider-hosted model" | L-REM |
| "Failure rate shifted across model versions" | L-REM, **and only as an association on a date** |
| "Our local model is representative" | L-REM, and it is a **weak** check: n = 5 per task gives a Wilson half-width of ± 0.33 |

> **L-REM is a smoke alarm, not a measurement.** It detects catastrophic divergence and nothing
> finer. Any claim requiring L-REM to be quantitative is not affordable under the budget and must
> not be made.

---

## H. What cannot be established by this project, at all

| # | Cannot establish | Why it is structural, not a resource problem |
|---|---|---|
| H1 | That findings generalise to real enterprise systems | Synthetic environment, our capability profiles, our fault mix. No amount of internal rigour reaches outside. |
| H2 | That the factor model is **complete** | You can detect inadequacy (§D1); you cannot prove adequacy. Absence of residual is consistent with a hidden factor that happens to be constant across every trial run. |
| H3 | Any causal claim about production behaviour | No production data, no intervention on production, no randomisation of anything real. |
| H4 | That a clean run means an agent is safe | Only that the explored region produced no violation at the stated sample sizes. |
| H5 | The real-world **frequency** of any interaction | Frequency is a property of a deployment. We authored the fault distribution. |
| H6 | That the benchmark's hidden mechanisms resemble real hidden mechanisms | **We authored those too.** The benchmark reduces circularity; it does not remove it. Only a mechanism authored by someone with no knowledge of the factor model is a genuine test, which is why `INTERACTION_FAILURE_BENCHMARK.md` §6 makes independent authorship a hard requirement. |
| H7 | That the model-adequacy instrument is *wanted* | A technical instrument with no owner in an org chart is a research artifact regardless of quality. `CONTINUE_OR_KILL` K4. |

**H2 and H6 together are the honest boundary of this project's central claim.** We can say "your
model does not explain this failure." We can never say "your model is complete." And our
confidence that we can detect inadequacy rests on a benchmark we wrote — unless §6's independent
authorship requirement is actually enforced, in which case it rests on a benchmark **one of us**
wrote, which is better and still not independent.
