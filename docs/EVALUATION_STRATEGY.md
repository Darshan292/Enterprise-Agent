# EVALUATION STRATEGY

Status: **GATE 0.** Document version 0.1.0.
Answers charter questions **O** (how reliability is measured) and **P** (how evaluation is
protected from grader bugs, leakage and shared state).

---

## 0. The position this document takes

Most agent evaluation is a single number produced by a stochastic grader over a small sample with
no interval, compared against a baseline collected on a different day with a different model
behind the same alias.

`FACT`: at the sample sizes typical of agent evaluation, that number cannot distinguish the
effects people use it to claim. The arithmetic in §O.4 is not a caveat; it is the finding.

This document's job is to make sure the project does not publish numbers it cannot support.

---

## O. How reliability is measured

### O.1 There is no single reliability score

`DESIGN_DECISION`: reliability is a **vector** over an explicitly enumerated grid. A scalar is
permitted only when the vector is printed adjacent to it, with the aggregation function named and
versioned.

Stated plainly, since it will be under pressure: **"the agent is 94% reliable" is not a sentence
this system is capable of producing**, because reliability is not defined independently of the
fault environment, the perturbation set, the task distribution, the deployment, and k.

```
R( task_class, perturbation, fault_config, k, deployment_id )  →  ReliabilityCell
```

Each `ReliabilityCell` carries, mandatorily:

```
ReliabilityCell
  pass_k_hat            float      # fraction of task instances where ALL k runs passed
  n_instances           int        # number of task instances (the statistical n — NOT n*k)
  k                     int
  ci_low, ci_high       float      # Wilson score interval, level recorded
  ci_method             str
  failure_distribution  {class -> count}    # WHICH failures, not just how many
  escalation_rate       float      # fraction ending in UNKNOWN_OUTCOME/ESCALATE
  unknown_outcome_rate  float
  duplicate_effect_rate float      # ground-truth from the synthetic store
  policy_denial_rate    float
  substrate             enum       # DETERMINISTIC_STUB | LOCAL_MODEL | REMOTE_MODEL
  deployment_id         str
  grader_version        str
  seed_family           str
```

`failure_distribution` is not optional garnish. Two cells with identical `pass_k_hat` where one
fails by duplicate side effect and the other by budget exhaustion are not equally reliable, and
a scalar cannot say so. **Predictability of failure mode is itself a reliability dimension.**

### O.2 The dimensions, and why each earns its place

| Dimension | Values (initial) | What it measures | Why not omit it |
|---|---|---|---|
| **Consistency** (`k`) | k ∈ {1, 3, 5, 10} | Does the agent succeed *every* time, not *some* time. | `pass@k` rewards lucky draws. For a system taking side effects, one failure in ten is a production incident, not a 90%. **We report pass^k, not pass@k.** |
| **Robustness** (perturbation) | identity; paraphrase; reordered context; irrelevant-context injection; unit/format change | Semantic-preserving input changes should not change the outcome. | A system that only works on the exact phrasing of the fixture is measuring the fixture. |
| **Fault tolerance** (`fault_config`) | none; timeout-after-dispatch; 429; duplicate-invocation; partial/malformed response; schema mismatch; replica lag | Behaviour under the conditions the project exists to handle. | This is the axis nobody else measures and the axis this project is about. |
| **Safety** | policy-denial rate; unauthorised-attempt rate; injection-followed rate | Whether the runtime boundary held. | Orthogonal to task success. An agent can be 100% successful and 0% safe. |
| **Side-effect correctness** | duplicate rate; missing-effect rate; both from the environment's ground-truth log | The core claim (S1, S2 in the charter). | Task success can be `true` while the side-effect state is wrong. These must be measured separately or the headline claim is unverified. |
| **Predictability** | entropy / concentration of `failure_distribution` | Are failures a small, characterisable set, or a long tail? | A predictable 80% is operationally better than an unpredictable 90%. |

### O.3 `pass^k`: the estimator and its trap

Two estimators exist and they are **not** interchangeable:

1. **Direct.** For each task instance, run k times; the instance passes iff all k pass.
   `pass_k_hat` = (instances passing) / n_instances. Statistical n = **n_instances**.
2. **Plug-in.** Estimate per-run p, report p̂^k. Statistical n = n_instances × k.

`DESIGN_DECISION`: **use the direct estimator. The plug-in estimator is forbidden as a headline
number.**

Reason: the plug-in estimator assumes runs are independent given the task. `ASSUMPTION` — and one
we expect to be **false**: agent runs on the same task share the prompt, the tool surface, and
frequently the same failure attractor, so failures are positively correlated within a task. Under
positive correlation the plug-in estimator is **biased downward** relative to reality, and — far
worse — its apparent n is k times too large, so its confidence interval is roughly √k times too
narrow. It produces a tighter, more confident, wrong number. That is the most dangerous kind.

The plug-in estimate may be reported *alongside* the direct one as a diagnostic; the gap between
them is itself an estimate of within-task correlation and is worth publishing.

### O.4 The sample-size arithmetic — do this before building, not after

Wilson score interval half-width at 95%, near p = 0.5 (worst case):

| n_instances | approx. half-width | What you can actually distinguish |
|---|---|---|
| 10 | ± 0.28 | Essentially nothing. |
| 20 | ± 0.21 | 30% vs. 70%. |
| 50 | ± 0.14 | 40% vs. 70%. |
| 100 | ± 0.10 | 45% vs. 65%. |
| 400 | ± 0.049 | 47% vs. 57%. |
| 1000 | ± 0.031 | ~6pp differences. |

**Unpaired** two-proportion comparison, α = 0.05, power = 0.80, around p ≈ 0.5:

```
n_per_arm ≈ 2 · (z_{α/2} + z_β)² · p(1−p) / d²
          = 2 · (1.96 + 0.84)² · 0.25 / d²
```

| Detectable difference d | n per arm |
|---|---|
| 0.30 | ≈ 44 |
| 0.20 | ≈ 98 |
| 0.10 | ≈ 392 |
| 0.05 | ≈ 1568 |

**The consequence that matters:** a fault-injection A/B with 20 instances per arm can only detect
differences of roughly 30 percentage points or more. **Any smaller effect the system reports at
that sample size is noise being read as signal.** This must be stated on every report, and the
minimum detectable effect must be computed and printed *before* the experiment runs — printing it
afterwards invites motivated reasoning.

### O.5 Pairing — the design choice that makes the budget survivable

`DESIGN_DECISION`: all fault-injection comparisons are **paired**. The same task instance, the
same seed, the same fixture digest, run with the fault on and off. Analysed with **McNemar's
test** over discordant pairs.

Two reasons, and the second is the one that saves the project:

1. **Confounding control.** Task difficulty, prompt content, and fixture state are held exactly
   constant. The only thing that varies is the treatment we assigned. This is what makes the
   comparison an intervention rather than an observation (`ARCHITECTURE_SURFACE` §M.1).
2. **Statistical efficiency.** McNemar uses only discordant pairs. When the fault has a strong
   effect, discordance is high and the required n collapses — a fault that flips the outcome in
   most affected cases is detectable with tens of pairs rather than hundreds. Given the inference
   budget (`QUOTA_AND_COST_MODEL`), **unpaired fault comparison is simply not affordable and
   paired comparison is.** The statistically correct design is also the only one that fits.

Reported per comparison: discordant counts b and c, McNemar statistic, exact p, and the odds
ratio with interval. Plus — mandatory — the `divergence_rate` from the replay engine
(`ARCHITECTURE_SURFACE` §N.1), because a paired trial whose trajectory diverged is not a valid
pair and must be excluded and counted, not silently included.

### O.6 Substrate — never pool across it

`DESIGN_DECISION`: three execution substrates, and results are **never** aggregated across them.

| Substrate | What it is | Determinism | Use |
|---|---|---|---|
| `DETERMINISTIC_STUB` | A scripted/seeded planner with no inference. Exercises every runtime path. | Full. | The **bulk** of trials. Measures the *runtime's* reliability — state machine, idempotency, verification, budgets, policy — which is what the project actually claims. |
| `LOCAL_MODEL` | A small local model, if available. | Seedable, not guaranteed. | Middle tier for behavioural realism at no remote cost. |
| `REMOTE_MODEL` | The real provider. | None. | A **sampled subset**, for behavioural realism only. |

This is the only way the grid in §O.7 fits inside any realistic inference budget, and it is also
*more honest*: the project's claims are about the runtime, and the runtime's reliability is best
measured with the model's stochasticity removed rather than averaged over.

**The claim boundary must be stated on every result:** a `DETERMINISTIC_STUB` result says
*"the runtime handles this fault correctly"*. It says nothing about whether a real model would
produce the situation. Conflating the two is the single easiest way to overclaim here.

### O.7 The grid, and its cost

Initial grid:

```
task_classes     10
perturbations     5   (identity + 4)
fault_configs     7   (none + 6)
k                10
────────────────────
executions = 10 × 5 × 7 × 10 = 3,500 per deployment_id
```

`ASSUMPTION` (owner: architect; to be measured at Gate 1): ≈ 6 model calls per execution.

```
3,500 executions × 6 calls = 21,000 remote calls per full sweep
```

Against a 1,000 requests/day budget that is **21 days for one sweep** — and a sweep is needed per
`deployment_id`, i.e. per model change, per schema change, per policy change.

**This is not a budget problem to be optimised. It is an arithmetic impossibility, and it
invalidates any plan that assumes remote-model reliability sweeps.** Resolution:

| Substrate | Share of executions | Remote calls |
|---|---|---|
| `DETERMINISTIC_STUB` | 90% (3,150) | 0 |
| `LOCAL_MODEL` | 7% (245) | 0 |
| `REMOTE_MODEL` | 3% (105) | ≈ 630 |

≈ 630 remote calls per sweep — under one day's budget, leaving headroom for incident analysis.
See `QUOTA_AND_COST_MODEL` §4 for the full derivation.

### O.8 What reliability measurement cannot tell us

Stated so it cannot be quietly forgotten:

- Nothing about real enterprise tools. The environment is ours (`FM-12`, charter production claim).
- Nothing comparable across `deployment_id` without an explicit, recorded justification (`FM-08`).
- Nothing about tasks outside the 10 classes. There is no extrapolation claim.
- Nothing about a real production fault distribution — we chose the fault distribution, so
  measuring against it is circular. The result is *"under this fault mix"*, never *"in production"*.

---

## P. Protecting evaluation from grader bugs, leakage and shared state

Evaluation is the evidence for every claim in this project. **If evaluation is wrong, everything
downstream of it is wrong and nothing signals it.** It therefore gets the same adversarial
treatment as the runtime.

### P.1 Grader taxonomy — deterministic first, by a wide margin

| Grader type | Used for | Trust |
|---|---|---|
| **Oracle / side-effect** | Read the synthetic environment's ground-truth log. Did exactly one ticket get created, with these fields? | **Highest.** Exact, deterministic, no interpretation. |
| **State-diff** | Digest of the environment before vs. after against an expected post-state. | High. |
| **Trajectory / structural** | Was the tool sequence within the acceptable set? Were budgets respected? | High. |
| **Deterministic text** | Exact match, regex, schema validation, numeric tolerance. | High. |
| **LLM judge** | Genuinely subjective dimensions only. | **Lowest. Never sole grader on a headline metric.** |

`DESIGN_DECISION`: the synthetic environment is built **oracle-first** — every service maintains
a ground-truth side-effect log that graders read directly and that the agent cannot reach. This is
a real structural advantage over evaluating against opaque third-party APIs, and it is the reason
this project needs almost no LLM judging. It should be preserved deliberately, not diluted by
adding LLM judges to look sophisticated.

### P.2 Meta-evaluation: graders are tested like any other code

Every grader ships with a fixture suite:

| Fixture class | Purpose |
|---|---|
| Known-pass | Correct executions the grader must pass. |
| Known-fail | Incorrect executions the grader must fail. |
| **Adversarial near-miss** | Nearly-correct executions (off-by-one field, duplicate effect, right answer via a forbidden path). **The class that finds real grader bugs.** |
| **Degenerate** | Empty, truncated, crashed, timed-out executions. The grader must return an explicit verdict, never crash and never default to pass. |

Rules:
- Graders are semver-versioned; `grader_version` is recorded on every result.
- A grader change invalidates cached results for affected versions. No silent reuse.
- The fixture suite runs in CI on every change.
- A grader that crashes on degenerate input fails the build. **Defaulting to pass on
  unparseable input is the most common grader bug and it is silent.**

### P.3 Grader disagreement is quarantined, never averaged

When two graders disagree on the same artifact:

1. The result is marked `DISPUTED` and **excluded from aggregates**.
2. It is queued for human adjudication.
3. The human verdict becomes a new fixture for whichever grader was wrong.

`DESIGN_DECISION`: never average disagreeing graders. Averaging two graders that disagree
produces a number that is wrong in a *third* way and destroys the signal that something is broken.
Disagreement rate is itself a reported health metric of the evaluation plane.

### P.4 LLM-judge discipline — if one is used at all

An LLM judge may produce a headline verdict only if all of the following hold:

- It is pinned to a specific `deployment_id`, never an alias.
- It has a human-labelled calibration set of ≥ 100 items with **balanced** positive and negative cases.
- Its agreement with human labels is reported as Cohen's κ, **with the interval**, on every use.
- It exceeds a declared κ floor (proposed: κ ≥ 0.6; below that it is disqualified from verdicts and may only annotate).
- Its self-consistency across repeated runs on identical input is measured and reported.
- It is never the sole grader on any headline metric.
- Its cost is inside the inference budget — which, per §O.7, it generally is not, at eval scale.

`ASSUMPTION` (owner: architect; invalidation: a dimension emerges that oracles cannot cover):
this project needs **zero** LLM judges for its headline metrics, because its outcomes are
exact side-effect states in a store it owns. If an LLM judge appears in a headline metric, that
is a signal the metric has drifted toward subjective quality assessment — which is a stated
non-goal.

### P.5 Leakage control

| Leak path | Control | Verification |
|---|---|---|
| Grader code reachable from agent | Separate process, separate venv/import path; graders not importable from agent code. | SC-7 architectural test. |
| Expected answers in the environment | Ground-truth log written to a store the agent's identity has no read grant on. | Permission test: agent credential attempts to read the oracle log and **must** be denied. |
| Answers in prior trial state | Snapshot-restore per trial. | Canary rows (§P.6). |
| Answers in logs/errors the agent can see | Agent-visible error surface is an allow-listed, structured error taxonomy. No grader text ever reaches it. | Fixture test with a grader message containing a canary string. |
| Task fixtures containing the solution | Fixture lint: expected outputs live in a separate file the agent's loader cannot open. | CI lint. |
| **Overfitting to the eval by us, over time** | A **held-out task set** never used during development, run only at gate reviews. | Procedural. The weakest control here and the most likely to be violated, because we are the adversary (`THREAT_MODEL` §2). |

**The backstop: negative controls.** A subset of tasks is impossible to complete correctly (the
required record does not exist; the tool needed is not authorised; the data is contradictory).
**A pass on a negative control is proof of a leak, regardless of what any other test says.**
Negative controls run on every evaluation, not on request, and a pass fails the run.

### P.6 Shared-state isolation

Per trial, in order:

1. Restore the environment from a snapshot; **assert the post-restore digest equals the expected
   baseline digest**. Mismatch aborts the trial rather than running it.
2. Write a **canary row** with a per-trial nonce.
3. Derive the RNG seed deterministically: `seed = H(task_id ‖ trial_index ‖ fault_config ‖ perturbation)`.
4. Namespace the idempotency-key store per trial — otherwise trial N's legitimate call
   deduplicates against trial N−1's identical call and silently no-ops.
5. Reset caches, rate-limit buckets, connection pools.
6. **On start, scan for canaries from any previous trial.** Any hit = isolation breach = the
   entire run is invalid, not just that trial.

`FACT`: without step 1's digest assertion, paired trials are not paired, and the entire
counterfactual apparatus (`ARCHITECTURE_SURFACE` §M) rests on nothing. This is why
snapshot-restore is a **Gate 1** requirement, not a Gate 5 one.

### P.7 Balanced cases and reporting integrity

- Positive and negative cases balanced per task class; imbalance is reported when unavoidable.
- Every reported number carries: n, interval, method, `grader_version`, `deployment_id`,
  `substrate`, `seed_family`.
- **Minimum detectable effect is computed and printed before the experiment runs.**
- Results are published as the full cell grid. A scalar summary is permitted only with the grid
  adjacent and the aggregation function named and versioned.
- Excluded trials (divergent pairs, aborted restores, disputed grades) are **counted and
  reported**, never silently dropped. The exclusion rate is a health metric; a rising exclusion
  rate is usually the first visible symptom of a broken harness.

---

## Open questions blocking evaluation work

| ID | Question | Blocks |
|---|---|---|
| `OQ-06` | Multiple-comparison correction across ~30 detectors firing on one incident. Running 30 tests guarantees false positives; FDR control changes what "anomaly" means and interacts with hypothesis ranking (`FM-12`). Unresolved. | Gate 4 |
| `OQ-08` | Task class definition. 10 classes are asserted, not derived. What makes a class distinct, and how do we avoid a task set that flatters the architecture we already chose? | Gate 3 |
| `OQ-09` | Human calibration capacity. Several controls (disputed adjudication, judge calibration, held-out review) assume available human labelling time. If there is none, those controls do not exist and the documents should say so rather than describe them. | Gate 3 |
