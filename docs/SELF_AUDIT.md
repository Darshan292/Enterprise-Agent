# SELF-AUDIT

Status: **GATE 0.95 — DESIGN ONLY.** Version 0.1.0.

The assurance engine is evaluated against deliberately invalid **evaluation packages**. It is not
told which defect was injected, or whether one was.

---

## 1. Why this is not optional

The engine's output is a judgement about whether an evaluation is trustworthy. If the engine is
itself untrustworthy, every output is wrong in an unknown direction with no error signal — the
same structure as a silent grader bug, one level up.

`FACT`: this project has already committed **three** of the defects below in its own work:
- **wrong statistical estimator** — pooled `ΣV/ΣR` for FDR, Gate 0.9;
- **invalid identifiability assumption** — B-20 requiring an unearnable verdict, Gate 0.75;
- **underpowered primary test** — 6 E2 defects, caught in `METRICS.md` §3 at this gate.

Each survived at least one adversarial review pass. The self-audit therefore has real, documented
instances to test against, not hypothetical ones.

---

## 2. The injected defect catalogue

Injected into the **evaluation package the engine audits**, or into the **engine's own analysis
configuration**, depending on class.

| ID | Defect | Injected into | Correct engine behaviour |
|---|---|---|---|
| **SA-1** | **Wrong statistical estimator** — FDR computed as pooled `ΣV/ΣR` instead of mean per-simulation FDP | Engine's analysis config | Detect the estimator mismatch against a reference computation; report `INVALID_EVALUATION(statistical)` |
| **SA-2** | **Leaked fixture** — expected outputs readable from the agent's working directory | Package | Deterministic: leak scan hit ⇒ `INVALID_EVALUATION` |
| **SA-3** | **Weak oracle** — mutation score 0.55, escaping all partial-application mutants | Package | Mutation suite flags below the 0.70 floor; results depending on it **withheld** |
| **SA-4** | **Environment confound** — results differ by worker, manifest varies within a cell | Package | Hermeticity assertion fires before any statistics |
| **SA-5** | **State leakage** — trial N reads trial N−1's residue | Package | Canary hit ⇒ **whole run invalid** |
| **SA-6** | **Hidden factor** — mechanism recorded as an ambient observable but not declared | Package | `MODEL_GAP_DETECTED`, naming the observable |
| **SA-7** | **Decoy factor** — inert factor correlated 0.7 with the true cause | Package | Falsify by intervention; decoy **absent** from the attributed set |
| **SA-8** | **Benchmark contamination** — task fixtures overlap agent-visible context | Package | n-gram / canary scan ⇒ `INVALID_EVALUATION` |
| **SA-9** | **Invalid multiple-testing assumption** — BH applied to a family redefined *after* seeing results | Engine's analysis config | Family digest mismatch against the frozen manifest ⇒ `INVALID_EVALUATION(statistical)` |
| **SA-10** | **Underpowered primary test** — the package's own headline claim rests on n below its MDE | Package | Report that the claim is unsupported at its n; **do not** report the claim |
| **SA-11** | *(null)* No defect | — | No finding |
| **SA-12** | *(null)* Legitimately high indeterminate rate | Package | No finding; high IR is not a defect |

**12 classes × 5 replications = 60 blind audits.**

---

## 3. Blinding

| Control | |
|---|---|
| Defect identity | Sealed registry, digest-recorded, opened at scoring only |
| Defect presence | Nulls interleaved; the engine is never told how many packages are defective |
| Injection point | The engine does not know whether the defect is in the package or in its own config |
| Operator | The person running the self-audit is not the person who injected |
| Tuning | **The engine build is frozen and digest-recorded before the first package runs.** Any modification voids the run. |

---

## 4. Scoring

| Metric | Threshold |
|---|---|
| **Self-audit detection rate** over SA-1..SA-10 | ≥ 0.70, and **every deterministic class (SA-2, SA-4, SA-5, SA-8) at 1.00** |
| **Self-audit false-alarm rate** over SA-11, SA-12 (n = 10) | 0/10 → upper bound 0.28; ≤ 1/10 required |
| **Config-class detection** (SA-1, SA-9) | **≥ 1 of 2.** These are the classes where the engine must audit *itself*, and they are the ones it is structurally least able to see. |

`DESIGN_DECISION`: the four deterministic classes must be caught **every time**. They have exact
tests (canary hit, manifest variation, overlap scan, mutation floor). Missing one is an
implementation bug, not a power limitation, and is a build failure.

---

## 5. The recursion terminates, and it terminates in a person

**Who audits the self-audit?**

Nothing does. The regress ends at human review. Three consequences, stated rather than buried:

| # | |
|---|---|
| T1 | The self-audit's reach is bounded by **the catalogue we wrote**. A defect class absent from §2 is invisible, and the self-audit reads clean. The three real defects in §1 are in the catalogue *because we found them by hand first* — the catalogue is a record of our past mistakes, not a generator of future ones. |
| T2 | SA-1 and SA-9 inject into the engine's own configuration, so the engine must detect a defect in its own reasoning. **A systematic error shared between the engine and its reference computation is undetectable by construction.** The reference must be independently implemented, and "independently" is a claim about people, not code. |
| T3 | A passing self-audit means *"the catalogued defect classes are caught."* It does not mean the engine is sound, and no phrasing of the result may imply otherwise. |

> **The honest summary: the self-audit converts "we believe the engine works" into "the engine
> catches the ten mistakes we already know how to make." That is a real improvement and a small
> one, and the distance between those two sentences is the limit of what recursion buys.**
