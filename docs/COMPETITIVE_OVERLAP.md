# COMPETITIVE OVERLAP

Status: **GATE 0.5.** Document version 0.1.0.

This document exists because the pivot was triggered by discovering that the previous thesis was
already solved. The purpose is to make that discovery repeatable: to state, before building, what
is borrowed, what is contested, and what would have to be true for this project to be redundant.

`ASSERTED_UNVERIFIED` applies to every product capability described below. These are
characterisations from general knowledge of the space, not from a current feature audit. Before
Gate 3, each **SEVERE** row requires a real check against current documentation (`OQ-02`).

---

## 1. Capabilities we CONSUME — build none of this

| Capability | Source | Our use |
|---|---|---|
| Distributed tracing, span trees, cost/latency attribution | OpenTelemetry + GenAI conventions; OpenLLMetry / OpenInference | Emit OTel-shaped telemetry. Our ledger is a superset with experiment fields. |
| Property-based and stateful test generation | Hypothesis (Python), QuickCheck family | Trial generation for states and sequences. |
| Covering-array generation | NIST ACTS, PICT, CAgen; published IPOG / IPOG-C algorithms | The planner. We implement or wrap; we do not invent. |
| Delta debugging | Published ddmin; C-Reduce, Perses lineage | The reducer's core loop. We adapt for stochasticity; the algorithm is theirs. |
| Statistical primitives | Wilson intervals, McNemar, Benjamini–Hochberg | Standard. No novelty claimed. |
| Sandboxing, egress control | Container runtime, network policy | Assumed present. Not built. |
| Durable execution, compensation | Temporal, Restate, DBOS | **Not adopted, not rebuilt.** Deferred behind a trigger. |
| Crash/error grouping heuristics | ClusterFuzz, Sentry grouping literature | Informs C7. We add a stability metric; the problem is theirs and unsolved. |

---

## 2. Existing research and tools that OVERLAP — we are not first

Ordered by how badly each threatens the thesis.

### 2.1 SEVERE — deterministic simulation testing

**Who:** FoundationDB's simulation lineage; **Antithesis**; TigerBeetle's VOPR.

**What they do:** run a system inside a deterministic simulator, inject fault combinations,
explore the state space, find a violation, **minimize** it, and hand back a **deterministically
reproducible** trace. That is, in outline, the entire workflow in `INTERACTION_MODEL.md`.

**Honest assessment:** Antithesis sells the exact loop — explore, detect, minimize, reproduce —
with a far stronger determinism story than ours, because they control the scheduler and we do
not (`OQ-05`, T-7). If they ship an agent adapter with a tool-capability factor model, the
differentiation collapses to a factor list and an oracle set.

**What we would concede:** their determinism is better. Their reduction is better. Their
engineering is years ahead.

**What is left:** the **agent-specific factor model** (tool capability profile, context defect,
policy change, model config as interacting factors), the **agent-reliability invariant set**, and
the **stochastic regression artifact format**. That is a genuinely smaller claim than the pivot
brief implies, and the charter says so.

### 2.2 SEVERE — combinatorial interaction testing

**Who:** NIST ACTS and the CIT literature; PICT; CAgen; 25 years of published work including the
empirical result that most failures are triggered by interactions of few factors.

**Honest assessment:** covering arrays, constraint handling, and t-wise coverage metrics are a
mature, published, free field. **We are applying CIT to a new domain, not advancing CIT.** Any
document implying otherwise is wrong.

**What is left:** the factor model and the constraint set for tool-using agents, which is domain
work, not method work.

### 2.3 SEVERE — fuzzing infrastructure

**Who:** OSS-Fuzz, ClusterFuzz, AFL corpus/testcase minimization.

**Honest assessment:** the full SCREEN → REDUCE → FINGERPRINT → REGRESS loop **is** the fuzzing
loop. ClusterFuzz already does minimization, bucketing, regression-range bisection and
reproducer generation at enormous scale.

**What is left:** fuzzing operates on inputs to a deterministic program and asserts crashes or
sanitizer trips. We operate on a **factor space** around a **stochastic agent** and assert
**semantic invariants** about side effects and authorization. The oracle is the difference; the
loop is not.

### 2.4 HIGH — chaos engineering

**Who:** Gremlin, Chaos Mesh, Litmus, Toxiproxy.

**Honest assessment:** fault injection with blast-radius control is commodity. A skeptic reads
this project as "chaos engineering with a covering array and an oracle."

**What is left:** chaos tools inject *infrastructure* faults and measure *service* health. They
have no model of tool capability profiles, context staleness, policy narrowing, or agent
re-planning — and no per-trial determinism, which is what makes reduction possible.

### 2.5 HIGH — model-based testing of stateful systems

**Who:** TLA+/TLC, P, stateful Hypothesis, QuickCheck state machines, Jepsen (for a different
question).

**Honest assessment:** state-machine model-based testing with invariant checking is mature. Our
execution state machine (§I) is a small model by their standards.

**What is left:** these tools model systems you can specify. A stochastic planner is not
specifiable, which is why our model covers the **runtime around** the agent and treats the agent
as an environment-driven generator.

### 2.6 HIGH — failure bucketing

**Who:** ClusterFuzz grouping, Sentry grouping, the crash-dedup literature.

**Honest assessment:** unsolved in general, and solved-well-enough by incumbents for their
domains. C7's honest statement of collision/splitting is a restatement of their known problem,
not a solution.

### 2.7 MEDIUM — agent evaluation harnesses

**Who:** Inspect, promptfoo, DeepEval, LangSmith datasets, Braintrust, W&B Weave.

**Honest assessment:** these measure output quality on datasets. Adding a fault plugin is not
architecturally hard. If any of them ships a fault matrix and a reducer, the gap narrows sharply.

**What is left:** they have no determinism guarantee, no fixture snapshot/restore, no ground-truth
effect log, and therefore no path to reduction or to a reproducible artifact. That is a real
architectural gap, and it is also a gap any of them could close in a quarter.

### 2.8 SOLVED — the previous thesis

**Verified tool calls, postcondition MCP implementations, exactly-once middleware, runtime
execution-proof work, current agent-control standards.**

This is why version 0.1.0 was retired. It is now a **supporting mechanism** (§I, §J of
`ARCHITECTURE_SURFACE.md`), and a *factor* in the experiment space rather than a product.

---

## 3. Our integration focus — stated as narrowly as it can honestly be stated

1. **Tool capability profile (CP1–CP8) as a varied experimental factor.** Not an assumption, not
   a blocker, not a survey finding. A dimension you degrade deliberately to see what breaks.
2. **An oracle set expressed as agent-reliability invariants** (INV-1..INV-9) evaluable
   deterministically against an environment ground-truth effect log the agent cannot read.
3. **A regression artifact format for stochastic, stateful failures** carrying a measured
   reproduction rate, a derived repetition count, and explicit invalidation conditions.
4. **Coverage accounting that reports the unexplored space**, the infeasible fraction, the
   constraints responsible, and screening-vs-adaptive attribution separately.

Four items. Everything else in this system is borrowed, and the documents name the source.

---

## 4. What we will NOT claim

- Not that interaction effects in agents are undiscovered.
- Not that covering arrays, ddmin, metamorphic testing, or fingerprinting are our contributions.
- Not exactly-once against arbitrary external systems.
- Not causality beyond controlled intervention inside a fixture.
- Not production readiness from synthetic evidence.
- Not a single "agent reliability score".
- Not that a clean suite means an agent is safe — only that the explored region produced no
  violation at the stated sample sizes.

---

## 5. The redundancy test

This project should be **stopped** if any of the following becomes true. These are checks, not
worries.

| # | Condition | Check |
|---|---|---|
| R1 | A deterministic-simulation vendor ships an agent adapter with a tool-capability factor model. | Quarterly. |
| R2 | An agent eval harness ships fault matrices **plus** trial determinism **plus** a reducer. | Quarterly. |
| R3 | The four items in §3 turn out to be re-derivable in under two weeks by a competent team on top of Hypothesis + ACTS + ClusterFuzz. | **Assess at Gate 3, honestly.** If yes, the contribution is a config file, not a platform. |
| R4 | `OQ-08` shows the engine detects externally-authored seeded bugs at a low rate. | Gate 3. A discovery tool that only finds what its author imagined is not a discovery tool. |
| R5 | The deterministic substrate proves to be testing the harness rather than any agent behaviour. | Gate 6, when L-LOC/L-REM canaries first disagree with L-DET. |
