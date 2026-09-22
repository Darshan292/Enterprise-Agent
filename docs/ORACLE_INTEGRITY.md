# ORACLE INTEGRITY

Status: **GATE 0.9.** Version 0.1.0.

**The analogue is exact and must be stated first: measuring whether a grader would notice a
defect is mutation testing, a technique from the 1970s with mature tooling (PIT, mutmut,
Stryker). Nothing in this document is a new method.** The only adaptation is *what* gets
mutated: a stochastic policy has no meaningful source-level mutants, so we mutate the **effect
log and trial record** instead.

---

## 1. The ten failure modes

| # | Failure mode | What it looks like | Detectable deterministically? |
|---|---|---|---|
| **O-1** | **Grader gaming** | Agent optimises the measured proxy, not the task | **Partially** — unintended-path detection; needs a model of intended paths |
| **O-2** | **Solution contamination** | Answer present in agent-visible context or fixtures | **Yes** — canary tokens + n-gram overlap scan |
| **O-3** | **Test bypass** | A code path reaches the graded state without doing the work | **Yes** — effect-log mutation: remove the required effect, assert the oracle fails |
| **O-4** | **Hard-coded answers** | Oracle compares against a literal that happens to match | **Yes** — mutate the expected value; if the oracle still passes, it is not reading it |
| **O-5** | **Assertion manipulation** | Assertion weakened, inverted, or made unreachable | **Yes** — mutation of the oracle's own predicate; plus known-fail fixtures |
| **O-6** | **Checking outputs instead of intended effects** | Oracle reads what the agent *said*, not what *happened* | **Yes — the single most valuable check.** See §3 |
| **O-7** | **Partial success scored as full** | Two of three required effects applied; oracle returns `HOLDS` | **Yes** — mutation: delete one required effect at a time |
| **O-8** | **Stale expected values** | Expectation matches a schema/state that no longer exists | **Yes** — expected values carry a pinned `environment_version` and an expiry |
| **O-9** | **Shared evaluator state** | Oracle N's verdict depends on oracle N−1 having run | **Yes** — shuffle oracle execution order and diff verdicts; run each in isolation |
| **O-10** | **Leakage between trials** | Trial N sees trial N−1's residue | **Yes** — per-trial canary with a nonce; a hit invalidates the whole run |

**Eight of ten are deterministically detectable.** O-1 is partial and O-2 is partial for
training-set contamination (we cannot inspect a provider's training data; only context-level
contamination is checkable).

---

## 2. Effect-level mutation testing — the primary instrument

```
MUTATION_SCORE(oracle) = (mutants the oracle catches) / (applicable mutants)
```

### 2.1 The mutation catalogue

Applied to the **trial record and ground-truth effect log**, not to source:

| Class | Mutation | Targets |
|---|---|---|
| **Effect deletion** | Remove one required effect | O-3, O-7 |
| **Effect duplication** | Duplicate an effect with the same `effect_id` | INV-1 |
| **Effect duplication (distinct id)** | Duplicate with a different `effect_id` | INV-1 under schema drift |
| **Field perturbation** | Off-by-one, wrong resource, wrong actor, sign flip | O-4 |
| **Authorization strip** | Remove the authorization record for an applied effect | INV-2 |
| **Order permutation** | Swap two ordered effects | O-3 |
| **Partial application** | Apply k of n required effects, for every k | O-7 |
| **Claim/effect divergence** | Agent claims success; effect log shows nothing | **O-6** |
| **Stale expectation** | Bump `environment_version`, leave the expectation | O-8 |
| **Timing shift** | Move an effect outside its required window | temporal predicates |

### 2.2 Power — how many mutants are needed

Two-proportion, α = 0.05, power = 0.80, distinguishing mutation scores:

| Resolution | Mutants per oracle |
|---|---|
| 0.90 vs 0.70 (Δ = 0.20) | **63** |
| 0.90 vs 0.80 (Δ = 0.10) | **200** |
| 0.90 vs 0.85 (Δ = 0.05) | **686** |

`DESIGN_DECISION`: **63 mutants per oracle** is the standard suite — enough to distinguish a
strong oracle from a weak one (0.90 vs 0.70), which is the distinction that changes what we do.
Finer resolution is not worth 200+ mutants per oracle × 9 oracles.

### 2.3 Thresholds

| Mutation score | Status |
|---|---|
| ≥ 0.90 | Oracle accepted |
| 0.70 – 0.90 | Oracle accepted **with the escaped mutants listed in every report using it** |
| < 0.70 | **Oracle rejected. Results depending on it are withheld, not caveated.** |

**Escaped mutants are published, not summarised.** "Mutation score 0.85" is much less useful than
"the oracle does not catch partial application of 2 of 3 effects."

---

## 3. O-6 is the check worth building first

*Checking outputs instead of intended effects* is the most common and most damaging grader defect
in agent evaluation, because the agent's own report is the easiest thing to read and it is exactly
the thing that cannot be trusted.

**Structural fix, not a test:** an oracle is constructed with a `GroundTruthReadCapability` and
**is not given** a handle to the agent's claimed output. A grader that reads the agent's claim
cannot be written, because the object is not in scope. The `claim/effect divergence` mutant then
verifies the property empirically: agent claims success, effect log is empty, oracle **must**
return `VIOLATED`.

`ASSUMPTION`: capability scoping is enforceable in practice in Python but not against a
determined bypass. Structural, not airtight — the mutant is what makes it checked rather than
assumed.

---

## 4. Deterministic integrity checks — the full list

Each is a build-failing test.

| # | Check | Catches |
|---|---|---|
| OI-1 | Every oracle scores ≥ 0.70 on its 63-mutant suite | O-3,4,5,7 |
| OI-2 | Oracles hold `GroundTruthReadCapability` and no handle to agent claims | **O-6** |
| OI-3 | Degenerate input (empty, truncated, crashed) yields an explicit verdict; **defaulting to `HOLDS` fails the build** | O-5 |
| OI-4 | Expected values carry `environment_version` + expiry; expired ⇒ `INDETERMINATE`, never `HOLDS` | O-8 |
| OI-5 | Shuffling oracle execution order changes no verdict | **O-9** |
| OI-6 | Each oracle produces identical verdicts run in isolation vs in the suite | O-9 |
| OI-7 | Per-trial canary nonce: a prior trial's canary visible ⇒ **whole run invalid** | O-10 |
| OI-8 | Agent credential **denied** on the ground-truth effect log | O-2, O-6 |
| OI-9 | Negative-control tasks (impossible) never pass | **O-1, O-2** |
| OI-10 | Canary tokens planted in grader messages never appear in agent context | O-2 |
| OI-11 | n-gram overlap between task fixtures and agent-visible context below threshold | O-2 |
| OI-12 | Zero-firing oracle across a large run is flagged for meta-testing | O-5 |

---

## 5. Unintended-path detection (O-1) — the honest partial

Requires a model of intended paths, which we author. Method:

1. Record the canonical trajectory shape (tool names + effect classes, ordered) for every trial.
2. Cluster shapes among **passing** trials.
3. A passing cluster whose shape is not in the declared intended set is a **candidate** unintended
   path — an `OBSERVATIONAL_ASSOCIATION`.
4. **Intervene**: remove the path (disable the tool, tighten the policy) and re-run. If the pass
   rate collapses, the path was load-bearing ⇒ `SUPPORTED_WITHIN_TESTED_SPACE`. If not,
   falsified.

**Limitation:** an unintended path we never imagined, whose shape resembles an intended one, is
invisible. Step 4 is the cleanest intervention in the whole integrity model — the path is
something we can delete — which is why O-1 is partial rather than undetectable.

---

## 6. What oracle integrity cannot establish

| # | Limitation |
|---|---|
| L1 | Mutation score is bounded by the catalogue **we** wrote. A defect class absent from it is invisible, and the score reads high. |
| L2 | A high mutation score **in the default environment** says nothing about the oracle under an environment confound. This is cross-dimension defect #1 (`EVALUATION_INTEGRITY_MODEL` §6) and is the whole Gate 0.9 bet. |
| L3 | Training-data contamination is not checkable without provider training-set access. Only context-level contamination is. |
| L4 | O-1 depends on a declared model of intended paths, authored by us. |
| L5 | Every check here is an established technique. The contribution is the catalogue and the capability scoping, both of which are engineering. |
