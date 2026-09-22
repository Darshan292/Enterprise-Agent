# STATISTICAL VALIDITY

Status: **GATE 0.9.** Version 0.1.0.
**Contains simulation results, not a simulation design.** Code in the session scratchpad; it is
analysis, not application code, and is not part of the repository.

---

## 0. An error caught in this audit, reported rather than patched

The first run of this simulation computed FDR as `Σ V / Σ R` pooled across simulations. That
estimator is biased — `E[V]/E[R] ≠ E[V/R]` — and it reported BH's realized FDR at **0.126–0.265**
under dependence, apparently showing that BH fails.

Recomputed correctly, as the mean of the per-simulation false discovery proportion, **BH controls
FDR at ≈ 0.05 under every dependence structure tested.** The original finding was an artifact of
my estimator, not a property of BH.

> Recorded because it is the cheapest possible demonstration of the Gate 0.9 thesis: an analysis
> that looked rigorous, produced a publishable-sounding result, and was wrong for a reason
> invisible in the output. It was caught by re-deriving the estimator, not by any tool.

---

## 1. The design under audit

Gate 0.75 specified BH-FDR at q = 0.05 over a family of **126 hypotheses** (14 ambient
observables × 9 oracles). `OQ-22` raised the concern that power might be near zero.

---

## 2. Simulation setup

m = 126, α = q = 0.05, 1500 simulations per condition. Test statistics drawn from a multivariate
normal with the stated correlation structure; non-null hypotheses given mean shift `z`.

| Condition | Structure |
|---|---|
| `indep` | Identity — independent nulls |
| `blocks` | ρ = 0.5 within each 9-hypothesis oracle block (correlated observables) |
| `temporal` | AR(1), ρ = 0.8 across the family (temporal dependence) |
| `alias` | ρ = 0.3 blocks **plus** ρ = 0.95 between paired observables (**factor aliasing**) |
| `repeat` | ρ = 0.99 within blocks (**repeated trials / near-duplicate tests**) |

Methods: **BH**, **Benjamini–Yekutieli** (BH with the harmonic-sum penalty, valid under arbitrary
dependence), **Westfall–Young maxT permutation** (FWER control, exploits the observed dependence).

---

## 3. Results

FDR = mean per-simulation V/R. FWER = P(at least one false rejection). Power = mean fraction of
non-nulls rejected.

| condition | k | z | BH FDR | BH FWER | BH power | BY FDR | BY FWER | BY power | WY FDR | WY FWER | WY power |
|---|---|---|---|---|---|---|---|---|---|---|---|
| indep | 0 | — | 0.061 | **0.061** | — | 0.010 | 0.010 | — | 0.065 | **0.065** | — |
| indep | 3 | 3.0 | **0.056** | 0.114 | **0.353** | 0.011 | 0.020 | 0.186 | 0.040 | 0.065 | 0.305 |
| indep | 3 | 4.0 | 0.051 | 0.159 | **0.752** | 0.010 | 0.029 | 0.581 | 0.024 | 0.065 | 0.675 |
| indep | 10 | 3.0 | 0.044 | 0.224 | 0.465 | 0.010 | 0.035 | 0.249 | 0.017 | 0.058 | 0.299 |
| blocks | 3 | 3.0 | **0.045** | 0.095 | 0.351 | 0.010 | 0.019 | 0.200 | 0.034 | 0.055 | 0.306 |
| blocks | 3 | 4.0 | 0.050 | 0.140 | 0.762 | 0.009 | 0.025 | 0.578 | 0.022 | 0.055 | 0.701 |
| temporal | 3 | 3.0 | **0.041** | 0.071 | 0.374 | 0.008 | 0.014 | 0.220 | 0.040 | 0.057 | 0.342 |
| temporal | 3 | 4.0 | 0.044 | 0.108 | 0.764 | 0.009 | 0.021 | 0.592 | 0.025 | 0.057 | 0.718 |
| alias | 3 | 3.0 | **0.060** | 0.117 | 0.351 | 0.016 | 0.026 | 0.192 | 0.039 | 0.059 | 0.283 |
| alias | 3 | 4.0 | 0.059 | 0.170 | 0.758 | 0.014 | 0.035 | 0.572 | 0.024 | 0.059 | 0.654 |
| alias | 10 | 3.0 | 0.057 | 0.255 | 0.466 | 0.012 | 0.043 | 0.255 | 0.023 | 0.055 | 0.281 |
| **repeat** | 3 | 3.0 | 0.038 | 0.045 | 0.395 | 0.007 | 0.009 | 0.224 | 0.045 | 0.053 | **0.492** |
| **repeat** | 3 | 4.0 | 0.037 | 0.049 | 0.749 | 0.009 | 0.011 | 0.601 | 0.036 | 0.053 | **0.816** |
| **repeat** | 10 | 3.0 | 0.036 | 0.061 | 0.498 | 0.007 | 0.010 | 0.299 | 0.035 | 0.052 | 0.489 |

### 3.1 Findings

**1. BH controls FDR at ≈ 0.05 under every structure tested**, including aliasing (0.057–0.060)
and near-duplicate tests (0.036–0.038). The block, temporal and repeat structures are positively
dependent, where BH's control is theoretically expected; the simulation confirms it empirically
and, notably, BH is not anti-conservative even under ρ = 0.95 aliasing.

**2. BY is 4–6× conservative and costs roughly half the power.** Realized FDR 0.007–0.017 against
a nominal 0.05; power 0.186 vs BH's 0.353 at z = 3, and 0.581 vs 0.752 at z = 4. **The audit
provides no evidence for adopting BY.** Its guarantee (arbitrary dependence) buys nothing over
BH's observed behaviour here, and it discards half the detections.

**3. WY maxT controls FWER at ≈ 0.055 across all conditions**, a *stronger* guarantee than FDR,
with power between BY and BH — **except under `repeat`, where it beats BH** (0.492 vs 0.395 at
z = 3; 0.816 vs 0.749 at z = 4). Under near-duplicate tests the max-statistic null distribution
narrows, so permutation *gains* power from the dependence that costs the others.

**4. BH's FWER inflates with the number of true effects** (0.114 at k = 3, 0.224 at k = 10). That
is FDR control behaving as designed, not a defect — but it means "BH found five associations" does
not mean five are real, and reports must say FDR, not "significant".

### 3.2 Decision

`DESIGN_DECISION`, and it is deliberately the **boring** one:

- **Keep BH at q = 0.05 as the default.** The audit validates the existing Gate 0.75 choice. It
  was not adopted because it looks better — it was retained because the simulation found no
  failure of control, and switching would cost power for a guarantee we do not need.
- **Use WY maxT for the sub-family containing near-duplicate or aliased observables**, where it
  is both valid under arbitrary dependence and *more* powerful.
- **Do not use BY.** Pure power loss.
- Report, on every association: raw p, adjusted p, q, method, **family size and family digest**.

> Note what this audit's deliverable is: *"the method you already had is fine, and the fancier
> alternative is worse."* An audit that always recommends change is selling change.

---

## 4. Power — the answer to `OQ-22`

Power is **not** near zero, but it has a hard floor. z = 4 is needed for power ≈ 0.75.
Translating z into a detectable proportion shift δ (p̄ = 0.30, balanced):

| n per arm | trials at the cell | wall-clock @2 s | δ at z = 3 | **δ at z = 4** |
|---|---|---|---|---|
| 100 | 200 | 7 min | 0.194 | **0.259** |
| 250 | 500 | 17 min | 0.123 | **0.164** |
| 500 | 1,000 | 33 min | 0.087 | **0.116** |
| **670** | **1,340** | **45 min** | 0.075 | **0.100** |
| 1,000 | 2,000 | 67 min | 0.061 | **0.082** |

> **`OQ-22` resolved: detecting a hidden factor that shifts failure probability by 0.10 costs
> ≈ 1,340 trials at the cell (45 minutes). Below δ ≈ 0.08 nothing is affordably detectable at any
> n we would run.** Every `INSUFFICIENT_OBSERVABILITY` and `SUPPORTED_WITHIN_TESTED_SPACE` verdict
> therefore carries its minimum detectable δ. Absence of evidence is reported as a bound.

---

## 5. Shrinking the family is worth more than changing the method

The nominal family is 14 × 9 = 126. The screening step in `ENVIRONMENT_CONFOUNDER_MODEL` §3.2
admits only dimensions that **actually varied** in the run.

If screening admits 4 dimensions instead of 14, the family falls from 126 to 36. The BH threshold
for the smallest p rises from q/126 to q/36 — a **3.5× less stringent** bar, worth far more power
than any choice among BH, BY and permutation.

**Two conditions make this legitimate rather than post-hoc narrowing:**
1. Screening uses **variance only** — how many distinct values a dimension took — and never looks
   at the outcome.
2. The screening result is **recorded with its digest before any association is computed**, and
   becomes part of the immutable manifest.

Violating either turns a power gain into a Type-I error factory.

---

## 6. Limitations of this audit

| # | Limitation |
|---|---|
| S1 | Normal test statistics were simulated directly. Real statistics come from finite-sample proportion tests and will be discrete and skewed, especially at low rates. |
| S2 | Five dependence structures with hand-chosen ρ. The real structure is unknown and is itself a hidden-variable problem. |
| S3 | Non-nulls all given the same effect size. Real effects vary, and mixed effect sizes change power in ways not simulated. |
| S4 | 1500 simulations gives a Monte Carlo standard error of ≈ 0.006 on a proportion near 0.05, so differences below ~0.015 in the FDR columns are not resolved. |
| S5 | The permutation arm used 300 replicates; more would tighten its critical value slightly and marginally raise its power. |
