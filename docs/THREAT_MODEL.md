# THREAT MODEL

Status: **GATE 0.5.** Document version 0.2.0 (full rewrite; supersedes 0.1.0).
Scope: the experiment engine, the execution adapter, the synthetic environment, and the agent
under test. Not a threat model for a production deployment.

---

## 0. Framing

> **The model is the system under test. The adapter is the controlled boundary. The experiment
> plane is the only thing that produces verdicts.**

In the pivoted design, security has two faces and both matter:

1. **Security as a factor.** Permission narrowing, allowlist changes and authorization boundaries
   are dimensions in the experiment space. Breaking them is the *point* — INV-2 is an oracle.
2. **Security of the engine.** A system whose purpose is injecting faults and replaying
   side-effecting traces is itself dangerous, and that danger is architectural.

---

## 1. Capability isolation — the primary control

`DESIGN_DECISION` (pivot correction 8): **illegal evidence production is made impossible by
construction, not forbidden by a linter.** Version 0.1.0 made source lint the architecture. That
was wrong: a lint is a review aid, and a review aid is not a boundary.

A capability is an unforgeable construction token held by exactly one component. A type that
requires a capability in its constructor cannot be built by a component that was never handed one.

| Capability | Sole holder | Gates construction of | If you lack it |
|---|---|---|---|
| `DispatchCapability` | Execution adapter | `DispatchHandle` — the only path to a side effect | You cannot reach the environment |
| `LedgerReadCapability` | Ledger reader | `ObservedEvent` | **You cannot manufacture an observation** |
| `OracleCapability` | Oracle engine | `OracleVerdict` | **Model output cannot become a verdict** |
| `ReducerCapability` | Failure reducer | `ReductionStep` | Reduction history cannot be fabricated |
| `InterventionCapability` | Phase-3 runner | `ControlledInterventionResult` | **Screening data cannot masquerade as an intervention** |
| `FixtureWriteCapability` | Fixture store writer | `FixtureVersion` | Fixtures are immutable to you |
| `GroundTruthReadCapability` | Oracle engine | Access to the effect log | The agent under test cannot see how it is graded |

**The narrator is the worked example.** It is a pure function `(EvidenceBundle) → str`. It is
never passed any capability. It therefore cannot construct an `ObservedEvent`, an `OracleVerdict`,
or a `ControlledInterventionResult` — not as a matter of policy, as a matter of type.

`ASSUMPTION` (owner: architect; invalidation: a reflection-based bypass is demonstrated in our own
codebase): in Python, construction tokens are enforceable in practice but not against a determined
bypass. This is a **structural** control — stronger than lint, weaker than a memory-safe
capability system. It is described as exactly that, and static lint is retained as an explicitly
**secondary** check.

---

## 2. Zones and boundaries

```
ZONE U — UNTRUSTED     model output; tool response bodies; retrieved content
ZONE S — SUT           agent under test, its planner, its model config (A FACTOR, not a fixture)
ZONE C — CONTROLLED    execution adapter, synthetic environment, fault injector, fixture store
ZONE X — EXPERIMENT    planner, runner, oracles, reducer, analyzer, compiler
ZONE G — GROUND TRUTH  environment effect log — ZONE X only, NEVER ZONE S
```

| Boundary | Threat | Enforcement | Unenforced residue |
|---|---|---|---|
| U → C | Model proposes an unauthorized, malformed or budget-exceeding action | Closed-set tool resolution; schema validation against `tool_schema_version`; policy decision; budget check; `effect_id` derivation. All deterministic, all before dispatch. | A semantically wrong but policy-valid action. Policy cannot express "wrong". INV-7 catches it after the fact. |
| U → agent context | Indirect prompt injection | Structural delimiting; `provenance = UNTRUSTED_EXTERNAL`; any extracted instruction re-traverses U → C | **Contained, not prevented.** See §4. |
| C → environment | Wrong identity, privilege escalation | `actor_identity` bound at the adapter from the trial record, never from model output. Credentials never in model context. | A correctly-identified action that should not have been taken. |
| **S → G** | **Leakage: the agent reads its own grading** | Separate DB credentials; `GroundTruthReadCapability` held only by the oracle engine | **Verified by a permission test, not assumed.** A pass on a negative control is the independent backstop. |
| Ledger write | Forged or mutated evidence | Append-only by grant: INSERT only, no UPDATE/DELETE | A compromised writer can still append. Detectable only by invariant scan. |
| Replay → world | A replay causes a real side effect | **Egress guard: `EXACT_REPLAY` and `CONTROLLED_REPLAY` must make zero outbound connections; a violation fails the run** | `LIVE_EXECUTION` is unguarded by definition and must never target a shared environment. |
| X → S | Oracle internals reachable by the agent | Separate process, separate import path, oracles not importable from agent code | Any shared resource nobody thought of. Negative controls are the backstop. |

---

## 3. Adversaries

| Adversary | Capability | In scope? |
|---|---|---|
| **Malicious content in tool output** | Embed instructions; trigger out-of-policy actions | **Yes — primary.** And it is also a **factor level** (F7 `context_defect`), so it is tested rather than merely defended against. |
| **A lying or malicious tool** | Return arbitrary data, misdeclare its capability profile, fabricate reconciliation responses | **Yes.** And CP misdeclaration is a *factor level* — we deliberately run with tools that lie about CP1–CP8 to see what INV-1 catches. |
| **The model, adversarial or merely confused** | Propose anything in its tool surface | **Yes.** The distinction is irrelevant to the control: all model output is untrusted regardless of intent. |
| **An insider with experiment-plane access** | Modify oracles, add constraints to suppress failures, edit recorded reproduction rates | **Partially.** Append-only ledger, immutable artifacts, mandatory constraint rationale make it detectable. Nothing prevents it. |
| **External network attacker** | Standard | **Out of scope.** Handled by the deployment environment. Stating that is part of the model. |
| **Us, optimising against our own oracles** | Tune the factor model or the constraints until runs look clean | **Yes, and most likely.** Mandatory constraint rationale, held-out regions, negative controls, `OQ-08`. |

---

## 4. Indirect prompt injection — stated honestly

`FACT`: content reaching a model context can carry instructions, and the model cannot reliably
separate data from instruction. This is a property of current model architecture, not a patchable
bug.

**The posture is containment, not prevention.** And in the pivoted design, containment is
*measured*: injection is a factor level, INV-2 and INV-5 are the oracles, and the output is a
measured violation rate under named conditions rather than a claim of safety.

| Layer | Control | What it achieves |
|---|---|---|
| Ingest | `provenance = UNTRUSTED_EXTERNAL`, structural delimiting, sensitivity class | Provenance is queryable afterwards. Does not stop the model reading it. |
| Proposal | Any action after untrusted content re-traverses the same policy gate | Blast radius = the identity's authorized capability set, no more |
| Policy | Least privilege per `actor_identity`; destructive actions require an approval obligation | **This is the real control.** |
| Measurement | Injection as a factor; INV-2/INV-5 as oracles; violation rate with an interval | **This is what the project adds.** Not a defence — a measurement of how well the defence holds under factor combinations. |

**Explicitly rejected:** prompt-level defences, model-based injection classifiers as a gate, any
control whose enforcement point is inside the model. They may reduce incidence; none is a
boundary, and treating one as a boundary is how the real control gets skipped.

**The honest claim:** this system does not prevent prompt injection. It **measures** how a given
agent, adapter and policy configuration holds up under injection combined with other factors, and
it produces a reproducible artifact when it does not.

---

## 5. Threats the engine itself introduces

### 5.1 Fault injection reaching a real system
The engine's purpose is breaking things. Egress guard as a **test**; hard environment separation;
`LIVE_EXECUTION` requires explicit per-invocation authorization and may never target a shared
environment. `EM-16`.

### 5.2 Replay re-firing side effects
`LIVE_EXECUTION` re-executes a recorded trajectory against real systems. Anyone who can trigger a
replay of a chosen trace can re-fire its effects. Mitigation: per-invocation authorization,
replay is itself an execution subject to the adapter and to policy, egress guard on the other two
modes.

### 5.3 The adapter aggregates privilege
It holds write credentials for every tool plus the reconciliation read scope. Structurally more
privilege than any agent needs, in one component. Mitigation: per-tool credential scoping;
credentials never loaded into a process that also holds model context. **Residual: structural.**
Centralised enforcement requires centralised privilege.

### 5.4 The CAS is a high-value target created by our own debugging needs
Everything the agent ever saw, indexed and retained, built because reduction and replay need it.
Mitigation: class-based retention; separate grants; the tested **"delete the CAS and the system
still functions"** property; synthetic data only.

### 5.5 `effect_id` as a side channel
`effect_id` is derived from a semantic key. A bare hash would let an outsider who guesses the
arguments confirm whether an action was performed. Mitigation: HMAC with a per-deployment secret.
One line, awkward to change later because the derivation is load-bearing for dedup across restarts.

### 5.6 Constraint suppression as an insider attack
Adding a constraint that excludes a failing combination makes the failure disappear from both the
run *and* the coverage denominator. Mitigation: mandatory `rationale` on every constraint;
`constraints_applied` and `infeasible_fraction` in every coverage report; constraint review is a
gate activity. `EM-08`.

---

## Q. What may safely be stored in telemetry

`DESIGN_DECISION`: **allow-list, never deny-list.** A deny-list enumerates the sensitive things
someone thought of.

### Q.1 Always permitted — `PUBLIC` / `INTERNAL`
Identifiers (`event_id`, `execution_id`, `trial_id`, `experiment_id`, `action_id`, `effect_id`,
`seq`, `attempt_no`, `trace_id`); time (`t_emit`, `t_recv`, `monotonic_ns`, `latency_ms`);
classification (`event_type`, `status`, `action_state`, `outcome_basis`, `abort_reason`,
`policy_decision`); versions (`agent_version`, `environment_version`, `oracle_version`,
`tool_schema_version`, `envelope_schema_version`, `policy_version`, `producer_version`); bounded
names (`tool_name`, `provider`, `model`, `agent_id`); counters and sizes; digests
(`input_digest`, `output_digest`, `request_digest`, `trace_digest`, `fixture_digest`,
`factor_assignment_digest`, `capability_profile_digest`); governance (`sensitivity_class`,
`redaction_applied`, `provenance_ref`); experiment data (`factor_assignment` levels, `seed`,
`invariant_results`).

**Factor assignments and levels are non-sensitive by construction** — they are level identifiers,
never values. This is a deliberate design property: the experiment metadata, which is what almost
all analysis reads, carries no payload.

### Q.2 Permitted with classification — `CONFIDENTIAL`
CAS only, never the envelope, never a metric dimension: tool arguments and responses; retrieved
document identifiers and excerpts; rendered prompts and completions; `resource_id` (**attribute
only**); `actor_identity` where it maps to a person (pseudonymised).

### Q.3 Cardinality rule
Metric dimensions come **only** from this closed set: `tool_name`, `event_type`, `status`,
`action_state`, `outcome_basis`, `policy_decision`, `provider`, `model`, `invariant_id`,
`oracle_id`, `environment_version`.

Everything else is a span or event attribute. Enforced by a CI lint over metric label sets —
cheap at Gate 1, effectively impossible to retrofit.

---

## R. What must be redacted or excluded

### R.1 Never collected
| Excluded | Reason |
|---|---|
| Real enterprise data, real PII, real credentials | Project constraint. Synthetic only. Procedural — and the strongest control in the system, which is worth being uncomfortable about. |
| Secrets in any form, including in error messages | Error messages are the usual leak. |
| **Chain-of-thought as a required field** | Never a telemetry requirement. No oracle, detector or reducer may depend on it. `RESTRICTED` if ever captured. |
| Raw reconciliation credentials | Same rule as tool credentials. |
| Oracle internals, expected answers, ground-truth values in any stream `ZONE S` can reach | Leakage (`EM-07`; `EVALUATION_STRATEGY` P.4). |

### R.2 Redacted on write
Before the CAS write, recorded via `redaction_applied`: pattern-matched secrets; synthetic-PII
patterns from the demonstration corpus; free-text fields on high-sensitivity tools per contract.

### R.3 The three tested properties
Without these, the section above is a policy document:

1. **Redaction coverage test.** Seeded synthetic secrets and PII flow through the full ingest
   path; zero survive to the CAS. CI.
2. **CAS deletion survivability.** Delete the CAS; run the full oracle suite; payload-dependent
   components return `INDETERMINATE`; nothing crashes; nothing silently passes.
3. **Retention enforcement.** Objects past their class retention are gone; referencing ledger rows
   remain valid with a `payload_expired` marker, not a dangling reference.

`OPEN_QUESTION` `OQ-13`: retention per `sensitivity_class` is unset, and it is in direct tension
with reduction and replay, which need payloads. A **policy decision with a named owner**, not an
engineering choice. Blocks Gate 4's reduction scope.

---

## 6. Controls that exist as tests

Anything not on this list is guidance and should be read as weaker.

| # | Test | Guards |
|---|---|---|
| SC-1 | Only the adapter holds `DispatchCapability`; no other module imports the transport client | 5.3 |
| SC-2 | The narrator, given a bundle and no capability, **cannot construct** `ObservedEvent` / `OracleVerdict` / `ControlledInterventionResult` | §1, `EM-02` |
| SC-3 | Metric label sets contain only allow-listed keys | Q.3 |
| SC-4 | `EXACT_REPLAY` / `CONTROLLED_REPLAY` make zero outbound connections | 5.1, 5.2, `EM-16` |
| SC-5 | Seeded secrets and synthetic PII never reach the CAS | R.3.1 |
| SC-6 | Full oracle suite passes with the CAS emptied; payload-dependent oracles return `INDETERMINATE` | R.3.2 |
| SC-7 | **The agent credential is denied on the ground-truth effect log** | S → G leakage |
| SC-8 | Negative-control tasks never pass | Leakage backstop |
| SC-9 | Ledger role has INSERT only | Evidence integrity |
| SC-10 | An action dispatched without a matching authorization record is caught by the invariant scan | INV-2 |
| SC-11 | Adapter `attempt_no` equals the synthetic environment's server-side request count | `EM-11` |
| SC-12 | An oracle crashing or defaulting to `HOLDS` on degenerate input fails the build | `EM-07` |
| SC-13 | Every constraint has a non-empty `rationale`; coverage reports `infeasible_fraction` | 5.6, `EM-08` |
| SC-14 | Regression artifacts are immutable; a re-measurement creates a new version | `REGRESSION_ARTIFACT_SPEC` §5 |
| SC-15 | Banned causal vocabulary absent from all output templates (**secondary** lint) | `ARCHITECTURE_SURFACE` L.2 |

SC-15 is marked secondary deliberately. The primary control against illegitimate causal claims is
that `ControlledInterventionResult` requires a capability only the Phase-3 runner holds; the lint
catches prose, not structure.
