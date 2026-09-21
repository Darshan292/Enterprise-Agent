# REGRESSION ARTIFACT SPECIFICATION

Status: **GATE 0.5.** Document version 0.1.0. Covers pivot capability **C8**.

---

## 0. The problem this format exists to solve

`FACT`: a conventional regression test assumes a **deterministic** test function and reports
pass/fail. An agent interaction failure is **stochastic** and **stateful**. Encoding it as a
pass/fail test produces one of two wrong outcomes:

- The failure reproduces at 0.4. The test is run once. It is a **coin flip reported as a signal**,
  and it will be quarantined as flaky and deleted within a month.
- The failure reproduces at 1.0 only under the exact fixture. The environment version bumps. The
  test now passes for a reason unrelated to the fix, and **nobody notices the coverage was lost**.

> **The core design commitment: a regression artifact carries a measured reproduction rate with a
> confidence interval, and is executed `r` times against a threshold — never once as pass/fail.**

A 0.4-reproducing artifact is a legitimate, valuable regression test. It is executed 20 times and
asserted against its recorded interval. Reproduction rates are data, not defects in the test.

---

## 1. Format

```yaml
artifact_version: "1.0"
artifact_id: <uuid>

# ─── PROVENANCE ────────────────────────────────────────────────────────────────
origin:
  failure_id:        <uuid>
  experiment_id:     <uuid>
  fingerprint:       <sha256>
  fingerprint_stability: 0.93          # C7.2. < 0.8 => artifact marked UNSTABLE
  discovered_at:     <iso8601>
  discovered_by:     screening | adaptive | manual

# ─── WHAT IS PINNED ────────────────────────────────────────────────────────────
manifest:
  agent_version:        <semver + digest>
  environment_version:  <semver + digest>
  adapter_version:      <semver>
  oracle_version:       <semver>          # oracle_id below is versioned separately
  planner_version:      <semver>          # for provenance only; not needed to re-run
  capability_profile:                     # CP1..CP8 in force. A profile change invalidates.
    CP1: DERIVABLE
    CP2: NON_IDEMPOTENT_WRITE
    CP3: REPLICA
    CP4: EVENTUAL
    CP5: HEURISTIC
    CP6: PARTIAL
    CP7: SIDE_EFFECT_FREE
    CP8: [ticket.write]

# ─── HOW TO RE-RUN ─────────────────────────────────────────────────────────────
execution:
  replay_mode:          CONTROLLED_REPLAY     # EXACT_REPLAY | CONTROLLED_REPLAY | LIVE_EXECUTION
  initial_state_digest: <sha256>              # fixture restore target. MUST match before running.
  fixture_ref:          <content_address>
  seed_family:          <seed_family_id>      # NOT a single seed — see §2
  seeds:                [<s1>, <s2>, ...]     # the exact seeds used to measure reproduction_rate
  factor_assignment:                          # MINIMIZED. levels only.
    F1_fault_kind:        timeout_after_dispatch
    F2_fault_timing:      after_dispatch
    F3_tool_consistency:  eventual
    F9_retry_budget:      loose
  minimality_confidence: 0.86                 # C6.2. NEVER "1-minimal".
  reduction_status:      REDUCED              # REDUCED | PARTIALLY_REDUCED | REDUCTION_UNSTABLE
  minimized_trace_ref:   <content_address>    # optional; for EXACT_REPLAY

# ─── WHAT MUST HOLD ────────────────────────────────────────────────────────────
expectation:
  invariant_id:       INV-1                   # no duplicate side effect
  oracle_id:          oracle.no_duplicate_effect
  oracle_version:     2.1.0
  expected_verdict:   VIOLATED                # this artifact asserts the BUG still reproduces
  violation_site:     adapter.reconcile:absent_under_eventual

# ─── HOW TO JUDGE THE RESULT ───────────────────────────────────────────────────
reproduction:
  measured_rate:      0.40
  n_measured:         50
  ci_low:             0.27                    # Wilson, 95%
  ci_high:            0.55
  ci_method:          wilson_95
  execution_protocol:
    repetitions:      20                      # r, derived from the interval — see §3
    assert:           rate_within_interval    # rate_within_interval | rate_at_least | exact
    threshold_low:    0.20
    threshold_high:   0.62
  divergence_rate:    0.04                    # trials dropped for trajectory divergence
  divergent_excluded: 2

# ─── WHEN THIS ARTIFACT STOPS MEANING ANYTHING ─────────────────────────────────
invalidation:
  invalidated_by:
    - environment_version_change              # any bump
    - capability_profile_change
    - oracle_version_major                    # major only; patch is compatible
    - factor_level_removed                    # a level in factor_assignment no longer exists
  revalidate_by:      <iso8601>               # scheduled re-measurement of reproduction_rate
  on_invalidation:    QUARANTINE              # QUARANTINE | REMEASURE | DELETE

# ─── WHAT THIS DOES NOT PROVE ──────────────────────────────────────────────────
limitations:
  claim_scope:        EXPERIMENT_RESULT       # NEVER PRODUCTION_BEHAVIOR
  synthetic_environment: true
  factor_model_authored_by_project: true
  external_validity:  >
    Reproduces under the recorded fixture, seed family and capability profile.
    Supports no claim about any real system, any other capability profile,
    or any environment version other than the one pinned above.
  known_gaps:
    - "Reduction budget was not exhausted; a smaller configuration may exist."
```

---

## 2. Seed family, not a seed

`DESIGN_DECISION`: pinning a single seed would make `reproduction_rate` meaningless — one seed
either reproduces or does not, and the rate would always be 0 or 1.

A **seed family** is a deterministic generator `seed_i = H(seed_family_id ‖ i)`. The artifact
records the family id and the specific seeds used for measurement, so:

- The measurement is reproducible (same seeds ⇒ same result, by INV-9).
- Re-execution can draw **fresh** seeds from the same family to test generalisation beyond the
  measured set — and a large gap between the measured rate and the fresh-seed rate is itself a
  finding (the failure was seed-specific, not factor-specific).

---

## 3. Deriving `repetitions` from the interval

The artifact must not invent `r`. It is derived from the measured rate and the tolerance you are
willing to accept.

For a target detection: if the true rate dropped to `p_fix` (the fix worked) from `p_0`
(the bug), the probability that `r` repetitions show **zero** reproductions when the bug is still
present at `p_0` is `(1 − p_0)^r`.

| `measured_rate` p₀ | r for (1−p₀)^r ≤ 0.05 | r for ≤ 0.01 |
|---|---|---|
| 0.90 | 2 | 2 |
| 0.50 | 5 | 7 |
| 0.40 | 6 | 10 |
| 0.20 | 14 | 21 |
| 0.10 | 29 | 44 |
| 0.05 | 59 | 90 |

> **A failure reproducing at 0.05 needs ~59 repetitions to be detected with 95% confidence.** If
> that is unaffordable, the honest response is to record the artifact as
> `LOW_POWER` and say so — not to run it 3 times and call it a regression test.

`DESIGN_DECISION`: `repetitions` defaults to the ≤ 0.05 column, is overridable downward only with
an explicit `power_waiver` field naming who accepted the reduced power, and the resulting
artifact is tagged `LOW_POWER` in every report.

---

## 4. Execution protocol

```
execute(artifact):
    1. Assert environment_version, agent_version, capability_profile match the manifest.
         mismatch -> INVALIDATED. Do not run. Do not report pass.
    2. Restore fixture; assert digest == initial_state_digest.
         mismatch -> ERROR. Not a pass, not a fail.
    3. Run `repetitions` trials with seeds from seed_family.
    4. Count trials where oracle_id@oracle_version returns expected_verdict.
       Exclude and count divergent trials.
    5. Compute observed rate + Wilson interval.
    6. Compare per execution_protocol.assert.
```

Four outcomes, and **three of them are not pass/fail**:

| Outcome | Meaning |
|---|---|
| `STILL_REPRODUCES` | Observed rate within the recorded interval. The bug is present. |
| `NO_LONGER_REPRODUCES` | Observed rate below `threshold_low`. **A candidate fix — not proof of one.** The rate may have moved for an unrelated reason. |
| `RATE_SHIFTED` | Observed rate **above** `threshold_high`. The bug got worse, or the environment changed. Investigate; do not report green. |
| `INVALIDATED` | A pinned version or digest no longer matches. **Not a pass.** |

`FACT`: `NO_LONGER_REPRODUCES` is not evidence the fix was correct. It is evidence that, under
this fixture and these seeds, the violation rate fell. Those are different statements and the
report uses the second one.

---

## 5. Suite-level properties

| Property | Why |
|---|---|
| Every artifact's `reproduction_rate` is **re-measured on a schedule**, not only on change. | Rates drift as the environment evolves. An artifact asserting an interval measured a year ago is asserting fiction. |
| `INVALIDATED` artifacts are **quarantined, never silently skipped**. | A skipped test reads as a passing test in every CI summary ever written. |
| Suite health reports: total, `STILL_REPRODUCES`, `NO_LONGER_REPRODUCES`, `RATE_SHIFTED`, `INVALIDATED`, `LOW_POWER`, and the **invalidated fraction per version bump**. | T-5: a suite where 60% invalidates on every environment bump is worse than no suite, and the number is the only way to see it. |
| Artifacts derived from `REDUCTION_UNSTABLE` failures are tagged and **excluded from interaction counts**. | An unstable reduction may be a different bug (C6.2). |
| Artifacts are **immutable**. A re-measurement creates a new version; it never edits the recorded rate. | The recorded rate is evidence. Editing evidence in place destroys the ability to see drift. |

---

## 6. Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-09` | Invalidation granularity. `environment_version_change` on any bump is safe but will invalidate the whole suite on a patch release. A semantic compatibility declaration per environment change is the alternative and requires discipline nobody has yet. | Gate 5 |
| `OQ-10` | Whether `NO_LONGER_REPRODUCES` should require an explicit human confirmation before an artifact is retired, given §4's caveat. Automatic retirement will silently delete coverage. | Gate 5 |
