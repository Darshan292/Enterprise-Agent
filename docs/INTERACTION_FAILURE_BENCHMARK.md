# INTERACTION FAILURE BENCHMARK

Status: **GATE 0.75 — DESIGN ONLY. NOT IMPLEMENTED.**
Document version: 0.1.0

The benchmark exists to answer one question: **does this project find and correctly characterize
failures that existing methods do not, and does it decline to guess when it cannot?**

It is designed to be capable of returning a negative answer. If it cannot fail us, it is not a
benchmark.

---

## 1. Construction rules

| Rule | Why |
|---|---|
| **R1.** Ground-truth mechanisms live in a sealed registry unreadable by the experiment plane until scoring. | Same isolation as `ZONE G`. A benchmark whose answers are readable is a leak (`EVALUATION_STRATEGY` P.4). |
| **R2.** Every case declares its ground truth as `(mechanism_class, required_conditions, decoys, observability)` — written **before** any run. | Post-hoc ground truth is not ground truth. |
| **R3.** At least one class has a mechanism that is **not recorded anywhere** in the telemetry. | The only test of `UNKNOWN_MODEL`. Without it the adequacy verdict is untested. |
| **R4.** Mechanisms are authored by someone who **did not write the factor model**, working only from the environment API. | The circularity test (risk #3). Enforced procedurally; see §6. |
| **R5.** Every arm gets an **identical trial budget**. | Otherwise every comparison is a budget comparison. |
| **R6.** Decoys are present in **≥ 30 %** of cases. | Detection without specificity is worthless, and specificity is the differentiator (`RESEARCH_VALIDITY` §D2). |
| **R7.** The benchmark includes cases where **the correct answer is "no failure exists"**. | A system that always finds something is a random generator with a report. |

---

## 2. Failure classes and cases

`mechanism_class` values: `SINGLE`, `PAIR`, `TRIPLE`, `HISTORY`, `CONCURRENCY`, `DECOY`,
`HIDDEN_RECORDED`, `ABSENT_UNRECORDED`, `NULL`.

`observability` values: `DECLARED` (mechanism is a declared factor) · `AMBIENT` (recorded in
telemetry but not declared as a factor) · `UNRECORDED` (not observable anywhere).

### 2.1 Controls — single factor

| ID | Mechanism (sealed) | Invariant | Observability | Purpose |
|---|---|---|---|---|
| **B-01** | `schema_drift = present` alone breaks argument validation | INV-3 | DECLARED | Floor control. Every arm must find this. An arm that misses it is broken. |
| **B-02** | `fault_kind = output_corruption` alone | INV-7 | DECLARED | Second floor control. |

### 2.2 Pairwise

| ID | Mechanism (sealed) | Invariant | Observability | Notes |
|---|---|---|---|---|
| **B-03** | `timeout_after_dispatch` × `tool_consistency = eventual` → reconciliation reads a stale replica, reports NOT_APPLIED, retry duplicates | INV-1 | DECLARED | **The textbook interaction.** Finding it proves nothing novel; *missing* it is disqualifying. |
| **B-04** | `policy_change = permission_narrowing` × `context_defect = stale` → agent re-plans using stale context onto a still-permitted tool and achieves the narrowed-away effect | INV-2 | DECLARED | Non-obvious. Neither factor alone violates anything. |
| **B-05** | `quota_pressure = near_limit` × `concurrency = 4` → shared bucket exhausts mid-execution | INV-6 | DECLARED | Tests whether the runtime dimension is reached. |

### 2.3 Three-way

| ID | Mechanism (sealed) | Invariant | Observability | Notes |
|---|---|---|---|---|
| **B-06** | `http_429` × `retry_budget = loose` × `concurrency = 4` → backoff resynchronises and amplifies | INV-6 | DECLARED | All three required; any two pass. |
| **B-07** | `schema_drift` × `hidden_retry` × `effect_class = non_idempotent_write` → the hidden retry replays the **old** schema and produces a second distinct effect | INV-1 | DECLARED | **Undetectable if `effect_id` were schema-derived** — validates the Gate 0.5 correction (`ADR` D6). If a run of this case passes, the identity model has regressed. |

### 2.4 History / state-dependent — outside the level-combination space

> `FACT`: these are **not reachable by any assignment of levels to factors.** Neither uniform
> random sampling nor a covering array can find them, because neither generates orderings. This
> is survivor **S1** in `RESEARCH_VALIDITY` §C.3 and the only detection-side differentiation left.

| ID | Mechanism (sealed) | Invariant | Observability | Notes |
|---|---|---|---|---|
| **B-08** | Fails only on the sequence `create → narrow_permission → update → restore_permission → update`; the second update carries an authorization decision cached before narrowing | INV-2 | DECLARED factors, **sequence not a factor** | Requires state-machine sequence generation. Arms A0–A3 should score ~0. |
| **B-09** | Fails only on the **N-th** operation on the same resource, N ≥ 3, due to accumulated version-vector growth crossing a payload limit | INV-3 | DECLARED factors, **history length not a factor** | Tests whether history *depth* is reachable. A fixed-length scenario never finds it. |
| **B-10** | Fails only when a compensation runs **after** a schema change that occurred between the original write and the compensation | INV-3 | DECLARED, **temporal ordering not a factor** | Temporal-effect case. |

### 2.5 Concurrency-dependent

| ID | Mechanism (sealed) | Invariant | Observability | Notes |
|---|---|---|---|---|
| **B-11** | Two workers interleave check-then-act on one resource; fails only on one specific interleaving. `concurrency = 4` makes it *possible*, never *determines* it | INV-1 | DECLARED factor enables; **interleaving is not a factor** | **Directly tests `OQ-05`.** If the coarse interleaving model cannot schedule this, the concurrency factor is decorative and K3 activates. |
| **B-12** | Two agents with disjoint, individually-safe permission sets jointly achieve a forbidden effect through shared state | INV-2 | DECLARED | The multi-agent interaction the charter cites. Tests `shared_state_peer`. |

### 2.6 Decoys

A decoy is a factor that **correlates with failure in the generator but is not necessary.**

| ID | Mechanism (sealed) | Decoy | Notes |
|---|---|---|---|
| **B-13** | True cause = B-03's pair. `latency_profile = bimodal` widens the timing window, raising failure probability from 0.25 to 0.75 — but the failure still occurs at `fast` | `latency_profile` | A reducer keeping `latency_profile` has produced a **false inclusion**. The core specificity metric. |
| **B-14** | True cause = B-06's triple. `quota_pressure = near_limit` co-occurs by construction under a **non-uniform** sampler but is causally inert | `quota_pressure` | **Tests whether adaptive/biased sampling manufactures confounding** — the risk `INTERACTION_MODEL` C3 flags. Under uniform sampling this decoy should vanish; under adaptive expansion it may not. |
| **B-15** | True cause is `HIDDEN_RECORDED` (see B-16), with `schema_drift` correlated at 0.8 | `schema_drift` | Decoy **plus** hidden true cause. Correct verdict: `PARTIALLY_EXPLAINED` or `UNEXPLAINED`, never `EXPLAINED(schema_drift)`. |

### 2.7 Hidden factors — recorded but not declared

The mechanism is visible in telemetry as an **ambient observable** but is not in the factor model.
Adequacy must recover it by residual analysis (`FACTOR_MODEL_ADEQUACY.md`).

| ID | Mechanism (sealed) | Ambient carrier | Correct verdict |
|---|---|---|---|
| **B-16** | Fails only when the fixture's **lineage depth** (restores since baseline) is odd — an artifact of an incremental-restore path | `fixture_lineage_depth` | `UNEXPLAINED` → then `EXPLAINED` after promotion |
| **B-17** | Fails only on **worker id ≡ 0 mod 2** — a worker-local connection-pool setting | `worker_id` | Same |
| **B-18** | Failure probability rises monotonically with **trial index** within a run — a slow resource leak | `trial_index` | Same. Also a direct test of `EM-12` isolation. |

### 2.8 Absent factors — not recorded anywhere

**The make-or-break class.** The mechanism depends on internal environment state the harness never
observes. **No analysis can attribute it.** The only correct verdict is `UNKNOWN_MODEL`.

| ID | Mechanism (sealed) | Correct verdict | Wrong answer |
|---|---|---|---|
| **B-19** | An internal counter in the ticket service flips behaviour every 7th write; never emitted, never logged, not derivable from any recorded field | `UNKNOWN_MODEL` | Any `EXPLAINED` |
| **B-20** | Same, **with a decoy** (`retry_budget`) correlated at 0.7 | `UNKNOWN_MODEL` | **`EXPLAINED(retry_budget)` — catastrophic misattribution** |

> **B-20 is the single cell that decides the project.** A system that confidently blames
> `retry_budget` for a mechanism it cannot observe is strictly worse than a system that finds
> nothing: it terminates investigation with a wrong answer and it does so with statistics
> attached. `CONTINUE_OR_KILL` makes this a hard KILL criterion.

### 2.9 Null cases

| ID | Mechanism | Correct behaviour |
|---|---|---|
| **B-21** | No injected mechanism. Environment is correct. | **No violation reported.** Any `VIOLATED` is a false positive from the harness or an oracle. |
| **B-22** | No mechanism, but a strong spurious correlation between two factors in the sampler | No violation; **no interaction reported** | Tests EM-05 (multiple comparisons manufacturing interactions). |

### 2.10 Coverage of required classes

| Required class | Cases |
|---|---|
| single-factor | B-01, B-02 |
| pairwise | B-03, B-04, B-05 |
| 3-way | B-06, B-07 |
| state/history-dependent | B-08, B-09, B-10 |
| concurrency-dependent | B-11, B-12 |
| decoy-factor | B-13, B-14, B-15 |
| hidden-factor | B-16, B-17, B-18 |
| factors absent from the declared model | **B-19, B-20** |
| null control | B-21, B-22 |

**22 cases, 9 classes.**

---

## 3. Arms — every arm gets the same trial budget

| Arm | Method | Represents |
|---|---|---|
| **A0** | Single-factor sweep + ddmin | Chaos engineering |
| **A1** | Uniform random sampling + ddmin | Fuzzing loop (ClusterFuzz-shaped) |
| **A2** | **Uniform random sampling + logistic interaction regression** | **Standard DOE/statistics. THE null hypothesis.** |
| **A3** | Pairwise covering array + ddmin | ACTS/PICT + a reducer — "existing tools composed" |
| **A4** | Pairwise CA + **fingerprint-aware stochastic ddmin** + **confirmatory factorial** + **adequacy instrument** | This project |
| **A5** | A2 + state-machine sequence generation | Stateful Hypothesis / model-based testing — the fair null for the HISTORY and CONCURRENCY classes |

`DESIGN_DECISION`: **A2 and A5 are the arms that matter.** A0/A1/A3 are context. If A4 does not
beat A2 on specificity and adequacy, and does not beat A5 on history/concurrency, the project is
"DOE with extra steps" and `RESEARCH_VALIDITY` §C.2 stands unrefuted.

**A5 is included deliberately even though it weakens our case**, because survivor S1
(history-dependence) is otherwise scored against arms that cannot generate sequences at all —
which would be a rigged comparison.

---

## 4. Metrics

### 4.1 Primary

| Metric | Definition | Where it decides things |
|---|---|---|
| **Detection rate** | Cases where ≥ 1 trial violated the ground-truth invariant | Expected near-tie A2/A3/A4 on DECLARED classes (per `RESEARCH_VALIDITY` §B.1) |
| **Localization precision** | \|reported ∩ truth\| / \|reported\| | — |
| **Localization recall** | \|reported ∩ truth\| / \|truth\| | — |
| **Decoy false-inclusion rate** | P(decoy ∈ reported set) over B-13..B-15 | **Primary differentiation metric** |
| **Adequacy verdict accuracy** | Confusion matrix over {EXPLAINED, PARTIALLY_EXPLAINED, UNEXPLAINED, UNKNOWN_MODEL} | Only A4 attempts this |
| **Catastrophic misattribution rate (CMR)** | P(verdict ∈ {EXPLAINED, PARTIALLY_EXPLAINED} ∧ named factor ∉ truth) over **B-19, B-20** | **The kill metric** |
| **Null false-positive rate** | P(violation reported) over B-21, B-22 | Harness integrity |

### 4.2 Secondary

Trials to first detection · trials to stable minimized artifact · wall-clock · reduction stability
(fraction of independent reductions yielding the same fingerprint) · L-DET → L-STO transfer rate
at n = 30 · interaction preservation (`REDUCTION_VALIDITY.md`).

### 4.3 Expected results — stated before the run

`HYPOTHESIS`. Recording predictions in advance is the only defence against reading whatever comes
back as a success.

| Class | A2 (null) | A4 (ours) | Our honest prediction |
|---|---|---|---|
| SINGLE | finds | finds | **tie** |
| PAIR | finds | finds | **tie on detection**, A4 better on precision |
| TRIPLE | finds | finds | **tie on detection**, A4 better on precision |
| DECOY | **rejects** (uniform assignment ⇒ independence) | rejects | **tie — and this is uncomfortable.** A4 wins only if the confirmatory factorial beats regression on small-n, which is unproven |
| HISTORY | **scores 0** | scores > 0 *if* sequence generation is implemented | A4 wins **only versus A2**; **A5 likely ties A4** |
| CONCURRENCY | scores 0 | **unknown — depends on `OQ-05`** | If coarse interleaving cannot schedule B-11, A4 scores 0 too |
| HIDDEN_RECORDED | no mechanism | recovers via residual clustering | **A4 wins. Clear.** |
| ABSENT_UNRECORDED | may fit a decoy | must return `UNKNOWN_MODEL` | **A4 wins if and only if it refuses to attribute** |
| NULL | low FP | low FP | tie |

> **Read the prediction table honestly: A4 ties the null on five of nine classes. It wins clearly
> on two — HIDDEN_RECORDED and ABSENT_UNRECORDED — and both are adequacy, not detection. That is
> the entire case for the project, and it is exactly `RESEARCH_VALIDITY` §D.**

---

## 5. Scoring protocol

1. Seal the ground-truth registry. Record its digest.
2. Declare arm budgets, metrics and the prediction table (§4.3). Record the digest.
3. Run all arms at equal budget, isolated per `EVALUATION_STRATEGY` P.5.
4. Score against the registry. **Scoring happens once.** No re-scoring after seeing results.
5. Any deviation from the pre-declared prediction table is reported as a deviation, not
   retrofitted into a narrative.
6. Transfer test (n = 30) for every A4 discovery on DECLARED classes.

---

## 6. The independence requirement — and the honest admission that it is imperfect

**R4 requires mechanisms authored by someone who did not write the factor model.** Enforcement:

- The mechanism author works only from the environment's public API and the invariant list.
- They never read `INTERACTION_MODEL.md` §C1.2 (the factor table).
- They commit mechanisms to the sealed registry before the factor model is shown to them.

`FACT`: this reduces circularity. **It does not eliminate it.** The author is still on this
project, still shares its assumptions about what agent failures look like, and still chose an
environment we built.

> **The benchmark is a test of whether the factor model covers mechanisms *one of us* invented
> without looking at it. That is strictly weaker than a test against mechanisms from the world,
> and every result from it inherits that limitation.** (`RESEARCH_VALIDITY` H6.)
>
> The only stronger test available is running the instrument against failures from a real agent
> incident log that nobody here authored. That requires data this project does not have. It is
> recorded as `OQ-18` and it is the strongest single upgrade to the project's evidence base.

---

## 7. Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-18` | Can we obtain real agent failure traces authored by nobody here? Without them, H6 is permanent. | Nothing — but it bounds every claim |
| `OQ-19` | Is B-11's interleaving schedulable under the coarse concurrency model? If not, the CONCURRENCY class is unscorable and K3 activates. | Benchmark run |
| `OQ-20` | Budget parity: A2 needs no fixture restores between configurations while A4 does. Is "equal trials" or "equal wall-clock" the fair comparison? Equal trials flatters A4. | Benchmark run |
