# BASELINE EXPERIMENT — PRE-REGISTRATION

Status: **GATE 0.95 — PRE-REGISTRATION DRAFT. NOT FROZEN. NOTHING RUN.**
Version 0.1.0

Freezing order, which is itself part of the protocol:
```
 1. freeze METRICS.md                  (done, this gate)
 2. freeze this document + registry     <-- pending: the E2 commission decision
 3. THEN build BASELINE-C
 4. THEN freeze the A-INT build (digest)
 5. THEN run
```
Building BASELINE-C before the metrics are frozen would let the metrics be shaped around what the
baseline happens to miss.

---

## 1. Global parameters

| Parameter | Value |
|---|---|
| Packages | **30** — 24 cross-dimension defects + 6 null controls |
| Replications | **5** per package (not 10 — replication buys per-defect precision, not population power) |
| Budget per package | **B = 2,000 trial-equivalents** (≈ 67 min at 2 s), **equal wall-clock** |
| Arms | A-INT · C0 · C1 (primary) · C2 (diagnostic) |
| Blinding | Packages shuffled, anonymised; defect count and null identity withheld from every arm |
| Primary test | Exact McNemar, paired on defects, **E2 subset**, α = 0.05, two-sided |
| Unit of analysis | **The defect** |
| Scoring | **Once**, against the sealed registry |

---

## 2. Per-defect pre-registration

`GT` = ground truth. `Pred C1` / `Pred A-INT` = our **pre-registered predictions**, recorded so
the outcome can contradict them. `Interv` = intervention required for correct resolution.

| ID | GT dimensions | obs | E | Pred C1 | Pred A-INT | Interv | Reps | Budget |
|---|---|---|---|---|---|---|---|---|
| **X-01** | env × oracle | O | E0 | **detect** | detect | N | 5 | B |
| **X-02** | env × oracle | P | E1 | detect | detect | N | 5 | B |
| **X-03** | env × oracle + decoy | O | E0 | detect, **decoy retained** | detect, decoy rejected | **Y** | 5 | B |
| **X-04** | env × contamination | P | E0 | detect | detect | N | 5 | B |
| **X-05** | env × contamination | O | **E2** | **detect** | detect | N | 5 | B |
| **X-06** | state × contamination | **U** | E0 | miss | **detect via canary** | Y | 5 | B |
| **X-07** | state × contamination | P | E1 | detect | detect | N | 5 | B |
| **X-08** | agent × grader + decoy | O | E0 | detect, **decoy retained** | detect, decoy rejected | **Y** | 5 | B |
| **X-09** | agent × grader | O | **E2** | **detect** | detect | N | 5 | B |
| **X-10** | stochasticity × oracle | O | E0 | detect | detect | N | 5 | B |
| **X-11** | stochasticity × oracle | O | E1 | **detect** | detect | **Y** | 5 | B |
| **X-12** | tool × evaluator + decoy | P | E0 | detect, decoy retained | detect, decoy rejected | Y | 5 | B |
| **X-13** | tool × evaluator | **U** | E1 | miss | **miss** | N | 5 | B |
| **X-14** | history × env | O | **E2** | **uncertain** | detect | N | 5 | B |
| **X-15** | history × env | P | E0 | detect | detect | N | 5 | B |
| **X-16** | concurrency × oracle + decoy | O | E0 | **attribute wrongly** | **INCONCLUSIVE(NON_SURGICAL)** | **NS** | 5 | B |
| **X-17** | concurrency × oracle | O | **E2** | **detect** (shuffle check) | detect | N | 5 | B |
| **X-18** | context × benchmark | O | E0 | detect (negative control) | detect | Y | 5 | B |
| **X-19** | context × benchmark | **U** | E1 | miss | **INSUFFICIENT_OBSERVABILITY** | N | 5 | B |
| **X-20** | dependency × scoring + decoy | O | E0 | detect, decoy retained | detect, decoy rejected | **Y** | 5 | B |
| **X-21** | dependency × scoring | O | **E2** | **detect** | detect | N | 5 | B |
| **X-22** | lineage × split | **U** | E0 | miss | **uncertain** | N | 5 | B |
| **X-23** | lineage × split | O | E1 | **uncertain** | detect | **Y** | 5 | B |
| **X-24** | order × grader | O | **E2** | **detect** (shuffle check) | detect | N | 5 | B |

### 2.1 Null controls

| ID | Contents | Pred C1 | Pred A-INT |
|---|---|---|---|
| **N-01** | clean | no finding | no finding |
| **N-02** | spurious correlation | **possible false positive** | no finding |
| **N-03** | high legitimate stochasticity | possible false positive | no finding |
| **N-04** | constant unfamiliar env dimension | no finding | no finding |
| **N-05** | legitimately high indeterminate rate | no finding | no finding |
| **N-06** | **externally authored VALID evaluation** | no finding | **uncertain — our own highest false-positive risk** |

---

## 3. What the predictions say about us

Counting the table above:

| | C1 predicted detect | A-INT predicted detect |
|---|---|---|
| All 24 | **18** | **21** |
| **E2 subset (6)** | **5** | **6** |

> **On the six E2 defects — the primary population — we predict C1 detects five and A-INT detects
> six. That is a single-defect margin, on n = 6.** Exact McNemar on one discordant pair gives
> p = 1.0. **Our own pre-registered prediction is that the primary test cannot reach
> significance.**

This is not a reason to change the prediction. It is the reason `METRICS.md` §3 exists and the
reason the E2 commission (§6) should happen before anything runs.

**Where we predict a genuine edge, it is concentrated in four places:**
- **X-16** — C1 attributes wrongly, A-INT returns `INCONCLUSIVE(NON_SURGICAL)`. *Refusal as the
  correct answer.*
- **X-03 / X-08 / X-12 / X-20** — decoy rejection by intervention.
- **X-06** — canary-based detection of an unrecorded residue.
- **X-19** — `INSUFFICIENT_OBSERVABILITY` instead of a guess.

**Every one of those is the falsification protocol, not the integration.** Which is precisely why
C2 exists — and why we expect C2 to close most of the gap.

---

## 4. False-positive behaviour, pre-registered

| Arm | Predicted FDR on nulls | Reasoning |
|---|---|---|
| C0 | high | six uncoordinated reports, no shared threshold discipline |
| C1 | **moderate** — N-02 and N-03 at risk | engineer sees a correlation and calls it |
| C2 | low | the protocol forces falsification before calling |
| A-INT | low, **except N-06** | tuned on our own conventions; an unfamiliar-but-valid design is our worst case |

**N-06 is where we predict our own worst behaviour.** It is in the registry for that reason.

---

## 5. Cost

| Item | Cost |
|---|---|
| 30 packages × 5 reps × 4 arms × B | ≈ 1.2 × 10⁶ trial-equivalents ≈ **28 days serial at 2 s**; parallelisable across packages |
| BASELINE-C build | **≈ 3 weeks, one engineer** (`BASELINE_C_SPEC` §6) |
| Defect authoring | dominant; E2 defects require external time |
| Self-audit (60 blind audits) | ≈ 2 days |
| **Remote LLM calls** | **0** |

---

## 6. The E2 commission decision — required before freezing

`METRICS.md` §3 and §3 above agree: **the primary test on 6 E2 defects cannot reach significance
under any realistic outcome.** Options:

| # | Option | Assessment |
|---|---|---|
| 1 | **Commission 18 more E2 defects (total 24)** | **Recommended.** Converts the primary claim from unpowered to testable. Cost: external author time. Everything else about the experiment stays as designed. |
| 2 | Run as designed; report the primary as a **non-result** | Honest, and wastes the compute |
| 3 | Promote E1 into the primary population | **Rejected** — E1 authors are team members; this reintroduces CH-2 |
| 4 | Lower α or switch to a one-sided test | **Rejected** — post-hoc evidential-bar adjustment is the defect this gate exists to prevent |

> **Do not freeze this document until option 1 or option 2 is chosen explicitly and recorded.**
> Discovering after the run that the primary test could never have passed would be the fourth
> self-inflicted validity defect in this project's history, and the first one we had been warned
> about in advance by our own arithmetic.
