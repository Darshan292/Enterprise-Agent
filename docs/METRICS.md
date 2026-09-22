# METRICS

Status: **GATE 0.95 — FROZEN BEFORE BASELINE-C IS BUILT.** Version 0.1.0.

**There is no single evaluation-integrity score, and none will be provided.** A composite would be
quoted without its components, and the components disagree in ways that matter.

---

## PRIMARY — Cross-Dimension Detection Rate (CDDR)

```
CDDR(arm) = (cross-dimension defects detected) / (cross-dimension defects present)

detected := the arm reports INVALID_EVALUATION or another non-assurance verdict
            AND names >= 1 of the two implicated dimensions
```

| Property | Value |
|---|---|
| Unit of analysis | **The defect.** Replications are summarised to a per-defect majority first. |
| Primary population | **E2 (externally authored) defects only** |
| Secondary population | E0/E1, reported separately, **cannot support a CONTINUE** |
| Primary comparison | A-INT vs **C1** |
| Diagnostic comparison | A-INT vs **C2** |
| Test | Exact McNemar (sign test) on paired defect outcomes, α = 0.05, two-sided |
| **Strict variant** | **CDDR-strict**: both implicated dimensions named. Reported always. |

### Why "≥ 1 dimension" and not "both"
Requiring both would score a correct detection with partial attribution as a miss, which
overstates the difficulty. Requiring neither would score "something is wrong somewhere" as a
detection, which overstates performance. Reporting both variants is the only honest option, and
**CDDR-strict is the number to quote if the two disagree**.

---

## SECONDARY

### S1 — False Assurance Rate (FAR)
```
FAR(arm) = P( arm reports NO defect | a defect is present )
```
over **all** defective packages, not only cross-dimension ones.

**`INDETERMINATE` does not count as assurance.** A system saying "I cannot tell" has not falsely
assured.

> **FAR alone is trivially gamed by always returning `INDETERMINATE`.** It is therefore
> **never reported or thresholded without S2 beside it.** The pair is the metric; neither half
> is.

### S2 — Indeterminate Rate
```
IR(arm) = P( arm returns INCONCLUSIVE / INDETERMINATE | any package )
```
Reported separately for defective and null packages. A high IR on defective packages is a
usefulness failure; a high IR on nulls is correct caution.

### S3 — False Discovery Rate on nulls
```
FDR_null(arm) = P( arm reports a defect | package is a null control )
```
An assurance system that fires on valid evaluations trains its users to ignore it, while looking
rigorous. N-04 (constant unfamiliar environment) and N-06 (externally authored *valid* evaluation)
are the informative cells here.

### S4 — Detection precision
```
precision = (correctly attributed dimensions) / (dimensions named)
```
**Decoys are the test.** X-03, X-08, X-12, X-16, X-20 each carry an inert correlated factor; naming
it is a precision failure.

### S5 — Detection recall
```
recall = (implicated dimensions named) / (implicated dimensions present)
```
Equals CDDR-strict when both dimensions are required.

### S6 — External-defect detection rate
CDDR restricted to **E2**. Listed separately from the primary because it is also a **kill gate**:
an advantage that exists only on E0/E1 is an advantage over our own imagination
(`KILL_CONTINUE_0_95` K-4).

### S7 — Experiment cost
Reported under **all three budget bases**: equal wall-clock (primary), equal trials, **equal
engineer-hours** (which strongly favours BASELINE-C and is reported because omitting it would be
convenient).

### S8 — Time to diagnosis
Wall-clock from package receipt to the arm's first correct finding. A correct verdict reached at
the budget ceiling is worth less than the same verdict at 20 % of it, and neither CDDR nor FAR
sees the difference.

### S9 — Minimum detectable effect
Computed and **printed before the run** for every thresholded metric.

| Metric | n | MDE |
|---|---|---|
| CDDR, E2 subset | **6 defects** | **Cannot reach p < 0.05 under realistic discordance** — see §3 |
| CDDR, all 24 | 24 defects | detects an 80 % win rate among 12 discordant (p = 0.039) |
| FAR | 112 per arm | 0.15 difference around 0.20 |
| FAR | 200 per arm | 0.10 difference around 0.15 |
| FDR_null | 30 (6 nulls × 5) | 0/30 → upper bound 0.116 |

### S10 — Route correctness
For defects requiring intervention (X-03, X-08, X-11, X-16, X-20, X-23): was the attribution
supported by a **recorded controlled intervention**, or by association alone?

This is the metric that separates A-INT from C1 **if anything does** — C1 has no protocol
requiring intervention before attribution, though its engineer may choose to intervene anyway.
C2 does. **If A-INT and C2 tie on S10, the protocol is the contribution and the engine is not.**

---

## 2. Metric pairings — rules that prevent gaming

| Rule | Prevents |
|---|---|
| **FAR is never reported without IR** | Always-`INDETERMINATE` winning FAR |
| **CDDR is never reported without FDR_null** | Always-`INVALID` winning CDDR |
| **CDDR is never reported without CDDR-strict** | "Something is wrong somewhere" counting as attribution |
| **Precision is never reported without recall** | Naming one dimension and stopping |
| **Every rate carries n, Wilson interval, and MDE** | Point estimates read as facts |
| **E2 and E0/E1 are never pooled** | Team-authored defects carrying the claim |
| **No composite score, at any level** | The components disagree; a composite hides it |

---

## 3. The primary metric is underpowered, and this is stated before the run

`FACT`: exact McNemar on **6 E2 defects** cannot reach p < 0.05 unless all six are discordant and
all six favour A-INT (p = 0.031) — a best case that assumes BASELINE-C detects **none** of the six
while A-INT detects all six. Any other pattern is not significant.

Three honest options, in order of preference:

1. **Commission 24 E2 defects.** The single cheapest improvement to this gate. Cost: one external
   author's time. It converts the primary claim from underpowered to testable.
2. **Run as designed and report the primary as a non-result**, with E0/E1 as descriptive only.
   `KILL_CONTINUE_0_95` treats an underpowered primary as a **non-result, not a pass**.
3. Lower the evidential bar. **Rejected** — it is the failure this whole gate exists to prevent.

> **Option 1 should be taken before anything is run.** Discovering afterwards that the primary
> test could never have passed would be a validity defect in our own experiment, of exactly the
> kind this project claims to detect — and it would be the third time.
