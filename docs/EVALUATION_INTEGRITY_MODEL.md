# EVALUATION INTEGRITY MODEL

Status: **GATE 0.9.** Version 0.1.0.

Five dimensions. For each: observed inputs, possible violations, deterministic tests, statistical
tests, controlled interventions, known limitations.

**Standing caveat, applied to all five:** every dimension has a mature direct analogue
(`GATE_0_9_RESEARCH_VALIDITY` §B.1). Nothing here is claimed as a new method. The claim under
test is that **defects spanning two dimensions escape all five analogues run independently**
(§6).

---

## 1. ENVIRONMENT_VALIDITY

*Is the result a property of the agent, or of the machine it ran on?*

| | |
|---|---|
| **Observed inputs** | Environment manifest digest per trial (see `ENVIRONMENT_CONFOUNDER_MODEL.md`); worker id; fixture lineage; resource-pressure samples; trial index; wall-clock; prior-trial residue digest. |
| **Possible violations** | Results vary with an environment dimension that is not part of the intended capability · environment drifts mid-run · workers differ · the fixture restore path leaves different states · resource pressure changes timing enough to change behaviour. |
| **Deterministic tests** | **Hermeticity assertion**: the environment-manifest digest must be **constant** across all trials in a cell; any variation is a defect, not a nuisance. Fixture post-restore digest equals declared baseline. Canary scan for prior-trial residue. |
| **Statistical tests** | Residual association against each ambient environment observable, corrected per `STATISTICAL_VALIDITY.md`. Changepoint test on the residual series ordered by trial index. |
| **Controlled interventions** | Deliberately **vary** a suspected dimension (worker assignment, tool version, resource pressure) as a randomised treatment and measure the effect. This is the only route from "associates" to "affects". |
| **Known limitations** | A dimension that is **constant** across the whole run cannot be detected as a confounder, and is also the most dangerous kind. Hermeticity is a measurement, not a guarantee — agent evals need a mutable world. Surgicality is limited: changing a tool version may change several things at once. |

---

## 2. FACTOR_SPACE_VALIDITY

*Does the declared model of what can vary actually cover what varies?*

| | |
|---|---|
| **Observed inputs** | Declared factor levels; the ambient observable set; per-cell outcome variance; the reproducibility floor measured on null cases. |
| **Possible violations** | A mechanism outside the declared factors drives results · declared factors are aliased and unidentifiable · the tested space is far narrower than the claimed space · constraints have silently shrunk the denominator. |
| **Deterministic tests** | Determinism screen (identical assignment + seed + fixture ⇒ identical trace digest); every constraint carries a rationale; coverage report states the infeasible fraction and the constraints responsible. |
| **Statistical tests** | Declared-factor fit (deviance explained); residual against the ambient ring; VIF and design-matrix condition number for aliasing. |
| **Controlled interventions** | Promote a candidate ambient observable to a declared factor and run a designed factorial. Falsify candidates per `IDENTIFIABILITY_AND_FALSIFICATION.md`. |
| **Known limitations** | **Adequacy is not provable, only inadequacy is detectable.** A hidden factor constant across all trials is invisible. A hidden factor perfectly aliased with a declared one is undetectable in principle without an intervention that breaks the alias. The ambient ring is authored by us. |

---

## 3. ORACLE_VALIDITY

*Would the grader actually notice if the agent were wrong?*

| | |
|---|---|
| **Observed inputs** | Oracle verdicts with versions; the environment ground-truth effect log; the agent's claimed outputs; oracle source and fixtures. |
| **Possible violations** | Grader gaming · hard-coded answers · assertion manipulation · test bypass · **checking outputs instead of intended effects** · partial success scored as full success · stale expected values · shared evaluator state · leakage between trials. Catalogue and per-defect checks in `ORACLE_INTEGRITY.md`. |
| **Deterministic tests** | **Effect-level mutation testing**: apply a catalogue of mutations to the effect log / trial record and assert each oracle flips to `VIOLATED`. **Mutation score is the primary oracle metric.** Plus: oracle reads the ground-truth log, not the agent's claim (capability check); degenerate input yields an explicit verdict, never a default `HOLDS`; expected values carry an expiry. |
| **Statistical tests** | Oracle firing-rate distribution across the run (a zero-firing oracle is flagged, not celebrated); inter-oracle disagreement rate; association between oracle verdict and trial index (drift). |
| **Controlled interventions** | Inject a **known-bad** trajectory that should fail and assert the oracle catches it; inject a **known-good** one and assert it passes; run an agent deliberately down an unintended-but-graded path. |
| **Known limitations** | **Mutation testing is a mature technique and the analogue is exact.** Mutation score is bounded by the mutation catalogue, which we author — a defect class absent from the catalogue is invisible. A high mutation score in the default environment says nothing about the oracle under an environment confound (§6). |

---

## 4. REPRODUCIBILITY_VALIDITY

*Is the reported number stable enough to mean anything?*

| | |
|---|---|
| **Observed inputs** | Repeated trials at fixed assignment and seed; repeated trials across seeds; trace digests; per-cell outcome series. |
| **Possible violations** | Harness nondeterminism mistaken for agent stochasticity · unreported variance · a single-run result reported as a rate · a rate reported without an interval · variance itself drifting across the run. |
| **Deterministic tests** | Fixed seed ⇒ identical trace digest (harness determinism). Fixture digest assertion pre-trial. Seed derivation is recorded and reproducible. |
| **Statistical tests** | Within-cell variance vs the measured floor; Wilson intervals on every rate; **minimum detectable effect printed before the run**; variance-stability test across the run. |
| **Controlled interventions** | Re-run the whole evaluation from the recorded manifest on a clean checkout and compare the distributions. |
| **Known limitations** | The analogue — flaky-test detection — is industrially mature. For agents, stochasticity is **intended**, so the question is whether the *rate* is stable, which needs far more trials than flaky-test detection normally uses. At n = 20 the Wilson half-width near p = 0.5 is ± 0.20; most agent evals report rates from fewer runs than that. |

---

## 5. ANTI-CHEATING / EVALUATION-GAMING VALIDITY

*Did the agent pass by doing the task, or by finding a way around it?*

| | |
|---|---|
| **Observed inputs** | Task fixtures; agent-visible context; the full trajectory; effect log; oracle internals reachability; prior-trial residue. |
| **Possible violations** | Solution contamination (the answer is in the context or the model's training) · grader gaming (optimising the metric, not the task) · unintended-path success · reading the ground-truth log · answers left by a prior trial · exploiting an oracle's partial-credit rule. |
| **Deterministic tests** | **Canary tokens** planted in the ground-truth log and grader messages — appearance in agent context is proof of a leak. **Negative controls**: tasks impossible to complete correctly; **a pass is proof of a leak, full stop**. Permission test: the agent credential must be **denied** on the ground-truth log. Contamination scan: n-gram overlap between task fixtures and agent-visible context. |
| **Statistical tests** | Pass rate on negative controls (must be at the false-positive floor); distribution of trajectory shapes among passing trials — a passing cluster with an anomalous shape is a candidate unintended path. |
| **Controlled interventions** | **Remove the suspected cheating path and re-run.** If the pass rate collapses, the path was load-bearing. This is the cleanest intervention in the whole model, because the path is something we can delete. |
| **Known limitations** | Contamination detection for text is mature; **trajectory-level cheating is less covered and is the narrowest genuine gap**. Canaries catch only injections that touch the canary. "Unintended path" requires a model of intended paths, which we author — an unintended path we never imagined looks like success. |

---

## 6. The claim under test: cross-dimension defects

Each dimension above is individually covered by a mature analogue. The Gate 0.9 thesis is that
**defects spanning two dimensions escape all five run independently**. Three constructible
examples, each seeded in `BENCHMARK_V2.md`:

| Cross-dimension defect | Dimensions | Why each single tool passes |
|---|---|---|
| Oracle with mutation score 0.95 that is bypassed **only** under a specific filesystem-ordering environment variation | ORACLE × ENVIRONMENT | Mutation testing runs in the default environment. Environment diffing sees a benign variation. Neither pairs them. |
| Contamination-clean, variance-in-tolerance eval whose pass rate is driven by fixture lineage correlating with the task split | ENVIRONMENT × FACTOR_SPACE | Variance is acceptable. Contamination is clean. The confound is in the assignment. |
| A cheating path that exists **only** when a prior trial leaked answer-bearing residue | ANTI-CHEATING × REPRODUCIBILITY | Contamination scans task text, not runtime state. Isolation checks look for canaries, not for residue that happens to encode the answer. |

> **If these three turn out to be detectable by a single dimension after all, kill condition F3
> fires and nothing in this document is differentiated.**
