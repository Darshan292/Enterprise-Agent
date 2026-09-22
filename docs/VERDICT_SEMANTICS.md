# VERDICT SEMANTICS

Status: **GATE 0.9.** Version 0.1.0. Replaces the Gate 0.75 vocabulary.

---

## 0. Why the old labels were wrong

| Old | Problem |
|---|---|
| `EXPLAINED` | Implies the model accounts for the failure **in general**. It can only ever mean *"within the factor levels and the observables we tested, at the power we had."* The word smuggles completeness. |
| `UNEXPLAINED` | Ambiguous between *"we found a gap"* and *"we found nothing"*. Those are different findings. |
| `UNKNOWN_MODEL` | Collapsed *"we ran the discriminating experiments and they came back negative"* with *"we did not run them"*. That collapse is what made B-20 unearnable (`IDENTIFIABILITY_AND_FALSIFICATION` §0). |
| *(missing)* | No verdict existed for **the evaluation itself being broken** — which, under the Gate 0.9 thesis, is the primary output. |

---

## 1. The six verdicts

| Verdict | One-line meaning |
|---|---|
| **`SUPPORTED_WITHIN_TESTED_SPACE`** | A named explanation survived controlled intervention, within the tested factor levels, observables and power. |
| **`PARTIALLY_SUPPORTED`** | A named explanation survived intervention but does not account for the whole effect; a residual remains above the floor. |
| **`MODEL_GAP_DETECTED`** | An observable **outside the declared factor model** associates with the residual and is a promotion candidate. The model of what can vary is incomplete in a **named** way. |
| **`INSUFFICIENT_OBSERVABILITY`** | Every observable candidate was **falsified by intervention**, the effect survives, and the observability inventory is complete. The instrumentation cannot see the mechanism. |
| **`INCONCLUSIVE`** | The discriminating experiment was not performed — unavailable, non-surgical, or underpowered. **An honest stop, not a finding.** |
| **`INVALID_EVALUATION`** | An integrity defect was found that makes the evaluation's result untrustworthy regardless of what the agent did. **The primary product output.** |

---

## 2. Exact evidence requirements

Every requirement is a field on the verdict record. A verdict missing any required field is
rejected at write time, not at review.

### 2.1 `SUPPORTED_WITHIN_TESTED_SPACE`

| Required | Note |
|---|---|
| Named explanation as a factor:level set | — |
| `OBSERVATIONAL_ASSOCIATION` above the reproducibility floor | the candidate step |
| **`CONTROLLED_INTERVENTION` in which the effect vanished or dropped below the floor** | mandatory; without it the verdict is `INCONCLUSIVE` |
| Surgicality check passed, with the observables checked listed | `IDENTIFIABILITY_AND_FALSIFICATION` Rule 2 |
| n per arm, effect size, CI, **minimum detectable difference** | — |
| Confirmatory subset sweep: no proper subset reproduces | decoy rejection |
| `tested_space` block: factor levels varied, observables recorded, fixture digest, seed family, layer | **what "within tested space" refers to** |
| Layer tag (L-DET / L-STO / L-REM) | never mixed |

**Explicitly does not mean:** that the explanation is complete, that no other explanation exists,
or that it holds outside the fixture.

### 2.2 `PARTIALLY_SUPPORTED`
All of 2.1, **plus** a residual above the floor that the supported explanation does not account
for, quantified with its interval, **plus** the outcome of residual analysis on that remainder
(which may itself be `MODEL_GAP_DETECTED`, `INSUFFICIENT_OBSERVABILITY` or `INCONCLUSIVE`, and is
recorded as a nested verdict).

### 2.3 `MODEL_GAP_DETECTED`

| Required |
|---|
| Named **ambient** observable (recorded, not a declared factor) |
| Association surviving multiple-testing correction: raw p, adjusted p, q, **family size and family digest** |
| Declared factors account for less than the residual floor |
| **Promotion recommendation**, with the note that promotion creates a new `environment_version` and invalidates prior coverage |
| Minimum detectable effect at the run's n |

**Does not require intervention.** This verdict is explicitly an `OBSERVATIONAL_ASSOCIATION` and
is labelled as such. It says *"here is a dimension you did not declare"*, not *"this is the
cause"*. Promotion followed by a designed run is what upgrades it.

### 2.4 `INSUFFICIENT_OBSERVABILITY` — the strictest verdict

| Required | Why |
|---|---|
| Effect reproduces above the floor, with n and CI | there is something to explain |
| **A falsification record for EVERY observable candidate**, each with: surgicality check, n per arm, effect, CI, minimum detectable difference | Rule 1: this verdict is only reachable by elimination-by-intervention |
| **Observability inventory with its digest** | the verdict is relative to what we record; `IDENTIFIABILITY_AND_FALSIFICATION` L5 |
| Explicit statement that no candidate survived and none was left untested | — |
| `INDETERMINATE` count = **0** | if any candidate went untested, the verdict is `INCONCLUSIVE` |

> **This is the verdict the system must be reluctant to reach.** It is the one that Gate 0.75
> demanded from observational data alone. It is now gated behind a complete falsification record,
> and a single untested candidate downgrades it.

### 2.5 `INCONCLUSIVE`

| Required |
|---|
| What was attempted |
| **Why the discriminating experiment was not performed**, from a closed enum: `NOT_SETTABLE`, `NON_SURGICAL`, `UNDERPOWERED`, `BUDGET_EXHAUSTED`, `UNSAFE` |
| The untested candidates, named |
| What would resolve it |

`INCONCLUSIVE` is a **correct and frequent** output. A system that rarely returns it is either
running interventions it cannot justify or attributing without them.

### 2.6 `INVALID_EVALUATION`

| Required |
|---|
| Dimension violated (one of the five in `EVALUATION_INTEGRITY_MODEL.md`) |
| The specific defect, from the catalogue in that document |
| **Deterministic evidence where a deterministic test exists** — e.g. an oracle mutation escaped; a canary from trial N−1 was visible in trial N; a contamination canary appeared in agent context |
| Impact statement: which reported results are affected and in which direction, where determinable |
| Whether the defect is **repairable** and how |

`DESIGN_DECISION`: `INVALID_EVALUATION` **dominates**. If it fires, no other verdict about the
agent is reported for the affected results — they are withheld, not caveated. A caveated number
gets quoted without its caveat.

---

## 3. Verdict precedence

```
INVALID_EVALUATION
    ▼  (if clear)
INSUFFICIENT_OBSERVABILITY  ·  MODEL_GAP_DETECTED
    ▼
SUPPORTED_WITHIN_TESTED_SPACE  ·  PARTIALLY_SUPPORTED
    ▼  (if any required evidence is missing at any level)
INCONCLUSIVE
```

`INCONCLUSIVE` is the **default**. Every other verdict must be earned by supplying its evidence
block. This inverts the usual failure mode, in which the confident verdict is the default and the
caveats are optional.

---

## 4. Banned phrasing

Not permitted in any output: *root cause* · *caused by* · *due to* · *because of* · *explained* ·
*proves* · *validates the evaluation* · *the evaluation is correct* · *randomized* ·
*natural experiment* · *complete*.

Primary enforcement is structural: no component holds a capability to construct a verdict at a
strength its evidence does not support. Lint over output templates is secondary.
