# BASELINE-C SPECIFICATION

Status: **GATE 0.95.** Version 0.1.0.

**Purpose: specify the strongest competent composition of existing techniques, implementable by an
independent engineer without any part of our architecture.**

> **Standing instruction to whoever implements this: your job is to beat A-INT. You are not a
> control group. If any component below can be made stronger with an existing tool, make it
> stronger and record the change. A weak baseline makes the whole experiment worthless, and it is
> the failure mode we are most exposed to.**

Implemented and operated by an **advocate owner** — someone whose stated interest is that
BASELINE-C wins (`GATE_0_95_BASELINE_CHALLENGE` §6.1) — who is not on the A-INT team.

---

## 1. Tiers

| Tier | Contents | Role |
|---|---|---|
| **C0** | The six components below, run independently, six reports. No cross-referencing. | Named so it can be **rejected** as the comparison. Reported, never primary. |
| **C1** | C0 + a shared trial corpus + shared identifiers + an engineer's unified cross-reference pass | **PRIMARY comparison** |
| **C2** | C1 + the falsification protocol executed by hand | **DIAGNOSTIC** — decides whether our contribution is the protocol or the engine |

---

## 2. Components

Each: established tooling, its input, its output, and the strength floor it must meet.

### 2.1 Environment fingerprinting

| | |
|---|---|
| Existing basis | Standard system-introspection tooling; lockfile digests; container image digests; `uname`/locale/timezone capture; `/proc` sampling for resource pressure |
| Input | Per-trial execution context |
| Output | Per-trial `env_manifest_digest` + per-dimension digests over the 16 dimensions in `ENVIRONMENT_CONFOUNDER_MODEL.md` §1 |
| **Strength floor** | Must record **all 16** dimensions and emit a per-dimension diff across trials, not just a single composite digest. A composite-only fingerprint cannot localise, and would be an artificially weak baseline. |

### 2.2 Reproducibility / flaky-test analysis

| | |
|---|---|
| Existing basis | Standard flaky-test detection: repeated execution, pass/fail sequence analysis, quarantine heuristics; seed-variance analysis as used in ML/RL |
| Input | Repeated trials at fixed and varied seeds |
| Output | Per-cell pass rate, Wilson interval, within-cell variance, determinism verdict (fixed seed ⇒ identical trace digest) |
| **Strength floor** | Must distinguish **harness nondeterminism** (fixed seed, differing trace) from **agent stochasticity** (varying seed). A tool reporting only "flaky" is below floor. |

### 2.3 Mutation / oracle testing

| | |
|---|---|
| Existing basis | Mutation testing (PIT / mutmut / Stryker lineage), adapted to mutate the **effect log and trial record** rather than source |
| Input | Oracle implementations + trial records + ground-truth effect log |
| Output | Per-oracle mutation score over the catalogue in `ORACLE_INTEGRITY.md` §2.1; **list of escaped mutants** |
| **Strength floor** | ≥ 63 mutants per oracle (the power floor for distinguishing 0.90 from 0.70). Must include the **claim/effect-divergence** mutant — the single most valuable check. |

### 2.4 Contamination checks

| | |
|---|---|
| Existing basis | n-gram overlap between task fixtures and agent-visible context; canary-token planting; near-duplicate detection (MinHash/shingling) |
| Input | Task fixtures, agent context transcripts, oracle messages |
| Output | Overlap scores, canary hits, duplicate clusters |
| **Strength floor** | Both directions: fixture → context **and** oracle-message → context. Canary tokens must be planted in the ground-truth log **and** in grader messages. |

### 2.5 Grader integrity checks

| | |
|---|---|
| Existing basis | Standard test-suite hygiene: assertion-coverage review, order-dependence detection (test shuffling), fixture-staleness checks, negative controls |
| Input | Oracle code + fixtures + execution records |
| Output | Order-dependence verdict, staleness verdict, negative-control pass rate, degenerate-input behaviour |
| **Strength floor** | Must include **shuffled oracle execution order** and **negative controls** (impossible tasks). A negative-control pass is a deterministic leak proof and must be surfaced. |

### 2.6 Fault injection

| | |
|---|---|
| Existing basis | Established chaos/fault-injection tooling, per-trial schedulable |
| Input | The environment; a fault catalogue |
| Output | Per-trial fault assignment + outcome |
| **Strength floor** | Seeded and per-trial reproducible. Unreproducible injection makes everything downstream unusable and would be below any competent standard. |

### 2.7 Regression analysis / statistical testing

| | |
|---|---|
| Existing basis | Logistic regression with interaction terms; BH-FDR; Wilson intervals; McNemar; changepoint detection; standard libraries throughout |
| Input | The pooled trial corpus with all recorded covariates |
| Output | Main effects, interaction terms, p-values with multiple-testing correction, residual diagnostics |
| **Strength floor** | **Must include pairwise interaction terms**, not main effects only. A main-effects-only baseline cannot see cross-dimension defects **by construction**, and using one would rig the experiment. **This is the single most important strength floor in the document.** |

---

## 3. C1 — the integration pass

The step that makes the comparison fair.

```
C1 = C0 outputs
   + a SHARED TRIAL CORPUS: every component reads the same trials,
     joined on (package_id, trial_id), with all covariates from every component
     available to the regression in 2.7
   + a competent engineer's unified cross-reference pass
   + one report per package
```

**The engineer is told the goal** — find validity defects — and is given **unlimited freedom** in
method. No scripted procedure, no checklist from us. They may:
- join any component's output to any other's;
- subset trials by one component's finding and re-run another's analysis on the subset;
- add covariates from environment fingerprinting into the regression;
- request additional trials within budget B;
- write any ad-hoc analysis they think of.

> **`ASSUMPTION`, and the most consequential one in the gate:** a competent engineer with a shared
> corpus and interaction-term regression **will** connect *"oracle mutation score 0.95"* with
> *"environment manifest varied on filesystem ordering"* with *"pass rate differs between the two
> manifest groups"*. That is what competent engineers do. If we are right that cross-dimension
> defects still escape C1, the gate passes. **Our own expectation is that C1 catches most of
> them**, which is why the pre-registered prediction in `KILL_CONTINUE_0_95.md` is pessimistic.

---

## 4. C2 — the falsification protocol by hand

C1 plus: before attributing a defect to any factor, the engineer **intervenes** on it — sets it
independently, holds the fixture and seed family fixed, re-runs, and compares. Where intervention
is unavailable or non-surgical, they record *"cannot attribute"*.

**The protocol is not withheld from C2.** `IDENTIFIABILITY_AND_FALSIFICATION.md` is handed over in
full. Withholding it would test whether we told them the method, not whether automating it is
worth anything.

> **C2 exists to answer one question: is our contribution the protocol, or the engine?**
> If C2 matches or beats A-INT, the protocol is the contribution, and the correct deliverable is a
> specification plus a conformance suite — not a platform.

---

## 5. What BASELINE-C is NOT permitted

| Prohibition | Why |
|---|---|
| Access to the sealed defect registry | Same as A-INT |
| Knowledge of defect count or null-control identity | Same as A-INT |
| Tuning against packages already scored | Same as A-INT |
| Being implemented by an A-INT team member | CH-1 |

BASELINE-C **is** permitted to read every other document in this repository, including our
reasoning about cross-dimension defects. Withholding our thinking from the baseline would be
weakening it by another route.

---

## 6. Implementability check

An independent engineer should be able to build C1 from this document, standard tooling, and the
information package in `GATE_0_95_BASELINE_CHALLENGE` §4. Estimated effort:

| Component | Effort |
|---|---|
| Environment fingerprinting | 1–2 days |
| Reproducibility analysis | 1–2 days |
| Effect-log mutation testing | **3–5 days** (the catalogue is the work) |
| Contamination checks | 1–2 days |
| Grader integrity checks | 2–3 days |
| Fault injection harness | 2–3 days |
| Regression with interactions | 1–2 days |
| C1 integration pass | 2–4 days |
| **Total** | **≈ 3 weeks for one competent engineer** |

> **Three weeks is the number this project must beat.** If A-INT's advantage over C1 does not
> justify the difference between three weeks and several months, the honest recommendation is to
> ship BASELINE-C and stop — and that recommendation is available regardless of how the benchmark
> scores.
