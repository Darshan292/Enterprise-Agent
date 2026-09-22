# E2 AUTHORSHIP PLAN

Status: **GATE 1.0 — PLAN ONLY. NO DEFECTS COMMISSIONED.** Version 0.1.0.

Answers the questions the brief requires **before** any external-author time is spent.

---

## 1. The questions, answered

| Question | Answer |
|---|---|
| **Number of independent external authors** | **4.** Not 1. |
| **Defect families per author** | **5 each = 20 E2 defects**, each author covering a different subset of dimension pairs |
| **Do all authors use the same synthetic substrate?** | **No — 2 of 4 are given a different substrate.** See §3. |
| **Does the same ontology/factor model constrain all defects?** | **No — 2 of 4 are not shown the invariant list.** See §4. |
| **How is common-method bias reported?** | Per-author detection rates, inter-author structural similarity, and a per-substrate breakdown. See §5. |

---

## 2. Why 4 × 5 rather than 1 × 18

`FACT`: 18 defects from one author are not 18 independent samples of "defects an outsider would
write." They are one author's **style**, replicated. Detection rate against them estimates how
well A-INT handles *that person's* imagination.

| Design | Independent author-samples | What the detection rate estimates |
|---|---|---|
| 1 × 18 | **1** | One author's style |
| 2 × 9 | 2 | Two styles |
| **4 × 5** | **4** | Four styles — enough to *see* an author effect, not enough to estimate it precisely |
| 8 × 3 | 8 | Better, and likely unaffordable |

`DESIGN_DECISION`: **4 authors.** Below 4 an author effect is invisible; above 4 the cost rises
faster than the information. **4 is a compromise and is recorded as one** — it gives 3 degrees of
freedom on the author effect, which detects a large effect and nothing subtle.

---

## 3. Substrate independence

| Group | Authors | Substrate |
|---|---|---|
| **S-A** | 2 authors, 10 defects | Our synthetic enterprise environment |
| **S-B** | 2 authors, 10 defects | **A different substrate** — a public agent-evaluation harness they choose, adapted to the package schema in `PRODUCT_INTERFACE.md` §1.1 |

**Why this matters more than the author count.** Every defect authored against our substrate
inherits our environment's affordances: what can break is bounded by what we built. An S-B defect
can exercise failure modes our environment cannot express.

**This also tests `PRODUCT_INTERFACE` for real.** If an outside author cannot express their defect
in the package schema, **the schema is wrong** — and that is a finding worth more than the defect.

`ASSUMPTION`: S-B authors can adapt a public harness to the schema in reasonable time. If they
cannot, the adapter cost in `AUTOMATION_UTILITY.md` §4 is worse than estimated, which is itself
the answer to `OQ-27`.

---

## 4. Ontology independence

| Group | Given | Tests |
|---|---|---|
| **O-1** (2 authors) | Invariant list + ambient observable set + package schema | Defects inside our ontology. **Upper bound** on detection. |
| **O-2** (2 authors) | **Package schema only**, plus the instruction *"make this evaluation invalid in a way that is hard to see"* | Defects outside our ontology. **The real test of the factor model's adequacy.** |

Crossed with substrate:

| | S-A (our substrate) | S-B (other substrate) |
|---|---|---|
| **O-1** (given ontology) | Author 1 — 5 defects | Author 3 — 5 defects |
| **O-2** (blind to ontology) | Author 2 — 5 defects | Author 4 — 5 defects |

> **Author 4 — blind to our ontology, on a different substrate — produces the only defects in the
> entire project that are not shaped by our assumptions twice over.** Five defects. If A-INT's
> detection rate collapses on that cell, the factor model is the binding constraint and no
> benchmark result means what it appears to.

**Cell-level detection rates are reported separately. They are the most informative five numbers
in the experiment and the least statistically powerful.** Both facts are stated together.

---

## 5. Reporting common-method bias

Three measures, all reported whatever they show.

### 5.1 Per-author detection rate
Detection rate on each author's 5 defects, with Wilson intervals. At n = 5 the interval half-width
near p = 0.5 is **± 0.33** — enough to reveal a total collapse, nothing finer. **An author effect
will be visible only if it is large.**

### 5.2 Inter-author structural similarity
For every defect, record a structural signature: `(dimension_pair, observability, intervention
availability, mechanism_family)`. Then:

```
similarity = (pairs of defects from DIFFERENT authors sharing a signature)
           / (pairs of defects from the SAME author sharing a signature)
```

| Value | Meaning |
|---|---|
| ≈ 1 | Authors converge; the defect space is genuinely constrained, and more authors would not have helped |
| ≪ 1 | Strong author-specific style; **4 authors is too few** and the detection rate is an average over incomparable populations |

### 5.3 Substrate and ontology breakdowns
Detection rate by S-A/S-B and by O-1/O-2, reported as a 2 × 2 with per-cell n = 5.

> **A detection rate high on O-1 and low on O-2 means A-INT detects defects our own ontology
> predicts and misses the rest.** That is the circularity failure this project has been circling
> since Gate 0.75, and this 2 × 2 is the first design that can actually see it.

---

## 6. Instructions to authors

Identical for all four except the ontology grouping.

```
You will receive an evaluation package schema and access to a substrate.
Construct 5 evaluation packages that each produce a PLAUSIBLE-LOOKING score
while being INVALID -- the score does not measure what the package claims.

Each defect must:
  - be real: an engineer inspecting the package carefully would agree it is invalid
  - be non-obvious: not detectable by reading the config alone
  - be documented in a sealed registry: mechanism, why the score is wrong,
    what would have to be observed or intervened on to establish it
  - declare whether a controlled intervention on the responsible variable is
    available, non-surgical, or impossible

Do NOT coordinate with the other authors.
Do NOT read this repository's documents beyond the package schema (O-2 authors).
```

---

## 7. What this plan still cannot fix

| # | |
|---|---|
| L1 | **20 E2 defects does not reach the 96 needed for statistical non-inferiority** (`AUTOMATION_UTILITY` §3.1). It reaches the 24 the primary McNemar needs at a realistic win rate — barely, and only combined with the 4 existing E2 defects. |
| L2 | Four authors gives **3 degrees of freedom** on the author effect. Large effects are visible; moderate ones are not. |
| L3 | S-B authors still work from **our package schema**. One ring of circularity is removed, not two. |
| L4 | **No E3 defects exist** (`OQ-18`). Every defect is deliberately planted by someone told to plant it. Real evaluations fail by accident, and accidental failures may look nothing like these. |
| L5 | The **cost** of this plan — 4 external authors, 2 substrate adaptations — is materially higher than 1 × 18. **It should not be commissioned before column F is measured** (`AUTOMATION_UTILITY` §4): if per-framework adapter cost kills the economics, none of these defects will ever be scored. |

**L5 is the operative constraint: measure the adapter cost first.**
