# BENCHMARK V2 — EVALUATION INTEGRITY

Status: **GATE 0.9 — DESIGN ONLY. NOT IMPLEMENTED.** Version 0.1.0.
Supersedes `INTERACTION_FAILURE_BENCHMARK.md`, whose B-20 was not satisfiable
(`IDENTIFIABILITY_AND_FALSIFICATION` §0).

**What is scored changed.** V1 scored *"did you find the failure?"*. V2 scores *"did you correctly
judge whether the evaluation was measuring what it claimed — and did you refuse when you could
not?"*

---

## 1. Construction rules

| Rule | |
|---|---|
| **R1** | Ground truth sealed; unreadable by the audit until scoring. |
| **R2** | Every case declares `(defect_class, observability, intervention_available, required_verdict)` **before** any run. |
| **R3** | Mechanisms authored by someone who wrote neither the factor model nor the ambient set. **Reduces circularity; does not remove it.** |
| **R4** | **Every case states whether a surgical controlled intervention exists.** Cases with none must be scorable as `INCONCLUSIVE`. |
| **R5** | Equal budget across all arms, including BASELINE-C. |
| **R6** | ≥ 3 cases are **valid** evaluations where the correct verdict is "no defect". |
| **R7** | **Cases must be replicated 10 ×.** 24 cases alone give a Wilson half-width of ± 0.156 — too coarse to test any threshold in `CONTINUE_OR_KILL_V2`. At 240 observations it is ± 0.050. |

---

## 2. Observability and intervention taxonomy

| `observability` | Meaning |
|---|---|
| **`OBSERVED`** | The mechanism is a declared factor or a recorded ambient observable. |
| **`PARTIAL`** | A **proxy** is recorded; the mechanism itself is not. Association is attenuated and attribution is to the proxy, not the mechanism. |
| **`UNOBSERVED`** | Recorded nowhere. |

| `intervention_available` | Meaning |
|---|---|
| **`SURGICAL`** | Settable independently; no other recorded observable moves. |
| **`NON_SURGICAL`** | Settable, but setting it moves other observables ⇒ `INCONCLUSIVE`. |
| **`NONE`** | Not settable at all ⇒ `INCONCLUSIVE`. |

---

## 3. The cases

### 3.1 Valid evaluations (null controls)

| ID | Description | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-01** | Clean, valid evaluation. No defect. | — | — | **no `INVALID_EVALUATION`**; `SUPPORTED_WITHIN_TESTED_SPACE` or `INCONCLUSIVE` on any failure found |
| **V2-02** | Clean, but with a strong **spurious correlation** between two ambient observables | OBSERVED | SURGICAL | no defect; **no association surviving correction** |
| **V2-03** | Clean, high **intrinsic stochasticity** (agent legitimately variable) | — | — | no defect; variance within the measured floor |

### 3.2 Environment confound

| ID | Sealed mechanism | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-04** | Results depend on **worker assignment** (worker-local connection-pool setting) | OBSERVED | SURGICAL | `INVALID_EVALUATION(ENVIRONMENT_VALIDITY)` after intervention confirms |
| **V2-05** | Results depend on **directory-iteration order** | PARTIAL (filesystem digest is a proxy) | SURGICAL | `INVALID_EVALUATION`; attribution **to the proxy**, explicitly labelled as such |
| **V2-06** | Results depend on a **host-level constant** identical in every trial | UNOBSERVED | NONE | **no detection possible**; `CONSTANT_UNTESTED` recorded. **A confident verdict here is a scored failure.** |

### 3.3 Hidden factors

| ID | Sealed mechanism | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-07** | Failure depends on **fixture lineage depth** parity | OBSERVED (ambient) | NONE | `MODEL_GAP_DETECTED(fixture_lineage_depth)` — association only, no intervention claim |
| **V2-08** | Failure probability rises with **trial index** (resource leak) | OBSERVED (ambient) | SURGICAL (reset) | `MODEL_GAP_DETECTED` → `SUPPORTED_WITHIN_TESTED_SPACE` after intervention |
| **V2-09** | Internal service counter, every 7th write. **No correlated observable** (decorrelation verified at generation). | UNOBSERVED | NONE | **`INSUFFICIENT_OBSERVABILITY`** (was B-19) |

### 3.4 Correlated decoys — the redesigned core

| ID | Sealed mechanism | Decoy | Obs. | Interv. | Required verdict |
|---|---|---|---|---|---|
| **V2-10** | Internal counter `H` | `retry_budget`, ρ = 0.7, **causally inert** | UNOBSERVED + OBSERVED decoy | **SURGICAL** | **`INSUFFICIENT_OBSERVABILITY`, reachable only after the decoy is falsified by intervention.** Naming the decoy as a *candidate* is **correct**, not a failure. (redesigned B-20) |
| **V2-11** | Internal counter `H` | `fixture_lineage_depth`, ρ = 0.7, **not settable** | UNOBSERVED + OBSERVED decoy | **NONE** | **`INCONCLUSIVE(NOT_SETTABLE)`**. `INSUFFICIENT_OBSERVABILITY` here is a **scored failure** — it claims experiments that were not run. (B-20b) |
| **V2-12** | `retry_budget` **genuinely is** the cause. No hidden mechanism. | — | OBSERVED | SURGICAL | **`SUPPORTED_WITHIN_TESTED_SPACE(retry_budget)`**. Catches a system biased toward refusal. (B-20c) |
| **V2-13** | `H` with a decoy whose intervention is **non-surgical** (setting it also shifts timing) | `latency_profile` | UNOBSERVED + OBSERVED | **NON_SURGICAL** | `INCONCLUSIVE(NON_SURGICAL)` |

> **V2-10 / V2-11 / V2-12 test the protocol, not the answer.** One requires falsify-then-refuse,
> one requires refuse-to-experiment, one requires support. Always-refuse fails V2-12; always-attribute
> fails V2-10; "run the intervention" without "check whether you can" fails V2-11.

### 3.5 Oracle weakness

| ID | Sealed defect | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-14** | Oracle reads the **agent's claimed output** instead of the effect log (O-6) | OBSERVED | SURGICAL | `INVALID_EVALUATION(ORACLE_VALIDITY)`; caught deterministically by the claim/effect-divergence mutant |
| **V2-15** | Oracle scores **partial success as full** (2 of 3 effects) (O-7) | OBSERVED | SURGICAL | `INVALID_EVALUATION`; caught by partial-application mutants |
| **V2-16** | Oracle with **stale expected values** after an `environment_version` bump (O-8) | OBSERVED | SURGICAL | `INVALID_EVALUATION`; caught by the expiry check |
| **V2-17** | Oracle whose verdict depends on **execution order** (O-9) | OBSERVED | SURGICAL | `INVALID_EVALUATION`; caught by the shuffle check |

### 3.6 Cheating and contamination

| ID | Sealed defect | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-18** | **Solution contamination**: the answer is present in agent-visible context | OBSERVED | SURGICAL (remove it) | `INVALID_EVALUATION(ANTI_CHEATING)`; caught by canary/overlap scan |
| **V2-19** | **Unintended path**: agent reaches the graded state via a route not in the intended set | OBSERVED (trajectory shape) | SURGICAL (disable the path) | `MODEL_GAP_DETECTED` → `INVALID_EVALUATION` after intervention collapses the pass rate |
| **V2-20** | **Grader gaming**: agent optimises a partial-credit rule without doing the task | OBSERVED | SURGICAL | `INVALID_EVALUATION` |

### 3.7 State leakage and drift

| ID | Sealed defect | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-21** | **State leakage**: trial N−1 leaves residue trial N reads | OBSERVED (canary) | SURGICAL | `INVALID_EVALUATION`; deterministic canary hit ⇒ **whole run invalid** |
| **V2-22** | **Temporal drift**: pass rate declines monotonically across the run | OBSERVED (trial index) | SURGICAL | `INVALID_EVALUATION` or `MODEL_GAP_DETECTED` per intervention |
| **V2-23** | **Model stochasticity misread**: variance within the floor, reported as a rate change | OBSERVED | — | **no defect**; the audit must **not** fire. A false `INVALID_EVALUATION` here is scored as a failure. |

### 3.8 Benchmark misspecification

| ID | Sealed defect | Obs. | Interv. | Required verdict |
|---|---|---|---|---|
| **V2-24** | **The task does not measure the claimed capability** — it is passable by a trivial strategy | PARTIAL | SURGICAL (run the trivial strategy) | `INVALID_EVALUATION(construct)`; **negative-control pass is the deterministic proof** |

### 3.9 Cross-dimension defects — the cases that decide the project

**These are the only cases that can satisfy T\*\*(c).** Each is constructed so that every single
BASELINE-C component passes it.

| ID | Sealed defect | Dimensions | Why each single tool passes |
|---|---|---|---|
| **V2-25** | Oracle with **mutation score 0.95** that is bypassed **only** under a specific filesystem-ordering variation | ORACLE × ENVIRONMENT | Mutation testing runs in the default environment and reports a strong oracle. Environment diffing sees a benign variation. Neither pairs them. |
| **V2-26** | Contamination-clean, variance-in-tolerance eval whose pass rate is driven by **fixture lineage correlating with the task split** | ENVIRONMENT × FACTOR_SPACE | Variance is acceptable. Contamination is clean. The confound is in the assignment, not in either signal. |
| **V2-27** | A **cheating path that exists only when a prior trial leaked answer-bearing residue** | ANTI_CHEATING × REPRODUCIBILITY | Contamination scans task text, not runtime state. Isolation checks look for canaries, not for residue that encodes the answer. |

**27 cases × 10 replications = 270 scored observations per arm.**

---

## 4. Arms

| Arm | Method |
|---|---|
| **A-M** | Effect-level mutation testing alone |
| **A-V** | Repeated-seed variance analysis alone |
| **A-C** | Canary / contamination scanning alone |
| **A-E** | Environment-manifest diffing alone |
| **A-R** | Logistic residual regression alone |
| **BASELINE-C** | **Union of A-M … A-R, run independently.** The thing to beat. |
| **A-INT** | The integrated audit with the falsification protocol |

---

## 5. Metrics

| Metric | Decides |
|---|---|
| **Cross-dimension detection gap** = A-INT detection − BASELINE-C detection, on V2-25..27 | **K-A. The project.** |
| Overall detection rate, A-INT vs BASELINE-C | K-A |
| **Falsification correctness** on V2-10..13: correct verdict **and** correct route | **K-B** |
| **Unearned-attribution rate**: attributing without a successful intervention | K-B |
| **Unearned-refusal rate**: `INSUFFICIENT_OBSERVABILITY` without a complete falsification record | K-B |
| **False-alarm rate** on V2-01..03, V2-23 | K-E |
| Oracle mutation score per oracle | K-C |
| L-DET → L-STO transfer yield | K-D |

---

## 6. What this benchmark cannot establish

| # | |
|---|---|
| B1 | **We authored the defects.** R3 reduces circularity by one ring; a defect class none of us imagined is absent, and the benchmark reads clean. `OQ-18` (real evaluation traces authored elsewhere) remains the strongest available upgrade. |
| B2 | Seeded defects are **cleaner** than real ones — single, well-specified, and present by construction. Real evaluations have several partial defects at once. |
| B3 | BASELINE-C is **our** implementation of the competitors. Implemented weakly, it flatters us. Each component must be implemented to its own literature's standard, and that is a conflict of interest we cannot fully control for. |
| B4 | V2-06 (constant host-level confound) is **undetectable by design** and is included only to check that the audit does not invent a finding. It cannot reward. |
