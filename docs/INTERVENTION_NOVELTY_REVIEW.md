# INTERVENTION NOVELTY REVIEW

Status: **GATE 1.1.** Version 0.1.0.

**The idea under review — our strongest surviving technical claim:**

> Controlled intervention before causal attribution, followed by explicit refusal when
> identification is impossible.

**Verdict, stated first: NOT NOVEL.** Established in six of the eight domains surveyed, and in two
of them it is the *defining* method. The composite — intervene, then refuse — is established in at
least two domains simultaneously.

---

## 1. Decomposing the claim

Three components, reviewed separately because they have different histories:

| # | Component |
|---|---|
| **N1** | Intervene (set a variable) rather than observe it, before attributing |
| **N2** | Recognise when the discriminating intervention is unavailable, and **refuse to attribute** |
| **N3** | Emit the refusal as a typed, machine-consumable status with its derivation |

---

## 2. Domain-by-domain review

### 2.1 Causal inference — **N1 and N2 are the foundations of the field**

`FACT` (not a citation, an entailment): the distinction between `P(Y|X)` and `P(Y|do(X))` is the
central object of modern causal inference. The **identifiability** question — *can this causal
quantity be computed from the available data and assumptions at all?* — is a formal problem with
established necessary-and-sufficient conditions, and *"not identifiable"* is a standard, expected
result.

| Our component | Status |
|---|---|
| N1 — intervene rather than observe | **The founding distinction of the field.** |
| N2 — refuse when identification impossible | **A named formal result.** Non-identifiability is an *output* of the theory, not an innovation on it. |

**Our B-20 error was rediscovering non-identifiability the hard way**, by writing a benchmark that
demanded it be violated. That is not a contribution; it is a demonstration that we had not applied
established theory.

### 2.2 AI assurance — **N2 and N3 are ~30 years old**

Assurance cases and safety cases (GSN, CAE) in safety-critical engineering structure an argument
as **claim → argument → evidence**, with explicit **defeaters**: recorded reasons a claim may fail
to be supported, including *insufficiency of evidence*.

| Our component | Status |
|---|---|
| N2 — explicit refusal | **A defeater.** Standard since the 1990s. |
| N3 — typed, structured, machine-consumable | Structured assurance-case notations are tool-supported and exchangeable |

**MQ-EVA's "Unsupported vs Contradicted" is an assurance-case distinction applied to AI
evaluation** — and it is finer than ours, because assurance-case practice already distinguishes
*unsupported* from *rebutted*. We reinvented the coarser half.

### 2.3 Scientific experiment automation — **the full loop is decades old**

Automated hypothesis → experiment design → execution → falsification → hypothesis revision was
demonstrated in autonomous "robot scientist" systems in the early 2000s, including active
selection of the *most discriminating* next experiment.

| Our component | Status |
|---|---|
| N1 | **The loop's core.** |
| N2 | Falsification is the loop's purpose; an inconclusive experiment is a normal outcome |

**This is the closest match to our whole protocol, and it predates the project by ~20 years.**

### 2.4 Fault localisation — **N1 established**

Statistical and mutation-based fault localisation, program slicing, and **delta debugging** all
operate by *changing* the program or input and observing the effect. ddmin — already in our own
design — is interventional by construction.

| Our component | Status |
|---|---|
| N1 | **Established** |
| N2 | **Partial.** Inconclusive localisation is recognised; typed refusal is less formalised. |

### 2.5 Causal debugging / causal inference for software — **direct match**

Work applying causal reasoning to software and systems explicitly addresses confounding in
observational telemetry and the need for intervention. Statistical debugging with controlled
perturbation is an established line.

| Our component | Status |
|---|---|
| N1 | **Established, and named** |
| N2 | Established in the causal framing it inherits |

### 2.6 Experiment infrastructure — **N1 at industrial scale**

A/B testing and experimentation platforms exist precisely to assign treatment rather than observe
it, with randomisation, power analysis and pre-registration built in. Guardrails against
attributing from observational comparisons are standard practice.

| Our component | Status |
|---|---|
| N1 | **Industrialised.** |
| N2 | **Partial** — underpowered results are reported as inconclusive; "cannot be experimented on" is usually an operational note rather than a typed output. |

### 2.7 Observability — **N1 partial, N2 weak**

Causal RCA products for traditional telemetry (Watchdog-class, Davis-class, Causely-class) infer
from observational data. **Most do not intervene**, and most produce a ranked hypothesis list
rather than a refusal.

| Our component | Status |
|---|---|
| N1 | **Partial** — mostly observational |
| N2 | **Weak** — ranking, not refusing. **The nearest thing to a gap in this review.** |

Chaos engineering supplies interventions but does not connect them to an attribution discipline.

### 2.8 Agent evaluation — **now covered**

| Our component | Status |
|---|---|
| N1 | AgentChaos-class supplies intervention; MQ-EVA's methodology covers validity assessment |
| N2 | **MQ-EVA: Unsupported vs Contradicted** |
| N3 | **MQ-EVA: machine-readable artifacts + deterministic derivation** |

---

## 3. Summary

| Domain | N1 intervene | N2 refuse | N3 typed refusal |
|---|---|---|---|
| Causal inference | **Founding** | **Formal result** | — |
| AI assurance (GSN/CAE) | — | **Defeaters, ~30 yr** | **Structured, tool-supported** |
| Scientific experiment automation | **Core loop** | **Core loop** | — |
| Fault localisation | **Established** | Partial | — |
| Causal debugging for software | **Established** | Established | — |
| Experiment infrastructure | **Industrialised** | Partial | — |
| Observability | Partial | **Weak** | Weak |
| Agent evaluation | Covered | **Covered (MQ-EVA)** | **Covered (MQ-EVA)** |

**N1: established in 6 of 8. N2: established in 4, partial in 2, weak in 1, covered in the 8th.
N3: covered in the two domains where it matters.**

### 3.1 The composite is not novel either

One might argue the *combination* is new. It is not: **causal inference supplies N1+N2 formally;
assurance cases supply N2+N3 in practice; automated experimentation supplies N1+N2 as a running
loop.** Each domain already holds a majority of the composite, and MQ-EVA holds all three in our
own domain.

---

## 4. The one residue, and its honest size

**Observability (§2.7) is the only surveyed domain where N2 is genuinely weak.** RCA products rank
hypotheses; they rarely say *"these two hypotheses are indistinguishable from what you record, and
here is the instrumentation or intervention that would separate them."*

`COMPETITIVE_GAP_MATRIX.md` §2 records the related gap (#40, intervention reachability) and
classifies it `D`. Three reasons it is not a thesis:

1. **The theory exists.** Structural identifiability analysis and optimal experimental design are
   mature in control theory and systems biology. Applying them to observability is *application*,
   which the §0 rule classifies as `COMPOSITION`.
2. **It is relative to instrumentation we do not control**, so it inherits every limit already
   documented.
3. **No owner.** A property, not a workflow. The same K-7 that went unanswered for three gates.

---

## 5. Conclusion

> **No novelty claim survives.** "Controlled intervention before attribution, then explicit refusal
> when identification is impossible" is the standard method of causal inference, the standard
> structure of assurance cases, and the operating loop of automated experimentation — and is now
> implemented in our own domain by MQ-EVA.
>
> **What this project actually did was apply established methods correctly.** That has value: the
> four defects in `CURRENT_THESIS_TERMINATION.md` §6 show how easily even a project *built around*
> these methods violates them. **But rigor in applying known methods is not a thesis, and it is
> not a product.**
