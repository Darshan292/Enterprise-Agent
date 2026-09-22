# CANDIDATE PROBLEM SPACE

Status: **GATE 1.1.** Version 0.1.0.

**Standing instruction honoured: do not force a pivot.** Outcome C — permanent closure — is
acceptable, and a replacement generated merely to avoid a kill is worse than a kill. Every
candidate below is reported with its rejection, including the ones that came closest.

---

## 1. Screening criteria

A candidate must satisfy **all** of:

real operational pain · current in 2026 · technically difficult · not a dashboard/reporting layer ·
meaningful algorithmic content · deterministic substrate where possible · LLM optional ·
demonstrable locally · useful to serious engineering teams · measurable failure mode · credible
baseline · **survives scrutiny against existing 2026 systems**

Automatically rejected: generic RAG · another agent framework · generic observability · another
evaluator · another benchmark · another chaos framework · **"integration" as novelty** ·
"AI-powered" versions of existing developer tools.

**And the rule that killed the last thesis:** *constructible straightforwardly from mature
methods* ⇒ **composition, not a thesis.**

---

## 2. Candidates derived from the project's own failure history

### P-1 — Intervention reachability as a computed system property

> Determine which of a system's variables can be **surgically** intervened on — set independently
> without perturbing others — and expose it as a computed property, prior to any investigation.

| Criterion | Assessment |
|---|---|
| Real pain | **Weak-moderate.** For most systems the answer is "read the config schema." The interesting cases may be a small minority. |
| Technically difficult | Partially — static analysis of configuration surfaces plus empirical perturbation probing |
| Algorithmic content | Moderate; structural identifiability theory is the relevant body |
| Baseline | None computes it — but nobody asks for it either |
| **Survives 2026 scrutiny** | **Concept survives. Product does not.** |

**REJECTED.** A property, not a workflow. No trigger event, no owner, no budget line — the same
K-7 that went unanswered for three gates. Structural identifiability and optimal experimental
design are mature; applying them here is `COMPOSITION` under our own rule.

*Condition to re-examine:* a named team states they cannot A/B-test something and do not know why.
That is an anecdote, not a market.

### P-2 — Instrumentation gap analysis ("what would I need to record to tell these apart?")

> Given telemetry and two candidate explanations, compute whether they are distinguishable from
> what is recorded, and if not, output the minimum additional instrumentation or intervention that
> would separate them.

| Criterion | Assessment |
|---|---|
| Real pain | **Strong.** SREs hit "we cannot tell from what we have" constantly. |
| Technically difficult | **Yes** — structural identifiability over a causal graph plus cost-bounded experiment design |
| Algorithmic content | **Real** |
| Deterministic | Yes |
| LLM | Optional |
| Baseline | Intuition and a senior engineer |
| **Survives 2026 scrutiny** | **Uncertain — and I cannot resolve it** |

**REJECTED, with the strongest residual regret of any candidate.** Three reasons:

1. **`COMPOSITION` under our own rule.** Structural identifiability is mature in control theory
   and systems biology; optimal experiment design is mature. Porting them to observability is
   application.
2. **It requires a causal graph of the system** — which nobody has, and constructing one is the
   actual hard problem. The candidate assumes away its own difficulty.
3. **I cannot verify the 2026 state of observability tooling.** Claiming a gap I cannot check
   would repeat exactly the error that produced five gates of narrowing.

*Condition to re-examine:* a verified survey of 2026 causal-RCA tooling showing none performs
identifiability analysis, **plus** a named team that would use it.

### P-3 — Statistical-validity linting for analysis code

> Detect methodological defects in analysis pipelines: wrong estimator, unit-of-analysis errors
> (clustering ignored), post-hoc family redefinition, required-n cited as available-n,
> underpowered claims.

| Criterion | Assessment |
|---|---|
| Real pain | **Demonstrated — by this project, four times** |
| Technically difficult | **Mixed.** Some checks trivial (n_required vs n_available); some hard (detecting ignored clustering) |
| Baseline | **statcheck exists** and there is a body of automated statistical-error-detection work |
| Audience | Researchers — **no budget line** |
| **Survives scrutiny** | **No** |

**REJECTED.** Partially solved, narrow, and the audience has no money. It is also suspiciously
close to "another evaluator," which is on the automatic-rejection list.

The *observation* behind it — that four defects arose in a project built to prevent them — is
valuable. The **tool** is not.

### P-4 — Deterministic replay of agent systems with live external dependencies

**REJECTED immediately.** Deterministic simulation testing owns this with a far stronger story
(they control the scheduler; we established at Gate 0.5 that we do not). Record/replay is mature.
`A`, not `D`.

### P-5 — Evidence-provenance for AI claims

**REJECTED immediately.** Supply-chain attestation and provenance frameworks are mature, and
MQ-EVA covers the evaluation-specific case (`COMPETITIVE_GAP_MATRIX` §1.1 #4, #12).

---

## 3. Section 4 of the brief: the deepest recurring problem

For each candidate the brief names: *"why is this not already a solved engineering problem?"* —
**and a weak answer is a rejection.**

| Candidate | Why not already solved? | Answer strength | Verdict |
|---|---|---|---|
| **Non-identifiability** | It *is* solved — formally, in causal inference. The gap is application discipline. | **Weak** | **REJECT** |
| **Confounding** | Solved: DOE, randomisation, causal inference. | **Weak** | **REJECT** |
| **Evidence provenance** | Largely solved: attestation, provenance graphs, assurance cases. | **Weak** | **REJECT** |
| **Experiment validity** | A surveyed field with an implemented system (MQ-EVA). | **Weak** | **REJECT** |
| **Intervention reachability** | Not named as an engineering property anywhere surveyed. | **Moderate** — but reduces to "is the config settable" in most cases | **REJECT (P-1)** |
| **Model misspecification** | Solved in statistics; diagnostics are mature. | **Weak** | **REJECT** |
| **Hidden state** | The general problem is undecidable; specific instances are ordinary engineering. | **Weak** (undecidability is not a market) | **REJECT** |
| **Observability limits** | Partly the same as identifiability; the causal-graph construction problem is the real obstacle and is unsolved because it is *hard*, not because it is neglected. | **Moderate** | **REJECT (P-2)** |
| **Evaluation gaming** | NIST documents it; MQ-EVA detects it. | **Weak** | **REJECT** |

**Nine candidates. Seven weak answers, two moderate, zero strong.**

### 3.1 The pattern across all nine

Every "deepest problem" this project exposed has the same shape:

> **The theory is mature. The gap is that practitioners do not apply it. The gap is discipline,
> not technique.**

That pattern is genuine and is the project's real finding. **But a discipline gap is addressed by
a specification, a checklist or a paper — not by an engine — and the specification position in
this domain is occupied.**

---

## 4. What the project's own history says about its intellectual content

An uncomfortable audit. What was actually hard across five gates?

| Contribution | Technically deep? |
|---|---|
| Covering-array vs random arithmetic (1.7×) | No — arithmetic anyone could do, and should have |
| Clustering / design-effect correction | No — standard |
| Identifiability trap in B-20 | No — standard, and we got it *wrong* first |
| C0/C1/C2 baseline tiering | No — standard experiment design |
| Frequency-derived automation thresholds | No — arithmetic |
| Required-n vs available-n catch | No — a **process** failure, not a technical problem |
| Ambient-observable residual method | No — residual diagnostics |
| Zero-unfavourable-discordance criterion | Mildly — a sensible substitution under a power constraint |

> **Nothing in five gates was technically deep. Everything was the correct application of standard
> methods.** The intellectual content of this project was **rigor**, and rigor is not a product.
>
> This is the single most useful sentence in the excavation, and it applies to every candidate
> above: each one that survived initial screening failed because the hard part was already solved
> by someone else and the remaining part was discipline.

---

## 5. Outcome

| Allowed outcome | |
|---|---|
| **A — Strong new thesis discovered** | **Not achieved.** Five candidates examined, all rejected: three immediately, two with reasons recorded. |
| **B — Research finding, no defensible product thesis** | **ACHIEVED.** §4, plus the four-defect catalogue and the reusable artifacts in `CURRENT_THESIS_TERMINATION` §5. |
| **C — Project permanently closed** | **RECOMMENDED for the product line.** |

**No candidate is nominated for restart.** P-2 is recorded with an explicit re-examination
condition, not as a plan. Nominating it would be generating a replacement to avoid a kill, which
the brief forbids and which would be the fifth self-inflicted validity defect in a project already
carrying four.
