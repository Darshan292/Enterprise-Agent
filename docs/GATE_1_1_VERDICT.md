# GATE 1.1 — VERDICT

Status: **TERMINAL FOR THE PRODUCT LINE.** Version 1.0.0 · 2026-09-22

---

## Verdict

> **Outcome B — a research finding with no defensible product thesis — and consequently
> Outcome C for the product line: permanently closed.**
>
> **No replacement thesis is nominated.**

| Question | Answer |
|---|---|
| Does a genuinely distinct, technically deep problem remain underneath? | **No.** |
| Is anything worth restarting around? | **No, at present.** One candidate (P-2) is recorded with an explicit re-examination condition. |
| Is the repository worth preserving? | **Yes** — as research lineage and as a case study. |
| Should any code be written? | **No.** None ever was. |

---

## The three findings that produce this verdict

**1. The competitive gap is one thin `D` out of forty capabilities.**
65 % class A (solved externally), 30 % composition, one genuine gap — intervention reachability —
which is a *property*, not a workflow, has no owner, and is largely constructible from mature
methods. (`COMPETITIVE_GAP_MATRIX.md`)

**2. The strongest surviving technical idea is not novel in any surveyed domain.**
"Intervene before attributing, then refuse when identification is impossible" is the founding
distinction of causal inference, the defeater structure of 30-year-old assurance cases, and the
operating loop of automated experimentation — and is now implemented in our own domain, with a
*finer* refusal vocabulary than ours. (`INTERVENTION_NOVELTY_REVIEW.md`)

**3. Every "deep problem" the project exposed has a weak answer to "why isn't this already
solved?"**
Seven of nine weak, two moderate, none strong. The recurring shape: **the theory is mature and the
gap is that practitioners do not apply it.** A discipline gap is closed by a specification or a
paper, not an engine — and the specification position is occupied.
(`CANDIDATE_PROBLEM_SPACE.md` §3)

---

## The finding worth keeping

> **Nothing across five gates was technically deep. Everything was the correct application of
> standard methods. The intellectual content of this project was rigor — and rigor is not a
> product.**

Supporting evidence, which is unusually direct: **a project explicitly built to detect invalid
evaluations committed four validity defects in its own decision instruments, each surviving at
least one adversarial review pass, each caught by re-deriving a number rather than by any
process.** Non-identifiable requirement (0.75) · biased estimator (0.9) · underpowered primary
(0.95) · required-n cited as available-n (1.0).

That is the strongest empirical evidence the project produced for the reality of the defect class
it set out to address — and simultaneously an argument against the product, because the defects
were caught by **a person choosing to check the arithmetic**, not by tooling.

---

## What is preserved

| Artifact | Reusable by |
|---|---|
| **The four-defect catalogue with the arithmetic that caught each** | Anyone running a methodologically serious evaluation |
| **C0/C1/C2 baseline tiering** | Anyone asking "does my tool beat a competent composition?" — the single most transferable pattern here |
| **Covering-array vs random arithmetic (1.7×, not 10⁵×)** | Anyone told covering arrays are transformative |
| **Frequency-derived automation thresholds** (~32 packages/yr maintenance break-even; ~215 cumulative payback) | Anyone justifying an automation build |
| **Required-n vs available-n error pattern** | Anyone pre-registering an experiment |
| 42 gate artifacts, dated, with pre-registered predictions recorded before outcomes | A case study in a thesis narrowing to nothing under honest review |

---

## The process lesson

```
Gate 0    runtime assurance              -> solved elsewhere
Gate 0.5  interaction discovery          -> covering arrays worth 1.7x over free
Gate 0.75 factor-model adequacy          -> residual diagnostics, established
Gate 0.9  evaluation-integrity auditing  -> 5 of 5 dimensions have mature analogues
Gate 0.95 cross-dimension vs composition -> the baseline is 3 weeks of work
Gate 1.0  automation of a known capability -> a cost point, contingent on frequency
Gate 1.1  the cost point                 -> occupied
```

Seven gates, monotonic narrowing, never a widening.

> **A claim that narrows at three consecutive adversarial gates without once widening is being
> progressively falsified, and the shape of the sequence is evidence before the endpoint is
> reached. Stop at the third and ask what is left, rather than running to exhaustion.**

We ran to exhaustion. **The artifacts are better for it. The calendar is not.** That trade is worth
making once, deliberately, and not by accident.

---

## What would reopen this

Recorded as conditions, not as a plan. None is being pursued.

| # | Condition |
|---|---|
| R1 | **MQ-EVA is verified not to exist, or to be materially weaker than reported.** Then Gate 1.0's conditional (B) returns — still gated on `OQ-25` volume and `OQ-27` adapter cost, both of which were already capable of killing it. |
| R2 | A verified 2026 survey of causal-RCA tooling shows none performs identifiability analysis, **and** a named team commits to using it (P-2). |
| R3 | A named owner appears with a measured ≥150 evaluation packages/year and a programmatic release gate. **This was K-7 and went unanswered across three gates.** |

**R1 is the one to check first if this is ever revisited** — because it is the only unverified
fact the termination touches, and because verifying it costs an afternoon.

---

## Final state

```
product line:            CLOSED
thesis:                  KILLED (not pivoted)
replacement thesis:      NONE NOMINATED
repository:              PRESERVED as research lineage
implementation_authorized: false  (never true at any gate)
application code written:  0 lines, across seven gates
```
