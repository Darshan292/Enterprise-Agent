# INTERACTION MODEL

Status: **GATE 0.5.** Document version 0.1.0.
Covers pivot capabilities **C1–C9**. The algorithms here are established and borrowed; the
factor model and the oracle set are the part that is ours.

---

## 0. The three-phase workflow

`DESIGN_DECISION`: screening, reduction and estimation are **different experiments with
different designs**, and conflating them is the most common way combinatorial testing produces
nonsense.

```
   PHASE 1 — SCREEN                PHASE 2 — REDUCE              PHASE 3 — ESTIMATE
   ────────────────                ────────────────              ──────────────────
   covering array over the         delta debugging over the      designed factorial over
   full factor space               failing assignment            the surviving factors
   t = 2 (then 3 if justified)     + repetition confirmation     + replication per cell
   r repetitions per row           + non-monotonicity check      + FDR correction

   DETECTS a failing               PRODUCES a minimal, stable    ESTIMATES the interaction
   configuration exists.           reproducer + fingerprint.     effect with an interval.
   CANNOT estimate effects.        CANNOT estimate effects.      CANNOT discover new factors.
```

> **`FACT` (property of covering arrays): a t-way covering array guarantees each t-way level
> combination appears *at least once*. At least once is n = 1. You cannot estimate an effect
> from n = 1.**
>
> Reporting an "interaction effect" computed from a covering array is unsupported. The array
> detects; Phase 3 estimates. This is the single most important constraint in the document and
> the easiest one to violate accidentally, because the covering-array data *looks* like a
> dataset.

---

## C1. Factor model

### C1.1 Structure

```
Factor
  id            str                 # stable, versioned
  dimension     enum                # FAULT | TOOL | CONTEXT | POLICY | RUNTIME | MODEL
  kind          enum                # FAULT | CONFIG | CONTEXT | TOOL | POLICY | MODEL
  levels        [Level]             # closed, ordered, each with a stable id
  default_level Level               # the "no defect" arm, where one exists
  applies_when  Constraint | null   # conditional factor
```

```
Constraint
  id            str
  kind          enum                # FORBIDS | REQUIRES | IMPLIES | AT_MOST_ONE
  expr          predicate over (factor_id, level_id) pairs
  rationale     str                 # REQUIRED. A constraint with no rationale is a bug hidden.
```

`DESIGN_DECISION`: constraints carry a mandatory `rationale`. An unexplained constraint is
indistinguishable from someone suppressing a combination that was failing. Every constraint is
reviewable and every constraint reduces the space we claim to have covered — so it appears in
the coverage report (C9), not just in the planner config.

### C1.2 The initial factor space

`ASSUMPTION` (owner: architect; revised at Gate 3 from measured trial cost):

| # | Factor | Dim | Levels | v |
|---|---|---|---|---|
| F1 | `fault_kind` | FAULT | none, timeout_after_dispatch, http_429, output_corruption, missing_output, partial_outage | 6 |
| F2 | `fault_timing` | FAULT | before_dispatch, after_dispatch, during_reconcile | 3 |
| F3 | `tool_consistency` (CP4) | TOOL | strong, bounded, eventual | 3 |
| F4 | `tool_effect_class` (CP2) | TOOL | pure_read, idempotent_read, idempotent_write, non_idempotent_write, unknown | 5 |
| F5 | `schema_drift` | TOOL | absent, present | 2 |
| F6 | `hidden_retry` | TOOL | absent, present | 2 |
| F7 | `context_defect` | CONTEXT | none, stale, missing, conflicting, oversized | 5 |
| F8 | `policy_change` | POLICY | none, permission_narrowing, approval_rule_change, allowlist_change | 4 |
| F9 | `retry_budget` | RUNTIME | tight, normal, loose | 3 |
| F10 | `concurrency` | RUNTIME | 1, 2, 4 | 3 |
| F11 | `latency_profile` | RUNTIME | fast, slow, bimodal | 3 |
| F12 | `quota_pressure` | RUNTIME | none, near_limit | 2 |
| F13 | `model_config` | MODEL | stub_deterministic, local, remote | 3 |
| F14 | `shared_state_peer` | MODEL | absent, present | 2 |

**`hidden_retry` (F6) is deliberately a factor**, not an assumption. Version 0.1.0 listed
"no retry beneath our gateway" as a dangerous assumption (DA-03) we could only test for. Making
it a *level* means we test the system's behaviour when the assumption is false, which is both
cheaper and more honest.

Likewise F3/F4 are the **capability profile as a factor** — the reframe that dissolves the old
blocking `OQ-01` (`PROJECT_CHARTER` §D.1).

### C1.3 Space size — the arithmetic that justifies covering arrays

```
full factorial = 6·3·3·5·2·2·5·4·3·3·3·2·3·2 = 6,998,400 configurations
```

At an `ASSUMPTION` of 2 s per deterministic trial, exhaustive enumeration is ≈ **162 days of
serial compute for a single repetition**, and one repetition of a stochastic system is
worthless. Exhaustive is not on the table; that is why covering arrays exist.

---

## C2. Experiment planner

### C2.1 Covering arrays

A *t*-way covering array `CA(N; t, k, (v₁…vₖ))` is a set of N rows such that every combination of
levels across every t-subset of factors appears in at least one row.

**Lower bound** = product of the t largest level counts.

| t | Lower bound | Practical N (IPOG-family, `ASSUMPTION` — measure at Gate 3) | vs. full factorial |
|---|---|---|---|
| 2 | 6 × 5 = **30** | ≈ 40–50 | ≈ **1.5 × 10⁵ ×** reduction |
| 3 | 6 × 5 × 5 = **150** | ≈ 250–400 | ≈ **2 × 10⁴ ×** reduction |

With `r` repetitions to absorb stochasticity:

| Design | Rows | r | Trials | @2 s serial |
|---|---|---|---|---|
| Pairwise screen | ≈ 45 | 5 | **225** | ≈ 8 min |
| 3-way screen | ≈ 350 | 5 | **1,750** | ≈ 58 min |

> **This is the number that makes the project viable.** A full interaction screen over a
> 7-million-configuration space costs minutes on the deterministic substrate and **zero LLM
> calls**. Screening is not the cost centre. Reduction is (C6).

`DESIGN_DECISION`: start at t = 2. Escalate to t = 3 only on a measured trigger — a 3-way
failure found by the reducer that pairwise screening missed. "t-wise when justified" means
justified by evidence, not by ambition.

### C2.2 Constrained generation

`FACT`: naive covering-array generation produces rows violating the constraints (e.g.
`fault_timing=during_reconcile` is meaningless when `CP5=NONE`). Two options:

| Option | Behaviour | Verdict |
|---|---|---|
| Generate then filter | Drops rows, **breaking coverage silently** | **Rejected.** Coverage claims become false with no error. |
| Constraint-aware generation (IPOG-C family) | Maintains coverage over the *feasible* space | **Adopted.** |

Either way, the coverage report states coverage over the **feasible** space and separately
reports the **infeasible** fraction with the constraints responsible. A combination excluded by
constraint is *not* a combination we covered.

### C2.3 Sequences and properties — what covering arrays cannot reach

`FACT`: a covering array covers combinations of **factor levels**. It does not cover **orderings,
timings, or state trajectories**. A failure requiring "write, then narrow permission, then
re-plan, then write again" is not a level combination.

Three complementary generators, all deterministic, all seeded:

| Generator | Covers | Borrowed from |
|---|---|---|
| **State-machine model-based testing** | Transition and transition-pair coverage over the execution state machine | Stateful model-based testing |
| **Property-based generation** | Generated initial states and action sequences under invariants | Hypothesis / QuickCheck family |
| **Metamorphic relations** | Cases with no known-correct output | Metamorphic testing literature |

### C2.4 Metamorphic relations — initial set

Relations testable **without** an oracle for "the correct answer". This matters because for most
agent tasks no such oracle exists.

| ID | Relation | Violation means |
|---|---|---|
| MR-1 | Semantically-equivalent task rephrasing ⇒ identical **effect set** (set of `effect_id`s applied) | Behaviour depends on surface form |
| MR-2 | Reordering **independent** context items ⇒ identical effect set | Order sensitivity where none is warranted |
| MR-3 | Adding **irrelevant** context ⇒ identical effect set | Context pollution changes behaviour |
| MR-4 | A fault that is injected then fully compensated ⇒ identical **final environment digest** | Compensation is incomplete |
| MR-5 | Identical run under `retry_budget = loose` vs `tight`, no fault ⇒ identical effect set | Budget leaks into behaviour absent faults |
| MR-6 | Permission **widening** never removes a previously-achievable effect | Authorization logic is non-monotone |

`DESIGN_DECISION`: MR-1 through MR-3 compare the **effect set**, not the text. Text comparison
would need an LLM judge; effect-set comparison is exact and free. This is deliberate — the
environment's ground-truth effect log is what makes the metamorphic relations deterministic.

### C2.5 Budget-aware scheduling

```
schedule(plan, budget) -> ordered trials
```
Deterministic, seeded, no LLM (`PROJECT_CHARTER` §D.10). Priority order:
1. Pairwise coverage rows not yet satisfied (coverage first).
2. Repetitions of rows whose outcomes are **discordant** across repeats — discordance is where
   stochasticity lives and where more n actually buys something.
3. Adaptive expansion (C3).
4. Remaining repetitions.

Budget is expressed in **trials** and **remote calls**, tracked separately, because their
constraints are unrelated (`QUOTA_AND_COST_MODEL`).

---

## C3. Adaptive search

`DESIGN_DECISION`: the first strategy is transparent and boring. Bayesian optimization, RL and
learned surrogates are **deferred behind a measured trigger** (`PROJECT_CHARTER` §F).

**Phase A — screen.** Run the covering array.
**Phase B — expand.** For each failing row, expand the **neighbourhood**: all configurations at
Hamming distance 1 in factor-level space, plus a balanced sample at distance 2. This is a
deterministic, auditable rule.
**Phase C — confirm.** Repeat surviving candidates to `r_confirm`.

Why not something cleverer: an adaptive search that concentrates on failing regions
**biases the coverage report**, because the sample is no longer balanced over the space. The
coverage accounting (C9) must therefore report screening coverage and adaptive coverage
**separately**, and interaction estimates (C5) must be computed only from Phase-3 designed
experiments, never from adaptively-sampled trials. A learned search would make this
bias harder to characterise, not easier — which is the real argument against starting there.

---

## C4. Deterministic oracles

An oracle is a pure function `(trial_record, environment_ground_truth) -> Verdict` with no
network access, no LLM, and a version. Detail and meta-testing in `EVALUATION_STRATEGY` §P.

| ID | Invariant | Evaluated against |
|---|---|---|
| INV-1 | No duplicate side effect: each `effect_id` appears at most once in the ground-truth effect log | Environment effect log |
| INV-2 | No unauthorized action: every applied effect has a matching authorization decision with sufficient scope | Ledger + policy version |
| INV-3 | Postcondition consistency: for every `VERIFIED_*` verdict, the declared postcondition holds against the authoritative source | Environment state + CP3/CP5 |
| INV-4 | Bounded retries: attempts per `effect_id` ≤ declared budget; total attempts ≤ execution budget | Ledger |
| INV-5 | No unsafe tool sequence: the applied sequence contains no forbidden pattern from the declared sequence policy | Ledger |
| INV-6 | No budget exhaustion: execution terminates without exhausting token, wall-clock or trial budget | Ledger + cost record |
| INV-7 | Task completion: the declared goal postcondition holds | Environment state |
| INV-8 | No unknown-outcome promotion: no `INDETERMINATE` outcome is recorded as `APPLIED` or `NOT_APPLIED` | Ledger state transitions |
| INV-9 | Determinism: identical `(fixture_digest, seed, assignment, versions)` ⇒ identical `trace_digest` | Two trial records |

`DESIGN_DECISION`: **INV-9 is an oracle, not a test-harness property.** If determinism silently
degrades — an unseeded RNG in a dependency, a dict-ordering change, a wall-clock read — every
downstream reduction and every regression artifact becomes invalid. Making it an invariant means
it is checked continuously rather than assumed. This is the most likely thing to rot (see
`FAILURE_TAXONOMY` ES-7).

Verdicts: `HOLDS` / `VIOLATED` / `INDETERMINATE` / `NOT_APPLICABLE`. `INDETERMINATE` is
first-class: an oracle that cannot evaluate (missing observability, `CP6=NONE`) must say so and
must never default to `HOLDS`.

---

## C5. Interaction analysis

### C5.1 What is estimated, and from what data

**Only Phase-3 designed experiments feed interaction estimates.** Not the covering array
(n = 1 per t-tuple), not the adaptive expansion (biased sample).

Phase-3 design: full factorial over the `m` factors surviving reduction, with `n` replications
per cell.

| m surviving | Levels example | Cells | n/cell | Trials |
|---|---|---|---|---|
| 2 | 3 × 3 | 9 | 30 | 270 |
| 3 | 3 × 3 × 2 | 18 | 20 | 360 |
| 4 | 3 × 3 × 2 × 2 | 36 | 15 | 540 |

All affordable on the deterministic substrate.

### C5.2 Estimators

| Tool | Use | Caveat that must be printed |
|---|---|---|
| **Contingency table** + Wilson interval per cell | Descriptive, first look | At n = 20, the Wilson half-width near p = 0.5 is **± 0.20**. Cell rates are imprecise; the *contrast* is the estimand, not the cell. |
| **Logistic model with interaction terms** `logit(p) ~ A + B + A:B` | The interaction estimate | Reports the coefficient, its interval, and the model's convergence status. Separation (a cell at 0 or 1) must be reported, not silently regularised away. |
| **McNemar** | **Only** where a genuine paired design exists — same fixture, same seed, one factor toggled, trajectory non-divergent | Divergent pairs are **excluded and counted**. A design where pairing is broken by divergence is not a paired design. |
| **Benjamini–Hochberg FDR** | Screening many factor pairs / many oracles | Mandatory. See C5.3. |

### C5.3 The multiple-comparison problem, quantified

With k = 14 factors, pairwise screening tests C(14, 2) = **91** factor pairs. Across 9 oracles
that is up to **819** hypotheses. At α = 0.05 uncorrected, the expected number of false positives
under a global null is ≈ **41**.

> **An engine that screens 819 hypotheses and reports "we found 40 interactions" has found
> nothing.**

`DESIGN_DECISION`: Benjamini–Hochberg FDR control at q = 0.05 across the **declared** hypothesis
family, where the family is fixed **before** the run and recorded in the experiment manifest.
Post-hoc family redefinition is the standard way this control is defeated, so the family is part
of the immutable manifest, not a reporting-time choice.

Every interaction result reports: raw p, adjusted p, q, family size, and the family's identity.

---

## C6. Failure minimization

### C6.1 The honest statement of the problem

`FACT`: ddmin assumes the test function is **deterministic** and failure is **monotone**
(removing a component never introduces a failure). **Neither holds here.** Agent runs are
stochastic; interaction failures are frequently non-monotone by definition — that is what makes
them interactions.

Applying textbook ddmin and reporting a "1-minimal" configuration would be wrong in a way that
looks right.

### C6.2 The adapted procedure

```
reduce(failing_assignment, oracle, fixture, seed_family, r_confirm):
    1. Reduce ONLY in CONTROLLED_REPLAY: frozen fixture, seeded substrate,
       model_config = stub_deterministic.              # removes one source of stochasticity
    2. Hierarchical order — reduce cheap dimensions first:
         factor levels → fault set → state mutations → action sequence → trace length
    3. For each candidate reduction, run r_confirm trials across the seed family.
         still fails at the SAME fingerprint in all r  -> accept the reduction
         passes in all r                               -> reject
         mixed, or fails with a DIFFERENT fingerprint  -> STOP this branch,
                                                          record REDUCTION_UNSTABLE
    4. Report minimality_confidence, NOT "1-minimal".
```

Three properties that matter:

- **Fingerprint-aware acceptance.** A reduction is accepted only if the failure is *the same
  failure*. Without this, the reducer happily walks from one bug to a different, smaller one and
  reports the destination as the minimum of the origin. This is the most common silent failure of
  applied delta debugging.
- **Non-monotonicity is reported, not hidden.** `REDUCTION_UNSTABLE` is a legitimate terminal
  state and appears in the artifact.
- **Reduction happens on the deterministic substrate only.** Reducing against a live model means
  `r_confirm` trials per candidate against a stochastic oracle — statistically unsound and
  unaffordable.

### C6.3 Cost — this is the cost centre

ddmin is O(n²) test executions worst case, O(n log n) typical, in `n` = active components.

```
n = 14 factors, r_confirm = 5
  typical:     14·log₂14 · 5  ≈    265 trials  ≈  9 min @ 2 s
  worst case:  14² · 5        =    980 trials  ≈ 33 min @ 2 s
```

A screening run producing **50 distinct failures** costs ≈ **13–27 hours** of reduction. Screening
was 8 minutes.

> **Budget the reducer, not the screener.** `max_reduction_trials` per failure is a required
> field on the experiment manifest, and hitting it yields a **partially reduced** artifact
> honestly labelled as such — never a claim of minimality that the budget did not buy.

---

## C7. Failure fingerprint

### C7.1 Composition

```
fingerprint = H(
    invariant_id,
    violation_site,              # oracle predicate + state-machine transition where it failed
    minimized_factor_levels,     # level IDs only
    canonical_action_shape,      # tool names + CP2 effect classes, in order. NO arguments.
    environment_version,
    agent_version,
    oracle_version
)
```

**Deliberately excluded:** timestamps, UUIDs, seeds, payload content, argument values, latencies.
Any of these makes every failure unique and destroys deduplication.

### C7.2 The honest limitation

`FACT`: failure bucketing is an unsolved problem in general, and both error directions are real:

| Error | Effect | Detection |
|---|---|---|
| **Collision** — two distinct defects share a fingerprint | Under-counts distinct bugs. Interaction stats then computed over a merged, heterogeneous bucket. | Within-bucket variance of the *unminimized* assignments. High variance ⇒ suspect merge. |
| **Splitting** — one defect yields several fingerprints because the reducer landed differently | Over-counts bugs. Inflates apparent failure diversity. | `fingerprint_stability` = fraction of independent reductions of the same original failure that yield the same fingerprint. |

`DESIGN_DECISION`: report **fingerprint clusters with a stability metric**, not fingerprints as
ground truth. `fingerprint_stability < 0.8` marks the cluster `UNSTABLE` and excludes it from
interaction counts. We do not pretend to have solved bucketing.

---

## C8. Regression compiler

Full format in `REGRESSION_ARTIFACT_SPEC.md`. The single most important property:

> A regression artifact for a stochastic failure carries a **measured reproduction rate with an
> interval**, and is executed `r` times against a threshold — never once as a pass/fail.

A failure that reproduces at 0.4 is real and worth a test. Running it once and reporting
pass/fail is a coin flip dressed as a signal.

---

## C9. Coverage accounting

`DESIGN_DECISION`: the coverage report states what was **not** explored. A coverage report that
only reports what was covered is marketing.

```
CoverageReport
  experiment_id, environment_version, agent_version, planner_version
  factor_coverage         {factor_id -> levels_exercised / levels_declared}
  pairwise_coverage       covered_feasible_pairs / total_feasible_pairs
  twise_coverage          {t -> covered / total_feasible}          # where t>2 was run
  infeasible_fraction     excluded_by_constraint / total_combinations
  constraints_applied     [constraint_id]                          # WHICH constraints shrank the space
  transition_coverage     transitions_exercised / transitions_declared
  transition_pair_coverage
  fault_coverage          {fault_kind -> trials}
  invariant_coverage      {oracle_id -> {HOLDS, VIOLATED, INDETERMINATE, NOT_APPLICABLE}}
  unexplored              explicit enumeration or a bounded characterisation
  sample_sizes            {cell -> n}
  screening_vs_adaptive   trials attributable to each, reported SEPARATELY
  divergence_excluded     trials dropped for trajectory divergence
  budget_truncated        bool + which budget
```

Three required disclosures that are easy to omit and dishonest to omit:

1. **`infeasible_fraction` + `constraints_applied`.** Constraints shrink the space we claim to
   have covered. A run at "100% pairwise coverage" with 60% of the space excluded by constraint
   has covered 40% of the space.
2. **`screening_vs_adaptive`.** Adaptive trials are a biased sample. Pooling them into a coverage
   percentage overstates balance.
3. **`invariant_coverage` including `INDETERMINATE`.** An oracle that returned `INDETERMINATE` in
   80% of trials has not validated anything, and a report showing only its `HOLDS` count reads as
   success.

> **An oracle that never fires looks identical to a reliable system.** Every oracle's firing rate
> across the whole run is reported, and a zero-firing oracle is flagged for meta-testing
> (`EVALUATION_STRATEGY` §P.2), not celebrated.

---

## Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-03` | Practical N for constrained covering-array generation over this exact factor space at t = 2 and t = 3. All planner cost arithmetic depends on it. | Gate 3 |
| `OQ-05` | Scope of the `concurrency` factor. Coarse adapter-controlled interleaving points, or true deterministic scheduling? The latter is a simulator build, not a factor. | Gate 2 |
| `OQ-06` | The declared hypothesis family for FDR. Fixed before the run, but *how* is it fixed without either over-broad correction (no power) or post-hoc narrowing (no control)? | Gate 5 |
| `OQ-07` | `r_confirm` and `fingerprint_stability` thresholds. Currently placeholders (5, 0.8). Must be derived from measured discordance rates, not chosen. | Gate 4 |
| `OQ-08` | How is the factor model itself validated? The engine can only find failures inside a space we authored. Candidate attack: seed known-interaction bugs authored by someone who did not write the factor model, and measure the detection rate. | Gate 3 |
