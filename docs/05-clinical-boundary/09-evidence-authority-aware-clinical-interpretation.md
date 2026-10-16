# Evidence/Authority-Aware Clinical Interpretation for Multimodal AUD Physiological Monitoring

**Technical research/design document**  
**Version:** 1.0  
**Research cut-off used for this document:** 16 October 2026  
**Primary design question:** *Given all available evidence, what is the strongest clinician-facing statement this system is actually justified in making?*

---

## Executive conclusion

The final interpretation layer should **not** be implemented as a single confidence score and should **not** convert reliable physiology directly into an AUD diagnosis. The recommended architecture is a **provenance-preserving evidence graph with an explicit authority ceiling**.

The system should maintain four quantities separately:

1. **truthworthiness of the observations** — whether the measurements/features/events are technically and physiologically credible;
2. **target specificity** — how specifically those observations support the particular AUD claim rather than nonspecific autonomic activation or another cause;
3. **clinical-evidence maturity** — what kind of evidence exists for making that claim at an individual-patient level and how well it has been validated;
4. **action authority** — whether prospective evidence shows that surfacing or acting on the claim improves clinically meaningful decisions/outcomes.

This separation follows the logic of IMDRF clinical evaluation, which distinguishes **clinical association**, **analytical validation**, and **clinical validation** rather than treating “the algorithm works” as one undifferentiated fact [1]. It is also consistent with current FDA clinical-decision-support guidance emphasizing that clinicians should be able to understand the basis of recommendations, the input information, validation, and known/unknown limitations [2].

The central rule is:

```text
strong measurement evidence
        does not imply
strong AUD-specific evidence
        does not imply
validated clinical probability
        does not imply
authority to recommend an intervention
```

The implementation should therefore use:

```text
UPSTREAM EVIDENCE
    ↓
claim-specific evidence graph
    ↓
hard-gate checks
    ↓
applicability-domain check
    ↓
clinical-evidence maturity + reference-standard quality
    ↓
specificity / uncertainty / calibration assessment
    ↓
AUTHORITY CEILING
    ↓
evidence accumulation only within that ceiling
    ↓
statement class + surfacing tier
    ↓
provenance + missing/conflicting evidence + reason codes
```

The recommended authority mechanism is **hybrid**:

- **weakest-link / ceiling logic** for facts that cannot be compensated for (e.g., outside validation domain, insufficient AUD specificity, invalid required signal, uncalibrated probability);
- **cumulative evidence** for factors that legitimately strengthen a claim within the permitted ceiling (e.g., persistence, repeated episodes, independent modalities, mature personal baseline, corroborating clinical observations).

For the system described here, the current literature generally supports **descriptive physiological**, **physiological-state**, and in selected settings **cautious AUD-compatibility** statements. It does **not** currently support treating wearable autonomic physiology alone as a general-purpose diagnostic authority for withdrawal, craving, relapse, or treatment decisions. This is especially important because a 2026 multicenter VHA study found that withdrawal scales incorporating vital signs were associated with longer treatment duration and did not reduce complicated withdrawal compared with CIWA-Ar-based management [25]. Reliable objective physiology can be useful evidence without being sufficient clinical evidence.

---

# 1. Scope and position in the pipeline

The assumed overall system is approximately:

```text
body
↓
sensing
↓
acquisition
↓
signal trustworthiness
↓
physiological decomposition
↓
confound-aware monitoring
↓
context-aware interpretation
↓
multimodal physiological fusion
↓
patient-specific / longitudinal modeling
↓
temporal physiological event modeling
↓
AUD-specific physiological meaning
↓
EVIDENCE / AUTHORITY-AWARE CLINICAL INTERPRETATION
↓
clinician-facing observation / alert / recommendation boundary
```

This document does **not** re-design signal quality, physiological decomposition, confound removal, context recognition, fusion, personalization, or temporal event detection. Those are treated as **upstream evidence producers**. Their outputs are consumed together with validation metadata and clinical corroboration.

The problem addressed here is narrower:

> **How much clinical meaning is the system justified in expressing, given the reliability, completeness, specificity, validation strength, applicability, uncertainty, and clinical consequences of the evidence?**

The exact order of fusion, personalization, temporal reasoning, and AUD-specific interpretation can differ internally. What matters at the authority layer is that the provenance and uncertainty of each upstream contribution remain available.

---

# 2. The conceptual distinction the system must preserve

A clinician-facing system must not collapse the following into one score:

| Quantity | Question | Example |
|---|---|---|
| Signal confidence | Is the measurement technically trustworthy? | ECG beat detection is reliable |
| Feature confidence | Is the derived metric reliable? | RMSSD estimated from an acceptable window |
| Physiological-state confidence | Is a physiological state genuinely present? | Sustained autonomic activation is supported |
| AUD-specificity | Does the physiological state distinguish an AUD process from alternatives? | How specifically does this support withdrawal? |
| Model confidence | What probability/score did a model output? | `p=0.82` |
| Calibration | Does `0.82` mean what it claims in the target population? | Events occur ≈82% among comparable cases |
| Clinical-evidence maturity | At what inferential level has this mapping been demonstrated? | group association vs external individual prediction |
| Clinical authority | What statement/action is justified? | description vs compatibility vs risk vs review alert |

A key error would be:

```text
autonomic_activation_confidence = 0.95
therefore
P(withdrawal) = 0.95
```

That inference is invalid unless a validated AUD-specific model establishes it in the relevant population and setting. High confidence that a nonspecific physiological state exists is not high specificity for its cause.

---

# 3. Research framework: what transfers and what does not

Two classes of literature are needed.

## 3.1 Authority-framework evidence

Clinical prediction, diagnostic testing, medical AI, decision support, uncertainty, selective prediction, and medical-device evaluation provide reusable machinery:

- clinical association vs analytical/clinical validation [1];
- transparent basis and independent clinician review [2];
- calibration rather than discrimination alone [3];
- uncertainty around individual risk estimates [4];
- abstention when uncertainty is high [5];
- decision-curve analysis and consequence-sensitive thresholds [6];
- likelihood ratios and pre-/post-test probability [7];
- prevalence-dependent predictive values [8];
- selective prediction/reject options [9,10];
- conformal set-valued uncertainty [11];
- dataset-shift/applicability monitoring [12];
- risk-of-bias and applicability assessment [13,17];
- transparent prediction-model reporting [14];
- diagnostic-AI reporting [15];
- early clinical evaluation of AI decision support [16];
- evidence quality for diagnostic strategies and patient-important outcomes [18];
- alarm-fatigue and actionability considerations [19,20].

These methods can define **how authority is granted**.

## 3.2 AUD-specific validation evidence

The actual strength of an AUD statement must come from AUD/SUD evidence. General medical-AI methodology cannot manufacture AUD specificity. AUD literature must establish whether an observation supports:

- withdrawal progression;
- craving/cue reactivity;
- intoxication/recovery;
- autonomic dysregulation;
- treatment/recovery trajectory;
- relapse-related risk;
- sleep/recovery deterioration.

This distinction is essential:

```text
general methodology can define:
    "what evidence would be enough"

AUD-specific studies must establish:
    "whether that evidence actually exists"
```

---

# 4. Operational definition of clinical authority

**Clinical authority** is the maximum strength of machine-generated clinical assertion or attention request that is justified for a specific claim in a specific patient at a specific time.

It is not a biological property and not simply a classifier probability.

Define a claim \(c\) and evidence state \(E_t\) at time \(t\). The system produces:

\[
A_{\max}(c,t) = \min\{
A_{\text{upstream}},
A_{\text{association}},
A_{\text{specificity}},
A_{\text{validation}},
A_{\text{reference}},
A_{\text{domain}},
A_{\text{calibration}},
A_{\text{corroboration}},
A_{\text{utility}}
\}
\]

where each term is an **authority ceiling**, not necessarily a numerical probability.

Then, within the resulting ceiling, cumulative evidence determines whether the claim reaches the lower or upper end of the allowed statement class:

\[
S(c,t) = f(E_{\text{independent}}, E_{\text{temporal}}, E_{\text{personal}}, E_{\text{clinical}})
\]

subject to:

\[
\text{statement\_level}(c,t) \leq A_{\max}(c,t)
\]

The `min` formulation is conceptual: implementation should use explicit rules rather than pretending all ceilings lie on a validated continuous metric.

### Why a ceiling rather than one weighted score?

Because certain defects cannot be compensated for.

Example:

```text
signal quality           excellent
HR deviation             large
HRV deviation            large
EDA                       elevated
multimodal agreement      strong
episode                   persistent
AUD specificity           low
```

A weighted sum might become “very high confidence.” The correct outcome is:

```text
physiological evidence    strong
AUD attribution ceiling   compatibility/association only
```

Adding more copies of the same nonspecific physiology cannot turn it into diagnostic specificity.

---

# 5. Claim-centric evidence graph

The authority layer should operate on a graph, not a flattened feature vector.

```text
ECG quality ──→ HR validity ───────────────┐
      └──────→ HRV validity ──────────────┤
PPG quality ──→ pulse-derived support ────┤
EDA quality ──→ EDA activation ───────────┤
IMU ──────────→ motion confound estimate ─┤
respiration ──→ respiratory evidence ─────┤
temperature ──→ thermal evidence ─────────┤
                                          ↓
                              physiological-state claim
                              "autonomic activation"
                                          ↓
personal baseline ────────────────────────┤
temporal persistence ─────────────────────┤
context/confounds ────────────────────────┤
                                          ↓
                              patient-relative episode claim
                                          ↓
AUD history / abstinence interval ────────┤
validated scale / clinician findings ─────┤
AUD literature evidence grade ────────────┤
                                          ↓
                              AUD-specific claim
                              "withdrawal-related activation"
                                          ↓
calibration / domain / clinical utility ──┤
                                          ↓
                              permitted statement
```

Every node and edge carries:

```yaml
claim_id:
claim_type:
value:
confidence:
specificity:
uncertainty:
source_ids:
time_interval:
population_domain:
validation_status:
reference_standard:
supporting_evidence:
contradictory_evidence:
missing_evidence:
```

The graph provides two advantages:

1. a clinician can inspect **why** a claim exists;
2. the software can determine **which link limits authority**.

---

# 6. Evidence vector

For every candidate clinician-facing claim, compute/store an evidence vector. These fields are deliberately separate.

```text
E(c,t) = {
    M: measurement validity,
    F: feature validity,
    C: confound resolution,
    X: context sufficiency,
    P: personalization maturity,
    U: multimodal support,
    T: temporal support,
    S: target/AUD specificity,
    G: clinical-evidence maturity,
    R: reference-standard quality,
    D: applicability-domain match,
    K: calibration quality,
    Q: uncertainty quality,
    H: clinical corroboration,
    O: outcome/actionability evidence
}
```

## 6.1 Recommended representation

Do **not** force all dimensions into `[0,1]`. Use quantities only where they have defensible meanings.

Example:

```yaml
measurement_validity:
  status: HIGH
  basis: "ECG SQI accepted; beat coverage 98%"
feature_validity:
  status: HIGH
confound_resolution:
  status: HIGH
context_sufficiency:
  status: MODERATE
personalization:
  status: HIGH
  baseline_days: 21
multimodal_support:
  status: HIGH
  independent_families: [cardiac, electrodermal]
temporal_support:
  status: HIGH
  duration_minutes: 42
aud_specificity:
  status: LOW_TO_MODERATE
  basis: "autonomic hyperactivity is nonspecific"
clinical_evidence_maturity:
  grade: CEM-2
reference_standard_quality:
  status: MODERATE
applicability:
  status: IN_DOMAIN
calibration:
  status: NOT_APPLICABLE
clinical_corroboration:
  status: MISSING
actionability_evidence:
  status: INSUFFICIENT
```

---

# 7. Evidence maturity: a claim-specific grade

A conventional evidence hierarchy should not be copied mechanically. GRADE explicitly warns that diagnostic-accuracy evidence can be strong for accuracy while remaining indirect for patient-important recommendations [18]. IMDRF similarly separates clinical association, analytical validation, and clinical validation [1].

For this system, use an engineering **Claim Evidence Maturity (CEM)** scale. It is an adaptation for software gating, not an official GRADE category.

| CEM | Minimum evidence demonstrated for the exact claim | Typical maximum role |
|---|---|---|
| **CEM-0** | mechanistic plausibility / hypothesis only | do not make AUD claim |
| **CEM-1** | replicated group-level association | associative background; no individual inference |
| **CEM-2** | within-person/longitudinal association or replicated state-linked change | cautious patient-level compatibility when current evidence matches; no validated probability |
| **CEM-3** | individual-level discrimination/prediction with proper subject-independent validation; defined reference standard | cautious model-based interpretation in matched setting; usually research/decision support |
| **CEM-4** | external validation in independent cohort(s), calibration in intended population, acceptable bias/applicability | calibrated probabilistic interpretation may be allowed |
| **CEM-5** | prospective clinical evaluation and evidence of net clinical benefit/safety for the intended workflow | action-oriented CDS may be allowed within validated scope |

### 7.1 CEM is not enough by itself

Every CEM grade is accompanied by:

```yaml
risk_of_bias: LOW | SOME_CONCERNS | HIGH
applicability: MATCHED | PARTIAL | OUTSIDE
reference_standard: STRONG | MODERATE | WEAK
replication: NONE | SINGLE | MULTIPLE_INDEPENDENT
population: ...
setting: ...
prediction_horizon: ...
```

PROBAST+AI should guide risk-of-bias/applicability assessment for prediction models [13]; QUADAS-2 is appropriate for diagnostic accuracy evidence [17]; STARD-AI/TRIPOD+AI define information that should be available when evaluating reported AI/diagnostic models [14,15].

### 7.2 Individual-level evidence is a separate boundary

The following are **not equivalent**:

```text
group A has lower RMSSD than group B
```

```text
RMSSD tracks state changes within the same patient
```

```text
RMSSD discriminates target vs non-target episodes
```

```text
RMSSD predicts a future outcome
```

```text
RMSSD has externally validated sensitivity/specificity
```

```text
using RMSSD changes clinical decisions and improves outcomes
```

The CEM grade explicitly prevents a group-level biomarker paper from being silently promoted into individual diagnosis.

---

# 8. Reference-standard quality

A model can only inherit the clinical meaning of its target label.

Store a `reference_standard_grade` for every clinical mapping:

| Grade | Example | Interpretation |
|---|---|---|
| **RS-A** | adjudicated clinician diagnosis/validated endpoint with appropriate objective data | strongest |
| **RS-B** | validated clinical scale used within its supported population; structured assessment | strong but target-limited |
| **RS-C** | well-defined self-report/EMA or laboratory state with clear protocol | useful for that construct |
| **RS-D** | retrospective chart label, heuristic proxy, weakly defined target | weak |
| **RS-E** | ambiguous/derived label with no defensible reference | cannot support high authority |

A craving model trained against a self-reported craving rating is a model of that **rating/construct**, not automatically a model of imminent relapse. An intoxication model trained against BrAC threshold can claim discrimination against that BrAC-defined state; it cannot silently become a model of “unsafe to drive” unless that endpoint was validated.

---

# 9. Authority levels and statement classes

Use a stable set of authority classes. The names are software policy labels; the thresholds for entering them must be validated for each claim.

| Authority | Permitted semantic scope | Example | What it does **not** authorize |
|---|---|---|---|
| **A0 — ABSTAIN** | no reliable higher-level interpretation | “AUD-specific interpretation not supported.” | forced classification |
| **A1 — MEASUREMENT** | report valid measurement | “Resting HR 92 bpm.” | abnormality/causation |
| **A2 — PATIENT-RELATIVE FEATURE** | report validated deviation | “Resting HR is elevated relative to personal baseline.” | autonomic syndrome |
| **A3 — PHYSIOLOGICAL STATE** | report supported physiological state | “Sustained multimodal autonomic activation is present.” | AUD attribution |
| **A4 — AUD COMPATIBILITY/ASSOCIATION** | relate state cautiously to AUD process | “Pattern is compatible with withdrawal-related autonomic hyperactivity, but is not diagnostically specific.” | diagnosis/probability unless separately validated |
| **A5 — CALIBRATED AUD PROBABILITY/RISK** | report validated probability for a precise target/horizon | “Externally validated model estimates X% risk of Y over Z horizon in this population.” | automatic intervention |
| **A6 — CLINICIAN-REVIEW CDS** | recommend review/escalated attention | “Clinical reassessment may be warranted.” | treatment prescription |
| **A7 — ACTION RECOMMENDATION** | action/treatment recommendation within a prospectively validated protocol | only where clinical utility and safety are established | autonomous diagnosis/treatment outside protocol |

**For the current physiological-only AUD evidence base, A7 should be considered unavailable.** A6 must be claim- and workflow-specific rather than the default consequence of a high physiological score.

---

# 10. Evidence-to-language mapping

Language must be treated as an output constraint, not cosmetic wording.

| Term | Recommended use |
|---|---|
| **observed / measured** | direct reliable measurement |
| **elevated/reduced relative to baseline** | patient-relative feature with mature enough baseline |
| **supports / evidence of [physiological state]** | multiple reliable inputs support a physiological construct |
| **associated with** | literature shows association; does not imply individual diagnosis |
| **compatible with** | current patient evidence is not inconsistent with a target and there is relevant clinical association, but specificity is limited |
| **suggestive of** | use sparingly; define internally as stronger than generic association but still non-diagnostic |
| **increases/decreases estimated likelihood/risk** | only for a validated model with clear target, population, and horizon |
| **likely / high probability** | only if probability is calibrated and a prespecified validated probability band exists |
| **diagnostic of / confirmed / definitive** | prohibited unless the system has been validated as a diagnostic test for that exact target and use; generally prohibited for the present platform |
| **treatment required / medication indicated** | prohibited unless incorporated into a validated clinical protocol with appropriate human authority |

Do **not** create a universal rule such as `p > 0.80 = likely`. Probability-language bands themselves require validation and usability testing.

---

# 11. Authority ceiling rules

The authority ceiling is the core safety mechanism.

## 11.1 Hard ceilings

Examples:

```text
AUD specificity LOW
→ max A3 or A4 depending on whether a supported association exists

only group-level association (CEM-1)
→ max A4 associative wording; no patient-specific probability

individual model only internally validated (CEM-3)
→ no externally asserted calibrated probability; max cautious A4/A5 research display

no external calibration
→ no numeric clinician-facing risk probability

outside validated patient population
→ A0 for model-based clinical claim; lower-level physiology may still be displayed

action utility not prospectively demonstrated
→ max A5/A6 depending on independently validated review criterion; no A7
```

## 11.2 Important principle

Authority ceilings are **claim specific**.

A case may simultaneously have:

```yaml
heart_rate_measurement: A1
personal_HR_deviation: A2
autonomic_activation: A3
withdrawal_attribution: A4
relapse_prediction: A0
```

There is no need for one global authority value.

---

# 12. Hard gates versus soft evidence

## 12.1 Hard gates

A hard gate prevents a specified higher-level claim.

### Gate H1 — required upstream evidence invalid

If a required signal/feature is not valid:

```text
do not use that evidence
```

If the claim cannot be evaluated without it:

```text
abstain from that claim
```

Do not automatically discard the entire episode if other claims remain supportable.

### Gate H2 — unresolved dominant confound

If a plausible confound can substantially explain the same physiological pattern and has not been resolved, cap causal/AUD attribution.

Example:

```text
tachycardia + EDA ↑ + intense physical activity
→ physiology may be real
→ withdrawal attribution blocked
```

### Gate H3 — missing claim-essential context

If the interpretation requires recent cessation/abstinence interval or another clinical fact and it is unknown, the system cannot manufacture the prior from physiology.

### Gate H4 — no clinical association for the target

Biological plausibility alone does not authorize an AUD-specific statement.

### Gate H5 — outside applicability domain

Examples:

- model validated only in adults but current patient is outside that population;
- severe arrhythmia invalidates HRV model assumptions;
- new sensor/feature pipeline differs materially from validation;
- medication/treatment state was excluded from development and substantially changes physiology.

### Gate H6 — probability requested but model is not calibrated/validated

A raw classifier score must not be shown as a clinical probability [3].

### Gate H7 — action recommendation without action-utility evidence

Accuracy does not establish benefit of acting. Decision-curve/prospective evidence is required for a strong action boundary [6,16,18].

## 12.2 Soft evidence

Soft deficits lower confidence but need not block every claim:

- one optional modality unavailable;
- personal baseline moderately mature rather than highly mature;
- temporal episode shorter than optimal but still valid;
- context partially observed;
- one independent modality disagrees;
- clinical corroboration unavailable for a low-authority descriptive output.

Soft evidence must still be visible in `missing_evidence` or `limitations`.

---

# 13. Evidence quantity, independence, and multimodal redundancy

Do not count feature columns as independent votes.

Five cardiac features extracted from the same R-R intervals:

```text
HR
RMSSD
SDNN
HF power
pNN50
```

do not equal five independent confirmations. Shared physiological pathways and preprocessing make them correlated.

## 13.1 Evidence families

Maintain dependency groups such as:

```yaml
cardiac:
  sources: [HR, RMSSD, SDNN, HF]
electrodermal:
  sources: [SCL, SCR_rate, SCR_amplitude]
respiratory:
  sources: [rate, variability, depth_proxy]
thermal:
  sources: [skin_temperature]
movement:
  sources: [acceleration, activity_state, posture]
clinical:
  sources: [withdrawal_scale, clinician_exam, alcohol_history]
```

## 13.2 Fusion rule

Prefer one of two approaches:

1. **jointly validated multivariable model** that learns dependencies; or
2. **rule/evidence graph with dependency-aware families**, where correlated features strengthen confidence inside a family but do not receive independent evidentiary votes.

Do not multiply independent likelihood ratios from correlated features unless conditional independence is empirically justified.

Multimodal agreement can increase confidence in a physiological state. It does **not** automatically increase target specificity if all modalities respond to the same nonspecific sympathetic activation.

---

# 14. Priors, prevalence, and Bayesian interpretation

The same physiology can have different clinical meaning under different prior conditions.

Examples of prior/context variables:

- known AUD diagnosis;
- recent reduction/cessation;
- abstinence interval;
- detoxification phase;
- previous severe withdrawal/seizure/delirium tremens;
- recent alcohol use;
- treatment setting;
- relevant medication exposure;
- clinician-entered risk factors.

These should be explicit inputs, not inferred from autonomic physiology.

## 14.1 When formal Bayes is valid

When a validated diagnostic test/model supplies likelihood ratios for the intended population:

\[
\text{posterior odds}
=
\text{prior odds} \times LR
\]

with:

\[
LR+ = \frac{\text{sensitivity}}{1-\text{specificity}},
\qquad
LR- = \frac{1-\text{sensitivity}}{\text{specificity}}
\]

Likelihood-ratio interpretation is standard diagnostic methodology [7]. Predictive values also depend on prevalence/pre-test probability [8].

## 14.2 When formal Bayes is **not** valid

Do not invent an LR from:

- group mean differences;
- p-values;
- correlation coefficients;
- feature importance;
- classifier confidence;
- AUROC alone;
- mechanistic plausibility.

If the AUD physiology literature does not provide validated diagnostic performance, represent the update qualitatively:

```yaml
prior_context: HIGH
aud_compatibility: MODERATE
posterior_probability: NOT_ESTIMATED
```

rather than manufacturing a probability.

## 14.3 Multiple evidence items

Never multiply separate LRs for HR, HRV and EDA unless the joint/conditional structure has been validated. Prefer a validated multivariable model or a single joint LR from a prospectively evaluated combination.

---

# 15. Calibration and probabilities

Any clinician-facing probability must answer:

```text
probability of WHAT?
over WHAT horizon?
in WHICH population?
under WHICH setting?
calibrated HOW?
validated WHERE?
```

Calibration is distinct from discrimination; a model can rank cases well and still provide misleading risk estimates [3].

## 15.1 Required calibration reporting

For A5 probability output, require at minimum:

- calibration plot/curve in external or temporally independent data;
- calibration intercept and slope where appropriate;
- Brier score or another proper scoring rule;
- discrimination (e.g., AUROC) as a separate metric;
- relevant calibration-in-the-large;
- subgroup and setting checks where sample size permits;
- recalibration plan under distribution shift.

Expected calibration error (ECE) may be reported for technical analysis but should not replace a calibration plot and clinically meaningful calibration assessment.

## 15.2 Recalibration methods

Potential methods include:

- logistic/Platt recalibration;
- isotonic regression;
- temperature scaling for suitable classification models;
- model intercept/slope updating.

Choice depends on sample size, model type, and validation design. Recalibration does not repair an invalid clinical association or a bad reference standard.

## 15.3 Uncertainty around the risk estimate

A point estimate alone can conceal substantial patient-level uncertainty. Riley et al. emphasize that uncertainty around individual risk estimates can matter for decisions [4]. The system should therefore support:

```yaml
estimated_probability: 0.62
prediction_horizon: "24 h"
uncertainty_interval: ...
calibration_population: ...
```

only when the method and validation support such an interval.

Where formal uncertainty intervals are not valid, expose qualitative uncertainty sources instead.

---

# 16. Aleatoric versus epistemic uncertainty

Separate:

### Aleatoric / irreducible ambiguity

Example:

```text
high-quality autonomic activation
but withdrawal, anxiety, pain, infection, medication effect, and other causes overlap
```

More sensor quality cannot eliminate this. It is primarily a **specificity limit**.

### Epistemic uncertainty

Example:

```text
model trained on a small detox cohort
current patient is from home monitoring with different medications
```

This can potentially be reduced by new validation data.

Recommended schema:

```yaml
uncertainty:
  measurement: LOW
  physiological_state: LOW
  target_ambiguity_aleatoric: HIGH
  model_epistemic: MODERATE
  applicability: MODERATE
```

A single “confidence 0.73” hides the distinction that matters clinically.

---

# 17. Abstention and selective prediction

The system must have a first-class reject state.

Recommended outputs:

```text
INSUFFICIENT_EVIDENCE
AMBIGUOUS
MIXED_EVIDENCE
OUT_OF_DOMAIN
OUT_OF_DISTRIBUTION
CLINICAL_INTERPRETATION_NOT_SUPPORTED
PROBABILITY_NOT_VALIDATED
```

Medical ML uncertainty work explicitly supports abstention for uncertain cases [5]. Selective-classification methods formalize the trade-off between coverage and risk [9,10].

## 17.1 Risk-coverage principle

A model that speaks on 100% of episodes may be less clinically useful than one that:

```text
makes high-level claims on 55% of episodes
and abstains on the rest
```

if the covered set has demonstrably lower error and clinically acceptable risk.

Coverage targets must be chosen in validation data based on the harm profile; they are not arbitrary engineering thresholds.

## 17.2 Abstention is claim specific

An episode can be:

```yaml
physiological_interpretation: SUPPORTED
withdrawal_interpretation: AMBIGUOUS
relapse_prediction: INSUFFICIENT_EVIDENCE
```

The UI should still show the supported lower-level observation.

---

# 18. Conformal prediction: useful, but not a source of clinical validity

Conformal prediction can produce set-valued predictions and an explicit reject/uncertainty behavior. In prostate pathology, conformal prediction substantially reduced errors among accepted predictions while rejecting uncertain cases and also detected external-data differences [11].

For this system, a future validated AUD-state classifier could return:

```text
possible_states =
{
  withdrawal_related_autonomic_activation,
  nonspecific_autonomic_activation
}
```

instead of forcing:

```text
withdrawal
```

## 18.1 Recommended role

Use conformal prediction as an **uncertainty/coverage layer after a clinically meaningful model exists**.

It should not be used to claim that the underlying labels are clinically valid.

## 18.2 Critical assumptions

Classical conformal guarantees depend on exchangeability/i.i.d.-like assumptions. Longitudinal patient data violate naïve random-window assumptions because repeated windows from the same patient are dependent.

Therefore:

- split/calibrate at patient level;
- avoid calibration/test windows from the same subject where subject-independent claims are intended;
- evaluate coverage by patient, setting, state and time;
- use online/adaptive variants only after specific validation;
- treat distribution shift as a separate problem.

---

# 19. Out-of-distribution detection and applicability domain

No single OOD method is sufficiently general for clinical deployment; a 2025 systematic review of dataset-shift methods in health prediction found heterogeneous approaches, limited standardization and relatively little external/prospective evaluation [12].

Use a layered applicability system:

## 19.1 Explicit rule-based applicability

At model registration, store:

```yaml
validated_population:
  age: ...
  diagnosis: ...
  setting: ...
  withdrawal_stage: ...
exclusions:
  - severe arrhythmia
  - ...
sensor_configuration:
  - ECG...
medication_context:
  known_supported: ...
prediction_horizon:
reference_standard:
```

If current case violates a critical exclusion:

```text
OUT_OF_DOMAIN
→ hard gate for the model-based claim
```

## 19.2 Statistical shift monitoring

Monitor distributions of:

- model inputs;
- latent/embedding representations where defensible;
- missingness patterns;
- sensor configuration;
- calibration residuals;
- outcome prevalence;
- subgroup performance.

Methods may include distance/statistical tests or model-based shift detectors. No detector should itself become the sole clinical safety argument.

## 19.3 Model uncertainty

High epistemic uncertainty or conformal rejection can reinforce abstention.

The safest hierarchy is:

```text
explicit domain rule
    +
dataset shift monitoring
    +
uncertainty/reject mechanism
```

not a single “OOD score.”

---

# 20. Missing, contradictory, and negative evidence

## 20.1 Missing evidence

Do not impute authority.

Example:

```yaml
EDA: AVAILABLE
ECG: MISSING
baseline: MATURE
context: KNOWN
```

Possible outcome:

```text
descriptive/EDA physiological claim may remain valid
multimodal claim downgraded
AUD claim may remain capped or abstain depending on required evidence
```

Whether an input is **required** or merely **optional** must be defined per claim/model.

## 20.2 Contradictory evidence

Preserve disagreement:

```yaml
supporting:
  HR: elevated
  EDA: elevated
contradictory:
  HRV: normal
  temperature: normal
state: MIXED_EVIDENCE
```

Do not average all values into a smooth confidence score that conceals conflict.

If a physiological pattern supports withdrawal but a validated clinical scale/clinician assessment does not, surface both. Higher-level authority should usually be downgraded pending resolution.

## 20.3 Negative evidence

Absence is negative evidence only when the marker has enough validated sensitivity under the current conditions.

```text
not detected
≠
ruled out
```

If EDA elevation has poor sensitivity for craving, normal EDA should not be used to “exclude craving.”

---

# 21. Evidence accumulation and decay

Evidence can accumulate across:

- independent modality families;
- temporal persistence;
- recurrence;
- corroborating clinical observations;
- repeated patient-specific episodes.

It can also decay.

## 21.1 Event-scoped evidence

Every evidence item carries:

```yaml
observed_at:
valid_until:
episode_id:
context_id:
treatment_state:
```

## 21.2 Invalidation/decay triggers

A claim should be recomputed when:

- episode resolves;
- treatment or medication changes;
- alcohol use/abstinence state changes;
- a new confound appears;
- context changes;
- contradictory evidence arrives;
- evidence exceeds its clinically justified time horizon.

Avoid generic exponential decay unless the decay constant itself has clinical justification. Prefer event- and state-dependent validity windows.

---

# 22. Clinical corroboration

Physiology can be:

```text
supportive evidence
```

without being:

```text
sufficient evidence
```

Clinical corroboration sources may include:

- CIWA-Ar or another validated withdrawal instrument where appropriate;
- mMINDS in settings where its use is supported;
- clinician examination;
- recent alcohol-use/cessation history;
- PAWSS-style history/risk information for complicated withdrawal risk;
- BAC/BrAC for intoxication-related interpretation;
- subjective craving/EMA for craving target labels;
- medication/treatment record;
- sleep diary/polysomnography/validated sleep scale where relevant.

The authority engine should specify per claim whether corroboration is:

```text
NOT_REQUIRED
SUPPORTIVE
REQUIRED_FOR_A5
REQUIRED_FOR_A6
```

For example, physiology alone may support A3 autonomic activation. A4 withdrawal compatibility may require known withdrawal context. A6 reassessment alert may require a validated clinical protocol, worsening clinical score, or outcome-linked rule.

---

# 23. Established withdrawal instruments: how they should be used

CIWA-Ar was developed as a 10-item clinical scale for quantifying alcohol-withdrawal severity [21]. It is a **clinical reference/corroboration instrument**, not a target for replacing all judgment with wearable autonomic data.

An integrative review of mMINDS identified nine relevant articles and favored its use in critical-care contexts, while also illustrating that scale performance and workflow applicability are setting dependent [22].

PAWSS is different: it is intended to predict risk of **complicated** alcohol withdrawal based largely on history/clinical risk factors rather than continuous wearable physiology. The initial development/pilot study was followed by prospective validation in 403 medically ill hospitalized patients; at cutoff ≥4, the validation study reported sensitivity 93.1%, specificity 99.5%, PPV 93.1% and NPV 99.5% [23,24]. These results demonstrate the importance of **pre-test clinical information**, not that the wearable system should reproduce PAWSS from physiology.

Most importantly, Roberts et al. (2026) compared CIWA-Ar management with scales incorporating vital signs across 19 VHA hospitals. Patients managed with vital-sign-containing scales had longer treatment duration, while complicated-withdrawal rates were not improved [25]. The conclusion for this architecture is not “vital signs are useless.” It is:

> **Objectivity and physiological validity do not by themselves establish incremental clinical utility.**

This directly motivates the A5→A6/A7 clinical-utility boundary.

---

# 24. AUD-specific evidence review and authority ceilings

This section grades **claim maturity**, not the biological importance of a signal. A low grade means “do not make a strong individual clinical claim from this evidence,” not “the physiology is irrelevant.”

## 24.1 Summary table

| AUD target | Main physiological evidence | Current evidence type | Individual-level validation | Specificity | Clinical corroboration | Current maximum justified physiological-system statement |
|---|---|---|---|---|---|---|
| **Withdrawal progression** | HR/HRV/baroreflex/autonomic hyperactivity; vital signs | group/state association; some within-state correlation | limited for wearable multimodal diagnosis | low–moderate; autonomic activation is nonspecific | **required for strong AUD claim** | A4: “compatible with withdrawal-related autonomic hyperactivity”; no physiology-only diagnosis |
| **Craving / cue reactivity** | HR, EDA/sweat activity, skin temperature, HRV cue reactivity | laboratory meta-analytic association; heterogeneous naturalistic within-person evidence | weak/nascent | low; stress/arousal overlap substantial | self-report/EMA often required as reference | A4 at most; often A3 in naturalistic monitoring |
| **Intoxication / impairment** | HR, EDA, skin temperature, accelerometry | early individual-level discrimination in controlled settings | preliminary/internal subject-wise evidence | moderate for trained target in controlled lab; generalization uncertain | BrAC/BAC/reference test required for validation | A4/research A5 only after external calibration; no “intoxicated” diagnosis from current wearable model alone |
| **Autonomic dysregulation** | reduced HRV/vagal activity in AUD | systematic review/meta-analysis, high heterogeneity | largely group-level | low for AUD cause | not necessarily required for physiological statement; required for AUD attribution | A3 “reduced autonomic/vagal indices”; A4 association only |
| **Treatment/recovery trajectory** | HRV increases with abstinence/time since last drink | small longitudinal/associational studies | limited | low; many factors affect HRV | treatment status/history required | A3 patient-relative recovery trend; A4 cautious AUD-recovery association |
| **Relapse-related risk** | cue-elicited HRV; sleep/insomnia; other psychophysiology | small prospective associations | limited; no robust externally calibrated wearable model found | low–moderate depending feature/context | clinical/behavioral factors strongly needed | research A4 predictive-association language; no patient-specific relapse probability |
| **Sleep/recovery deterioration** | actigraphy, sleep duration/quality; insomnia associations | systematic/meta-analytic group evidence; prospective relapse associations | sleep state can be measured; relapse inference limited | sleep disturbance specific to sleep, not AUD/relapse cause | clinical context required for AUD/relapse claim | A3 sleep/recovery deterioration; A4 “associated with poorer AUD outcomes” only |

The table is intentionally conservative. Stronger authority can be earned later by claim-specific external/prospective validation.

---

## 24.2 Withdrawal progression

### Evidence

Acute withdrawal is physiologically compatible with autonomic hyperactivity. An early HRV study comparing 31 people with alcoholism with healthy volunteers found increased HR and decreased HRV during withdrawal relative to later measurements [27]. Bär et al. examined autonomic measures including HRV and baroreflex sensitivity in acute withdrawal and abstinence; the study supports autonomic dysregulation during withdrawal, but it is small and not a diagnostic-accuracy validation of a wearable system [28].

Clinical withdrawal severity is ordinarily established through a broader clinical assessment rather than HR or HRV alone. CIWA-Ar [21], mMINDS-related evidence [22], and PAWSS for *risk of complicated withdrawal* [23,24] illustrate distinct clinical constructs and should not be conflated.

The 2026 Roberts multicenter analysis is particularly important to authority: simply adding vital signs to withdrawal scales did not improve complicated-withdrawal outcomes and was associated with longer treatment [25].

### Evidence classification

```yaml
group_level_association: SUPPORTED
within_state_change: SUPPORTED_IN_SMALL_STUDIES
individual_discrimination: INSUFFICIENT_FOR_GENERAL_WEARABLE_CLAIM
individual_prediction: INSUFFICIENT_FOR_PHYSIOLOGY_ALONE
diagnostic_utility: NOT_ESTABLISHED_FOR_MULTIMODAL_WEARABLE_PHYSIOLOGY
monitoring_utility: PLAUSIBLE / PARTIALLY_SUPPORTED
clinical_decision_utility: NOT_ESTABLISHED_FOR_PHYSIOLOGY_ALONE
CEM: 2
```

A future model trained against prospectively adjudicated withdrawal outcomes could reach CEM-3/4. Current generic autonomic evidence should not be labeled that way.

### Maximum statement

With strong trusted physiology, low motion, appropriate withdrawal context and sustained multimodal autonomic activation:

> **“Sustained multimodal autonomic activation is present relative to the patient's baseline. In the current withdrawal context, this pattern is compatible with withdrawal-related autonomic hyperactivity, but it is not diagnostically specific.”**

If corroborated by a validated clinical withdrawal assessment that is worsening, the system may say:

> **“Physiological deterioration is concordant with worsening clinical withdrawal assessment; clinician reassessment may be appropriate according to the configured clinical protocol.”**

It still should not autonomously state:

> “Withdrawal confirmed.”

unless a future diagnostic claim is prospectively validated for that exact use.

---

## 24.3 Craving and cue reactivity

Carter and Tiffany's meta-analysis of 41 addiction cue-reactivity studies found physiological cue effects smaller than subjective craving effects; across addiction groups, cue exposure was associated with modest HR and sweat-gland changes and decreased skin temperature [29]. This supports biological/cue association, not a unique craving signature.

A 2023 systematic review of alcohol cue-exposure pharmacotherapy trials found substantial methodological heterogeneity and relatively sparse psychophysiological outcome reporting compared with craving outcomes [30].

Naturalistic evidence is more cautionary. Van Lier et al. monitored ten people with AUD for 100 days using cardiovascular/EDA wearables plus EMA. Cardiovascular associations with high self-reported craving were generally negligible-to-weak and strongly person dependent; EDA associations were also weak for most participants. The authors found no unique one-to-one physiological mapping to craving/lapse and concluded that physiology-based imminent relapse alerting was not viable from their current results [31].

### Evidence classification

```yaml
laboratory_group_association: SUPPORTED
naturalistic_within_person_association: HETEROGENEOUS
individual_discrimination: NOT_ROBUSTLY_ESTABLISHED
prediction_of_imminent_craving: NOT_ESTABLISHED
diagnostic_utility: NOT_ESTABLISHED
monitoring_utility: EXPLORATORY
clinical_decision_utility: NOT_ESTABLISHED
CEM: 1-2
```

### Maximum statement

When the physiology is abnormal:

> **“Autonomic arousal is present.”**

If it occurs during a validated alcohol-cue protocol:

> **“The autonomic response occurred during alcohol-cue exposure and is compatible with cue reactivity; physiology alone does not establish subjective craving.”**

In naturalistic use, avoid:

> “Patient is craving alcohol.”

Use self-report/EMA as the reference if the target is subjective craving. Physiological monitoring can be used to **prompt assessment**, not replace the construct without validation.

---

## 24.4 Intoxication / recovery

Kaczor et al. (2026) studied 28 healthy participants in a controlled police-academy intoxication laboratory using wrist HR, skin temperature, EDA and accelerometry with repeated BrAC and field-sobriety assessment. An XGBoost approach achieved accuracy up to 0.80 for impairment classification; the study supports feasibility but is small and controlled [34].

The study is valuable because it crosses from pure group association toward individual discrimination. It does **not** yet establish external calibration across devices, populations, drinking patterns, comorbidities, medications or real-world ambulatory settings.

### Evidence classification

```yaml
group_association: SUPPORTED
individual_discrimination: PRELIMINARY
subject_independent_validation: PRESENT_WITHIN_SMALL_STUDY
external_validation: NOT_ESTABLISHED
calibration_across_settings: NOT_ESTABLISHED
clinical_decision_utility: NOT_ESTABLISHED
CEM: 3 (preliminary, high applicability constraints)
```

### Maximum statement now

In research mode:

> **“Wearable pattern is similar to patterns associated with BrAC-defined intoxication/impairment in the development setting.”**

Do not deploy:

> “Patient is intoxicated.”

as a clinical claim until independent external validation and calibration exist for the exact target.

If a contemporaneous BrAC/BAC measurement is available, that measurement—not wearable autonomic inference—should supply the higher-specificity evidence.

---

## 24.5 Autonomic dysregulation in AUD

Cheng et al. performed a systematic review and meta-analysis; 15 studies entered quantitative analysis. AUD groups had lower parasympathetic HRV overall (pooled Hedges' g approximately -0.43), with substantial heterogeneity (I² ≈83.8%); total variability and RMSSD also differed, whereas some parasympathetic indices including HF were not consistently different [26].

This is strong evidence for a **group-level association** and weak evidence for individual AUD diagnosis. High heterogeneity is itself an authority-limiting fact.

### Evidence classification

```yaml
replicated_group_association: SUPPORTED
meta_analysis: YES
heterogeneity: HIGH
individual_diagnostic_accuracy: NOT_ESTABLISHED
within_person_monitoring: PARTIAL
AUD_specificity: LOW
CEM: 1-2
```

### Maximum statement

Patient-level:

> **“Cardiac autonomic indices are reduced relative to this patient's established baseline.”**

Literature context:

> **“Reduced HRV has been reported at group level in AUD populations, but this finding is nonspecific and should not be interpreted as diagnostic of AUD.”**

Do not use a low RMSSD to infer the presence/severity of AUD.

---

## 24.6 Treatment response / physiological recovery

Eddie et al. studied 42 treatment-engaged adults during early AUD recovery and found HRV increased with time since last drink after accounting for several covariates, while HR did not show the expected decrease [33]. Other abstinence studies similarly suggest autonomic measures can change over recovery, but these are not validated treatment-response surrogate endpoints.

### Evidence classification

```yaml
longitudinal_association: SUPPORTED_IN_SMALL_COHORTS
causal_treatment_response: NOT_ESTABLISHED
validated_surrogate_endpoint: NO
individual_action_threshold: NO
CEM: 2
```

### Maximum statement

> **“The patient's resting HRV trajectory has moved toward their recovery-period baseline over the observed interval.”**

Where current data match published recovery associations:

> **“This direction of change is compatible with psychophysiological recovery reported during abstinence, but it is not a validated measure of treatment efficacy.”**

Do not say:

> “Treatment is working.”

from HRV alone.

---

## 24.7 Relapse-related physiological risk

Garland et al. reported that cue-elicited high-frequency HRV reactivity and attentional bias predicted relapse occurrence/timing at six-month follow-up in a residential treatment sample [32]. This is prospective and clinically interesting, but it remains a relatively small, protocol-specific signal rather than a broadly externally calibrated ambulatory relapse model.

Naturalistic wearable evidence from van Lier et al. is less supportive for imminent lapse detection [31]. This disagreement should be preserved: laboratory cue-reactivity prognostic associations are not equivalent to reliable continuous real-world relapse prediction.

Sleep is another risk-associated domain. Brower et al. found baseline insomnia associated with later relapse in a treatment cohort; among 74 followed participants, relapse occurred in 60% with baseline insomnia versus 30% without, and insomnia remained predictive after covariate adjustment [35]. This does not make insomnia a specific cause or an individually calibrated relapse alarm.

### Evidence classification

```yaml
prospective_association: SUPPORTED_FOR_SELECTED_FACTORS
external_wearable_risk_model: NOT_ESTABLISHED
calibrated_patient_probability: NOT_ESTABLISHED
prediction_horizon_generalizability: LIMITED
incremental_clinical_utility: NOT_ESTABLISHED
CEM: 2 to early-3 depending exact predictor/protocol
```

### Maximum statement

> **“The observed feature has been associated with relapse outcomes in prior studies.”**

For a current patient, absent an externally validated calibrated model:

> **“Relapse probability cannot be estimated from the current physiological evidence.”**

Do not output:

```text
relapse risk = 80%
```

without target-, horizon-, population- and calibration-specific evidence.

---

## 24.8 Sleep / recovery deterioration

A 2026 systematic review/meta-analysis of actigraphy-based sleep outcomes in SUDs included nine studies (n=1,366); for AUD, actigraphy showed lower total sleep time versus controls (mean difference about -44.7 min in the pooled analysis), while subjective sleep quality was also poorer [36].

This can support objective longitudinal sleep/recovery monitoring. The authority for **sleep deterioration** is much stronger than authority for the causal statement **AUD is worsening** or the prediction **relapse is imminent**.

### Evidence classification

```yaml
sleep_disturbance_group_association: SUPPORTED
objective_actigraphy_feasibility: SUPPORTED
individual_AUD_specificity: LOW
relapse_specific_prediction: NOT_ESTABLISHED_FROM_ACTIGRAPHY_ALONE
CEM_sleep_state: 2-3 depending validated sleep algorithm
CEM_relapse_from_sleep: 2
```

### Maximum statement

> **“Sleep duration/recovery has deteriorated relative to this patient's baseline.”**

Potential literature-context annotation:

> **“Sleep disturbance is common in AUD and has been associated with poorer outcomes, but the current sleep change is not specific for AUD deterioration or relapse.”**

---

# 25. Detailed study-extraction table

The table below extracts the attributes most relevant to **authority**. “NR” means not reported in the source material reviewed for this document; it must not be silently invented.

| Study | Population / n | Setting / target | Inputs | Reference standard | Main performance/result | Validation | Authority implication |
|---|---:|---|---|---|---|---|---|
| Sullivan et al. 1989 [21] | withdrawal patients; original scale validation | clinical withdrawal severity | 10 CIWA-Ar clinical items | clinical withdrawal assessment | validity/reliability reported in original development | original validation | useful corroboration; not wearable physiology ground truth for all settings |
| Maldonado et al. 2015 [24] | 403 medically ill inpatients | complicated AWS risk | PAWSS clinical/history items | daily CIWA-Ar + clinical monitoring for AWS | Se 93.1%, Sp 99.5%, PPV 93.1%, NPV 99.5% at cutoff 4 | prospective single-system validation | strong evidence that pre-test clinical/history data can have high predictive value; not a physiology model |
| Roberts et al. 2026 [25] | 588 hospital patients across 19 VHA hospitals (461 CIWA-Ar; 127 scales with vital signs) | management outcomes | withdrawal scales ± vital signs | complicated withdrawal / treatment duration | vital-sign scales: longer treatment; no difference in complicated withdrawal (adjusted OR 1.00, 95% CI 0.41–2.46) | retrospective multicenter cohort | objective vital signs do not automatically add clinical utility |
| Cheng et al. 2019 [26] | 15 studies in quantitative meta-analysis | AUD vs controls/autonomic function | HRV | AUD group status | pooled parasympathetic HRV g ≈ -0.43; high heterogeneity I²≈83.8% | meta-analysis of mostly group studies | group association; cannot infer individual AUD diagnosis |
| Weise et al. 1985 [27] | 31 alcoholics + 106 healthy volunteers | withdrawal/abstinence | HR, HRV/MMA | clinical withdrawal/abstinence state | withdrawal HR higher and variability lower than later measures | comparative repeated observations | supports withdrawal physiology, not diagnostic specificity |
| Bär et al. 2006 [28] | 20 acute withdrawal patients analyzed; abstinent/control comparisons | acute withdrawal/autonomic regulation | HRV, BPV, BRS, ICG | clinical withdrawal scales/state | autonomic/BRS abnormalities in acute withdrawal | small clinical study | supports mechanism/state association; low diagnostic authority |
| Carter & Tiffany 1999 [29] | 41 cue-reactivity studies across addictions | cue reactivity | HR, sweat activity, skin temp, self-report | cue vs neutral condition / craving | physiological effects modest; craving effect larger | meta-analysis | physiology is cue associated but not equivalent to craving |
| van Lier et al. 2022 [31] | 10 AUD patients, 100 days | naturalistic craving/lapse | wearable cardiovascular + EDA + context/EMA | self-reported craving/lapse | physiology-craving association generally negligible-to-weak and person dependent | intensive longitudinal case design | strong caution against naturalistic physiology-only craving/relapse alerts |
| Garland et al. 2012 [32] | alcohol-dependent residential-treatment participants | six-month relapse | cue HF-HRV + attentional bias | relapse follow-up | predictors associated with occurrence/timing of relapse | prospective cohort in protocol-specific setting | promising prognostic association; insufficient for calibrated ambulatory probability |
| Eddie et al. 2023 [33] | 42 treatment-engaged adults | early recovery | HRV/HR | time since last drink + AUD clinical covariates | HRV increased with time since last drink; HR did not | observational cross-/longitudinal recovery association | trend monitoring candidate, not validated treatment surrogate |
| Kaczor et al. 2026 [34] | 28 healthy adults | controlled intoxication lab | wrist HR, temp, EDA, accel | BrAC + field-sobriety testing | ML accuracy up to 0.80 | small development study with within-study subject-wise evaluation | feasibility/early discrimination; needs external validation/calibration |
| Brower et al. 2001 [35] | 172 treatment patients; 74 followed | insomnia and relapse | sleep questionnaire + PSG | treatment follow-up relapse | 60% relapse with baseline insomnia vs 30% without in followed cohort | prospective follow-up | sleep is prognostic context, not a specific relapse detector |
| Paredes Naveda et al. 2026 [36] | 9 studies; n=1,366 | SUD sleep, including AUD | actigraphy + subjective sleep | group/clinical diagnosis | AUD TST pooled MD ≈ -44.7 min; poor subjective sleep | systematic review/meta-analysis | validates sleep disturbance association; not AUD-specific causal inference |

### Study-extraction rule for future papers

For every new study inserted into the evidence registry, require:

```yaml
study_id:
population:
sample_size:
setting:
target_state:
target_definition:
reference_standard:
physiological_inputs:
model_or_features:
sensitivity:
specificity:
auroc:
ppv:
npv:
calibration:
prediction_horizon:
validation_split:
patient_level_split: true/false
external_validation:
prospective_or_retrospective:
clinical_outcome:
incremental_value_over_standard_care:
risk_of_bias:
applicability:
limitations:
```

If a field is absent, store `NOT_REPORTED`; do not infer it.

---

# 26. Evidence grade by AUD target

Use the following **current registry defaults**, which can be upgraded only by new evidence review.

```yaml
withdrawal_related_autonomic_activation:
  CEM: 2
  specificity: LOW_TO_MODERATE
  max_authority_without_clinical_corroboration: A4
  probability_allowed: false

craving_from_ambulatory_physiology:
  CEM: 1-2
  specificity: LOW
  max_authority: A3
  max_authority_in_controlled_cue_context: A4
  probability_allowed: false

intoxication_from_wearable_multimodal_model:
  CEM: 3_PRELIMINARY
  specificity: MODERATE_WITHIN_DEVELOPMENT_PROTOCOL
  max_authority_current_deployment: A4
  probability_allowed: false
  upgrade_requirement: independent external validation + calibration

autonomic_dysregulation_in_AUD:
  CEM: 1-2
  specificity: LOW
  max_authority_patient_level: A3
  literature_association_annotation: A4

recovery_trajectory_from_HRV:
  CEM: 2
  specificity: LOW
  max_authority: A3
  literature_compatibility: A4

relapse_risk_from_physiology:
  CEM: 2_to_early_3
  specificity: LOW_TO_MODERATE_PROTOCOL_DEPENDENT
  max_authority: A4
  numeric_probability_allowed: false

sleep_recovery_deterioration:
  CEM_for_sleep_state: 2_to_3
  max_authority_for_sleep: A3
  CEM_for_relapse_inference: 2
  max_authority_for_relapse: A4_ASSOCIATION_ONLY
```

These are **document-level recommendations based on the cited evidence**, not regulatory classifications.

---

# 27. Incremental value: physiology must beat the information already available

A new physiological signal is clinically useful only if it adds value beyond available clinical data.

For each proposed higher-level AUD model compare:

```text
baseline model:
    history + setting + clinical scale + treatment status

vs

augmented model:
    baseline model + multimodal physiology
```

Evaluate:

- discrimination (e.g., AUROC);
- calibration;
- sensitivity at clinically relevant specificity, and vice versa;
- Brier/proper scoring rules;
- net benefit with decision-curve analysis [6];
- alert burden;
- subgroup performance;
- prospective workflow outcomes.

Net reclassification improvement can be exploratory, but should not substitute for calibration and decision utility.

The Roberts 2026 result [25] is a concrete warning that “contains more objective physiological information” does not guarantee better outcomes.

---

# 28. Decision-curve analysis and consequence-sensitive thresholds

A high AUROC does not tell the system when to alert. Decision-curve analysis evaluates net benefit across threshold probabilities by encoding the relative consequences of false-positive and false-negative decisions [6].

Use DCA only when:

- the target outcome is clinically defined;
- a valid prediction probability exists;
- the intended action is defined;
- the validation cohort is representative enough.

For a review alert, define:

```text
threshold probability = point at which
expected harm of missing the target event
is judged equivalent to
expected harm/burden of unnecessary review
```

The threshold must be determined with clinical stakeholders and validated; do not derive it from model accuracy alone.

Different actions legitimately have different thresholds:

```text
background display     lower evidence threshold
trend highlight        ↓
attention flag         ↓
clinician notification ↓
high-priority review   higher consequence threshold
treatment action       highest evidence + utility requirement
```

---

# 29. Alert and surfacing tiers

Authority and alert severity are related but not identical.

## Tier S0 — store only / background

Use for valid low-consequence measurements or weak evidence.

```text
No interruption.
Visible in trend/history.
```

## Tier S1 — descriptive trend highlight

Example:

> “Resting HR and EDA remain above personal baseline.”

Requires A2/A3, but no AUD attribution.

## Tier S2 — clinician attention flag

Example:

> “Persistent multimodal autonomic activation is present; current context makes withdrawal-related activation a plausible contributor, but the pattern is nonspecific.”

Requires A3/A4 and enough persistence/relevance to justify attention.

## Tier S3 — clinician review notification

Example:

> “Persistent physiological deterioration plus worsening validated withdrawal assessment meets the configured review criterion. Clinical reassessment may be appropriate.”

Requires claim-specific A6 evidence/protocol.

## Tier S4 — urgent escalation

Only when an independent validated clinical criterion/protocol authorizes it. Physiology alone should not create an emergency AUD diagnostic claim under the current evidence base.

---

# 30. Alert fatigue, suppression, and cooldown

Truth is not the same as actionability. Drew et al. observed 2.56 million monitor alarms over 31 days in 461 ICU patients; among annotated arrhythmia alarms, 88.8% were false positives, and many true alarms were nevertheless non-actionable [19]. A 2026 systematic review of alert-fatigue measurement found major inconsistency in how alert fatigue itself is operationalized, reinforcing the need to measure alert burden and responses explicitly [20].

## 30.1 Episode-based alert object

Do not issue one alert per processing window.

```yaml
episode_alert:
  episode_id:
  opened_at:
  current_authority:
  current_surface_tier:
  peak_authority:
  last_material_change:
  acknowledgement:
  status: OPEN | RESOLVING | CLOSED
```

## 30.2 Repeated evidence rule

```text
same episode + no meaningful change
→ update evidence internally
→ no new alert

same episode + crosses validated escalation boundary
→ update existing alert and escalate

episode resolves
→ close after validated recovery criteria

new independent episode after refractory/recovery period
→ new episode
```

## 30.3 What counts as material change?

- new independent modality becomes concordant;
- clinically meaningful worsening;
- clinical corroboration added;
- authority class changes;
- major contradictory evidence appears;
- applicability/confound status changes;
- urgent validated criterion becomes satisfied.

Cooldown periods must be episode-/protocol-specific rather than arbitrary.

---

# 31. Escalation and de-escalation

## 31.1 Escalation factors

Authority may increase when:

- the event persists beyond the duration validated for the target;
- independent physiological families agree;
- deviation from mature personal baseline grows;
- the episode recurs in a target-consistent pattern;
- clinical context becomes more supportive;
- a validated clinical scale corroborates;
- objective reference evidence (e.g., BrAC for an intoxication target) becomes available.

Crucially, these factors cannot exceed the claim's authority ceiling.

```text
more persistence
cannot upgrade CEM-1 group association
into diagnostic CEM-4 probability
```

## 31.2 De-escalation factors

Decrease authority when:

- physiology normalizes;
- a confound becomes plausible;
- context becomes uncertain;
- a previously available modality fails;
- contradictory clinical evidence emerges;
- model enters OOD state;
- episode no longer meets persistence;
- treatment/context changes invalidate old evidence.

Store de-escalation reasons in the audit trail.

---

# 32. Clinical significance versus statistical abnormality

A personal z-score answers:

> “How unusual is this relative to this patient's baseline?”

It does not answer:

> “Is this clinically important?”

Therefore:

```text
|z| = 3
```

must not automatically create an alert.

A feature needs a second link:

```text
patient-relative abnormality
    ↓
validated relation to clinically meaningful target/outcome
```

Where a minimal clinically important difference (MCID), validated severity band, or clinically meaningful change threshold exists for the actual measure, it can be registered. For the physiological AUD markers reviewed here, no universal wearable MCID should be invented.

---

# 33. Risk communication and prediction horizons

Every predictive claim must include its horizon.

Correct pattern:

```text
"Model-estimated probability of complicated withdrawal
within the next 24 hours..."
```

Incorrect:

```text
"withdrawal risk = 72%"
```

For relapse:

```text
six-month relapse association
```

cannot be translated into:

```text
risk in the next 2 hours
```

without new validation.

The horizon is part of the model identity and applicability domain.

---

# 34. Clinical action boundary

Separate five stages:

```text
OBSERVATION
    ↓
PHYSIOLOGICAL INTERPRETATION
    ↓
AUD COMPATIBILITY / RISK INTERPRETATION
    ↓
CLINICIAN REVIEW RECOMMENDATION
    ↓
CLINICAL ACTION
```

A machine may be permitted to reach an earlier stage while later stages remain human/clinical-protocol responsibilities.

Recommended current division:

| Stage | Machine role |
|---|---|
| Observation | generate if upstream quality passes |
| Physiological interpretation | generate with physiological evidence |
| AUD compatibility | generate cautiously when context + literature mapping support it |
| Calibrated AUD probability | only for CEM-4 claim-specific models |
| Review recommendation | only with validated consequence-sensitive protocol |
| Treatment/diagnosis | clinician-authorized / validated protocol; not generated from current physiology alone |

FDA CDS guidance also emphasizes the clinician's ability to independently review the basis of recommendations [2]. This architecture intentionally preserves that basis.

---

# 35. Human-in-the-loop and clinician feedback

## 35.1 Clinician confirmation

Require clinician review before:

- accepting an AUD-specific high-priority interpretation as a clinical conclusion;
- urgent escalation not independently defined by an established clinical criterion;
- treatment recommendation;
- model outputs outside ordinary validated conditions.

## 35.2 Feedback states

Clinician feedback can be captured as:

```text
CONFIRMED
REJECTED
PARTIALLY_SUPPORTED
UNRESOLVED
NOT_REVIEWED
```

plus an optional reason.

Do **not** silently use feedback for online retraining. It can feed a curated validation/retraining dataset after governance, label-quality review, versioning and prospective evaluation.

---

# 36. Explainability: evidence, not feature-importance theater

Every surfaced clinical interpretation should answer:

```text
What was observed?
Why does the system think the physiological state is real?
Why is an AUD process being mentioned?
What evidence contradicts it?
What required evidence is missing?
What population was the mapping validated in?
What level of statement is allowed?
Why was the clinician notified?
```

Recommended fields:

```yaml
supporting_evidence:
contradictory_evidence:
missing_evidence:
alternative_explanations:
clinical_evidence_grade:
reference_standard_quality:
applicability_status:
calibration_status:
specificity:
authority_level:
authority_limiter:
```

Generic SHAP/feature importance can be supplementary for a validated ML model, but it is not a substitute for the causal/provenance chain.

---

# 37. Counterfactual explanation

Counterfactuals are useful only when they express actual gating logic.

Safe examples:

```text
"Authority is capped at A4 because this model lacks independent external validation."

"Clinical-review authority would become eligible if the configured validated withdrawal scale also crossed its protocol threshold."

"Withdrawal attribution is currently blocked because recent cessation timing is unknown."
```

Avoid speculative claims:

```text
"Confidence would be 92% if ECG were available"
```

unless the model truly defines and validates that counterfactual.

---

# 38. Authority matrix

| Evidence condition | Max output | Example |
|---|---|---|
| valid measurement only | A1 | “HR=92 bpm.” |
| valid personal baseline deviation | A2 | “Resting HR is +3.2 SD relative to baseline.” |
| robust multimodal physiological state, confounds addressed | A3 | “Sustained autonomic activation detected.” |
| target has AUD association + relevant context, specificity limited | A4 | “Compatible with withdrawal-related autonomic activation; nonspecific.” |
| externally validated, calibrated model in-domain | A5 | defined probability/risk with horizon and uncertainty |
| A5 + demonstrated review threshold/net benefit or validated clinical corroboration protocol | A6 | “Clinical reassessment may be warranted.” |
| prospective clinical utility/action protocol | A7 | action recommendation within exact validated scope |
| target outside domain / critical evidence missing | A0 for that target | “AUD-specific interpretation not supported.” |

---

# 39. Validation-strength-to-authority matrix

| Validation status | What it demonstrates | Maximum default authority |
|---|---|---|
| mechanistic plausibility only | biologically possible | no AUD patient-level claim |
| replicated group association | groups differ / state associated | association/compatibility annotation only |
| within-person longitudinal association | marker tracks state for some individuals | cautious compatibility/monitoring |
| internal individual model with subject-independent split | preliminary discrimination/prediction | cautious research CDS; no broad probability claim |
| external validation + calibration | generalization to specified external cohort | A5 possible in matched domain |
| prospective workflow validation | live safety/performance/human factors | A6 possible |
| demonstrated clinical utility/net benefit | action improves decisions/outcomes sufficiently | A7 possible within protocol |

Internal cross-validation on windows from the same participants is **not** equivalent to subject-independent validation. Patient leakage should lower evidence grade sharply.

---

# 40. Allowed and forbidden wording

## 40.1 Preferred vocabulary

### Descriptive

> “HR and EDA are elevated relative to the personal resting baseline.”

### Physiological

> “Multimodal evidence supports sustained autonomic activation.”

### AUD compatibility

> “In the current withdrawal context, the pattern is compatible with withdrawal-related autonomic hyperactivity, but is not diagnostically specific.”

### Mixed evidence

> “Autonomic activation is present, but AUD-specific interpretation is uncertain because HRV is not concordant and recent activity may contribute.”

### Missing evidence

> “Physiological activation is present; withdrawal-specific interpretation is not supported because recent cessation timing and clinical withdrawal assessment are unavailable.”

### Probabilistic, only when validated

> “In the model's validated population, the calibrated estimated probability of **[exact target]** within **[horizon]** is **X%** (uncertainty: ...).”

### Review-oriented

> “Persistent deterioration plus clinical corroboration meets the configured review criterion; clinical reassessment may be appropriate.”

## 40.2 Forbidden by default

Do not produce from current physiology alone:

```text
"Patient has alcohol withdrawal."
"Withdrawal confirmed."
"Patient is craving alcohol."
"Relapse will occur."
"Patient will relapse soon."
"Medication is required."
"Patient is intoxicated."
"Treatment is failing."
"Emergency withdrawal detected."
```

The prohibition can be relaxed only for a future claim that receives adequate claim-specific validation and governance.

---

# 41. Proposed machine-readable model

Use a **claim object**, not one global interpretation object.

```yaml
clinical_interpretation:
  interpretation_id: "..."
  patient_episode_id: "..."
  generated_at: "..."

  claim:
    claim_id: "withdrawal_related_autonomic_activation"
    target_definition: "..."
    claim_class: "AUD_COMPATIBILITY"
    prediction_horizon: null

  observation:
    physiological_state: "sustained_multimodal_autonomic_activation"
    onset: "..."
    duration_minutes: 42
    personal_deviation:
      HR_z: 3.2
      RMSSD_z: -2.6

  evidence_state:
    measurement_validity: HIGH
    feature_validity: HIGH
    confound_resolution: HIGH
    context_sufficiency: HIGH
    personalization_maturity: HIGH
    temporal_support: HIGH

    multimodal_support:
      status: HIGH
      independent_families:
        - cardiac
        - electrodermal
      correlated_features:
        cardiac:
          - HR
          - RMSSD

    aud_specificity:
      status: MODERATE
      empirical_likelihood_ratio: null
      note: "autonomic activation is not specific for withdrawal"

    clinical_evidence:
      CEM: 2
      risk_of_bias: SOME_CONCERNS
      reference_standard_quality: MODERATE
      external_validation: false
      clinical_utility_demonstrated: false

    applicability:
      status: IN_DOMAIN_FOR_ASSOCIATION
      model_probability_domain: NOT_APPLICABLE

    calibration:
      status: NOT_APPLICABLE
      numeric_probability_allowed: false

  corroboration:
    alcohol_cessation_context: PRESENT
    withdrawal_scale: MISSING
    clinician_exam: MISSING

  uncertainty:
    measurement: LOW
    physiology: LOW
    target_ambiguity_aleatoric: MODERATE_TO_HIGH
    model_epistemic: MODERATE
    applicability: LOW

  supporting_evidence:
    - code: HR_PERSONAL_ELEVATION
      value: "+3.2 SD"
    - code: RMSSD_PERSONAL_SUPPRESSION
      value: "-2.6 SD"
    - code: EDA_ELEVATION
    - code: LOW_MOTION
    - code: REST_CONTEXT
    - code: SUSTAINED_EVENT
      value: "42 min"

  contradictory_evidence: []

  missing_evidence:
    - VALIDATED_WITHDRAWAL_SCALE
    - CLINICIAN_ASSESSMENT

  authority:
    level: A4
    statement_class: COMPATIBILITY
    ceiling: A4
    limiting_factors:
      - AUD_SPECIFICITY_LIMITED
      - EXTERNAL_DIAGNOSTIC_VALIDATION_ABSENT
      - CLINICAL_CORROBORATION_MISSING

  allowed_statement:
    text: >
      Sustained multimodal autonomic activation is present relative to the
      patient's personal resting baseline. Motion and activity are unlikely
      explanations. In the current clinical context, the pattern is compatible
      with withdrawal-related autonomic hyperactivity, but is not diagnostically
      specific. Correlation with clinical withdrawal assessment is recommended.

  surfacing:
    tier: CLINICIAN_ATTENTION
    new_interruptive_alert: false
    episode_update: true

  abstention:
    required_for_higher_claims: true
    reason_codes:
      - PROBABILITY_NOT_VALIDATED
```

---

# 42. Authority reason codes

Use compact, auditable reason codes.

## Positive/support

```text
HIGH_SIGNAL_CONFIDENCE
HIGH_FEATURE_CONFIDENCE
CONFOUND_UNLIKELY
CONTEXT_SUFFICIENT
MATURE_PERSONAL_BASELINE
MULTIMODAL_SUPPORT
INDEPENDENT_MODALITY_SUPPORT
SUSTAINED_EVENT
RECURRING_EVENT
CLINICAL_CORROBORATION_PRESENT
EXTERNAL_VALIDATION_PRESENT
CALIBRATION_ACCEPTABLE
IN_VALIDATED_DOMAIN
```

## Limiters

```text
AUD_SPECIFICITY_LIMITED
GROUP_ASSOCIATION_ONLY
INDIVIDUAL_VALIDATION_LIMITED
EXTERNAL_VALIDATION_ABSENT
CALIBRATION_NOT_ESTABLISHED
CLINICAL_CORROBORATION_MISSING
REFERENCE_STANDARD_WEAK
MISSING_REQUIRED_MODALITY
UNRESOLVED_CONFOUND
CONTEXT_INCOMPLETE
PERSONAL_BASELINE_IMMATURE
CONFLICTING_EVIDENCE
OUT_OF_DOMAIN
OUT_OF_DISTRIBUTION
PREDICTION_HORIZON_UNSUPPORTED
CLINICAL_UTILITY_NOT_ESTABLISHED
INSUFFICIENT_EVIDENCE
```

---

# 43. Final authority algorithm

```text
INPUT
    candidate claim C
    upstream evidence graph G
    current patient/context metadata P
    evidence-registry entry R for C
    available clinical corroboration H

1. BUILD CLAIM-SPECIFIC EVIDENCE VIEW
    Retrieve only nodes relevant to C.
    Preserve source, timestamps, uncertainty, and dependency groups.

2. VERIFY REQUIRED UPSTREAM EVIDENCE
    Remove invalid measurements/features.
    If a claim-required input is invalid:
        return ABSTAIN for C or cap at lower claim.

3. CHECK DOMINANT CONFOUNDS AND ESSENTIAL CONTEXT
    If unresolved confound can explain target pattern:
        cap AUD attribution.
    If target-essential context is missing:
        cap/abstain according to registry.

4. DETERMINE PHYSIOLOGICAL-STATE SUPPORT
    Aggregate evidence within dependency groups.
    Evaluate independent modality families.
    Record supporting and contradictory evidence.

5. APPLY PERSONAL AND TEMPORAL EVIDENCE
    Determine magnitude vs mature personal baseline.
    Determine persistence, recurrence, trend, recovery.
    Do not treat statistical abnormality as clinical significance automatically.

6. RETRIEVE CLAIM EVIDENCE MATURITY
    CEM, risk of bias, reference-standard grade,
    replication, individual-validation status,
    external/prospective validation, clinical utility.

7. CHECK APPLICABILITY DOMAIN
    Compare patient, setting, sensor pipeline, treatment state,
    exclusions, target definition and prediction horizon.
    If critical mismatch:
        OUT_OF_DOMAIN → abstain from model-based claim.

8. DETERMINE TARGET SPECIFICITY
    If validated LR/joint diagnostic model exists:
        use it under intended conditions.
    Else:
        assign qualitative specificity supported by literature.
    Never infer specificity from physiological confidence.

9. HANDLE PRIOR PROBABILITY
    Use explicit clinical/history prior inputs.
    If validated LR exists, Bayesian update may be used.
    Otherwise keep prior/context qualitative.

10. CHECK CALIBRATION
    If a numeric probability is to be surfaced:
        require externally appropriate calibration evidence.
    Else:
        prohibit numeric probability.

11. HANDLE UNCERTAINTY / SELECTIVE PREDICTION
    Evaluate epistemic and aleatoric uncertainty.
    Apply reject/abstention policy.
    If conformal set used, report set and coverage conditions.

12. HANDLE MISSING / CONFLICTING / NEGATIVE EVIDENCE
    Preserve each category explicitly.
    Do not average contradictions away.
    Treat absence as negative only when sensitivity justifies it.

13. COMPUTE AUTHORITY CEILING
    Amax = minimum claim authority allowed by:
        upstream validity,
        specificity,
        CEM/validation,
        reference standard,
        applicability,
        calibration,
        corroboration requirement,
        clinical-utility evidence.

14. ACCUMULATE SOFT SUPPORT WITHIN CEILING
    persistence + independent modalities + personal deviation
    + corroboration can move confidence within Amax
    but never exceed Amax.

15. SELECT STATEMENT TEMPLATE
    Choose wording class corresponding to A0–A7.
    Insert exact evidence and limitations.
    Never use stronger causal/diagnostic wording than permitted.

16. SELECT SURFACING TIER
    background / trend / attention / review / urgent
    using validated consequence-sensitive policy.
    Do not equate abnormality with alert.

17. APPLY EPISODE DEDUPLICATION
    update existing episode unless a material escalation/de-escalation occurs.

18. EMIT PROVENANCE + AUDIT RECORD
    evidence IDs, model/version, registry version,
    thresholds, authority limiter, statement text,
    timestamp, clinician interactions.

OUTPUT
    claim object
    statement
    authority level
    surfacing tier
    reason codes
    support/conflict/missing evidence
```

---

# 44. Pseudocode

```python
def authorize_claim(claim, evidence_graph, patient_context, registry, corroboration):
    spec = registry[claim.type]
    ev = evidence_graph.view_for(claim)

    # 1. Required technical/physiological gates
    if not ev.required_measurements_valid(spec):
        return lower_or_abstain(claim, "REQUIRED_EVIDENCE_INVALID")

    # 2. Confound/context gates
    if ev.dominant_confound_unresolved(spec):
        spec.cap_authority("A3", "UNRESOLVED_CONFOUND")

    if not patient_context.has_required_context(spec):
        spec.cap_authority(spec.context_missing_ceiling,
                           "CONTEXT_INCOMPLETE")

    # 3. Physiological state
    phys = infer_physiological_state_dependency_aware(ev)

    # 4. Applicability
    domain = check_applicability(spec.domain, patient_context, ev)
    if domain.hard_mismatch:
        return emit_lower_physiology_or_abstain(
            phys, claim, "OUT_OF_DOMAIN"
        )

    # 5. Specificity and evidence maturity
    evidence_meta = spec.claim_evidence
    specificity = evaluate_target_specificity(
        claim, phys, patient_context, evidence_meta
    )

    ceilings = [
        ceiling_from_upstream(ev),
        ceiling_from_specificity(specificity),
        ceiling_from_CEM(evidence_meta.CEM),
        ceiling_from_reference_standard(evidence_meta.reference_standard),
        ceiling_from_domain(domain),
        ceiling_from_corroboration(corroboration, spec),
        ceiling_from_clinical_utility(evidence_meta),
    ]

    # 6. Probability is a separate privilege
    probability = None
    if claim.requests_probability:
        if not spec.externally_validated or not spec.calibration.acceptable:
            ceilings.append(("A4", "CALIBRATION_NOT_ESTABLISHED"))
        else:
            probability = spec.model.predict_calibrated(ev, patient_context)

    # 7. Uncertainty / reject
    uncertainty = decompose_uncertainty(ev, spec, domain)
    if spec.reject_policy.rejects(uncertainty):
        return emit_lower_physiology_or_abstain(
            phys, claim, "INSUFFICIENT_EVIDENCE"
        )

    # 8. Final hard ceiling
    Amax, limiting_reasons = minimum_authority(ceilings)

    # 9. Soft support can determine placement but not cross Amax
    support = dependency_aware_support(
        phys=phys,
        temporal=ev.temporal,
        personal=ev.personal,
        multimodal=ev.multimodal,
        corroboration=corroboration,
    )

    A = choose_level_at_or_below(Amax, support)

    # 10. Output
    statement = render_statement(
        authority=A,
        claim=claim,
        physiology=phys,
        probability=probability,
        support=ev.support,
        conflict=ev.conflict,
        missing=ev.missing,
        limitations=limiting_reasons,
    )

    surfacing = choose_surface_tier(A, claim, spec, ev)

    return build_claim_object(
        claim, A, statement, surfacing, ev,
        specificity, evidence_meta, domain,
        uncertainty, limiting_reasons
    )
```

The pseudocode deliberately avoids a magic `overall_confidence = weighted_sum(...)`.

---

# 45. Central example

## Input

```text
ECG quality: excellent
EDA quality: excellent
HR: +3.2 SD above personal baseline
RMSSD: -2.6 SD below baseline
EDA: elevated
motion: low
context: resting
episode duration: 42 min
multimodal confidence: high
```

## Step 1 — measurement/feature authority

Strong.

```text
A1/A2 supported
```

## Step 2 — physiological-state authority

Independent cardiac + electrodermal support, low motion, resting context, persistence:

```text
sustained autonomic activation
A3 supported
```

## Step 3 — AUD-specificity

Autonomic hyperactivity is compatible with withdrawal, but nonspecific. Current AUD physiological literature establishes association/state support more strongly than validated wearable diagnostic specificity [25,27,28].

```text
AUD specificity: moderate at best in appropriate withdrawal context
CEM: 2
```

## Step 4 — clinical corroboration

Assume:

```text
recent cessation/withdrawal context: known
validated withdrawal score: unavailable
clinician assessment: unavailable
```

## Step 5 — authority ceiling

```text
excellent physiology          → would permit A3+
appropriate AUD context       → supports A4
limited target specificity    → ceiling A4
CEM-2                         → ceiling A4
no external diagnostic model  → no A5 probability
no clinical corroboration     → no A6 review claim based solely on AUD attribution
```

## Output

```yaml
measurement_evidence: STRONG
physiological_evidence: STRONG
physiological_state: SUSTAINED_AUTONOMIC_ACTIVATION
aud_specificity: MODERATE
clinical_evidence_maturity: CEM-2
authority: A4_COMPATIBILITY
probability: NOT_ESTIMATED
```

Clinician-facing:

> **“Sustained multimodal autonomic activation is present relative to the patient's personal resting baseline. Motion and activity are unlikely explanations. In the current clinical context, the pattern is compatible with withdrawal-related autonomic hyperactivity, but is not diagnostically specific. Correlation with clinical withdrawal assessment is recommended.”**

Not permitted:

> “Patient is experiencing withdrawal.”

The key result is that better sensor quality would not change this final ceiling; the limiter is **clinical specificity/validation**, not measurement trustworthiness.

---

# 46. Model validation requirements before authority upgrades

A new model should not automatically inherit authority because its AUROC is high.

## 46.1 Development / internal validation

Minimum:

- participant-level rather than window-level split for subject-independent claims;
- no leakage of adjacent windows or baseline-derived features across train/test;
- pre-specified target definition;
- explicit reference standard;
- missingness and preprocessing included inside the validation pipeline;
- confidence intervals for discrimination;
- calibration assessment;
- relevant subgroup/error analysis.

**Maximum default:** CEM-3, not A5 deployment probability.

## 46.2 External validation

Require an independent cohort with:

- same intended clinical target;
- relevant patient spectrum;
- intended sensor configuration or a bridged/validated equivalent;
- same prediction horizon;
- outcome/reference-standard collection independent of model output;
- calibration and discrimination;
- error analysis and applicability assessment.

External validation can support CEM-4 and A5, but only for the exact validated scope.

## 46.3 Prospective clinical validation

Evaluate the live system in the intended workflow:

- availability and missingness;
- clinician interaction;
- alert burden;
- time-to-review;
- false-positive and false-negative consequences;
- human factors;
- overrides;
- performance drift;
- patient-important/workflow outcomes.

DECIDE-AI emphasizes early live clinical evaluation rather than assuming in-silico performance translates to care [16].

## 46.4 Clinical utility

A6/A7 should require evidence that the specific surfacing/action strategy offers net benefit compared with existing care. Decision-curve analysis can be part of preclinical/retrospective evaluation [6], but prospective workflow evaluation is preferable for consequential actions.

---

# 47. Internal versus external validation and patient leakage

A repeated-measures physiological dataset can produce deceptively good performance if windows from one participant occur in both training and test sets.

For claims intended to generalize to new patients:

```text
random window split
→ unacceptable as primary validation
```

Prefer:

```text
grouped participant split
leave-one-subject-out where appropriate
external cohort
prospective cohort
```

For **personalized within-patient** models, a different claim can be made, but the evaluation must match it:

```text
train on patient's historical data
predict genuinely future unseen episodes
```

and should report how much patient history is required before the model becomes eligible.

---

# 48. Endpoint and target-label quality

Do not merge targets simply because they share autonomic activation.

Examples:

```text
"stress"
"craving"
"withdrawal"
"relapse risk"
"intoxication"
"sleep deterioration"
```

must remain separate claim IDs with explicit definitions.

Each study added to the registry must store:

```yaml
target_label:
  name:
  operational_definition:
  reference_standard:
  timing:
  assessor_blinding:
```

If two studies define “craving” differently, aggregate them only if scientifically defensible. If one study predicts a cue-elicited laboratory rating and another predicts three-hour naturalistic EMA, their evidence should remain separate unless a synthesis explicitly establishes equivalence.

---

# 49. Evidence conflicts across studies

Do not vote-count papers.

When findings disagree, compare:

- population;
- severity;
- abstinence/withdrawal stage;
- sensor;
- preprocessing;
- posture/activity;
- context;
- target definition;
- reference standard;
- medication;
- sampling window;
- model;
- patient-level vs window-level validation.

Then record:

```yaml
literature_consistency:
  status: CONSISTENT | MIXED | CONFLICTING
  explanation:
```

Mixed evidence lowers evidence maturity or specificity.

The contrast between laboratory cue-reactivity findings [29,30] and highly heterogeneous naturalistic craving physiology [31] is a concrete example: both can be true in their respective settings, but the laboratory association cannot simply be exported into an ambulatory craving detector.

---

# 50. Replication and meta-analysis

Replication increases confidence in an **association**, but does not automatically establish individual diagnosis.

The HRV meta-analysis in AUD [26] provides stronger evidence that autonomic differences are real at group level, but high heterogeneity and lack of validated individual diagnostic performance still cap patient-level authority.

For each meta-analysis store:

```yaml
effect_size:
heterogeneity:
number_of_studies:
publication_bias_assessment:
population_consistency:
individual_prediction_data_present:
```

A meta-analysis of group differences usually upgrades confidence in CEM-1; it does not jump straight to CEM-4.

---

# 51. Audit trail and reproducibility

Every clinician-facing statement should be reproducible later.

Store:

```yaml
audit:
  interpretation_id:
  patient_episode_id:
  evidence_snapshot_hash:
  evidence_ids:
  model_id:
  model_version:
  preprocessing_version:
  evidence_registry_version:
  authority_policy_version:
  thresholds_version:
  statement_template_version:
  applicability_result:
  authority_level:
  authority_ceiling:
  limiter_reason_codes:
  statement_text:
  surfaced_at:
  clinician_viewed_at:
  clinician_feedback:
```

Model updates must not retroactively change historical interpretations without explicit reprocessing/version attribution.

---

# 52. Regulatory / CDS principles relevant to the design

This is not a regulatory classification document. Two external principles are nevertheless useful.

IMDRF's SaMD clinical-evaluation framework distinguishes clinical association, analytical validation, and clinical validation [1]. This directly supports having separate evidence dimensions rather than one confidence score.

FDA's January 2026 CDS guidance describes the importance, where relevant, of enabling health-care professionals to independently review the basis for a recommendation, including information about intended use/patient population, required inputs, algorithm development/validation, data relied on, and known/unknown aspects [2]. This supports the proposed provenance object and explicit limitations.

Because this platform processes physiological signals and may make clinical claims, regulatory status will depend on intended use and implementation. That determination is outside the scope of this engineering document.

---

# 53. Failure modes and mitigations

| Failure mode | Why it is wrong | Mitigation |
|---|---|---|
| High signal confidence becomes high diagnostic confidence | technical validity ≠ specificity | separate `physiology_confidence` and `aud_specificity` |
| Group difference becomes patient diagnosis | ecological/inferential leap | CEM grade + patient-level validation requirement |
| Five correlated HRV features become five votes | double counting | dependency groups / joint validated model |
| Raw softmax becomes clinical probability | often uncalibrated | probability privilege requires calibration |
| AUROC chooses alert threshold | ignores consequence | DCA / consequence-sensitive threshold |
| Same probability used in detox and general population | PPV/pre-test probability differs | setting-specific model/domain |
| Unknown context silently treated as normal | creates false certainty | `missing_evidence` + hard/soft context gates |
| OOD patient receives confident output | generalization unsupported | explicit domain + shift monitoring + abstention |
| “Not detected” becomes “ruled out” | poor sensitivity may explain absence | negative evidence only with validated sensitivity |
| Repeated weak windows become certainty | correlated temporal repetition | episode model + dependence-aware accumulation |
| Alert every minute | alert fatigue | episode deduplication/cooldown |
| Strong physiology triggers treatment suggestion | action utility not demonstrated | A6/A7 clinical-utility gate |
| Clinician feedback silently retrains model | label drift/unsafe adaptation | curated offline retraining + version validation |
| Personal z-score treated as clinical severity | statistical abnormality ≠ clinical importance | outcome/MCID link required |
| Prediction omits horizon | uninterpretable risk | target identity includes horizon |
| “Withdrawal likely” when specificity remains low | wording exceeds evidence | authority-to-language templates |
| Missing external validation ignored | internal performance overgeneralized | external-validation ceiling |
| Weak reference label hidden | model predicts proxy, not clinical target | reference-standard grade |
| Contradiction averaged away | hides uncertainty | explicit `contradictory_evidence` |

---

# 54. Minimum implementation components

The authority layer can be implemented as these services/modules:

```text
1. Evidence Adapter
   consumes outputs from upstream physiological modules

2. Evidence Graph Store
   preserves claims, dependencies, provenance and time

3. Clinical Evidence Registry
   claim-specific literature/validation metadata

4. Applicability Engine
   checks patient/setting/sensor/horizon domain

5. Authority Policy Engine
   hard gates + ceilings + soft evidence

6. Uncertainty/Abstention Engine
   model uncertainty, selective prediction, conformal if validated

7. Statement Renderer
   only renders language permitted by authority class

8. Alert Episode Manager
   persistence, dedupe, cooldown, escalation/de-escalation

9. Clinician Explanation API
   support/conflict/missing evidence and basis

10. Audit Store
    versioned reproducibility
```

## 54.1 Evidence registry schema

```yaml
claim_registry:
  claim_id:
  clinical_construct:
  operational_definition:
  prediction_horizon:
  intended_population:
  intended_setting:
  exclusions:

  required_upstream_evidence:
  optional_upstream_evidence:
  dominant_confounds:
  required_context:

  literature:
    studies: []
    CEM:
    risk_of_bias:
    reference_standard_grade:
    replication:
    literature_consistency:

  specificity:
    qualitative:
    validated_LR_plus:
    validated_LR_minus:
    diagnostic_model_id:

  model:
    model_id:
    internal_validation:
    external_validation:
    calibration:
    applicability_domain:

  corroboration:
    required_for_A4:
    required_for_A5:
    required_for_A6:

  authority_policy:
    max_without_corroboration:
    max_with_corroboration:
    probability_allowed:
    action_allowed:

  surfacing_policy:
    passive:
    attention:
    review:
    urgent:
```

---

# 55. Testing the authority layer

Unit tests should target **semantic overclaiming**, not only software correctness.

Examples:

```text
TEST: perfect sensor + low AUD specificity
EXPECT: max A4, never A5/A6 from physiology alone

TEST: high raw classifier score + no calibration
EXPECT: probability hidden; CALIBRATION_NOT_ESTABLISHED

TEST: external model + patient outside domain
EXPECT: OUT_OF_DOMAIN; model claim abstained

TEST: EDA/HR support + HRV conflict
EXPECT: contradiction retained, not silently averaged

TEST: repeated identical event windows
EXPECT: one episode, no alert spam

TEST: withdrawal context missing
EXPECT: autonomic A3 allowed, withdrawal A4 blocked/capped

TEST: relapse model validated for 6 months
QUERY: risk next 24 h
EXPECT: PREDICTION_HORIZON_UNSUPPORTED

TEST: five HRV features agree
EXPECT: one cardiac evidence family, not five independent votes

TEST: normal EDA with marker sensitivity unknown
EXPECT: no "craving ruled out"

TEST: new clinical corroboration arrives
EXPECT: authority recomputed and potentially escalated
```

---

# 56. Research/validation roadmap

## Phase 1 — non-diagnostic observational system

Implement A0-A4 only.

Goals:

- evidence graph;
- patient-relative physiological statements;
- cautious compatibility language;
- provenance;
- conflict/missing evidence;
- episode surfacing without interruptive high-priority claims.

This is compatible with the current literature strength.

## Phase 2 — claim-specific individual models

Choose one narrow target first, e.g.:

```text
worsening clinically assessed withdrawal over a defined horizon
```

not “AUD state.”

Collect:

- intended-use cohort;
- rigorous clinical reference;
- repeated multimodal physiology;
- patient-level splits;
- medication/context;
- clinical scale/history.

Develop and report according to TRIPOD+AI/PROBAST+AI principles [13,14].

## Phase 3 — external calibration

Validate on independent sites/populations. Only then consider A5 probabilities.

## Phase 4 — prospective decision support

Test whether surfacing improves clinician review or outcomes without unacceptable alert burden. Use DECIDE-AI-style live evaluation [16] and consequence-sensitive analyses [6].

## Phase 5 — action-level claims

Only after demonstrated clinical utility and an explicit clinical protocol should A7 be considered.

---

# 57. Why the framework should not use one “authority score”

A score such as:

```text
authority = 0.87
```

would be meaningful only if “0.87” had a validated target and calibration.

Instead:

```yaml
measurement_confidence: HIGH
physiological_state_confidence: HIGH
target_specificity: MODERATE
claim_evidence_maturity: CEM-2
reference_standard_quality: MODERATE
applicability: MATCHED_FOR_ASSOCIATION
numeric_probability_valid: false
overall_authority: A4_COMPATIBILITY
authority_limiter:
  - AUD_SPECIFICITY_LIMITED
  - EXTERNAL_VALIDATION_ABSENT
```

is more interpretable and prevents categories of confidence from being conflated.

---

# 58. Requirement-to-design trace

The major requested concerns map to implementation components as follows:

| Requirement family | Implemented by |
|---|---|
| evidence hierarchy / statement strength | A0–A7 authority taxonomy |
| confidence vs specificity | separate evidence-vector fields |
| evidence grade | CEM + risk-of-bias + reference standard |
| association vs prediction vs diagnosis | CEM definitions |
| provenance | claim evidence graph + audit store |
| evidence redundancy | dependency groups |
| prior probability/Bayes | explicit clinical prior; LR only if validated |
| calibration | probability privilege gate |
| uncertainty | decomposed uncertainty object |
| abstention | first-class A0 + selective/reject policies |
| conformal prediction | optional uncertainty layer, not validity source |
| OOD/applicability | explicit domain + shift monitoring |
| missing/conflicting/negative evidence | separate output arrays/states |
| evidence accumulation/decay | episode graph + validity windows |
| clinical corroboration | per-claim corroboration requirements |
| clinical scales | external corroboration nodes, not assumed ground truth everywhere |
| incremental value | baseline-vs-augmented validation + DCA |
| consequence-sensitive thresholds | surfacing tiers + DCA |
| alert fatigue | episode dedupe/cooldown |
| escalation/de-escalation | recompute claim authority on material evidence changes |
| language calibration | authority-locked statement templates |
| validation maturity | CEM-3/4/5 boundaries |
| patient-level splits | model-validation requirement |
| target/reference-standard quality | registry metadata |
| evidence conflicts/replication | registry consistency fields |
| human-in-loop | review/action authority boundaries |
| audit | versioned interpretation record |

---

# Recommended Implementation

This is the **single consolidated evidence/authority-aware clinical interpretation system to build**.

## Evidence hierarchy

Implement these claim classes:

```text
A0  ABSTAIN / interpretation unsupported
A1  reliable measurement
A2  patient-relative physiological feature
A3  physiological state
A4  AUD compatibility / association
A5  externally validated calibrated AUD probability/risk
A6  clinician-review decision support
A7  action recommendation under prospectively validated protocol
```

Do not implement A5-A7 generically. They are unlocked **per claim** by validation.

## Hard gates

For every claim, register hard gates:

1. invalid required signal/feature;
2. unresolved dominant confound;
3. missing claim-essential context;
4. no AUD clinical association;
5. outside validated population/setting/sensor/horizon;
6. numeric probability without adequate external calibration;
7. action recommendation without demonstrated clinical utility.

A hard gate caps or blocks only the affected claim; lower-level physiology can still be surfaced.

## Soft evidence

Use the following as confidence modifiers **within** the ceiling:

- optional modality missing;
- baseline maturity;
- multimodal agreement;
- personal deviation magnitude;
- episode duration;
- recurrence;
- context completeness;
- clinical corroboration.

Never let soft accumulation override a hard ceiling.

## AUD evidence grades

Use Claim Evidence Maturity:

```text
CEM-0 mechanistic only
CEM-1 replicated group association
CEM-2 within-person/longitudinal association
CEM-3 internally validated individual discrimination/prediction
CEM-4 external validation + calibration
CEM-5 prospective clinical utility
```

Always pair CEM with:

```text
risk of bias
reference-standard grade
applicability
replication
literature consistency
```

Current default maximums from the reviewed literature:

```text
withdrawal physiology        CEM-2 → A4 max without stronger model
craving physiology           CEM-1/2 → A3/A4 depending cue context
intoxication wearable model  preliminary CEM-3 → A4 deployment ceiling pending external validation
autonomic dysregulation      CEM-1/2 → A3 + association annotation
recovery HRV                 CEM-2 → A3/A4 compatibility only
relapse physiology           CEM-2/early-3 → A4; no numeric probability
sleep deterioration          CEM-2/3 for sleep state; A3; relapse inference A4 association only
```

## Specificity representation

Store separately:

```yaml
physiology_confidence:
target_specificity:
clinical_evidence_maturity:
```

Never derive `P(withdrawal)` from `P(autonomic activation)`.

Where a validated joint LR/model exists, store it. Otherwise use qualitative specificity and explicitly set:

```yaml
posterior_probability: NOT_ESTIMATED
```

## Calibration

Numeric risk/probability is a **privileged output**.

Enable it only if:

- exact target and horizon are defined;
- subject-independent validation exists;
- independent external validation exists;
- calibration curve/intercept/slope and proper score are acceptable;
- current patient/setting is in-domain;
- prevalence/prior context is appropriate.

Recalibration does not repair weak clinical association or bad labels.

## Applicability domain

Every model/claim stores:

```text
population
setting
target
horizon
sensor configuration
required inputs
exclusions
treatment/medication constraints
reference standard
```

Check explicit domain first, then distribution shift, then model uncertainty. A critical domain mismatch forces abstention for the model-based claim.

## Abstention

Implement these states:

```text
INSUFFICIENT_EVIDENCE
AMBIGUOUS
MIXED_EVIDENCE
OUT_OF_DOMAIN
OUT_OF_DISTRIBUTION
CLINICAL_INTERPRETATION_NOT_SUPPORTED
PROBABILITY_NOT_VALIDATED
```

The system must be allowed to provide a lower-level physiological statement while abstaining from AUD attribution.

## Authority levels

The final authority is:

```text
Amax =
minimum ceiling imposed by
upstream validity,
specificity,
CEM/validation,
reference standard,
applicability,
calibration,
required corroboration,
clinical utility
```

Then cumulative support determines the strongest statement **at or below Amax**.

This hybrid ceiling+cumulative design is preferred over either a pure weakest-link score or a weighted average.

## Statement templates

A1:

> “Resting HR is 92 bpm.”

A2:

> “Resting HR is elevated relative to the patient's established baseline.”

A3:

> “Multimodal evidence supports sustained autonomic activation.”

A4:

> “In the current clinical context, the pattern is compatible with withdrawal-related autonomic hyperactivity, but is not diagnostically specific.”

A5, only validated:

> “The externally validated model estimates a **[X%]** probability of **[exact outcome]** over **[horizon]** in the current validated population; uncertainty/calibration information: **[...]**.”

A6:

> “Persistent deterioration plus clinical corroboration meets the validated review criterion; clinical reassessment may be appropriate.”

A7:

Only language explicitly authorized by the validated clinical protocol and human-governance design.

## Alert/surfacing tiers

Map authority and consequence to:

```text
S0 background/store
S1 trend highlight
S2 clinician attention flag
S3 clinician review notification
S4 urgent protocol escalation
```

A4 does not automatically mean S2/S3. The event also needs clinical relevance/persistence and a validated surfacing policy.

## Clinical corroboration

For withdrawal-related A4+, require current withdrawal context. For stronger review-level claims, incorporate a validated clinical assessment/history according to the intended setting.

Physiology is ordinarily:

```text
supportive
```

until an exact claim-specific model demonstrates that it is sufficient.

For craving, retain self-report/EMA as the reference construct unless future evidence validates an objective surrogate. For intoxication, BrAC/BAC remains higher-specificity corroboration. For relapse, do not turn biomarker association into patient-specific probability.

## Escalation/de-escalation

Recompute authority whenever a **material event** occurs:

Escalate on:

- longer validated persistence;
- new independent modality support;
- larger patient-relative deviation with clinical relevance;
- recurrence;
- clinical corroboration;
- worsening validated clinical endpoint.

De-escalate on:

- recovery;
- confound discovery;
- context loss;
- sensor/input loss;
- contradiction;
- OOD;
- treatment/context change.

Never exceed the evidence ceiling simply because an episode lasts longer.

## Conflict handling

Store:

```yaml
supporting_evidence: [...]
contradictory_evidence: [...]
missing_evidence: [...]
```

If conflict is clinically important, return `MIXED_EVIDENCE` and lower/abstain at higher levels.

## Evidence provenance

Every claim edge stores:

```text
source signal/feature
time interval
quality/confidence
dependency family
personal baseline version
context/confound status
study/evidence registry entry
model/version
reference standard
validation status
```

Clinician explanations should expose this chain directly.

## Output schema

Use the claim object in Section 41.

The core required top-level fields are:

```yaml
claim:
observation:
evidence_state:
specificity:
clinical_evidence:
applicability:
calibration:
corroboration:
uncertainty:
supporting_evidence:
contradictory_evidence:
missing_evidence:
authority:
allowed_statement:
surfacing:
abstention:
audit:
```

## Final design rule

The system should ask, in order:

```text
1. Is the evidence technically trustworthy?
2. What physiological claim does it actually support?
3. How specific is that physiology for the proposed AUD target?
4. What inferential level has the AUD mapping actually achieved in research?
5. Does that evidence apply to this patient, setting, sensor and horizon?
6. Is any numeric probability externally validated and calibrated?
7. What corroborating/conflicting/missing clinical evidence exists?
8. What is the maximum statement class allowed?
9. Is the event important enough to surface or interrupt?
10. What exact evidence and limitations must accompany the statement?
```

The architecture should therefore preserve the central chain:

```text
measurement is trustworthy
≠
physiology is meaningful
≠
AUD interpretation is specific
≠
clinical conclusion is justified
≠
clinical action is beneficial
```

**Each `≠` is a separately validated evidence boundary.**

---

# References

[1] International Medical Device Regulators Forum (IMDRF), SaMD Working Group.  
**“Software as a Medical Device (SaMD): Clinical Evaluation.”** IMDRF/SaMD WG/N41FINAL:2017. 21 Sep 2017.  
URL: https://www.imdrf.org/documents/software-medical-device-samd-clinical-evaluation

[2] U.S. Food and Drug Administration.  
**“Clinical Decision Support Software: Guidance for Industry and Food and Drug Administration Staff.”** Final Guidance, January 2026 (reissued 29 Jan 2026).  
URL: https://www.fda.gov/regulatory-information/search-fda-guidance-documents/clinical-decision-support-software

[3] Van Calster B, McLernon DJ, van Smeden M, Wynants L, Steyerberg EW, et al.  
**“Calibration: the Achilles heel of predictive analytics.”** *BMC Medicine*. 2019;17:230.  
DOI: 10.1186/s12916-019-1466-7. PMID: 31842878.  
URL: https://pubmed.ncbi.nlm.nih.gov/31842878/

[4] Riley RD, Collins GS, Kirton L, Snell KIE, Ensor J, Whittle R, et al.  
**“Uncertainty of risk estimates from clinical prediction models: rationale, challenges, and approaches.”** *BMJ*. 2025;388:e080749.  
DOI: 10.1136/bmj-2024-080749.  
URL: https://www.bmj.com/content/388/bmj-2024-080749

[5] Kompa B, Snoek J, Beam AL.  
**“Second opinion needed: communicating uncertainty in medical machine learning.”** *npj Digital Medicine*. 2021;4:4.  
DOI: 10.1038/s41746-020-00367-3. PMID: 33402680.  
URL: https://www.nature.com/articles/s41746-020-00367-3

[6] Vickers AJ, Elkin EB.  
**“Decision curve analysis: a novel method for evaluating prediction models.”** *Medical Decision Making*. 2006;26(6):565-574.  
DOI: 10.1177/0272989X06295361. PMID: 17099194.  
URL: https://pubmed.ncbi.nlm.nih.gov/17099194/

[7] Deeks JJ, Altman DG.  
**“Diagnostic tests 4: likelihood ratios.”** *BMJ*. 2004;329:168-169.  
DOI: 10.1136/bmj.329.7458.168. PMID: 15258077.  
URL: https://pubmed.ncbi.nlm.nih.gov/15258077/

[8] Altman DG, Bland JM.  
**“Diagnostic tests 2: predictive values.”** *BMJ*. 1994;309:102.  
DOI: 10.1136/bmj.309.6947.102. PMID: 8038641.  
URL: https://pubmed.ncbi.nlm.nih.gov/8038641/

[9] Geifman Y, El-Yaniv R.  
**“Selective Classification for Deep Neural Networks.”** *Advances in Neural Information Processing Systems (NeurIPS)*. 2017.  
URL: https://arxiv.org/abs/1705.08500

[10] Geifman Y, El-Yaniv R.  
**“SelectiveNet: A Deep Neural Network with an Integrated Reject Option.”** *Proceedings of the 36th International Conference on Machine Learning*. PMLR 97:2151-2159, 2019.  
URL: https://proceedings.mlr.press/v97/geifman19a.html

[11] Olsson H, et al.  
**“Estimating diagnostic uncertainty in artificial intelligence assisted pathology using conformal prediction.”** *Nature Communications*. 2022;13:7761.  
DOI: 10.1038/s41467-022-34945-8. PMID: 36522311.  
URL: https://pubmed.ncbi.nlm.nih.gov/36522311/

[12] Silva GFDS, Barcellos Filho FN, Wichmann RM, da Silva Junior FC, Chiavegatto Filho ADP.  
**“Strategies for detecting and mitigating dataset shift in machine learning for health predictions: A systematic review.”** *Journal of Biomedical Informatics*. 2025;170:104902.  
DOI: 10.1016/j.jbi.2025.104902. PMID: 40876698.  
URL: https://pubmed.ncbi.nlm.nih.gov/40876698/

[13] Moons KGM, Damen JAA, Kaul T, Hooft L, Andaur Navarro C, Dhiman P, et al.  
**“PROBAST+AI: an updated quality, risk of bias, and applicability assessment tool for prediction models using regression or artificial intelligence methods.”** *BMJ*. 2025;388:e082505.  
DOI: 10.1136/bmj-2024-082505. PMID: 40127903.  
URL: https://pubmed.ncbi.nlm.nih.gov/40127903/

[14] Collins GS, Moons KGM, Dhiman P, Riley RD, Beam AL, et al.  
**“TRIPOD+AI statement: updated guidance for reporting clinical prediction models that use regression or machine learning methods.”** *BMJ*. 2024;385:e078378.  
DOI: 10.1136/bmj-2023-078378.  
URL: https://www.bmj.com/content/385/bmj-2023-078378

[15] Sounderajah V, Guni A, Liu X, Collins GS, Karthikesalingam A, et al.  
**“The STARD-AI reporting guideline for diagnostic accuracy studies using artificial intelligence.”** *Nature Medicine*. 2025;31:3283-3289.  
DOI: 10.1038/s41591-025-03953-8.  
URL: https://www.nature.com/articles/s41591-025-03953-8

[16] Vasey B, Nagendran M, Campbell B, Clifton DA, Collins GS, Denaxas S, et al.  
**“Reporting guideline for the early-stage clinical evaluation of decision support systems driven by artificial intelligence: DECIDE-AI.”** *Nature Medicine*. 2022;28:924-933.  
DOI: 10.1038/s41591-022-01772-9. PMID: 35585198.  
URL: https://pubmed.ncbi.nlm.nih.gov/35585198/

[17] Whiting PF, Rutjes AWS, Westwood ME, Mallett S, Deeks JJ, Reitsma JB, et al.  
**“QUADAS-2: a revised tool for the quality assessment of diagnostic accuracy studies.”** *Annals of Internal Medicine*. 2011;155(8):529-536.  
DOI: 10.7326/0003-4819-155-8-201110180-00009. PMID: 22007046.  
URL: https://pubmed.ncbi.nlm.nih.gov/22007046/

[18] Schünemann HJ, Oxman AD, Brozek J, Glasziou P, Jaeschke R, Vist GE, et al.  
**“GRADE: grading quality of evidence and strength of recommendations for diagnostic tests and strategies.”** *BMJ*. 2008;336:1106-1110.  
DOI: 10.1136/bmj.39500.677199.AE.  
URL: https://www.bmj.com/content/336/7653/1106

[19] Drew BJ, Harris P, Zègre-Hemsey JK, Mammone T, Schindler D, Salas-Boni R, et al.  
**“Insights into the Problem of Alarm Fatigue with Physiologic Monitor Devices: A Comprehensive Observational Study of Consecutive Intensive Care Unit Patients.”** *PLOS ONE*. 2014;9(10):e110274.  
DOI: 10.1371/journal.pone.0110274. PMID: 25338067.  
URL: https://pubmed.ncbi.nlm.nih.gov/25338067/

[20] Ray CE, Wilson GM, Hughes AM, Goedken CC, Liu EPF, Fitzpatrick MA, et al.  
**“Alert fatigue measurement in clinical decision support: a systematic review.”** *Journal of the American Medical Informatics Association*. 2026;33(8):1523-1531.  
DOI: 10.1093/jamia/ocag064. PMID: 42148822.  
URL: https://pubmed.ncbi.nlm.nih.gov/42148822/

[21] Sullivan JT, Sykora K, Schneiderman J, Naranjo CA, Sellers EM.  
**“Assessment of alcohol withdrawal: the revised clinical institute withdrawal assessment for alcohol scale (CIWA-Ar).”** *British Journal of Addiction*. 1989;84(11):1353-1357.  
DOI: 10.1111/j.1360-0443.1989.tb00737.x. PMID: 2597811.  
URL: https://pubmed.ncbi.nlm.nih.gov/2597811/

[22] Trojand T, Morgan J, Shamoun CJ.  
**“Using the Modified Minnesota Detoxification Scale to Evaluate Alcohol Withdrawal Syndrome: An Integrative Review.”** *Critical Care Nurse*. 2025;45(2):60-68.  
DOI: 10.4037/ccn2025568. PMID: 40168010.  
URL: https://pubmed.ncbi.nlm.nih.gov/40168010/

[23] Maldonado JR, Sher Y, Ashouri JF, Hills-Evans K, Swendsen H, Lolak S, Miller AC.  
**“The ‘Prediction of Alcohol Withdrawal Severity Scale’ (PAWSS): systematic literature review and pilot study of a new scale for the prediction of complicated alcohol withdrawal syndrome.”** *Alcohol*. 2014;48(4):375-390.  
DOI: 10.1016/j.alcohol.2014.01.004. PMID: 24657098.  
URL: https://pubmed.ncbi.nlm.nih.gov/24657098/

[24] Maldonado JR, Sher Y, Das S, Hills-Evans K, Frenklach A, Lolak S, Talley R, Neri E.  
**“Prospective Validation Study of the Prediction of Alcohol Withdrawal Severity Scale (PAWSS) in Medically Ill Inpatients.”** *Alcohol and Alcoholism*. 2015;50(5):509-518.  
DOI: 10.1093/alcalc/agv043. PMID: 25999438.  
URL: https://pubmed.ncbi.nlm.nih.gov/25999438/

[25] Roberts AE, Yarbrough P, Vargas J, Tuck M, Trubitt M, Smeraglio A, Shah MN, et al.  
**“The Association of Alcohol Withdrawal Severity Scales that Incorporate Vital Signs and Withdrawal Outcomes: A Multicenter Cohort Study.”** *Journal of Addiction Medicine*. 2026.  
DOI: 10.1097/ADM.0000000000001698. PMID: 42360115.  
URL: https://pubmed.ncbi.nlm.nih.gov/42360115/

[26] Cheng YC, Huang YC, Huang WL.  
**“Heart rate variability as a potential biomarker for alcohol use disorders: A systematic review and meta-analysis.”** *Drug and Alcohol Dependence*. 2019;204:107502.  
DOI: 10.1016/j.drugalcdep.2019.05.030. PMID: 31494439.  
URL: https://doi.org/10.1016/j.drugalcdep.2019.05.030

[27] Weise F, Müller D, Krell D, Kielstein V, Koch RD.  
**“Heart rate variability of chronic alcoholics in withdrawal and abstinence.”** *Clinical Neurology and Neurosurgery*. 1985;87(2):95-98.  
DOI: 10.1016/0303-8467(85)90104-0. PMID: 4028594.  
URL: https://pubmed.ncbi.nlm.nih.gov/4028594/

[28] Bär KJ, Boettger MK, Boettger S, Grotelüschen M, Neubauer R, Jochum T, et al.  
**“Reduced baroreflex sensitivity in acute alcohol withdrawal syndrome and in abstained alcoholics.”** *Drug and Alcohol Dependence*. 2006;85(1):66-74.  
DOI: 10.1016/j.drugalcdep.2006.03.014. PMID: 16650658.  
URL: https://pubmed.ncbi.nlm.nih.gov/16650658/

[29] Carter BL, Tiffany ST.  
**“Meta-analysis of cue-reactivity in addiction research.”** *Addiction*. 1999;94(3):327-340.  
DOI: 10.1046/j.1360-0443.1999.9433273.x. PMID: 10605857.  
URL: https://pubmed.ncbi.nlm.nih.gov/10605857/

[30] Meredith LR, Burnette EM, Nieto SJ, Du H, Donato S, Grodin EN, et al.  
**“Testing pharmacotherapies for alcohol use disorder with cue exposure paradigms: A systematic review and quantitative synthesis of human laboratory trial methodology.”** *Alcohol: Clinical and Experimental Research*. 2023;47(9):1629-1645.  
DOI: 10.1111/acer.15143. PMID: 37423771.  
URL: https://pubmed.ncbi.nlm.nih.gov/37423771/

[31] van Lier HG, Noordzij ML, Pieterse ME, Postel MG, et al.  
**“An ideographic study into physiology, alcohol craving and lapses during one hundred days of daily life monitoring.”** *Addictive Behaviors Reports*. 2022;16:100443.  
DOI: 10.1016/j.abrep.2022.100443. PMID: 35855973.  
URL: https://pubmed.ncbi.nlm.nih.gov/35855973/

[32] Garland EL, Franken IHA, Howard MO.  
**“Cue-elicited heart rate variability and attentional bias predict alcohol relapse following treatment.”** *Psychopharmacology*. 2012;222(1):17-26.  
DOI: 10.1007/s00213-011-2618-4. PMID: 22203318.  
URL: https://pubmed.ncbi.nlm.nih.gov/22203318/

[33] Eddie D, Pietrzak A, Ham J.  
**“Time Since Last Drink is Positively Associated with Heart Rate Variability in Outpatients with Alcohol Use Disorder: Further Evidence of Psychophysiological Recovery in Early Alcohol Use Disorder Recovery.”** *Applied Psychophysiology and Biofeedback*. 2023;48(4):433-437.  
DOI: 10.1007/s10484-023-09597-z. PMID: 37436518.  
URL: https://pubmed.ncbi.nlm.nih.gov/37436518/

[34] Kaczor EE, Golleru M, Carter DC, Painter OB, Kelleran KJ, Lynch JJ, Clemency BM, Cavuoto LA, Chai PR.  
**“Detecting Ethanol Intoxication and Impairment Using Wearable Biosensors.”** *Proceedings of the Annual Hawaii International Conference on System Sciences*. 2026;2026:3532-3542.  
PMID: 41550855. PMCID: PMC12805382.  
URL: https://pubmed.ncbi.nlm.nih.gov/41550855/

[35] Brower KJ, Aldrich MS, Robinson EA, Zucker RA, Greden JF.  
**“Insomnia, self-medication, and relapse to alcoholism.”** *American Journal of Psychiatry*. 2001;158(3):399-404.  
DOI: 10.1176/appi.ajp.158.3.399. PMID: 11229980.  
URL: https://pubmed.ncbi.nlm.nih.gov/11229980/

[36] Paredes Naveda AM, Oliva HNP, Pulido-Saavedra A, Ghadimi DJ, Moore JT, Morgan PT, Potenza MN, Angarita GA.  
**“Actigraphy-based sleep outcomes in substance use disorders: A systematic review and meta-analysis.”** *Sleep Medicine Reviews*. 2026;88:102297.  
DOI: 10.1016/j.smrv.2026.102297. PMID: 42030609.  
URL: https://pubmed.ncbi.nlm.nih.gov/42030609/

---

## Closing design position

The practical value of this layer is that it gives the system a way to say **less** when less is justified and **more** only after the missing inferential link has actually been validated.

For this AUD platform, the correct engineering ambition is therefore not:

```text
make every sensor vote for a diagnosis
```

but:

```text
preserve the evidentiary chain,
know which link is weak,
cap the claim at that link,
and expose exactly why.
```

That is the boundary between a trustworthy physiological monitoring system and an overconfident clinical inference system.
