# KILL / PIVOT / CONTINUE — GATE 0.95

Status: **PRE-REGISTERED DECISION INSTRUMENT. NOTHING RUN.** Version 0.1.0.

Thresholds set before BASELINE-C exists and before any package is scored. **The project is not
protected from being killed**, and two of the five kill conditions can fire without the benchmark
being run at all.

---

## 0. Settled before the experiment

| # | Finding | Source |
|---|---|---|
| **P1** | **The primary test as currently specified cannot reach significance.** 6 E2 defects; our own pre-registered prediction is C1 detects 5, A-INT detects 6 — one discordant pair, exact p = 1.0. | `BASELINE_EXPERIMENT` §3 |
| **P2** | **12 cross-dimension defects is arithmetically inadequate** — significance requires *every* discordant defect to favour A-INT. 24 is the minimum tolerating a realistic 80 % win rate. | `GATE_0_95_BASELINE_CHALLENGE` §8.2 |
| **P3** | **Replication does not buy population power.** 12 defects × 10 reps ≈ 13 effective observations at realistic ICC, not 120. | §8.1 |
| **P4** | **BASELINE-C C1 is ≈ 3 weeks for one competent engineer.** That is the number a multi-month build must beat. | `BASELINE_C_SPEC` §6 |
| **P5** | **Every edge we predict is the falsification protocol, not the integration** — decoy rejection, refusal on non-surgical cases, `INSUFFICIENT_OBSERVABILITY` instead of a guess. | `BASELINE_EXPERIMENT` §3 |

> **P5 is the finding that should change what gets built, whatever the benchmark says.**

---

## 1. CONTINUE — all five required

| # | Condition | Measurement |
|---|---|---|
| **C-1** | **Statistically defensible advantage over C1 on CDDR**, computed on **E2 defects**, exact McNemar p < 0.05 | `METRICS.md` PRIMARY |
| **C-2** | **FAR not materially worse than C1**: upper bound of (FAR_A-INT − FAR_C1) < 0.10 at n = 112 per arm — **reported jointly with Indeterminate Rate**, and IR on *defective* packages not more than 0.15 above C1's | S1 + S2 |
| **C-3** | **≥ 1 externally authored (E2) defect detected by A-INT and missed by C1** | S6 |
| **C-4** | **Self-audit passes**: overall detection ≥ 0.70; **all four deterministic classes at 1.00**; ≥ 1 of 2 config-injection classes; false alarms ≤ 1/10 | `SELF_AUDIT.md` §4 |
| **C-5** | **The advantage is not explained by a weak baseline.** The advocate owner of BASELINE-C, reviewing the scored results, **states in writing that C1 was implemented to the strength floors** and identifies no component they would now strengthen. A refusal, or a named weakness, voids C-1. | `BASELINE_C_SPEC` §2 |

**C-5 has a veto.** It is the only defence against CH-1 that does not depend on our own judgement.

---

## 2. KILL — any one is sufficient

| # | Condition |
|---|---|
| **K-1** | **A-INT is statistically indistinguishable from C1** on CDDR over E2 defects (p ≥ 0.05) |
| **K-2** | **The advantage disappears when BASELINE-C is implemented competently** — i.e. C-5 fails, or a post-hoc strengthening of any C1 component erases the gap |
| **K-3** | **FAR materially worse**: lower bound of (FAR_A-INT − FAR_C1) > 0.10; **or** IR on defective packages > 0.40, which is a refusal machine regardless of FAR |
| **K-4** | **The result depends primarily on team-authored defects** — the advantage is significant on E0/E1 and absent on E2 |
| **K-5** | **Self-audit exposes a validity failure the architecture cannot detect** — any deterministic class below 1.00, or both config-injection classes missed |
| **K-6** | **The E2 commission is declined and the primary is run underpowered anyway.** An experiment known in advance to be incapable of supporting its claim is not evidence. |
| **K-7** | **No workflow owner within the 60-day window** (carried from Gate 0.9 K-F) |

**K-6 and K-7 can fire without running anything.**

---

## 3. PIVOT — the ambiguous outcomes

Per the standing instruction: **ambiguity resolves to PIVOT, never to CONTINUE.**

| # | Condition | Pivot target |
|---|---|---|
| **P-1** | **A-INT > C1 but ≈ or < C2** | **The protocol is the contribution; the engine is not.** Ship `IDENTIFIABILITY_AND_FALSIFICATION.md` + `VERDICT_SEMANTICS.md` as a **specification**, plus a **conformance suite** any evaluation harness can run against itself. Stop building a platform. |
| **P-2** | C-1 passes, C-3 fails (no E2 defect uniquely detected) | Narrow to the defect classes where an edge exists, and commission E2 defects in exactly those classes before claiming anything |
| **P-3** | C-1 passes on CDDR but fails CDDR-strict (dimension attribution is weak) | Reposition from *attribution* to *flagging*: "this evaluation is suspect" rather than "here is why" |
| **P-4** | Everything passes except C-4's config-injection classes | Keep the engine, **drop every self-assurance claim**, require external review of the engine's own statistics |
| **P-5** | Primary is underpowered **and** option 2 was chosen knowingly | Treat the run as descriptive; re-gate after the E2 commission |

### 3.1 P-1 is the most likely outcome, and it is not a failure

`BASELINE_EXPERIMENT` §3 predicts our edge lives entirely in decoy rejection, refusal on
non-surgical cases, and `INSUFFICIENT_OBSERVABILITY` — all of which **C2 performs by hand**.

If a careful human running the protocol matches the engine, the finding is that **the protocol is
the product**. A specification plus a conformance suite is cheaper to build, far more likely to be
adopted, has no maintenance question, and sidesteps the ownership problem (K-7) because it can be
run by whoever already owns the evaluation.

> **That outcome would make this entire five-gate sequence worthwhile, and none of the earlier
> gates could have produced it.**

---

## 4. Decision rule

First matching row wins.

| # | Condition | Decision |
|---|---|---|
| 1 | K-6 or K-7 | **KILL** (no run required) |
| 2 | C-5 fails (baseline owner names a weakness) | **KILL** (K-2) |
| 3 | K-5 (deterministic self-audit class below 1.00) | **KILL** |
| 4 | K-3 (FAR materially worse, or IR > 0.40 on defective packages) | **KILL** |
| 5 | K-1 or K-4 | **KILL** |
| 6 | A-INT > C1 but ≈ or < C2 | **PIVOT (P-1)** — ship the protocol and conformance suite |
| 7 | C-1 passes, C-3 fails | **PIVOT (P-2)** |
| 8 | C-1 passes, CDDR-strict fails | **PIVOT (P-3)** |
| 9 | All of C-1..C-5 pass **and** A-INT > C2 | **CONTINUE**, MVDC-scoped only |

---

## 5. Anti-gaming

1. Metrics frozen before BASELINE-C is built; registry and thresholds digest-recorded before any run.
2. **A-INT build frozen and digest-recorded before the first package.** Any modification voids the run.
3. **Scoring once.** No re-scoring.
4. MARGINAL is FAIL for C-1 and C-3.
5. BASELINE-C is owned by an **advocate** who is not on the A-INT team, and holds the C-5 veto.
6. No package added after results are seen; removal only for demonstrable malformation, reported.
7. Every pre-registered metric is reported, **including those that went against us**.
8. A KILL leaves every artifact in the repository with the result attached.

---

## 6. Pre-registered prediction

`HYPOTHESIS`, recorded so the outcome can contradict it.

| Criterion | Predicted | Confidence |
|---|---|---|
| C-1 (advantage over C1 on E2) | **FAIL as currently powered** — passes only if the E2 commission happens | high |
| C-2 (FAR not worse) | PASS | medium-high |
| C-3 (≥ 1 E2 defect uniquely detected) | **UNCERTAIN** — our own prediction table gives a one-defect margin | low |
| C-4 (self-audit) | PASS on deterministic classes; **UNCERTAIN on config-injection** | medium |
| C-5 (baseline not weak) | **the real risk** — we expect C1 to be strong | — |
| **A-INT vs C2** | **≈ or < C2** | medium |
| **Decision** | **Row 6 → PIVOT (P-1)**, with row 5 → KILL the second most likely | — |

### 6.1 Why the prediction is pessimistic

C1 gives a competent engineer a shared trial corpus and interaction-term regression. Connecting
*"oracle mutation score 0.95"* to *"environment manifest varied"* to *"pass rate differs between
the two groups"* is what competent engineers do with that data. Our edge is concentrated in
**refusal and falsification discipline**, and C2 has both.

**Estimated probability of a clean CONTINUE (row 9): ~15 %.** Down from 35 % at Gate 0.9, because
specifying C2 honestly made the comparison harder — which is what specifying it honestly was for.
