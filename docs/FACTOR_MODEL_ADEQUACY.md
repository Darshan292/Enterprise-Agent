# FACTOR MODEL ADEQUACY

Status: **GATE 0.75 — METHOD DESIGN. NOT IMPLEMENTED.**
Document version: 0.1.0

This is the method behind the project's smallest defensible differentiation
(`RESEARCH_VALIDITY` §D). Everything else in the system is either standard or a 1.7 × efficiency
gain over free.

---

## 0. The governing constraint

> **The factor model must not define what counts as a possible explanation.**

A system that can only explain failures in terms of its own declared factors will explain every
failure in terms of its own declared factors. That is the circularity risk (#3), and it is not
fixed by adding factors — adding factors moves the boundary without removing it.

The fix is structural: **maintain a set of observables strictly larger than the factor set, and
require that a verdict of `EXPLAINED` survive a test against that larger set.**

```
  UNRECORDED  ────────────────────────────────────────────────────────┐
  (not observable anywhere)                                           │  → UNKNOWN_MODEL
                                                                      │     is the ONLY
  ┌──── AMBIENT OBSERVABLES (recorded, NOT declared as factors) ────┐  │     correct verdict
  │                                                                 │  │
  │   ┌──── DECLARED FACTORS (the experiment design varies these) ─┐│  │
  │   │  F1..F14                                                   ││  │
  │   └─────────────────────────────────────────────────────────────┘│  │
  │   trial_index · worker_id · fixture_lineage_depth · sequence     │  │
  │   signature · interleaving signature · history depth · ...       │  │
  └──────────────────────────────────────────────────────────────────┘  │
                                                                        │
  A verdict of EXPLAINED must survive tests against the AMBIENT ring. ───┘
```

---

## 1. The four verdicts

| Verdict | Meaning | Actionable next step |
|---|---|---|
| **`EXPLAINED`** | Declared factors account for the failure. Residual at or below the reproducibility floor. Confirmatory factorial passes and rules out decoys. | Compile a regression artifact. |
| **`PARTIALLY_EXPLAINED`** | Declared factors account for a significant share, but residual remains **above** the floor and is not fully attributed. | Compile the artifact with a recorded residual; investigate. |
| **`UNEXPLAINED`** | Declared factors account for nothing beyond chance, **but an ambient observable associates with the residual.** | **Promote that observable to a declared factor** and re-run. This is the model-improvement path. |
| **`UNKNOWN_MODEL`** | The failure is real and reproducible, and **neither declared factors nor any ambient observable associates with it.** | **Refuse to attribute.** Recommend additional instrumentation. Name what was tested and ruled out. |

`DESIGN_DECISION`: `UNEXPLAINED` and `UNKNOWN_MODEL` are **different and the difference is the
product**. `UNEXPLAINED` says *"your model is incomplete and here is the missing dimension."*
`UNKNOWN_MODEL` says *"your model is incomplete and your instrumentation cannot see the missing
dimension."* Collapsing them would leave only the first, which is the one an ordinary regression
diagnostic already gives.

**`UNKNOWN_MODEL` is the honest refusal, and a system that never returns it is guessing.**

---

## 2. The method — five levels, evaluated in order

Each level is deterministic. None uses an LLM.

### L0 — Determinism screen (separate harness bugs from hidden factors)

Fix `(factor_assignment, seed, fixture_digest, versions)`. Run n times.

| Result | Conclusion |
|---|---|
| Identical `trace_digest` every time | Harness is deterministic. Proceed. |
| Any mismatch | **INV-9 violation.** This is `EM-01` (determinism rot), **not** a hidden factor. Halt adequacy analysis — its premise is broken. |

`FACT`: without L0, every hidden-factor finding is confounded with harness nondeterminism, and
the two are indistinguishable from the outside. L0 is not optional.

### L1 — Establish the reproducibility floor

At a fixed assignment, vary **only the seed**. Some variance is expected and irreducible: model
sampling in L-STO, fault-timing jitter, scheduling.

The floor is measured **on the null cases** (`INTERACTION_FAILURE_BENCHMARK` B-21), never assumed:

```
floor = the 95th percentile of within-cell outcome variance observed across null cells
```

**Residual at or below the floor is not evidence of anything.** This is the most common way
adequacy instruments manufacture findings, and measuring the floor empirically is the only
defence.

### L2 — Declared-factor explanation

Fit `violation ~ declared factors (main effects + interactions to order t)` by logistic
regression over the trial set.

Report, mandatorily:
- McFadden pseudo-R² (deviance explained), with the caveat that it is not variance explained
- Residual deviance
- **Condition number / VIF over the design matrix** — see §3.6
- Convergence status and any separation

### L3 — Ambient association (the step that breaks circularity)

Test the **residual** against every observable in the pre-declared ambient set (§4). Each test is
a hypothesis in a family fixed **before** the run; BH-FDR at q = 0.05 across the whole ambient
family.

| Outcome | Contributes to verdict |
|---|---|
| No ambient observable survives FDR, residual ≤ floor | `EXPLAINED` |
| No ambient observable survives FDR, residual > floor | **`UNKNOWN_MODEL`** |
| An ambient observable survives FDR, declared factors explain little | `UNEXPLAINED` + promotion candidate |
| An ambient observable survives FDR, declared factors explain part | `PARTIALLY_EXPLAINED` |

### L4 — Confirmatory factorial (decoy rejection)

For any factor set the analysis implicates, run a designed factorial over those factors with
replication and confirm that **no proper subset reproduces the failure**. A factor that can be
dropped without changing the failure rate is a decoy and is removed from the verdict.

`EXPLAINED` requires L4 to pass. A verdict of `EXPLAINED` that skipped L4 is downgraded to
`PARTIALLY_EXPLAINED` and labelled `UNCONFIRMED`.

### 2.1 Verdict decision table

| Residual vs floor | Declared factors explain | Ambient survives FDR | L4 confirms | **Verdict** |
|---|---|---|---|---|
| ≤ floor | yes | — | yes | `EXPLAINED` |
| ≤ floor | yes | — | no | `PARTIALLY_EXPLAINED` (UNCONFIRMED) |
| > floor | substantially | no | yes | `PARTIALLY_EXPLAINED` |
| > floor | substantially | yes | yes | `PARTIALLY_EXPLAINED` + promotion candidate |
| > floor | no | yes | — | `UNEXPLAINED` + promotion candidate |
| > floor | no | **no** | — | **`UNKNOWN_MODEL`** |
| L0 fails | — | — | — | **`INVALID` — INV-9 violation, not a verdict** |

---

## 3. The seven required concerns

### 3.1 Hidden variables

Handled by L3. A hidden variable is detectable **only if it leaves a trace in an ambient
observable**. If it does not, L3 finds nothing and the verdict is `UNKNOWN_MODEL` — which is
correct, and is the behaviour `INTERACTION_FAILURE_BENCHMARK` B-19/B-20 tests.

**Detection power.** A hidden factor splitting trials evenly and shifting failure probability by
δ requires, per cell, at 95 % / 80 % power:

| δ (shift in p) | n per arm | Total trials at the cell | @2 s |
|---|---|---|---|
| 0.30 | 44 | **88** | ≈ 3 min |
| 0.20 | 95 | **190** | ≈ 6 min |
| 0.10 | 357 | **714** | ≈ 24 min |
| 0.05 | 1,425 | **2,850** | ≈ 95 min |

> **A hidden factor shifting failure probability by less than ~0.1 is not affordably detectable
> per-cell, and the instrument must say so rather than report "no hidden factor found".** Every
> `EXPLAINED` and `UNKNOWN_MODEL` verdict carries the **minimum detectable δ** at its sample size.
> Absence of evidence is reported as a bound, never as evidence of absence.

### 3.2 Unmodeled concurrency

`concurrency` is a declared factor, but the **interleaving** is not — a factor level makes an
interleaving *possible*, never determines it.

Ambient observable: **`interleaving_signature`** — a canonical hash of the observed cross-worker
operation order. If the residual associates with it while the `concurrency` level is held
constant, concurrency is a live dimension that the factor model does not capture.

`OQ-19`: whether the coarse interleaving model can even *produce* distinct signatures. If it
cannot, this concern is undetectable and the concurrency factor is decorative — which activates
`CONTINUE_OR_KILL` K3.

### 3.3 State and history

Not reachable by any level assignment (`RESEARCH_VALIDITY` §C.3, survivor S1). Handled entirely
through ambient observables:

| Observable | Detects |
|---|---|
| `sequence_signature` — canonical hash of the action shape (tool names + effect classes, ordered) | Order-dependent failures (B-08) |
| `history_depth` — operations on the most-touched resource | N-th-operation failures (B-09) |
| `prior_trial_fingerprint` | Carry-over / isolation breach (`EM-12`) |
| `fixture_lineage_depth` | Restore-path artifacts (B-16) |

> **This is the most useful structural property of the design: history-dependence is detected by
> noticing that the residual associates with a sequence signature, *without the planner having to
> enumerate sequences*.** Sequence generation improves detection *rate*; residual analysis
> provides detection *at all*, for free, from data already collected.

### 3.4 Temporal effects

Ambient observables: `trial_index`, `wall_clock_bucket`, `process_uptime`, `trial_duration_ms`.

Two tests: association of residual with each (FDR-corrected), and a **changepoint test on the
residual series ordered by trial index**. A monotone drift indicates a leak or accumulation
(B-18); a changepoint indicates a state transition mid-run.

`FACT`: a temporal drift is also the signature of an isolation failure. The instrument cannot
distinguish "a real temporal mechanism" from "our per-trial reset is broken" — both are reported
as the same finding, and the isolation canaries (`EVALUATION_STRATEGY` P.5) are what
disambiguate. This is stated because reporting a leak as a discovery would be the most
embarrassing possible failure of the instrument.

### 3.5 External dependencies

In **L-DET** there are none by construction: the egress guard (SC-4) asserts zero outbound
connections, so "external dependency" is a category that cannot apply. Any residual attributed to
one in L-DET is a guard failure.

In **L-STO / L-REM** they are real. Ambient observables: provider latency, error class, rate-limit
headers, `provider_switch` events. A residual associating with these means the finding is about
the provider, not the agent — and must be reported as such rather than as an agent property.

### 3.6 Correlated factors

Under uniform random sampling or a covering array, declared factors are approximately
independent by construction. **Under adaptive expansion they are not** (`INTERACTION_MODEL` C3,
`EM-06`), and correlated factors have unidentifiable coefficients.

Mechanism: compute VIF and the design-matrix condition number for every analysis. When two
factors are aliased beyond threshold, the instrument reports **`ALIASED(A, B)`** and **refuses to
attribute between them**. It does not pick one.

`DESIGN_DECISION`: interaction estimates are computed **only** on Phase-3 designed factorials
(`ADR` D3), where assignment is balanced by construction. Adaptive-expansion data never feeds an
attribution. `ALIASED` is the escape hatch for the cases that slip through.

### 3.7 Factor interactions

Handled by `INTERACTION_ANALYSIS.md`. The adequacy-specific point: **a failure explained only by
a high-order interaction among declared factors is still `EXPLAINED`** — order is not
inadequacy. Inadequacy is about *which dimensions exist*, not about how many must combine.

---

## 4. The ambient observable set

Declared **before** any run, versioned, and part of the immutable experiment manifest. Adding an
observable mid-analysis is post-hoc fishing and defeats the FDR control.

| Observable | Class | Detects |
|---|---|---|
| `trial_index` | temporal | drift, leaks (B-18) |
| `wall_clock_bucket` | temporal | time-of-day / external coupling |
| `trial_duration_ms` | temporal | timing-window mechanisms |
| `process_uptime_s` | temporal | accumulation |
| `worker_id` | infrastructure | worker-local config (B-17) |
| `fixture_lineage_depth` | infrastructure | restore-path artifacts (B-16) |
| `prior_trial_outcome` | isolation | carry-over |
| `prior_trial_fingerprint` | isolation | carry-over |
| `sequence_signature` | history | order dependence (B-08) |
| `history_depth` | history | N-th operation (B-09) |
| `distinct_resources_touched` | history | breadth effects |
| `interleaving_signature` | concurrency | unmodeled interleaving (B-11) |
| `retry_count_observed` | runtime | hidden retries beneath the adapter |
| `provider_latency_ms` / `provider_error_class` | external | L-STO/L-REM only |

**14 ambient observables × 9 oracles = up to 126 hypotheses.** BH-FDR at q = 0.05 over the
pre-declared family; family size and identity reported with every verdict (`EM-05`).

### 4.1 Promotion protocol

When L3 implicates an ambient observable:

1. Report `UNEXPLAINED` with the candidate named and its effect size.
2. **A human decides** whether to promote it to a declared factor. Automatic promotion would let
   the model grow to fit noise, and the FDR family would be redefined every run.
3. Promotion creates a **new `environment_version`** and invalidates prior coverage claims
   (the factor space changed).
4. Re-run and confirm the verdict becomes `EXPLAINED`.

Promotion history is itself a reported metric: **the rate at which the factor model has to grow
is the most direct available measure of how inadequate it was.** A model requiring frequent
promotion was badly specified; a model requiring none may be complete, or may simply be untested
against anything new.

---

## 5. What this method cannot do

| # | Limitation | Why structural |
|---|---|---|
| A1 | **Cannot prove adequacy.** | Absence of residual is consistent with a hidden factor that was constant across every trial. Only inadequacy is detectable. (`RESEARCH_VALIDITY` H2.) |
| A2 | Cannot detect a hidden factor shifting p by less than the minimum detectable δ at the run's sample size | §3.1. Reported as a bound on every verdict. |
| A3 | Cannot distinguish a real temporal mechanism from a broken per-trial reset | §3.4. Isolation canaries disambiguate; the instrument alone cannot. |
| A4 | Cannot detect a hidden factor perfectly correlated with a declared one | It is aliased and reports as the declared factor. **Undetectable in principle** without an intervention that breaks the correlation. |
| A5 | **The ambient set is also authored by us** | Circularity is reduced by one ring, not removed. A mechanism outside both rings yields `UNKNOWN_MODEL` — correct, and also the limit of the instrument. |

> **A5 is the honest boundary. The instrument's contribution is that it *names* the boundary and
> refuses to attribute past it. It does not transcend it.**

---

## 6. Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-21` | The reproducibility floor must be measured before any adequacy verdict is meaningful. Is the null-case set large enough to estimate a 95th percentile? | Benchmark run |
| `OQ-22` | 126 ambient hypotheses at q = 0.05 is aggressive correction; power to detect a real ambient association at n = 200 per cell may be near zero. If so the instrument returns `UNKNOWN_MODEL` on cases it should catch — **a false-negative failure mode that looks like honesty.** Power analysis required before the benchmark. | Benchmark run |
| `OQ-19` | Can the coarse concurrency model produce distinct `interleaving_signature` values at all? | Gate 2 |
