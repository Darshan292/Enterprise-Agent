# FAILURE TAXONOMY

Status: **GATE 0.** Document version 0.1.0.

Two distinct things live in this document and they must not be confused:

- **§1 — Domain failure classes.** Failures of *the agent under observation*. These are what the
  system is built to detect. They are the input.
- **§2 — Architectural failure modes.** Failures of *the system we are proposing to build*.
  These are the adversarial review. They are the reason this document is longer than §1.

A system that catalogues §1 and ignores §2 is a system that will fail in exactly the ways it
claims to prevent. §2 is the load-bearing section.

---

## §1 Domain failure classes (what we detect)

Carried from `PROJECT_CONTEXT.core_failure_modes`, organised by the layer that must detect them
and by whether detection can be deterministic.

| Class | Layer | Deterministic detection possible? | Primary signal |
|---|---|---|---|
| `rate_limit` | Provider adapter | **Yes** | HTTP 429 + header reconciliation |
| `retry_storm` | Runtime gateway | **Yes** | attempt count vs. `ExecutionBudget` |
| `tool_timeout_after_dispatch` | Runtime gateway | **Yes** | transport timeout → `UNKNOWN_OUTCOME` |
| `duplicate_side_effect` | Verification | **Yes** | postcondition probe returns cardinality > 1 for one `idempotency_key` |
| `partial_response` | Tool adapter | **Yes** | schema validation failure / truncation marker |
| `schema_drift` | Tool contract | **Yes** | `tool_schema_version` ≠ pinned version, or contract test diff |
| `stale_context` | Context assembly | **Yes, if validity intervals are modelled** | fact's `valid_to` < execution time |
| `conflicting_facts` | Context assembly | **Yes** | two retrieved claims with same key, incompatible values, overlapping validity |
| `retrieval_miss` | Retrieval | **Partially** | oracle-known required document absent from retrieved set (eval only) |
| `context_rot` | Context assembly | **No — measurable only statistically** | degradation of task success vs. context length, held out |
| `wrong_tool` | Evaluation | **Yes, against a reference trajectory** | tool chosen ∉ acceptable set |
| `wrong_arguments` | Evaluation / gateway | **Yes** | schema-valid but semantically wrong vs. oracle |
| `state_drift` | Verification | **Yes** | `state_fingerprint` diverges from expected post-state |
| `indirect_prompt_injection` | Security | **Partially** | canary tokens in tool output followed by an out-of-policy proposal |
| `privilege_abuse` | Policy | **Yes** | action requires a scope the `actor_identity` lacks |
| `unauthorized_data_transfer` | Policy | **Yes** | egress action with `sensitivity_class` above the destination's clearance |
| `destructive_action` | Policy | **Yes** | action declared destructive without an approval obligation satisfied |
| `reward_or_eval_hacking` | Evaluation | **Partially** | grader passes while oracle side-effect check fails |
| `unapproved_inter_agent_communication` | Policy | **Yes** | out-of-scope, no second agent exists yet |
| `grader_bug` | Meta-evaluation | **Yes, against grader fixtures** | grader fails its own known-good/known-bad suite |
| `provider_regression` | Reliability harness | **Statistically only** | pass^k shift across `deployment_id`, with CI |

**Note on the "partially" and "no" rows.** Three of these — `context_rot`, `retrieval_miss`
outside eval, and `indirect_prompt_injection` in the general case — are **not deterministically
detectable**, and no amount of architecture makes them so. `context_rot` is a statistical
property of a distribution, not a property of a run. Injection detection by canary catches only
injections that happen to interact with the canary. The honest system reports these as
statistical or partial signals and never as a verdict on a single execution. Anything that
claims per-run detection of context rot is guessing.

---

## §2 Architectural failure modes — adversarial review of our own design

Each mode is stated as a failure of *this architecture*, not of agents in general.

---

### FM-01 — Duplicate side effects occur despite the gateway

- **Failure.** Two identical side effects reach a backing store even though the runtime gateway
  was in the path and adjudicated the action.
- **Cause.** Three distinct mechanisms, and conflating them is itself a design error:
  (a) the tool declared `accepts_idempotency_key=true` but the server's key retention window is
  shorter than our retry backoff, so the second attempt is treated as new;
  (b) `consistency_model` mis-declared as `STRONG` when the tool is actually eventually
  consistent, producing the §I.2 trap — a false `VERIFIED_FAILURE` followed by a legitimate-looking
  retry;
  (c) a retry issued by a layer *beneath* our gateway (HTTP client, SDK, service mesh) that we
  never observe, so our attempt counter reads 1 while the server saw 3.
- **Impact.** The single failure the project exists to prevent. Occurring even once in the demo
  invalidates the core claim. In a real deployment: duplicate tickets, duplicate notifications,
  duplicate financial effects.
- **Detection.** Postcondition probe returns cardinality > 1 for one `idempotency_key`. In the
  synthetic environment, a ground-truth side-effect counter per trial asserts `count == 1`.
  For (c) specifically: compare gateway `attempt_no` against a server-side request counter the
  synthetic environment exposes — **any divergence is a bug in our dispatch path**, and this
  check must exist from Gate 1 because it is the only way to see through our own abstraction.
- **Mitigation.** Derived idempotency keys (§J.3); `UNDECLARED` consistency default (§I.2);
  **disable all retry behaviour in every HTTP client and SDK beneath the gateway** and assert
  that in a contract test; the `BLIND_WRITE — NEVER RETRY` row of the capability matrix.
- **Residual risk.** **High, and irreducible for real tools.** We cannot see retries inside a
  vendor SDK or a service mesh we do not control. The synthetic environment will show zero
  duplicates because we wrote both ends. That result is weaker evidence than it will feel like.

---

### FM-02 — Retry storm exhausts the execution and amplifies load

- **Failure.** A transient fault produces a cascade of retries that consumes the budget and,
  in a shared environment, degrades the very service that was failing.
- **Cause.** Per-call retry counters that multiply across nesting levels; retries without
  jitter synchronising across concurrent executions; the agent *re-planning* a failed action as
  a "new" action, which resets any per-action counter.
- **Impact.** Quota exhaustion, self-inflicted denial of service, an incident that looks like a
  provider outage but originates with us.
- **Detection.** `D-LOOP-REPEAT`; `ExecutionBudget.max_retries_total` consumption rate;
  ratio of attempts to distinct `action_id`s.
- **Mitigation.** Shared execution-level retry pool, not per-call (§K.1). Full jitter on
  backoff. **Re-planning a semantically identical action reuses the `action_id`** — matched on
  `(tool_name, canonical_args_digest)` — so a re-plan cannot launder a retry into a fresh budget.
- **Residual risk.** Medium. The re-plan detection is digest-based; a model that changes one
  irrelevant argument defeats it. Argument canonicalisation would need semantic normalisation
  per tool, which is real work not yet scoped. Logged as a known gap.

---

### FM-03 — Rate-limit amplification turns a throttle into an outage

- **Failure.** Receiving 429s causes behaviour that increases request rate.
- **Cause.** Retrying the 429 itself; treating 429 as a transient network error in a generic
  retry wrapper; multiple concurrent executions each independently retrying against one shared
  credential bucket; ignoring `Retry-After`.
- **Impact.** Daily quota destroyed in minutes. On a free tier with 1000 RPD, a 20-execution
  storm can consume an entire day (`QUOTA_AND_COST_MODEL` §2).
- **Detection.** 429 count per window; local bucket vs. header-reported remaining divergence;
  time-to-quota-exhaustion projection.
- **Mitigation.** Local token bucket enforced **before** dispatch, shared across all executions
  per `(provider, model, credential)`. 429 is never retried against the same bucket before its
  reset. Circuit breaker per provider. Header reconciliation — never hard-code limits.
- **Residual risk.** Medium-low for our own traffic. **Zero control** over other processes
  sharing the same credential, which on a free tier is a realistic scenario. Mitigation: one
  credential per environment, asserted at startup.

---

### FM-04 — `UNKNOWN_OUTCOME` saturation: correct and useless

- **Failure.** The system becomes correct and useless. Most actions terminate in
  `UNKNOWN_OUTCOME → ESCALATE` because real tools do not expose probes, idempotency keys, or
  compensation.
- **Cause.** The capability matrix (§J.2) is honest. Honesty is expensive when the underlying
  tool surface is poor.
- **Impact.** **Strategic, not technical.** The product reduces to "an expensive way to page a
  human". The synthetic environment hides this completely, because we implemented the tools and
  naturally gave them probes.
- **Detection.** Report the **escalation rate** as a headline metric from Gate 2 onward, not as a
  footnote. If it is above ~20% on a realistic tool mix, the thesis needs rework.
- **Mitigation.** Partial and unsatisfying: probe-by-natural-key for tools without an explicit
  probe (`OQ-03`, with its own false-positive risk); a "shadow ledger" that records intent
  before dispatch so a human can reconcile manually; pushing probe implementation onto tool
  owners as a contract requirement.
- **Residual risk.** **Highest strategic risk in the project.** `OQ-01` exists to measure it
  against three real API surfaces *before* Gate 2. Deferring that measurement is the most likely
  way this project builds the wrong thing well.

---

### FM-05 — The verification probe manufactures the bug it prevents

- **Failure.** A probe against an eventually-consistent store returns "absent" for a write that
  succeeded. The system records `VERIFIED_FAILURE`, retries, and creates a duplicate.
- **Cause.** Treating probe absence as proof of non-occurrence. Structurally identical to
  treating a timeout as proof of non-occurrence — the exact error project rule 7 forbids,
  re-introduced one layer down.
- **Impact.** Worse than having no probe: it converts an honest `UNKNOWN` into a confident wrong
  answer, and the confidence is what authorises the damaging retry.
- **Detection.** Contract test per tool: write, probe immediately, probe after the declared
  bound, assert declared `consistency_model` matches observed. Fault injector includes a
  replica-lag fault specifically to exercise this path.
- **Mitigation.** The §I.2 consistency matrix. `UNDECLARED` default. Absence never yields
  `VERIFIED_FAILURE` under `EVENTUAL`. Probe budget exhaustion yields `ESCALATE`, never a guess.
- **Residual risk.** Medium. Depends on tools declaring their consistency model truthfully. Real
  vendor APIs frequently do not document theirs at all, which is precisely why `UNDECLARED` must
  stay the default and must not be "temporarily" overridden for a demo.

---

### FM-06 — Schema drift silently invalidates keys, detectors and baselines

- **Failure.** A tool's schema changes. Idempotency keys change (by design, §J.3), so
  deduplication silently stops working across the boundary. Detectors keyed on field paths
  silently stop matching. Historical reliability baselines silently become incomparable.
- **Cause.** Schema version is recorded but not *enforced* as a comparability boundary.
- **Impact.** The most insidious mode in the list, because everything continues to run and every
  number continues to look plausible. Nothing alerts.
- **Detection.** Contract tests pinned to `tool_schema_version`, run in CI against the live tool
  contract. A **comparability guard** in the reliability harness: refuse to aggregate across
  differing `tool_schema_version` unless explicitly overridden with a recorded justification.
- **Mitigation.** Schema version in the idempotency key and in `deployment_id`. Detectors
  declare the schema versions they support and return `UNKNOWN` — never a false negative — on
  unsupported versions. A detector that silently returns "no anomaly" on a schema it cannot
  parse is a bug class of its own.
- **Residual risk.** Medium. Detector schema-support declarations will drift from reality unless
  CI enforces them. Requires a test that runs every detector against every recorded schema
  version and asserts a non-crash, non-silent outcome.

---

### FM-07 — Stale context accepted as current

- **Failure.** The agent acts on a superseded policy document or an expired fact and the system
  records the execution as successful.
- **Cause.** Retrieval returns by relevance; relevance is not recency and is not validity. The
  proposed BM25/dense/RRF stack optimises similarity, which is orthogonal to truth-at-time-T.
- **Impact.** Correct-looking execution, wrong outcome. Invisible to outcome adjudication, which
  only answers "did it happen exactly once", never "should it have".
- **Detection.** Only possible if validity intervals are modelled: every context artifact carries
  `valid_from` / `valid_to` / `superseded_by`, and a deterministic check fires when
  `valid_to < execution_time`. **Without that schema, this is undetectable** — which is the
  argument for modelling it, rather than any citation.
- **Mitigation.** Temporal validity as a first-class field on context artifacts; a deterministic
  staleness check **before** the LLM sees the context, not after; supersession edges in the
  knowledge store.
- **Residual risk.** Medium-high. Validity intervals must be populated by whoever owns the source
  data. In a real enterprise most documents have no reliable validity metadata. We will
  demonstrate this against a synthetic corpus where we control the metadata — which proves the
  mechanism and proves nothing about the real case.

---

### FM-08 — Provider or model drift invalidates every baseline

- **Failure.** A provider silently updates a model behind a stable alias. All prior reliability
  measurements become incomparable, and a real regression is indistinguishable from noise.
- **Cause.** Model aliases are not versions. Providers rarely announce weight changes. Free
  tiers rotate the model pool.
- **Impact.** The reliability harness — the thing that produces the project's headline numbers —
  loses its meaning without any visible error.
- **Detection.** Pin and record the fully-qualified model identifier and every advertised
  fingerprint/version field on **every** call, into `deployment_id`. Run a small fixed **canary
  prompt set** on a schedule and alert on distribution shift. A canary is a weak detector, but
  it is the only one available from outside.
- **Mitigation.** `deployment_id` as the comparability boundary for all aggregation, exactly as
  with schema (FM-06). Reliability results are **never** pooled across `deployment_id`. Report
  the identifier alongside every number.
- **Residual risk.** **High and irreducible.** A silent weight change behind an unchanged
  identifier is undetectable except statistically, and the canary set needed to detect small
  shifts is itself an inference cost we largely cannot afford (`QUOTA_AND_COST_MODEL` §4).

---

### FM-09 — Execution graph explosion

- **Failure.** Graph construction becomes the system's bottleneck or exhausts memory.
- **Cause.** `resource_contention` edges are O(n²) in actions touching a shared resource. A long
  execution operating repeatedly on one ticket generates a quadratic edge set. Adding
  cross-execution edges multiplies it.
- **Impact.** Analysis latency, OOM, and — worse — pressure to "solve it" by adding a graph
  database, which relocates the cost without fixing the quadratic definition.
- **Detection.** Edge count and build time per incident, with a hard ceiling that fails loudly.
- **Mitigation.** Per-edge-type budgets. Contention edges materialised only between *adjacent*
  actions on a resource (a chain, O(n)), with transitive reachability computed on demand rather
  than stored. Depth-bounded traversal. Lazy construction — build the graph for one incident,
  not for the corpus.
- **Residual risk.** Low-medium, provided the O(n²) definition is never introduced. The real risk
  is that someone adds it later for a "richer" analysis; the edge-budget ceiling is what catches
  that.

---

### FM-10 — High-cardinality telemetry kills the metrics backend

- **Failure.** Metric series count explodes; queries slow; ingestion is throttled or billed
  catastrophically.
- **Cause.** `resource_id`, `idempotency_key`, `execution_id`, `input_digest` are all unbounded.
  Any of them used as a metric label is fatal. OTel makes adding an attribute to a metric
  trivially easy, which is exactly the problem.
- **Impact.** Observability of the observability system fails — and it fails under load, i.e.
  precisely when it is needed.
- **Detection.** Series-count monitoring with a hard budget. **A CI lint that inspects every
  metric label set against an allow-list of bounded-cardinality keys.**
- **Mitigation.** A strict allow-list of metric dimensions: `tool_name`, `event_type`, `status`,
  `action_state`, `outcome_basis`, `policy_decision`, `provider`, `model`, `deployment_id`.
  Everything else is a span/event attribute only, never a metric dimension. Declared at Gate 1
  and lint-enforced — this is cheap now and near-impossible to retrofit.
- **Residual risk.** Low if lint-enforced from Gate 1. High if deferred, because by then the
  violating code is everywhere.

---

### FM-11 — Silent fallback between providers with different guarantees

- **Failure.** Primary is rate-limited; the system falls back; output characteristics change;
  results are pooled as though they came from one system.
- **Cause.** Fallback implemented as an exception handler rather than as a recorded state
  transition. This is an explicitly-listed anti-pattern in the bootstrap and it is the easiest
  one to reintroduce accidentally, because a silent fallback looks like good engineering.
- **Impact.** Reliability numbers become a blend of two distributions with no way to unmix them.
  A regression in one provider is masked by the other.
- **Detection.** Every fallback emits a `provider_switch` event carrying both identities and the
  reason. Alert on fallback rate. Any analysis window containing a switch is flagged.
- **Mitigation.** Fallback changes `deployment_id`, which is the comparability boundary, so
  pooling is structurally prevented rather than discouraged. Fallback is **off by default** in
  the evaluation plane — an eval trial that cannot reach its pinned provider **fails** rather
  than silently substituting.
- **Residual risk.** Low, given the `deployment_id` discipline. The residual is human: someone
  will want to pool the numbers to get a bigger n. The comparability guard must refuse, loudly.

---

### FM-12 — False causal inference: a spurious hypothesis is promoted

- **Failure.** The system presents a plausible, well-evidenced, wrong explanation, and someone
  acts on it.
- **Cause.** Temporal precedence plus a plausible mechanism is extremely persuasive and
  extremely weak. Confounding by deployment, load, or task difficulty. Multiple-comparison
  effects: running 30 detectors over one incident guarantees some fire by chance. Selection bias:
  we only analyse incidents, so incident-only patterns look causal.
- **Impact.** Directly destroys the project's central claim. A confidently wrong RCA is worse
  than no RCA, because it terminates investigation.
- **Detection.** Adversarial evaluation: **inject a known fault, then check whether the ranked
  hypothesis list contains the true cause and where.** Report top-1 and top-3 hit rate *and*
  the false-promotion rate on incidents where a *decoy* correlated event was injected alongside
  the true cause. The decoy arm is the part that matters; without it the evaluation only measures
  sensitivity, never specificity.
- **Mitigation.** The label lattice (§L.2) forbids a statistical test from producing a `FACT`.
  Refutation carries −4 in the rubric, so replay can sink a hypothesis. `INSUFFICIENT_EVIDENCE`
  is emitted when the top two candidates are within the margin. The word "root cause" is banned
  from all output templates and lint-enforced.
- **Residual risk.** **High.** This is a limit of the epistemics, not of the implementation.
  Multiple-comparison correction across detectors is `OQ-06` and currently unresolved.

---

### FM-13 — Replay fidelity illusion

- **Failure.** A counterfactual replay produces a clean, plausible result that describes a
  trajectory which never occurred and could not occur.
- **Cause.** Trajectory divergence (§N.1). The counterfactual changes a decision; recorded
  responses from the original trajectory continue to be served; everything downstream is
  off-policy fiction. The replay does not error. It looks like a success.
- **Impact.** Invalidates every counterfactual claim the system makes — which is the only
  genuine causal evidence it has. Undermines the one thing §D calls distinctive.
- **Detection.** Request-digest comparison at every replay decision point. `divergence_from_seq`
  recorded. `divergence_rate` carried on **every** `COUNTERFACTUAL_EVIDENCE` record, mandatory
  at the schema level.
- **Mitigation.** Halt-at-divergence when only recorded responses exist. Seeded deterministic
  stub with `post_divergence_fidelity=STUB` when one is configured. Counterfactuals with
  divergence rate > 0.5 report `INCONCLUSIVE`.
- **Residual risk.** Medium. The honest consequence is that many interesting counterfactuals are
  simply not answerable. Accepting a large `INCONCLUSIVE` rate is the correct behaviour and will
  be under continuous pressure from anyone who wants a cleaner demo.

---

### FM-14 — The LLM judge is unreliable and its unreliability is invisible

- **Failure.** An LLM grader's verdicts are biased (position, verbosity, self-preference),
  unstable across runs, or drift with the provider — and the eval reports its output as ground
  truth.
- **Cause.** Using a stochastic, unversioned, unvalidated component as a measuring instrument.
- **Impact.** Every downstream reliability number inherits the judge's error with no error bar.
- **Detection.** Judges are versioned artifacts with their own regression suite of human-labelled
  fixtures. Report inter-rater agreement against human labels (κ). Report judge self-consistency
  across repeated runs on identical input. **Any judge below a declared κ floor is disqualified
  from producing a pass/fail verdict.**
- **Mitigation.** Deterministic graders wherever the task admits one — and in this project most
  do, because outcomes are side effects in a store we own and can check exactly. LLM judges are
  restricted to genuinely subjective dimensions and are never the sole grader on a headline
  metric. Judge cost is also budget-prohibitive at eval scale (`QUOTA_AND_COST_MODEL` §4), which
  conveniently reinforces the right design.
- **Residual risk.** Low **in this project specifically**, because our environment is synthetic
  and oracle-checkable. This is a genuine structural advantage and should be used, not diluted by
  adding LLM judges for appearances.

---

### FM-15 — Benchmark leakage and evaluation hacking

- **Failure.** The agent scores well by exploiting the evaluation rather than by doing the task.
- **Cause.** Grader logic reachable from agent code paths. Task fixtures containing the expected
  answer. Reused environment state across trials leaving an earlier trial's answer in place. The
  agent reading the grader's assertion messages from a shared log.
- **Impact.** The headline number is meaningless and the failure is silent by construction —
  everything passes.
- **Detection.** A **negative control suite**: tasks that are impossible to complete correctly.
  Any pass on a negative control is a leak, full stop. Plus canary values placed in the
  environment that only grader code should ever touch — if they appear in agent context, there
  is a path that should not exist.
- **Mitigation.** Grader process isolation (TB-5): separate process, separate DB credentials, no
  shared filesystem, no overlapping environment variables, graders not importable from agent
  code. Fresh environment per trial from a snapshot digest. Balanced positive/negative cases.
- **Residual risk.** Medium. Isolation is only as good as the weakest shared resource, and shared
  resources accumulate silently. The negative-control suite is the backstop and must run on every
  eval, not on request.

---

### FM-16 — Shared state leaks across evaluation trials

- **Failure.** Trial N is affected by trial N−1. Paired counterfactual trials are not actually
  paired, so the measured effect is confounded.
- **Cause.** Environment not reset; caches warm; rate-limit buckets shared; RNG state carried
  over; database sequences not reset; the idempotency-key store retaining keys across trials
  (which would make a legitimate second trial silently deduplicate against the first).
- **Impact.** Destroys pairing, which destroys the one genuine intervention we have (§M.1). The
  numbers look fine.
- **Detection.** **Canary rows** written at trial start with a per-trial nonce: if a canary from
  trial N−1 is visible in trial N, isolation is broken. Assert the environment snapshot digest
  matches the expected baseline before every trial.
- **Mitigation.** Snapshot-restore by digest per trial (§N.2 — a Gate 1 requirement, not Gate 5).
  Per-trial seeded RNG derived from `(task_id, trial_index, fault_config)`. Per-trial
  idempotency-key namespace. Explicit cache and bucket reset.
- **Residual risk.** Low-medium, and entirely dependent on the environment being built
  snapshot-capable from the start. Retrofitting snapshot-restore is a rewrite, which is why it is
  pulled forward to Gate 1.

---

### FM-17 — Security policy bypass at the action boundary

- **Failure.** A high-risk action executes without a valid authorisation decision.
- **Cause.** A code path that dispatches without traversing the gateway — a direct tool call in a
  test helper, a "quick" debug path, a framework's built-in tool executor invoked directly, a
  compensation path that skips authorisation because it is "just cleanup". Or: policy enforced
  by prompt instruction rather than by runtime control.
- **Impact.** The system's central security claim is false. Every other control is downstream of
  this one.
- **Detection.** **The tool dispatcher is the only object holding the transport credential**, and
  it refuses to execute without a valid, signed `AuthorizationDecision` for that exact
  `action_id`. A bypass then fails closed rather than succeeding silently. Plus: an architectural
  test asserting no module other than the gateway imports the transport client.
- **Mitigation.** Capability-style construction — the credential is not reachable except through
  the gateway. Compensation actions traverse authorisation like any other action (§J.4). Policy
  decisions are recorded with `policy_version`; an action with no decision record is a detectable
  ledger inconsistency, checked by a nightly invariant scan.
- **Residual risk.** Medium. The architectural test catches accidental bypass. It does not catch
  a deliberate one, and it does not catch a bypass at a layer beneath us (see FM-01c).

---

### FM-18 — Sensitive data accumulates in telemetry and the CAS

- **Failure.** Prompts, tool payloads, retrieved documents and chain-of-thought accumulate in
  storage that was designed for debugging and inherits no data-protection controls.
- **Cause.** Root-cause analysis genuinely needs payloads (§H.4). The privacy-safe design
  (hashes only) is the RCA-useless design. This tension is real and cannot be engineered away;
  it can only be decided and bounded.
- **Impact.** The largest real-world liability in the system. Also the largest storage cost. Also
  the thing that makes the system unadoptable in a regulated environment if got wrong.
- **Detection.** Every CAS object carries a `sensitivity_class`; scan for unclassified objects.
  Retention-age monitoring per class. Redaction coverage tests using seeded synthetic PII that
  **must** be redacted — a redactor with no test corpus is decoration.
- **Mitigation.** Allow-list, never deny-list, for telemetry fields (`THREAT_MODEL` §Q/R).
  Digests in the envelope, payloads in a separately-governed CAS. Class-based retention.
  Chain-of-thought is **never a required field** and is `RESTRICTED` when present at all.
  The tested property: **delete the entire CAS and the system still functions**, with
  payload-dependent detectors degrading to `UNKNOWN` rather than crashing or silently passing.
- **Residual risk.** Medium. Redaction is best-effort and cannot be complete. The only robust
  control in this project is the standing rule that no real enterprise or personal data enters
  the system at all — synthetic only. That rule is a project constraint, not a technical control,
  and it is exactly the kind of rule that erodes under demo pressure.

---

### FM-19 — Clock skew and ordering corruption

- **Failure.** Events are ordered wrongly; precedence-based reasoning produces inverted causal
  candidates.
- **Cause.** Wall-clock timestamps across processes; NTP correction stepping backwards;
  containers with unsynchronised clocks; a producer buffering events and emitting them late.
- **Impact.** Corrupts the one condition the project's own causal rules declare *necessary*
  (`KNOWLEDGE_GRAPH.causal_evidence_rules`). Every hypothesis built on precedence becomes
  unsound, silently.
- **Detection.** Monitor `t_emit` vs. `t_recv` skew distribution; alert on negative or
  large-positive skew. Assert `seq` monotonicity per execution at ingest — a gap or a repeat is a
  hard error, not a warning.
- **Mitigation.** `seq` and `causal_parent_event_id` are authoritative; `t_emit` is advisory and
  lint-forbidden from cross-process precedence reasoning (§H.2). Where no causal edge exists and
  processes differ, the answer is `UNKNOWN` rather than a timestamp comparison.
- **Residual risk.** Low **while the control plane is single-writer**. The moment it is not, this
  reverts to high and requires Lamport clocks — which is why it is flagged now (T-8) rather than
  after scaling.

---

### FM-20 — Partial infrastructure failure produces plausible-but-wrong analysis

- **Failure.** The CAS is unavailable, or ingest is lagging, or one detector crashed — and the
  analysis completes anyway, over an incomplete ledger, with no indication that it is partial.
- **Cause.** Components treated as available rather than as possibly-degraded. A detector that
  throws and is caught into a "no anomaly" result. Analysis run over a ledger range that is still
  being written.
- **Impact.** Confident analysis of incomplete evidence. Structurally the same error as
  "absence of evidence as evidence of absence" (§L.3), arriving through an operational door
  rather than a logical one.
- **Detection.** Every analysis output carries a **coverage manifest**: which detectors ran, at
  which versions, over which `seq` range, with which CAS availability, and which were skipped or
  failed. Ingest-lag check before analysis: refuse if the execution's terminal event has not
  landed.
- **Mitigation.** A failed detector produces an explicit `UNKNOWN` evidence record naming the
  failure — **never** an absent record, and never a "no anomaly" result. CAS unavailability
  degrades payload-inspecting detectors to `UNKNOWN`, recorded in the manifest. Analysis over an
  incomplete execution is refused rather than caveated.
- **Residual risk.** Low-medium. Depends on every detector author honouring the
  "fail to `UNKNOWN`, never to silence" contract. Enforceable by a test harness that injects a
  detector crash and asserts an `UNKNOWN` record appears.

---

### FM-21 — Vendor and framework lock-in through the back door

- **Failure.** Business logic quietly acquires dependencies on one provider's response shape, one
  MCP revision, or one agent framework's abstractions.
- **Cause.** Provider response objects passed directly into core logic. Error taxonomies borrowed
  from an SDK. MCP types used as internal types. Retry semantics inherited from a framework.
- **Impact.** Directly violates a stated project constraint. Makes the provider-unavailable
  degradation path (§S) untestable, because the code cannot run without the SDK.
- **Detection.** An **import-boundary test**: core analysis modules may not import any provider,
  MCP, or framework package. Run in CI. This is a five-line test that prevents a six-month
  problem.
- **Mitigation.** Adapters normalise to internal types at the boundary. An internal error
  taxonomy independent of any SDK. MCP is one adapter behind an internal tool protocol, added at
  Gate 7 (`CHARTER` §F). A **null/deterministic provider adapter** exists from Gate 1 and the
  full test suite runs against it — which makes L3 degradation (§S) continuously tested rather
  than aspirational.
- **Residual risk.** Low, given the import-boundary test. It is one of the highest
  value-per-line controls available and should be written before any adapter exists.

---

### FM-22 — Grader bugs invalidate the evaluation silently

- **Failure.** A grader has a defect. Results are wrong. Nothing indicates it.
- **Cause.** Graders are code, and code has bugs, but graders are usually the only unverified
  component in an evaluation pipeline because there is nothing "beneath" them to check against.
- **Impact.** Every reliability number is wrong, in an unknown direction, with no error signal.
  This is the failure that invalidates the *evidence for everything else*.
- **Detection.** **Meta-evaluation**: each grader has a fixture suite of known-pass and
  known-fail cases, including adversarial near-misses, run in CI on every change. Graders are
  versioned; grader version is recorded with every result. Cross-grader disagreement on the same
  artifact is surfaced and **quarantined**, never averaged — averaging two graders that disagree
  produces a number that is wrong in a new way.
- **Mitigation.** Deterministic graders with oracle checks against the synthetic environment's
  ground-truth side-effect log. Grader changes invalidate cached results for the affected
  version. A human-calibration path for any subjective dimension.
- **Residual risk.** Medium. The fixture suite only covers anticipated cases. The negative-control
  suite (FM-15) is the independent backstop: if an impossible task ever passes, a grader is
  broken regardless of what its fixtures say.

---

## §3 Summary of residual risk

| Residual | Modes | Meaning |
|---|---|---|
| **High** | FM-01, FM-04, FM-08, FM-12 | Cannot be engineered away within this project. Must be stated in every report that touches them. |
| **Medium** | FM-02, FM-05, FM-06, FM-07, FM-13, FM-15, FM-17, FM-18, FM-22 | Reducible with discipline; will regress without enforced tests. |
| **Low** | FM-03, FM-09, FM-10, FM-11, FM-14, FM-16, FM-19, FM-20, FM-21 | Controlled, provided the control is built at the stated gate and not deferred. |

**The four High rows are the honest summary of this project's limits.** FM-04 is the one that
decides whether the product is worth building; FM-12 is the one that decides whether its output
can be trusted; FM-01 and FM-08 are the two where the synthetic environment will flatter us most.
