# CONTINUE / PIVOT / KILL

Status: **GATE 0.75. DECISION INSTRUMENT.**
Document version: 0.1.0

This is the most important artifact in the repository. Its thresholds were set **before** the
benchmark exists, from arithmetic that is already done, and they are set so that KILL is a
realistic outcome of a competently-run experiment — not a rhetorical gesture.

**Rule of construction:** no threshold in this document was chosen by asking "what would we
likely achieve?" Each was chosen by asking "below what value would a competent outsider say this
was not worth building?"

---

## 0. What is already decided by arithmetic, before any experiment

Three findings are settled and require no benchmark. They constrain the decision space before the
first trial runs.

| # | Finding | Source | Consequence |
|---|---|---|---|
| **F1** | Uniform random sampling covers **95.4 %** of two-way level combinations at the same budget as the designed pairwise array. Reaching 99 % costs **1.7 ×** the array's budget. At t = 3 the array has **no** advantage. | `RESEARCH_VALIDITY` §B.1 | **"Covering-array testing for agents" is dead as a thesis.** It is a 1.7 × optimization over free. |
| **F2** | Under uniform assignment a decoy is independent of the true cause, so **random sampling + interaction regression rejects decoys too.** | `RESEARCH_VALIDITY` §C.2 | The sampling design is an optimization; the **analysis** is standard DOE. Neither is a thesis. |
| **F3** | `r_confirm = 5` alone produces an over-inclusive minimum with probability **0.99** at p = 0.4. It is defensible only because a confirmatory sweep follows. | `REDUCTION_VALIDITY` §2 | Reduction differentiation rests entirely on **M4**, not on stochastic ddmin. |

> **Taken together: three of the seven things Gate 0.5 presented as contributions are, on
> arithmetic alone, either free elsewhere or wrong as specified. The project's case now rests on
> a narrower base than it did a day ago, and that narrowing happened without running anything.**

---

## 1. The seven decision criteria

Each is measured on `INTERACTION_FAILURE_BENCHMARK.md`, all arms at equal trial budget, scored
once against a sealed registry.

---

### K1 — Hidden-failure discovery

**Measures:** whether the adequacy instrument recovers a mechanism that exists in telemetry but
is not a declared factor. Cases B-16, B-17, B-18, each replicated 10 × with independent seed
families.

| Result | Verdict |
|---|---|
| ≥ 2 of 3 cases: `UNEXPLAINED` returned **with the correct ambient observable named** in ≥ 7/10 replications | **PASS** |
| 1 of 3 | MARGINAL |
| **0 of 3** | **FAIL** |

**This is the primary differentiation metric.** No arm in `COMPETITIVE_OVERLAP` attempts it.

---

### K2 — Catastrophic misattribution (the hard kill)

**Measures:** whether the system confidently blames a declared factor for a mechanism it cannot
observe. Cases B-19, B-20, each replicated 10 × ⇒ 20 scored verdicts.

```
CMR = P( verdict ∈ {EXPLAINED, PARTIALLY_EXPLAINED} ∧ named factor ∉ ground truth )
```

| CMR | Verdict |
|---|---|
| ≤ 0.10 | **PASS** |
| 0.10 – 0.20 | MARGINAL — remediate and re-run once |
| **> 0.20** | **KILL, unconditional** |

> **Rationale, stated plainly: a system that names a cause it cannot have observed is worse than
> a system that finds nothing. It terminates investigation with a wrong answer and attaches
> statistics to it. There is no configuration of the rest of the project that compensates, so
> this criterion has no override.**

B-20 — absent mechanism with a correlated decoy — is the single cell that decides this.

---

### K3 — Differentiation against the null (A2)

**Measures:** whether we beat uniform random sampling + logistic interaction regression, which is
two standard techniques and costs nothing.

| Metric | Threshold for PASS |
|---|---|
| Decoy false-inclusion rate (B-13..B-15) | **A4 ≤ 0.5 × A2**, margin surviving a paired test |
| Localization exact-match (all DECLARED classes) | **A4 − A2 ≥ 0.15** |
| Detection rate | *No threshold.* `RESEARCH_VALIDITY` §B.1 predicts a tie; a tie here is expected and is **not** evidence of anything. |

> **Prediction on record (`INTERACTION_FAILURE_BENCHMARK` §4.3): K3 is predicted to FAIL or tie.**
> The prediction is recorded here so that a tie cannot later be narrated as a win.

---

### K4 — Substrate transfer

**Measures:** existential risk #2 — is L-DET testing agents or testing our harness?

```
TRANSFER_YIELD = (L-DET discoveries reproducing in L-STO at ≥ 50 %, n = 30) / (attempted)
```

| Yield | Verdict |
|---|---|
| ≥ 0.50 | **PASS** |
| 0.30 – 0.50 | MARGINAL — L-DET findings may be reported only as runtime claims (C1–C7), never as agent claims |
| **< 0.30** | **KILL** |

**Additional unconditional kill:** if `OQ-16` resolves that **no local model is available**, L-STO
does not exist, C8–C12 become unestablishable, and risk #2 is unmitigable by any means available
to this project. That alone forces KILL or a PIVOT to an explicitly runtime-only product.

---

### K5 — Reachability of the classes outside the level space

**Measures:** survivor S1 — history and concurrency dependence, the only detection-side
differentiation that F1 and F2 leave standing.

Cases B-08 .. B-12 (5 cases).

| Result | Verdict |
|---|---|
| A4 detects **and correctly classifies** ≥ 3 of 5, **and** ≥ 2 more than A2 | **PASS** |
| A4 detects 1–2, or does not beat A5 (stateful model-based testing) | MARGINAL — drop history/concurrency from the claim |
| **A4 detects 0 of 5** | **FAIL** |

Note: **A5 is the fair null here**, not A2. If A4 merely ties A5, we have re-implemented stateful
model-based testing and must say so.

---

### K6 — Reduction advantage

**Measures:** whether M4 earns its 3.7 × cost over M3.

| Metric | PASS |
|---|---|
| Interaction preservation (exact set match), R7 | **M4 ≥ 0.70 and M4 − M3 ≥ 0.15** |
| False reduction rate, R5 | M4 ≤ 0.10 |

Failure here does not kill the project; it deletes the reduction contribution and leaves only
adequacy.

---

### K7 — Harness integrity (gating precondition)

| Metric | PASS |
|---|---|
| Null false-positive rate (B-21, B-22) | ≤ 0.05 |
| INV-9 determinism violations across the benchmark | **0** |
| Isolation canary hits | **0** |

> **K7 is evaluated first. If it fails, no other criterion is scored, because every other number
> is then a measurement of our own bugs.**

---

## 2. The decision rule

Evaluated in order. First matching row wins.

| # | Condition | Decision |
|---|---|---|
| 1 | K7 FAIL | **HALT** — fix the harness; nothing else is measurable |
| 2 | **K2 FAIL (CMR > 0.20)** | **KILL — unconditional, no override** |
| 3 | K4 FAIL (transfer < 0.30), **or** no local model exists | **KILL** (or PIVOT to an explicitly runtime-only tool, if an owner exists for that) |
| 4 | **K1 FAIL and K3 FAIL** | **KILL** — no adequacy win and no detection win. The clone hypothesis is confirmed and F1/F2 stand unrefuted. |
| 5 | K1 PASS, K3 FAIL, K5 FAIL | **PIVOT** — adequacy instrument only. Delete the discovery framing, the covering-array headline, and the history/concurrency claims. |
| 6 | K1 PASS, K3 FAIL, K5 PASS | **PIVOT** — adequacy + out-of-level-space discovery. Delete the covering-array and interaction-detection framing. |
| 7 | K1 PASS, K3 PASS, K4 PASS, K5 PASS, K2 PASS, K7 PASS | **CONTINUE** — full thesis survives |
| 8 | K6 FAIL, otherwise as above | Apply the above; **additionally delete the reduction contribution** |

**Rows 5 and 6 are the realistic outcomes.** Row 7 requires K3 to pass, which the pre-registered
prediction says it will not.

---

## 3. Cost of running this decision

| Item | Estimate |
|---|---|
| Benchmark environment + 22 sealed mechanisms | the dominant engineering cost |
| Trials: 6 arms × 22 cases × replications | ≈ 10⁵ deterministic trials ≈ **2–3 days** serial at 2 s |
| L-STO transfer: 30 × (A4 discoveries) | local inference only, **0 remote calls** |
| L-REM | **0** — not needed for this decision |
| Remote LLM calls for the entire decision | **0** |

> **The decision costs zero inference budget and a few days of compute, against a build estimated
> in months. The ratio is the argument for running it before Gate 1 rather than after Gate 4.**

---

## 4. Anti-gaming rules

Recorded because the most likely person to soften these thresholds is us.

1. Thresholds and the §4.3 prediction table are **digest-recorded before the run**. Changing one
   afterwards is a recorded deviation, reported with the result.
2. **Scoring happens once.** No re-scoring after seeing results.
3. A **MARGINAL** result is not a PASS. Row 4 of the decision rule treats MARGINAL as FAIL for
   K1 and K3.
4. Adding a benchmark case after seeing results is prohibited. Cases may be *removed* only if
   demonstrably malformed, and the removal is reported.
5. "The benchmark was unfair to us" is only admissible if the unfairness was **documented before
   the run** (`OQ-20` on budget parity is the one such pre-registered caveat).
6. A KILL decision means the artifacts stay in the repository with the result attached. It does
   not mean the work is deleted or the finding is buried.

---

## 5. Pre-registered expected outcome

`HYPOTHESIS`, recorded before the benchmark exists so that the result can contradict it:

| Criterion | Predicted |
|---|---|
| K7 harness integrity | PASS |
| K1 hidden-failure discovery | **PASS** (2–3 of 3) |
| K2 catastrophic misattribution | **PASS**, but this is the least confident prediction in the table — the instrument must actively refuse, and refusal is harder to engineer than attribution |
| K3 differentiation vs A2 | **FAIL or tie** |
| K4 transfer yield | **UNKNOWN** — no basis for a prediction; this is genuinely open |
| K5 history/concurrency | **MARGINAL** — likely ties A5; B-11 may be unreachable per `OQ-19` |
| K6 reduction advantage | PASS |
| **Decision** | **Row 5 or 6 → PIVOT** |
