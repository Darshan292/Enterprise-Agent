# ARCHITECTURE SURFACE

Status: **GATE 0.5 — PIVOTED PROPOSAL UNDER REVIEW. NOT APPROVED FOR IMPLEMENTATION.**
Document version: 0.2.0 (full rewrite; supersedes 0.1.0)

Companions: `PROJECT_CHARTER.md`, `INTERACTION_MODEL.md`, `REGRESSION_ARTIFACT_SPEC.md`,
`FAILURE_TAXONOMY.md`, `THREAT_MODEL.md`, `EVALUATION_STRATEGY.md`, `QUOTA_AND_COST_MODEL.md`,
`COMPETITIVE_OVERLAP.md`, `ADR/0001-initial-architecture.md`.

---

## The governing loop

```
  model factors → plan experiments → run trials → evaluate oracles
       ↑                                                 ↓
       └── compile regression ← fingerprint ← reduce ← detect violation
                                      ↓
                              estimate interaction (designed follow-up only)
```

The agent is **inside** the loop as the system under test. It is never the thing that runs the
loop. An LLM appears at exactly one optional point: narrating a completed bundle.

---

## E. Minimum architecture — the single diagram

```
   SYSTEM UNDER TEST          ║  CONTROLLED BOUNDARY  ║      EXPERIMENT PLANE (deterministic)
 ══════════════════════════════╬══════════════════════╬════════════════════════════════════════
                               ║                      ║
  ┌─────────────────────────┐  ║ ╔══════════════════╗ ║  ┌──────────────────────────────────┐
  │ AGENT UNDER TEST        │  ║ ║ EXECUTION        ║ ║  │ FACTOR MODEL           (C1)      │
  │  planner + model        │──╫▶║ ADAPTER          ║ ║  │  factors, levels, constraints    │
  │  model_config is a      │  ║ ║                  ║ ║  └────────────────┬─────────────────┘
  │  FACTOR, not a fixture  │◀─╫─║ holds the ONLY   ║ ║                   ▼
  └─────────────────────────┘  ║ ║ DispatchCapability║ ║ ┌──────────────────────────────────┐
                               ║ ║                  ║ ║  │ EXPERIMENT PLANNER     (C2)      │
  ┌─────────────────────────┐  ║ ║ 1 authorize      ║ ║  │  covering arrays (t=2, then 3)   │
  │ SYNTHETIC ENVIRONMENT   │◀─╫─║ 2 budget         ║ ║  │  constraint-aware generation     │
  │  kb│hr│ticket│iam│notify│  ║ ║ 3 effect_id      ║ ║  │  state-machine sequences         │
  │                         │  ║ ║ 4 dispatch       ║ ║  │  metamorphic relations           │
  │  CAPABILITY PROFILE     │  ║ ║ 5 reconcile      ║ ║  │  budget-aware schedule           │
  │  CP1..CP8 IS A FACTOR   │  ║ ║ 6 adjudicate     ║ ║  └────────────────┬─────────────────┘
  │                         │  ║ ╚═════════╤════════╝ ║           ┌───────┴────────┐
  │  ┌───────────────────┐  │  ║           │          ║           ▼        (C3)    ▼
  │  │ GROUND-TRUTH      │  │  ║           │ events   ║  ┌────────────────┐ ┌──────────────┐
  │  │ EFFECT LOG        │  │  ║           ▼          ║  │ RUNNER +       │ │ ADAPTIVE     │
  │  │ agent cannot read │  │  ║  ┌──────────────────┐║  │ FAULT INJECTOR │ │ EXPANSION    │
  │  └─────────┬─────────┘  │  ║  │ CANONICAL EVENT  │║  │ seeded, 1 trial│ │ Hamming d=1  │
  │            │            │  ║  │ LEDGER           │║  └───────┬────────┘ └──────────────┘
  │  ┌─────────▼─────────┐  │  ║  │ append-only      │║          │ trial records
  │  │ VERSIONED FIXTURE │◀─╫──╫──│ (execution, seq) │║          ▼
  │  │ STORE             │  │  ║  └────────┬─────────┘║  ┌──────────────────────────────────┐
  │  │ snapshot/restore  │  │  ║           │          ║  │ ORACLE ENGINE          (C4)      │
  │  │ BY DIGEST         │  │  ║           └──────────╫─▶│ INV-1..INV-9, versioned, pure    │
  │  └───────────────────┘  │  ║                      ║  │ holds the only OracleCapability  │
  └─────────────────────────┘  ║                      ║  └────────────────┬─────────────────┘
                               ║                      ║          VIOLATED │
  ┌──────────────────────────────────────────────┐    ║                   ▼
  │ FAILURE REDUCER               (C6)           │    ║  ┌──────────────────────────────────┐
  │  ddmin + r_confirm + fingerprint-aware       │◀───╫──│ FAILURE RECORD                   │
  │  accept. CONTROLLED_REPLAY only.             │    ║  └──────────────────────────────────┘
  │  REDUCTION_UNSTABLE is a terminal state.     │    ║
  └───────────────────┬──────────────────────────┘    ║
                      ▼                                ║
  ┌──────────────────────────────┐   ┌─────────────────────────────────────────────────┐
  │ FINGERPRINT + CLUSTER  (C7)  │──▶│ INTERACTION ANALYZER              (C5)          │
  │ stability metric, not truth  │   │ ONLY Phase-3 designed factorials.               │
  └───────────────┬──────────────┘   │ Contingency + logistic interaction + BH-FDR.    │
                  │                  │ Covering-array data CANNOT feed this.           │
                  ▼                  └─────────────────────────────────────────────────┘
  ┌──────────────────────────────┐
  │ REGRESSION COMPILER    (C8)  │──▶  regression artifact: manifest, seed, levels,
  │ reproduction_rate + interval │     fixture digest, oracle, replay mode, limitations
  └───────────────┬──────────────┘
                  ▼
  ┌──────────────────────────────┐        ┌──────────────────────────────────────────┐
  │ COVERAGE ACCOUNTING    (C9)  │───────▶│ OPTIONAL LLM NARRATOR                    │
  │ reports the UNEXPLORED space │        │ pure fn (EvidenceBundle) -> str.         │
  └──────────────────────────────┘        │ Holds NO capability token, therefore     │
                                          │ CANNOT construct an ObservedEvent, an    │
                                          │ OracleVerdict, or a ReductionStep.       │
                                          │ Delete it: structured output unchanged.  │
                                          └──────────────────────────────────────────┘
```

**Read the diagram as a claim about capability, not about boxes.** The narrator is not
*forbidden* from producing an oracle verdict; it is *unable* to, because it is never handed an
`OracleCapability`. That is the correction from 0.1.0, where the same guarantee was a linter.

---

## G. Trust boundaries and capability isolation

`DESIGN_DECISION` (pivot correction 8): the primary control is **typed capability tokens**, not
source lint. A capability is an unforgeable construction token held by exactly one component.

| Capability | Held by | Gates construction of | Consequence of not holding it |
|---|---|---|---|
| `DispatchCapability` | Execution adapter only | `DispatchHandle` — the only path to a side effect | No component can reach the environment |
| `LedgerReadCapability` | Ledger reader only | `ObservedEvent` | **The narrator cannot manufacture an observation** |
| `OracleCapability` | Oracle engine only | `OracleVerdict` | **No model output can become a verdict** |
| `ReducerCapability` | Failure reducer only | `ReductionStep` | Reduction history cannot be fabricated |
| `FixtureWriteCapability` | Fixture store writer only | `FixtureVersion` | Fixtures are immutable to everyone else |
| `InterventionCapability` | Phase-3 runner only | `ControlledInterventionResult` | Screening data cannot masquerade as an intervention |

`ASSUMPTION` (owner: architect; invalidation: a reflection-based bypass is demonstrated): in
Python, construction tokens are enforceable in practice but not against a determined bypass. This
is a **structural** control, stronger than lint, weaker than a memory-safe capability system. It
is described as exactly that and nothing more. Static lint remains as a **secondary** check
(`THREAT_MODEL` SC-list).

### G.1 Zones

```
ZONE U — UNTRUSTED     model output; tool response bodies; retrieved content
ZONE S — SUT           agent under test, its planner, its model config (A FACTOR)
ZONE C — CONTROLLED    execution adapter, synthetic environment, fault injector, fixture store
ZONE X — EXPERIMENT    planner, runner, oracles, reducer, analyzer, compiler
ZONE G — GROUND TRUTH  environment effect log. Readable by ZONE X only. NEVER by ZONE S.
```

`ZONE G` unreachable from `ZONE S` is the property that makes every oracle deterministic and
makes LLM judges unnecessary (`EVALUATION_STRATEGY` §P.1). It is enforced by separate database
credentials and verified by an agent-credential permission test, not by convention.

---

## H. Canonical event and experiment model

### H.1 Hierarchy

```
Experiment  →  Trial  →  Execution  →  Step  →  Attempt
                                              (one dispatch of one Action)
```

An **Action** is the logical unit carrying one adjudicated state. Multiple `Attempt`s, one
`Action`, one state. A retry is a new `Attempt`; a re-plan is a new `Step`.

### H.2 Ordering

`seq` (monotonic per execution, assigned by the single-writer adapter) and
`causal_parent_event_id` are **authoritative**. `t_emit` is **advisory** and may not be used for
cross-process precedence. `t_recv` is for ingest SLO and retention only.

`ASSUMPTION` (DA-06): single-writer per execution makes `seq` trivially correct. Multi-writer
requires a Lamport clock, designed before scaling.

### H.3 Effect identity — pivot correction 2

Three **separate** fields. Conflating any two is a correctness bug.

| Field | Derivation | Stable across |
|---|---|---|
| `effect_id` | `HMAC(deployment_secret, execution_id ‖ logical_step_id ‖ semantic_key(args))` | retries, serialization changes, schema changes |
| `request_digest` | `sha256(canonical_wire_bytes)` | nothing — changes on any byte change |
| `schema_version` | declared by the tool contract | tool contract revisions |

**Rules (enforced at the adapter, not by review):**
1. A retry of the same logical effect **must** present the same `effect_id`.
2. `semantic_key` changed but the planner did not declare a new logical step ⇒ **fail closed**
   (`ABORTED`, reason `EFFECT_IDENTITY_VIOLATION`). Never a silent new key.
3. `request_digest` changed with `effect_id` unchanged ⇒ serialization or schema drift. Recorded
   as an observation; does **not** change dedup identity.
4. A tool that cannot supply `semantic_key` has `CP1 = UNSTABLE`, which forces
   `CP2 = NON_IDEMPOTENT_WRITE` treatment regardless of what it declares.

### H.4 Event envelope (deltas from 0.1.0 marked)

```
  event_id, execution_id, trial_id [ADDED], experiment_id [ADDED]
  seq, causal_parent_event_id, parent_event_id, trace_id
  t_emit (advisory), t_recv, monotonic_ns
  event_type, envelope_schema_version
  agent_version [ADDED], environment_version [ADDED], oracle_version [ADDED]
  actor_identity, agent_id
  factor_assignment_digest [ADDED]          # which experiment cell this trial is
  seed [ADDED], fixture_digest [ADDED]
  tool_name, tool_schema_version, resource_id      # resource_id: ATTRIBUTE ONLY, never a metric label
  action_id, effect_id [RENAMED from idempotency_key], request_digest [ADDED], attempt_no
  action_state, outcome_basis                      # TRANSPORT | RECONCILE | COMPENSATION | DECLARED | NONE
  capability_profile_digest [ADDED]                # CP1..CP8 in force for this call
  status, latency_ms, retry_count
  input_cid, output_cid, input_digest, output_digest
  policy_decision, policy_version
  sensitivity_class, redaction_applied, provenance_ref
```

Payload separation is retained from 0.1.0: digests in the envelope, payloads in a
sensitivity-classed content-addressed store. The tested property stands — **delete the CAS and
the system still functions**, with payload-dependent components degrading to `INDETERMINATE`.

### H.5 Experiment entities

Exactly as the pivot spec defines them.

```
Experiment   agent_version, environment_version, factors[], constraints[], seed,
             repetitions, budget, oracle_ids[], expected_invariants[],
             hypothesis_family [ADDED — fixed before the run, for FDR]

Factor       id, dimension, levels[], kind = FAULT|CONFIG|CONTEXT|TOOL|POLICY|MODEL

Trial        experiment_id, factor_assignment, seed, initial_state_digest,
             outcome, invariant_results[], trace_digest

Failure      failure_id, invariant_id, fingerprint, factor_assignment,
             minimized_assignment, minimized_trace, evidence_refs[],
             reduction_steps[], regression_artifact_id,
             minimality_confidence [ADDED], fingerprint_stability [ADDED]
```

`hypothesis_family` is added because FDR control is defeated by post-hoc family redefinition
(`INTERACTION_MODEL` C5.3). Putting it in the immutable manifest is the only way the control
holds.

---

## I. Action state machine

States are those of `KNOWLEDGE_GRAPH.execution_state_machine`. Terminology corrected: the
outcome vocabulary is `APPLIED` / `NOT_APPLIED` / `INDETERMINATE`, matching CP6.

```
  PLANNED ──deny/budget/schema/identity-violation──▶ ABORTED (terminal, reason enum)
     │ authorize (deterministic: policy, budget, schema, effect_id)
     ▼
  AUTHORIZED ──dispatch──▶ DISPATCHED
                               │
        ┌──────────────────────┼──────────────────────┐
     2xx│                4xx/5xx│               timeout│/reset
        ▼                      ▼                      ▼
  ACKNOWLEDGED          VERIFIED_FAILURE         UNKNOWN_OUTCOME
  (tool CLAIMS success;  (only when the          (INDETERMINATE.
   outcome_basis=         authoritative source    MAY BE TERMINAL.)
   TRANSPORT. NOT proof)  confirms NOT_APPLIED)        │
        │                                              │ retry legality = §J lookup
        │ reconcile (CP3/CP5)                          │
        ▼                                     ┌────────┴────────┐
  VERIFIED_SUCCESS                       RETRY_SAFE         NEVER_RETRY
  (outcome_basis=RECONCILE)              → DISPATCHED       → ESCALATE
                                                              (stays INDETERMINATE)
  VERIFIED_FAILURE ──compensate (depth 1)──▶ COMPENSATED
```

### I.1 The reconciliation trap — retained and generalised

`FACT`: a reconciliation read against a non-authoritative source (`CP3 = REPLICA`) under
`CP4 = EVENTUAL` can report `NOT_APPLIED` for an effect that was applied. Concluding
`VERIFIED_FAILURE` and retrying produces the duplicate the mechanism exists to prevent.

| CP3 | CP4 | Reconcile returns "absent" ⇒ |
|---|---|---|
| `AUTHORITATIVE` | `STRONG` | `NOT_APPLIED` |
| `AUTHORITATIVE` | `BOUNDED`, elapsed ≥ bound | `NOT_APPLIED` |
| `AUTHORITATIVE` | `BOUNDED`, elapsed < bound | `INDETERMINATE` — re-probe after bound |
| `AUTHORITATIVE` | `EVENTUAL` | **`INDETERMINATE` — never `NOT_APPLIED` on absence alone** |
| `REPLICA` | any | **`INDETERMINATE`** |
| `NONE` | any | **`INDETERMINATE`** |
| any | `UNDECLARED` | **`INDETERMINATE`** (the default) |

`UNDECLARED` is the default so that the safe configuration is the lazy one.

**In the pivoted design this table is not just a safety mechanism — it is a factor.** F3
(`tool_consistency`) and the CP3 level are varied deliberately, and INV-1 (no duplicate side
effect) is the oracle that catches the engine getting it wrong. The mechanism is under test by
the experiment engine rather than assumed correct.

---

## J. Retry legality by semantic effect class — pivot correction 6

HTTP verb and `read_only` are **removed** as criteria. The lookup is over
`(CP2 effect class, CP1 identity stability, CP3 reconciliation source)`.

| CP2 effect class | CP1 | CP3 | Verdict |
|---|---|---|---|
| `PURE_READ` | any | any | `RETRY_SAFE` |
| `IDEMPOTENT_READ` | any | any | `RETRY_SAFE` |
| `IDEMPOTENT_WRITE` | `STABLE` / `DERIVABLE` | any | `RETRY_SAFE` (same `effect_id`) |
| `IDEMPOTENT_WRITE` | `UNSTABLE` | any | **`RECONCILE_FIRST`** — idempotency is meaningless without a stable identity |
| `NON_IDEMPOTENT_WRITE` | any | `AUTHORITATIVE` | `RECONCILE_THEN_DECIDE` (per §I.1) |
| `NON_IDEMPOTENT_WRITE` | any | `REPLICA` / `NONE` | **`NEVER_RETRY` → `INDETERMINATE`** |
| `UNKNOWN` | any | any | **`NEVER_RETRY` → `INDETERMINATE`** |

The `UNKNOWN` row exists because pivot correction 7 requires it: for a system whose adapter does
not supply the capability, the honest output is `INDETERMINATE` / `UNVERIFIABLE`.

Each CP property is verified by a contract test. **A declared property that fails its contract
test fails the build.** A tool cannot lie about itself in a way that survives CI — and when we
*want* it to lie (to test what breaks), that is a declared factor level, not a defect.

---

## L. Evidence model — pivot correction 3

`FACT` (logical): a deterministic parse of an untrusted response is a deterministic derivation
**about what the source reported**, not a fact about the world. The 0.1.0 label lattice equated
derivation method with epistemic truth. Corrected to **five orthogonal fields plus a claim
scope**.

```
Evidence
  claim_type         OBSERVED_EVENT | DETERMINISTIC_DERIVATION | STATISTICAL_ASSOCIATION
                     | CONTROLLED_INTERVENTION_RESULT | HYPOTHESIS | INDETERMINATE
  claim_scope        WHAT_THE_SOURCE_REPORTED | ENVIRONMENT_STATE | EXPERIMENT_RESULT
                     | PRODUCTION_BEHAVIOR        ← NEVER ASSERTABLE BY THIS SYSTEM
  observation_source LEDGER | ENVIRONMENT_GROUND_TRUTH | TOOL_RESPONSE | MODEL_OUTPUT
                     | HUMAN | EXTERNAL_SYSTEM
  provenance         source_refs[] + trust_class ∈ {TRUSTED_INTERNAL, UNTRUSTED_EXTERNAL}
  assurance_level    ENVIRONMENT_VERIFIED | ADAPTER_VERIFIED | SELF_REPORTED | UNVERIFIED
  derivation_method  DIRECT_OBSERVATION | DETERMINISTIC_RULE | STATISTICAL_TEST
                     | CONTROLLED_EXPERIMENT | REDUCTION | LLM_SYNTHESIS
  producer_id, producer_version, params, source_refs[]   # all REQUIRED
```

### L.1 The rules that actually bind

1. **`claim_scope = PRODUCTION_BEHAVIOR` is unconstructible.** No capability produces it. This is
   pivot correction 7 made structural.
2. **A `TOOL_RESPONSE` source with `SELF_REPORTED` assurance can only carry
   `claim_scope = WHAT_THE_SOURCE_REPORTED`**, regardless of how deterministic the parse was.
   Deterministic derivation over an untrusted payload is not a world fact.
3. **`claim_scope = ENVIRONMENT_STATE` requires `observation_source = ENVIRONMENT_GROUND_TRUTH`
   and `assurance_level = ENVIRONMENT_VERIFIED`.** This is why the ground-truth effect log
   exists and why `ZONE S` must not reach it.
4. **`CONTROLLED_INTERVENTION_RESULT` requires `derivation_method = CONTROLLED_EXPERIMENT`, a
   Phase-3 design, `n`, an interval, and `divergence_rate`.** Screening data cannot be relabelled
   as an intervention — the `InterventionCapability` is held only by the Phase-3 runner.
5. **`derivation_method = LLM_SYNTHESIS` permits only `claim_type ∈ {HYPOTHESIS,
   INDETERMINATE}`.** The narrator holds no capability, so it cannot construct anything else.
6. **Absence is not evidence.** There is no "nothing found" record. A claim with no support
   renders `INDETERMINATE` plus a coverage manifest naming what ran, at which version, over which
   range — which distinguishes *we looked and found nothing* from *we did not look*.

### L.2 Causal vocabulary — pivot correction 4

| Permitted | Type | Requires |
|---|---|---|
| "X preceded Y in n of m trials" | `STATISTICAL_ASSOCIATION` | n, m |
| "Version change V coincided with a rate shift" | `STATISTICAL_ASSOCIATION` | **observational.** Not a natural experiment. |
| "Toggling factor F, same fixture and seed, changed the violation rate from a/n to b/n, Δ [CI], divergence rate v" | `CONTROLLED_INTERVENTION_RESULT` | Phase-3 design, all four numbers |

**Banned in all output:** *root cause*, *caused by*, *due to*, *because of*, *natural
experiment*, *randomized*. Removed from the vocabulary because assignment in a covering array is
**systematic**, not random; where we do randomise, we say so and show the randomisation.

---

## N. Replay modes — pivot correction 9

| | `EXACT_REPLAY` | `CONTROLLED_REPLAY` | `LIVE_EXECUTION` |
|---|---|---|---|
| Definition | Recorded inputs and outputs replayed | Frozen deterministic fixtures and substrate | Fresh external calls |
| External effects | **None.** Egress guard; violation fails the run. | **None.** Same guard. | Real |
| Model | Served from ledger | Seeded deterministic stub, or ledger | Live, stochastic |
| Factors variable | No | **Yes — this is the point** | Yes, confounded |
| Deterministic | Yes, for the analysis path | Yes, given seed + fixture digest | **No. Not replay.** |
| Supports | "Our analysis produces X over this trace" | "Under this fixture and seed, this assignment yields k/n" | "On this date, n runs gave this distribution" |
| Never called | — | — | **"replay"** |

### N.1 Trajectory divergence

`CONTROLLED_REPLAY` serves recorded responses. A changed factor changes a decision. Subsequent
recorded responses are then **off-policy** — recorded against a different request in a different
state. Continuing produces a plausible trajectory that never occurred, with **no error signal**.

- Compare `request_digest` at every replay decision point.
- Match ⇒ serve; mismatch ⇒ **diverged**.
- Seeded stub available ⇒ continue with `post_divergence_fidelity = STUB` and
  `divergence_from_seq = s`.
- Recorded responses only ⇒ **halt at `s`**, report `TRUNCATED_AT_DIVERGENCE`. Do not fabricate.
- `divergence_rate` is a **required field** on every `CONTROLLED_INTERVENTION_RESULT`.
- Divergent trials are **excluded from paired analyses and counted** — a broken pair is not a
  pair (`INTERACTION_MODEL` C5.2).
- `divergence_rate > 0.5` ⇒ the result reports `INDETERMINATE`, not a finding.

**Any trajectory divergence invalidates downstream counterfactual interpretation unless
explicitly handled by one of the two branches above.** That sentence is the contract.

---

## Build gates — six, each with an exit test

`DESIGN_DECISION`: eight gates became six. Gate 0.5 is this review. Gates 1–6 follow.

---

### GATE 1 — Environment, Ledger, Adapter

- **Objective.** A trial can be executed, observed, and **byte-reproduced**.
- **Scope.** Canonical event ledger (append-only, `seq`-ordered); versioned fixture store with
  snapshot/restore **by digest**; execution adapter holding the sole `DispatchCapability`;
  five synthetic services with **declarable CP1–CP8 profiles**; ground-truth effect log
  unreachable from `ZONE S`; deterministic stub model adapter.
- **Invariants.** INV-9 (determinism). INV-8 (no `INDETERMINATE` promotion). Ledger append-only.
- **Exit tests.** 1000 trials run twice ⇒ identical `trace_digest` for every pair. Fixture
  restore ⇒ digest equals the declared baseline. Agent credential **denied** on the ground-truth
  log. Narrator module, given a bundle and no capability, fails to construct an `ObservedEvent`.
  Full suite passes with zero LLM availability.
- **Why not premature.** Everything downstream — reduction, fingerprinting, regression artifacts
  — is invalid without byte-reproducibility. Snapshot-restore-by-digest cannot be retrofitted;
  adding it later is a rewrite of every component that touched state.
- **Deferred.** Oracles beyond INV-8/9. Any planner. Any reduction. All of Neo4j, ClickHouse,
  Qdrant, Temporal, OPA-as-process, MCP, frontend.

---

### GATE 2 — Oracles, Fault Injector, Runner

- **Objective.** A single trial under a named factor assignment yields a versioned oracle verdict.
- **Scope.** Oracle engine holding the sole `OracleCapability`; INV-1..INV-9; fault injector
  (seeded, per-trial schedulable); single-trial runner; effect identity (§H.3); retry legality
  table (§J); reconciliation matrix (§I.1).
- **Invariants.** All of INV-1..INV-9 evaluable. Every oracle returns `INDETERMINATE` rather than
  defaulting to `HOLDS` when it cannot evaluate.
- **Exit tests.** Oracle meta-suite: known-pass, known-fail, **adversarial near-miss**,
  degenerate input. An oracle that crashes or defaults to `HOLDS` on degenerate input **fails
  the build**. Effect-identity contract tests: serialization change ⇒ same `effect_id`; semantic
  change without a declared new step ⇒ `ABORTED / EFFECT_IDENTITY_VIOLATION`. Replica-lag fault
  ⇒ `INDETERMINATE`, never `NOT_APPLIED`. Server-side request count equals adapter `attempt_no`.
- **Why not premature.** A planner without trustworthy oracles generates volume, not signal. An
  oracle that silently never fires is indistinguishable from a reliable system — this gate exists
  to make that impossible before any large run.
- **Deferred.** Covering arrays. Reduction. Interaction analysis. `OQ-05` (concurrency scope)
  must be resolved *within* this gate before F10 is implemented.

---

### GATE 3 — Experiment Planner and Coverage

- **Objective.** A bounded, constraint-aware plan over the factor space, with honest coverage
  accounting.
- **Scope.** Factor model (C1); constraint-aware covering-array generation at t = 2;
  state-machine sequence generation; metamorphic relations MR-1..MR-6; budget-aware scheduler;
  coverage report (C9) including the **unexplored** space.
- **Invariants.** Coverage report states `infeasible_fraction` and `constraints_applied`.
  Screening and adaptive trials reported **separately**. Every reported number carries n.
- **Exit tests.** Generated array verifiably covers every feasible pair (checked by an independent
  verifier, not the generator). **S2:** the engine finds ≥ 1 two-factor interaction failure whose
  single-factor arms both pass at n ≥ 50. Seeded-bug detection: bugs authored by someone who did
  not write the factor model are detected at a measured rate (`OQ-08`).
- **Why not premature.** The cost arithmetic (`INTERACTION_MODEL` C2.1) shows screening is minutes
  and reduction is hours. Planning before reduction means failures arrive faster than they can be
  processed — which is the correct order, because the reducer's budget must be sized from real
  failure volume.
- **Deferred.** t = 3 (until a 3-way failure is found that pairwise missed). Adaptive search
  beyond Hamming-distance expansion. All learned search.

---

### GATE 4 — Failure Reduction and Fingerprinting

- **Objective.** A failure reduces to a stable, minimal-with-confidence reproducer.
- **Scope.** ddmin with `r_confirm` repetition confirmation; **fingerprint-aware acceptance**;
  non-monotonicity detection (`REDUCTION_UNSTABLE`); hierarchical reduction order; fingerprint
  composition and clustering with `fingerprint_stability`; per-failure reduction budget.
- **Invariants.** A reduction is accepted only if the failure retains the **same fingerprint**.
  `minimality_confidence` is reported; "1-minimal" is never claimed. Budget exhaustion yields a
  **partially reduced** artifact labelled as such.
- **Exit tests.** Seeded **non-monotone** fixture ⇒ reducer emits `REDUCTION_UNSTABLE`, not a
  confident minimum (**S3**). Seeded fingerprint-collision fixture ⇒ cluster flagged `UNSTABLE`
  via within-bucket variance. Reduction of a known 3-factor interaction recovers exactly those
  three factors in ≥ x% of independent runs (x measured, not asserted).
- **Why not premature.** Reduction is the cost centre (13–27 h for 50 failures). Building it
  before the planner produces real failure volume would size its budget from imagination.
- **Deferred.** Automated root-cause attribution (does not exist and will not be built). Cross-
  failure clustering beyond fingerprint identity.

---

### GATE 5 — Interaction Estimation and Regression Compilation

- **Objective.** A minimized failure becomes (a) an interaction estimate with an interval and
  (b) a re-executable regression artifact.
- **Scope.** Phase-3 designed factorials over surviving factors; contingency + logistic
  interaction terms; BH-FDR over the **pre-declared** `hypothesis_family`; McNemar only where
  pairing genuinely holds; regression compiler per `REGRESSION_ARTIFACT_SPEC.md`.
- **Invariants.** Interaction estimates derive **only** from Phase-3 data — the
  `InterventionCapability` makes covering-array data structurally ineligible. Every estimate
  carries raw p, adjusted p, q, family size and family identity. Every regression artifact
  carries a measured `reproduction_rate` with an interval.
- **Exit tests.** **S4:** an artifact re-executes from the artifact alone on a clean checkout and
  reproduces at its recorded rate within its recorded interval. Null-factor control: a factor
  known to have no effect must **not** survive FDR across ≥ 20 independent screens. A
  0.4-reproduction-rate artifact is executed r times against a threshold, never once.
- **Why not premature.** Estimation requires minimized factor sets (Gate 4) — running factorials
  over 14 factors is 7 million cells. Reduction is what makes estimation affordable.
- **Deferred.** Bayesian optimization, learned surrogates, causal-graph inference. All require a
  measured trigger showing the transparent method insufficient.

---

### GATE 6 — Model Canary Boundary and Reporting

- **Objective.** The model becomes a *system under test* on a fixed, budgeted canary boundary,
  and the whole run is reportable with its limits.
- **Scope.** L-LOC and L-REM canary suites (fixed, declared, separately reported); provider
  adapter with token-metered local budget and header reconciliation; optional narrator (pure
  function, no capabilities); MCP adapter behind the internal tool protocol; export of coverage +
  limitations + artifacts.
- **Invariants.** L-DET / L-LOC / L-REM are **never** pooled into one statistic. No remote call
  is required by any deterministic component. `provider_switch` is always recorded, never silent.
- **Exit tests.** **S5:** full pipeline with provider egress blocked produces byte-identical
  structured output. Canary budget enforced by a counter; exceeding it aborts with a partial
  report rather than making an extra call. Report contains the unexplored space and the
  external-validity limitation verbatim.
- **Why not premature.** The model is the most expensive and least reproducible factor. Bringing
  it in last means every other component has been validated deterministically first, so a canary
  discrepancy localises to the model rather than to the harness.
- **Deferred.** A2A, multi-agent swarms, any frontend beyond a static export, real-credential
  integration.

---

## T. What breaks at enterprise scale

| # | Limit | Why | Earliest signal | What would change |
|---|---|---|---|---|
| T-1 | **Reduction cost dominates** | O(n²)·r per failure; 50 failures ≈ 13–27 h | Reduction hours / screening hours | Parallel reduction across failures; per-failure budget caps producing partial artifacts |
| T-2 | **Factor space grows with v²** | Pairwise N scales with the two largest level counts; real tool surfaces mean large v | Generated N vs. budget | Hierarchical factor models; per-adapter sub-experiments |
| T-3 | **Fixture storage** | One snapshot per trial × 10⁵ trials | Fixture store bytes/day | Content-addressed dedup; states are similar, so this may work — measure before assuming |
| T-4 | **Determinism rots silently** | One unseeded RNG in a dependency, one dict-ordering change, one wall-clock read | INV-9 violation rate | Nothing else — this is why INV-9 is an oracle and not an assumption |
| T-5 | **Regression suite invalidation** | Artifacts pin `environment_version`; a bump invalidates a large fraction at once | Invalid-artifact fraction per version bump | Explicit invalidation conditions per artifact; scheduled re-validation |
| T-6 | **Trial wall-clock** | 2 s/trial × 200k trials ≈ 4.6 days serial | Trials/hour | Parallelism — which reintroduces shared-state risk (`EM-12`) and must be isolated per worker |
| T-7 | **Concurrency factor needs a scheduler** | Deterministic testing of F10 requires controlled interleaving; arbitrary thread scheduling requires a simulator | `OQ-05` | Scope F10 to adapter-controlled interleaving points, or accept a much larger build |
| T-8 | **Ledger volume** | 200+ events per execution × 10⁵ trials | Ingest lag p99 | Partitioning, then the ClickHouse trigger |
| T-9 | **Capability-profile maintenance** | Declaring CP1–CP8 for hundreds of real adapters is work nobody will fund | Adapter count vs. profiled count | `UNKNOWN` defaults that route to `INDETERMINATE` — correct, and a smaller product |
| T-10 | **Single-writer `seq`** | Parallel runners break the ordering guarantee | Concurrent-writer errors | Per-trial writer isolation (natural here), or Lamport clocks |
| T-11 | **Fault injection must never reach production** | The engine's purpose is breaking things | — (design-time) | Hard environment separation; egress guard as a test, not a convention |
| T-12 | **The factor model is our own hypothesis** | Coverage is measured against a space we authored | — (structural) | Nothing, within this project. `OQ-08` attacks it; the limitation stands in every report. |
