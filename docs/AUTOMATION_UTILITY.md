# AUTOMATION UTILITY

Status: **GATE 1.0.** Version 0.1.0.
Corrects the Gate 0.95 reading of C2 and defines the pre-registerable automation criterion.

---

## 1. The correction

Gate 0.95 concluded: *"if C2 matches A-INT, the protocol is the contribution and the engine is
not."* **That is wrong as stated.** A capability a human can perform is not thereby worthless to
automate — otherwise compilers, CI, linters and type checkers would not exist.

The corrected question separates three claims that Gate 0.95 collapsed:

| Claim | Question | Answered by |
|---|---|---|
| **Correctness advantage** | Does A-INT find defects C2 misses? | Column A |
| **Automation advantage** | Does A-INT preserve C2's correctness at materially lower human cost? | Columns B, C, D |
| **Economic advantage** | Does the saving repay build and maintenance at the user's volume? | Columns E, F |

**These are never combined into a composite.** An engine can have an automation advantage and no
correctness advantage (legitimate), or a correctness advantage and no economic one (a research
result, not a product). Collapsing them hides which.

---

## 2. The six-column matrix

Values marked `PRED` are pre-registered predictions; `MEASURE` is filled from the experiment.
`ASSUMPTION` on all timings pending `OQ-27`.

| | **A. Technical correctness** (CDDR, E2 defects) | **B. Human effort** (min/package, recurring) | **C. Time to first correct finding** | **D. Throughput** (packages/engineer-day) | **E. Recurring operational cost** (after build exists) | **F. Setup / adaptation cost** (per new framework) |
|---|---|---|---|---|---|---|
| **C0** | `PRED` low — no cross-referencing | ~120 (reads 6 reports, no synthesis) | fast but often wrong | ~4 | tool runtime + 2 h human | ~2 days |
| **C1** | `PRED` moderate–high | **~215** (`PRIMARY_USER_AND_WORKFLOW` §2.1) | ~2–3 h | **~2.2** | tool runtime + 3.6 h human | ~3 days |
| **C2** | `PRED` **high** | **~215 + intervention time already counted** | ~3–4 h (interventions are serial) | ~2.0 | tool runtime + 3.6 h human | ~3 days |
| **A-INT** | `PRED` ≈ C2 | **~20** (review artifact, approve or dispute) | ~20 min wall-clock + compute | **~14** | compute + 0.33 h human + amortised maintenance | **`MEASURE` — the risk, see §4** |

### 2.1 What the matrix says before any run

- **Column A**: we predict a **tie** with C2. `BASELINE_EXPERIMENT` §3 already predicted our edge
  lives in decoy rejection and refusal — exactly what C2 does by hand.
- **Columns B–D**: a **~10× effort reduction** and **~7× throughput** gain. This is the real claim.
- **Column E**: 0.33 h vs 3.6 h per package, against 105 h/yr maintenance.
- **Column F**: **unmeasured, and the largest threat.** See §4.

> **The honest summary of this matrix: we predict no correctness advantage, a large automation
> advantage, and an economic advantage that is entirely contingent on two unmeasured numbers —
> package volume (`OQ-25`) and per-framework adaptation cost (`OQ-27`, column F).**

---

## 3. The Automation Utility Criterion (AUC-1)

Pre-registerable. Thresholds derived from the workflow model, not chosen for impressiveness.

> **A-INT satisfies AUC-1 if EITHER:**
>
> **(A) Correctness route.** A-INT detects strictly more E2 defects than C2, with exact McNemar
> p < 0.05 on paired defect outcomes.
>
> **OR**
>
> **(B) Automation route — all four required:**
> **(B1) Non-inferiority on correctness:** **zero unfavourable discordance** — A-INT misses no
> defect that C2 detects, across the entire defect registry.
> **(B2) Effort:** recurring human minutes per package ≤ **0.25 × C2's**, measured by timed
> observation of both, ≥ 10 packages each.
> **(B3) Time to first correct finding:** ≤ **0.5 × C2's** median.
> **(B4) Economics:** at the user's **measured** package volume, net annual saving exceeds
> maintenance **and** cumulative payback ≤ **3 years**.

### 3.1 Why "zero unfavourable discordance" instead of statistical non-inferiority

`FACT` (arithmetic): a non-inferiority margin derived the standard way — preserve 50 % of C2's
advantage over C0, i.e. δ = 0.175 — requires **96 defects per arm**. A margin of 0.30 requires 33.
**We have 24, whose MDE is 0.35 — wider than any margin worth stating.**

| Margin δ | n per arm required |
|---|---|
| 0.175 | **96** |
| 0.20 | 74 |
| 0.30 | 33 |
| *available* | **24 (MDE 0.35)** |

So statistical non-inferiority is **not measurable at any defect count we will reach**. Three
options were available:

| Option | Assessment |
|---|---|
| Widen the margin to 0.35 | A margin permitting A-INT to be 35 points worse is not a correctness floor |
| Commission 96 E2 defects | Not affordable |
| **Zero unfavourable discordance** | **Adopted.** Strict, measurable at n = 24, and errs against us |

**Zero unfavourable discordance is a stricter test than statistical non-inferiority.** If A-INT
and C2 were truly equal with a 30 % discordance rate, P(zero unfavourable) ≈ **0.008** — we would
almost certainly fail. Choosing it means we are likely to fail B1, which is the point of a
correctness floor.

### 3.2 Why 0.25× for effort (B2)

Derived, not chosen. Recurring manual cost is 3.6 h. Automation must leave the human enough time
to *meaningfully review* the artifact — a tool whose output nobody reads is worse than no tool.
Reviewing a structured artifact, checking the coverage section and approving or disputing is
estimated at **20 minutes**, which is 0.093 × 3.6 h. **0.25× (54 minutes) is therefore a
deliberately lenient threshold** — it permits A-INT to be nearly three times slower than our own
estimate and still pass. A threshold set at 0.1× would be tuned to our prediction, which is
precisely the error this gate exists to avoid.

### 3.3 Why ≤ 3 years for payback (B4)

Below the plausible lifetime of an evaluation harness before a framework migration forces
re-adaptation. A payback horizon longer than the thing being automated is not a payback.

### 3.4 AUC-1 cannot be satisfied by assertion

| Sub-criterion | Measured how |
|---|---|
| B1 | The defect registry, scored once |
| B2 | **Timed observation of a human performing C2 and a human reviewing an A-INT artifact**, ≥ 10 packages each, same person where possible |
| B3 | Wall-clock from package receipt to first correct finding, both arms |
| B4 | The user's **measured** volume (`OQ-25`) and the **measured** build cost (`OQ-27`) |

**B2 and B3 require a human study, not a simulation.** No part of AUC-1 is satisfiable from the
benchmark alone.

---

## 4. Column F is the threat that could erase everything

**Per-framework adaptation cost is unmeasured and it interacts adversely with the user choice.**

The primary user was selected *because* a shared-infrastructure team has high package volume
(`PRIMARY_USER_AND_WORKFLOW` §3). But volume comes from serving **many product teams**, and many
teams means **many evaluation frameworks** — each needing its own `TrialSource`, `EffectLog` and
`EnvironmentProbe` adapter.

```
ASSUMPTION: adapter cost = 3 engineer-days = 24 h per framework
```

| Frameworks | Adapter cost | Packages/yr needed just to repay adapters (at 3.25 h saved) |
|---|---|---|
| 1 | 24 h | 7 |
| 5 | 120 h | 37 |
| 10 | 240 h | 74 |
| 20 | 480 h | **148** |

> **At 20 frameworks, adapter cost alone consumes the entire saving from 148 packages/year — the
> whole reason the user was chosen.** The variable that supplies frequency is the same variable
> that supplies setup cost.

`DESIGN_DECISION`: **column F must be measured before the E2 commission.** Building one adapter
against one real third-party evaluation framework, and timing it, is a few days of work that could
invalidate the entire economic case. It costs far less than the 18 external defects and it should
be done first.

---

## 5. Conclusions kept separate, as required

| Advantage | Predicted | Basis |
|---|---|---|
| **Correctness** | **None over C2.** Possibly over C1. | Column A; `BASELINE_EXPERIMENT` §3 |
| **Automation** | **Large — ~10× effort, ~7× throughput.** | Columns B, C, D |
| **Economic** | **Unresolved. Contingent on volume (`OQ-25`) and adapter cost (`OQ-27`/column F), either of which can make it negative.** | Columns E, F |

**No composite. No single utility score.** The three answers differ, and which one is true
determines whether the product is an engine, a protocol, or nothing.
