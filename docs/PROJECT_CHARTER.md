# PROJECT CHARTER — Agent Reliability Experiment Engine

Working title: **Agent Failure Laboratory**
Status: **GATE 0.5 — ARCHITECTURE PIVOT. NO IMPLEMENTATION AUTHORIZED.**
Document version: 0.2.0 (full rewrite; supersedes 0.1.0)
Date: 2026-09-21

---

## 0. What changed and why

Version 0.1.0 centred the product on **unknown side-effect adjudication and postcondition
verification**. That centre is retired.

**Reason:** the problem is directly addressed by current research and tooling — verified
tool-call / postcondition work, postcondition MCP implementations, exactly-once middleware,
runtime execution-proof work, and current agent-control standards. Building a product whose
thesis is a solved problem produces a worse version of something that already exists.

**Retained, demoted:** runtime assurance, outcome adjudication, postcondition verification,
telemetry, replay, and policy enforcement are **supporting mechanisms**. They are how trials are
executed and observed. They are not the product.

**New centre:** an interaction-aware, stateful **experiment engine and failure laboratory**.

### 0.1 Truth discipline

Every substantive statement carries one label from the model in §L. The old six-label scheme is
replaced (see `ARCHITECTURE_SURFACE.md` §L) because it conflated *how a statement was derived*
with *how true it is*.

Planning-document labels used below: `FACT`, `ASSERTED_UNVERIFIED`, `ASSUMPTION`,
`DESIGN_DECISION`, `OPEN_QUESTION`.

### 0.2 Citation audit — retained from 0.1.0

`PROJECT_CONTEXT.json` labels ten claims `FACT` with 2026 citations that nobody on this project
has verified. That adjudication stands: no citation enters the architecture. Every mechanism is
justified from first principles or from a measurement this project will take itself. See
`TASK_STATE.json` → `dangerous_assumptions.DA-12`.

---

## A. What exact problem are we solving?

### A.1 The problem statement

> Given a tool-using agent, its environment model, tool contracts, and explicit reliability and
> safety invariants, **systematically explore combinations** of runtime faults, tool behaviour,
> context and state mutations, policy and configuration changes, and model behaviour; identify
> invariant violations; **minimize** the failing configuration and trace; and **compile** the
> result into a reproducible regression test.

### A.2 The hard part, stated precisely

`FACT`: single-factor robustness testing is cheap and widely done. Inject a timeout, see what
breaks. The failures that reach production are **interaction effects across layers and over
state and time**, which single-factor testing cannot reach by construction.

Worked examples the engine must be able to find:

| Interaction | Why neither factor alone shows it |
|---|---|
| timeout-after-dispatch **+** retry policy **+** eventual consistency | Each is individually handled. Together, the reconciliation probe reads a stale replica, reports NOT_APPLIED, and authorises the duplicating retry. |
| stale context **+** tool schema change | Stale context alone yields a wrong-but-valid call. Schema change alone yields a clean contract error. Together, the stale context supplies arguments that happen to validate under the new schema and silently target the wrong resource. |
| permission narrowing **+** re-planning | Narrowing alone produces a clean denial. Re-planning alone is normal. Together, the agent routes around the denial via a different tool that is still permitted and achieves the forbidden effect. |
| 429 **+** nested retry **+** parallel workers | Each worker's backoff is correct in isolation. Together they resynchronise and amplify. |
| model drift **+** changed policy | Either alone is detectable by a canary. Together, the drift changes which tools are selected and the policy change makes a previously-unused path newly reachable. |
| two individually-safe agents **+** shared state | Each satisfies its invariants in isolation. Interleaved, one's write invalidates the other's precondition between check and act. |

`DESIGN_DECISION`: the factor model, the covering-array planner, the reducer and the interaction
analyzer exist **only** to make this class reachable, minimizable and regression-testable. Any
component that does not serve that is out of scope.

### A.3 The claim, narrowed

**We do not claim these combinations are unexplored by industry.** They are known, and several
are named in vendor postmortems and research.

The claim is narrower and is the whole claim:

> An **integrated, reusable, experiment-driven workflow** that makes interaction effects in
> tool-using agents **measurable, reducible, and regression-testable** — with explicit coverage
> accounting and explicit limits on what each result supports.

Every word is load-bearing. *Integrated* because the pieces exist separately. *Reusable* because
a one-off investigation is not a product. *Measurable* because detection without estimation is
an anecdote. *Reducible* because an unminimized failure is not actionable. *Regression-testable*
because a failure you cannot re-run is a story.

### A.4 Falsifiable success criteria

| # | Criterion | Falsification test |
|---|---|---|
| S1 | A trial is byte-reproducible: same `(fixture_digest, seed, factor_assignment, agent_version, environment_version)` ⇒ same `trace_digest`. | Run 1000 trials twice. Any digest mismatch fails the gate. |
| S2 | The engine finds at least one **2-factor interaction failure that neither factor produces alone**, in the synthetic environment, without that interaction being pre-scripted. | Run the pairwise screen. Verify the failing pair's single-factor arms pass at n≥50 each. |
| S3 | A found failure minimizes to a configuration that reproduces at a **recorded, measured rate**, and the reducer reports non-monotonicity when it occurs rather than hiding it. | Seeded non-monotone fixture: reducer must emit `REDUCTION_UNSTABLE`, not a confident minimum. |
| S4 | A minimized failure compiles to a regression artifact that **re-executes from the artifact alone** on a clean checkout and reproduces at its recorded rate ± its recorded interval. | Artifact replay test in CI. |
| S5 | The entire deterministic pipeline — plan, run, oracle, reduce, fingerprint, analyze, compile — runs with **zero LLM availability**, producing byte-identical structured output. | Block provider egress. Diff structured output. |
| S6 | Coverage reports state what was **not** explored, with sample sizes, and no result is reported without its n and interval. | Report serialiser test. |
| S7 | No oracle verdict, observed event, or reduction step can be constructed by any component that does not hold the corresponding capability token. | Type-level: the narrator module is not passed a capability; attempting construction is a type error, not a lint warning. |

If S1 or S5 fails, the project has failed. S1 because without determinism there is no
minimization and no regression artifact. S5 because an experiment engine that needs an LLM to
schedule its own experiments has no claim to rigour.

---

## B. Explicit non-goals

### B.1 Not the product centre
- Generic observability / tracing / dashboards.
- Generic RAG.
- **Standalone postcondition verification.** (Demoted from 0.1.0's centre. Retained as a mechanism.)
- Generic red-teaming or jailbreak research.
- Generic multi-agent demo.

### B.2 Added by this review
| Non-goal | Why |
|---|---|
| **Proving exactly-once execution against arbitrary external systems.** | We can prove what the controlled environment and its oracle support. For anything else the correct output is `INDETERMINATE` / `UNVERIFIABLE`. See §D.4. |
| **A causal engine.** | We run *controlled interventions* inside a fixture. That supports statements about the experiment, never about production. |
| **Autonomous remediation.** | Out of scope until adjudication and reduction are shown correct over a long period. |
| **A deterministic concurrency simulator.** | The `concurrency` factor is scoped to *coarse interleaving points the adapter controls*, not arbitrary thread scheduling. Full deterministic scheduling is a different and much larger build. See `OQ-05`. |
| **A UI as a milestone.** | Every artifact must be produced, tested and exportable headlessly. Rendering is last. |
| **Supporting arbitrary third-party agent frameworks initially.** | One reference agent behind a documented adapter contract. Adapters are a long tail with no architectural content. |

---

## C. What exists already — four explicit categories

`COMPETITIVE_OVERLAP.md` holds the detail. The charter records the four-way split the pivot
requires.

### C.1 Existing capabilities we **consume** (build none of this)
| Capability | Source |
|---|---|
| Tracing, span trees, cost/latency attribution | OpenTelemetry + GenAI conventions, OpenLLMetry/OpenInference |
| Property-based and stateful testing primitives | Hypothesis (Python) |
| Covering-array generation | Established CT tooling / published IPOG-family algorithms |
| Delta debugging | Published ddmin and its descendants |
| Sandboxing and egress control | Container/runtime layer; not ours |
| Durable execution, compensation | Temporal et al. — **not adopted**, not rebuilt |
| Statistical primitives | Wilson intervals, McNemar, Benjamini–Hochberg — standard |

### C.2 Existing research and tools that **overlap** (we are not first)
| Overlap | Who | Severity |
|---|---|---|
| Deterministic simulation testing: explore fault combinations, minimize, reproduce | FoundationDB simulation lineage; Antithesis; TigerBeetle VOPR | **SEVERE** — closest commercial overlap |
| Chaos engineering with fault matrices | Gremlin, Chaos Mesh, Litmus | HIGH |
| Combinatorial interaction testing | NIST ACTS, PICT, CAgen; 25 years of published CIT literature | **The algorithms are theirs** |
| Test-case minimization + regression compilation | ClusterFuzz, OSS-Fuzz, C-Reduce, Perses | **The loop is theirs** |
| Crash bucketing / failure fingerprinting | ClusterFuzz grouping, Sentry grouping | HIGH |
| Model-based testing of stateful systems | TLA+/P, stateful Hypothesis, QuickCheck-family | HIGH |
| Agent eval harnesses with fault plugins | Inspect, promptfoo, DeepEval, LangSmith, Braintrust | MEDIUM |
| Verified tool calls / postcondition MCP / exactly-once middleware | The 0.1.0 thesis | **Solved. Demoted to mechanism.** |

### C.3 Our integration focus (the only defensible claim)
1. A **factor model specific to tool-using agents** — fault × tool-capability × context × policy ×
   runtime × model — where **tool capability profile is itself a varied factor**, not an assumed
   constant.
2. An **oracle set expressed as agent-reliability invariants** rather than as crash/assert
   signals, evaluable deterministically against an environment ground-truth log.
3. A **regression artifact format for stochastic, stateful agent failures**, carrying a measured
   reproduction rate and explicit invalidation conditions, rather than a pass/fail test.
4. **Coverage accounting over the agent factor space**, including the unexplored remainder.

Everything else is borrowed, and the documents say so.

### C.4 What we will **NOT** claim
- Not that interaction effects in agents are undiscovered.
- Not that the algorithms are novel. They are textbook and we cite them as such.
- Not exactly-once against arbitrary external systems (§D.4).
- Not causality beyond controlled intervention inside a fixture.
- Not production readiness, from synthetic evidence.
- Not a single "agent reliability score".
- Not that a passing experiment suite means an agent is safe. It means the explored region
  produced no invariant violation, at the stated sample sizes.

---

## D. Mandatory corrections applied

The pivot's ten corrections, and where each now lives.

### D.1 `OQ-01` reframed — capability profile becomes a factor

**Old:** block the first runtime implementation on *"what percentage of real APIs expose
idempotency keys and probes?"*

**New:** a **semantic capability taxonomy**. Each adapter declares an 8-property profile:

| # | Property | Values |
|---|---|---|
| CP1 | Stable logical action identity | `STABLE` / `DERIVABLE` / `UNSTABLE` |
| CP2 | Idempotent effect semantics | `PURE_READ` / `IDEMPOTENT_READ` / `IDEMPOTENT_WRITE` / `NON_IDEMPOTENT_WRITE` / `UNKNOWN` |
| CP3 | Authoritative reconciliation source | `AUTHORITATIVE` / `REPLICA` / `NONE` |
| CP4 | Consistency / freshness bound | `STRONG` / `BOUNDED(ms)` / `EVENTUAL` / `UNDECLARED` |
| CP5 | Postcondition predicate | `EXACT` / `HEURISTIC` / `NONE` |
| CP6 | Outcome discrimination | can distinguish `APPLIED` / `NOT_APPLIED` / `INDETERMINATE`: `FULL` / `PARTIAL` / `NONE` |
| CP7 | Probe safety | `SIDE_EFFECT_FREE` / `COSTLY` / `UNSAFE` |
| CP8 | Required authorization scope | declared scope set |

**The payoff, and it is the single most important consequence of the pivot:** the capability
profile is a **CONFIG factor in the experiment space**. We do not need to know what fraction of
real APIs are well-behaved in order to start. We *degrade the profile deliberately* and measure
what breaks. `CP3=NONE, CP4=EVENTUAL, CP6=NONE` is a level, not an obstacle.

The real-API survey survives, demoted, as **`OQ-01`: adapter-prioritization study and
external-validity limitation** — it tells us which profile levels deserve the most trials and
bounds what our results generalise to. It blocks **nothing**.

### D.2 Idempotency semantics fixed — three separate identities

`FACT` (logical): deriving a deduplication key from schema version means a serialization change
silently creates a *different* side-effect identity for the *same* logical effect. That is a
correctness bug the 0.1.0 design introduced. Corrected:

| Field | Meaning | Changes when |
|---|---|---|
| `effect_id` | **Stable logical action identity.** The identity of the intended effect. | The *semantic* target or intent changes. Never on serialization or schema change. |
| `request_digest` | Exact wire representation of one attempt. | Any byte changes. Used for replay matching and drift detection. **Never for dedup.** |
| `schema_version` | Contract version in force. | Tool contract changes. Recorded, never mixed into `effect_id`. |

`effect_id = H(execution_id ‖ logical_step_id ‖ semantic_key(args))` where `semantic_key` is a
tool-declared extractor over the semantically-identifying arguments, HMAC'd with a per-deployment
secret (§THREAT_MODEL §4.5).

**Rules:**
- A retry of the same logical effect **retains the same `effect_id`**, whatever the serialization.
- If `semantic_key` changes between attempts, that is a **new logical action** if the planner
  intended one, or a **failed-closed contract violation** if it did not. It is never a silent new
  key.
- A tool that cannot supply `semantic_key` has `CP1=UNSTABLE`, which forces `CP2` treatment as
  `NON_IDEMPOTENT_WRITE` regardless of what it claims.

### D.3 Evidence semantics fixed — five orthogonal fields

`FACT` (logical): extraction method is not epistemic truth. A deterministic parse of a lying
tool's response is a deterministic derivation **about what the tool said**, not a fact about the
world. The 0.1.0 label lattice made that error. Corrected to five independent fields plus an
explicit claim scope — see `ARCHITECTURE_SURFACE.md` §L.

The governing rule: **`claim_scope` is never `PRODUCTION_BEHAVIOR`.** This system cannot assert
it, at any assurance level, by any derivation method.

### D.4 Causal language fixed

- "Randomized intervention" is **removed**. Treatment assignment in a covering array is
  *systematic*, not random. Within a designed follow-up, assignment is *balanced*, not random,
  unless we explicitly randomise — and where we do, we say so and show the randomisation.
- Fault on/off experiments are **`CONTROLLED_INTERVENTION_RESULT`**.
- A deployment or version change is **`STATISTICAL_ASSOCIATION`** — observational. The phrase
  "natural experiment" is removed; nothing in our design justifies it.
- Banned from all output: *root cause*, *caused by*, *due to*, *because of*, *natural experiment*,
  *randomized*. Lint is a secondary check; the primary control is that no component holds a
  capability to construct a claim at those types (§D.8).

### D.5 The 90/7/3 split is removed — three separate evidence layers

`DESIGN_DECISION`: the arbitrary percentage split is gone. Three layers, **never combined into a
single statistic**:

| Layer | What it is | Sizing | What it supports |
|---|---|---|---|
| **L-DET** | Deterministic runtime/oracle contract suite against the stub planner and fixture substrate. | **Exhaustive where feasible**, covering-array bounded otherwise. | Claims about the *runtime, adapters and oracles*. Nothing about models. |
| **L-LOC** | Local-model behavioural canary. | **Fixed small set**, declared up front. | Claims about *one local model's behaviour under named factors*. |
| **L-REM** | Remote-provider canary. | **Fixed budgeted set**, declared up front. | Claims about *that provider/model on that date*. Nothing beyond. |

There is no weighting, no pooling, no aggregate. A report showing all three shows three tables.

### D.6 Retry safety by semantic effect class

HTTP verb and a `read_only` boolean are removed as retry criteria. Retry legality is a lookup
over `(CP2 effect class, CP1 identity stability, CP3 reconciliation source)` — see
`ARCHITECTURE_SURFACE.md` §J. The `UNKNOWN` class exists and routes to `INDETERMINATE`.

### D.7 Central claim narrowed

> The platform proves what its **controlled environment and its oracle** support. For any system
> whose adapter does not supply the required capability, the output is **`INDETERMINATE` /
> `UNVERIFIABLE`**, and that is a first-class, frequently-correct result.

"Exactly-once against arbitrary external systems" is unprovable by this or any system, and the
documents say so.

### D.8 Capability isolation replaces source lint as the primary control

`DESIGN_DECISION`: illegal evidence production is made **impossible by construction**, not
forbidden by a linter.

- `ObservedEvent` can only be constructed with a `LedgerReadCapability`.
- `OracleVerdict` can only be constructed with an `OracleCapability`.
- `ReductionStep` can only be constructed with a `ReducerCapability`.
- `DispatchHandle` can only be obtained with a `DispatchCapability`.
- The narrator is a pure function `(EvidenceBundle) -> str`. It is **never passed any capability
  token**, so it cannot construct any of the above. Not "must not" — cannot.

Static lint is retained as a **secondary** defence only, and the documents no longer describe it
as the architecture.

### D.9 Replay claims made precise

| Mode | Definition | Claim it supports |
|---|---|---|
| `EXACT_REPLAY` | Recorded inputs and outputs replayed; **no external effects**. | "Our analysis produces X over this recorded trace." |
| `CONTROLLED_REPLAY` | Frozen deterministic fixtures and substrate; factors toggled. | "Under this fixture and seed, this factor assignment yields this outcome at rate k/n." |
| `LIVE_EXECUTION` | Fresh external calls. Stochastic. **Not replay-equivalent** and not called replay. | "On this date, against this provider, n runs produced this distribution." |

**Any trajectory divergence invalidates downstream counterfactual interpretation** unless
explicitly handled — the divergence detector is mandatory and `divergence_rate` is a required
field on every intervention result. See `ARCHITECTURE_SURFACE.md` §N.

### D.10 LLM demoted to optional narration

The LLM is the **system under test** or an **optional narrator**. It is never required for:
fault scheduling, experiment design arithmetic, authorization, state transitions, oracle
evaluation, anomaly math, hypothesis ranking, replay integrity, benchmark scoring, evidence
bookkeeping, or failure reduction.

Criterion S5 tests this on every CI run.

---

## E. Minimum architecture

Eleven components, one process, PostgreSQL plus a filesystem fixture store. Detail in
`ARCHITECTURE_SURFACE.md`.

```
Agent Under Test → Execution Adapter → Canonical Event Ledger
                                     → Versioned Environment State + Fixture Store
                                     → Oracle Engine
Experiment Planner → Runner / Fault Injector → (trials)
Failure Reducer → Failure Fingerprint → Interaction Analyzer → Regression Compiler
                                                              → Optional LLM Narrator
```

**Deferred with measured triggers:** Neo4j, Qdrant, ClickHouse, Temporal, OPA as a process, A2A,
Bayesian optimization, RL-based test generation, multi-agent swarms, any frontend.

---

## F. What should NOT be built yet

| Deferred | Trigger |
|---|---|
| Graph database | Factor/execution graph queries exceed 500 ms p95 from relational tables **and** the query set is genuinely graph-shaped. |
| ClickHouse | Ledger exceeds 10⁸ rows **or** coverage queries exceed 5 s p95 on indexed Postgres. |
| Vector database | A retrieval-dependent factor is introduced **and** lexical baseline miss rate is measured and unacceptable. |
| Temporal | A trial must survive runner restart **and** the in-process recovery path has failed a documented test. |
| OPA process | Policy factor levels exceed ~50 rules **or** non-engineers must author them. Until then: a pure decision table behind an OPA-shaped query interface. |
| Bayesian optimization / RL test generation | Transparent adaptive search (C3) has been measured and shown insufficient on a named metric. |
| MCP adapter | Gate 6. One adapter behind the internal tool protocol. |
| Deterministic concurrency scheduler | `OQ-05` resolved and the coarse interleaving model shown insufficient. |
| Frontend | After Gate 6, over artifacts that already pass headless tests. |

---

## G–T and the rest

- Trust boundaries, event model, state machine, retry semantics, evidence model, replay, scale
  limits → `ARCHITECTURE_SURFACE.md`
- Factor space, covering arrays, reduction, fingerprinting, interaction analysis →
  `INTERACTION_MODEL.md`
- Regression artifact format → `REGRESSION_ARTIFACT_SPEC.md`
- Failure classes, architectural failure modes, adversarial review → `FAILURE_TAXONOMY.md`
- Oracles, coverage accounting, statistics, leakage → `EVALUATION_STRATEGY.md`
- Telemetry, redaction, capability isolation → `THREAT_MODEL.md`
- Budgets and degradation → `QUOTA_AND_COST_MODEL.md`
- Competitive detail → `COMPETITIVE_OVERLAP.md`
- Decisions of record → `ADR/0001-initial-architecture.md`

---

## Production claim

`FACT`: this is a prototype and reference implementation. All results are obtained against a
synthetic environment and a factor model **this project authored**. A synthetic environment can
demonstrate that a mechanism works; it cannot demonstrate that the mechanism matters at real
scale against real systems. No claim of production readiness or of real-world reliability
improvement is supported by any artifact here.

**The sharpest version of that limitation, which belongs in every report:** the factor model is
our own hypothesis about what can go wrong. The engine can only find failures inside a space we
imagined. That is the weakest possible property for a discovery tool, and no amount of coverage
accounting fixes it — coverage is measured *against our own space*. `OQ-08` exists to attack it.
