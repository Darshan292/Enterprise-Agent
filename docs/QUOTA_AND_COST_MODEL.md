# QUOTA AND COST MODEL

Status: **GATE 0.5.** Document version 0.2.0 (full rewrite; supersedes 0.1.0).

---

## 0. What changed, and why it matters more than it sounds

Version 0.1.0 had a reliability sweep costing **21,000 remote calls against a 1,000/day budget —
21 days per sweep**. That arithmetic was the binding constraint on the whole design.

In the pivoted architecture **the critical path contains zero LLM calls**. Planning, running,
oracle evaluation, reduction, fingerprinting, interaction estimation and regression compilation
are all deterministic (`PROJECT_CHARTER` §D.10).

The cost model therefore changes shape completely:

| | 0.1.0 | 0.2.0 |
|---|---|---|
| Dominant cost | Remote inference | **Trial compute** |
| Binding constraint | Provider RPD | **Reduction wall-clock** |
| LLM role | In the measurement loop | **Fixed canary + optional narration, off the critical path** |
| Failure mode | Arithmetically impossible sweep | Reduction backlog |

> **The cost centre is the reducer, not the planner and not the provider.** That is the single
> most counter-intuitive number in this document and the one most likely to be planned around
> wrongly.

---

## 1. Trial cost — the dominant term

`ASSUMPTION` (owner: architect; **measure at Gate 1**, `OQ-14`): 2 s per deterministic trial,
comprising fixture restore, agent run against the stub planner, oracle evaluation, ledger write.

### 1.1 Phase costs

| Phase | Design | Trials | @2 s serial | Remote calls |
|---|---|---|---|---|
| **Screen (pairwise)** | CA ≈ 45 rows × r = 5 | 225 | **8 min** | **0** |
| **Screen (3-way)** | CA ≈ 350 rows × r = 5 | 1,750 | **58 min** | **0** |
| **Adaptive expansion** | Hamming d=1 around each failing row | ≈ 40 × failures | varies | **0** |
| **Reduce** (per failure) | ddmin, `r_confirm` = 5 | 265 typical – 980 worst | **9–33 min** | **0** |
| **Estimate** (per surviving interaction) | Phase-3 factorial, 18 cells × n=20 | 360 | **12 min** | **0** |
| **Regression execution** (per artifact) | `repetitions` from §3 of the artifact spec | 6–59 | **< 2 min** | **0** |

### 1.2 The inversion, quantified

```
Screening a 7-million-configuration space:      8 minutes
Reducing 50 failures found by that screen:      13 – 27 hours
```

A screen that finds more failures makes the run **slower**, not faster. Planning capacity without
reduction capacity produces a backlog, not results.

`DESIGN_DECISION`: `max_reduction_trials` per failure is a **required field** on every experiment
manifest. Hitting it produces a `PARTIALLY_REDUCED` artifact, honestly labelled — never a claim
of minimality the budget did not buy.

### 1.3 Scaling levers, in order of preference

| Lever | Effect | Cost |
|---|---|---|
| Cap reduction budget per failure | Bounds the backlog directly | Larger minimized configurations |
| Deduplicate by fingerprint **before** reducing | Reduce once per cluster, not once per trial | Depends on fingerprint quality (`EM-04`) |
| Parallel reduction across failures | Near-linear | **Reintroduces `EM-12` (shared state)**; requires per-worker fixture isolation |
| Reduce trial wall-clock | Linear on everything | Engineering; measure first (`OQ-14`) |
| Raise `r_confirm` | Better reduction stability | **Linear cost increase.** The wrong lever to pull first. |

---

## 2. LLM budgets — now a side channel

`ASSERTED_UNVERIFIED` (from `PROJECT_CONTEXT.json`; never hard-coded — see §2.3):

| Provider | Stated limit |
|---|---|
| Groq, several models | 30 RPM, 1,000 RPD, 8,000 TPM |
| OpenRouter free tier | 50 requests/day |

### 2.1 TPM binds, not RPM

Let **S** = average tokens per call.

```
effective_calls_per_minute = min(30, 8000 / S)
```

| S | TPM-implied calls/min | Binding |
|---|---|---|
| 267 | 30.0 | crossover |
| 1,000 | 8.0 | **TPM** |
| 3,000 | **2.67** | **TPM** |
| 8,000 | 1.0 | **TPM** |

> `FACT`: at any realistic agent call size, the 30 RPM headline is irrelevant. The real burst
> ceiling is ≈ 2–4 calls/minute. A backoff policy tuned to 30 RPM produces continuous 429s, which
> is how a throttle becomes an outage.

**Consequence:** the local bucket meters **tokens**, not requests. A request-metered bucket is
calibrated against the wrong limit and protects nothing.

### 2.2 Daily ceiling

```
tokens_per_day ≤ min(RPD · S, TPM · 1440) = min(1000·S, 11,520,000)
```
Crossover at S = 11,520. Below that, **RPD binds**: ≈ 1,000 calls/day, and sustaining the
TPM-limited rate exhausts RPD in **≈ 6.25 hours**.

### 2.3 Limits are discovered, never hard-coded

Two separate mechanisms:
- **Local budget** — what we allow ourselves. Configured, enforced before dispatch, authoritative.
- **Provider limit** — discovered from response headers. Used to **lower** the local budget when
  more restrictive, never to raise it.

This is the only arrangement that stays correct when a provider silently changes its limits.
`OQ-15`: whether the provider exposes token-level headers or only request-level. If request-level
only, local token estimation carries error that must be quantified and buffered.

---

## 3. LLM partitions

The engine does not consume these. Only the canary layers and narration do.

| Partition | Calls/day | Purpose |
|---|---|---|
| **L-REM canary suite** | 500 | Fixed, declared task set. `EVALUATION_STRATEGY` O.2. |
| **Narration** | 200 | ≤ 2 calls per report ⇒ ≈ 100 reports/day. |
| **Development** | 200 | Headroom. |
| **Reserve** | 100 | Never allocated; absorbs miscounting. |

Each partition has its own bucket. **Exhausting one never borrows from another.**

### 3.1 L-REM canary sizing

`ASSUMPTION`: 6 calls per execution, S ≈ 3,000.

```
10 declared tasks × 5 repetitions × 6 calls = 300 remote calls per canary run
```

Inside the 500-call partition, leaving room for a second run or a re-check the same day.

**What 300 calls buys, stated honestly:** at n = 5 per task, the Wilson half-width near p = 0.5 is
**± 0.33**. That is enough to notice a catastrophic shift and nothing else.

> **L-REM is a smoke alarm, not a measurement.** It is reported as a distribution on a date. It is
> never compared across dates as though the difference were an effect, and it never enters an
> interaction estimate.

### 3.2 Narration sizing

TPM = 8,000. A 6,000-token prompt consumes 75% of a minute's entire budget.

Budget per call: **≤ 3,000 tokens prompt, ≤ 1,000 completion**. Two calls = 8,000 tokens =
**exactly one minute of TPM**.

The evidence bundler is therefore a hard-capped compressor, deterministic and priority-ordered:

| Priority | Content | Budget |
|---|---|---|
| 1 | Invariant violated, violation site, oracle id + version | 300 tok |
| 2 | Minimized factor assignment + `minimality_confidence` | 400 tok |
| 3 | Fingerprint + `fingerprint_stability` | 200 tok |
| 4 | Interaction estimate: coefficient, interval, adjusted p, family size | 400 tok |
| 5 | Coverage summary + **unexplored space** | 400 tok |
| 6 | Reduction steps, abridged | 300 tok |
| 7 | Redacted payload excerpts, **only if budget remains** | ≤ 1,000 tok |
| | **Total** | **≤ 3,000 tok** |

Payloads are last and optional. Overflow sets `bundle_truncated: true` with the dropped categories
named — never silently trimmed, because silent truncation means the narration describes a subset
the reader believes is the whole.

Narration is **idempotent on the bundle digest**: re-reading an unchanged failure returns the
cached text and costs zero. This removes the most common source of accidental budget burn.

---

## 4. Degradation

| Level | Trigger | Behaviour | Structured output |
|---|---|---|---|
| **L0** | Primary healthy, partition available | ≤ 2 narration calls per report | 100% |
| **L1** | Sustained 429, breaker open, or partition ≥ 90% | Fallback provider. **`provider_switch` event emitted; `deployment_id` changes; output marked.** | 100% |
| **L2** | Fallback unavailable or exhausted (50/day) | Local model if configured | 100% |
| **L3** | No inference at all | **Template-rendered report** | **100%** |

**L1 is never silent.** A silent fallback mixes two distributions under one label and corrupts
every comparison downstream.

### 4.1 What L3 contains

Everything. Coverage report with the unexplored space; every oracle verdict with version; every
failure with fingerprint, stability and minimized assignment; every reduction history; every
interaction estimate with interval, adjusted p and family; every regression artifact with its
reproduction rate and invalidation conditions; the full external-validity limitation.

**What is missing at L3 is one paragraph of English.**

> **If the L3 report is not compelling on its own, the architecture has an LLM dependency it has
> not admitted to — and in this design that would be a Gate-0.5 finding, not a polish item.**

Criterion **S5** tests this on every CI run: provider egress blocked, structured output must be
byte-identical.

### 4.2 Why this is not a hedge

Everything with consequence — which factor combinations were explored, which invariants were
violated, which configuration is minimal, whether an artifact reproduces — is deterministic. An
LLM cannot make any of them more correct, and the capability model prevents it from making them at
all (`THREAT_MODEL` §1).

The LLM compresses a correct structured report into readable prose. Useful, and completely
non-essential. Building the system so the non-essential part is the only part that can fail is
the correct dependency ordering; outage resilience falls out of it for free.

---

## 5. Cost accounting

```
TrialCost      trial_id, wall_clock_ms, fixture_restore_ms, oracle_eval_ms, remote_calls (=0 in L-DET)
ReductionCost  failure_id, trials_used, budget, terminated_by ∈ {CONVERGED, BUDGET, UNSTABLE}
LLMCost        remote_calls, prompt_tokens, completion_tokens, provider, model,
               partition, degradation_level, budget_remaining_at_start
```

Four reported metrics, each a health signal in its own right:

| Metric | Healthy | Meaning of drift |
|---|---|---|
| `reduction_hours / screening_hours` | Expected ≫ 1 | If it approaches 1, screening is not finding much |
| `terminated_by=BUDGET` fraction | Low | Rising ⇒ the reduction budget is too small and artifacts are degrading |
| `calls_per_report` | ≤ 2 | Rising ⇒ the bundler is failing to compress, or someone added a call |
| **`L3_rate`** | **High is good** | The fraction of reports produced with zero inference. Evidence that §4.2 holds. **Reported with approval, not as a degradation statistic.** |

---

## 6. Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-14` | Actual wall-clock per trial. **Every number in §1 scales with it** and all of it is currently an assumption. | Gate 3 planning |
| `OQ-15` | Token-level vs. request-level rate headers. §2.1 requires token metering. | Gate 6 |
| `OQ-16` | Is a local model available in this environment at all? If not, the ladder loses L2 and drops to L0/L1/L3, and the L-LOC evidence layer does not exist. | Gate 6 |
| `OQ-17` | Parallel reduction: how are per-worker fixtures isolated without multiplying `EM-12` risk? The obvious answer (a fixture copy per worker) multiplies T-3 storage. | Gate 4 |
