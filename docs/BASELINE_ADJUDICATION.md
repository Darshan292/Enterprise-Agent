# BASELINE ADJUDICATION

Status: **GATE 1.0.** Version 0.1.0.
Replaces the Gate 0.95 advocate **veto** with a challenge process carrying a standard of review.

---

## 1. What was wrong with the veto

Gate 0.95 C-5 let the BASELINE-C advocate invalidate the headline result by stating in writing
that C1 was not implemented to the strength floors.

**A single person's subjective statement should not be able to void a statistical experiment.**
Two failure directions, both real:

| Direction | Consequence |
|---|---|
| Advocate is **too generous** | The veto never fires, CH-1 is uncontrolled, and the experiment can be won with a weak baseline |
| Advocate is **too demanding** | Any result unfavourable to the baseline is vetoed, and no result is ever reportable |

A veto has no standard of review, so neither direction is correctable.

---

## 2. The replacement: ADVOCATE CHALLENGE

### 2.1 Roles

| Role | Requirement |
|---|---|
| **Advocate** | Implements and operates BASELINE-C. Stated interest that it wins. **Not on the A-INT team.** |
| **Adjudicator** | Rules on challenges. **Neither on the A-INT team nor the advocate.** Needs enough engineering judgement to read a strength floor and a diff. |
| **A-INT team** | May file a written response to a challenge. **No vote.** |

### 2.2 Filing a challenge

After results are scored and before the report is finalised, the advocate may file one or more
challenges. Each **must** contain:

```
advocate_challenge:
  challenge_id
  component:            which BASELINE-C component (BASELINE_C_SPEC section 2.x)
  strength_floor_cited: the EXACT preregistered floor allegedly violated
  evidence:             what was implemented vs what the floor requires
  remedy_proposed:      the specific change
  affected_packages:    which scored packages the remedy would change
```

> **A challenge that cites no preregistered strength floor is automatically NOT UPHELD.**
> "I could have done better" is not a challenge. "Section 2.7 requires pairwise interaction terms
> and the implementation fitted main effects only" is.

### 2.3 Standard of review

The adjudicator answers one question, and only this question:

> **Does the challenge identify a concrete deviation from a strength floor that was preregistered
> in `BASELINE_C_SPEC.md` before BASELINE-C was built?**

| Ruling | Test |
|---|---|
| **UPHELD** | A preregistered floor is cited **and** the implementation demonstrably fell short of it |
| **NOT UPHELD — no floor** | No preregistered floor cited |
| **NOT UPHELD — floor met** | The floor is cited but the implementation met it |
| **OUT OF SCOPE** | The challenge proposes a *new* capability not in the preregistered spec |

The adjudicator does **not** rule on whether the baseline "could have been better in principle."
That question has no end and no standard.

### 2.4 Consequences — repair, never void

| Ruling | Consequence |
|---|---|
| **UPHELD** | **The component is fixed and the affected packages are RE-RUN for both arms.** The experiment is repaired, not discarded. Both the original and re-run results are reported. |
| **NOT UPHELD** | Result stands. The challenge and the ruling are **published verbatim in the report as a recorded dissent.** |
| **OUT OF SCOPE** | Result stands; the proposed capability is recorded as a limitation for the next iteration. |

> **The critical change: an upheld challenge triggers a re-run, not an invalidation.** The old
> veto destroyed information; this preserves it and fixes the defect. Re-running one component
> against 24 packages is affordable; discarding the whole experiment is not.

### 2.5 Bounds on the process

| Bound | Value |
|---|---|
| Challenge window | 10 working days after scoring |
| Maximum challenges | 3 per experiment. More suggests the spec's floors were inadequate, which is itself the finding. |
| Re-run budget | Pre-allocated: 30 % of the original compute is reserved for upheld challenges |
| Adjudication | Written, published, with reasons |
| **Escalation** | If the adjudicator declares themselves unable to rule, the challenge is recorded as **UNRESOLVED** and the affected result is reported **with the dispute attached** — not silently as a pass |

---

## 3. Anti-gaming, both directions

| Risk | Control |
|---|---|
| A-INT team pressures the advocate to under-challenge | The advocate's stated interest is that BASELINE-C wins; challenges are published whether upheld or not |
| Advocate challenges everything to void the result | Maximum 3; a challenge citing no floor is automatically not upheld; upheld challenges cause **re-runs**, not voids, so there is no incentive to fish |
| Adjudicator is captured | Not on either team; rulings are written and published with reasons |
| Strength floors were too weak to begin with | **An experiment ending with zero upheld challenges *and* an advocate who says the floors were too weak is itself a finding**, reported as a limitation on the result's strength |
| Post-hoc floor invention | Only floors preregistered before BASELINE-C was built are citable |

---

## 4. Relationship to the decision tree

The old C-5 **vetoed** C-1. The new process does not.

| Adjudication outcome | Effect on the decision |
|---|---|
| No challenges, or all NOT UPHELD | Decision proceeds on the scored results |
| Any UPHELD | **Re-run first**, then decide on the re-run results |
| UNRESOLVED | Decision proceeds, **with the dispute reported alongside the result**; a CONTINUE carrying an unresolved baseline dispute is downgraded to **PIVOT** |

`DESIGN_DECISION`: an unresolved dispute downgrades CONTINUE to PIVOT rather than being ignored or
treated as a KILL. Proceeding over an unresolved dispute would restore the original problem —
someone's unexamined opinion determining the outcome — in the opposite direction.
