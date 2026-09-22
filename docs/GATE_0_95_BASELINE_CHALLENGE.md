# GATE 0.95 — BASELINE CHALLENGE

Status: **GATE 0.95 — PROTOCOL DESIGN. NOT FROZEN. BENCHMARK NOT RUN.**
Version 0.1.0 · 2026-09-22

**Converged thesis under test (Agent Evaluation Assurance):**
> An agent evaluation can produce a plausible score while being invalid because of interactions
> among environment, oracle, agent behaviour, state, contamination, stochasticity and evaluation
> design. A useful assurance system should detect these cross-dimension validity defects, use
> controlled intervention where attribution is possible, refuse unsupported conclusions, and
> produce an auditable assurance report.

**No novelty is claimed for any mechanism** — mutation testing, fuzzing, fault injection, model
adequacy, DOE, metamorphic testing, contamination detection, grader auditing, statistical testing,
replay, reduction. They are mechanisms. **The only question is whether integrating them around
agent-evaluation validity produces measurable value beyond a competent composition.**

---

## 0. This gate designs the experiment that judges us

Every degree of freedom we hold is a channel through which we can win unfairly. Naming them and
closing them structurally is the work; promising to be fair is not a control.

| # | Cheating channel | Structural closure |
|---|---|---|
| CH-1 | **Weak baseline** | Three tiers (§2); the primary comparison is against the *strong* tier; adversarial baseline ownership (§6) |
| CH-2 | **Defect authorship bias** | External-authorship tiers E0–E3 (§5); primary metric computed on **E2+ only** |
| CH-3 | **Budget definition** | Three bases reported; equal-wall-clock is primary and **penalises us** (§3) |
| CH-4 | **Information asymmetry** | BASELINE-C receives the *identical* information package (§4) |
| CH-5 | **Metric selection** | Metrics frozen and digest-recorded before the baseline is built (`METRICS.md`) |
| CH-6 | **Analysis flexibility** | Test, unit of analysis and n frozen before the run (§7) |
| CH-7 | **Integration tax placed on the baseline** | §2 — this is the subtlest and most important one |

### 0.1 CH-7 — the channel that would invalidate the whole gate

Our claim is *"integration creates value beyond composition."* If BASELINE-C is defined as five
tools emitting five separate reports that nobody cross-references, then we are testing *"integrated
versus not integrated"* — which we win by definition and which proves nothing about whether the
integration is **hard**.

If BASELINE-C is defined as five tools **plus a competent engineer who cross-references them**,
the comparison is honest and we may well lose.

**Both must be run, as separate tiers, and the primary comparison is against the harder one.**

---

## 1. The exact comparison

| Arm | Definition |
|---|---|
| **A-INT** | Our evaluation-assurance approach: integrated signal collection, cross-dimension analysis, the falsification protocol (`IDENTIFIABILITY_AND_FALSIFICATION.md`), verdict semantics with mandatory refusal (`VERDICT_SEMANTICS.md`), auditable assurance report. |
| **C0** | Five mature tools, run independently, five separate reports. **No cross-referencing.** |
| **C1** | C0 **plus** a competent engineer's unified pass: shared trial corpus, shared identifiers, hand-written cross-reference of the five outputs, single report. |
| **C2** | C1 **plus** the falsification protocol executed **by hand**: the engineer intervenes on candidates before attributing, and records `INCONCLUSIVE` where intervention is unavailable. |

**Primary comparison: A-INT vs C1.**
**Diagnostic comparison: A-INT vs C2.**

### 1.1 What each result means — the reframe that makes this gate worth running

| Outcome | Interpretation | Decision |
|---|---|---|
| A-INT ≈ C0, loses to C1 | Integration adds nothing a shared corpus does not | **KILL** |
| A-INT > C1, > C2 | Integration *and* automation both add value | **CONTINUE**, MVDC-scoped |
| A-INT > C1, **≈ or < C2** | **The falsification protocol is the contribution; the engine is not.** A careful human running the protocol matches or beats our automation. | **PIVOT — ship the protocol and a conformance suite, not an engine** |
| A-INT ≈ C1 | Composition wins | **KILL** |

> **C2 losing to us is not the only good outcome, and C2 beating us is not simply a failure.**
> It is a finding about *what to build*: if a human executing the protocol matches the engine,
> then the deliverable is a **specification plus a conformance test suite** that any evaluation
> harness can run against itself — far cheaper, far more likely to be adopted, and it sidesteps
> the maintenance question entirely. This is the single most useful thing this experiment can
> tell us, and the previous gates could not have discovered it.

---

## 2. Baseline strength rules

1. **Every BASELINE-C component uses an existing, established implementation or a faithful
   implementation of its published method.** Nothing is reimplemented worse.
2. **C1's integration pass is performed by an engineer who is told the goal** — find validity
   defects — and is given unlimited freedom in how to cross-reference. No scripted procedure.
3. **C2's engineer may run any intervention they think of.** Interventions are not reserved to us.
4. **If a BASELINE-C component has a known stronger variant, the stronger variant is used.**
5. **The baseline engineer may read every document in this repository except the defect registry.**
   Withholding our reasoning from the baseline would be CH-1 by another route.

---

## 3. Equal-budget rules

Three bases. **All three are reported.** Primary is declared before the run.

| Basis | Definition | Who it favours |
|---|---|---|
| **Equal wall-clock** *(PRIMARY)* | Same total compute minutes per package | **Penalises A-INT** — controlled interventions cost runs that detection could have used |
| Equal trials | Same number of executions | Neutral-to-favourable for A-INT |
| **Equal engineer-hours** | Same human time, including setup | **Strongly favours BASELINE-C** — it is off-the-shelf; we are not |

`DESIGN_DECISION`: **equal wall-clock is primary because it is what an operator actually pays, and
because it is the basis that costs us most.** Equal engineer-hours is reported because it is the
honest economic comparison and because omitting it would be convenient.

Budget per package: **B = 2,000 deterministic trial-equivalents** (≈ 67 min at 2 s). Both arms
may spend it however they choose. A-INT's interventions come out of B; so do C2's.

---

## 4. Equal-information rules

> **BASELINE-C receives the identical information package.** Otherwise the experiment measures
> *"did we tell the baseline what to look at"*, not *"is integration valuable"*.

The package, handed to both arms:

| Item | Contents |
|---|---|
| Evaluation package under audit | Task fixtures, agent, oracle code, trial records, effect log, environment manifests per trial, reported score |
| **Factor model** | The declared factor list and levels |
| **Ambient observable set** | All 14–16 recorded observables, with definitions |
| **Invariant list** | INV-1..INV-9 |
| **Ground-truth effect log access** | Read access, for both arms |
| Environment control | The ability to set any settable dimension — **for both arms** |
| Intervention capability | Both arms may run controlled interventions within budget B |
| The five tools of BASELINE-C | **Available to A-INT as well.** A-INT is not forbidden from using mutation testing. |

**Withheld from both:** the defect registry, the defect class, the number of defects in a package,
and whether a package is a null control.

---

## 5. Permitted tools, and prohibited knowledge leakage

### 5.1 Permitted
Any established implementation of: environment fingerprinting · reproducibility/flaky-test
analysis · mutation testing · contamination detection · grader integrity checks · fault injection ·
regression analysis · repeated execution · standard statistical testing. Both arms, freely.

### 5.2 Prohibited leakage

| Channel | Control |
|---|---|
| Defect registry | Sealed, digest-recorded, opened only at scoring |
| Package labelling | Packages are shuffled and anonymised; null controls are indistinguishable from defective ones |
| Defect count | Neither arm is told how many defects a package contains, or whether it contains any |
| Author identity | Neither arm is told who authored a defect (E-tier is revealed only at scoring) |
| Tuning on the benchmark | **A-INT may not be modified after any package has been run.** Frozen build, digest-recorded. |
| Baseline handicapping | The baseline engineer is **not** a member of the A-INT team (§6) |
| Post-hoc metric choice | Metrics frozen before the baseline is built |

### 5.3 The leakage we cannot close
We authored the environment, the factor model, the ambient set and most defects. A-INT was
designed with all of that in view. **BASELINE-C was not designed at all — it is assembled from
general-purpose tools.** This is a structural advantage to A-INT that no protocol removes, and it
is why the primary metric is restricted to externally-authored defects (§6.2).

---

## 6. External authorship tiers

| Tier | Definition | Bias |
|---|---|---|
| **E0** | Authored by the A-INT team | **Maximal** |
| **E1** | Authored by a team member who has not read the factor model or ambient set | High |
| **E2** | **Authored by someone outside the project**, given only the environment API and the instruction *"make this evaluation invalid in a way that is hard to see"* | Moderate |
| **E3** | Harvested from a real evaluation incident nobody here authored | Minimal — **none available** (`OQ-18`) |

### 6.1 Adversarial baseline ownership
`DESIGN_DECISION`: **BASELINE-C is implemented and operated by a person whose stated interest is
that BASELINE-C wins.** Not a neutral party — an advocate. A neutral implementer under-invests;
an advocate finds the cross-references a tired engineer would miss. This is the only reliable
defence against CH-1, and it costs one person's time.

### 6.2 The primary metric is computed on E2 defects only
E0 and E1 results are reported separately and **cannot support a CONTINUE**. An advantage visible
only on team-authored defects is an advantage over our own imagination.

---

## 7. Evaluation protocol

```
 1. FREEZE   metrics (METRICS.md), defect registry, budget basis, statistical test,
             unit of analysis, and the A-INT build. Record digests. THEN build BASELINE-C.
 2. BUILD    BASELINE-C to §2 rules, by the advocate owner (§6.1).
 3. SHUFFLE  packages anonymised, order randomised, null controls interleaved.
 4. RUN      both arms, equal wall-clock budget B per package, blind to ground truth.
 5. RECORD   each arm's verdict per package: defect detected? which dimensions named?
             verdict class? interventions run? time to first finding?
 6. SCORE    once, against the sealed registry. No re-scoring.
 7. REPORT   all three budget bases; E0/E1 and E2 separately; every pre-registered metric,
             including those that went against us.
```

**Nothing is run before step 1 completes.** This document is not frozen yet.

---

## 8. Statistical comparison — and the finding that resizes the benchmark

### 8.1 The unit of analysis is the DEFECT, not the replication

`FACT`: replications of the same defect are **clustered**. They reduce measurement noise for that
defect; they add nothing to the population claim *"A-INT detects cross-dimension defects better
than BASELINE-C."*

Design effect `DE = 1 + (m−1)ρ` for m replications at intra-cluster correlation ρ:

| ρ | m = 5 | m = 10 |
|---|---|---|
| 0.5 | 40 of 120 effective | 22 of 120 |
| 0.8 | 29 of 120 | 15 of 120 |
| 0.95 | 25 of 120 | **13 of 120** |

> **12 defects × 10 replications is not 120 independent observations. It is closer to 13.**

### 8.2 Required number of distinct cross-dimension defects

Exact McNemar on paired defect-level outcomes. `d` = discordant defects, `k` = favouring A-INT:

| N defects | discordance δ | fraction f favouring A-INT | d | k | exact p |
|---|---|---|---|---|---|
| **12** | 0.50 | **1.00** (perfect) | 6 | 6 | **0.031 SIG** |
| **12** | 0.50 | 0.80 | 6 | 5 | 0.219 — not significant |
| 12 | 0.35 | 1.00 | 4 | 4 | 0.125 — not significant |
| 16 | 0.35 | 1.00 | 6 | 6 | 0.031 SIG |
| 20 | 0.50 | 0.80 | 10 | 8 | 0.109 — not significant |
| **24** | 0.50 | **0.80** | 12 | 10 | **0.039 SIG** |
| 24 | 0.35 | 1.00 | 8 | 8 | 0.008 SIG |
| 32 | 0.50 | 0.80 | 16 | 13 | 0.021 SIG |

> **`FACT`: 12 cross-dimension defects reaches significance only if *every* discordant defect
> favours A-INT. Any dissent at all and it fails. 12 is arithmetically inadequate for a
> realistic result.**
>
> **N = 24 is the minimum that tolerates a realistic 80 % win rate among discordant defects.**
> The brief's floor of 12 is therefore raised to **24 cross-dimension defects**, with **5
> replications** rather than 10 — replications buy per-defect precision, not population power,
> and the budget is better spent on more defects.

### 8.3 Frozen analysis plan
- **Primary test:** exact McNemar (sign test) on paired defect-level detection, A-INT vs C1,
  **restricted to E2 defects**, α = 0.05, two-sided.
- **Unit:** the defect. Replications summarised to a per-defect majority outcome before testing.
- **Secondary:** A-INT vs C2, same test, reported as a diagnostic, not as the headline.
- **FAR and Indeterminate rate:** two-proportion with Wilson intervals; n = 112 per arm detects a
  0.15 difference around 0.20, n = 200 detects 0.10 around 0.15.
- **No composite score.** Ever.
- Any deviation from this plan is reported as a deviation alongside the result.
