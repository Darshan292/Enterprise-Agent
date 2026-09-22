# REDUCTION VALIDITY

Status: **GATE 0.75.** Document version 0.1.0.

Compares four reduction methods against seven measurable criteria, and corrects a Gate 0.5
parameter that the arithmetic shows was wrong.

---

## 1. The four methods

| # | Method | Definition |
|---|---|---|
| **M1** | **ddmin** | Textbook delta debugging. One trial per accept/reject decision. Assumes a deterministic, monotone test. |
| **M2** | **Stochastic ddmin** | ddmin with `r` repetitions per decision; a removal is rejected only if all `r` pass. |
| **M3** | **Failure-aware reduction** | M2 plus **fingerprint-aware acceptance**: a removal is accepted only if the failure retains the *same* fingerprint. |
| **M4** | **Interaction-aware reduction** | M3 plus a **confirmatory subset sweep**: a designed factorial over the candidate set with replication, dropping any factor whose removal does not change the rate. |

---

## 2. The two error modes

A reduction step removes a subset and tests. Two ways to be wrong, and they are **not**
symmetric:

**Type A — over-inclusion.** The reduced configuration genuinely still fails (at rate `p`), but
by chance every trial passed. The removal is rejected, an unnecessary factor is retained, and the
reported minimum is too large. **Decoys survive this way.**

```
P(one decision errs) = (1 − p)^r
```

**Type B — bug-walking.** The reduced configuration fails, but with a *different* mechanism. M1
and M2 see "still fails", accept the removal, and continue reducing **a different bug**, reporting
the destination as the minimum of the origin. Only M3 prevents it, and the fingerprint comparison
costs nothing.

### 2.1 Type A compounds, and this is where Gate 0.5 was wrong

ddmin performs `D ≈ n log₂ n ≈ 54` test executions for n = 14 factors. A per-decision error rate
compounds across all of them:

```
P(at least one over-inclusion) = 1 − (1 − (1−p)^r)^D
```

| reproduction rate p | r = 1 | r = 3 | **r = 5** | r = 12 | r = 27 |
|---|---|---|---|---|---|
| 0.9 | 1.00 | 0.05 | **0.00** | 0.00 | 0.00 |
| 0.6 | 1.00 | 0.97 | **0.43** | 0.00 | 0.00 |
| **0.4** | 1.00 | 1.00 | **0.99** | 0.11 | 0.00 |
| **0.2** | 1.00 | 1.00 | **1.00** | 0.98 | 0.12 |

`r` required for P(no over-inclusion) ≥ 0.95:

| p | r | ddmin trials (= 54 r) |
|---|---|---|
| 0.9 | 4 | 216 |
| 0.6 | 8 | 432 |
| **0.4** | **14** | **756** |
| 0.2 | 32 | 1,728 |
| 0.1 | 67 | 3,618 |

> **`FACT` (arithmetic): `r_confirm = 5`, the value carried in Gate 0.5, gives a 99 % probability
> of at least one over-inclusion at p = 0.4 and a 100 % probability at p = 0.2. As a standalone
> reduction parameter it is wrong by a factor of roughly three.**
>
> Gate 0.5 flagged `r_confirm` as a placeholder (`OQ-07`) but reported reduction costs as though
> 5 were adequate. It is not, **for M3**.

### 2.2 Why M4 rescues `r = 5` — the argument for interaction-aware reduction

Over-inclusion is cheap to fix *afterwards*, because the confirmatory sweep tests subsets with
n = 20 replication — far more power per decision than any single ddmin step. Total cost for a
true set of m = 3 (levels 3 × 3 × 2 ⇒ 18 cells):

| r during search | ddmin trials | likely candidate size | sweep cells × n=20 | **total** |
|---|---|---|---|---|
| 3 | 162 | m = 5 | 108 × 20 = 2,160 | 2,322 |
| **5** | **270** | **m = 4** | **36 × 20 = 720** | **990** |
| 12 | 648 | m = 3 | 18 × 20 = 360 | 1,008 |
| 27 | 1,458 | m = 3 | 18 × 20 = 360 | 1,818 |

> **`r = 5` is defensible only because the confirmatory sweep follows it.** Standing alone it
> produces an over-inclusive minimum almost always. The optimum is a **cheap, deliberately sloppy
> search followed by a powered confirmation** — 990 trials at r = 5 versus 1,008 at r = 12, with
> r = 5 reaching the answer through a design that also rejects decoys.
>
> This is the strongest available argument that **M4 is the method and M3 is not merely a cheaper
> version of it.**

`DESIGN_DECISION`: `r` is **measured, not chosen**. Estimate `p` at the original failing
configuration (n = 20, Wilson interval), then set `r` from the table, clamped to [3, 8] because
M4's sweep absorbs the residual. Report the estimated `p`, the chosen `r`, and the resulting
per-decision error bound on every artifact.

---

## 3. Comparison against the seven criteria

True set m = 3, n = 14 factors, p = 0.4, 2 s per trial. `ASSUMPTION` on p and trial cost.

| Criterion | **M1** ddmin | **M2** stochastic | **M3** failure-aware | **M4** interaction-aware |
|---|---|---|---|---|
| **Trial count** | 54 | 270 (r=5) | 270 | **990** |
| **Wall-clock** @2 s | 1.8 min | 9 min | 9 min | **33 min** |
| **Reproduction probability** of the artifact | unmeasured | measured, but of a possibly-wrong set | measured | **measured, of a confirmed set** |
| **Minimality confidence** | none — "1-minimal" is unjustified under stochasticity | per-decision bound (1−p)^r | same | **subset-sweep evidence: every proper subset tested at n = 20** |
| **False reduction rate** (Type A, over-inclusion) | **1.00** | **0.99** at r=5 | **0.99** at r=5 | **≈ 0** after the sweep |
| | (Type B, bug-walking) | unbounded | unbounded | **0 by construction** | **0** |
| **Stability** (same fingerprint across independent reductions) | low | moderate | high | **high, and measured** |
| **Interaction preservation** — does the minimized set equal the true set? | over-inclusive; decoys retained | over-inclusive | over-inclusive | **exact, to the sweep's power** |

### 3.1 The honest read

- **M1 is unusable** on a stochastic system. Not "suboptimal" — its Type A rate is 1.00.
- **M2 → M3 is free** and removes an unbounded error mode. There is no reason to run M2.
- **M3 → M4 costs 3.7 ×** and is the only step that actually fixes over-inclusion and rejects
  decoys.
- **M4's advantage over M3 is measurable and predicted to be large.** It is also the only one of
  the four that is not in a textbook — and it is an obvious composition of two textbook methods,
  so the contribution is engineering, not research (`RESEARCH_VALIDITY` §D3).

---

## 4. Measurable acceptance criteria

Every criterion is scored on the benchmark (`INTERACTION_FAILURE_BENCHMARK` §4), at equal trial
budget across methods.

| # | Criterion | Definition | Target for M4 |
|---|---|---|---|
| R1 | Trial count to stable artifact | Median trials per failure | Report; no target |
| R2 | Wall-clock | Median minutes per failure | Report; no target |
| R3 | Reproduction probability | Measured rate of the minimized configuration, with Wilson interval | Within the recorded interval on re-execution |
| R4 | Minimality confidence | Fraction of proper subsets tested at n ≥ 20 that failed to reproduce | ≥ 0.95 of subsets tested |
| R5 | **False reduction rate** | P(reported set ⊃ true set), over benchmark cases with known ground truth | **≤ 0.10** |
| R6 | Stability | Fraction of independent reductions of the same original failure yielding the same fingerprint | ≥ 0.80 |
| R7 | **Interaction preservation** | Exact-match rate: reported set == true set | **≥ 0.70**, and strictly greater than M3's |

`DESIGN_DECISION`: **R5 and R7 are the criteria that decide whether M4 earns its 3.7 × cost.** If
M4 does not beat M3 on R7 by a margin of at least 0.15, the confirmatory sweep is not paying for
itself and the reduction contribution collapses to "ddmin with repetitions and a hash", which is
an afternoon of work on top of an existing reducer.

---

## 5. What reduction validity cannot establish

| # | Limitation |
|---|---|
| L1 | A minimized set is minimal **with respect to the declared factor space**. A failure additionally requiring an undeclared condition reduces to a set that is minimal and incomplete simultaneously. Only the adequacy verdict catches this. |
| L2 | The subset sweep's power bounds every minimality claim. At n = 20, "no proper subset reproduces" means "none above ≈ 0.20" (`INTERACTION_ANALYSIS` §4). |
| L3 | Reduction runs in `CONTROLLED_REPLAY` with a stub planner. Whether it reduced the *right* failure — the one a real agent produced — is `OQ-04` and is unresolved. |
| L4 | Non-monotone failures terminate in `REDUCTION_UNSTABLE`, which is correct and is **not** a minimized artifact. The rate of `REDUCTION_UNSTABLE` is itself a reported metric; a high rate means the method does not apply to this failure population. |
