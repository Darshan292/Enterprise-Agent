# COMPETITIVE AND RESEARCH REVIEW — 2026

Status: **GATE 1.0.** Version 0.1.0. Supersedes `COMPETITIVE_OVERLAP.md`.

**Epistemic status of this document.** Every external system and publication below is
`ASSERTED_UNVERIFIED` — supplied in the brief or characterised from general knowledge of the
space, and **not** verified against current documentation by anyone on this project. The
assistant's own knowledge ends before several of the dated items. **`OQ-24` blocks Gate 1.0 exit
on reading the primary sources.**

**Standing rule, carried forward:** *"nobody has built this"* is invalid reasoning and appears
nowhere below.

---

## 1. No novelty is claimed for

Fault injection · contamination detection · trajectory analysis · constraints · audit trails ·
interaction testing · reduction · statistical testing · replay · mutation testing · model adequacy ·
DOE · metamorphic testing · grader auditing.

These are **mechanisms**. They are cited as prior art, used where useful, and claimed nowhere.

---

## 2. The 2026 landscape, and what each covers

| System / work | What it provides | Which of our steps it closes |
|---|---|---|
| **NIST — agentic evaluation probes and audit trails** | Standards for *what to record* during agent evaluation, and a trail that can be inspected afterwards | **Observability supply.** Raises the floor on input quality; performs no analysis. **Strongly complementary: it makes our minimum-observability requirements more likely to be met.** |
| **NIST — evaluation cheating / grader gaming / contamination** | Names and characterises the defect classes; detection guidance for contamination and grader gaming | **Anti-cheating detection**, and — more importantly — **institutional legitimation of the problem.** |
| **Microsoft AgentRx** | Automated trajectory diagnosis | **Trajectory analysis.** Diagnoses *the agent*. Different object from *the evaluation*, and one adapter from overlapping. |
| **AgentChaos-class** | Agent fault injection | **Intervention mechanism.** Consume, do not rebuild. |
| **Current agent-evaluation frameworks** | Run evaluations, compute scores, manage datasets, run judges | **The host.** Any of them could add an assurance layer. |
| **Systematic survey on validity-centered agent evaluation (Sept 2026)** | Names and organises the field | **The framing itself.** See §5. |

### 2.1 What the survey's existence implies

A systematic survey means the field has a **name**, a **literature**, and — as surveys invariably
carry — an **enumerated open-problems list**.

Three consequences, and the third is the serious one:

1. **Problem identification is no longer a contribution.** "Agent evaluations can be invalid" is
   now a surveyed research area, not an insight.
2. **Legitimacy rises.** A named field with NIST attention is easier to justify building for.
3. **The survey's open-problems section is the correct place to check whether our remaining claim
   is already named — and we have not read it.** If §4's capability appears there as an open
   problem, our claim is "we are working on a known open problem," which is respectable and much
   weaker than differentiation. If it appears there as *solved*, this project ends.

> **`OQ-24` is therefore not a citation-hygiene task. Reading one survey could close this project
> in an afternoon, and it costs less than any other action available.**

---

## 3. Coverage of the actual workflow

The eleven steps of the primary user's workflow (`PRIMARY_USER_AND_WORKFLOW` §2.2), against the
combined landscape:

| # | Step | Covered by | Gap |
|---|---|---|---|
| 1 | Environment fingerprinting | Hermetic build tooling; NIST audit trails | — |
| 2 | Reproducibility / flakiness | Flaky-test infrastructure | — |
| 3 | Mutation / oracle testing | Mutation testing (mature) | — |
| 4 | Contamination scanning | NIST contamination work; overlap/canary methods | — |
| 5 | Grader integrity | NIST grader-gaming work; test-hygiene practice | minor |
| 6 | Fault injection | AgentChaos-class | — |
| 7 | Trajectory diagnosis | AgentRx-class | — |
| 8 | Audit trail | NIST probes | — |
| 9 | **Cross-referencing the above into a validity judgement** | **nothing** | **a human does it** |
| 10 | **Falsifying a candidate before attributing** | **nothing** | **a human does it, when they think to** |
| 11 | **Emitting an earned, machine-checkable "claim not supported" status** | **nothing** | **no artifact exists** |

**Eight of eleven steps are solved by existing work.** The three that are not consume **44 % of
the recurring human time** and produce the actual judgement.

---

## 4. What exactly remains underserved

### 4.1 The answer that must be rejected

**"Integration."** Rejected, per the brief and on its own merits: integration is what C1 achieves
with a shared trial corpus and a competent engineer, in about three weeks.

### 4.2 The answer that survives

> **An *earned*, machine-checkable status asserting that an evaluation's own claim is not
> supported by its own evidence — derived only after a falsification attempt, carrying the
> derivation as a required schema field, and emitted in a form a release gate can block on.**

Three properties, each load-bearing:

| Property | Why it is not covered |
|---|---|
| **Negative** | Every listed system produces *findings*. None produces a defensible **refusal**. A tool that says "I looked and I cannot tell, and here is the experiment that would settle it" is a different output type from anything in §2. |
| **Earned** | Any framework could add an `unsupported: true` field in an afternoon. The contribution is that the field is **only settable after a falsification attempt**, and that the derivation (`earned_by`) is a required schema field — so a consumer can check whether the refusal was earned rather than asserted. |
| **Gateable** | Machine-readable and consumed by a release gate. Not a report a human reads and files. |

### 4.3 The honest character of this claim

**`FACT`: this is not a technical capability gap. It is a cost point.**

C2 — a competent human running the protocol — produces exactly this output. What does not exist
is producing it at **marginal human cost ≈ 0**, repeatedly, inside CI.

**Is "a cost point" a legitimate thesis?** Yes, and there is good precedent: compilers, CI,
linters, type checkers and fuzzing infrastructure are all "a human could do this, at a cost." Each
became indispensable by making an existing capability nearly free at frequency.

**But it is legitimate *only at frequency*, and the frequency required here is high** — ~32
packages/year to cover maintenance, ~150 to repay the build inside two years
(`PRIMARY_USER_AND_WORKFLOW` §3). **The differentiation is economic, which makes the frequency
measurement (`OQ-25`) the whole project.**

### 4.4 Why this is not merely integration

| | Integration (C1) | The §4.2 claim |
|---|---|---|
| Output | A unified report | A **status field** with a **required derivation** |
| Consumer | A human | **A release gate** |
| On failure to determine | Prose hedging | **A typed refusal with a closed-enum reason and the experiment that would resolve it** |
| Checkable by a third party | No | **Yes — conformance suite** |

The distinction is **not** that we join five tools. It is that the output is a **typed, earned,
machine-consumable refusal** rather than a document. That is a different artifact with a different
consumer — which is why it does not reduce to integration, and also why it is small.

---

## 5. Threats this review does not remove

| # | Threat |
|---|---|
| T1 | **The survey may already name this as an open problem, or as solved** (`OQ-24`). Unread. |
| T2 | **An evaluation framework could add the status field.** The moat is the derivation discipline and the conformance suite, both of which are a specification — copyable by anyone who reads it, which is the point of publishing it. |
| T3 | **AgentRx-class systems are one adapter from auditing evaluations rather than agents.** Better-resourced, already built. |
| T4 | **NIST's direction is toward standardising exactly this kind of assurance artifact.** If a standard emerges, conforming is the product and the engine is a commodity implementation. **This is simultaneously the largest threat and the best argument for building the protocol and conformance suite first.** |
| T5 | The economic claim rests on two unmeasured numbers (`OQ-25`, `OQ-27`), either of which can make it negative. |

> **T4 deserves the last word. If a standards body defines the assurance artifact, the durable
> asset is the specification and its conformance suite — not the engine. That is the same
> conclusion `GATE_1_0_PRODUCT_THESIS.md` §4 reaches from the economics, arrived at independently
> from the competitive side.**
