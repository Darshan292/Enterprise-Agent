# PRIMARY USER AND WORKFLOW

Status: **GATE 1.0.** Version 0.1.0.

---

## 1. Choosing one user

"All agent developers" is not a user. Four candidates, evaluated against the constraint that
decides the automation thesis: **frequency**.

| Candidate | Est. packages/yr | Has the artifacts? | Consequence of a bad eval | Verdict |
|---|---|---|---|---|
| **AI safety / evaluation researcher** | 3–10 | Yes | Reputational, scientific | **Rejected.** At 10/yr the engine saves ~33 h against ~105 h/yr maintenance. Negative forever. The protocol serves them; the engine cannot. |
| **Benchmark / evaluation engineer (single suite)** | 10–40 | Yes | A benchmark that misleads its consumers | **Rejected.** Below the ~32/yr maintenance break-even, or barely at it. They also know their own suite too well to need an audit of it. |
| **Enterprise AI platform / evaluation team, gating its own releases** | ~72 | Yes | Shipping a broken agent | **Marginal.** 5.4-year payback. Not enough to justify a build. |
| **AI platform team operating SHARED evaluation infrastructure across multiple product teams** | **150–400** | Yes — they own the harness | Shipping a broken agent, **across several products** | **SELECTED** |
| QA / reliability engineer | Varies | Partially | Flaky suite | Rejected — mature flaky-test infrastructure already covers most of their need. |

### 1.1 The selected user

> **The engineer who owns shared evaluation infrastructure for an AI platform team serving
> multiple product teams** — the person other teams' agents must pass through before release.

**Not** the person who writes the agent. **Not** the person who writes the benchmark. The person
who operates the gate, and who is accountable when a green evaluation turns out to have been
meaningless.

`ASSUMPTION` (owner: architect; **invalidated by `OQ-25`**): such a role exists, is staffed, and
processes ≥ 150 evaluation packages per year. **This is the single most consequential unverified
assumption in the project.** The frequency requirement is *derived* (§3), not chosen, and it is
high enough that this role is rare.

---

## 2. The workflow

| | |
|---|---|
| **Trigger event** | A product team submits an agent for a release gate, **or** an existing suite's result shifts without an explained cause, **or** a model/dependency/environment version changes under an existing suite. |
| **Input artifacts already available** | The evaluation harness and its config; task fixtures; grader/oracle code; per-trial records and logs; the agent under test or its trace; CI environment manifests; the reported score. **All of this already exists** — the platform team owns the harness. No new instrumentation is required to start. |
| **Output they need** | A go/no-go on *whether the evaluation result can be trusted*, with the specific defect named when there is one, and an explicit "cannot determine" when there is not. Machine-readable, so the gate blocks automatically. |
| **Frequency** | `OQ-25`. Assumed 150–400/yr for a shared-infrastructure team. |
| **Consequence of a bad evaluation** | An agent ships on a green score that measured nothing — the failure surfaces in production, and the evaluation that missed it is trusted by every subsequent team. **The second-order cost is larger than the first: a discredited gate stops being used.** |

### 2.1 Current manual workflow, and its cost

The C2 protocol, timed per step. `ASSUMPTION` throughout; `OQ-27` should replace these with
measured times.

| Step | First-time | Recurring |
|---|---|---|
| Read eval config / harness | 30 m | 30 m |
| Environment fingerprint review | 20 m | 10 m |
| Reproducibility check (active time) | 30 m | 15 m |
| Mutation / oracle testing setup + review | 90 m | 25 m |
| Contamination scan | 20 m | 5 m |
| Grader integrity checks | 40 m | 15 m |
| Cross-reference pass | 60 m | 45 m |
| Falsification interventions (3 × 20 m) | 60 m | 50 m |
| Write the report | 45 m | 20 m |
| **Total** | **6.6 h** | **3.6 h** |

**The recurring figure is what matters.** First-time cost is paid once per framework; recurring
cost is paid per package, forever, and is what an engine can remove.

Note where the recurring time concentrates: **cross-reference (45 m) and falsification
interventions (50 m) are 44 % of it.** Those are precisely the two steps no existing tool performs
(`COMPETITIVE_2026.md` §3). The scans and mutation runs are already scriptable and are not where
the human time goes.

### 2.2 Why existing tools do not complete this workflow

| Step | Covered by existing tooling? |
|---|---|
| Environment fingerprinting | **Yes** — mature |
| Reproducibility / flakiness | **Yes** — mature |
| Mutation / oracle testing | **Yes** — mature |
| Contamination scanning | **Yes** — mature |
| Grader integrity checks | **Mostly** |
| Fault injection | **Yes** — AgentChaos-class |
| Trajectory diagnosis | **Yes** — AgentRx-class |
| Audit trail / probes | **Yes** — NIST-class |
| **Cross-referencing the above into a validity judgement** | **No tool. A human does it.** |
| **Falsifying a candidate before attributing** | **No tool. A human does it, when they think to.** |
| **Emitting an earned, machine-checkable "claim not supported" status** | **No tool.** |

> Eight of eleven steps are solved. **The unsolved three are the ones that consume 44 % of the
> recurring human time and produce the actual judgement** — which is why the claim is economic
> rather than technical, and why `COMPETITIVE_2026.md` refuses to call it "integration".

---

## 3. The frequency requirement, derived

Not chosen. Computed from the workflow model.

```
recurring manual cost   = 3.6 h / package   (§2.1)
A-INT recurring cost    = 0.33 h / package  (review the artifact, approve or dispute)
saving                  = 3.25 h / package

ASSUMPTION  build = 700 h (~4 person-months, MVDC scope)      <- OQ-27
ASSUMPTION  maintenance = 15 %/yr = 105 h/yr
```

| Packages / yr | Hours saved | Net of maintenance | Payback on build |
|---|---|---|---|
| 6 | 20 | **−86** | never |
| 24 | 78 | **−27** | never |
| 32 | 105 | **0** | **maintenance break-even** |
| 50 | 162 | +58 | 12.2 yr |
| 72 | 234 | +129 | 5.4 yr |
| **150** | **488** | **+382** | **1.8 yr** |
| 300 | 975 | +870 | 0.8 yr |

> **Two thresholds, both derived:**
> **32 packages/year** covers maintenance alone — below this the engine is a liability.
> **~150 packages/year** repays the build inside 2 years, which is the threshold T1.0 adopts.

Every figure scales linearly with the build estimate (`OQ-27`) and with the saving per package.
Halving the saving doubles both thresholds.

---

## 4. Why this user might still be the wrong choice

Stated because selecting a user is the easiest place to be optimistic.

| # | Risk |
|---|---|
| U1 | **The role is rare.** Shared evaluation infrastructure across multiple product teams is a large-organisation structure. Most places have neither the volume nor the role. |
| U2 | **They have probably already built something.** A team at 150+ packages/year has felt this pain and scripted part of it. The engine competes with their in-house scripts, not with nothing. |
| U3 | **They are the hardest user to sell.** They own the harness, have opinions about it, and adopting an external assurance layer means admitting their own gate may be unsound. |
| U4 | **Adapter cost per framework is theirs to pay.** Onboarding a new evaluation framework is setup cost (`AUTOMATION_UTILITY.md` column F), and a platform team serving many teams has many frameworks. **High framework diversity can erase the frequency advantage entirely** — the thing that gives them volume is also the thing that raises their setup cost. |

**U4 is the sharpest.** The user selected *for* frequency may be the user whose setup cost scales
with the same variable. `AUTOMATION_UTILITY.md` column F must be measured, not assumed.
