# CROSS-DIMENSION DEFECTS

Status: **GATE 0.95 — REGISTRY DESIGN. SEALED CONTENTS NOT YET AUTHORED.** Version 0.1.0.

**24 cross-dimension defects + 6 null controls = 30 packages.** The count is set by arithmetic,
not preference: 12 defects reaches significance only if *every* discordant defect favours A-INT
(`GATE_0_95_BASELINE_CHALLENGE` §8.2). **24 is the minimum that tolerates a realistic 80 % win
rate among discordant defects.** Replications: **5**, not 10 — replication buys per-defect
precision, not population power.

---

## 0. Required properties, and where each is met

| Requirement | Met by | Count |
|---|---|---|
| ≥ 12 cross-dimension defects | X-01 … X-24 | **24** |
| ≥ 4 externally authored (E2) | X-05, X-09, X-14, X-17, X-21, X-24 | **6** |
| ≥ 3 containing decoy factors | X-03, X-08, X-12, X-16, X-20 | **5** |
| ≥ 2 involving unobserved variables | X-06, X-13, X-19, X-22 | **4** |
| ≥ 2 requiring controlled intervention to resolve | X-03, X-08, X-11, X-16, X-20, X-23 | **6** |
| ≥ 2 null controls | N-01 … N-06 | **6** |

Authorship tiers: **E0** = A-INT team · **E1** = team member blind to the factor model ·
**E2** = outside the project, given only the environment API · **E3** = real incident (none
available, `OQ-18`).

---

## 1. The registry

`obs` = observability of the true mechanism: **O** observed · **P** partially observed (proxy
only) · **U** unobserved.
`int` = surgical controlled intervention available: **Y** / **N** / **NS** (non-surgical).

### 1.1 Environment × Oracle

| ID | Dimensions | Mechanism (to be sealed) | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-01** | env × oracle | Oracle scores 0.95 on mutation testing but is bypassed **only** under reverse filesystem-iteration order, which makes an unintended path reachable | O | Y | E0 | — |
| **X-02** | env × oracle | Oracle's timeout is shorter than the slow-host latency profile, so slow trials are scored `HOLDS` on truncated evidence | P | Y | E1 | — |
| **X-03** | env × oracle | As X-01, plus `latency_profile` correlated at 0.7 with the failing group but causally inert | O + decoy | Y | E0 | **`latency_profile`** |

### 1.2 Environment × Contamination

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-04** | env × contamination | A cached artifact from a prior build contains the answer; present only on workers whose image layer was reused | P | Y | E0 | — |
| **X-05** | env × contamination | A shared temp directory retains a prior run's solution file; visible only when the working directory is not reset | O | Y | **E2** | — |

### 1.3 State × Contamination

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-06** | state × contamination | Trial N−1 writes answer-bearing residue that trial N reads; the residue field is **not recorded** in telemetry | **U** | Y (disable reset) | E0 | — |
| **X-07** | state × contamination | The evaluation's own warm cache is seeded from a run whose tasks overlap the test split | P | Y | E1 | — |

### 1.4 Agent × Grader

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-08** | agent × grader | The agent discovers a partial-credit rule and optimises it; passing trajectories cluster into an unintended shape. `retry_budget` co-occurs at 0.7 and is inert | O + decoy | Y | E0 | **`retry_budget`** |
| **X-09** | agent × grader | The agent emits a claim the grader reads instead of the effect log; effects are never applied | O | Y | **E2** | — |

### 1.5 Model stochasticity × Oracle

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-10** | stochasticity × oracle | The oracle is order-dependent; agent stochasticity changes effect ordering, so the verdict varies with sampling rather than with correctness | O | Y | E0 | — |
| **X-11** | stochasticity × oracle | Reported pass rate comes from a single run per task; true variance is 0.25, so the score is a draw, not an estimate. **Requires intervention (re-run at n) to establish.** | O | **Y** | E1 | — |

### 1.6 Tool behaviour × Evaluator

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-12** | tool × evaluator | A tool's eventual consistency makes the evaluator's post-check read a stale replica; passes are recorded as failures at a rate that varies with load. `concurrency` correlated at 0.65, inert | P + decoy | Y | E0 | **`concurrency`** |
| **X-13** | tool × evaluator | A tool silently truncates a field above a length the evaluator never inspects; **truncation is not recorded anywhere** | **U** | N | E1 | — |

### 1.7 History × Environment

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-14** | history × env | Failure requires ≥ 3 operations on one resource **and** a worker with a smaller connection pool; neither alone suffices | O | Y | **E2** | — |
| **X-15** | history × env | A long trial crosses a log-rotation boundary, losing the evidence the oracle needs; scored `HOLDS` on absence | P | Y | E0 | — |

### 1.8 Concurrency × Oracle

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-16** | concurrency × oracle | Two workers interleave check-then-act; the oracle's snapshot is taken between them and sees a consistent-looking but impossible state. `latency_profile` correlated 0.7, inert | O + decoy | **NS** (setting concurrency also shifts timing) | E0 | **`latency_profile`** |
| **X-17** | concurrency × oracle | The oracle holds shared state across parallel evaluations; verdicts depend on which worker ran first | O | Y (shuffle) | **E2** | — |

### 1.9 Context × Benchmark

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-18** | context × benchmark | The task is solvable from context alone without using any tool; the benchmark claims to measure tool use | O | Y (run the trivial strategy) | E0 | — |
| **X-19** | context × benchmark | Context length correlates with task difficulty in the fixture set, so the score measures context handling, not the claimed capability. **The generating variable is not recorded.** | **U** | N | E1 | — |

### 1.10 Dependency drift × Scoring

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-20** | dependency × scoring | A transitive dependency changed a default serialisation; the scorer's equality check now fails on semantically identical output. `model_config` correlated 0.6, inert | O + decoy | Y (pin) | E0 | **`model_config`** |
| **X-21** | dependency × scoring | A library upgrade changed float formatting; the scorer's tolerance was written against the old format | O | Y | **E2** | — |

### 1.11 Fixture lineage × Task split

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-22** | lineage × split | Fixtures for the "hard" split were generated from a later snapshot containing a schema change; difficulty is confounded with schema. **Lineage-to-split mapping is not recorded.** | **U** | N | E0 | — |
| **X-23** | lineage × split | Restore depth correlates with split assignment; deeper restores leave residue that helps. **Requires intervention (force uniform depth) to establish.** | O (ambient) | **Y** | E1 | — |

### 1.12 Execution order × Grader

| ID | Dimensions | Mechanism | obs | int | E | Decoy |
|---|---|---|---|---|---|---|
| **X-24** | order × grader | The grader's expected-value cache is populated by the first task it sees; later tasks are scored against the wrong expectation | O | Y (shuffle) | **E2** | — |

### 1.13 Null controls

| ID | Contents | Correct behaviour |
|---|---|---|
| **N-01** | Clean, valid evaluation | No defect reported |
| **N-02** | Clean, with a strong spurious correlation between two ambient observables | No defect; **no association surviving correction** |
| **N-03** | Clean, high legitimate agent stochasticity (true variance 0.30) | No defect; variance within the measured floor |
| **N-04** | Clean, but with a **constant** environment dimension that differs from our usual host | No defect; `CONSTANT_UNTESTED` noted. **A confident finding here is scored as a failure.** |
| **N-05** | Clean, with an oracle that legitimately returns `INDETERMINATE` often | No defect; high indeterminate rate is **not** a defect |
| **N-06** | Clean, **externally authored** (E2) — the outside author was asked to build a *valid* evaluation | No defect. Tests whether we flag unfamiliar-but-valid designs. |

> **N-06 is the most informative null.** A system tuned on our own conventions may flag anything
> built differently. That failure mode is invisible against nulls we wrote ourselves.

---

## 2. Scoring a detection

A defect counts as **detected** when the arm reports `INVALID_EVALUATION` (or the correct
non-assurance verdict) **and names at least one of the two implicated dimensions**.

| Metric | Requirement |
|---|---|
| **CDDR** (primary) | Names ≥ 1 implicated dimension |
| **CDDR-strict** (secondary) | Names **both** dimensions |
| **Attribution correctness** | For decoy cases, the decoy is **not** in the final attributed set |
| **Route correctness** | For intervention-required cases (X-03, X-08, X-11, X-16, X-20, X-23), attribution is supported by a recorded intervention, not by association alone |

**X-16 is deliberately non-surgical.** The correct outcome is `INCONCLUSIVE(NON_SURGICAL)`, not an
attribution. Both arms are scored on this, and a confident attribution is a failure for either.

---

## 3. What this registry cannot establish

| # | |
|---|---|
| R1 | **18 of 24 defects are E0/E1 — authored by us.** Only 6 are E2. The primary metric is computed on the E2 subset, which means the primary statistical test runs on **6 defects**, far below the N = 24 the power analysis requires. **This is the registry's central weakness.** Mitigations: commission more E2 defects (the cheapest single improvement to the whole gate), and report E0/E1 as strictly secondary. |
| R2 | E2 authors still work from an environment we built. Circularity is reduced, not removed. |
| R3 | Seeded defects are cleaner than real ones: one defect per package, well specified, present by construction. Real evaluations carry several partial defects at once. |
| R4 | No E3 defects exist (`OQ-18`). Every result is about defects someone deliberately planted. |

> **R1 is not a caveat, it is a design flaw we have not yet fixed.** With 6 E2 defects, the exact
> McNemar test cannot reach p < 0.05 even if A-INT wins **all** discordant pairs (d = 6, k = 6 gives
> p = 0.031 only when all six are discordant *and* all favour us — an implausible best case).
> **Commissioning 24 E2 defects, or accepting that the primary claim is underpowered and saying so,
> are the only honest options.** `KILL_CONTINUE_0_95.md` treats an underpowered primary as a
> non-result, not as a pass.
