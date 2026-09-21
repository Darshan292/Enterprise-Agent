# ADR-0001 — Initial Architecture

- **Status:** PROPOSED (Gate 0). Not accepted. Acceptance requires resolution of `OQ-01`, `OQ-03`, `OQ-05`.
- **Date:** 2026-09-21
- **Supersedes:** none
- **Context documents:** `PROJECT_CHARTER.md`, `ARCHITECTURE_SURFACE.md`, `FAILURE_TAXONOMY.md`, `THREAT_MODEL.md`, `EVALUATION_STRATEGY.md`, `QUOTA_AND_COST_MODEL.md`

---

## Context

We are asked to build a platform that determines whether a tool-using AI agent executed safely
and correctly, detects operational failures, verifies side effects, evaluates reliability under
faults, reconstructs evidence-backed causal hypotheses, and safely replays executions.

Three facts constrain every choice:

1. **`FACT`** — roughly 70% of the described surface (tracing, eval harnesses, dashboards,
   retrieval, sandboxing, durable execution, chaos injection) is already commodity, shipped by
   well-resourced vendors. Building it evenly produces a worse version of several existing products.
2. **`FACT`** — the inference budget (≈1,000 calls/day, TPM-bound to ≈2–4 calls/minute at realistic
   call sizes) makes any design with a remote model in the evaluation loop arithmetically
   impossible, not merely expensive (`QUOTA_AND_COST_MODEL` §4.1).
3. **`FACT`** — causal identification from observational agent traces is not possible. There is no
   randomisation, no instrument, and numerous unmeasured confounders.

This ADR records the five decisions that follow from those constraints and that everything else
depends on.

---

## Decision 1 — The runtime gateway is the sole enforcement boundary; the model only proposes

**Decision.** All model output is a *proposal record*. A single runtime gateway performs, in
order and deterministically: schema validation → policy authorisation → budget/rate accounting →
idempotency-key derivation → dispatch → postcondition probe → state adjudication. The gateway is
the only component holding transport credentials. No other module may import the transport client.

**Alternatives considered.**
- *Policy as prompt instruction.* Rejected: not enforceable; the model is the untrusted component.
- *Policy via tool annotations / MCP metadata.* Rejected: annotations arrive over the same channel,
  from the same party, as the payload. Anything that can lie about one can lie about the other.
  They are **inputs to** policy, never policy.
- *Enforcement distributed across agent framework hooks.* Rejected: every hook is a bypass waiting
  to be written, and bypasses fail open.

**Consequences.**
- (+) Security claims become testable (`THREAT_MODEL` SC-1, SC-10) rather than asserted.
- (+) A single, auditable authorisation record per action.
- (−) The gateway is a synchronous single point of failure (`T-9`) and an aggregation of privilege
  (`THREAT_MODEL` §4.2). Both are accepted and named rather than mitigated away.
- (−) Inline probes sit on the critical path and roughly double side-effecting action latency
  (`T-1`). Moving them async yields a strictly weaker safety property and must be a recorded
  trade, not a silent optimisation.

---

## Decision 2 — Outcome is an adjudicated state with `UNKNOWN_OUTCOME` first-class; retry legality comes from a contract-tested capability matrix

**Decision.** An `Action` (not an attempt) carries one state from the fixed set in
`KNOWLEDGE_GRAPH.execution_state_machine`. Transport-level success yields `ACKNOWLEDGED`, never
`VERIFIED_SUCCESS` — a 2xx is the tool's *claim*, not proof. Verification requires a postcondition
probe (`outcome_basis=PROBE`). Timeout yields `UNKNOWN_OUTCOME`, which may be **terminal**.

Retry legality is a lookup, not a judgement:

| read-only | idem-key | probe | verdict |
|---|---|---|---|
| true | — | — | `RETRY_SAFE` |
| false | true | — | `RETRY_SAFE` (same key) |
| false | false | true | `PROBE_THEN_DECIDE` |
| false | false | false | **`BLIND_WRITE — NEVER RETRY` → `ESCALATE`** |

Each declared property is verified by a contract test; a tool that mis-declares fails the build.

Every tool additionally declares `consistency_model`, defaulting to `UNDECLARED`. Under
`EVENTUAL` or `UNDECLARED`, a negative probe **never** yields `VERIFIED_FAILURE` — only
`UNKNOWN_OUTCOME`.

**Alternatives considered.**
- *Binary success/failure.* Rejected: this collapse is precisely where duplicate side effects are
  born. It forces a guess at the one moment guessing is most expensive.
- *Probe-then-retry without a consistency model.* **Rejected as actively harmful.** Against an
  eventually-consistent store it converts an honest `UNKNOWN` into a confident wrong
  `VERIFIED_FAILURE`, and that confidence authorises the duplicating retry. The safety mechanism
  would manufacture the bug it exists to prevent (`FM-05`).
- *Retry policy as per-tool configuration.* Rejected: configuration drifts from reality silently.
  A contract-tested declaration cannot.

**Consequences.**
- (+) This is the project's distinctive contribution (`PROJECT_CHARTER` §D). Everything else is
  scaffolding for it.
- (+) A tool cannot lie about itself in a way that survives CI.
- (−) **`FM-04`: for real tool surfaces lacking probes and idempotency keys, the honest output is
  "UNKNOWN, escalate to a human" at a possibly high rate.** The synthetic environment hides this
  entirely because we implemented the tools. This is the largest strategic risk in the project and
  `OQ-01` exists to measure it before Gate 2.

---

## Decision 3 — An evidence label is a function of extraction method, enforced at write time

**Decision.** `Evidence.label` is constrained by `Evidence.extraction_method` via a database check
constraint, not by convention:

| extraction_method | permitted labels |
|---|---|
| `DETERMINISTIC_RULE` | `FACT`, `OBSERVATION`, `UNKNOWN` |
| `STATISTICAL_TEST` | `CORRELATION`, `OBSERVATION`, `UNKNOWN` |
| `REPLAY_EXPERIMENT` | `COUNTERFACTUAL_EVIDENCE`, `OBSERVATION`, `UNKNOWN` |
| `HUMAN_ASSERTION` | `HYPOTHESIS`, `OBSERVATION`, `UNKNOWN` |
| `LLM_SYNTHESIS` | **`HYPOTHESIS`, `UNKNOWN` only** |

An LLM cannot produce a `FACT`. It cannot produce a `CORRELATION`. It cannot even produce an
`OBSERVATION`, because an observation implies faithful reading of a source and we have no
mechanism that verifies faithfulness.

Absence is not evidence: there is no "nothing found" record. A claim with no supporting records
renders as `UNKNOWN` plus a **coverage manifest** naming which detectors ran, at which versions,
over which range — which is what distinguishes *"we looked and found nothing"* from
*"we did not look"*.

**Alternatives considered.**
- *Label as a free field with review discipline.* Rejected: review discipline decays; a check
  constraint does not.
- *Confidence scores instead of labels.* Rejected: a number with no ground truth is false
  precision that gets quoted as though it meant something.

**Consequences.**
- (+) Charter criterion S3 becomes a static graph walk rather than a judgement call.
- (+) "LLM-based RCA with no deterministic evidence layer" becomes structurally impossible.
- (−) Some genuinely useful LLM readings of payloads are unusable as observations until a
  deterministic extractor is written for them. Accepted; the extractor is the right artifact anyway.

---

## Decision 4 — Causal claims only from paired fault-injection interventions, with divergence-aware replay

**Decision.** The system makes exactly two kinds of causal-adjacent statement, structurally
distinct in every output:

- **Association** (observational): "X preceded Y in n of m executions." The words *cause*,
  *root cause*, *due to*, *because of* are banned from output templates and lint-enforced.
- **Intervention** (paired replay): "Same task, same seed, same fixture digest, fault F toggled:
  failure rate a/n vs. b/n, difference d [CI], divergence rate v."

There is no third kind. Hypothesis candidates are generated by deterministic graph traversal
(precedence by `seq`/causal edge — **never** wall clock; data-dependency; change-point join on
deployment events; historical recurrence) and ranked by an **ordinal rubric with named criteria**,
not a learned probability. Refutation carries −4 and can sink a hypothesis.
`INSUFFICIENT_EVIDENCE` is a required output when the top two candidates are within margin.

Every hypothesis record is invalid without a `falsification_test` naming the replay that would
refute it, stated **before** the replay runs.

Replay has three modes with published fidelity contracts, and `CONTROLLED_REPLAY` carries a
mandatory **divergence detector**: when a counterfactual changes a decision, subsequent recorded
responses are off-policy, and the replay is truncated or marked `STUB` — never continued as though
valid. `divergence_rate > 0.5` ⇒ `INCONCLUSIVE`.

**Alternatives considered.**
- *LLM-generated root-cause narratives.* Rejected: the project's founding objection.
- *Learned causal ranking.* Rejected: no ground-truth labels for "actual cause" exist at any
  scale we will reach, so the model would be fitting noise and reporting it as probability.
- *Continue replay past divergence.* **Rejected as the most dangerous available shortcut**: it
  produces clean, plausible, fabricated results with no error signal (`FM-13`).

**Consequences.**
- (+) Every causal claim rests on a treatment we assigned ourselves. This is genuine interventional
  evidence, and it is more than correlation-mining over traces can deliver.
- (+) Pairing is also what makes the statistics affordable — McNemar over discordant pairs needs
  tens of trials where an unpaired comparison needs hundreds (`EVALUATION_STRATEGY` §O.5).
- (−) Many interesting counterfactuals will return `INCONCLUSIVE`. That is correct and will be
  under continuous pressure from anyone who wants a cleaner demo.
- (−) `FM-12` (false causal inference) remains **High** residual risk. Multiple-comparison
  correction across ~30 detectors is unresolved (`OQ-06`).

---

## Decision 5 — Minimum viable infrastructure, and a deterministic substrate for evaluation

**Decision.** One process. PostgreSQL. No message broker, no graph database, no vector database,
no workflow engine, no separate policy server, no frontend at Gate 0–5. Each deferral carries a
**measured trigger** (`PROJECT_CHARTER` §F), not a date and not a preference.

The reliability harness runs ~90% of trials against a `DETERMINISTIC_STUB` planner, ~7% local,
~3% remote. Results are **never pooled across substrate**.

Specifically:

- **Graph:** an in-process property graph built on demand from ledger events. A graph database
  does not fix an O(n²) edge definition; it relocates the bill. Contention edges are materialised
  as an O(n) chain with transitive reachability computed on demand.
- **Policy:** a pure-Python decision table behind an **OPA-shaped query interface**
  (`input` dict → `{allow, reasons, obligations}`), so migrating to OPA later is a swap, not a
  rewrite. Trigger: >50 rules, or non-engineer authorship, or hot policy updates.
- **Temporal / durable execution:** **not adopted.** Temporal solves durable retry and
  compensation properly, and we are not rebuilding it. The layer we are building is the one
  Temporal does not have: *deciding whether a side effect occurred when the answer is unknown*,
  and using that decision to gate the retry Temporal would otherwise perform. Our internal state
  machine exists to hold that adjudication, not to provide durability. Trigger for adopting
  Temporal: an execution must survive control-plane restart **and** the internal recovery path has
  failed a documented test.
- **MCP:** one adapter behind an internal tool protocol, at Gate 7. Binding early couples us to a
  churning spec whose version we have not verified (`OQ-07`).

**Alternatives considered.**
- *Build the full stack now for "enterprise-grade" architecture.* Rejected: explicitly named
  architectural theatre in the project constraints. Every component added now is a component whose
  failure modes must be modelled, tested and operated before it has earned its place.
- *Remote model in the evaluation loop.* Rejected on arithmetic: 21,000 calls against a 1,000/day
  budget is 21 days per sweep, per `deployment_id`.

**Consequences.**
- (+) The evaluation sweep drops from 21 days to under 2, with statistical power preserved.
- (+) The provider-unavailable path (L3) is exercised on **every CI run** rather than described in
  a document.
- (+) Fewer components means the failure taxonomy is about our design rather than about our
  dependencies.
- (−) `T-8`: single-writer `seq` ordering is correct only while the control plane is one process.
  Scaling requires Lamport clocks, and that must be designed before scaling, not after.
- (−) The stub substrate does not reproduce the *distribution of situations* a real model creates.
  Reliability is measured under a scenario distribution we authored. This belongs in every report.

---

## Decisions explicitly deferred

| Deferred | Why |
|---|---|
| Async probe with deferred adjudication (`T-1`) | Changes the safety property. Needs a measured latency problem first. |
| Multiple-comparison correction policy (`OQ-06`) | Affects what "anomaly" means; interacts with hypothesis ranking. Needs Gate 4 data. |
| CAS retention per sensitivity class (`OQ-05`) | A policy decision with an owner, not an engineering one. Blocks replay scope. |
| Probe delegation vs. central read privilege (`OQ-02`) | Depends on `OQ-01`'s findings about real tool surfaces. |
| Natural-key probing for tools without explicit probes (`OQ-03`) | Trades a missing side effect against a duplicate one. Domain policy, not engineering. |

---

## How this ADR gets falsified

This architecture should be revised, not defended, if any of the following is observed:

1. **`OQ-01` finds that >50% of real enterprise write endpoints support neither an idempotency key
   nor a postcondition probe.** Decision 2's value proposition collapses to "escalate to a human"
   and the project needs a different centre.
2. **The L3 (zero-inference) report is not compelling to a stakeholder.** There is an undeclared
   LLM dependency and Decision 3's ordering is wrong.
3. **Divergence rates in `CONTROLLED_REPLAY` routinely exceed 0.5.** Decision 4's interventional
   evidence is mostly unobtainable and the causal claim must shrink further.
4. **Stub-substrate reliability results fail to predict remote-substrate results on the 3% sample.**
   Decision 5's substrate split is measuring the wrong thing.
5. **Measured tokens-per-call (`OQ-10`) differs from the 3,000 planning assumption by more than
   2×.** Every number in `QUOTA_AND_COST_MODEL` moves and the partitioning must be re-derived.

Each of these is a test, scheduled at a named gate. An architecture with no falsification
conditions is a preference, not a design.
