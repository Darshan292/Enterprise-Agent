# ENVIRONMENT CONFOUNDER MODEL

Status: **GATE 0.9.** Version 0.1.0.

**The analogue: hermetic build and test systems (Bazel, Nix) exist precisely to eliminate
environmental confounders, and they are mature.** The difference here is that an agent evaluation
**cannot be fully hermetic** — it needs a mutable world with services, state and timing. So
hermeticity becomes something we **measure per dimension** rather than something we guarantee.

---

## 1. The candidate dimensions — and the instruction not to assume relevance

The brief lists these. **Most are probably irrelevant, and asserting otherwise without measurement
is the error this document exists to avoid.**

| # | Dimension | Prior on relevance | Cheap to record? |
|---|---|---|---|
| E-01 | OS / kernel version | Low in-container, **high** across hosts | Yes |
| E-02 | Shell | Very low | Yes |
| E-03 | Filesystem type / **directory-iteration order** | **Medium — ordering is a classic silent confounder** | Yes |
| E-04 | Working directory | Low, unless paths leak into behaviour | Yes |
| E-05 | Dependency versions | **High** | Yes (lockfile digest) |
| E-06 | Tool / service versions | **High** | Yes |
| E-07 | Network availability | **High** in L-STO/L-REM, **nil** in L-DET (egress guard) | Yes |
| E-08 | Service versions (synthetic env) | **High** | Yes |
| E-09 | Timing / clock / latency profile | **High** | Yes |
| E-10 | Resource pressure (CPU, memory, disk) | **Medium** — changes timing | Yes, sampled |
| E-11 | Worker assignment | **Medium** — proxies everything worker-local | Yes |
| E-12 | Previous-run state | **High** — the leakage channel | Yes (residue digest) |
| E-13 | Fixture lineage | **High** — restore-path artifacts | Yes |
| E-14 | Context state | **High** | Yes |
| E-15 | Locale / timezone / encoding | Low, occasionally decisive | Yes |
| E-16 | Random-source availability | Low | Yes |

`DESIGN_DECISION`: **record all sixteen, declare none relevant, and let the screening step (§3)
decide.** Recording is cheap; assuming is not. A dimension promoted to relevant without
measurement is a hypothesis wearing a fact's clothes.

---

## 2. The environment manifest

Per trial:
```
env_manifest_digest = H(sorted(dimension_id : value_digest for all 16))
```
Plus the per-dimension digests, so a manifest mismatch can be localised to the dimension
responsible rather than merely detected.

---

## 3. Four-step detection method

### Step 1 — Hermeticity assertion (deterministic)

> **Within a single experiment cell, `env_manifest_digest` must be constant. Any variation is a
> defect, not a nuisance.**

This is the cheapest and strongest check in the document. It fires *before* any statistics, and a
variation localises immediately to the differing dimension.

Cell-level verdict: `HERMETIC` / `NON_HERMETIC(dimensions)`. A non-hermetic cell's results are
**withheld**, not adjusted.

### Step 2 — Variance screening (which dimensions actually vary?)

Across the whole run, measure how many distinct values each dimension took.

| Observed | Meaning |
|---|---|
| **1 distinct value** | Cannot confound this run. **But cannot be ruled out as a constant hidden factor either** — and constants are the most dangerous confounders, because no analysis of this run can see them. Recorded as `CONSTANT_UNTESTED`. |
| **2+ distinct values** | A live confounder candidate ⇒ Step 3. |

`FACT`: this step is what stops the ambient family exploding. Only dimensions that actually varied
enter the statistical tests, which shrinks the multiple-testing family from a nominal 16 × 9 = 144
to the handful that moved — with a material power gain (`STATISTICAL_VALIDITY` §5). **The family
is fixed by the screening result, and the screening result is recorded before any association is
computed**, so this is not post-hoc family narrowing.

### Step 3 — Association (statistical)

Residual association against each varying dimension, corrected per `STATISTICAL_VALIDITY.md`.
Result is an `OBSERVATIONAL_ASSOCIATION` and a **candidate only**.

### Step 4 — Deliberate variation (controlled intervention)

The only step that licenses a claim that the dimension *affects* results.

| Dimension | Surgical intervention available? |
|---|---|
| Worker assignment | **Yes** — assign deliberately |
| Resource pressure | **Yes** — apply a load generator |
| Tool / service version | **Yes** — pin alternately |
| Timing / latency profile | **Yes** — already a declared factor |
| Directory-iteration order | **Yes** — force reverse order |
| Previous-run state | **Yes** — deliberately leave residue |
| **Fixture lineage depth** | **No** — a consequence of the restore path, not an input ⇒ `INCONCLUSIVE` (this is benchmark case B-20b) |
| OS / kernel | Partially — requires a second host; rarely surgical |

Outcome per `IDENTIFIABILITY_AND_FALSIFICATION.md` §2, including the surgicality check: after
`do(dimension := value)`, **no other recorded dimension may move outside its null band**. Changing
a tool version usually moves several, which is precisely when the honest answer is `INCONCLUSIVE`.

---

## 4. What this cannot establish

| # | Limitation |
|---|---|
| C1 | **A dimension constant across the entire run cannot be detected.** `CONSTANT_UNTESTED` names the blind spot; it does not remove it. Only running on a genuinely different environment does. |
| C2 | The sixteen dimensions are **our** list. Dimension seventeen is invisible, and this is the environment-level restatement of factor-model incompleteness. |
| C3 | Most interventions are **not surgical**. A tool-version change moves dependency digests, timing and sometimes behaviour at once. `INCONCLUSIVE` will be a frequent and correct output. |
| C4 | Hermeticity assertion catches *variation*, not *wrongness*. A uniformly wrong environment is perfectly hermetic. |
| C5 | Recording sixteen dimensions per trial is cheap; **interpreting** them is a multiple-testing problem that eats power (`STATISTICAL_VALIDITY`). Step 2 is the mitigation and it is not free. |
