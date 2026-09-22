# GATE 0.9 — RESEARCH VALIDITY: EVALUATION INTEGRITY

Status: **GATE 0.9 — EVALUATION INTEGRITY. IMPLEMENTATION NOT AUTHORIZED.**
Version 0.1.0 · 2026-09-22 · supersedes the Gate 0.75 framing

**Standing rule for this document:** *"nobody has built this"* is **invalid reasoning** and is not
used anywhere below. Absence of an identical product is evidence about markets, not about
importance. The project survives only if the capability is technically nontrivial, empirically
measurable, useful to a real workflow, **hard to reproduce by composing existing tools**, and
worth maintaining after the demo.

---

## 0. The epistemic flaw that forced this gate

Gate 0.75 benchmark case **B-20** specified: an unobserved mechanism, a visible decoy correlated
with it at 0.7, and required verdict `UNKNOWN_MODEL`. Blaming the decoy was scored as
"catastrophic misattribution" and made a hard KILL criterion.

**That requirement is not satisfiable, and the fault is ours.**

`FACT` (identifiability): let `H` be unobserved and `D` observed, with `corr(H, D) = 0.7`. From
observational data alone the two worlds

- `D → failure`, and
- `H → failure`, `H → D` (or a common parent),

produce **identical joint distributions over everything we can see**. No amount of residual
analysis distinguishes them. The Gate 0.75 method would correctly report `D` as a promotion
candidate — the behaviour the benchmark called catastrophic. **We wrote a benchmark that punished
our method for being correct, and made it the kill gate.**

The fix is not a better estimator. It is a different kind of evidence: **intervene on `D`.**
`IDENTIFIABILITY_AND_FALSIFICATION.md` specifies the protocol, and the consequence is structural —
`INSUFFICIENT_OBSERVABILITY` becomes reachable **only after a falsification attempt**, never from
observation.

> This error is worth recording rather than quietly patching. A project whose stated purpose is
> detecting invalid evaluations shipped an invalid evaluation into its own kill criterion, and it
> survived three adversarial review passes. That is the single best piece of evidence that the
> defect class is real — and it is also evidence that the defect is caught by careful reading, not
> only by tooling.

---

## A. Exact falsifiable thesis

### A.1 The thesis as given

> "An agent evaluation integrity engine that determines whether an AI-agent evaluation is actually
> measuring the intended capability, by detecting hidden environmental factors, confounding, state
> leakage, oracle weakness, evaluation cheating paths, unsupported factor assumptions, and
> insufficient observability, and by using controlled interventions to falsify plausible
> explanations."

### A.2 Why that is not yet testable

Every listed detection has a mature, named, direct analogue (§B). Stated as above, the thesis is
satisfied by **running five existing tools and putting the output in one report**. That is not a
thesis; it is an integration ticket.

### A.3 T** — the falsifiable form

> **T\*\*:** There exists a class **D** of evaluation-integrity defects in stochastic, stateful,
> tool-using agent evaluations such that:
>
> **(a) Consequence.** Each causes the evaluation to measure something other than the intended
> capability, with a measurable effect on the reported result.
>
> **(b) Specifiable.** Each is detectable by a deterministic test, a statistical test, or a
> controlled intervention that we can write down in advance.
>
> **(c) COMPOSITION-RESISTANT.** **A naive composition of existing single-purpose tools —
> mutation testing + flaky-test analysis + contamination scanning + confounder regression, run
> independently and unioned — misses a materially large share of D.**
>
> **(d) Falsification-disciplined.** For defects that are not observationally identifiable, the
> engine performs a controlled intervention and returns `INCONCLUSIVE` when intervention is
> unavailable — rather than attributing.
>
> **(e) Externally real.** D occurs in evaluations **we did not author**.

**(c) is the whole thesis.** (a), (b), (d) and (e) are necessary and none of them is sufficient,
because all four are satisfiable by the composed baseline. If (c) fails, the correct output of
this project is a short paper describing the composition, not a platform.

---

## B. Existing systems that overlap

`ASSERTED_UNVERIFIED` for every external capability below, including the four supplied in the
brief. `OQ-02` requires a documentation check before any build.

### B.1 Direct analogue per dimension — the uncomfortable table

| Our dimension | Mature direct analogue | Maturity | What is genuinely different for agents |
|---|---|---|---|
| **ORACLE_VALIDITY** | **Mutation testing** (since the 1970s; PIT, mutmut, Stryker). "Inject defects, check whether the tests catch them" *is* oracle-validity measurement. | **Very high** | We cannot mutate the system under test — a stochastic policy has no meaningful source-level mutants. We mutate the **effect log / trajectory record** instead. A real adaptation; a small one. |
| **REPRODUCIBILITY_VALIDITY** | **Flaky-test detection and quarantine infrastructure** at industrial scale; seed-variance analysis in ML/RL. | **Very high** | Agent flakiness is intended behaviour, not a defect, so the question shifts from "is this flaky?" to "is the flakiness rate itself stable?". A reframing, not a new method. |
| **ANTI-CHEATING** | **Contamination detection** (n-gram overlap, canary strings, membership inference) and the **reward-hacking / specification-gaming** literature. NIST has documented solution contamination and grader gaming (per brief). | **High** | Trajectory-level cheating paths (reaching the graded state by an unintended route) are less covered than text-level contamination. The narrowest real gap. |
| **ENVIRONMENT_VALIDITY** | **Hermetic build/test systems** (Bazel, Nix) whose entire purpose is eliminating environmental confounders; confounder detection in statistics. Microsoft has documented hidden environmental variables changing agent eval results (per brief). | **Very high** | Agent evals cannot be fully hermetic — they need a mutable world. So hermeticity becomes a *measurement* rather than a guarantee. |
| **FACTOR_SPACE_VALIDITY** | Residual diagnostics; DOE; **construct-validity** literature in ML measurement ("the benchmark does not measure what it claims"). | **High** | Nothing methodologically. The application is ours. |

**Five of five dimensions have mature direct analogues.** Gate 0.75 estimated four of five; the
correct number is worse.

### B.2 Adjacent agent-specific systems

| System | What it does | Relation |
|---|---|---|
| **Microsoft Research AgentRx** (per brief) | Automated trajectory diagnosis | **Diagnoses the agent.** We audit the evaluation. Genuinely different object — and a competitor one feature away from overlapping. |
| **AgentChaos-like systems** (per brief) | Agent fault injection | Supplies the intervention mechanism we need. **Consume, do not rebuild.** |
| **NIST evaluation-cheating documentation** (per brief) | Names contamination and grader gaming | Institutional recognition of the problem. Raises legitimacy, lowers novelty-of-problem. |
| Agent eval harnesses (Inspect, promptfoo, DeepEval, LangSmith, Braintrust) | Run evals | The **host** for an integrity audit. Any of them could add one. |

### B.3 The composed baseline, named precisely

This is the thing to beat, and it is not hypothetical:

```
BASELINE-C =  effect-level mutation testing        (oracle strength)
           ∪  repeated-seed variance analysis      (reproducibility)
           ∪  canary/contamination scanning        (cheating)
           ∪  environment-manifest diffing         (confounders)
           ∪  logistic residual regression         (factor gaps)
```
Five existing techniques, run independently, results unioned. An engineer who knows all five
assembles this in **one to three weeks**. Under the project's own rule 4, **T\*\*(c) is the only
thing standing between this project and that fortnight.**

---

## C. What remains genuinely differentiated

Exactly two candidates. Both are narrow. Neither is a method.

### C.1 — Cross-dimension integrity defects (the only load-bearing candidate)

A defect that **no single dimension's tool detects, because it lives in the interaction between
dimensions.** Worked examples, each realisable in the benchmark:

| Cross-dimension defect | Why each single tool passes it |
|---|---|
| An oracle with a **high mutation score** that is nonetheless bypassed **only under a specific environment confound** (e.g. a filesystem-ordering difference makes an unintended path reachable). | Mutation testing runs in the default environment and reports a strong oracle. Environment diffing sees a benign variation. Neither looks at the pair. |
| **Seed variance within tolerance** and **contamination scan clean**, but the eval's pass rate is driven by a fixture-lineage artifact that correlates with the task split. | Variance analysis sees acceptable flakiness. Contamination sees no overlap. The confound is in the *assignment*, not in either signal. |
| A **cheating path that only exists when a prior trial leaked state** — the agent reads a residue left by trial N−1 that encodes the answer. | Contamination scanning checks task text, not runtime state. Isolation checks look for canaries, not for answer-bearing residue. |

This is the Gate 0.5 interaction thesis **applied recursively to the audit itself**. It is the
only construction that can satisfy T\*\*(c), and it is directly measurable: run BASELINE-C and the
integrated audit on the same seeded defects and count what each misses.

### C.2 — The falsification discipline

Mandatory controlled intervention before any attribution, plus `INCONCLUSIVE` when intervention
is unavailable (`IDENTIFIABILITY_AND_FALSIFICATION.md`).

**Honest assessment:** this is **methodology, not software**. It is a protocol and a verdict
vocabulary. It is easy to copy and hard to sell. Its value is that it prevents the exact error
this project made in B-20 — which is real, and is worth roughly a checklist.

### C.3 — What is NOT differentiated

Mutation testing · flaky-test detection · contamination scanning · confounder regression ·
DOE · metamorphic testing · fault injection · causal inference · regression testing · residual
diagnostics · model adequacy. **All established. All cited as such. None claimed.**

---

## D. What this project cannot claim

| # | Cannot claim | Why |
|---|---|---|
| D1 | Novelty of any constituent method | §B.1, §C.3 |
| D2 | That an audit pass means the evaluation is valid | Only that the **tested** integrity dimensions showed no defect at the stated power. Validity is not provable, only falsifiable. |
| D3 | Completeness of the integrity dimension set | Same structure as factor-model incompleteness: detectable, never provable. |
| D4 | That hidden mechanisms are identifiable from observation | §0. Identification requires intervention. |
| D5 | Production or regulatory relevance | No real evaluations, no real agents, no external data. |
| D6 | Importance because no identical product exists | **Explicitly banned reasoning.** |
| D7 | That the seeded benchmark defects resemble real ones | We author them. `OQ-18` remains the strongest available upgrade to the evidence base. |

---

## E. Minimum viable differentiated capability (MVDC)

> **Detect evaluation-integrity defects that arise from interactions *between* integrity
> dimensions, which BASELINE-C misses when its five components are run independently — and, for
> any candidate explanation that is not observationally identifiable, falsify it by controlled
> intervention or return `INCONCLUSIVE`.**

MVDC is **one capability**, not five. Everything else in the system exists to feed it:
the dimension model supplies the signals, the benchmark supplies the ground truth, the
verdict vocabulary supplies the honest output, the statistics supply the error control.

**MVDC is measurable in one experiment** (`CONTINUE_OR_KILL_V2` K-A) and that experiment can
return a number that ends the project.

---

## F. Conditions under which the project should be killed

Full thresholds, sample sizes and power in `CONTINUE_OR_KILL_V2.md`. The conditions:

| # | Kill condition | Rationale |
|---|---|---|
| **F1** | **BASELINE-C detects ≥ 0.80 of what the integrated audit detects**, over the seeded defect set at N = 240 paired observations. | T\*\*(c) fails. The project is a fortnight of glue and should be written up as such. |
| **F2** | The engine **attributes** a non-identifiable mechanism to an observable **without** a successful falsification, at a rate whose upper bound exceeds 0.20. | It reproduces the B-20 error it exists to prevent. |
| **F3** | Cross-dimension defects prove **not constructible** — every seeded "interaction" defect turns out to be detectable by one dimension alone. | C.1 was imagination. Nothing is left. |
| **F4** | No local stochastic agent is available (`OQ-16`), so no claim about agent evaluations can be supported at all. | Risk #2, unmitigable. |
| **F5** | **No identified owner for the workflow after 60 days of looking.** | Not a technical objection. The most likely actual cause of death, and the one this project has deferred three times. |

`F5` is stated as a kill condition deliberately. It has appeared as a footnote in every prior
gate. Promoting it to a kill condition is the only way it gets tested rather than noted.
