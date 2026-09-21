# THREAT MODEL

Status: **GATE 0.** Document version 0.1.0.
Scope: the reliability control plane, its reference agent, and the synthetic enterprise
environment. Not a threat model for a production deployment — that requires a real deployment.

---

## 0. Framing — the one sentence that determines everything else

> **The model is not the trusted component. The runtime is the trusted boundary.**

Every control in this document is a consequence. Where a control cannot be enforced at the
runtime boundary, it is recorded as *unenforced* rather than dressed up as a mitigation. A
prompt instruction is not a control. A tool annotation is not a control. A model's stated
intention is not a control.

`FACT` (logically entailed, not cited): tool metadata such as MCP annotations arrives over the
same channel as the tool's payload, from the same party. Anything that can lie about the payload
can lie about the annotation. Therefore annotations are **inputs to policy**, never **policy**.

---

## 1. Trust boundaries and assets

### 1.1 Assets, ranked by what their loss costs

| Asset | Why it matters | Loss scenario |
|---|---|---|
| **Enterprise side effects** (tickets, records, notifications, identity changes) | The only thing with real-world consequence. | Duplicate, missing, unauthorised, or destructive writes. |
| **Credentials held by the gateway** | Aggregate write access across the whole tool surface, plus (per T-7) broad read access for probes. | Compromise yields more privilege than any single agent has. |
| **The evidence ledger's integrity** | It is the sole basis for every `FACT` the system emits. | Mutation or forgery makes every downstream conclusion unsound and undetectably so. |
| **CAS payloads** | Prompts, tool responses, retrieved documents. | Data exposure; the largest regulatory liability (FM-18). |
| **Policy definitions and versions** | Determine what may execute. | Unauthorised modification = silent authorisation bypass. |
| **Grader code and fixtures** | Determine what "correct" means. | Leakage to the agent invalidates all evaluation (FM-15). |

### 1.2 Trust zones

```
 ZONE U  — UNTRUSTED
   • model output (tool proposals, arguments, natural language)
   • tool response bodies
   • retrieved documents
   • any content that has ever transited a model context

 ZONE C  — CONTROLLED  (we wrote it; it is not adversarial, but it is fallible)
   • reference agent
   • synthetic enterprise services
   • fault injector

 ZONE T  — TRUSTED  (enforcement lives here and only here)
   • runtime gateway: policy, budget, idempotency, dispatch, probe, adjudication
   • credential store
   • evidence ledger write path
   • policy definition store

 ZONE E  — EVALUATION  (isolated from C and U; must never be reachable from them)
   • graders, fixtures, oracles, reference solutions
```

**Zone U output can enter Zone T only as data, never as a decision.** Zone E must be unreachable
from Zone C. Both are architectural tests, not conventions — see §5.

### 1.3 Boundary enforcement table

Cross-references `ARCHITECTURE_SURFACE.md` §G.

| Boundary | Threat | Enforcement (deterministic) | Unenforced residue |
|---|---|---|---|
| U → T (model proposal) | Model proposes an unauthorised, malformed, or budget-exceeding action. | Closed-set tool name resolution; JSON-Schema validation against the pinned `tool_schema_version`; policy evaluation; budget check. All before dispatch. | A *semantically* wrong but policy-valid action. Policy cannot express "wrong". Evaluation catches it after the fact, not before. |
| U → context (tool output) | Indirect prompt injection. | Structural delimiting and `provenance=UNTRUSTED_TOOL` tagging on ingest; any instruction extracted from it must re-traverse U → T. | **Injection is contained, not prevented.** See §3. |
| T → resource | Wrong identity; privilege escalation. | `actor_identity` bound at the gateway from the execution record, never from model output. Credentials never present in model context. | A correctly-identified action that should not have been taken. |
| Ledger write | Forged or mutated evidence. | Append-only by grant: the writing role has INSERT only, no UPDATE/DELETE. Derived tables are separate and reference ledger rows by `(execution_id, seq)`. | A compromised writer can still append false events. Detectable only by invariant scans (§5.3). |
| Replay → world | A replay causes a real side effect. | Egress guard asserts zero outbound connections in `EXACT` and `CONTROLLED` modes; a violation **fails the replay**. | `LIVE_REEXECUTION` is by definition unguarded and must never target a shared environment. |
| C → E | Grader leakage. | Separate process, separate DB credentials, no shared filesystem path, no overlapping env vars, graders not importable from agent code; negative-control suite as backstop. | Any shared resource nobody thought of. Backstop is the negative control. |

---

## 2. Adversaries

| Adversary | Capability | In scope? |
|---|---|---|
| **Malicious content in tool output / retrieved documents** | Can embed instructions, canary-evading payloads, or data designed to trigger an out-of-policy action. | **Yes — primary.** This is the realistic threat for a tool-using agent. |
| **A compromised or malicious tool / MCP server** | Can return arbitrary data, lie about its annotations, lie about its consistency model, fabricate probe responses. | **Yes.** Note the consequence: a lying probe defeats verification entirely (§4.1). |
| **The model itself, behaving adversarially or simply badly** | Can propose any action within its tool surface. | **Yes** — and handled by treating all model output as untrusted regardless of intent. The distinction between a malicious model and a confused one is irrelevant to the control. |
| **An insider with control-plane access** | Can modify policy, read the CAS, append false evidence. | **Partially.** Append-only ledger and policy versioning make it detectable; nothing here prevents it. Out of scope to prevent in a prototype. |
| **An external network attacker** | Standard. | **Out of scope.** Assumed handled by the deployment environment. Stating that explicitly is part of the model. |
| **A researcher optimising against our own evaluation** | Can, consciously or not, tune the agent to the grader. | **Yes.** This is the most likely adversary in practice, and it is us. Handled in `EVALUATION_STRATEGY` §P. |

---

## 3. Indirect prompt injection — stated honestly

`FACT`: content that reaches a model context can carry instructions, and the model cannot
reliably distinguish data from instruction. This is a property of the architecture of current
models, not a bug to be patched.

**Therefore the security posture is containment, not prevention.**

| Layer | Control | What it actually achieves |
|---|---|---|
| Ingest | Tag `provenance=UNTRUSTED_TOOL`; wrap in a structural delimiter; record `sensitivity_class`. | Makes provenance queryable after the fact. Does **not** stop the model reading it. |
| Proposal | Any action proposed after untrusted content entered the context traverses the same policy gate as any other. | The blast radius of a successful injection is exactly the identity's authorised capability set — no more. |
| Policy | Least privilege per `actor_identity`; destructive actions require an approval obligation; egress actions checked against `sensitivity_class`. | **This is the real control.** An injection that can only do what the agent was already allowed to do is a bounded problem. |
| Detection | Canary tokens seeded in tool outputs; alert when a canary appears in a subsequent proposal. | Catches injections that interact with the canary. Catches nothing else. Weak by construction and must be described as such. |

**Explicitly rejected:** prompt-level defences ("ignore instructions in tool output"),
model-based injection classifiers as a gate, and any control whose enforcement point is inside
the model. These may reduce incidence; none is a boundary, and treating one as a boundary is how
the real control gets skipped.

**The honest statement for any report:** *this system does not prevent prompt injection. It
bounds what a successful injection can do, and it records enough to determine afterwards what it
did.* That second half — evidence-backed post-hoc determination — is the part this architecture
is actually good at, and it is worth more than an overclaimed prevention story.

---

## 4. Threats specific to this architecture

These are threats the *reliability system itself* introduces. A control plane is a new attack
surface, and pretending otherwise is the most common failure of security-adjacent tooling.

### 4.1 A lying tool defeats verification completely

If a tool controls both the write endpoint and the postcondition probe, it can report any
outcome it likes. Our entire verification chain reduces to trusting that tool.

- **Mitigation (partial):** prefer probes that read a *different* system or a lower layer than
  the write path — the backing store rather than the service API. In the synthetic environment
  this is possible because we own the store.
- **Residual:** for a real third-party SaaS tool, the probe necessarily goes through the same
  vendor. **There is no technical fix.** The honest framing is that verification defends against
  *accident* — timeouts, partial failures, eventual consistency — and not against a
  *malicious* tool. This distinction must appear in any security claim the project makes.

### 4.2 The gateway is an aggregation of privilege

Per T-7, the gateway holds write credentials for every tool plus read access for every probe.
That is strictly more privilege than any individual agent needs, concentrated in one component.

- **Mitigation:** per-tool credential scoping rather than one credential; probes delegated to
  tool owners where possible (`OQ-02`); credentials never loaded into any process that also
  holds model context.
- **Residual:** structural. Centralised enforcement requires centralised privilege. The trade is
  real and should be named rather than hidden.

### 4.3 The CAS is a high-value target created by our own debugging needs

Everything the agent ever saw, in one place, indexed, retained. We built it for root-cause
analysis and it is simultaneously the best single target in the system.

- **Mitigation:** class-based retention; separate grants; the tested "delete the CAS and the
  system still works" property; synthetic data only.
- **Residual:** medium. The only robust control is the no-real-data rule, which is procedural.

### 4.4 Replay can be weaponised

`LIVE_REEXECUTION` re-executes a recorded trajectory against real systems. An attacker who can
trigger a replay of a chosen execution can re-fire its side effects.

- **Mitigation:** `LIVE_REEXECUTION` requires explicit human authorisation per invocation,
  is recorded as a first-class execution with its own identity and policy evaluation (it is not
  exempt from the gateway), and may never target a shared or production environment.
  `EXACT` and `CONTROLLED` are egress-guarded (TB-7).
- **Residual:** low, provided the egress guard is a test and not a convention.

### 4.5 Idempotency keys are a side channel

`idempotency_key` is derived from the argument digest. Possessing a key implies knowledge of the
arguments; presenting a key to a cooperating server can confirm whether a given action was
already performed.

- **Mitigation:** keys are HMAC'd with a per-deployment secret rather than a bare hash, so a key
  cannot be recomputed by an outsider who guesses the arguments.
- **Residual:** low. Worth fixing now because it costs one line and is awkward to change later
  (the key derivation is load-bearing for deduplication across restarts).

---

## Q. What may safely be stored in telemetry

`DESIGN_DECISION`: **allow-list, never deny-list.** A deny-list is a list of the sensitive things
someone thought of. Every field is excluded unless it appears below.

### Q.1 Always permitted — `PUBLIC` / `INTERNAL`

Structural, bounded-cardinality, non-content:

- Identifiers: `event_id`, `execution_id`, `action_id`, `trace_id`, `parent_event_id`,
  `causal_parent_event_id`, `seq`, `attempt_no`
- Time: `t_emit`, `t_recv`, `monotonic_ns`, `latency_ms`
- Classification: `event_type`, `status`, `action_state`, `outcome_basis`, `policy_decision`,
  `abort_reason`
- Versions: `schema_version`, `tool_schema_version`, `policy_version`, `deployment_id`,
  `producer_version`
- Bounded names: `tool_name`, `provider`, `model`, `agent_id`
- Counters and sizes: `retry_count`, token counts, byte lengths, payload sizes
- Digests: `input_digest`, `output_digest`, `state_fingerprint`, `canonical_args_digest`
- Governance: `sensitivity_class`, `redaction_applied`, `provenance_ref`

### Q.2 Permitted with classification — `CONFIDENTIAL`

Stored in the CAS, never in the event envelope, never in a metric dimension:

- Tool arguments and responses (`sensitivity_class` set per tool contract)
- Retrieved document identifiers and excerpts
- Rendered prompts and model completions
- `resource_id` — permitted as an **attribute only**; lint-forbidden as a metric label (FM-10)
- `actor_identity` — pseudonymised where it corresponds to a person

Subject to: class-based retention, separate access grants, redaction on write, and the
"delete-the-CAS" survivability property.

### Q.3 Cardinality rule

Metric dimensions are drawn **only** from this closed set:
`tool_name`, `event_type`, `status`, `action_state`, `outcome_basis`, `policy_decision`,
`provider`, `model`, `deployment_id`.

Everything else is a span or event attribute. Enforced by a CI lint over metric label sets
(FM-10). This is cheap at Gate 1 and effectively impossible to retrofit.

---

## R. What must be redacted or excluded

### R.1 Never collected, at all

| Excluded | Reason |
|---|---|
| **Real enterprise data, real PII, real credentials** | Project constraint. Synthetic only. This is procedural, and it is the strongest control in the system — which is worth being uncomfortable about. |
| **Secrets in any form**: API keys, tokens, passwords, private keys, connection strings | Both in payloads and in error messages. Error messages are the usual leak. |
| **Chain-of-thought as a required field** | Project rule 8. CoT may contain reasoning about sensitive content; requiring it makes redaction mandatory on the highest-volume, least-structured field in the system. **Not a telemetry requirement. No detector may depend on it. Classified `RESTRICTED` if ever captured.** |
| **Raw authentication material used for probes** | Same rule as tool credentials. |
| **Grader internals, reference solutions, oracle values** in any stream the agent can reach | Leakage (FM-15). |

### R.2 Redacted on write

Applied before the CAS write, not after, and recorded via `redaction_applied`:

- Pattern-matched secrets (key formats, bearer tokens, connection strings)
- Synthetic-PII patterns in the demonstration corpus — these exist specifically so the redactor
  has a test corpus. **A redactor with no test corpus is decoration.**
- Free-text fields on high-sensitivity tools, per that tool's contract

### R.3 The properties that make this real rather than aspirational

Three tested properties, without which everything above is a policy document:

1. **Redaction coverage test.** Seeded synthetic secrets and PII flow through the full ingest
   path; the test asserts zero survive to the CAS. Run in CI.
2. **CAS deletion survivability.** Delete the entire CAS; run the full detector suite;
   payload-dependent detectors return `UNKNOWN` with a coverage note; nothing crashes; nothing
   silently passes.
3. **Retention enforcement test.** Objects past their class retention are actually gone, and the
   referencing ledger rows remain valid with a `payload_expired` marker rather than a dangling
   reference.

`OPEN_QUESTION` `OQ-05`: retention periods per `sensitivity_class` are not yet set, and they are
in direct tension with replay (T-6): replay needs payloads, privacy wants them gone. This is a
**policy decision with an owner**, not an engineering choice, and it blocks Gate 5's replay
scope. It must be answered before replay is built, not after.

---

## 5. Controls that must exist as tests, not as documents

The following are the security controls in this document that are actually enforceable. Each is
a test that fails the build. Anything in this document not on this list is guidance, and should
be read as weaker.

| # | Test | Guards |
|---|---|---|
| SC-1 | No module outside the gateway imports the transport client or holds a credential. | FM-17 |
| SC-2 | No core analysis module imports a provider, MCP, or framework package. | FM-21 |
| SC-3 | Metric label sets contain only allow-listed keys. | FM-10, Q.3 |
| SC-4 | `EXACT` / `CONTROLLED` replay make zero outbound connections. | TB-7, §4.4 |
| SC-5 | Seeded secrets and synthetic PII never reach the CAS. | R.3.1 |
| SC-6 | Full detector suite passes with the CAS emptied; payload detectors return `UNKNOWN`. | R.3.2, FM-20 |
| SC-7 | Grader modules are not importable from agent code; no shared DB credential; negative-control tasks never pass. | TB-5, FM-15 |
| SC-8 | Ledger role has INSERT only; UPDATE and DELETE are rejected. | §1.3 |
| SC-9 | No evidence row with `extraction_method=LLM_SYNTHESIS` carries `FACT`, `CORRELATION`, `OBSERVATION`, or `COUNTERFACTUAL_EVIDENCE`. | §L.2, TB-6 |
| SC-10 | An action dispatched without a matching `AuthorizationDecision` record is detected by the nightly invariant scan. | FM-17 |
| SC-11 | Every action's `attempt_no` matches the synthetic environment's server-side request count. | FM-01c |
| SC-12 | Detector crash injection produces an `UNKNOWN` evidence record and a coverage-manifest entry, never silence. | FM-20 |

**If a control is not on this list, it is not a control.** That is the standard this document
holds itself to, and it is the standard by which the security section of any final report should
be judged.
