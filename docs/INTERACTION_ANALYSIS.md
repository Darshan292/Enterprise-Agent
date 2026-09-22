# INTERACTION ANALYSIS

Status: **GATE 0.75.** Document version 0.1.0.

How the system distinguishes seven failure kinds. Supersedes nothing in `INTERACTION_MODEL.md`;
that document covers *generation*, this one covers *discrimination*.

**Vocabulary constraint:** the word *causal* is not used. The design supports **controlled
intervention within a fixture** and nothing stronger (`ARCHITECTURE_SURFACE` L.2). Statements
are either associations or intervention results.

---

## 1. The seven kinds and their discriminating tests

| Kind | Discriminating test | Evidence type |
|---|---|---|
| **Single-factor** | Failure rate at level ℓ of F differs from baseline, **with all other factors marginalised**; and the confirmatory one-factor design reproduces it | `CONTROLLED_INTERVENTION_RESULT` |
| **2-way interaction** | In the 2-factor confirmatory factorial, **neither main effect alone reproduces** the failure and the interaction term is non-zero after FDR | `CONTROLLED_INTERVENTION_RESULT` |
| **3-way interaction** | In the 3-factor factorial, **no 2-subset reproduces** it and the 3-way term is non-zero after FDR | `CONTROLLED_INTERVENTION_RESULT` |
| **Higher-order (≥4)** | No proper subset of the minimized set reproduces it | `CONTROLLED_INTERVENTION_RESULT`, **with a power caveat** — see §4 |
| **History-dependent** | Failure rate is **invariant to factor assignment** but associates with `sequence_signature` or `history_depth` | `STATISTICAL_ASSOCIATION` first; intervention only if the sequence can be constructed deliberately |
| **Concurrency-dependent** | Invariant to assignment at fixed `concurrency` level, but associates with `interleaving_signature` | `STATISTICAL_ASSOCIATION`; intervention only if interleaving is schedulable (`OQ-19`) |
| **Unexplained** | Residual above the reproducibility floor; no declared factor and no ambient observable survives FDR | **`INDETERMINATE` / `UNKNOWN_MODEL`** |

---

## 2. The discrimination procedure

```
 1. Minimize (REDUCTION_VALIDITY.md) -> candidate factor set M, |M| = m
 2. If m = 0:
       -> not a factor-level failure. Go to step 6.
 3. Confirmatory factorial over M's levels, n replications per cell
       for each proper subset S ⊂ M:
           does S reproduce the failure at a rate above the floor?
       if any singleton reproduces  -> SINGLE_FACTOR (M was over-inclusive; decoys present)
       if no (m-1)-subset reproduces -> ORDER m INTERACTION
       else                          -> reduce M to the largest reproducing subset, repeat
 4. Decoy check: any f ∈ M whose removal does not change the rate is a DECOY -> drop it
 5. Report order = |M| after decoy removal
 6. Residual analysis (FACTOR_MODEL_ADEQUACY.md L3) on the ambient set:
       sequence_signature / history_depth   -> HISTORY_DEPENDENT
       interleaving_signature               -> CONCURRENCY_DEPENDENT
       other ambient hit                    -> UNEXPLAINED + promotion candidate
       nothing survives FDR                 -> UNKNOWN_MODEL
```

`DESIGN_DECISION`: **step 3's subset sweep is the discriminator, not the regression coefficient.**
A non-zero interaction coefficient is consistent with several structures; "no proper subset
reproduces it" is a direct, interpretable, intervention-based statement that needs no modelling
assumption. The regression is reported alongside as corroboration, never as the verdict.

---

## 3. Cost of the subset sweep

Full factorial over m factors at their declared levels, n = 20 per cell:

| m | Example levels | Cells | Trials | @2 s |
|---|---|---|---|---|
| 2 | 3 × 3 | 9 | 180 | 6 min |
| 3 | 3 × 3 × 2 | 18 | 360 | 12 min |
| 4 | 3 × 3 × 2 × 2 | 36 | 720 | 24 min |
| 5 | 3 × 3 × 3 × 2 × 2 | 108 | 2,160 | 72 min |

`DESIGN_DECISION`: confirmatory factorials are run up to **m = 4**. At m ≥ 5 the sweep is run on a
**fractional** design and the verdict is reported as `ORDER ≥ 5, UNRESOLVED` rather than as a
specific order. Claiming a resolved 5-way interaction from a fractional design would be
unsupported, and 5-way interactions are rare enough that the honest non-answer costs little.

---

## 4. Power, and the caveat that must accompany every order claim

At n = 20 per cell the Wilson half-width near p = 0.5 is **± 0.20**. A "no proper subset
reproduces it" conclusion is therefore a statement about detectable rates, not about zero.

> **Every interaction-order claim carries its minimum detectable rate.** "No 2-subset reproduced
> the failure at n = 20 per cell" means *no 2-subset reproduced it above ≈ 0.20*. A subset
> reproducing at 0.05 would be recorded as non-reproducing and the order would be overstated by
> one.

This is the main way interaction order gets inflated, and it is inflated in the direction that
makes findings sound more interesting. The bound is printed, not implied.

---

## 5. Association vs intervention — which kinds can be which

| Kind | Can reach `CONTROLLED_INTERVENTION_RESULT`? |
|---|---|
| Single, 2-way, 3-way, higher-order | **Yes** — factor levels are assigned by us in a designed factorial |
| History-dependent | **Only if** the sequence is deliberately constructible by the planner. Otherwise it stays `STATISTICAL_ASSOCIATION`, however strong. |
| Concurrency-dependent | **Only if** the interleaving is schedulable (`OQ-19`). Otherwise `STATISTICAL_ASSOCIATION`. |
| Unexplained | Never. `INDETERMINATE` / `UNKNOWN_MODEL`. |

`FACT`: an association discovered in ambient data is an association no matter how large the
effect. Promotion to intervention requires the ability to **assign** the condition, not merely to
observe it. The `InterventionCapability` is held only by the Phase-3 runner, which by construction
can only assign declared factors — so an ambient finding **cannot** be relabelled as an
intervention until the observable is promoted to a factor and a designed run is executed.

---

## 6. What a report contains for each failure

```
failure_id, fingerprint, fingerprint_stability
kind                      SINGLE | ORDER_2 | ORDER_3 | ORDER_>=4 | ORDER_>=5_UNRESOLVED
                          | HISTORY_DEPENDENT | CONCURRENCY_DEPENDENT | UNEXPLAINED
minimized_set             [factor:level]           after decoy removal
decoys_rejected           [factor:level]           WITH the evidence that rejected each
subset_sweep              {subset -> rate, CI}     the discriminating data
min_detectable_rate       from n per cell          the §4 caveat, numeric
adequacy_verdict          EXPLAINED | PARTIALLY_EXPLAINED | UNEXPLAINED | UNKNOWN_MODEL
ambient_associations      [observable, effect, raw p, adj p, q, family size]
evidence_type             CONTROLLED_INTERVENTION_RESULT | STATISTICAL_ASSOCIATION | INDETERMINATE
layer                     L-DET | L-STO | L-REM        never mixed
transfer                  n, reproduced, lower bound   for L-DET findings
```

`decoys_rejected` is listed with its evidence deliberately: **the decoys a system rejects are
better evidence of its specificity than the factors it reports.** A report that never rejects
anything has not demonstrated discrimination.
