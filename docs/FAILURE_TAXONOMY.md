# FAILURE TAXONOMY

Status: **GATE 0.5.** Document version 0.2.0 (full rewrite; supersedes 0.1.0).

Three sections, in increasing order of importance to this review:

- **§1** — failure classes of *the agent under test*. These are what the oracles detect. They are
  the input, and in the pivoted design each is a **factor combination plus an invariant**.
- **§2** — architectural failure modes of *the experiment engine we propose to build*.
- **§3** — the **mandatory adversarial review**: 10 ways this is still a clone, 10 ways the engine
  produces misleading results, 10 ways it fails at enterprise scale, 5 reasons to kill it.

§3 is the point of the document. It is not softened.

---

## §1 Failure classes — designed for from day one

Each row maps a failure class to the **factor levels that produce it** and the **invariant that
catches it**. A class with no invariant is a class we cannot detect, and two of them are marked
exactly that.

| # | Class | Producing factors | Invariant | Deterministically detectable? |
|---|---|---|---|---|
| 1 | Timeout after dispatch | F1=`timeout_after_dispatch`, F2=`after_dispatch` | INV-8 | **Yes** |
| 2 | Ambiguous / unknown outcome | F1 + F3=`eventual` + CP3=`REPLICA`/`NONE` | INV-8 | **Yes** — the state must be `INDETERMINATE`, never promoted |
| 3 | Duplicate retry | F1 + F4=`non_idempotent_write` + F9=`loose` | INV-1 | **Yes** — ground-truth effect log counts `effect_id` occurrences |
| 4 | Stale context | F7=`stale` | INV-7, INV-3 | **Yes**, given validity intervals on context artifacts |
| 5 | Schema drift | F5=`present` | INV-3, INV-4 | **Yes** |
| 6 | 429 / retry storm | F1=`http_429`, F12=`near_limit`, F10>1 | INV-4, INV-6 | **Yes** |
| 7 | Tool output corruption | F1=`output_corruption` | INV-3, INV-7 | **Yes** |
| 8 | Missing tool output | F1=`missing_output` | INV-8 | **Yes** |
| 9 | Tool-call omission | model behaviour; F13 | INV-7 | **Yes**, against the goal postcondition |
| 10 | Tool-result ignored | model behaviour; F7=`conflicting` | INV-7 | **Partially** — detectable only when it changes the effect set |
| 11 | Unauthorized tool use | F8=`permission_narrowing`/`allowlist_change` | INV-2 | **Yes** |
| 12 | Shared-state interaction between agents | F14=`present`, F10>1 | INV-1, INV-3, INV-5 | **Yes**, given deterministic interleaving (`OQ-05`) |
| 13 | Model / provider drift | F13=`remote` across dates | — | **No.** Statistical only, via the L-REM canary. Never a per-trial verdict. |
| 14 | Evaluation leakage | — | negative controls | **Yes** — an impossible task that passes is proof of a leak |
| 15 | Grader / oracle failure | — | oracle meta-suite | **Yes**, against known-pass/known-fail/near-miss/degenerate fixtures |
| 16 | Partial infrastructure outage | F1=`partial_outage` | INV-6, INV-8 + coverage manifest | **Yes** |

**Row 13 has no invariant and cannot have one.** Model drift is a property of a distribution
across dates, not of a trial. Any system offering a per-run "the model regressed" verdict is
guessing. Row 10 is partial for the same structural reason: an ignored tool result that does not
change the effect set is indistinguishable from a result that was correctly deemed irrelevant.

---

## §2 Architectural failure modes of the experiment engine

Each: failure / cause / impact / detection / mitigation / residual.

---

### EM-01 — Determinism rots silently and invalidates everything downstream
- **Cause.** One unseeded RNG in a transitive dependency; a dict or set iteration order change; a
  wall-clock read inside the agent; thread scheduling; a hash seed.
- **Impact.** **Total.** Reduction becomes noise, fingerprints become unstable, every regression
  artifact becomes fiction. And it degrades gradually, so there is no moment where it breaks.
- **Detection.** INV-9 as a **continuously evaluated oracle**, not a harness assumption. 1000-trial
  double-run in CI.
- **Mitigation.** Seed injection at every entry point; frozen `PYTHONHASHSEED`; a clock capability
  the agent must go through; INV-9 failure blocks the build.
- **Residual.** **HIGH.** We do not control the scheduler (T-7). Antithesis does; that is their
  advantage and we should not pretend to match it.

### EM-02 — Covering-array data used to estimate interaction effects
- **Cause.** The array *looks* like a dataset. n = 1 per t-tuple is invisible in a dataframe.
- **Impact.** Reported "interactions" that are single observations. Destroys the central claim.
- **Detection.** `InterventionCapability` is held only by the Phase-3 runner, so screening data is
  **structurally ineligible** to produce a `CONTROLLED_INTERVENTION_RESULT`.
- **Mitigation.** Capability isolation (primary); schema requiring `n`, interval and design type
  (secondary).
- **Residual.** LOW — this is the cleanest thing capability isolation buys.

### EM-03 — The reducer walks to a different bug and reports it as the minimum
- **Cause.** ddmin accepts any reduction that still "fails". A different failure is still a
  failure.
- **Impact.** The minimized configuration reproduces *a* bug, not *the* bug. The regression
  artifact protects against something nobody investigated.
- **Detection.** Fingerprint-aware acceptance: a reduction is accepted only if the fingerprint is
  unchanged.
- **Mitigation.** C6.2. Plus `REDUCTION_UNSTABLE` as a terminal state.
- **Residual.** MEDIUM — depends entirely on fingerprint quality (EM-04).

### EM-04 — Fingerprint collision or splitting corrupts every count
- **Cause.** Bucketing is unsolved. Too specific ⇒ every failure unique. Too loose ⇒ distinct bugs
  merge.
- **Impact.** Failure counts, interaction counts and suite size are all wrong, in an unknown
  direction.
- **Detection.** `fingerprint_stability` across independent reductions; within-bucket variance of
  unminimized assignments.
- **Mitigation.** Report clusters with stability, exclude `UNSTABLE` clusters from counts, never
  present a fingerprint as identity.
- **Residual.** **MEDIUM-HIGH.** We have not solved it and say so.

### EM-05 — Multiple comparisons manufacture interactions
- **Cause.** C(14,2) = 91 pairs × 9 oracles = up to 819 hypotheses. At α = 0.05 uncorrected, ≈ 41
  false positives under a global null.
- **Impact.** "We found 40 interactions" means nothing.
- **Detection.** Null-factor control: a factor with no effect must not survive FDR across ≥ 20
  screens.
- **Mitigation.** BH-FDR at q = 0.05 over a **pre-declared** `hypothesis_family` in the immutable
  manifest.
- **Residual.** MEDIUM. Post-hoc family narrowing is the standard defeat and the manifest is the
  only defence.

### EM-06 — Adaptive search biases coverage, and coverage is then over-reported
- **Cause.** Phase B concentrates trials on failing neighbourhoods. The sample is no longer
  balanced.
- **Impact.** Coverage percentages overstate balance; any statistic pooled across screening and
  adaptive trials is confounded.
- **Detection.** `screening_vs_adaptive` reported separately in every coverage report.
- **Mitigation.** Interaction estimates never use adaptive data (EM-02's capability rule covers
  this too).
- **Residual.** LOW if reported separately; the risk is a future "simplification" that pools them.

### EM-07 — An oracle that never fires reads as a reliable system
- **Cause.** An oracle with a bug, an over-narrow predicate, or a permanently-`NOT_APPLICABLE`
  guard produces the same report as a system with no violations.
- **Impact.** The most dangerous silent failure in the engine. Everything looks green.
- **Detection.** Every oracle's firing rate across the whole run is reported. Zero-firing oracles
  are flagged. Oracle meta-suite with adversarial near-miss and degenerate fixtures.
- **Mitigation.** An oracle that defaults to `HOLDS` on unparseable input **fails the build**.
  `INDETERMINATE` is mandatory where evaluation is impossible.
- **Residual.** MEDIUM. Meta-fixtures only cover anticipated cases; negative controls are the
  independent backstop.

### EM-08 — Constraints silently shrink the space that coverage is measured against
- **Cause.** A constraint added to suppress a noisy combination. Coverage is then computed over
  the feasible space and reported as coverage.
- **Impact.** "100% pairwise coverage" with 60% of the space constrained away is 40% coverage.
- **Detection.** `infeasible_fraction` and `constraints_applied` are mandatory fields in the
  coverage report; every constraint carries a mandatory `rationale`.
- **Mitigation.** Constraint review is a gate activity, not a config change.
- **Residual.** MEDIUM. The incentive to add a constraint to quiet a failure is permanent.

### EM-09 — Trajectory divergence produces plausible fiction
- **Cause.** A changed factor changes a decision; recorded responses go off-policy; the replay
  does not error.
- **Impact.** Invalidates counterfactual interpretation with no error signal.
- **Detection.** `request_digest` comparison at every decision point; `divergence_rate` mandatory.
- **Mitigation.** Halt-at-divergence or stub-continuation with `post_divergence_fidelity=STUB`;
  divergent pairs excluded and counted; > 0.5 ⇒ `INDETERMINATE`.
- **Residual.** MEDIUM. Many interesting counterfactuals are simply unanswerable, and saying so
  will be under permanent pressure.

### EM-10 — Effect identity collapses and duplicates become undetectable
- **Cause.** `effect_id` conflated with `request_digest` or contaminated with `schema_version`
  (the 0.1.0 bug). A serialization change then creates a new identity for the same logical effect.
- **Impact.** INV-1 cannot fire. The engine reports no duplicates because it cannot see them.
- **Detection.** Effect-identity contract tests: serialization change ⇒ same `effect_id`; semantic
  change without a declared new step ⇒ `ABORTED / EFFECT_IDENTITY_VIOLATION`.
- **Mitigation.** Three separate fields (`ARCHITECTURE_SURFACE` §H.3); `CP1=UNSTABLE` forces
  `NON_IDEMPOTENT_WRITE` treatment.
- **Residual.** LOW now that the fields are separated; it was HIGH in 0.1.0.

### EM-11 — Hidden retries beneath the adapter
- **Cause.** An HTTP client, SDK or mesh retries without our knowledge. Our `attempt_no` reads 1;
  the server saw 3.
- **Impact.** Duplicate-detection claims are false.
- **Detection.** SC-11: adapter `attempt_no` reconciled against the synthetic environment's
  server-side request count. Any divergence is our bug.
- **Mitigation.** **F6 `hidden_retry` is a factor** — we test the case where the assumption is
  false rather than assuming it away. This is the pivot working as intended.
- **Residual.** LOW in the synthetic environment, **HIGH for any real adapter**, and the
  synthetic result does not transfer.

### EM-12 — Shared state leaks across trials and fabricates interactions
- **Cause.** Fixture not restored; warm caches; carried RNG; `effect_id` store retained across
  trials, so trial N's legitimate call deduplicates against trial N−1.
- **Impact.** Apparent interaction that is actually carry-over. Paired designs stop being paired.
- **Detection.** Per-trial canary rows with a nonce; post-restore digest assertion **before** the
  trial runs.
- **Mitigation.** Snapshot-restore by digest (Gate 1); per-trial `effect_id` namespace; seeded
  RNG derived from `(experiment_id, assignment, trial_index)`.
- **Residual.** LOW-MEDIUM, and it rises the moment trials run in parallel (T-6).

### EM-13 — Regression suite rot
- **Cause.** Artifacts pin `environment_version`. A bump invalidates a large fraction at once.
- **Impact.** A suite where 60% invalidates per release is worse than none — it trains people to
  ignore it.
- **Detection.** Invalidated-fraction-per-version-bump as a reported suite-health metric.
- **Mitigation.** Explicit invalidation conditions; scheduled re-measurement; `QUARANTINE` rather
  than silent skip.
- **Residual.** MEDIUM. `OQ-09` (invalidation granularity) is unresolved.

### EM-14 — Low-power regression artifacts reported as pass/fail
- **Cause.** A 0.05-reproducing failure run 3 times. `(1−0.05)³ = 0.86` chance of a false "fixed".
- **Impact.** Coverage silently lost; the bug returns and the suite says green.
- **Detection.** `repetitions` derived from `measured_rate` (`REGRESSION_ARTIFACT_SPEC` §3);
  `LOW_POWER` tag on any artifact with a `power_waiver`.
- **Mitigation.** `NO_LONGER_REPRODUCES` is not "fixed" — it is a candidate, and the report says so.
- **Residual.** LOW if the derivation is enforced; the pressure to run fewer repetitions is real.

### EM-15 — Sensitive data accumulates in the CAS
- **Cause.** Reduction and replay need payloads; payload retention is the liability.
- **Impact.** Largest storage cost and largest regulatory exposure, simultaneously.
- **Detection.** Unclassified-object scan; seeded-PII redaction test; retention-age monitoring.
- **Mitigation.** Allow-list telemetry; digests in the envelope, payloads in a governed CAS;
  **tested property: delete the CAS and the system still functions**, payload-dependent components
  degrading to `INDETERMINATE`.
- **Residual.** MEDIUM. The strongest control is procedural (synthetic data only) and procedural
  controls erode under demo pressure.

### EM-16 — Fault injection reaches a real system
- **Cause.** The engine's purpose is breaking things. A misconfigured adapter points at something
  real.
- **Impact.** Catastrophic and unrecoverable by any downstream control.
- **Detection.** Egress guard as a **test**: `EXACT_REPLAY` and `CONTROLLED_REPLAY` must make zero
  outbound connections, and a violation fails the run.
- **Mitigation.** Hard environment separation; `LIVE_EXECUTION` requires explicit per-invocation
  authorization and may never target a shared environment.
- **Residual.** LOW technically, MEDIUM operationally.

---

## §3 Adversarial review of the revised design

Required by the pivot brief. Not softened.

### §3.1 Ten ways the new thesis could still be a clone

1. **Antithesis / deterministic simulation testing.** They ship explore → detect → minimize →
   reproduce, with a stronger determinism story because they control the scheduler and we do not.
   An agent adapter plus a factor list is plausibly a quarter of work for them.
2. **Chaos engineering with a matrix.** Gremlin / Chaos Mesh / Litmus plus a covering array is a
   weekend. A skeptic reads this whole project as "chaos with extra steps".
3. **Combinatorial interaction testing.** ACTS, PICT, CAgen and 25 years of CIT literature own the
   method. We contribute a factor model, which is a config file in their terms.
4. **Delta debugging.** ddmin is textbook. C-Reduce and Perses are mature. Our stochastic
   adaptation is an engineering detail, not a contribution.
5. **ClusterFuzz / OSS-Fuzz.** The screen → reduce → fingerprint → regress loop **is** the fuzzing
   loop, already running at a scale we will never approach.
6. **Property-based / model-based testing.** Stateful Hypothesis and QuickCheck state machines
   already generate sequences against invariants. We are a domain application.
7. **Agent eval harnesses.** Inspect, promptfoo, DeepEval, LangSmith, Braintrust. A fault plugin
   plus a fixture snapshot closes most of the gap, and any of them could do it.
8. **Crash bucketing.** Sentry and ClusterFuzz grouping already solve fingerprinting well enough
   for their domains, and we explicitly do **not** solve it better.
9. **Sandboxed agent runtimes with record/replay.** E2B / Modal / Daytona plus replay is adjacent
   and heavily funded.
10. **Whatever a frontier lab ships internally next quarter.** This is exactly the tooling they
    need for agent robustness and do not currently sell. That is simultaneously the opportunity
    and the largest clone risk, and we have no visibility into it.

> **Honest verdict: every algorithm here is borrowed. The only defensible contributions are the
> agent-specific factor model, the agent-reliability invariant set, and the stochastic regression
> artifact format. That is three things, not a platform. `COMPETITIVE_OVERLAP` R3 asks whether a
> competent team could re-derive all three in two weeks on top of Hypothesis + ACTS + ClusterFuzz.
> If the answer at Gate 3 is yes, this is a configuration, not a product.**

### §3.2 Ten ways the experiment engine produces misleading results

1. **Covering-array data used for estimation.** n = 1 per t-tuple. The array detects; it cannot
   estimate. The data looks like a dataset and will be treated as one.
2. **Stochastic ddmin.** The "minimized" configuration may be an artifact of which repetitions
   happened to fail, not a property of the failure.
3. **Non-monotone failures.** Removing a factor changes the failure rather than removing it. A
   naive reducer reports a smaller configuration that is a different bug.
4. **Fingerprint collision or splitting.** Bug counts and interaction counts are computed over
   wrong buckets, in an unknown direction.
5. **Multiple comparisons.** 819 hypotheses, ≈ 41 expected false positives uncorrected. Without
   FDR over a pre-declared family, every screen "finds interactions".
6. **The factor model is our own hypothesis.** The engine can only find failures in a space we
   imagined. This is selection bias at the design level and **no amount of coverage accounting
   detects it** — coverage is measured against our own space.
7. **Oracle silence.** An oracle that never fires is indistinguishable from a reliable system in
   every report format that shows only violations.
8. **Synthetic fault distribution.** Interaction rates are properties of the fixture and the fault
   mix we chose. They are not estimates of anything real.
9. **Survivorship in minimization.** We only reduce failures we found. The unexplored remainder is
   absent from the narrative unless the coverage report forces it in.
10. **Seed reuse and state carry-over.** An apparent interaction that is actually leakage between
    trials. Paired designs are especially vulnerable because the pairing is what carries the leak.
11. *(bonus)* **Reproduction-rate rounding.** A 0.4-reproducing artifact executed once is a coin
    flip reported as a signal, in both directions.

### §3.3 Ten ways this fails at enterprise scale

1. **Reduction cost dominates.** O(n²)·r per failure. 50 failures ≈ 13–27 hours. Screening was 8
   minutes. The economics are inverted from what people expect.
2. **Factor space grows with v².** Pairwise N scales with the two largest level counts. Real tool
   surfaces mean large v, and the array grows faster than intuition suggests.
3. **Fixture storage.** One snapshot per trial at 10⁵ trials. Content-addressed dedup may save it;
   that is a measurement, not a plan.
4. **Trial wall-clock.** 2 s × 200k trials ≈ 4.6 days serial. Parallelism reintroduces EM-12.
5. **Determinism rots** (EM-01). Gradual, silent, and fatal to everything downstream.
6. **Regression suite invalidation.** A suite where a majority invalidates per release trains
   people to ignore it, which is worse than having no suite.
7. **Concurrency needs a scheduler.** Testing F10 deterministically requires controlled
   interleaving. Arbitrary thread scheduling requires a simulator — a far larger build than the
   factor list implies (`OQ-05`, T-7).
8. **Capability-profile maintenance.** CP1–CP8 for hundreds of real adapters is work nobody will
   fund. The honest default is `UNKNOWN` → `INDETERMINATE`, which is a much smaller product.
9. **Real agents are often not headlessly re-runnable.** The entire method assumes you can run the
   agent thousands of times from a restored state. Many production agents cannot be.
10. **Isolation of a system whose job is injecting faults.** It must never touch production, and
    that guarantee is operational, not architectural.
11. *(bonus)* **Ledger volume.** 200+ events per execution × 10⁵ trials, with the same
    high-cardinality hazards as any telemetry system.

### §3.4 Five reasons to kill the project entirely

1. **Deterministic simulation vendors already do this, better.** Antithesis sells the loop with a
   real scheduler. If they add an agent adapter, the remaining differentiation is a factor list
   and an oracle set — weeks of work for them, and they have the harder half already built.

2. **The deterministic substrate means we are mostly testing our own harness.** If the bulk of
   trials run against a stub planner, "agent reliability" is a misnomer: we are testing a retry
   state machine and a reconciliation table. The interesting failures involve *model behaviour*,
   and those are exactly the trials we cannot afford to run at volume. The method's affordability
   and its relevance are in direct opposition, and the pivot does not resolve that — it just
   labels the layers separately (`PROJECT_CHARTER` §D.5).

3. **Concurrency and shared state — the most valuable interaction class — may be out of reach.**
   Without deterministic scheduling, F10 and F14 cannot be reduced or turned into regression
   artifacts. With it, the project becomes a simulator build, which is a different and much larger
   project than this brief describes. There is no comfortable middle, and `OQ-05` may well resolve
   against us.

4. **Nobody's job is "agent failure lab".** There is no budget line, no on-call rotation, and no
   existing workflow this slots into. Tooling without an owner in the org chart becomes a research
   artifact regardless of quality. This is not a technical objection and it is the most likely
   cause of death.

5. **The factor model is our own hypothesis, so the discovery claim is circular.** A tool that can
   only find failures its authors already imagined is not a discovery tool — it is an
   elaborate checklist with good statistics. `OQ-08` exists to measure this, and **if externally
   authored seeded bugs are detected at a low rate, reason 5 alone is sufficient to stop.**

---

## §4 Residual risk summary

| Residual | Modes | Meaning |
|---|---|---|
| **HIGH** | EM-01 (determinism rot), EM-04 (fingerprinting), EM-11 for real adapters | Not solvable within this project. Must appear in every report that touches them. |
| **MEDIUM** | EM-03, EM-05, EM-07, EM-08, EM-09, EM-13, EM-15, EM-16 | Reducible with enforced tests; will regress without them. |
| **LOW** | EM-02, EM-06, EM-10, EM-12, EM-14 | Controlled, provided the control lands at its stated gate. |

**EM-01 is the one that matters most.** Without byte-reproducibility, reduction, fingerprinting
and the entire regression artifact format are meaningless — and it is the failure mode that
degrades gradually rather than breaking. That is why INV-9 is an oracle evaluated on every run
rather than a property assumed at Gate 1.
