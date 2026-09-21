# ARCHITECTURE SURFACE

Status: **GATE 0 — PROPOSAL UNDER ADVERSARIAL REVIEW. NOT APPROVED FOR IMPLEMENTATION.**
Document version: 0.1.0
Companion documents: `PROJECT_CHARTER.md`, `FAILURE_TAXONOMY.md`, `THREAT_MODEL.md`,
`EVALUATION_STRATEGY.md`, `QUOTA_AND_COST_MODEL.md`, `ADR/0001-initial-architecture.md`.

---

## The governing principle

```
observe  →  verify  →  analyze  →  hypothesize  →  replay  →  decide
```

Each arrow is a narrowing. `observe` produces the largest set and the weakest claims.
`decide` produces the smallest set and the strongest. **No stage may emit a claim stronger than
the stage that fed it.** This is enforced structurally: the evidence schema carries a label, and
the label is a function of the extraction method, not of anyone's confidence.

The rejected alternative — `observe → ask LLM → trust answer` — fails because it collapses all
six stages into one, discards the narrowing, and makes the strength of the final claim a
function of the model's prose style.

---

## E. Minimum architecture — the single diagram

```
  UNTRUSTED ZONE                    ║  TRUSTED RUNTIME  ║          DETERMINISTIC ANALYSIS PLANE
 ═══════════════════════════════════╬═══════════════════╬═══════════════════════════════════════
                                    ║                   ║
  ┌──────────────────┐   proposed   ║ ╔═══════════════╗ ║
  │ MODEL            │──tool call──▶║ ║   RUNTIME     ║ ║   append-only
  │ remote LLM       │              ║ ║   GATEWAY     ║ ║   ┌──────────────────────────────┐
  │ output = a       │◀──context────║ ║               ║ ║══▶│ EVIDENCE LEDGER              │
  │ PROPOSAL, never  │              ║ ║ 1 authorize   ║ ║   │  events(execution_id, seq)   │
  │ an authorization │              ║ ║ 2 budget      ║ ║   │  + CAS payload store (CID)   │
  └──────────────────┘              ║ ║ 3 idem-key    ║ ║   │  + sensitivity class per obj │
                                    ║ ║ 4 DISPATCH    ║ ║   └───────────────┬──────────────┘
  ┌──────────────────┐   result     ║ ║ 5 probe       ║ ║                   │ pure reads
  │ TOOL OUTPUT      │─────────────▶║ ║ 6 ADJUDICATE  ║ ║                   ▼
  │ untrusted data.  │              ║ ╚═══════╤═══════╝ ║   ┌──────────────────────────────┐
  │ NEVER an         │              ║         │         ║   │ DETECTOR SUITE               │
  │ instruction.     │              ║         │side     ║   │ pure fns, versioned, no LLM  │
  └────────▲─────────┘              ║         │effect   ║   └───────────────┬──────────────┘
           │                        ║         ▼         ║                   ▼
  ┌────────┴──────────────────────┐ ║                   ║   ┌──────────────────────────────┐
  │ SYNTHETIC ENTERPRISE ENV      │◀╫─────────          ║   │ EXECUTION GRAPH (in-process) │
  │  kb │ hr │ ticket │ iam │ noti│ ║                   ║   │ built on demand from ledger  │
  │  declared schemas + versions  │ ║                   ║   └───────────────┬──────────────┘
  │  ┌─────────────────────────┐  │ ║                   ║                   ▼
  │  │ FAULT INJECTOR          │  │ ║                   ║   ┌──────────────────────────────┐
  │  │ = THE INTERVENTION      │  │ ║                   ║   │ HYPOTHESIS GENERATOR         │
  │  │ seeded, per-trial       │  │ ║                   ║   │ deterministic candidates,    │
  │  └───────────▲─────────────┘  │ ║                   ║   │ ordinal rubric, ranked list, │
  └──────────────┼────────────────┘ ║                   ║   │ each with a FALSIFIER        │
                 │                  ║                   ║   └───────────────┬──────────────┘
                 │ treatment on/off ║                   ║                   │ falsifier
                 │                  ║                   ║                   ▼
  ┌──────────────┴────────────────────────────────────┐ ║   ┌──────────────────────────────┐
  │ REPLAY ENGINE                                     │◀╫───│ REPLAY REQUEST               │
  │  EXACT       — ledger-served, no external I/O     │ ║   └──────────────────────────────┘
  │  CONTROLLED  — frozen fixtures + fault toggled    │─╫──────────┐
  │  LIVE        — real calls, NO determinism claim   │ ║          │ paired trials + divergence
  │  ┌─────────────────────────────────────────────┐  │ ║          ▼
  │  │ DIVERGENCE DETECTOR                         │  │ ║   ┌──────────────────────────────┐
  │  │ truncates replay at first off-policy point  │  │ ║   │ COUNTERFACTUAL_EVIDENCE      │
  │  └─────────────────────────────────────────────┘  │ ║   │ n, effect, interval,         │
  └───────────────────────────────────────────────────┘ ║   │ divergence_rate — or refusal │
                                                        ║   └───────────────┬──────────────┘
  ┌──────────────────────────────────────────────┐      ║                   ▼
  │ RELIABILITY HARNESS  R(k, perturbation,fault)│──────╫──▶┌──────────────────────────────┐
  │ isolated env per trial, seeded, statistics   │      ║   │ EVIDENCE BUNDLER  token-capped│
  └──────────────────────────────────────────────┘      ║   └───────────────┬──────────────┘
                                                        ║          0–2 calls│ OPTIONAL
                                                        ║                   ▼
                                                        ║   ┌──────────────────────────────┐
                                                        ║   │ LLM EXPLAINER                │
                                                        ║   │ narration ONLY. Cannot emit   │
                                                        ║   │ FACT. Removable without loss │
                                                        ║   │ of any structured output.    │
                                                        ║   └──────────────────────────────┘
```

**Read the diagram as a claim about trust, not about boxes.** The double line is the only thing
that matters: everything to its left proposes, the gateway decides, everything to its right
observes what was decided. The model never crosses the line. Tool output never crosses the line.

---

## G. Trust boundaries

`DESIGN_DECISION`: the runtime gateway is the *only* enforcement point. Everything else is
either a proposer or an observer.

| # | Boundary | Left side (untrusted) | Right side (trusted) | Enforcement mechanism |
|---|---|---|---|---|
| TB-1 | **Model → Gateway** | Model output: tool name, arguments, natural language | Gateway | Schema validation (deterministic), policy decision (deterministic), budget check (deterministic). Model output is a *proposal record*, persisted as such. |
| TB-2 | **Tool output → Agent context** | Tool response bodies, retrieved documents, notification text | Agent context assembly | Tool output is tagged `provenance=UNTRUSTED_TOOL` and wrapped with a structural delimiter on ingest. No instruction extracted from it is ever executed without passing TB-1 again. Injection is *contained*, not *prevented* — see `THREAT_MODEL` §3. |
| TB-3 | **Gateway → Enterprise resource** | — | The side-effecting call itself | Identity is bound at the gateway from the execution's `actor_identity`, never from model output. Credentials are never present in model context. |
| TB-4 | **Ledger write path** | Anything upstream | Append-only ledger | Ledger is append-only by schema constraint (no UPDATE/DELETE grant to the writing role). Derived interpretations live in separate tables that reference ledger rows; they never mutate them. |
| TB-5 | **Agent plane ↔ Evaluation plane** | Agent process | Grader process | Separate OS process, separate DB credentials, no shared filesystem path, no environment variable overlap. Graders are not importable from agent code. See `EVALUATION_STRATEGY` §P. |
| TB-6 | **LLM Explainer → Evidence** | Explainer output | Evidence graph | Explainer output is written with `extraction_method=LLM_SYNTHESIS`, which the schema forbids from carrying `FACT` or `COUNTERFACTUAL_EVIDENCE`. Enforced at write, not at review. |
| TB-7 | **Replay engine → Real world** | Replay | External systems | `EXACT` and `CONTROLLED` modes run with the tool dispatcher bound to a ledger-backed or fixture-backed implementation. A network egress guard asserts zero outbound calls in those modes and fails the replay if violated. This is a test, not a convention. |

### G.1 The boundary that does not exist and must be acknowledged

There is **no** boundary between the control plane and the synthetic environment's backing store
at Gate 1–3, because the verification probe needs to read it. That means the control plane can
read enterprise state. In a real deployment this is a significant privilege and a real attack
surface. `OQ-02`: define the minimal read scope a postcondition probe requires, and whether it
can be delegated to the tool owner rather than held by the control plane.

---

## H. Canonical execution and event model

### H.1 The three-level hierarchy

```
Execution      one agent run against one task, under one deployment
  └── Step     one model turn: context → model → proposal(s)
        └── Attempt   one dispatch of one tool call, including its probe
```

`DESIGN_DECISION`: three levels, not two, because retries must be distinguishable from new
decisions. A retry is a new `Attempt` under the *same* logical action; a re-plan is a new `Step`.
Collapsing these makes `FM-02` (retry storm) and `FM-13` (loop) indistinguishable from normal
work.

`Action` is the logical unit that `Attempt`s belong to. An `Action` has one state (§I); an
`Attempt` has a transport-level result. **Multiple attempts, one action, one state.** This is
the single most important modelling choice in the system, and the one the KNOWLEDGE_GRAPH's flat
`execution → CALLS → tool` edge obscures.

### H.2 Ordering — the clock problem, confronted

`FACT`: wall-clock timestamps across processes are not a reliable order. `KNOWLEDGE_GRAPH.json`
lists `timestamp` as a minimum field and `causal_evidence_rules` states
*"temporal_precedence is necessary but insufficient"*. **If wall-clock order is unreliable, then
the one condition the causal rules declare *necessary* is unverifiable.** That is a hole in the
proposal, not a detail.

`DESIGN_DECISION`: four ordering fields, with explicit and different semantics.

| Field | Source | Semantics | Used for |
|---|---|---|---|
| `seq` | Monotonic counter, per `execution_id`, assigned by the emitting process | **Total order within an execution. Authoritative.** | All ordering, all graph edges, all precedence reasoning. |
| `causal_parent_event_id` | Explicit | Happens-before edge. Authoritative across executions. | Cross-execution precedence. |
| `t_emit` | Producer wall clock (with the producer's monotonic delta recorded) | **Advisory.** May be skewed, may go backwards. | Latency computation *within one process only*; human display. |
| `t_recv` | Ledger ingest wall clock | Authoritative for ingest ordering and retention only. | SLO on ingest lag; never for causality. |

**Hard rule:** no detector, graph edge, or hypothesis criterion may use `t_emit` to establish
precedence between events from different processes. A lint test scans detector source for this.
Where cross-process precedence is genuinely needed and no causal edge exists, the answer is
`UNKNOWN`, not a timestamp comparison.

`ASSUMPTION` (owner: architect; invalidation: control plane becomes multi-process):
a single-writer control plane makes `seq` trivially correct. When that stops being true,
`seq` needs a Lamport clock and this section must be revisited before, not after.

### H.3 Event envelope

Superset of `KNOWLEDGE_GRAPH.event_minimum_fields`. Deltas from that list are marked.

```
EventEnvelope
  # identity & order
  event_id                 uuid7                       # sortable, but seq is authoritative
  execution_id             uuid
  seq                      int64          [ADDED]      # authoritative order
  trace_id                 str                         # OTel correlation
  parent_event_id          uuid | null
  causal_parent_event_id   uuid | null    [ADDED]
  # time
  t_emit                   timestamptz                 # advisory (was: timestamp)
  t_recv                   timestamptz    [ADDED]
  monotonic_ns             int64 | null   [ADDED]      # producer monotonic, for intra-process latency
  # what
  event_type               enum                        # closed set, versioned
  schema_version           str                         # of THIS envelope
  # actors
  agent_id                 str
  actor_identity           str                         # the identity the ACTION runs as
  deployment_id            str            [ADDED]      # pins model+policy+toolset+code versions
  # subject
  provider                 str | null
  model                    str | null
  tool_name                str | null
  tool_schema_version      str | null                  # (was: schema_version, ambiguous)
  resource_id              str | null                  # HIGH CARDINALITY — attribute only, never a metric dimension
  action_id                uuid | null    [ADDED]      # logical action this attempt belongs to
  attempt_no               int            [ADDED]
  idempotency_key          str | null     [ADDED]
  # outcome
  status                   enum
  action_state             enum | null    [ADDED]      # §I state after this event
  outcome_basis            enum           [ADDED]      # TRANSPORT | PROBE | COMPENSATION | DECLARED | NONE
  latency_ms               int | null
  retry_count              int
  # payload references — NOT payloads
  input_cid                str | null                  # (was: input_hash) content id into CAS
  output_cid               str | null                  # (was: output_hash)
  input_digest             str | null                  # sha256, for equality without retrieval
  output_digest            str | null
  # governance
  policy_decision          enum | null                 # ALLOW | DENY | ALLOW_WITH_OBLIGATIONS
  policy_version           str | null     [ADDED]
  sensitivity_class        enum                        # PUBLIC | INTERNAL | CONFIDENTIAL | RESTRICTED
  provenance_ref           str | null
  redaction_applied        bool           [ADDED]
```

### H.4 Why `input_hash` alone was wrong — and what replaces it

The bootstrap and knowledge graph specify `input_hash` / `output_hash`. `FACT`: a hash supports
**equality testing** and nothing else. You cannot root-cause a failure from a hash. You cannot
build a counterfactual from a hash. You cannot show a reviewer why the agent did something from
a hash.

So a hash-only design does not solve the sensitivity problem; it *relocates* it, because the
moment root-cause analysis is actually attempted, someone will start logging payloads next to
the hashes with no controls at all.

`DESIGN_DECISION`: split the concern explicitly.
- **Envelope** carries `*_digest` (sha256) — cheap, non-sensitive, sufficient for equality,
  duplicate detection, cache keys, and drift detection.
- **Content-addressed store (CAS)** holds the payload under `*_cid`, with its own
  `sensitivity_class`, its own retention policy, its own access control, and its own redaction
  record. Separate table, separate grants, separately droppable.
- **Deleting the CAS must leave the ledger valid and every detector functional.** This is a
  test: run the full detector suite with the CAS emptied; only payload-inspecting detectors may
  degrade, and they must degrade to `UNKNOWN`, not crash and not silently pass.

That property — *the system still works with all payloads deleted* — is what makes the privacy
story real rather than aspirational.

---

## I. Execution state machine

`DESIGN_DECISION`: states are exactly those in `KNOWLEDGE_GRAPH.execution_state_machine`. No
states invented. The contribution is the *transition rules* and the *guards*, which the
knowledge graph does not specify.

State belongs to an **Action**, not to an attempt and not to an execution.

```
                            ┌──────────┐
                            │ PLANNED  │  model proposed a tool call
                            └────┬─────┘
              policy DENY / budget exhausted / schema invalid
                    ┌────────────┼────────────────────────────┐
                    ▼            ▼ policy ALLOW                │
              ┌──────────┐  ┌────────────┐                     │
              │ ABORTED  │  │ AUTHORIZED │                     │
              │ terminal │  └─────┬──────┘                     │
              └──────────┘        │ gateway sends request      │
                    ▲             ▼                            │
                    │       ┌────────────┐                     │
                    │       │ DISPATCHED │ ◀───────────────┐   │
                    │       └─────┬──────┘   retry (only   │   │
                    │             │          if RETRYABLE, │   │
         ┌──────────┼─────────────┼──────────┐  see §J)    │   │
         │          │             │          │             │   │
  transport 2xx  transport 4xx/5xx    timeout / conn reset │   │
         │          │             │          │             │   │
         ▼          │             │          ▼             │   │
  ┌─────────────┐   │             │   ┌────────────────┐   │   │
  │ACKNOWLEDGED │   │             │   │ UNKNOWN_OUTCOME│───┘   │
  │ tool CLAIMS │   │             │   │  FIRST CLASS.  │       │
  │ success.    │   │             │   │  MAY BE        │       │
  │ NOT proof.  │   │             │   │  TERMINAL.     │       │
  └──────┬──────┘   │             │   └───┬────────┬───┘       │
         │          │             │       │probe   │ no probe  │
         │ probe    │             │       │        │ available │
         ▼          ▼             ▼       ▼        ▼           │
  ┌──────────────────────┐  ┌──────────────────┐  ┌──────────┐ │
  │  VERIFIED_SUCCESS    │  │ VERIFIED_FAILURE │  │ ESCALATE │─┘
  │  postcondition holds │  │ postcond absent  │  │ (human)  │
  │  terminal            │  │ AND outside      │  │ stays    │
  └──────────────────────┘  │ consistency wnd  │  │ UNKNOWN  │
                            └────────┬─────────┘  └──────────┘
                                     │ compensation available
                                     ▼
                            ┌────────────────┐
                            │  COMPENSATED   │ terminal
                            └────────────────┘
```

### I.1 The guards that make this non-trivial

| Transition | Guard | Rationale |
|---|---|---|
| `PLANNED → AUTHORIZED` | Policy `ALLOW`, budget available, arguments validate against `tool_schema_version`, identity resolved. All deterministic. **No LLM.** | Rule 5 of the project constraints. |
| `PLANNED → ABORTED` | Policy `DENY`, or budget exhausted, or schema invalid. `abort_reason` is a closed enum. | The knowledge graph has no `DENIED` state; `ABORTED` + reason covers it without inventing one. |
| `DISPATCHED → ACKNOWLEDGED` | Transport-level success only. **`outcome_basis=TRANSPORT`.** | A 2xx is the tool's *claim*. Claims are evidence, not proof. This is why `ACKNOWLEDGED` exists as a distinct state from `VERIFIED_SUCCESS` — and it is the state most systems wrongly treat as terminal success. |
| `ACKNOWLEDGED → VERIFIED_SUCCESS` | Postcondition probe returns the expected state. **`outcome_basis=PROBE`.** | See §I.2 for the case where no probe exists. |
| `DISPATCHED → UNKNOWN_OUTCOME` | Timeout, connection reset, or any condition where the request may or may not have been received. | `FACT`: a timeout is not evidence of non-occurrence. Project rule 7. |
| `UNKNOWN_OUTCOME → DISPATCHED` (retry) | **Only if** the capability matrix (§J) returns `RETRY_SAFE`. | This guard is the product. |
| `UNKNOWN_OUTCOME → ESCALATE` | Retry not safe and no probe available. | Terminal for the automated system. Correct answer, small product, see `OQ-01`. |
| `* → VERIFIED_FAILURE` | Probe confirms absence **and** the consistency window has elapsed. | See §I.2. |

### I.2 The consistency-window trap — the safety mechanism that creates the bug

**This is the most dangerous flaw in the naive version of this design and it must be fixed
before Gate 2.**

The proposal says: on timeout, probe; if the postcondition does not hold, it is safe to retry.

`FACT`: many real enterprise APIs are **eventually consistent**. A write commits to a primary;
a read served from a replica does not see it for some window. Therefore:

1. `ticket.create` commits successfully.
2. Response is lost → `UNKNOWN_OUTCOME`.
3. Probe runs immediately, hits a replica, returns "not found".
4. System concludes `VERIFIED_FAILURE` and retries.
5. **Duplicate side effect — caused by the mechanism we built to prevent duplicate side effects.**

This is strictly worse than not probing at all, because it converts an honest `UNKNOWN` into a
confident wrong answer.

`DESIGN_DECISION` — the fix, which is mandatory, not optional:

- Every tool **must declare** `consistency_model ∈ {STRONG, BOUNDED_STALENESS, EVENTUAL, UNDECLARED}`
  and, if bounded, a `staleness_bound_ms`.
- A probe returning "absent" is interpreted as:

  | consistency_model | elapsed < bound | elapsed ≥ bound |
  |---|---|---|
  | `STRONG` | `VERIFIED_FAILURE` | `VERIFIED_FAILURE` |
  | `BOUNDED_STALENESS` | **`UNKNOWN_OUTCOME` (re-probe after bound)** | `VERIFIED_FAILURE` |
  | `EVENTUAL` | `UNKNOWN_OUTCOME` | **`UNKNOWN_OUTCOME` — never `VERIFIED_FAILURE` on absence alone** |
  | `UNDECLARED` | `UNKNOWN_OUTCOME` | `UNKNOWN_OUTCOME` |

- `UNDECLARED` is the default. A tool that has not declared its consistency model can never
  produce `VERIFIED_FAILURE` from a negative probe. This makes the safe configuration the lazy
  one, which is the only kind of safe default that survives contact with engineers.
- Probes are budgeted: `max_probe_attempts`, `probe_backoff`, and a hard deadline. Exhausting
  the probe budget yields `ESCALATE`, never a guess.

`OPEN_QUESTION` `OQ-03`: under `EVENTUAL` with no idempotency key, a negative probe is
permanently uninformative, so the action is permanently `UNKNOWN`. Is a "probe by natural key
with a wide time window and fuzzy match" acceptable, given it can produce false positives
(matching a *different* record)? A false positive here suppresses a needed retry — a *missing*
side effect rather than a duplicate. Which error is worse is a domain policy question, not an
engineering one. It must be configurable per tool and the default must be stated.

### I.3 What the state machine does NOT do

It does not make the execution correct. An action can be `VERIFIED_SUCCESS` for a ticket that
should never have been created. Outcome adjudication answers *"did it happen exactly once?"*,
never *"should it have happened?"*. The second question belongs to policy (before) and
evaluation (after). Conflating them is a category error and inflates what the system claims.

---

## J. Non-atomic tool calls — the capability matrix

`DESIGN_DECISION`: retry legality is a **lookup in a decision table**, not a judgement.

### J.1 Declared tool properties

Each tool declares, in its contract:

| Property | Type | Verified by |
|---|---|---|
| `is_read_only` | bool | Contract test: call twice, assert backing store digest unchanged. |
| `accepts_idempotency_key` | bool | Contract test: same key twice → one effect, second returns the first result. |
| `has_postcondition_probe` | bool | Contract test: probe exists, is read-only, and detects a known write. |
| `has_compensation` | bool | Contract test: compensation restores the pre-state digest. |
| `consistency_model` | enum | Contract test: write-then-immediate-read behaviour matches the declaration. |

**A declared property that fails its contract test fails the build.** A tool cannot lie about
itself in a way that survives CI. This is what converts the capability matrix from documentation
into a mechanism.

### J.2 The decision table

Input: action state `UNKNOWN_OUTCOME`. Output: the only legal next transition.

| `is_read_only` | `accepts_idem_key` | `has_probe` | Verdict | Next |
|---|---|---|---|---|
| true | — | — | `RETRY_SAFE` | Retry immediately. Read-only ⇒ no side effect to duplicate. |
| false | true | — | `RETRY_SAFE` | Retry with the **same** key. Server deduplicates. |
| false | false | true | `PROBE_THEN_DECIDE` | Probe under §I.2 rules. Retry only on `VERIFIED_FAILURE`. |
| false | false | false | **`BLIND_WRITE — NEVER RETRY`** | `ESCALATE`. Terminal for automation. |

The bottom row is the point of the whole exercise. A non-idempotent, unverifiable write that
timed out is **irrecoverable by any automated means**, and every system that retries it is
generating duplicates it cannot detect. Naming that row, and refusing to retry it, is worth more
than the rest of the analysis plane combined.

### J.3 Idempotency key derivation

`idempotency_key = sha256(execution_id ‖ action_id ‖ canonical_json(arguments) ‖ tool_schema_version)`

`DESIGN_DECISION`: derived, not random, so it survives control-plane restart and is stable across
attempts. `action_id` (not `attempt_no`) is included so retries share a key. `tool_schema_version`
is included so a schema change deliberately produces a *different* key — silently reusing a key
across a schema change is a correctness hazard.

`OPEN_QUESTION` `OQ-04`: key stability across *replay*. In `CONTROLLED_REPLAY` the same key
would be derived. If a replay ever touched a real system it would be deduplicated against the
original — which is accidentally safe, but relies on the server's key retention window. Since
TB-7 forbids external I/O in replay modes anyway, this is currently moot; it stops being moot the
moment `LIVE_REEXECUTION` runs against a shared environment. Must be resolved before Gate 5.

### J.4 Compensation

Compensation is offered, never assumed. `has_compensation=true` obliges the tool to supply a
compensating operation whose contract test restores the pre-state digest. Compensation is itself
a side-effecting action: it gets its own `action_id`, its own state machine instance, and can
itself end in `UNKNOWN_OUTCOME`. **There is no bottom.** The recursion terminates at `ESCALATE`
with a bounded depth of 1 — we do not compensate a failed compensation automatically.

---

## K. Rate limits, retries, and runaway loops

Full arithmetic in `QUOTA_AND_COST_MODEL.md`. Mechanisms here.

### K.1 Budgets are per-execution and hierarchical

`DESIGN_DECISION`: the single most common cause of retry storms is that retry counters are
per-call. Three nested calls with three retries each is nine calls, and nobody wrote "9"
anywhere.

```
ExecutionBudget            # allocated once, decremented by everything beneath
  max_steps                # hard ceiling on model turns
  max_actions              # hard ceiling on tool dispatches, retries INCLUDED
  max_retries_total        # shared pool across ALL actions in this execution
  max_tokens_total
  max_wall_clock_ms
  max_probe_attempts_total
  max_cost_units
```

Exhausting any budget transitions the execution to `ABORTED` with the exhausted budget named.
Budget exhaustion is a **normal, expected, reported outcome**, not an error — the harness counts
it as a distinct failure class (`FM-02`).

### K.2 Rate limiting

- **Local token bucket first.** Per `(provider, model, credential)`, enforced *before* dispatch.
  Never discover a limit by being rejected.
- **Reconcile from response headers.** Parse remaining/reset/limit headers; treat the local
  bucket as an estimate that the authoritative headers correct. `ASSERTED_UNVERIFIED`: specific
  header names per provider must be read from the live response, not hard-coded from docs
  (project rule: do not hard-code provider limits).
- **On 429: no retry against the same bucket until the reset.** A 429 retry is the mechanism by
  which one rate limit becomes an outage (`FM-03`).
- **Two-level circuit breaker.** Per-provider (open on sustained 429/5xx) and global (open on
  aggregate error rate). Open breaker → degradation ladder (§S).

### K.3 Runaway-loop detection — three independent detectors

A loop is not one thing. Three orthogonal detectors, all deterministic, all pure functions over
the ledger:

| Detector | Signal | Threshold |
|---|---|---|
| `D-LOOP-REPEAT` | Same `(tool_name, canonical_args_digest)` dispatched ≥ N times within one execution | N configurable, default 3 |
| `D-LOOP-NOPROGRESS` | `state_fingerprint` (digest of the agent's accumulated task state) unchanged across M consecutive steps, while actions were dispatched | M default 4 |
| `D-LOOP-OSCILLATE` | Cycle in the action→state-transition sequence: A→B→A→B with period ≤ P | P default 2, ≥ 2 cycles |

`DESIGN_DECISION`: three detectors because the failure modes are genuinely different. Repetition
catches a stuck tool. No-progress catches a model that is "working" but achieving nothing.
Oscillation catches two subsystems fighting. A single "loop detector" misses two of the three.

`ASSUMPTION` (owner: architect; invalidation: a legitimate task requires > 3 identical calls):
identical repeated calls indicate a fault rather than legitimate work. Batch or polling
workloads violate this. Mitigation: exempt tools may declare `polling=true`, which swaps
`D-LOOP-REPEAT` for a rate-based variant. Do not let the exemption become the default.

---

## L. Evidence representation

### L.1 The evidence record

```
Evidence
  evidence_id        uuid
  claim_id           uuid                 # the statement this supports or refutes
  label              enum                 # FACT | OBSERVATION | CORRELATION |
                                          # HYPOTHESIS | COUNTERFACTUAL_EVIDENCE | UNKNOWN
  polarity           enum                 # SUPPORTS | REFUTES | INCONCLUSIVE
  extraction_method  enum                 # DETERMINISTIC_RULE | STATISTICAL_TEST |
                                          # REPLAY_EXPERIMENT | HUMAN_ASSERTION | LLM_SYNTHESIS
  producer_id        str                  # detector or experiment id
  producer_version   str                  # semver. REQUIRED.
  source_refs        [ledger_ref]         # immutable (execution_id, seq) pairs. REQUIRED, non-empty.
  params             json                 # thresholds/config in effect. REQUIRED.
  computed_at        timestamptz
  reproducible       bool                 # can this be recomputed from source_refs alone?
```

### L.2 The label lattice — enforced at write time

`DESIGN_DECISION`: the permissible label is a **function of the extraction method**. Not a
choice. Enforced by a database check constraint and a schema validator, so violating it is not
possible without a migration and a code review.

| `extraction_method` | Permitted labels | Forbidden |
|---|---|---|
| `DETERMINISTIC_RULE` | `FACT`, `OBSERVATION`, `UNKNOWN` | everything else |
| `STATISTICAL_TEST` | `CORRELATION`, `OBSERVATION`, `UNKNOWN` | **`FACT`**, `COUNTERFACTUAL_EVIDENCE` |
| `REPLAY_EXPERIMENT` | `COUNTERFACTUAL_EVIDENCE`, `OBSERVATION`, `UNKNOWN` | `FACT` |
| `HUMAN_ASSERTION` | `HYPOTHESIS`, `OBSERVATION`, `UNKNOWN` | `FACT` |
| `LLM_SYNTHESIS` | `HYPOTHESIS`, `UNKNOWN` | **`FACT`**, `CORRELATION`, `COUNTERFACTUAL_EVIDENCE`, `OBSERVATION` |

Consequences worth stating plainly:
- **A statistical test can never produce a `FACT`.** p < 0.05 is a correlation with a number on it.
- **A replay experiment can never produce a `FACT`.** It produces counterfactual evidence bounded
  by the replay's fidelity contract (§N).
- **An LLM can only ever produce a `HYPOTHESIS`.** It cannot even produce an `OBSERVATION`,
  because an observation implies faithful reading of a source, and we have no mechanism that
  verifies faithfulness. If we want the LLM's reading of a payload as an observation, a
  deterministic extractor must produce it instead.

### L.3 Absence of evidence

`KNOWLEDGE_GRAPH.causal_evidence_rules` states: *"absence of evidence must not be represented as
evidence of absence."* Enforced structurally: there is no "no evidence found" evidence record.
A claim with zero supporting records is rendered as `UNKNOWN` with `coverage` metadata naming
which detectors ran, at which versions, over which ledger range. That metadata is what lets a
reader distinguish *"we looked and found nothing"* from *"we did not look"* — and those are
different states that a naive implementation renders identically.

---

## M. Causal hypotheses without pretending correlation is causation

### M.1 The honest position

`FACT`: from observational traces alone, causal identification is not possible here. There is no
randomisation of the treatment, no instrument, no natural experiment, and the confounders
(deployment version, task difficulty, time of day, provider load, prompt content) are
unmeasured and numerous.

`FACT`: what we *do* have is randomisation over **injected faults**, because we assign the
treatment ourselves, per trial, from a seeded RNG.

`DESIGN_DECISION`: the system makes exactly two kinds of causal-adjacent statement, and they are
typographically and structurally distinct in every output.

| Statement type | Source | Permitted phrasing | Forbidden phrasing |
|---|---|---|---|
| **Association** | Observational ledger analysis | "X preceded Y in n of m executions"; "X and Y co-occur at rate r" | "X caused Y"; "root cause"; "due to"; "because of" |
| **Intervention** | Paired replay trials with the fault toggled | "With F injected, failure rate was a/n; with F removed under otherwise identical conditions, b/n. Difference d [CI]. Divergence rate v." | "F is the root cause"; any statement omitting n, CI, or divergence rate |

**There is no third kind.** The phrase "root cause" does not appear in any system output. A
lint test over output templates enforces this, because the word will otherwise reappear the
first time someone writes a summary.

### M.2 Candidate generation — deterministic, not learned

Hypothesis candidates are generated by graph traversal, not by a model.

1. **Seed**: the anomaly(ies) a detector fired on. Deterministic.
2. **Backward reachability**: traverse the execution graph backwards along `causal_parent`,
   `data_dependency` (output CID of A appears in input of B), and `resource_contention`
   (same `resource_id`) edges, bounded by depth and by `seq` precedence.
3. **Change-point join**: intersect the reachable set with `deployment` change events
   (model version, tool schema version, policy version) within the window.
   `KNOWLEDGE_GRAPH` models `deployment CHANGES model/schema/policy` — **this is the highest-value
   edge in the graph and it should be treated as the primary spine of hypothesis generation**,
   because a deployment is the closest thing to a naturally-occurring intervention in
   observational data.
4. **Recurrence join**: count prior executions where the same candidate pattern preceded the same
   anomaly class.
5. **Dedupe and rank.**

### M.3 Ranking — an ordinal rubric, not a probability

`DESIGN_DECISION`: the rank is a transparent sum over named, individually-inspectable criteria.
It is **not** a probability and is never presented as one.

| Criterion | Points | Justification |
|---|---|---|
| Logical precedence established (`seq` / causal edge — **not** wall clock) | +1 | Necessary, per the project's own rules. Never sufficient. |
| Direct data dependency (candidate's output CID appears in the failing input) | +2 | Mechanistic linkage, not mere co-occurrence. |
| Change-point alignment (a deployment changed a relevant version in-window) | +2 | Nearest available natural experiment. |
| Historical recurrence (≥ 3 prior co-occurrences) | +1 | Weak. Repeated correlation is still correlation. |
| Counterfactual replay shows effect | +4 | The only genuine intervention. Dominates. |
| Counterfactual replay shows **no** effect | −4 | **Refutation must be able to sink a hypothesis.** |
| Alternative candidate explains the same evidence equally well | −1 per alternative | Penalise non-discrimination explicitly. |

**Why not a learned model or a probability?** Because we have no labelled ground truth for "the
actual cause" and never will at this scale. A number produced by a model with no ground truth is
a false precision that will be quoted in a review as though it meant something. An ordinal sum
over named criteria is auditable: a reader can disagree with the +2 and recompute.

`ASSUMPTION` (owner: architect; invalidation: two criteria are shown to be near-perfectly
correlated across ≥ 100 incidents): the criteria are sufficiently independent that summing them
is meaningful. If precedence and data-dependency turn out to be the same signal in practice, the
weights are double-counting and must be collapsed.

### M.4 Every hypothesis carries its own falsifier

`DESIGN_DECISION`: a hypothesis record is invalid without a `falsification_test` field
specifying the concrete replay configuration that would refute it — mode, fault toggled, n
trials, and the outcome that would count as refutation. Stated before the test is run.

This does three things: it forces the hypothesis to be about a *mechanism* rather than a vibe;
it prevents post-hoc reinterpretation of whatever the replay returns; and it makes "this
hypothesis is untestable with our current harness" an explicit, visible outcome rather than a
silent one.

### M.5 The permitted refusal

`"INSUFFICIENT_EVIDENCE"` is a first-class, frequently-correct output. The hypothesis ranker is
required to emit it when the top candidate's score is below a threshold or when the top two
candidates are within a configured margin. A ranked list that always has a winner is a ranked
list that is lying on hard cases — and hard cases are the only ones anyone needs this for.

---

## N. Replay modes — three different contracts

`FACT`: project rule 9 — never claim live LLM replay is deterministic. The three modes exist to
make the *strength of claim* explicit, so the word "replay" cannot smuggle a determinism
assumption.

| | `EXACT_REPLAY` | `CONTROLLED_REPLAY` | `LIVE_REEXECUTION` |
|---|---|---|---|
| **Model calls** | Served from ledger. Zero inference. | Served from ledger, or a seeded deterministic stub. Zero remote inference. | Real. Current model. |
| **Tool calls** | Served from ledger. | Executed against **frozen fixtures** — a synthetic env snapshot restored from a digest. | Real tools. |
| **External I/O** | **Zero.** Asserted by an egress guard; violation fails the replay. | **Zero.** Same guard. | Real. |
| **Deterministic?** | Yes, bit-for-bit, for the analysis code path. | Yes, given the same seed and fixture digest. | **No. Never claimed.** |
| **Can inputs be changed?** | No. It is a re-projection of what happened. | **Yes — this is the point.** Toggle a fault, change a tool response, change a policy version. | Yes, but confounded with model drift. |
| **Permitted claim** | "Our detectors/graph/analysis produce X over this trace." | "Under this fixture and seed, toggling F changed the outcome from A to B in k/n trials, with divergence rate v." | "In n live runs today, the outcome distribution was D. Not comparable to any other date." |
| **Forbidden claim** | "The agent would do this again." | "F causes this failure in general." | "This reproduces the incident." |
| **Cost** | Zero inference. | Zero remote inference (stub) or bounded (recorded). | Full. Budget-governed. See §S. |

### N.1 Trajectory divergence — the flaw most replay systems hide

**This is the second-most-important correctness issue in the design.**

`CONTROLLED_REPLAY` works by serving recorded responses. But a counterfactual *changes* the
inputs. The moment the change alters a decision the agent makes, every subsequent recorded
response is **off-policy**: it was recorded in response to a different request, in a different
state. Continuing to serve it produces a trajectory that never occurred and could never occur.

Naively, the replay looks like it succeeded. The output is plausible. It is fiction.

`DESIGN_DECISION` — the **divergence detector**, mandatory:

- At every replay decision point, compute `request_digest` and compare with the recorded one.
- **Match** → serve the recorded response. Replay remains valid.
- **Mismatch** → the trajectory has diverged. Then:
  - if a seeded deterministic stub model is in use, continue and mark
    `divergence_from_seq = s`, `post_divergence_fidelity = STUB`;
  - if only recorded responses are available, **halt the replay at `s`** and report
    `TRUNCATED_AT_DIVERGENCE`. Do not fabricate.
- Every `COUNTERFACTUAL_EVIDENCE` record carries `divergence_rate` = fraction of trials that
  diverged before reaching the outcome under test.
- **A counterfactual with a high divergence rate is reported as weak or invalid, not as a
  result.** Threshold configured, default: > 0.5 ⇒ `INCONCLUSIVE`.

This is uncomfortable because it means many interesting counterfactuals will return
`INCONCLUSIVE`. That is the correct answer, and a system that returns a confident answer there
is producing exactly the fake root-cause analysis this project exists to replace.

### N.2 What `CONTROLLED_REPLAY` requires of the environment

Fixtures are not optional detail. `CONTROLLED_REPLAY` requires the synthetic environment to
support snapshot-restore by digest, so every trial starts from a byte-identical state. Without
it, paired trials are not paired and the counterfactual is confounded by state carry-over
(`FM-16`). This is a Gate 1 requirement, not a Gate 5 one — the environment must be built
snapshot-capable from the start, because retrofitting it is a rewrite.

---

## S. Operating without an LLM provider

Full treatment in `QUOTA_AND_COST_MODEL.md` §S. Architectural summary:

**`FACT` (by construction, and tested at criterion S4):** every stage of
`observe → verify → analyze → hypothesize → replay → decide` is deterministic. The LLM appears
exactly once, at the end, as narration over a bundle that is already complete.

The degradation ladder:

| Level | Condition | Behaviour |
|---|---|---|
| L0 | Primary provider healthy | Up to 2 calls per incident for narration. |
| L1 | Primary rate-limited or degraded | Fallback provider, **with a recorded `provider_switch` event** — never silent. Behavioural guarantees differ; output is marked. |
| L2 | All remote providers unavailable | Local small model if configured, else skip to L3. |
| L3 | No inference at all | **Template-rendered structured report.** All evidence, all hypotheses, all rankings, all counterfactuals, all state adjudications present. Only the prose paragraph is absent. |

**The system's value proposition must survive at L3.** If a stakeholder demo is not compelling
at L3, the architecture has an LLM dependency it has not admitted to, and that is a finding, not
a polish item.

---

## T. What makes this fail at enterprise scale

Honest list. Each is a real limit of *this* design, not a generic scaling homily.

| # | Failure at scale | Why | Earliest signal | What would have to change |
|---|---|---|---|---|
| T-1 | **Inline probes double tool latency** | The verification probe sits on the critical path of every side-effecting call. An agent doing 20 writes pays 20 extra round trips. | p95 action latency in the harness. | Async probe with a deferred adjudication queue — which means the agent proceeds under `ACKNOWLEDGED` and the verdict arrives later. That is a *different* and weaker safety property, and the trade must be stated, not slid in. |
| T-2 | **`resource_id` cardinality** | Unbounded. If it ever becomes a metric dimension, the metrics backend dies. | Metric series count. | Enforced now: `resource_id` is an event attribute only. A lint test forbids it in metric label sets. Cheap to enforce at Gate 1, near-impossible to retrofit. |
| T-3 | **Ledger write amplification** | Every attempt, probe and state transition is a row. A single execution can emit 200+ events. At 10^4 executions/day that is 2×10^6 rows/day on Postgres. | Ingest lag (`t_recv − t_emit`) p99. | Batch ingest, partition by day, then the ClickHouse trigger in `CHARTER` §F. |
| T-4 | **Graph explosion on long executions** | `resource_contention` edges are O(n²) in actions touching the same resource. A 500-action execution on one resource is 125k edges. | Graph build time p95. | Edge-type budgets and lazy materialisation before a graph DB. A graph DB does not fix an O(n²) edge definition — it just makes the explosion someone else's storage bill. |
| T-5 | **CAS payload volume and its retention obligation** | Full prompt/response retention is the largest storage cost *and* the largest privacy liability simultaneously. | CAS bytes/day; retention policy age. | Sampling + class-based retention + the "delete the CAS, system still works" property (§H.4). |
| T-6 | **Replay storage** | Replay needs payloads. Payload retention is T-5. Replay fidelity and privacy are in **direct tension** and this cannot be engineered away — only decided. | Replayable-fraction of incidents older than the retention window. | A stated policy: which incident classes keep payloads, for how long, under whose approval. Currently `OQ-05`. |
| T-7 | **Control-plane read access to enterprise state** | Probes require read access to the resources tools write. Aggregated across an enterprise this is an extraordinarily privileged account. | — (design-time) | Delegate probes to tool owners (probe-as-a-service), or accept the privilege and protect it accordingly. `OQ-02`. |
| T-8 | **Single-writer `seq`** | H.2's ordering guarantee assumes one writer per execution. Horizontal scaling breaks it. | Concurrent-writer errors. | Lamport clocks, designed before scaling, not after. |
| T-9 | **Policy engine as a synchronous SPOF** | Every action blocks on it. | Policy decision latency p99; breaker trips. | Local decision cache with bounded staleness — and a bounded-staleness *authorisation* decision is a security trade-off requiring explicit sign-off, not a performance tweak. |
| T-10 | **Reliability eval cost grows multiplicatively** | k × perturbations × fault-intensities × tasks. Exceeds any inference budget almost immediately (see `QUOTA_AND_COST_MODEL` §3). | Trials/day achieved vs. required. | Deterministic model substrate for the bulk of trials, remote model on a sampled subset, with results reported **per substrate** and never pooled. |
| T-11 | **Detector version drift across a long ledger** | Evidence carries `producer_version`, so a two-year-old incident was analysed by a detector that no longer exists. Re-analysis changes historical conclusions. | Version spread in the evidence table. | Detector versions are immutable and retained; re-analysis creates *new* evidence rows and never mutates old ones. Comparisons across versions must be explicitly opted into. |
| T-12 | **Every claim is scoped to a synthetic environment** | The deepest limit. Fault distributions, consistency models, and API behaviours in the synthetic env were chosen by us. Reliability measured against our own assumptions is circular. | — (structural) | Nothing, within this project. The honest mitigation is the production-claim paragraph in `PROJECT_CHARTER.md` and never weakening it. |
