# TERMINATION OF THE AGENT EVALUATION ASSURANCE THESIS

Status: **KILLED.** Not pivoted, not paused, not deferred.
Version 1.0.0 · 2026-09-22 · Terminal document for the product branch.

---

## 1. Decision

The **Agent Evaluation Assurance** product thesis (T1.0, `GATE_1_0_PRODUCT_THESIS.md` §1.2) is
**terminated**. No further investment in:

- E2 defect commissioning
- third-party adapter construction
- the C1/C2 baseline experiment
- A-INT implementation
- the conformance suite
- the automation economic model

The repository and all gate artifacts are **preserved as research lineage**
(§5), not deleted.

---

## 2. Trigger

`OQ-24` returned a concrete result: **MQ-EVA (Maple Quanta Evaluation Assurance)**, an existing
public 2026 system, substantially overlaps T1.0 across thirteen dimensions including the two we
had identified as our only surviving differentiation.

### 2.1 Overlap, as reported

| Our claimed capability | MQ-EVA |
|---|---|
| Object of evaluation = claim + evidence | **Covered** |
| Evaluation-validity assessment | **Covered** |
| Construct validity | **Covered** |
| Evidence integrity | **Covered** |
| Statistical reliability | **Covered** |
| Contamination / leakage | **Covered** |
| Production / environment validity | **Covered** |
| Failure characterisation | **Covered** |
| Evaluation independence | **Covered** |
| **Machine-readable evidence/claim artifacts** | **Covered** — our `claim_supported` field |
| **Unsupported vs Contradicted semantics** | **Covered, and finer than ours** |
| Reproducible evidence | **Covered** |
| No composite score | **Covered** |
| **Deterministic derivation of assurance results** | **Covered** — our `earned_by` derivation requirement |
| Implementation + public methodology | **Exists. Ours does not.** |

### 2.2 The two rows that ended it

`COMPETITIVE_2026.md` §4.2 had narrowed our entire remaining claim to a capability with three
load-bearing properties: **negative** (a defensible refusal), **earned** (derivation as a required
field), **gateable** (machine-consumable).

- **"Unsupported vs Contradicted semantics"** is the negative property — and it is a **finer
  distinction than we made.** Our vocabulary had `SUPPORTED / NOT_SUPPORTED / UNDETERMINED`.
  Separating *"the evidence does not support this"* from *"the evidence contradicts this"* is a
  distinction we never drew. **They are ahead of us on the axis we called our differentiation.**
- **"Deterministic derivation of assurance results"** is the earned property. It was the answer to
  *"anyone can emit `unsupported: true`"*. It is emitted by something that already exists.

**Both surviving properties are covered, one of them better than ours, by a system with a public
methodology and a running implementation.**

### 2.3 Epistemic status of the trigger — stated plainly

`ASSERTED_UNVERIFIED`. **I cannot independently verify MQ-EVA's existence or capabilities.** The
assistant's knowledge ends before this system's reported publication, and the characterisation
above is relayed from the `OQ-24` result, not checked against primary sources by me.

**This does not weaken the decision, because the decision is overdetermined (§3).** But it does
mean one thing must be recorded: *if this termination is ever revisited, the first step is to
verify MQ-EVA directly, not to re-read this document.*

---

## 3. The kill is overdetermined — it does not rest on MQ-EVA

**This matters.** A termination resting on a single unverified fact is fragile. This one is not.
Before `OQ-24` returned anything, the project already stood at:

| # | Condition already true at Gate 1.0 | Source |
|---|---|---|
| 1 | **8 of 11 workflow steps already solved** by existing work | `COMPETITIVE_2026` §3 |
| 2 | Differentiation already reduced from a capability to **a cost point** | `COMPETITIVE_2026` §4.3 |
| 3 | Economics contingent on **two unmeasured numbers** (`OQ-25`, `OQ-27`), either of which could make them negative | `AUTOMATION_UTILITY` §5 |
| 4 | **Column F could invert the economics outright**: the variable supplying frequency (many teams) is the variable supplying setup cost (many frameworks) | `AUTOMATION_UTILITY` §4 |
| 5 | The primary statistical test **could not reach significance** on 6 E2 defects | `BASELINE_EXPERIMENT` §6 |
| 6 | Correctness advantage over C2 **predicted to be zero by us** | `BASELINE_EXPERIMENT` §3 |
| 7 | **No workflow owner identified** across three gates (K-7) | `CONTINUE_OR_KILL_V2` |
| 8 | Pre-registered probability of a clean CONTINUE: **~15 %** | `KILL_CONTINUE_0_95` §6 |

> **MQ-EVA closed a door that was already nearly shut.** Conditions 3, 4 and 7 were each
> independently capable of ending the project, and none of them had been resolved favourably. The
> honest reading is that the thesis was failing on economics and ownership, and the competitive
> result removed the last reason to keep spending to find out.

---

## 4. What specifically is dead

| Killed | Why |
|---|---|
| T1.0 as a product thesis | §2, §3 |
| The engine (A-INT) | Its only differentiation is covered |
| **The conformance suite** | Gate 1.0 §4 argued it survives a KILL. **That reasoning is now void:** a conformance suite is only valuable for a specification people adopt, and MQ-EVA's public methodology occupies that position. Writing a competing standard against an implemented one with no users of our own is not a research contribution, it is a hobby. |
| The C1/C2 experiment | It measures an advantage we no longer claim |
| The E2 commission | The most expensive item in the plan, for a comparison that no longer matters |
| The automation economic model | Correct as analysis; attached to a dead thesis |

### 4.1 The reversal on the conformance suite

Gate 1.0 concluded *"build the protocol and conformance suite first in both branches — it is the
only artifact that survives a KILL."* **That conclusion is retracted.**

It rested on the protocol having standalone value as a specification. A specification competes on
adoption, and adoption goes to the one with an implementation and a public methodology. We would
be publishing a standard for a practice that already has one, with no users, no implementation and
no institutional position. **The argument was right in general and wrong here, and it was wrong
because it was written before `OQ-24` returned.**

---

## 5. What is preserved, and why

The repository stands. Nothing is deleted or rewritten to look better in hindsight.

| Preserved | Value |
|---|---|
| All 42 gate artifacts, Gates 0 → 1.0 | A complete, dated record of a thesis narrowing under adversarial review until it disappeared |
| **The four self-inflicted validity defects**, each with the arithmetic that caught it | §6 — the most transferable output of the project |
| The covering-array vs. random-sampling arithmetic (1.7×, not 10⁵×) | A reusable debunking of a common assumption in combinatorial testing advocacy |
| The C0/C1/C2 baseline-tiering pattern | Reusable by anyone asking "does my tool beat a competent composition?" |
| The frequency-derived automation threshold model | Reusable: ~32 packages/yr to cover maintenance, ~215 cumulative to repay build |
| Every KILL/PIVOT instrument and its pre-registered predictions | Evidence that the predictions were recorded *before* the outcomes |

---

## 6. The most useful thing this project produced

**Four validity defects, committed by a project whose stated purpose was detecting validity
defects, each surviving at least one adversarial review pass, each caught by re-deriving a number
rather than by any process:**

| # | Gate | Defect | Caught by |
|---|---|---|---|
| 1 | 0.75 | **Non-identifiable requirement.** B-20 demanded `UNKNOWN_MODEL` for an unobserved mechanism with a correlated decoy — observationally impossible — and made it the hard kill gate, scoring correct behaviour as catastrophic. | Re-deriving the identifiability condition |
| 2 | 0.9 | **Biased estimator.** FDR computed as pooled `ΣV/ΣR`, reporting that BH fails under dependence. It does not. | Re-deriving the estimator |
| 3 | 0.95 | **Underpowered primary.** The headline test ran on 6 E2 defects and could not reach significance under any realistic outcome. | Computing exact McNemar before the run |
| 4 | 1.0 | **Required-n cited as available-n.** `n = 112` was the sample size *needed* for MDE 0.15; the experiment had ~29. A threshold was set against precision that did not exist. | Recomputing the denominator |

> **A project explicitly built to detect invalid evaluations produced four invalid evaluations of
> its own, in its own decision instruments, under adversarial review, in five gates.** That is not
> an embarrassment to be minimised. It is the strongest empirical evidence this project generated
> for the reality of the defect class — and it is evidence that the defects were caught by
> *arithmetic performed by a person who chose to check*, not by tooling.
>
> **Which is also, uncomfortably, an argument against the product that was being proposed.**

---

## 7. Status

```
thesis:                  Agent Evaluation Assurance (T1.0)
status:                  KILLED
date:                    2026-09-22
trigger:                 OQ-24 / MQ-EVA (ASSERTED_UNVERIFIED)
overdetermined:          yes -- 8 independent conditions, 3 individually sufficient
repository:              PRESERVED as research lineage
implementation:          NEVER AUTHORIZED at any gate
lines of application code written across the entire project:  0
```

The next gate asks whether anything underneath is worth restarting around. **It is permitted to
answer no.**
