# SUBSTRATE VALIDITY

Status: **GATE 0.75.** Document version 0.1.0.

Addresses existential risk #2: **the deterministic substrate may primarily test our own harness
rather than real agent behaviour.**

---

## 1. The three layers

| | **L-DET** | **L-STO** | **L-REM** |
|---|---|---|---|
| Planner | Scripted stub | **Unscripted local model** | Unscripted, provider-hosted |
| Trajectory | **Authored by us** | Emergent | Emergent |
| Determinism | Byte-reproducible (INV-9) | Seeded, not guaranteed | **None** |
| Cost | ≈ 2 s, 0 remote calls | Local inference, 0 remote calls | Remote calls, budgeted |
| Affordable n | 10³–10⁵ | 10²–10³ | **10¹** |
| Wilson half-width at the affordable n | ± 0.03 @ n=1000 | ± 0.10 @ n=100 | **± 0.33 @ n=5** |
| Subject of the claim | **Our runtime** | Agent behaviour | Provider behaviour on a date |

> **The central asymmetry: precision and relevance run in opposite directions.** L-DET is precise
> and least relevant; L-REM is relevant and least precise. No amount of engineering changes this,
> and pooling the layers would produce a number that inherits the worst of both.

---

## 2. Claim → layer sufficiency matrix

`S` = sufficient · `N` = necessary but not sufficient · `—` = contributes nothing · `✗` = cannot
be established at all.

| # | Claim | L-DET | L-STO | L-REM | Sufficient set |
|---|---|---|---|---|---|
| C1 | "The adapter mishandles factor combination X" | **S** | — | — | L-DET |
| C2 | "Oracle O fires on constructed trial T" | **S** | — | — | L-DET |
| C3 | "Invariant I is violated under assignment A (in the substrate)" | **S** | — | — | L-DET |
| C4 | "Reduction of failure F is stable" | **S** | — | — | L-DET |
| C5 | "Factor set M is confirmed; decoy D rejected" | **S** | — | — | L-DET |
| C6 | "INV-9 holds" | **S** | — | — | L-DET |
| C7 | "The declared factor model does not explain failure F (in the substrate)" | **S** | — | — | L-DET |
| C8 | **"A real agent can reach the state that triggers F"** | — | **N** | — | **L-STO** |
| C9 | **"F is reachable without adversarial scripting"** | — | **N** | — | **L-STO** |
| C10 | "The failure rate under X is r, for agents" | — | **N** | — | L-STO, with its interval |
| C11 | "A history-dependent failure arises from planning, not from our scripted sequence" | — | **N** | — | L-STO |
| C12 | **"An L-DET discovery transfers"** | **N** | **N** | — | **L-DET + L-STO (the transfer test)** |
| C13 | "F occurs against a provider-hosted model" | — | — | **N** | L-REM |
| C14 | "Failure rate shifted across model versions" | — | — | **N** | L-REM, **as an association on a date only** |
| C15 | "The local model is representative of provider models" | — | **N** | **N** | L-STO + L-REM, **weakly** |
| C16 | "This occurs in production" | ✗ | ✗ | ✗ | **Not establishable** |
| C17 | "The factor model is complete" | ✗ | ✗ | ✗ | **Not establishable** (`RESEARCH_VALIDITY` H2) |
| C18 | "This agent is safe" | ✗ | ✗ | ✗ | **Not establishable** |

### 2.1 The rows that matter

**C1–C7 are all L-DET-sufficient, and all seven are claims about our own code.** That is risk #2
stated as a table: the substrate where we can afford 10⁵ trials produces only claims about the
thing we built.

**C8–C12 require L-STO and nothing else will do.** Every claim with the word *agent* in it lives
here, at 100× less precision.

**C12 — the transfer test — is the bridge, and it is the only reason L-DET findings are worth
anything beyond debugging our own adapter.**

---

## 3. The transfer test

For each L-DET discovery, attempt reproduction in L-STO at the same factor assignment with an
**unscripted** local agent, **n = 30** independent attempts.

| Transfer rate observed | One-sided 95 % lower bound | Interpretation |
|---|---|---|
| 30/30 | **0.92** | Transfers. The L-DET finding is about agents. |
| 24/30 (80 %) | **0.66** | Transfers usefully. |
| 15/30 (50 %) | 0.34 | Marginal. Report as substrate-specific-or-not-determined. |
| ≤ 6/30 (20 %) | 0.09 | **Does not transfer. The finding was about our harness.** |

`DESIGN_DECISION`: **n = 30.** It separates ≥ 0.92 from ≤ 0.66, which is the only distinction
that changes what we do, and 30 local runs is affordable.

### 3.1 The aggregate metric that decides risk #2

```
TRANSFER_YIELD = (L-DET discoveries reproducing in L-STO at ≥ 50 %) / (L-DET discoveries attempted)
```

This single number answers "is the deterministic substrate testing our own harness?" It is a
gate criterion in `CONTINUE_OR_KILL.md`, not a diagnostic.

### 3.2 What a transfer failure actually means

Three distinguishable causes, and they are **not** equally bad:

| Cause | Meaning | Action |
|---|---|---|
| The stub scripted a trajectory no planner would produce | **The finding is an artifact.** Risk #2 confirmed for that finding. | Discard; tighten the stub's trajectory distribution toward observed L-STO trajectories. |
| The real agent recovers where the stub did not | The runtime failure is real but the agent masks it | Keep as a runtime finding (C1), **not** as an agent finding (C8) |
| L-STO is underpowered at n = 30 for a low-rate failure | Neither confirmed nor refuted | Report `INDETERMINATE`; raise n if the failure matters |

Reporting all three as "did not transfer" would be the easy and wrong summary.

---

## 4. Non-pooling rules

`DESIGN_DECISION`, enforced structurally rather than by convention:

1. **No statistic is computed over a trial set spanning more than one layer.** A layer tag is
   part of the aggregation key; the aggregation function refuses a mixed key.
2. **No weighted average across layers.** There is no defensible weight, because the layers
   estimate different quantities.
3. **No single reliability score.** Not per layer either — each layer reports a coverage report
   plus a failure set plus verdicts, per `EVALUATION_STRATEGY` O.1.
4. **A report showing multiple layers shows multiple tables**, each with its own n, interval and
   claim scope.
5. The **only** permitted cross-layer quantity is `TRANSFER_YIELD`, which is explicitly a
   relationship *between* layers and is reported as such — never as a reliability measure.

### 4.1 Why the temptation will be constant

L-DET produces thousands of trials with tight intervals. L-STO produces hundreds with loose ones.
L-REM produces tens with intervals so wide they embarrass the report. **The pressure to blend
them into one number will be permanent and will come from people who mean well.** The structural
refusal (rule 1, in the aggregation key) is what holds; the prose here will not.

---

## 5. What each layer legitimately proves — restated as sentences

**L-DET.** *"Under our fixture, our capability profiles and our scripted planner, factor
combination X causes invariant I to be violated in k of n trials, and the minimized set survives a
confirmatory factorial."* — A claim about our runtime. Precise. Cheap. Narrow.

**L-STO.** *"An unscripted local agent, given the same conditions, reached the same violation in k
of n attempts."* — A claim about agent behaviour with one model. Imprecise. Affordable. This is
the only claim class that justifies calling the project *agent* reliability.

**L-REM.** *"On this date, against this provider and model alias, n runs produced this outcome
distribution."* — A smoke alarm. Detects catastrophic divergence and nothing finer.

---

## 6. Open questions

| ID | Question | Blocks |
|---|---|---|
| `OQ-16` | Is a local model available at all in this environment? **If not, L-STO does not exist, C8–C12 become unestablishable, and risk #2 is unmitigable.** That alone would force a PIVOT or KILL. | Everything |
| `OQ-23` | If the stub's trajectory distribution is tuned toward observed L-STO trajectories (§3.2), the transfer test becomes partly circular — we would be tuning the substrate to pass its own validity check. Needs a held-out set of L-STO trajectories never used for tuning. | Gate 3 |
| `OQ-20` | Budget parity across layers: equal trials favours L-DET; equal wall-clock favours L-DET more. There may be no fair comparison, only a declared one. | Benchmark run |
