# EVALUATION STRATEGY

Status: **GATE 0.5.** Document version 0.2.0 (full rewrite; supersedes 0.1.0).

In the pivoted design, "evaluation" has two distinct meanings and conflating them was a flaw in
0.1.0:

- **§O — Evaluating the agent under test.** Oracles, invariants, evidence layers, statistics.
- **§P — Evaluating the evaluator.** Oracle meta-testing, leakage, shared state, coverage honesty.

§P is the more important half. The engine's entire output is oracle verdicts; an oracle bug is
not a defect in one number, it is a defect in **all** of them, and it is silent.

---

## O. Evaluating the agent under test

### O.1 There is no reliability score

`DESIGN_DECISION`: the output is a **coverage report plus a failure set plus interaction
estimates**, each with sample sizes and intervals. There is no scalar, and no aggregation
function is provided — because providing one guarantees it becomes the only number anyone quotes.

The closest thing to a summary is:

```
Over the explored region (pairwise coverage c over the feasible space, with
infeasible fraction f excluded by constraints k₁..kₙ), invariants INV-1..INV-9 were
evaluated across N trials. V distinct failure fingerprints were found, of which S
were stable. Interaction estimates are reported for M factor pairs surviving
BH-FDR at q=0.05 over a declared family of size F. The unexplored region is U.
```

Every clause is load-bearing. Deleting any of them makes the statement misleading.

### O.2 The three evidence layers — pivot correction 5

The 90/7/3 execution split is removed. Three layers, **never pooled into one statistic**, each
with its own claim scope.

| Layer | What runs | Sizing | `claim_scope` | What it does NOT support |
|---|---|---|---|---|
| **L-DET** | Stub planner + fixture substrate. Full oracle suite. | **Exhaustive where feasible**; covering-array bounded otherwise. Thousands of trials. | `EXPERIMENT_RESULT` about the **runtime, adapters and oracles** | Nothing about model behaviour. The planner is scripted. |
| **L-LOC** | Local model, fixed declared task set. | **Fixed small set.** Declared before the run. | `EXPERIMENT_RESULT` about **one local model under named factors** | Nothing about any other model. |
| **L-REM** | Remote provider, fixed budgeted set. | **Fixed budgeted set.** Declared before the run. | `EXPERIMENT_RESULT` about **that provider/model on that date** | Nothing beyond that date and that model alias. |

`FACT`: these measure different things. L-DET answers *"does the runtime handle this factor
combination correctly?"* L-REM answers *"does this model produce situations the runtime mishandles?"*
A weighted average of the two answers neither question.

**The uncomfortable consequence, stated because it is a real limitation and not a caveat:** most
trials run in L-DET, where the planner is a stub. L-DET results are therefore mostly about *our
own runtime*, not about agents. This is `FAILURE_TAXONOMY` §3.4 kill-reason 2 and the layering
does not dissolve it — it only stops us from hiding it behind a blended number.

### O.3 Oracles

An oracle is a pure function `(trial_record, ground_truth) → Verdict`, versioned, with no network
access and no LLM.

| Type | Used for | Trust |
|---|---|---|
| **Ground-truth effect oracle** | Reads the environment's effect log. Did exactly one effect with this `effect_id` apply? | **Highest.** Exact. |
| **State-digest oracle** | Environment digest before/after vs. expected post-state | High |
| **Ledger/structural oracle** | Attempt counts, authorization records, sequence policy | High |
| **Metamorphic oracle** | MR-1..MR-6: effect-set equality across transformations | High, and needs **no** notion of a correct answer |
| **LLM judge** | — | **Not used.** See O.4. |

Verdicts: `HOLDS` / `VIOLATED` / `INDETERMINATE` / `NOT_APPLICABLE`.

`DESIGN_DECISION`: **`INDETERMINATE` is mandatory where evaluation is impossible.** An oracle that
cannot evaluate — missing observability, `CP6 = NONE`, truncated trace — must return
`INDETERMINATE`. Defaulting to `HOLDS` on unparseable input **fails the build** (Gate 2 exit
test). This is the most common oracle bug and it is silent in every report format that counts
only violations.

### O.4 Why there are no LLM judges

`DESIGN_DECISION`: this project uses **zero** LLM judges, and that is an architectural advantage
to be preserved rather than a gap to be filled.

The reason is structural: the synthetic environment maintains a **ground-truth effect log** that
`ZONE S` cannot read. Every invariant in INV-1..INV-9 is a predicate over exact state. There is
nothing subjective left for a judge to judge.

Adding an LLM judge would import: position and verbosity bias, instability across runs, drift with
the provider, a calibration burden (κ against human labels, with intervals), a versioning burden,
and a per-trial inference cost that the budget cannot absorb at experiment scale. All of that, to
grade something we can check exactly.

If a dimension ever appears that oracles cannot cover, that is a signal the metric has drifted
toward subjective output quality — which is a stated non-goal — not a signal that we need a judge.

### O.5 Statistics

| Tool | Use | Mandatory disclosure |
|---|---|---|
| **Wilson score interval** | Every proportion | n, method, level. At n = 20 near p = 0.5, half-width is ± 0.20 — printed, not implied. |
| **Contingency tables** | First look at a factor pair | Cell n. |
| **Logistic model with interaction terms** | The interaction estimate, Phase-3 data only | Coefficient, interval, convergence status, separation warnings. |
| **McNemar** | Paired designs only, where pairing genuinely holds | Discordant counts b and c; divergent pairs excluded **and counted**. |
| **Benjamini–Hochberg FDR** | Screening many hypotheses | Raw p, adjusted p, q, family size, family identity. |

**Minimum detectable effect is computed and printed *before* the experiment runs.** Printing it
afterwards invites motivated reasoning about what the run "showed".

Unpaired two-proportion sizing, α = 0.05, power = 0.80, near p = 0.5:

| Detectable difference | n per arm |
|---|---|
| 0.30 | ≈ 44 |
| 0.20 | ≈ 98 |
| 0.10 | ≈ 392 |
| 0.05 | ≈ 1568 |

On the deterministic substrate these are all affordable. On L-REM none of them are — which is
precisely why L-REM is a **canary**, reported as a distribution on a date, and never as a
comparison.

### O.6 Pairing, and when it is not real

Paired designs (same fixture digest, same seed, one factor toggled) are used where they hold,
because McNemar over discordant pairs is far more efficient than an unpaired comparison.

`FACT`: **trajectory divergence breaks pairing.** A pair where the trajectories diverged is not a
pair. Divergent pairs are excluded and counted, and `divergence_rate` appears on the result. A
design with a high divergence rate is reported as `INDETERMINATE`, not as a paired result with a
smaller n.

### O.7 What the results cannot tell us

Stated so it cannot quietly disappear from a report:

- Nothing about real systems. Synthetic environment, our fault mix, our capability profiles.
- Nothing outside the factor space. Coverage is measured **against our own space** (`OQ-08`).
- Nothing comparable across `environment_version` or `agent_version` without an explicit recorded
  justification.
- Nothing about model behaviour from L-DET.
- Nothing about any date other than the run date, from L-REM.
- **No `claim_scope = PRODUCTION_BEHAVIOR`, ever, at any assurance level.**

---

## P. Evaluating the evaluator

### P.1 Oracle meta-testing

Every oracle ships a fixture suite, run in CI on every change:

| Fixture class | Purpose |
|---|---|
| Known-holds | Correct trials the oracle must pass |
| Known-violates | Incorrect trials the oracle must catch |
| **Adversarial near-miss** | Off-by-one field, duplicate with a different `request_digest` but the same `effect_id`, correct outcome reached via a forbidden path. **This class finds the real bugs.** |
| **Degenerate** | Empty, truncated, crashed, timed-out, missing-ground-truth trials. The oracle must return an explicit verdict — **crashing or defaulting to `HOLDS` fails the build.** |
| **Mutation** | A deliberately mutated trial record where the oracle *must* flip its verdict. An oracle insensitive to a mutation it should catch is broken. |

Oracles are semver-versioned. `oracle_version` is recorded on every verdict, every failure record
and every regression artifact. An oracle change invalidates cached verdicts for affected versions;
there is no silent reuse.

### P.2 The zero-firing oracle

> **An oracle that never fires is indistinguishable from a reliable system.**

Every run reports each oracle's verdict distribution — `HOLDS` / `VIOLATED` / `INDETERMINATE` /
`NOT_APPLICABLE` — across all trials. An oracle with zero `VIOLATED` across a large run is
**flagged for meta-testing**, not celebrated. An oracle with a high `INDETERMINATE` rate has not
validated anything, and a report showing only its `HOLDS` count reads as success.

### P.3 Oracle disagreement is quarantined, never reconciled by averaging

Where two oracles cover overlapping ground and disagree on the same trial:

1. Mark the trial `DISPUTED` and **exclude it from aggregates**.
2. Queue for human adjudication.
3. The verdict becomes a new fixture for whichever oracle was wrong.

Averaging disagreeing oracles produces a number that is wrong in a third way and destroys the
signal that something is broken. Disagreement rate is a reported health metric of the evaluation
plane.

### P.4 Leakage control

| Leak path | Control | Verification |
|---|---|---|
| Agent reads the ground-truth effect log | `ZONE G` unreachable from `ZONE S`: separate DB credentials | **Permission test: the agent credential attempts the read and must be denied.** |
| Oracle code reachable from agent | Separate process, separate import path; oracles not importable from agent code | Architectural test |
| Expected answers in fixtures | Expected outputs in a separate file the agent's loader cannot open | CI lint |
| Answers in prior trial state | Snapshot-restore per trial | Canary rows (P.5) |
| Oracle messages in agent-visible errors | Agent-visible error surface is an allow-listed structured taxonomy | Fixture test with a canary string in an oracle message |
| **Us, over time** | A **held-out factor region** and a **held-out task set**, never used during development, run only at gate reviews | Procedural. The weakest control here, and the one most likely to be violated, because we are the adversary. |

**The backstop: negative controls.** A subset of tasks is impossible to complete correctly (the
record does not exist; the required tool is not authorized; the data is contradictory).

> **A pass on a negative control is proof of a leak, regardless of what any other test says.**

Negative controls run on **every** experiment, not on request, and a pass fails the run.

### P.5 Per-trial isolation

Ordered, and step 1 is not optional:

1. Restore fixture; **assert post-restore digest equals the declared baseline**. Mismatch aborts
   the trial rather than running it.
2. Write a canary row with a per-trial nonce.
3. Derive the seed deterministically: `seed = H(experiment_id ‖ factor_assignment ‖ trial_index)`.
4. Namespace the `effect_id` store per trial — otherwise trial N's legitimate call deduplicates
   against trial N−1's identical call and silently no-ops.
5. Reset caches, rate buckets, connection pools.
6. **Scan for canaries from any previous trial.** A hit means the isolation breach invalidates the
   **entire run**, not just that trial.

`FACT`: without step 1's digest assertion, paired trials are not paired, and every interaction
estimate rests on nothing. This is why snapshot-restore-by-digest is a **Gate 1** requirement.

### P.6 Reporting integrity

- Every number carries n, interval, method, `oracle_version`, `environment_version`,
  `agent_version`, layer (L-DET / L-LOC / L-REM).
- **MDE printed before the run.**
- Excluded trials — divergent pairs, failed restores, disputed verdicts, budget truncations — are
  **counted and reported**, never silently dropped. A rising exclusion rate is usually the first
  visible symptom of a broken harness.
- Coverage reports the **unexplored** space, the `infeasible_fraction`, and the constraints
  responsible (`INTERACTION_MODEL` C9).
- Screening and adaptive trials reported separately.
- No layer is pooled with another.

---

## Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-06` | How is the FDR hypothesis family fixed before the run without either over-broad correction (no power) or post-hoc narrowing (no control)? | Gate 5 |
| `OQ-08` | How is the factor model itself validated? Proposed attack: seeded bugs authored by someone who did not write the factor model, detection rate measured. **If the rate is low, `FAILURE_TAXONOMY` §3.4 reason 5 applies.** | Gate 3 |
| `OQ-11` | Is human capacity available for disputed-verdict adjudication and held-out review? If not, delete those controls from the documents rather than describing controls that do not exist. | Gate 2 |
| `OQ-12` | What is the L-LOC / L-REM task set, and who declares it? Declared "before the run" is meaningless without a named owner and a change log. | Gate 6 |
