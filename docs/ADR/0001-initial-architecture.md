# ADR-0001 — Architecture of the Agent Reliability Experiment Engine

- **Status:** PROPOSED (Gate 0.5). Not accepted. Acceptance requires `OQ-05`, `OQ-08`, `OQ-13`.
- **Date:** 2026-09-21
- **Version:** 0.2.0 — **full rewrite. Supersedes ADR-0001 v0.1.0 in its entirety.**
- **Supersedes:** the "runtime assurance / postcondition verification" thesis and its five decisions.

---

## Context

An external review established that the previous centre of gravity — **unknown side-effect
adjudication and postcondition verification** — is already directly addressed by current research
and tooling: verified tool-call and postcondition work, postcondition MCP implementations,
exactly-once middleware, runtime execution-proof work, and current agent-control standards.

Building a product whose thesis is a solved problem produces a worse copy of something that
exists. The centre must change.

Three facts constrain the replacement:

1. `FACT` — the failures that matter are **interaction effects across layers and over state and
   time**. Single-factor robustness testing cannot reach them by construction.
2. `FACT` — the algorithms needed (covering arrays, ddmin, metamorphic testing, property-based
   testing, fingerprinting, FDR) are **all established and borrowed**. There is no algorithmic
   novelty available here, and claiming any is falsifiable in one search.
3. `FACT` — every one of those algorithms requires **byte-level reproducibility** of a trial.
   Without it, reduction is noise and a regression artifact is fiction.

---

## Decision 1 — Pivot to an experiment engine; demote runtime assurance to a mechanism

**Decision.** The product is an **interaction-aware, stateful experiment engine and failure
laboratory**: explore factor combinations → detect invariant violations → minimize → fingerprint
→ estimate interaction → compile a regression artifact.

Postcondition verification, outcome adjudication, telemetry, replay and policy enforcement are
**supporting mechanisms** — how trials execute and are observed. They are also **factors in the
experiment space**, which is strictly more useful than being the product.

**Alternatives considered.**
- *Continue the 0.1.0 thesis.* Rejected: solved elsewhere.
- *Position as observability.* Rejected: saturated, well-capitalised, and a stated non-goal.
- *Position as agent evaluation.* Rejected: saturated, and measures output quality rather than
  reliability under fault.

**Consequences.**
- (+) The demoted mechanisms become testable subjects instead of assumed-correct infrastructure.
- (+) `OQ-01` stops being a blocker (Decision 2).
- (−) The remaining defensible claim is **three items** (`COMPETITIVE_OVERLAP` §3), not a platform.
- (−) Overlap with deterministic-simulation vendors is **severe** and our determinism story is
  weaker because we do not control the scheduler.

---

## Decision 2 — Capability profile is a varied factor, not an assumption or a blocker

**Decision.** Replace *"what percentage of real APIs expose idempotency keys and probes?"* with a
semantic capability taxonomy CP1–CP8 (`PROJECT_CHARTER` §D.1), declared per adapter, contract-
tested, and **varied as a CONFIG factor in the experiment space**.

We do not need to know how well-behaved real APIs are in order to start. We **degrade the profile
deliberately** and measure what breaks. `CP3=NONE, CP4=EVENTUAL, CP6=NONE` is a level.

The real-API survey survives, demoted, as an **adapter-prioritization study and an
external-validity limitation**. It blocks nothing.

**Alternatives considered.**
- *Survey first, build second.* Rejected: makes the whole project hostage to a number that only
  bounds generalisation, not correctness.
- *Assume well-behaved tools.* Rejected: assumes away the interesting half of the space.

**Consequences.**
- (+) The single largest strategic risk of 0.1.0 — `UNKNOWN_OUTCOME` saturation, where the honest answer for most real tools was "escalate to a human" — becomes an
  experimental dimension rather than an existential question.
- (+) `hidden_retry` likewise becomes a **factor** rather than a dangerous assumption we could
  only hope was false.
- (−) Declaring CP1–CP8 for hundreds of real adapters is unfunded work (T-9). The honest default
  is `UNKNOWN` → `INDETERMINATE`, which is a smaller product.

---

## Decision 3 — Three phases with different designs: screen, reduce, estimate

**Decision.** Screening, reduction and estimation are **separate experiments with separate
designs**, and data does not flow between them freely.

> `FACT`: a t-way covering array guarantees each t-way combination appears **at least once**. At
> least once is n = 1. **Covering arrays detect; they cannot estimate.**

- **Phase 1 — screen.** Constraint-aware covering array at t = 2 (t = 3 on a measured trigger),
  r repetitions. Detects that a failing configuration exists.
- **Phase 2 — reduce.** ddmin with `r_confirm` repetition confirmation, **fingerprint-aware
  acceptance**, non-monotonicity detection. Produces a minimal-with-confidence reproducer.
- **Phase 3 — estimate.** Designed factorial over surviving factors with replication, logistic
  interaction terms, BH-FDR over a **pre-declared** hypothesis family.

Enforced structurally: `ControlledInterventionResult` requires an `InterventionCapability` held
only by the Phase-3 runner, so screening data is **structurally ineligible** to become an
interaction estimate.

**Alternatives considered.**
- *One dataset, one analysis.* Rejected: the covering array looks like a dataset in a dataframe
  and will be analysed as one. This is `EM-02` and it is the most likely quiet failure.
- *Textbook ddmin.* Rejected: assumes determinism and monotonicity. Neither holds. Reporting
  "1-minimal" would be wrong in a way that looks right.

**Consequences.**
- (+) Screening a 7 × 10⁶ configuration space costs ≈ 8 minutes and zero LLM calls.
- (−) **Reduction is the cost centre**: 13–27 hours for 50 failures. The economics are inverted
  from expectation and must be planned around (`QUOTA_AND_COST_MODEL` §1.2).
- (−) `REDUCTION_UNSTABLE` and `INCONCLUSIVE` will be common. That is correct and will be under
  permanent pressure from anyone wanting a cleaner result.

---

## Decision 4 — Capability isolation replaces source lint as the primary control

**Decision.** Illegal evidence production is **impossible by construction**. `ObservedEvent`,
`OracleVerdict`, `ReductionStep`, `ControlledInterventionResult`, `DispatchHandle` and
`FixtureVersion` each require a construction token held by exactly one component. The narrator is
a pure function holding **no** capability, so it cannot construct any of them.

Static lint is retained, explicitly **secondary**.

**Alternatives considered.**
- *Source lint as the architecture (0.1.0).* Rejected: a lint is a review aid, not a boundary.
- *Database check constraints on labels (0.1.0).* Retained as a **secondary** defence, but the
  constraint fires at write time — after the wrong object already exists and has been passed
  around.

**Consequences.**
- (+) `EM-02` (screening data as an intervention) and narrator-manufactured observations become
  type errors rather than review findings.
- (−) `ASSUMPTION`: in Python, construction tokens are enforceable in practice but not against a
  determined bypass. **Structural, not airtight**, and the documents say exactly that.

---

## Decision 5 — Evidence is five orthogonal fields plus a claim scope

**Decision.** The 0.1.0 label lattice equated derivation method with epistemic truth. That was
wrong: a deterministic parse of a lying tool's response is a deterministic derivation **about what
the tool said**, not a fact about the world.

Five independent fields — `claim_type`, `observation_source`, `provenance` (with trust class),
`assurance_level`, `derivation_method` — plus `claim_scope`.

The rules that bind (`ARCHITECTURE_SURFACE` §L.1):
- `claim_scope = PRODUCTION_BEHAVIOR` is **unconstructible**. No capability produces it.
- A `TOOL_RESPONSE` source at `SELF_REPORTED` assurance can only carry
  `WHAT_THE_SOURCE_REPORTED`, however deterministic the parse.
- `ENVIRONMENT_STATE` requires the ground-truth effect log at `ENVIRONMENT_VERIFIED` assurance.
- `CONTROLLED_INTERVENTION_RESULT` requires a Phase-3 design, n, interval and `divergence_rate`.

Causal vocabulary corrected: **"randomized" is removed** — assignment in a covering array is
*systematic*, not random. A version change is `STATISTICAL_ASSOCIATION`, not a "natural
experiment". *Root cause*, *caused by*, *due to*, *because of* are banned from output.

**Consequences.**
- (+) Pivot correction 7 becomes structural: for an unsupported system the output is
  `INDETERMINATE` / `UNVERIFIABLE`, and nothing can claim otherwise.
- (−) More fields to populate correctly, and a combination matrix that needs its own tests.

---

## Decision 6 — Effect identity is three separate fields

**Decision.** `effect_id` (stable logical identity), `request_digest` (exact wire bytes),
`schema_version` (contract version) are **separate and never conflated**.

`effect_id = HMAC(secret, execution_id ‖ logical_step_id ‖ semantic_key(args))`.

**0.1.0's derivation mixed `schema_version` into the key.** That meant a serialization change
silently created a *different* side-effect identity for the *same* logical effect — a correctness
bug the design itself introduced. Corrected:

- A retry of the same logical effect retains the same `effect_id`, whatever the serialization.
- A semantic change without a declared new logical step **fails closed**
  (`ABORTED / EFFECT_IDENTITY_VIOLATION`). Never a silent new key.
- A tool that cannot supply `semantic_key` is `CP1=UNSTABLE`, forcing `NON_IDEMPOTENT_WRITE`
  treatment.

Retry legality is a lookup over `(CP2 effect class, CP1, CP3)` — **HTTP verb and `read_only` are
removed as criteria**, and `UNKNOWN` routes to `INDETERMINATE`.

**Consequences.**
- (+) INV-1 (no duplicate side effect) becomes evaluable. Under 0.1.0's derivation it silently
  could not fire across a schema change.
- (+) The semantic effect class makes `UNKNOWN` a first-class, honest verdict.

---

## Decision 7 — Deterministic critical path; three unpooled evidence layers; six gates

**Decision.** Three parts, one rationale.

**(a) The critical path contains zero LLM calls.** The LLM is the system under test or an optional
narrator. It is never required for fault scheduling, experiment arithmetic, authorization, state
transitions, oracle evaluation, anomaly math, hypothesis ranking, replay integrity, scoring,
evidence bookkeeping, or failure reduction. Criterion **S5** tests this on every CI run.

**(b) The arbitrary 90/7/3 split is removed.** Three evidence layers — **L-DET** (deterministic,
exhaustive where feasible), **L-LOC** (fixed local canary), **L-REM** (fixed budgeted remote
canary) — **never pooled into one statistic**. Each answers a different question; a weighted
average answers none.

**(c) Minimum infrastructure, six gates.** One process, PostgreSQL, filesystem fixture store. No
graph DB, no vector DB, no ClickHouse, no Temporal, no OPA process, no A2A, no Bayesian
optimization, no RL test generation, no multi-agent swarm, no frontend. Each deferral carries a
measured trigger. Gates 1–6 in `ARCHITECTURE_SURFACE.md`, each with objective, scope, invariants,
exit tests, a reason it is not premature, and explicit deferrals.

**Consequences.**
- (+) The 21-day-sweep problem of 0.1.0 disappears entirely: screening costs minutes and zero
  remote calls.
- (+) The provider-unavailable path is exercised on every CI run rather than described.
- (−) **The honest cost:** most trials run in L-DET against a stub planner. L-DET results are
  mostly about *our runtime*, not about agents. The layering makes that visible; it does not
  dissolve it. This is `FAILURE_TAXONOMY` §3.4 kill-reason 2 and it is not resolved by this ADR.

---

## Decisions explicitly deferred

| Deferred | Why |
|---|---|
| Concurrency factor scope (`OQ-05`) | Coarse adapter-controlled interleaving vs. true deterministic scheduling. The latter is a simulator build. **Blocks acceptance.** |
| Factor-model validation method (`OQ-08`) | Seeded externally-authored bugs, detection rate measured. **Blocks acceptance** — a low rate triggers kill-reason 5. |
| CAS retention per sensitivity class (`OQ-13`) | Policy decision with a named owner. Blocks reduction scope. **Blocks acceptance.** |
| FDR family definition (`OQ-06`) | Needs Gate 3 firing-rate data. |
| `r_confirm` and `fingerprint_stability` thresholds (`OQ-07`) | Currently placeholders. Must be derived from measured discordance, not chosen. |
| Parallel reduction isolation (`OQ-17`) | Multiplies either `EM-12` risk or T-3 storage. |

---

## How this ADR gets falsified

Revise, do not defend, if any of these is observed:

1. **INV-9 (determinism) cannot be held.** Reduction, fingerprinting and every regression artifact
   become meaningless. This is the architecture's foundation and `EM-01` says it degrades
   *silently*.
2. **`OQ-08` shows a low detection rate on externally-authored seeded bugs.** The engine finds
   only what its authors imagined. `FAILURE_TAXONOMY` §3.4 reason 5 applies and the project should
   stop.
3. **`OQ-05` resolves toward requiring a deterministic scheduler.** F10 and F14 — the most valuable
   interaction classes — are then either out of reach or the project becomes a simulator build.
   Neither is the project described here.
4. **`COMPETITIVE_OVERLAP` R3 assesses true at Gate 3** — a competent team can re-derive all three
   contributions in two weeks on Hypothesis + ACTS + ClusterFuzz. The contribution is a
   configuration, not a platform.
5. **Reduction backlog exceeds screening throughput by more than an order of magnitude sustained.**
   The workflow does not close and produces failures faster than it can characterise them.
6. **L-DET results fail to predict L-LOC / L-REM results at Gate 6.** The deterministic substrate
   is testing the harness, not any agent behaviour.

Each is a test at a named gate. An architecture with no falsification conditions is a preference.
