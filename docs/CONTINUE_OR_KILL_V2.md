# CONTINUE / PIVOT / KILL — V2

Status: **GATE 0.9 DECISION INSTRUMENT.** Version 0.1.0.
Supersedes `CONTINUE_OR_KILL.md`, whose thresholds were stated without power analysis and whose
hard kill (B-20 misattribution) demanded an unearnable verdict.

**Every threshold below carries: sample size, minimum detectable effect, false-positive risk, and
power.** A threshold without those four is a preference wearing a number.

---

## 0. Settled before any experiment

| # | Finding | Source | Consequence |
|---|---|---|---|
| **G1** | **All five integrity dimensions have mature direct analogues** — mutation testing, flaky-test infrastructure, contamination detection, hermetic build systems, construct-validity/DOE. | `GATE_0_9_RESEARCH_VALIDITY` §B.1 | Every individual capability is commodity. Only the composition can differentiate, and rule 4 says composition must be *hard*. |
| **G2** | **BASELINE-C** — five existing techniques run independently and unioned — is assemblable by a competent engineer in **1–3 weeks**. | §B.3 | This, not a research system, is the thing to beat. |
| **G3** | B-20 as specified in Gate 0.75 was **not observationally identifiable**. The kill criterion demanded an impossible verdict and scored the correct behaviour as catastrophic. | `IDENTIFIABILITY_AND_FALSIFICATION` §0 | The defect class is real — we committed it ourselves, and it survived three adversarial passes. |
| **G4** | BH-FDR at q = 0.05 **controls FDR under all five dependence structures tested**; BY costs half the power for nothing; power requires z ≈ 4, i.e. δ ≈ 0.10 at 1,340 trials per cell, with δ < 0.08 unreachable. | `STATISTICAL_VALIDITY` §3–4 | The statistics are sound and the audit's own recommendation is "keep what you had". `OQ-22` resolved. |

---

## 1. Criteria, with power

All measured on `BENCHMARK_V2.md`: 27 cases × 10 replications = **270 paired observations per
arm**, scored once against a sealed registry.

---

### K-A — The composed-baseline gap **(the project)**

**Measures T\*\*(c): does the integrated audit detect defects BASELINE-C misses?**

| | |
|---|---|
| Primary statistic | `GAP = detection(A-INT) − detection(BASELINE-C)`, paired by case |
| Sample size | 270 paired observations; **27 on the cross-dimension cases V2-25..27** (3 × 10) |
| Test | McNemar on discordant pairs |
| **Power** | With ψ = 3 discordance ratio, **29 discordant pairs** give power 0.80 at α = 0.05. With ψ = 2, **69** are needed. |
| **Minimum detectable effect** | At N = 270, Wilson half-width is **± 0.050**. At N = 27 (cross-dimension only) it is **± 0.16** — enough to distinguish a gap of 0.60 from 1.00, **not** enough to distinguish 0.80 from 0.90. |
| False-positive risk | α = 0.05 on a single pre-declared primary test. No secondary test is promoted. |

| Result | Decision |
|---|---|
| `detection(BASELINE-C) / detection(A-INT) ≥ 0.80` **overall**, upper CI bound above 0.75 | **KILL** |
| Cross-dimension gap ≤ 0.20 (A-INT catches ≤ 1 more of the 3 cross-dimension classes) | **KILL** |
| Overall ratio ≤ 0.60 **and** cross-dimension gap ≥ 0.40 | **PASS** |
| Between | MARGINAL — treated as FAIL (§3, rule 3) |

> **The honest limitation, stated here rather than after the run:** three cross-dimension cases at
> 10 replications gives ± 0.16. That resolves "clear gap" from "no gap" and nothing finer. If the
> true gap is 0.25, this experiment cannot establish it, and the decision defaults to KILL.
> **Increasing cross-dimension cases from 3 to 12 would halve the interval and is the single
> cheapest improvement to this instrument.**

---

### K-B — Falsification correctness

**Measures whether the engine runs the protocol rather than having memorised one answer.**
Cases V2-10 (falsify-then-refuse), V2-11 (refuse-to-experiment), V2-12 (support),
V2-13 (non-surgical), V2-09 (no candidate), V2-06 (undetectable) — 6 cases × 10 = **60 observations**.

Two error rates:
- **Unearned attribution** — attributing without a successful intervention
- **Unearned refusal** — `INSUFFICIENT_OBSERVABILITY` without a complete falsification record

| | |
|---|---|
| Sample size | 60 |
| **Power** | Observing **≤ 4 / 60** errors gives a 95% upper bound of **0.159 < 0.20** → PASS. Observing 0/40 gives 0.088; 2/40 gives 0.165. **n = 60 is the minimum that tolerates any error at all while still resolving the threshold.** |
| Minimum detectable effect | Distinguishes a true rate of 0.05 from 0.25 |
| False-positive risk | One-sided 95% |

| Result | Decision |
|---|---|
| Both error rates' upper bounds < 0.20 | **PASS** |
| Either upper bound > 0.20 | **KILL** |

> This replaces Gate 0.75's CMR criterion, which demanded an unearnable verdict. It now scores the
> **route**, not only the answer.

---

### K-C — Oracle-validity instrument works

| | |
|---|---|
| Statistic | Mutation score per oracle on its 63-mutant suite |
| Sample size | **63 mutants × 9 oracles = 567** |
| **Power** | 63 mutants per arm distinguishes a score of 0.90 from 0.70 at power 0.80. Resolving 0.90 vs 0.80 needs **200**; 0.90 vs 0.85 needs **686**. |
| Minimum detectable effect | **0.20** in mutation score |
| PASS | The instrument correctly flags all four seeded weak oracles (V2-14..17) **and** does not flag oracles that pass their suites |

Failure deletes the oracle contribution; it does not kill the project.

---

### K-D — Substrate transfer

| | |
|---|---|
| Statistic | `TRANSFER_YIELD` = L-DET findings reproducing in L-STO at ≥ 50 % |
| Sample size | **n = 30** per finding |
| **Power** | 30/30 → one-sided 95 % lower bound **0.92**; 24/30 → **0.66**; 15/30 → **0.36**. n = 20 gives 0.88 / 0.62 / 0.33 — nearly as good, so n = 30 is chosen for margin, not necessity. |
| Minimum detectable effect | Separates ≥ 0.92 from ≤ 0.66 |
| KILL | yield < 0.30, **or `OQ-16` resolves that no local model exists** |

---

### K-E — False-alarm rate (the criterion that stops us inventing findings)

Cases V2-01, V2-02, V2-03, V2-23 (valid evaluations and legitimate stochasticity) — 4 × 10 = **40**.

| | |
|---|---|
| **Power** | 0/40 → upper bound **0.088**; 2/40 → **0.165** |
| PASS | Upper bound < 0.10, i.e. **≤ 1 false `INVALID_EVALUATION` in 40** |
| KILL | Upper bound > 0.25 |

> **An audit that fires on valid evaluations is worse than no audit**: it trains its users to
> ignore it, and it does so while appearing rigorous. V2-06 and V2-23 exist specifically to make
> this measurable.

---

### K-F — Workflow ownership (non-technical, and promoted deliberately)

| | |
|---|---|
| Statistic | A named person or team who will run the audit on an evaluation they own |
| Window | **60 days** |
| KILL | None identified |

Not measurable by simulation, not resolvable by better engineering, and it has appeared as a
footnote in every prior gate. Promoting it to a kill criterion is the only way it gets tested
rather than noted.

---

## 2. Decision rule

First matching row wins.

| # | Condition | Decision |
|---|---|---|
| 1 | K-E FAIL (false alarms on valid evaluations) | **HALT** — nothing else is measurable |
| 2 | **K-A FAIL** (BASELINE-C ≥ 0.80, or cross-dimension gap ≤ 0.20) | **KILL.** T\*\*(c) fails. Write up the composition as a short report and stop. |
| 3 | K-B FAIL (unearned attribution or refusal > 0.20) | **KILL.** It reproduces the error it exists to prevent. |
| 4 | K-D FAIL (transfer < 0.30, or no local model) | **KILL**, or PIVOT to an explicitly runtime-only tool if an owner exists |
| 5 | K-F FAIL after 60 days | **KILL** |
| 6 | K-A PASS, K-B PASS, K-C FAIL | **PIVOT** — drop the oracle-validity contribution, keep cross-dimension + falsification |
| 7 | All PASS | **CONTINUE**, scoped to the MVDC only (`GATE_0_9_RESEARCH_VALIDITY` §E) |

---

## 3. Anti-gaming

1. Thresholds, the case registry and the §4 prediction table are **digest-recorded before the run**.
2. **Scoring happens once.** No re-scoring after seeing results.
3. **MARGINAL is FAIL** for K-A and K-B.
4. No case may be added after results are seen. Removal only for demonstrable malformation, reported.
5. **BASELINE-C must be implemented to each component's own literature standard.** A weak baseline
   is the easiest way to fake K-A, and it is the failure mode we are most exposed to
   (`BENCHMARK_V2` B3). Ideally its implementation is reviewed by someone not invested in the
   outcome.
6. A KILL leaves the artifacts in the repository with the result attached.

---

## 4. Pre-registered prediction

`HYPOTHESIS`, recorded so the result can contradict it.

| Criterion | Predicted | Confidence |
|---|---|---|
| K-E false alarms | PASS | high |
| K-C oracle instrument | PASS | high |
| K-B falsification correctness | PASS | medium — refusal is harder to engineer than attribution |
| K-D transfer | UNKNOWN | none — genuinely open |
| **K-A composed-baseline gap** | **~35 % chance of PASS** | This is the project. |
| K-F owner within 60 days | **UNKNOWN, and pessimistic** | — |
| **Decision** | **Row 2 (KILL) is the single most likely outcome; row 7 (CONTINUE, MVDC-scoped) is the second.** | — |

**Why 35 %.** For K-A to pass, cross-dimension defects must be both *constructible* and
*genuinely invisible* to five independent tools. The construction argument in
`EVALUATION_INTEGRITY_MODEL` §6 is plausible but untested, and the most likely failure is mundane:
BASELINE-C's residual-regression component picks up the environment × oracle interaction as an
ordinary association, and the gap collapses.

---

## 5. Cost of running this decision

| Item | Cost |
|---|---|
| Benchmark environment + 27 sealed defects (independently authored) | dominant engineering cost |
| BASELINE-C, five components to literature standard | **substantial and unavoidable — the baseline is the experiment** |
| Trials: 7 arms × 27 cases × 10 replications, plus mutation suites and falsification runs | ≈ 10⁵ deterministic trials, **2–4 days** serial at 2 s |
| L-STO transfer | local inference only |
| **Remote LLM calls** | **0** |

> The decision costs zero inference budget and a few days of compute. **Its real cost is building
> BASELINE-C honestly** — and that is also the only way the answer means anything.
