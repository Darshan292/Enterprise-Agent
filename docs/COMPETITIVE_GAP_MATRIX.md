# COMPETITIVE GAP MATRIX

Status: **GATE 1.1.** Version 0.1.0.

**Epistemic status:** every external system below is `ASSERTED_UNVERIFIED`. MQ-EVA in particular
is relayed from the `OQ-24` result and is not verified by me against primary sources.

---

## 0. Classification rules — applied strictly

| Class | Definition |
|---|---|
| **A — Solved externally** | An existing system does this, in production or with a public methodology |
| **B — Partially solved** | Done well in an adjacent domain, or done incompletely here |
| **C — Adjacent** | A neighbouring system is one adapter away |
| **D — Genuinely missing** | Not done, **and** not straightforwardly constructible from mature methods |

**The rule that does the work:**

> **Something is NOT classified `D` merely because no single product combines it.** If it can be
> constructed straightforwardly by composing mature existing methods, it is classified
> **`COMPOSITION`**, which is not a thesis.
>
> The test: *could a competent engineer who knows the constituent methods build it in weeks rather
> than invent it?* If yes → `COMPOSITION`.

---

## 1. The matrix

### 1.1 Evaluation-validity capabilities

| # | Capability | MQ-EVA | NIST probes | AgentRx | AgentChaos | Eval frameworks | Sept-2026 survey | **Class** |
|---|---|---|---|---|---|---|---|---|
| 1 | Claim + evidence as the evaluated object | ● | | | | | ◐ | **A** |
| 2 | Evaluation-validity assessment | ● | ◐ | | | | ● | **A** |
| 3 | Construct validity | ● | | | | | ● | **A** |
| 4 | Evidence integrity | ● | ● | | | | | **A** |
| 5 | Statistical reliability | ● | | | | ◐ | ◐ | **A** |
| 6 | Contamination / leakage detection | ● | ● | | | ◐ | ● | **A** |
| 7 | Production / environment validity | ● | ◐ | | | | ◐ | **A** |
| 8 | Failure characterisation | ● | | ● | ◐ | | ◐ | **A** |
| 9 | Evaluation independence | ● | ◐ | | | | ◐ | **A** |
| 10 | Machine-readable claim/evidence artifact | ● | ● | | | | | **A** |
| 11 | **Unsupported vs Contradicted semantics** | **●** | | | | | | **A** — *finer than ours* |
| 12 | Reproducible evidence | ● | ● | | ● | ◐ | | **A** |
| 13 | No composite score | ● | | | | | ◐ | **A** |
| 14 | **Deterministic derivation of assurance results** | **●** | | | | | | **A** |

● full · ◐ partial

**Fourteen of fourteen: class A.** Including both properties `COMPETITIVE_2026` §4.2 had reduced
our entire claim to.

### 1.2 Mechanism capabilities

| # | Capability | Mature source | **Class** |
|---|---|---|---|
| 15 | Fault injection | AgentChaos-class, chaos engineering | **A** |
| 16 | Mutation testing | PIT / mutmut / Stryker lineage (1970s method) | **A** |
| 17 | Effect-log mutation (our adaptation) | mutation testing + a catalogue | **COMPOSITION** |
| 18 | Flaky-test / reproducibility analysis | industrial flaky-test infrastructure | **A** |
| 19 | Environment fingerprinting | hermetic build systems (Bazel, Nix) | **A** |
| 20 | Contamination scanning | n-gram overlap, canaries, membership inference | **A** |
| 21 | Trajectory diagnosis | AgentRx-class | **A** |
| 22 | Audit trails / probes | NIST | **A** |
| 23 | Combinatorial interaction testing | ACTS/PICT, 25 yr of CIT literature | **A** |
| 24 | Delta debugging / reduction | ddmin, C-Reduce, Perses | **A** |
| 25 | Stochastic ddmin (r-confirmation) | ddmin + repeated sampling | **COMPOSITION** |
| 26 | Failure fingerprinting / bucketing | ClusterFuzz, Sentry grouping | **A** |
| 27 | Multiple-testing correction | BH / BY / Westfall–Young | **A** |
| 28 | Metamorphic testing | established literature | **A** |
| 29 | Record / replay | rr, deterministic simulation (FDB, Antithesis) | **A** |
| 30 | Assurance cases with defeaters | **GSN / CAE — safety-critical, ~30 years old** | **A** |

### 1.3 Capabilities we claimed, re-classified

| # | Our claim | Honest class | Why |
|---|---|---|---|
| 31 | Agent-specific factor model | **COMPOSITION** | A domain factor list for an established CIT method. A config file in CIT terms. |
| 32 | Cross-dimension defect detection | **COMPOSITION** | Interaction testing (#23) with a domain-specific factor set |
| 33 | Stochastic regression artifact format | **COMPOSITION** | A schema plus a reproduction-rate field. Days of work. |
| 34 | Factor-model adequacy adjudication | **B / COMPOSITION** | Residual diagnostics (mature) against an ambient observable set. Construct-validity work (#3) covers the framing. |
| 35 | Refusal when identification impossible | **A** | Assurance-case defeaters (#30); MQ-EVA's Unsupported (#11); Pearl's identifiability. See `INTERVENTION_NOVELTY_REVIEW.md`. |
| 36 | Earned derivation of an assurance verdict | **A** | MQ-EVA #14 |
| 37 | Gateable machine-readable artifact | **A** | MQ-EVA #10; NIST #22 |
| 38 | Controlled intervention before attribution | **A** | See `INTERVENTION_NOVELTY_REVIEW.md` §2 — established in six of eight surveyed domains |
| 39 | C0/C1/C2 baseline tiering | **COMPOSITION** | Standard experiment design. Useful as a *practice*, not a product. |
| 40 | **Intervention reachability as a system property** | **D (thin)** | See §2 — the only D in the matrix, and it is not a product |

---

## 2. The single `D`, and why it does not rescue the project

**#40 — Intervention reachability.** Not *"perform the intervention"* (that is #38, class A), but:

> Given a system, determine **which of its variables can be surgically intervened on** — settable
> independently, without perturbing others — and expose that as a **first-class, computed property
> of the system**, prior to and independent of any specific investigation.

Every verdict our design could reach was determined by this property, and **nothing in the
surveyed landscape computes it.** Systems assume intervention is available (chaos, A/B, DOE) or
assume it is not (observational causal inference). The *boundary* is treated as background.

### 2.1 Why it is thin

| Objection | Force |
|---|---|
| For most systems the answer is "read the config schema" | **Strong.** The interesting cases — where settability is emergent, entangled, or environment-dependent — may be a small minority. |
| Surgicality is only checkable against *recorded* variables | **Strong.** An intervention perturbing something unrecorded passes the check and is invalid. The property is relative to instrumentation, so it inherits every limit of §1.3 #34. |
| Who buys it? | **Decisive.** It is a *property*, not a workflow. No trigger event, no owner, no budget line. The same K-7 that went unanswered for three gates. |
| Constructible? | Partly. A static pass over configuration surfaces plus an empirical perturbation probe is weeks, not years. Under the §0 rule that is close to `COMPOSITION`. |

### 2.2 Verdict on #40

**A genuine conceptual gap, classified `D` honestly, and not a product.** It is recorded in
`CANDIDATE_PROBLEM_SPACE.md` as candidate P-1 and rejected there on the grounds above.

---

## 3. Tally

| Class | Count | Share |
|---|---|---|
| **A — solved externally** | 26 | **65 %** |
| **B — partially solved** | 1 | 2.5 % |
| **C — adjacent** | 0 | 0 % |
| **COMPOSITION** | 12 | 30 % |
| **D — genuinely missing** | **1 (thin, not a product)** | 2.5 % |

> **Of forty capabilities examined, one is genuinely missing, it is thin, and it is not a
> product.** Thirty percent are compositions — which under this project's own stated rule are
> explicitly not theses.

---

## 4. The finding that should have arrived five gates earlier

Every gate narrowed the claim, and every narrowing was correct. The sequence:

```
Gate 0    runtime assurance / unknown side effects   -> solved elsewhere
Gate 0.5  interaction-failure discovery              -> covering arrays worth 1.7x over random
Gate 0.75 factor-model adequacy                      -> residual diagnostics, established
Gate 0.9  evaluation-integrity auditing              -> five of five dimensions have mature analogues
Gate 0.95 cross-dimension defects vs composition     -> C1 is three weeks of work
Gate 1.0  automation of an existing capability       -> a cost point, contingent on frequency
Gate 1.1  the cost point                             -> MQ-EVA occupies it
```

**Each step was a legitimate narrowing, and the trajectory was monotonic.** A claim that narrows
at every gate and never widens is a claim being progressively falsified, and **the shape of the
sequence was itself evidence long before the endpoint.**

The decision rule this suggests, for any future project: **if a thesis narrows at three
consecutive adversarial gates without a single widening, stop at the third and ask whether
anything is left — rather than running the sequence to exhaustion.** We ran it to exhaustion. The
artifacts are better for it; the calendar is not.
