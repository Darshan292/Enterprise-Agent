# STATISTICAL CORRECTIONS

Status: **GATE 1.0.** Version 0.1.0.
Corrects an error in `METRICS.md` / `KILL_CONTINUE_0_95.md` and specifies replication handling.

---

## 1. The FAR denominator error

### 1.1 What was wrong

`METRICS.md` stated *"n = 112 per arm detects a 0.15 difference around 0.20"* and
`KILL_CONTINUE_0_95.md` C-2 thresholded on a 0.10 difference at that n.

**112 was the n *required* to achieve an MDE of 0.15. It was never the n *available*.** A power
calculation was transcribed into the experimental design as though it were the design. The
threshold was then set against a precision the experiment does not have.

`FACT`: this is the same class of error as the Gate 0.9 pooled-FDR estimator and the Gate 0.75
unearnable verdict — **the fourth self-inflicted validity defect in this project's history, and
the third caught by re-deriving a number rather than by any process.**

### 1.2 The actual denominator

```
registry: 24 defective packages x 5 replications = 120 defective observations per arm
```

Replications are **clustered within package**. Design effect `DE = 1 + (m−1)ρ`, m = 5:

| ICC ρ | DE | Effective n |
|---|---|---|
| 0.5 | 3.0 | 40 |
| 0.8 | 4.2 | **29** |
| 0.95 | 4.8 | 25 |

**No observations are excluded.** The brief permits retaining 112 only with a preregistered
exclusion of eight observations for a valid reason. **There is no such reason, so 112 is
discarded.**

### 1.3 Corrected MDE

`MDE = (z_{α/2} + z_β)·√(2p̄(1−p̄)/n)` at α = 0.05, power = 0.80, p̄ = 0.20:

| n | Basis | MDE |
|---|---|---|
| 24 | package-level (the defensible unit) | **0.323** |
| 29 | clustered effective n at ρ = 0.8 | **0.294** |
| 120 | naive observation-level | 0.145 — **invalid, ignores clustering** |
| 112 | the figure previously cited | 0.150 — **not an available n** |

> **Corrected: the FAR MDE is ≈ 0.29–0.32, not 0.15. The C-2 threshold of 0.10 is unmeasurable by
> roughly a factor of three.**

### 1.4 Corrected C-2

| | Old | **New** |
|---|---|---|
| Statistic | FAR difference upper bound < 0.10 at n = 112 | **Paired FAR discordance over 24 packages**, exact McNemar; plus the descriptive difference with its Wilson interval |
| Threshold | 0.10 | **A-INT's FAR must not exceed C1's on more than 2 of 24 packages** (zero-tolerance-adjacent, measurable at this n) |
| Reported with | Indeterminate Rate | **Unchanged — IR reporting remains mandatory.** FAR alone is gamed by always returning `INDETERMINATE`. |
| MDE printed | 0.15 (wrong) | **0.29–0.32** |

### 1.5 Every other n in the pre-registration, re-derived

| Claim | Required n | Available n | Status |
|---|---|---|---|
| CDDR primary, E2 subset | 24 defects | **6** | **UNDERPOWERED** — `BASELINE_EXPERIMENT` §6, unresolved |
| CDDR, all defects | 24 | 24 | adequate at 80 % win rate among discordant |
| FAR difference 0.15 | 112 | 24–29 | **not measurable** → replaced (§1.4) |
| Non-inferiority δ = 0.175 | 96 | 24 | **not measurable** → replaced by zero unfavourable discordance |
| FDR on nulls | — | 6 packages × 5 | 0/30 → upper bound 0.116 |
| Self-audit deterministic classes | — | 4 × 5 | exact requirement (1.00), no power needed |

**Three of six claims were specified against an n the experiment does not have.** All three are
now either replaced with a measurable criterion or flagged as underpowered.

---

## 2. Replication information is preserved, not collapsed

### 2.1 The rule

**The defect remains the population unit for the primary comparison.** The majority
classification feeds McNemar. **But the underlying distribution is retained and emitted** — it is
never discarded before analysis.

### 2.2 What is retained, per unit

```
replication_detail:
  unit_id, n_replications
  per_replication_outcomes: [bool]        # the raw 5
  detection_rate: float                   # k/5
  interval: {low, high, method: wilson}
  majority_classification: DETECTED | MISSED
  instability: float                      # fraction disagreeing with the majority
```

### 2.3 Why instability is itself a finding

| Observed | Majority | Meaning |
|---|---|---|
| 5/5 | DETECTED | Reliable detection |
| 3/5 | DETECTED | **Detected 60 % of the time.** Collapsing to "DETECTED" hides that an operator running it once has a 40 % chance of missing it. |
| 2/5 | MISSED | Detected 40 % of the time — **not the same as never** |

> **A system detecting a defect 3 of 5 times is meaningfully different from one detecting it 5 of
> 5, and the majority classification erases the difference.** Mean instability across the registry
> is a **reported headline metric** alongside CDDR, for both arms. An arm with equal CDDR and
> higher instability is worse, and only the retained distribution shows it.

### 2.4 Emission

The per-replication detail appears in `assurance_artifact.reproduction.replication_detail`
(`PRODUCT_INTERFACE` §3.1) and in section 7 of the human report. It is not an appendix.

---

## 3. What remains underpowered after these corrections

| # | |
|---|---|
| U1 | **The CDDR primary on 6 E2 defects still cannot reach significance.** Corrections here do not fix it; only more external defects or an honest non-result does. |
| U2 | Statistical non-inferiority is unreachable at any affordable n. The discordance rule substitutes a **stricter** criterion, not an equivalent one. |
| U3 | ICC is assumed, not measured. If replication outcomes are more independent than ρ = 0.8, effective n rises and the MDE improves — the first ten packages should be used to **measure ICC** and the MDEs re-derived before scoring. |
