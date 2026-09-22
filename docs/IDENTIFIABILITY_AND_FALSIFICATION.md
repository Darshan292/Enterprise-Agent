# IDENTIFIABILITY AND FALSIFICATION

Status: **GATE 0.9 — MANDATORY.** Version 0.1.0.

Corrects the epistemic flaw in Gate 0.75 case B-20. Every verdict in `VERDICT_SEMANTICS.md`
depends on the distinctions made here.

---

## 1. Four evidence kinds

| Kind | Definition | What it licenses | What it never licenses |
|---|---|---|---|
| **`OBSERVATIONAL_ASSOCIATION`** | A relationship measured in data whose assignment we did not control. | "X and the outcome co-vary at rate r [CI]." A **candidate**. | Any statement that X matters. Any exclusion of an unobserved alternative. |
| **`CONTROLLED_INTERVENTION`** | We **set** X ourselves, independently of everything else, holding the fixture and seed family fixed, and measure the outcome. | "Setting X to a changed the rate from p₀ to p₁ [CI], within this fixture." | Generalisation beyond the fixture. Identification of anything we did not set. |
| **`FALSIFICATION`** | A controlled intervention whose result is **incompatible** with a stated candidate explanation. | "X is not sufficient for the failure." / "X is not necessary." A candidate is **removed**. | "Therefore Y is the cause." Falsifying one candidate promotes none. |
| **`INDETERMINATE`** | The discriminating intervention was **not performed** — unavailable, unsafe, non-surgical, or underpowered. | Nothing. It is an honest stop. | Everything. Particularly: it must never be reported as `INSUFFICIENT_OBSERVABILITY`, which is a stronger claim. |

### 1.1 The identifiability statement

`FACT`: let `H` be unobserved and `D` observed with `corr(H, D) = ρ > 0`. The causal structures

```
  (i)  D ──▶ failure                    (ii)  H ──▶ failure
                                              H ──▶ D
```
induce **identical distributions over all observed variables**. No estimator — residual
regression, mutual information, conditional independence testing over observables, or any
amount of data — separates them. They are **observationally equivalent**.

> **Therefore: a system given only observational data cannot be required to identify `D` as a
> decoy. Requiring it, as Gate 0.75 B-20 did, is requiring the impossible and scoring the method
> as catastrophic for behaving correctly.**

The distributions differ under **intervention**: `do(D := d)` breaks the `H → D` edge. In world
(i) the failure rate changes; in world (ii) it does not. That is the only discriminator, and it
is not statistical — it is experimental.

---

## 2. The falsification protocol

Applied to **every** candidate explanation before any attribution. Deterministic, no LLM.

```
FALSIFY(candidate D, failure F, fixture, seed_family):

 1. OBSERVE      measure assoc(D, F) on observational trials.
                 no association above the floor -> D is not a candidate. STOP.

 2. CHECK        is a SURGICAL intervention on D available?
                 surgical = D can be set independently, and no other recorded
                 observable shifts beyond its null band when we do.
                 not available -> INDETERMINATE. STOP. (Do NOT proceed to 5.)

 3. INTERVENE    do(D := each level), balanced, fixture and seed family held fixed,
                 n per arm from the power table in section 4.
                 verify surgicality: assert no other recorded observable moved.
                 surgicality check FAILS -> INDETERMINATE. STOP.

 4. COMPARE      rate at do(D := off)  vs  rate at do(D := on)

      persists, difference within the null band  -> D FALSIFIED as sufficient
      vanishes or drops below the floor          -> D SUPPORTED within this fixture
      partial, interval excludes both 0 and full -> D is a MODIFIER, not the mechanism
      intervals too wide to distinguish          -> INDETERMINATE (underpowered)

 5. ACCOUNT      only if EVERY observable candidate has been falsified,
                 AND the failure still reproduces above the floor,
                 AND the observability inventory is complete and recorded:
                      -> INSUFFICIENT_OBSERVABILITY
                 otherwise -> INCONCLUSIVE
```

### 2.1 Three rules that make this bind

**Rule 1 — Falsification is asymmetric.** Falsifying `D` removes `D`. It promotes nothing. There
is no step in which eliminating candidates produces a positive claim about an unobserved
mechanism. The strongest conclusion available after falsifying everything is *"nothing we can
observe accounts for this"*, which is a statement about our instrumentation, not about the world.

**Rule 2 — Surgicality is verified, not assumed.** An intervention that also perturbs `H` is not
an intervention on `D`. We cannot check this against `H` (unobserved), but we **can** check that
no *other recorded* observable moved outside its null band. That is a necessary, not sufficient,
condition — and it is the strongest check available. Failing it yields `INDETERMINATE`.

**Rule 3 — `INDETERMINATE` and `INSUFFICIENT_OBSERVABILITY` are different and the difference is
the product.** `INDETERMINATE` = *we did not do the experiment*. `INSUFFICIENT_OBSERVABILITY` =
*we did the experiments, they all came back negative, and the residual survives*. Gate 0.75
collapsed these into `UNKNOWN_MODEL`, which is why it demanded an unearnable verdict.

---

## 3. B-19 and B-20 redesigned

### 3.1 B-19 — unobserved mechanism, **no** correlated observable

| | |
|---|---|
| **Sealed mechanism** | An internal counter in the ticket service flips behaviour every 7th write. Never emitted, never logged, not derivable from any recorded field. |
| **Correlated observables** | **None by construction.** The generator explicitly decorrelates the mechanism from every ambient observable (verified at generation time by checking each association against the null band). |
| **Intervention available?** | **No** — nothing observable to intervene on. |
| **Required path** | Step 1 finds no candidate above the floor. Failure reproduces above the floor. Observability inventory complete. |
| **Required verdict** | **`INSUFFICIENT_OBSERVABILITY`** |
| **Wrong answers** | Any attribution; or `INCONCLUSIVE` (the inventory *was* complete and no candidate existed — there was nothing left to do). |

B-19 is **fair** under the old design too: with no correlated observable, there is nothing to
misattribute to. It was never the problematic case.

### 3.2 B-20 — unobserved mechanism **with** a correlated observable decoy (redesigned)

| | |
|---|---|
| **Sealed mechanism** | Same internal counter `H`. |
| **Decoy** | `retry_budget` (`D`), generated with `corr(H, D) = 0.7`. **`D` has no effect on the failure.** |
| **Intervention available?** | **Yes.** `retry_budget` is a declared factor the runner can set. |
| **Required path** | 1. Observe: `D` associates with `F`. **Reporting `D` as a candidate here is CORRECT, not a defect.** 2. Surgicality check on `do(retry_budget)` passes. 3. Intervene, balanced, n per §4. 4. Failure **persists** at `do(D := off)` → **`D` FALSIFIED**. 5. All observable candidates falsified, failure survives, inventory complete → **`INSUFFICIENT_OBSERVABILITY`**. |
| **Required verdict** | **`INSUFFICIENT_OBSERVABILITY`, reached only via step 4.** |
| **Scored failures** | (a) Attributing to `D` **without** running the intervention. (b) Reporting `INSUFFICIENT_OBSERVABILITY` **without** the falsification record. (c) Reporting `INCONCLUSIVE` when the intervention was available and adequately powered. |
| **Explicitly NOT a failure** | Naming `D` as a candidate at step 1. **This is what Gate 0.75 wrongly scored as catastrophic.** |

### 3.3 B-20b — the new hard case: **decoy that cannot be intervened on**

Added because the redesigned B-20 is now passable by any system that runs the protocol, and a
benchmark everyone passes measures nothing.

| | |
|---|---|
| **Sealed mechanism** | Same `H`. |
| **Decoy** | `fixture_lineage_depth` — an **ambient observable**, correlated at 0.7, which the runner **cannot set independently** (it is a consequence of the restore path, not an input). |
| **Intervention available?** | **No.** Step 2 fails the surgicality check. |
| **Required verdict** | **`INCONCLUSIVE`**, with the reason recorded as *"discriminating intervention unavailable: `fixture_lineage_depth` is not settable"*. |
| **Scored failures** | Attributing to the decoy; **or** reporting `INSUFFICIENT_OBSERVABILITY` (which would claim the experiments were done when they were not). |

**B-20b is the case that separates a disciplined engine from one that has merely memorised the
protocol.** The tempting wrong answer — `INSUFFICIENT_OBSERVABILITY` — is exactly the answer a
system optimised on B-20 would give.

### 3.4 B-20c — decoy that **survives** falsification

| | |
|---|---|
| **Sealed mechanism** | `retry_budget` genuinely **is** the cause. No hidden mechanism. |
| **Design intent** | The mirror image. A system biased toward refusal will call this `INSUFFICIENT_OBSERVABILITY` and be wrong in the opposite direction. |
| **Required verdict** | **`SUPPORTED_WITHIN_TESTED_SPACE(retry_budget)`**, via a successful intervention at step 4 where the failure vanishes. |

> **B-20, B-20b and B-20c together test the protocol rather than the answer.** One requires
> falsification then refusal, one requires refusal-to-experiment, one requires support. A system
> that always refuses fails B-20c; a system that always attributes fails B-20; a system that has
> learned "run the intervention" but not "check whether you can" fails B-20b.

---

## 4. Power for the falsification step

The intervention is a two-proportion comparison at `do(D := on)` vs `do(D := off)`, balanced.
α = 0.05, power = 0.80, p̄ ≈ 0.4:

| Difference we must be able to detect | n per arm | trials per falsification |
|---|---|---|
| 0.30 | 44 | **88** |
| 0.20 | 95 | **190** |
| 0.10 | 357 | **714** |

`DESIGN_DECISION`: the default falsification runs at **n = 95 per arm (190 trials, ≈ 6 min)**,
giving a minimum detectable difference of **0.20**. A "persists" conclusion therefore means
*"the rate did not change by more than 0.20"* — and that bound is printed on the verdict. A
decoy with a genuine effect of 0.15 would be falsified in error at this n, which is why the
bound is part of the output rather than a footnote.

---

## 5. What falsification cannot do

| # | Limitation |
|---|---|
| L1 | **It cannot identify the hidden mechanism.** Ever. It removes candidates. |
| L2 | It is valid **within the fixture and seed family**. Nothing generalises beyond them. |
| L3 | Surgicality is checked against *recorded* observables only. An intervention that perturbs something unrecorded passes the check and is still invalid. |
| L4 | Underpowered interventions produce `INDETERMINATE` that is easy to mistake for a finding. The minimum detectable difference is printed on every verdict for this reason. |
| L5 | `INSUFFICIENT_OBSERVABILITY` depends on the **observability inventory being complete** — i.e. on us having correctly listed what we record. That inventory is authored by us, so the verdict is relative to it and is recorded with its digest. |
