# GATE 1.0 — PRODUCT THESIS & AUTOMATION UTILITY CHALLENGE

Status: **GATE 1.0. NO IMPLEMENTATION. NO BENCHMARK EXECUTION. NO E2 COMMISSION.**
Architecture frozen **only as a research hypothesis**. Version 0.1.0 · 2026-09-22

---

## 0. The correction this gate accepts, and the challenge it adds

**Accepted.** Gate 0.95 read C2 wrongly. *"A competent human can execute the protocol"* does not
imply *"the engine is unnecessary."* Compilers did not become unnecessary because people can write
assembly. The question is **recurring cost at frequency**, not one-shot capability.

**The challenge that correction invites.** An automation thesis is **entirely contingent on how
often the workflow runs.** A tool saving 3 hours per run, run six times a year, saves 20 hours
annually and costs months to build and more to maintain. So the load-bearing number in this whole
gate is not correctness — it is **packages per year**, and §2 of `PRIMARY_USER_AND_WORKFLOW.md`
computes what it has to be.

> **`FACT` (from the workflow model): the engine must cover ~32 packages/year to pay its
> maintenance alone, and ~215 cumulative packages to repay its build. At 72 packages/year —
> a release-gating team running two releases a month across three suites — payback is 5.4 years.
> At 150/year it is 1.8 years.**
>
> **The automation thesis therefore requires a user operating at CI-like frequency or across a
> portfolio of teams. It does not survive a single team gating its own releases.**

---

## 1. The frozen product thesis

### 1.1 The candidate wording, challenged

> *"A reusable evaluation-assurance system automatically executes the validity/falsification
> protocol for agent evaluations, detects cross-dimension validity defects, refuses unsupported
> attribution, and produces a machine-readable assurance artifact while materially reducing the
> recurring human effort required to perform the same protocol manually."*

**Four defects:**

| # | Problem |
|---|---|
| 1 | **"materially reducing"** is unquantified — the same vagueness as "integrated workflow", relocated to the efficiency claim. |
| 2 | **No correctness floor.** It claims efficiency without stating what correctness must be preserved. An engine that is fast and wrong satisfies it. |
| 3 | **No frequency condition**, so the economic claim is unfalsifiable. Every automation claim is true at infinite frequency and false at zero. |
| 4 | **No user and no artifact consumer.** "Machine-readable" is only valuable if something downstream reads it and acts. |

### 1.2 The frozen thesis

> **T1.0 —** For an **AI platform team operating shared evaluation infrastructure across multiple
> product teams**, processing **≥ 150 evaluation packages per year**, a reusable assurance engine:
>
> **(a)** automatically executes the falsification protocol on an evaluation package supplied
> through a declared adapter;
> **(b)** **misses no validity defect that a competent human executing the same protocol (C2)
> detects** — zero unfavourable discordance across the defect registry;
> **(c)** emits a machine-readable assurance artifact carrying an **earned** `claim_supported`
> status — set only after a falsification attempt — **on which a release gate can programmatically
> block**;
> **(d)** reduces recurring human effort from **≈ 3.6 h to ≤ 0.5 h per package**; and
> **(e)** does so at a total cost of ownership that repays its build within **≤ 3 years** at the
> user's measured package volume.

**Falsifiable on five independent axes.** (a) is a capability check. (b) is a strict, measurable
correctness floor that errs against us (`STATISTICAL_CORRECTIONS.md` §2). (c) is binary — either a
gate blocks on it or it does not. (d) is measured in minutes. **(e) is arithmetic on the user's own
volume and can fail before anything is built.**

### 1.3 What T1.0 deliberately does not claim

No novelty for fault injection, contamination detection, trajectory analysis, constraints, audit
trails, interaction testing, reduction, statistical testing, replay or mutation testing. **Nor for
"integration"** — `COMPETITIVE_2026.md` §4 rejects that answer explicitly.

---

## 2. Product-vs-protocol decision tree — frozen before any benchmark

| # | Condition | Decision |
|---|---|---|
| **1** | A-INT > C1 **and** A-INT materially > C2 on correctness | **Engine thesis supported** |
| **2** | A-INT ≈ C2 on correctness (zero unfavourable discordance) **and** meets the automation-utility criterion (`AUTOMATION_UTILITY.md` §3) | **Engine thesis supported through automation** |
| **3** | A-INT ≈ C2 **and** no meaningful automation advantage | **Pivot to protocol + conformance suite** |
| **4** | A-INT ≈ C1 | **KILL** |
| **5** | C2 materially outperforms A-INT | **Kill the engine thesis** unless a strong economic advantage survives — and a strong economic advantage on top of worse correctness is not acceptable, so in practice **KILL** |
| **6** | No defensible user or recurring workflow identified | **KILL** |

### 2.1 Row 6 can fire today

Row 6 requires no benchmark, no baseline and no defects. It requires one number: the **measured**
annual evaluation-package volume of one named candidate user. If it is below ~50, rows 1–5 are
irrelevant because the tool cannot repay itself at any correctness level.

---

## 3. Gate 1.0 exit criteria

The next gate may be authorised only when every row is explicit and recorded.

| # | Criterion | Status |
|---|---|---|
| 1 | **One primary user** | ✅ AI platform team operating shared evaluation infrastructure — `PRIMARY_USER_AND_WORKFLOW.md` §1 |
| 2 | **One primary workflow** | ✅ Pre-release evaluation-package assurance — §2 |
| 3 | **One exact product interface** | ✅ `PRODUCT_INTERFACE.md` — third-party implementable |
| 4 | **One falsifiable product thesis** | ✅ T1.0, §1.2 |
| 5 | **One defined C2 comparison** | ✅ `AUTOMATION_UTILITY.md` §2 six-column matrix |
| 6 | **One defensible automation-utility criterion** | ✅ `AUTOMATION_UTILITY.md` §3, derived from the workflow model |
| 7 | **Corrected statistical denominators** | ✅ `STATISTICAL_CORRECTIONS.md` — n = 112 was the n *required*, never available; real MDE ≈ 0.29–0.32 |
| 8 | **Corrected baseline adjudication** | ✅ `BASELINE_ADJUDICATION.md` — veto replaced by a challenge with a standard of review |
| 9 | **Corrected E2 authorship plan** | ✅ `E2_AUTHORSHIP_PLAN.md` — 4 authors × 5, two substrates, bias reported |
| 10 | **Competitive differentiation survives challenge** | ⚠️ **CONDITIONAL** — survives as an *economic* claim only; `COMPETITIVE_2026.md` §4. **Blocked on `OQ-24`.** |

**Row 10 is not clean, and row 1's frequency assumption is unverified.** Neither is a reason to
proceed quietly.

---

## 4. The Gate 1.0 answer: engine, engine+protocol, protocol, or nothing

**Conditional: (B) engine + protocol — if and only if a named user's measured volume is ≥ 150
packages/year. Otherwise (C) protocol/conformance standard.**

Reasoning, in order of force:

1. **The capability exists.** C2 — a human running the protocol — does everything the engine does.
   `COMPETITIVE_2026.md` §4 finds no *technical* capability underserved after combining NIST's
   probes and cheating work, AgentRx, AgentChaos and current frameworks. What is underserved is a
   **cost point**.
2. **A cost point is a legitimate product thesis** — compilers, CI, linters and type checkers are
   all "a human could do this, at a cost." But it is legitimate **only at frequency**, and the
   frequency required here is high.
3. **The protocol has standalone value regardless.** A specification plus a conformance suite is
   useful to every user, including those below the frequency threshold, and costs a fraction of
   the engine. It should be written **first and separately**, not as a consolation prize.
4. **(A) engine-only is rejected outright.** Shipping an engine whose protocol is undocumented
   means nobody can check whether the engine is right, and the protocol is the part that
   generalises.
5. **(D) nothing** remains live: rows 4, 5 and 6 of the decision tree, plus the carried kill
   conditions from Gate 0.95 (K-6 declining a powered design, K-7 no owner).

`DESIGN_DECISION`: **build the protocol and conformance suite first, in both branches.** It is
required for (B), it *is* (C), it is cheap, and it is the only artifact that survives a KILL.

---

## 5. Open questions blocking Gate 1.0 exit

| ID | Question | Blocks |
|---|---|---|
| **`OQ-24`** | **The September 2026 systematic survey on validity-centered agent evaluation has not been read.** A survey means the field has a name and an enumerated open-problems list. That list is the correct place to check whether our remaining claim is already named. **Reading it could close this project in an afternoon.** | Exit criterion 10 |
| **`OQ-25`** | **Measured annual evaluation-package volume for one named candidate user.** Decides decision-tree row 6 and thesis clause (e). | Everything |
| **`OQ-26`** | Does any candidate user's release gate actually have a programmatic block point an artifact could feed? If not, thesis clause (c) is decorative. | Thesis (c) |
| **`OQ-27`** | Is the ~700 h build estimate defensible for the MVDC scope? Every economic conclusion scales with it. | Automation criterion |
| `OQ-16` | Local stochastic agent availability (carried) | L-STO claims |
| `OQ-18` | Real evaluation traces authored outside the project (carried) | External validity |

**`implementation_authorized` remains `false`.**
