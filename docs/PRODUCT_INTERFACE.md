# PRODUCT INTERFACE

Status: **GATE 1.0 — INTERFACE SPECIFICATION.** Version 0.1.0.

**Implementability requirement: a third-party engineer must be able to implement this interface
without seeing our architecture.** Nothing below references an internal component. Where a
behaviour is required, the required *observable* behaviour is specified, not the mechanism.

---

## 1. INPUT

### 1.1 Evaluation package schema

```yaml
evaluation_package:
  package_id: str
  declared_claim:                      # REQUIRED — what the evaluation asserts
    capability: str                    # e.g. "multi-step ticket resolution with tool use"
    metric: str                        # e.g. "task_success_rate"
    value: float
    n: int
    interval: {low: float, high: float, method: str} | null
  harness:
    framework: str
    version: str
    config_digest: str
  fixtures:
    task_set_ref: str
    expected_values_ref: str | null    # if present, MUST NOT be agent-readable (§1.4 MO-6)
    split_assignment_ref: str | null
  oracles:                             # >=1
    - oracle_id: str
      version: str
      source_ref: str
      reads: [EFFECT_LOG | AGENT_OUTPUT | ENV_STATE]   # declared, and verified (§2.1)
  trials:                              # >=1 per task; see MO-1
    - trial_id, task_id, seed, replication_index
      outcome, oracle_verdicts: {oracle_id: HOLDS|VIOLATED|INDETERMINATE|NOT_APPLICABLE}
      env_manifest_digest, trace_ref, effect_log_ref
      agent_context_ref | null
  environment:
    manifest_schema_version: str
    dimensions: {dimension_id: value_digest}           # see MO-2
  agent:
    agent_id, version, model_id | null, sampling_params | null
```

### 1.2 Required adapters

Three. An implementer supplies these; everything else is generic.

| Adapter | Contract | Why it cannot be generic |
|---|---|---|
| **`TrialSource`** | `iter_trials() -> Trial`; `rerun(task_id, seed, factor_overrides) -> Trial` | **`rerun` is what makes controlled intervention possible.** A package whose harness cannot re-run a task under specified overrides can only ever receive observational verdicts (§3.4). |
| **`EffectLog`** | `effects(trial_id) -> [Effect]` where `Effect = (effect_id, resource, operation, applied_at_seq)` | Ground truth for what the agent *did*, independent of what it *claimed*. Without it, oracle validity is unverifiable. |
| **`EnvironmentProbe`** | `manifest() -> {dimension_id: value_digest}`; `settable() -> [dimension_id]` | `settable()` determines which interventions are available, and therefore which verdicts are reachable. |

### 1.3 Optional information

Improves resolution; never required. Absence degrades verdicts toward `INCONCLUSIVE`, never toward
a false pass.

| Optional | Enables |
|---|---|
| Declared factor model | Higher-resolution attribution |
| Ambient observable set beyond the required minimum | Model-gap detection |
| Intended-trajectory model | Unintended-path detection |
| Historical packages from the same harness | Drift and regression comparison |
| Oracle source code | Mutation testing (without it, only black-box mutation of the trial record) |

### 1.4 Minimum observability (MO)

A package failing any MO receives `INSUFFICIENT_INPUT` and **is not audited**. This is a refusal
to produce a verdict, not a failing verdict.

| ID | Requirement | Why |
|---|---|---|
| **MO-1** | ≥ 2 replications per task at differing seeds | With one run per task, no variance is estimable and the headline number is a draw, not an estimate |
| **MO-2** | `env_manifest_digest` per trial | Hermeticity is unassessable without per-trial manifests |
| **MO-3** | An effect log distinct from agent-claimed output | Otherwise oracle validity cannot be checked at all |
| **MO-4** | Stable `trial_id` and `task_id` joinable across all artifacts | Cross-referencing is impossible without a join key |
| **MO-5** | Declared claim with `n` | Without n, no claim can be checked against its own power |
| **MO-6** | Expected values not readable from any agent-visible path | A leak makes every result meaningless; this is checked, not trusted |

### 1.5 Invariants asserted over the package

| ID | Invariant |
|---|---|
| **P-1** | No effect appears more than once per `effect_id` in the effect log |
| **P-2** | Every applied effect has a corresponding authorization record, where the package supplies one |
| **P-3** | Oracles declaring `reads: EFFECT_LOG` demonstrably do not vary with agent-claimed output |
| **P-4** | `env_manifest_digest` is constant within any group of trials the package treats as comparable |
| **P-5** | No canary planted in expected values or oracle messages appears in agent context |
| **P-6** | Fixed `(task_id, seed)` yields an identical `trace_ref` digest where the harness claims determinism |
| **P-7** | The declared claim's `n` is sufficient for its stated interval at the stated method |

---

## 2. PROCESS

### 2.1 Candidate detection

Deterministic checks first; statistics only on what survives.

```
 1. MO gate                  -> INSUFFICIENT_INPUT and stop, if any MO fails
 2. Deterministic invariants P-1..P-7
 3. Oracle probe             mutate the trial record; assert declared `reads` matches behaviour
 4. Hermeticity              manifest constant within comparable groups
 5. Leak scan                canaries + n-gram overlap, both directions
 6. Variance                 per-task pass rate, interval, harness-determinism check
 7. Residual association     outcome ~ declared factors, then residual ~ ambient observables
```

Steps 1–5 are **deterministic and produce findings without statistics.** They are the checks a
third party can reimplement exactly, and a conformance suite can test.

### 2.2 Controlled intervention

For every candidate `X` surviving §2.1:

```
 a. is X in EnvironmentProbe.settable() OR a factor TrialSource.rerun accepts?
       no  -> INCONCLUSIVE(NOT_SETTABLE). STOP. Do not attribute.
 b. surgicality: set X; assert no other recorded observable moves beyond its null band
       fails -> INCONCLUSIVE(NON_SURGICAL). STOP.
 c. balanced rerun at each level of X, fixture and seed family fixed, n from the power table
 d. compare:
       effect persists within the null band  -> X FALSIFIED as sufficient
       effect vanishes / drops below floor   -> X SUPPORTED within the tested space
       partial, interval excludes 0 and full -> X is a MODIFIER
       intervals too wide                    -> INCONCLUSIVE(UNDERPOWERED)
```

### 2.3 Refusal semantics

| Condition | Emitted |
|---|---|
| MO failure | `INSUFFICIENT_INPUT` |
| Candidate not settable / non-surgical / underpowered | `INCONCLUSIVE` + closed-enum reason |
| All candidates falsified, effect survives, observability inventory complete, **zero untested candidates** | `INSUFFICIENT_OBSERVABILITY` |
| Any untested candidate remains | `INCONCLUSIVE` — **never** `INSUFFICIENT_OBSERVABILITY` |

> **Refusal is the default.** Every non-refusal verdict must carry its evidence block or it is
> rejected at emission.

### 2.4 Evidence handling

Every finding carries: `evidence_kind ∈ {OBSERVATIONAL_ASSOCIATION, CONTROLLED_INTERVENTION,
FALSIFICATION, INDETERMINATE}`, source refs into the package, producer id and version, parameters
in force, and the minimum detectable effect at the n used.

**A finding derived from a controlled intervention and a finding derived from an association are
never rendered with the same weight or the same wording.**

### 2.5 Statistical handling

- Multiple testing: BH-FDR at q = 0.05 over a **family fixed before analysis**, with family size
  and digest reported. Permutation (maxT) for sub-families containing near-duplicate or aliased
  observables.
- Every rate: n, Wilson interval, method.
- **Minimum detectable effect computed and emitted for every non-finding.** An absence is reported
  as a bound, never as evidence of absence.
- Clustered data (replications within task) analysed with the cluster as the unit; the
  per-replication distribution is retained and emitted (§3.1).

---

## 3. OUTPUT

### 3.1 Machine-readable assurance artifact

```yaml
assurance_artifact:
  schema_version: "1.0"
  artifact_id, package_id, produced_at
  engine: {name, version, build_digest}

  validity_status: VALID_WITHIN_TESTED_SPACE | DEFECT_FOUND | INSUFFICIENT_INPUT
                 | INCONCLUSIVE | INSUFFICIENT_OBSERVABILITY

  claim_supported:                     # THE GATE FIELD
    status: SUPPORTED | NOT_SUPPORTED | UNDETERMINED
    earned_by: [FALSIFICATION_ATTEMPTED, DETERMINISTIC_CHECKS_PASSED, POWER_ADEQUATE]
    # `earned_by` is REQUIRED and non-empty for status=SUPPORTED.
    # An artifact asserting SUPPORTED without it is INVALID BY SCHEMA.
    blocking_recommendation: BLOCK | ALLOW | ALLOW_WITH_NOTE

  findings:
    - finding_id, defect_class, dimensions_implicated: [str]
      evidence_kind, verdict, confidence_basis
      intervention: {performed: bool, factor, levels, n_per_arm, effect, interval, mde} | null
      falsified_candidates: [{candidate, method, n, mde}]
      source_refs: [...]

  unsupported_claims:                  # claims the package makes that its evidence cannot carry
    - claim_ref, reason, required_n, actual_n, mde

  reproduction:
    package_digest, engine_build_digest, seeds_used, fixture_digests
    rerun_command_template
    replication_detail:                # section 9 of the brief -- NOT collapsed
      - unit_id, n_replications, per_replication_outcomes: [bool]
        detection_rate, interval, majority_classification
        instability: float             # fraction of replications disagreeing with the majority

  coverage:
    dimensions_checked, dimensions_not_checked
    observables_recorded, observables_absent
    interventions_available, interventions_performed, interventions_unavailable_reasons
    unexplored_space: str
    minimum_detectable_effects: {check_id: float}
```

**Three fields do the product's work.** `claim_supported.status` is what a gate blocks on.
`claim_supported.earned_by` is what makes it trustworthy — **anyone can emit `SUPPORTED`; the
schema requires the derivation.** `coverage.unexplored_space` is what stops the artifact being
read as a clean bill of health.

### 3.2 Human-readable report

Fixed section order, and it leads with what was *not* established:

1. **Verdict and what it does not mean** — one paragraph, no hedging
2. **Unsupported claims** — the package's own claims its evidence cannot carry
3. **Findings** — each with evidence kind, intervention performed or why not, and MDE
4. **Falsified candidates** — what was ruled out and how. *A system that never rules anything out
   has not demonstrated discrimination.*
5. **Coverage and unexplored space**
6. **Reproduction instructions**
7. **Replication detail** — per-unit distributions, not majorities alone

### 3.3 Validity status semantics

| Status | Meaning | Gate action |
|---|---|---|
| `VALID_WITHIN_TESTED_SPACE` | No defect found among the checks performed, at the stated MDEs | ALLOW_WITH_NOTE |
| `DEFECT_FOUND` | A defect was found and attributed | **BLOCK** |
| `INSUFFICIENT_INPUT` | MO failure; not audited | **BLOCK** — an unauditable evaluation is not a passed one |
| `INCONCLUSIVE` | Candidates remain untested | ALLOW_WITH_NOTE |
| `INSUFFICIENT_OBSERVABILITY` | All candidates falsified, effect survives | **BLOCK** |

`VALID_WITHIN_TESTED_SPACE` **never** means valid. The suffix is not decoration and may not be
dropped in any rendering.

### 3.4 Degradation when adapters are partial

| Missing | Consequence |
|---|---|
| `TrialSource.rerun` | **No interventions possible.** Every attribution caps at `OBSERVATIONAL_ASSOCIATION`; no finding can reach `SUPPORTED`. |
| `EffectLog` | MO-3 fails ⇒ `INSUFFICIENT_INPUT` |
| `EnvironmentProbe.settable()` empty | All environment candidates ⇒ `INCONCLUSIVE(NOT_SETTABLE)` |

> **A package that cannot be re-run cannot be assured — only described.** That limit belongs in
> the sales conversation, not in a footnote.

---

## 4. Conformance

An independent implementation conforms if, over the conformance suite, it: produces identical
`validity_status` and `claim_supported.status`; emits `earned_by` for every `SUPPORTED`; refuses
with the correct closed-enum reason on every non-settable and non-surgical case; and emits an MDE
for every non-finding.

**The conformance suite is the deliverable in branch (C) and a required component in branch (B).**
