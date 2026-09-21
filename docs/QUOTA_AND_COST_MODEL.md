# QUOTA AND COST MODEL

Status: **GATE 0.** Document version 0.1.0.
Answers charter question **S** (usefulness without a provider) and supplies the arithmetic behind
`EVALUATION_STRATEGY` §O.7 and `ARCHITECTURE_SURFACE` §K.

---

## 0. Why this document exists before any code

`FACT`: inference budget is not an operational detail of this project. It is a **hard
architectural constraint** that eliminates entire designs. A plan that requires 21,000 remote
calls against a 1,000/day quota is not expensive — it is impossible, and discovering that at
Gate 3 instead of Gate 0 wastes the two gates in between.

The arithmetic below is done first, and the architecture is shaped to fit it.

---

## 1. Input budgets — and their status

| Provider | Stated limit | Label |
|---|---|---|
| Groq (several relevant models) | 30 RPM, 1,000 RPD, 8,000 TPM | `ASSERTED_UNVERIFIED` — model- and org-specific; from `PROJECT_CONTEXT.json`. |
| OpenRouter (free tier) | 50 requests/day | `ASSERTED_UNVERIFIED` — free pool and limits change. |

`DESIGN_DECISION`: **these numbers are used for planning arithmetic only. They are never
hard-coded.** The runtime discovers actual limits from provider response headers and enforces a
*locally configured* budget that is independent of them. Two separate mechanisms:

- **Local budget** — what we allow ourselves. Configured. Enforced before dispatch. Authoritative.
- **Provider limit** — what they allow. Discovered from headers. Used to *lower* the local budget
  when it is more restrictive, never to raise it.

This is the only arrangement that is correct when the provider silently changes its limits, which
`PROJECT_CONTEXT` explicitly warns they do.

---

## 2. Which limit actually binds — the result most plans get wrong

Three limits, and they bind at different timescales. Let **S** = average tokens per call
(prompt + completion).

### 2.1 Burst rate: TPM dominates RPM

```
effective_calls_per_minute = min( RPM , TPM / S )
                           = min( 30 , 8000 / S )
```

| S (tokens/call) | TPM-implied calls/min | Binding limit | Effective RPM |
|---|---|---|---|
| 266 | 30.1 | RPM and TPM tie | 30 |
| 1,000 | 8.0 | **TPM** | 8.0 |
| 2,000 | 4.0 | **TPM** | 4.0 |
| 3,000 | 2.67 | **TPM** | 2.67 |
| 4,000 | 2.0 | **TPM** | 2.0 |
| 8,000 | 1.0 | **TPM** | 1.0 |

**The crossover is S = 8000/30 ≈ 267 tokens per call.** Realistic agent calls — a system prompt,
tool schemas, conversation state, a tool result — are an order of magnitude larger than that.

> **`FACT`: at any realistic call size, the 30 RPM headline is irrelevant. TPM binds, and the
> real burst ceiling is roughly 2–4 calls per minute, not 30.** A retry/backoff policy tuned to
> 30 RPM will produce continuous 429s, which per `FM-03` is exactly how a throttle becomes an
> outage.

**Consequence for §K:** the local token bucket must meter **tokens**, not requests. A
request-metered bucket is calibrated to the wrong limit and will not protect anything.

### 2.2 Daily ceiling: RPD vs. TPM

```
tokens_per_day ≤ min( RPD · S , TPM · 1440 )
               = min( 1000 · S , 11,520,000 )
```

Crossover at **S = 11,520 tokens/call**.

- S < 11,520 → **RPD binds the day.** Daily ceiling = 1,000 calls.
- S > 11,520 → **TPM binds the day.**

At a planning value of S = 3,000: daily ceiling is 1,000 calls / 3,000,000 tokens, and
sustaining TPM-limited throughput (2.67 calls/min) exhausts RPD in **≈ 6.25 hours**.

> **Planning rule: assume ~1,000 calls/day and ~6 hours of continuous usable throughput. Plan
> everything against that, not against 30 RPM × 1440.**

### 2.3 OpenRouter free tier

50 requests/day is **5% of primary capacity**. It is an emergency narration path for roughly 25
incidents/day at 2 calls each. It is **not** a fallback for evaluation, and any plan that treats
it as spare capacity is off by a factor of twenty.

---

## 3. Where the tokens go — allocation

`DESIGN_DECISION`: the daily budget is partitioned in advance and enforced per partition. An
unpartitioned budget is consumed by whatever runs first, which will be evaluation, which will
starve incident analysis — the thing the system is for.

| Partition | Share | Calls/day | Purpose |
|---|---|---|---|
| Incident analysis (narration) | 40% | 400 | 0–2 calls per incident ⇒ ≈ 200–400 incidents/day. |
| Reliability sweep (`REMOTE_MODEL` subset) | 35% | 350 | The 3% remote sample of §O.7. |
| Model-drift canary | 5% | 50 | Fixed prompt set, scheduled (`FM-08`). |
| Development / debugging | 15% | 150 | Headroom. |
| Reserve | 5% | 50 | Never allocated; absorbs miscounting. |

Each partition has its own bucket. **Exhausting one partition never borrows from another** —
borrowing reproduces the starvation the partitioning exists to prevent.

---

## 4. The evaluation budget — where the architecture is forced

From `EVALUATION_STRATEGY` §O.7:

```
grid  = 10 task_classes × 5 perturbations × 7 fault_configs × k=10
      = 3,500 executions per deployment_id
```

`ASSUMPTION` (owner: architect; measure at Gate 1): ≈ 6 model calls per execution, S ≈ 3,000.

### 4.1 The naive plan, priced

```
3,500 executions × 6 calls      =  21,000 remote calls
21,000 × 3,000 tokens           =  63,000,000 tokens

Days at RPD 1,000/day           =  21.0 days   ← binding
Days at TPM ceiling (11.52M/day)=   5.5 days
```

**21 days for one sweep.** A sweep is required per `deployment_id` — per model change, per tool
schema change, per policy change. A single model alias update invalidates three weeks of work.

> **This is not a cost problem. It is an arithmetic impossibility, and it kills any design in
> which reliability is measured with a remote model in the loop.**

The honest options are: (a) shrink the grid until it is too small to support any conclusion
(§O.4: below ~50 instances per cell you cannot distinguish 40% from 70%); (b) pay for capacity,
which the project constraints exclude; or (c) **change what executes**.

### 4.2 The substrate split — the forced design decision

`DESIGN_DECISION`: the reliability harness runs predominantly against a
**`DETERMINISTIC_STUB`** planner. Not as a cost optimisation — as the only design that fits, and,
as `EVALUATION_STRATEGY` §O.6 argues, the design that measures what the project actually claims.

| Substrate | Share | Executions | Calls/exec | Remote calls |
|---|---|---|---|---|
| `DETERMINISTIC_STUB` | 90% | 3,150 | 0 | **0** |
| `LOCAL_MODEL` | 7% | 245 | 0 (local) | **0** |
| `REMOTE_MODEL` | 3% | 105 | 6 | **630** |
| **Total** | | 3,500 | | **630** |

```
630 calls  →  under one day's 1,000-call budget
           →  fits the 350-call reliability partition across two days
           →  leaves the incident-analysis partition untouched
```

**Result: 21 days → under 2 days, with the statistical power of the sweep preserved, because
90% of the grid exercises the runtime paths the project's claims are about.**

The 3% remote sample cannot support cell-level statistics. It is reported as a **behavioural
realism check** with its own n and interval, never pooled with the stub results
(`EVALUATION_STRATEGY` §O.6).

### 4.3 What this costs us, stated plainly

The stub planner does not produce the *distribution of situations* a real model produces. It
exercises the runtime correctly, but the runtime is being fed scenarios we designed. Reliability
measured this way is reliability of the runtime under a fault distribution and a scenario
distribution both authored by us.

That is a real limitation and it belongs in every report. It is also strictly better than the
alternative, which is a 20-instance remote sweep whose confidence interval is ±0.21 and which can
therefore detect nothing.

---

## 5. Incident analysis budget — the 0–2 call target

`PROJECT_CONTEXT` sets 0–2 remote calls per incident analysis. The TPM arithmetic makes this a
hard ceiling, not an aspiration.

### 5.1 Evidence bundle sizing, derived from TPM

TPM = 8,000. A call whose prompt is 6,000 tokens consumes 75% of a minute's entire budget and
stalls everything else — including any concurrent execution.

Budget per call: **≤ 3,000 tokens prompt, ≤ 1,000 tokens completion = 4,000 tokens.**
Two calls = 8,000 tokens = **exactly one minute of TPM.** One incident analysis stalls the
provider for one minute. That is the design constraint, and it is tight.

> **The evidence bundler is therefore a hard-capped compressor, not a serialiser.** It must select
> ≤ 3,000 tokens of evidence from a trace that may contain hundreds of events and megabytes of
> payload. Selection is deterministic and priority-ordered:

| Priority | Content | Approx. budget |
|---|---|---|
| 1 | Adjudicated action states and transitions for the failing action(s) | 400 tok |
| 2 | Top-3 ranked hypotheses with their rubric criteria and scores | 600 tok |
| 3 | Counterfactual results: n, effect, interval, divergence rate | 300 tok |
| 4 | Detector firings with detector id and version | 400 tok |
| 5 | Change-point events in window (deployment / schema / policy) | 300 tok |
| 6 | Truncated payload excerpts, redacted, only if budget remains | ≤ 1,000 tok |
| | **Total** | **≤ 3,000 tok** |

**Payloads are last and optional.** If the bundle overflows, payloads are dropped first. If it
still overflows, the excess is reported as `bundle_truncated: true` with the dropped categories
named — never silently trimmed, because silent truncation means the narration is describing a
subset the reader thinks is the whole.

### 5.2 Call budget enforcement

- Hard cap of 2 calls per incident, enforced by a counter, not by convention.
- The call is **idempotent on the bundle digest**: re-analysing an unchanged incident returns the
  cached narration and costs zero. Trivial to implement, and it removes the most common source of
  accidental budget burn (repeated viewing).
- Exceeding the cap is an `ABORTED` analysis with a partial report, not a silent extra call.

### 5.3 What 0 calls produces

Everything except the prose paragraph: adjudicated states, evidence records, ranked hypotheses
with criteria, counterfactual results, coverage manifest, timeline. See §S.

---

## S. How the system stays useful when the LLM provider is unavailable

### S.1 The structural claim, and how it is tested

`FACT` (by construction): the LLM appears at exactly one point in
`observe → verify → analyze → hypothesize → replay → decide` — after `decide`, as narration over
a bundle that is already complete. It is downstream of every conclusion.

Tested, not asserted, by charter criterion **S4**: run the full suite with provider egress
blocked; diff the structured output against a run with the provider available; **require
byte-identical structured output.** Any difference is an undeclared LLM dependency and is a bug.

`FM-21`'s import-boundary test (SC-2) plus a `DETERMINISTIC_STUB` adapter present from Gate 1
mean this path is exercised on **every CI run**, not on request. That is what makes it real
rather than a paragraph.

### S.2 The degradation ladder

| Level | Trigger (deterministic) | Behaviour | Output completeness |
|---|---|---|---|
| **L0** | Primary healthy, partition budget available | ≤ 2 calls, full narration | 100% |
| **L1** | Primary 429 sustained, breaker open, **or** partition ≥ 90% consumed | Switch to fallback. **`provider_switch` event emitted. `deployment_id` changes. Output marked.** | 100% structured; narration from a different model with different guarantees, explicitly labelled |
| **L2** | Fallback also unavailable or exhausted (50/day) | Local small model if configured | 100% structured; narration lower quality, labelled |
| **L3** | No inference available at all | **Template-rendered structured report** | 100% structured; **no prose paragraph** |

**L1 is never silent.** A silent fallback is a listed anti-pattern (`FM-11`) and it corrupts
every comparison downstream by mixing two distributions under one label.

### S.3 What L3 actually contains

This is the honest test of whether the architecture has an undeclared LLM dependency. At L3, an
incident report still contains:

- The full execution timeline with authoritative `seq` ordering
- Every action's adjudicated state and `outcome_basis` — including every `UNKNOWN_OUTCOME` and
  every `ESCALATE`, with the capability-matrix verdict that produced it
- Duplicate-side-effect determination from the ground-truth probe
- Every detector firing, with detector id, version, parameters, and source refs
- The ranked hypothesis list with each rubric criterion and its points shown
- Counterfactual replay results: n, effect, interval, divergence rate — or `INCONCLUSIVE`
- The coverage manifest: what ran, at what version, over what range, what was skipped
- Policy decisions with `policy_version`
- Budget and rate-limit accounting

**What is missing at L3 is one paragraph of English.**

> **If the L3 report is not compelling on its own, the architecture has an LLM dependency it has
> not admitted to, and that is a Gate-0 finding — not a polish item for later.**

This is the single sharpest test in the whole design, and it should be run against a stakeholder,
not just against CI: show someone the L3 report and see whether they ask where the summary is or
whether they start reading the evidence.

### S.4 Why this is not a hedge

The system's outputs with real consequence — *did this side effect happen exactly once*, *is this
retry legal*, *did removing this fault change the outcome*, *is this policy decision correct* —
are **all** deterministic. An LLM cannot make any of them more correct, and rule 5 forbids it from
making them at all.

The LLM's contribution is compressing a correct structured report into readable prose. That is
genuinely useful and completely non-essential. Building the system so that the non-essential part
is the only part that can fail is not a hedge against outages — it is the correct dependency
ordering, and the outage-resilience falls out of it for free.

---

## 6. Cost accounting

Every execution and every analysis carries a `CostRecord`:

```
CostRecord
  remote_calls        int
  prompt_tokens       int
  completion_tokens   int
  provider, model, deployment_id
  partition           enum     # which budget partition was charged
  degradation_level   enum     # L0 | L1 | L2 | L3
  budget_remaining_at_start  int
```

Reported per execution, per incident, per sweep, and per day. Two derived metrics are tracked
as reliability signals in their own right:

- **`calls_per_incident`** — must stay ≤ 2. A rise means the bundler is failing to compress, or
  someone added a call.
- **`L3_rate`** — the fraction of analyses completed with zero inference. A *high* L3 rate is not
  a problem; it is evidence that S.1 holds. It should be reported with pride rather than treated
  as a degradation statistic.

---

## 7. Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-10` | Actual tokens-per-call (S) for the reference agent. Every number here scales with it and all of it is currently an assumption. Must be measured at Gate 1 and this document revised. | Gate 3 planning |
| `OQ-11` | Whether the provider exposes token-level rate headers or only request-level. §K's bucket must meter tokens (§2.1); if headers are request-only, local estimation carries error that must be quantified and buffered. | Gate 2 |
| `OQ-12` | Whether a `LOCAL_MODEL` substrate is available in the target environment at all. If not, the ladder collapses from four levels to three (L0/L1/L3) and the 7% local share of §4.2 must move to `DETERMINISTIC_STUB`. | Gate 3 |
