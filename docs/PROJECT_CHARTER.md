# PROJECT CHARTER — Agent Reliability & Runtime Assurance

Status: **GATE 0 — ARCHITECTURE REVIEW. NO IMPLEMENTATION AUTHORIZED.**
Document version: 0.1.0
Last updated: 2026-09-21

---

## 0. Truth-label discipline

Every substantive statement in this repository's planning documents carries one of these labels.
The labels are not decoration; downstream documents and code are forbidden from promoting a
statement to a stronger label without a recorded reason.

| Label | Meaning | Who may assert it |
|---|---|---|
| `FACT` | Verifiable by inspection of this system, its code, its recorded data, or a primary source we have actually read. | Deterministic code, or a human who checked the source. |
| `ASSERTED_UNVERIFIED` | Claimed by an upstream input (bootstrap doc, citation, vendor doc) that nobody on this project has independently verified. | Anyone, on ingest. |
| `ASSUMPTION` | A stance we chose in order to proceed. Falsifiable. Has an owner and an invalidation trigger. | Architect. |
| `HYPOTHESIS` | A candidate explanation consistent with the evidence, not established. | Hypothesis generator, human. |
| `DESIGN_DECISION` | A choice with stated alternatives and consequences. Recorded in an ADR. | Architect, via ADR. |
| `OPEN_QUESTION` | Unresolved. Blocks or constrains work. Has a resolution method. | Anyone. |

**Hard rule:** no component, LLM or otherwise, may emit `FACT`. `FACT` is produced only by
deterministic extraction from an immutable ledger record. See `ARCHITECTURE_SURFACE.md` §L.

---

## 0.1 Citation audit — first adversarial finding

`PROJECT_CONTEXT.json` contains a `design_truths` array in which **ten** claims are labelled
`"type": "FACT"` and attributed to sources dated 2026.

**Finding:** the project's own first-principles rule is "do not assume the LLM is correct" and
"do not invent". That rule applies to the project's inputs, not only to its outputs. None of
those ten citations has been verified by anyone on this project. Several of them —
specifically the named papers *"Towards a Science of AI Agent Reliability"* and *"Verified Tool
Calls Improve LLM Agent Reliability Under Non-Atomic Failures"*, the *"Temporal Validity in
Retrieval Memory"* paper, the *Datadog State of AI Engineering 2026* figures, and the
*"MCP 2026-07-28"* protocol revision — cannot be confirmed from the architect's own knowledge.
Treating them as `FACT` is exactly the error the charter exists to prevent.

**Disposition:**

| Claim (abridged) | Original label | Adjudicated label | Reason |
|---|---|---|---|
| Agent behaviour is stochastic and varies across runs | FACT | `FACT` | Directly observable; will be measured by our own harness at Gate 3. Independent of the citation. |
| Reliability needs consistency/robustness/predictability/safety, not single-run success | FACT | `DESIGN_DECISION` | This is a *modelling choice we endorse*, not an empirical fact. Adopt it because single-run success is provably not identifiable under stochasticity — not because a paper says so. |
| Non-atomic tool failures produce duplicate side effects absent idempotency/verification | FACT | `FACT` | True by construction of distributed systems (two-generals). Provable in our own synthetic environment at Gate 2. Citation not required. |
| Provider rate limits are a material failure source and amplify retries | FACT | `FACT` | Directly observable against Groq/OpenRouter. Will be measured, not assumed. |
| Long contexts suffer relevance/noise problems | FACT | `ASSERTED_UNVERIFIED` | Plausible and widely reported, but we have not measured it and it is not load-bearing before Gate 6. Do not build on it. |
| Stale facts are a structural retrieval problem absent temporal validity modelling | FACT | `DESIGN_DECISION` | A schema choice (model validity intervals), not an empirical finding. |
| Indirect prompt injection can cause agents to act on embedded instructions | FACT | `FACT` | Demonstrable in our own environment at Gate 7; we will demonstrate it rather than cite it. |
| Tool metadata/annotations are hints, not enforcement | FACT | `FACT` | True by construction: annotations are data supplied by the same untrusted channel as the payload. Logically entailed, not empirical. |
| OpenTelemetry GenAI conventions exist and warn message content may be sensitive | FACT | `ASSERTED_UNVERIFIED` | Convention names and stability level must be pinned against the actual spec before we depend on field names. See `OQ-07`. |
| Existing platforms already offer tracing/eval/monitoring; do not clone | FACT | `FACT` | Independently true and strategically load-bearing. See §C. |

**Consequence:** citations do not enter the architecture. Every mechanism in this system must be
justifiable from first principles or from a measurement we will take ourselves. Where a
mechanism's *only* justification is an unverified citation, it is deferred, not built.

---

## A. What exact problem are we solving?

### A.1 The one-sentence problem statement

> When a tool-using AI agent finishes acting on enterprise systems, nobody can currently answer,
> with evidence rather than inference, whether every side effect it attempted actually happened
> exactly once — and this project builds the deterministic runtime that can answer it, prove
> failures, and test causal explanations by intervention rather than correlation.

### A.2 The problem restated as a concrete, observable defect

An agent calls `ticket.create(...)`. The HTTP client times out at 30 seconds. The agent's
framework retries. Four things are now simultaneously possible and the system cannot distinguish
between them:

1. The request never reached the service. No ticket exists. Retry is correct.
2. The request reached the service, the ticket was created, the response was lost. Retry
   creates a duplicate. A human is now paged twice.
3. The request reached the service, failed a validation rule, the error response was lost.
   Retry fails identically. The agent burns budget and reports success on a hallucinated
   ticket ID.
4. The request is still in flight and will commit after the retry commits. Retry creates a
   duplicate with a *race*, and the ordering of the two writes is non-deterministic.

**`FACT`:** a timeout carries zero information about whether the side effect occurred. Every
system that retries on timeout without a verification step is, in the formal sense, guessing.

**`FACT`:** this is not an LLM problem. It is a distributed-systems problem that agent
frameworks re-introduced by wrapping non-idempotent HTTP calls in automatic retry loops driven
by a stochastic planner.

The problem generalises. The same structural gap produces:

- an agent that reports "I updated the record" when it updated nothing;
- an incident review that concludes "the model regressed" when a tool's schema changed;
- a reliability number that says 94% when the true rate under a 200ms latency increase is 61%;
- a policy control that exists only as a sentence in a system prompt.

### A.3 What "solved" means — the falsifiable success criteria

The project succeeds if, at Gate 5, the following are demonstrably true in the synthetic
environment. Each is a test, not an opinion.

| # | Criterion | Falsification test |
|---|---|---|
| S1 | No side-effecting tool call is ever classified `VERIFIED_SUCCESS` on the basis of a transport-level response alone. | Inject a tool that returns HTTP 200 and writes nothing. System must classify `VERIFIED_FAILURE`, not success. |
| S2 | Under injected timeout-after-dispatch, the system produces zero duplicate side effects across 200 trials, or explicitly reports `UNKNOWN_OUTCOME` and refuses to retry. | Fault injector; count rows in the synthetic backing store; assert `count == 1 OR state == UNKNOWN_OUTCOME/ABORTED`. |
| S3 | Every causal statement the system emits carries a label from the truth-label set, and no statement labelled `FACT` has an LLM anywhere in its derivation chain. | Static check over the evidence graph: walk provenance edges, assert no `LLM_SYNTHESIS` extraction method under any `FACT`. |
| S4 | The full analysis pipeline — detection, graph, hypothesis ranking, replay, evidence — produces identical output with the LLM provider unreachable, minus the narrative paragraph. | Run the suite with network egress to providers blocked. Diff structured output. Must be byte-identical. |
| S5 | A counterfactual claim ("removing fault F removed the failure") is only emitted when backed by ≥ n paired replay trials with a reported effect size and interval, and is never phrased as root cause. | Schema-level: `CounterfactualEvidence` requires `n`, `effect`, `interval`, `divergence_rate`. Rejected at write time otherwise. |
| S6 | Reliability is reported as a vector with per-cell sample sizes and intervals, never as a single scalar without its vector adjacent. | Report serialiser test. |

If S1, S2, and S4 do not hold, the project has failed regardless of how much else works.

---

## B. Explicit non-goals

Carried from `PROJECT_CONTEXT.json`, plus additions the architecture review adds.

### B.1 Inherited non-goals
- Generic chatbot.
- Generic RAG assistant.
- Dashboard-only observability clone.
- Standalone red-team product.
- Generic multi-agent showcase.
- Production certification claim.
- Real enterprise credential integration in v1.

### B.2 Non-goals added by this review

| Non-goal | Why it is excluded |
|---|---|
| **Competing with LLM tracing/eval vendors on tracing or eval breadth.** | See §C. That market is saturated and well-capitalised. We would lose, and losing there does not advance the actual thesis. We ingest telemetry; we do not sell telemetry. |
| **A general-purpose causal inference engine.** | We have no interventional access to the model's internals and no base rates. We can only do interventions on the *environment*. Claiming more is fraud. See §M in `ARCHITECTURE_SURFACE.md`. |
| **Automatic remediation / self-healing.** | An automatic actor on top of a system whose outcome adjudication is still unproven would amplify every failure mode in `FAILURE_TAXONOMY.md`. Adjudication must be shown correct for a long time before it is allowed to act. |
| **Supporting arbitrary third-party agent frameworks at Gate 1.** | Framework adapters are a long tail with no architectural content. One reference agent we control, behind a documented ingestion contract. Adapters are a post-Gate-5 exercise. |
| **A user-facing UI as a milestone.** | Gate 8 in the bootstrap is a dashboard, and "dashboard-only observability clone" is a stated non-goal. These are in tension. Resolution: every artifact the UI would show must be produced, tested and exportable headlessly *first*. The UI is a rendering of artifacts that already pass tests. It is never the deliverable. |
| **Inference-time model improvement.** | We measure and constrain the agent. We do not fine-tune, prompt-optimise, or route models for quality. Different project. |
| **Multi-tenancy.** | Single logical tenant through Gate 5. Multi-tenant isolation of an append-only evidence ledger is a real design problem and adding it now buys nothing testable. |
| **Distributed deployment.** | Single-process control plane until a measurement forces otherwise. See ADR-0001. |

---

## C. What already solves parts of this — honest competitive assessment

**`FACT`: most of the surface area described in the bootstrap is commodity.** Pretending
otherwise is the fastest way to build something that nobody needs.

| Capability | Already solved, well, by | Our position |
|---|---|---|
| Distributed tracing of LLM/agent calls, span trees, latency/token/cost attribution | OpenTelemetry + OpenLLMetry/OpenInference; LangSmith; Arize Phoenix; W&B Weave; Braintrust; Datadog LLM Observability; Azure AI Foundry observability; Langfuse | **Consume, do not rebuild.** Emit OTel-shaped telemetry. Our event contract is a *superset* with adjudication fields they do not have. |
| Dataset-based offline eval, LLM-as-judge, regression gates in CI | LangSmith, Braintrust, Phoenix, Weave, promptfoo, DeepEval, Inspect | **Do not compete.** Our evaluation plane exists to measure *reliability under fault*, which is a different axis from output quality. |
| Prompt management, versioning, A/B | LangSmith, Braintrust, Langfuse, PromptLayer | **Out of scope entirely.** |
| Guardrails: PII, jailbreak, toxicity, topical filters | NeMo Guardrails, Guardrails AI, Azure Content Safety, Bedrock Guardrails, Lakera | **Out of scope.** Our policy layer authorises *actions on resources*, not text content. |
| Agent sandboxing / egress control | E2B, Modal, Daytona, gVisor, container network policy | **Depend on, do not rebuild.** Our threat model assumes such a layer exists (§ THREAT_MODEL). |
| Durable execution, retries, compensation (saga) | Temporal, Restate, DBOS, AWS Step Functions | **Genuinely overlapping.** Temporal solves durable retry and compensation properly. We are *not* rebuilding Temporal. We are building the layer Temporal does not have: deciding whether a side effect occurred when the answer is unknown, and using that decision to gate the retry Temporal would otherwise perform. See ADR-0001 §Temporal. |
| Idempotency keys, exactly-once-effect semantics | Stripe, payment infrastructure generally; well-understood pattern | **Adopt the known pattern.** Novelty is not in the key; it is in what you do for tools that *do not support one*. |
| Deterministic record/replay of distributed systems | Antithesis, FoundationDB's simulation, rr, Jepsen (for a different question) | **Adopt the mindset.** We are not building a deterministic simulator of the world; we are building replay over a recorded trace with an explicit, published fidelity contract. |
| Fault injection / chaos | Chaos Mesh, Gremlin, Toxiproxy, Litmus | **Adopt. Do not rebuild.** Our fault injector is in-process for the synthetic environment because it must be *deterministically schedulable per-trial*, which network-level chaos tools are not. |
| Causal RCA on traditional telemetry | Datadog Watchdog, Dynatrace Davis, Causely, Netflix's Edgar-lineage work | **Nearest neighbour.** Their substrate is metrics/traces on deterministic services. Ours is a stochastic planner where the same input legitimately produces different actions — which breaks the assumptions most of these rely on. |

### C.1 The uncomfortable conclusion

**`ASSUMPTION` (high confidence, owner: architect, invalidation: a competitor ships
`UNKNOWN_OUTCOME` adjudication as a product):** roughly 70% of what the bootstrap describes is
already a commodity. If this project builds all eight gates evenly, it produces a worse
LangSmith with a worse Temporal attached. The only defensible strategy is to be *narrow and
deep* on the part nobody has built, and *thin and standards-compliant* everywhere else.

---

## D. What is genuinely distinctive

Stated as narrowly as it can honestly be stated. Anything broader is marketing.

### D.1 The core claim

> **Treating the outcome of a side-effecting tool call as an adjudicated verdict with an explicit
> `UNKNOWN` state, and using controlled fault-injection replay as the only permitted source of
> causal evidence about that verdict.**

Three components, none individually novel, whose *combination* is not shipped anywhere we know of:

**(1) `UNKNOWN_OUTCOME` as a first-class, terminal-capable state.**
Every framework we are aware of collapses tool-call outcomes to success/failure. That collapse
is where duplicate side effects are born. Making "we do not know" a state the runtime can *stay
in*, refuse to retry from, and escalate on, is a small change with large consequences.

**(2) A tool capability matrix that mechanically determines retry legality.**
Not a policy someone writes. A decision table over four declared, testable properties of each
tool — `is_read_only`, `accepts_idempotency_key`, `has_postcondition_probe`,
`has_compensation` — that produces a *deterministic* verdict on whether this call may be
retried, must be probed first, or must escalate to a human. A tool that declares a property it
does not have fails a contract test. This turns "be careful with retries" into a compile-time
property.

**(3) Causal claims sourced only from interventions we control.**
We cannot randomise the model. We *can* randomise fault injection. Therefore: the only
counterfactual statement this system is permitted to make is of the form *"across n paired
trials on the same task and seed, with fault F injected vs. not injected, the failure rate
differed by X [CI]."* That is a genuine randomised intervention with a measurable effect. It is
narrow, it is honest, and it is more than correlation-mining over traces can ever deliver.

### D.2 What is explicitly NOT distinctive

Tracing. Span trees. Cost dashboards. LLM-as-judge. Vector retrieval. Graph visualisation.
Multi-agent orchestration. A "reliability score". We will build the minimum of these needed to
support D.1 and no more. Any pull toward expanding them is scope failure and should be
challenged in review.

### D.3 The strategic risk of D.1

**`OPEN_QUESTION` (`OQ-01`, blocking for Gate 2 design):** the entire distinctive claim depends
on tools exposing postcondition probes. In the synthetic environment we control this and it is
trivially true. In any real enterprise, a large fraction of tools will expose no read-after-write
probe, no idempotency key, and no compensation. If that fraction is high, the honest output of
this system for most real tools is *"UNKNOWN, escalate to a human"* — which is correct, valuable,
and also a much smaller product than the bootstrap implies. This must be confronted before Gate 2,
not after. Resolution method: enumerate the tool surface of 3 real enterprise SaaS APIs
(ticketing, identity, HR) from public API documentation and classify each write endpoint against
the four capability properties. Report the actual percentages.

---

## E. Minimum architecture

Detailed in `ARCHITECTURE_SURFACE.md`. Summarised here as the charter-level commitment:

**Eight components. One process. One database. No message broker. No graph database. No vector
database. No workflow engine. No separate policy server.**

1. **Event Contract & Evidence Ledger** — append-only events + content-addressed payload store.
2. **Reference Agent** — one, instrumented, ours.
3. **Synthetic Enterprise Environment** — five services, declared schemas, deterministic fault injection.
4. **Runtime Gateway** — the trust boundary. Policy, budget, idempotency, dispatch, probe, adjudicate.
5. **Detector Suite** — pure functions over the ledger.
6. **Execution Graph** — in-process, built on demand from ledger events.
7. **Replay Engine** — exact + controlled, with divergence detection.
8. **Reliability Harness** — repeated runs under a fault grid, with statistics.

Plus one optional, strictly-bounded, always-removable component:

9. **Evidence Bundler + LLM Explainer** — 0–2 calls, narration only, never in a `FACT` derivation.

---

## F. What should NOT be built yet

Each deferral carries the measurement that would justify building it. "It would be nice" is not
a trigger. "This measurement exceeded this threshold" is.

| Deferred | Trigger to build |
|---|---|
| Neo4j / any graph DB | Execution graph construction for a single incident exceeds 500ms at p95 in-process, *or* cross-execution graph queries become a required feature with > 10^6 nodes. |
| ClickHouse / columnar store | Ledger exceeds 10^8 events *or* detector scans exceed 5s at p95 on Postgres with correct indexes. |
| Qdrant / vector DB | Gate 6 is reached *and* lexical retrieval (BM25 + temporal/provenance filter) demonstrably misses a documented class of queries. Measure the miss rate first. |
| Temporal / durable execution engine | An execution must survive control-plane process restart *and* the internal state machine's recovery path has failed a documented test. Not before. |
| OPA / Rego as a separate process | Policy rule count exceeds ~50, *or* non-engineers must author policy, *or* policy must be updated without redeploy. Until then: a pure-Python decision table behind an OPA-shaped query interface (`input` dict → `{allow, reasons, obligations}`) so migration is a swap, not a rewrite. |
| MCP server/client implementation | Gate 7. The tool interface is an internal Python protocol through Gate 6; MCP is one adapter behind it. Binding to MCP early couples us to a churning spec — see `OQ-07`. |
| A2A / inter-agent protocol | Not before a second agent has a measured reason to exist. Currently there is none. |
| Next.js frontend | After Gate 5, and only over artifacts that already pass headless tests. |
| Rerankers, RRF, dense embeddings | Gate 6, after the lexical baseline's failure modes are measured. |
| Multi-tenancy, RBAC on the control plane, audit export signing | Post-Gate-8. Real requirements, wrong time. |
| Any second LLM provider beyond one primary + one fallback | Never, unless a measured capability gap exists. Two providers is already a source of silent behavioural drift (`FM-11`). |

---

## G–T

Answered in the companion documents:

- **G. Trust boundaries** → `THREAT_MODEL.md` §1, `ARCHITECTURE_SURFACE.md` §G
- **H. Canonical execution/event model** → `ARCHITECTURE_SURFACE.md` §H
- **I. Execution state machine** → `ARCHITECTURE_SURFACE.md` §I
- **J. Non-atomic tool calls** → `ARCHITECTURE_SURFACE.md` §J
- **K. Rate limits, retries, runaway loops** → `ARCHITECTURE_SURFACE.md` §K, `QUOTA_AND_COST_MODEL.md`
- **L. Evidence representation** → `ARCHITECTURE_SURFACE.md` §L
- **M. Causal hypotheses without correlation-as-causation** → `ARCHITECTURE_SURFACE.md` §M
- **N. Replay mode semantics** → `ARCHITECTURE_SURFACE.md` §N
- **O. Reliability measurement** → `EVALUATION_STRATEGY.md` §O
- **P. Protecting evaluation from grader bugs / leakage / shared state** → `EVALUATION_STRATEGY.md` §P
- **Q. What may be stored in telemetry** → `THREAT_MODEL.md` §Q
- **R. What must be redacted or excluded** → `THREAT_MODEL.md` §R
- **S. Usefulness without an LLM provider** → `QUOTA_AND_COST_MODEL.md` §S
- **T. What breaks at enterprise scale** → `ARCHITECTURE_SURFACE.md` §T

---

## Production claim

**`FACT`:** this is a prototype and reference implementation. Every result it produces is
obtained against a synthetic environment the project itself wrote. No claim of production
readiness, enterprise fitness, or real-world reliability improvement is supported by any
artifact in this repository, and no such claim may be made in any document, README, demo, or
report produced here. A synthetic environment can demonstrate that a mechanism *works*; it
cannot demonstrate that the mechanism *matters* at real scale against real APIs.

Saying otherwise is the single easiest way to destroy this project's credibility with the exact
audience it is aimed at.
