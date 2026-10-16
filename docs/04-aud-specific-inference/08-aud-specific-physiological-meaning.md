# AUD-Specific Physiological Meaning Layer

**Technical research and implementation design**  
**Evidence cutoff:** 16 October 2026  
**Scope:** AUD-specific physiological interpretation only

---

## Executive summary

This document specifies the layer that answers:

> **Given a trustworthy, decomposed, confound-aware, context-corrected physiological state, what does that physiological state legitimately indicate about the patient's alcohol use disorder (AUD) / substance-use-disorder (SUD) condition?**

The layer sits after generic physiological interpretation and before clinical action:

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
AUD-SPECIFIC PHYSIOLOGICAL MEANING   ← this document
↓
clinical meaning / action
```

The central design conclusion is that there is **no defensible single mapping from a physiological variable to an AUD state**. A high heart rate is not withdrawal; elevated EDA is not craving; low HRV is not relapse. The useful mappings are conditional, temporal, multimodal, and often personalized.

The layer should therefore output **AUD-specific physiological evidence**, not diagnoses or treatment instructions. The most defensible outputs are:

1. **withdrawal-related physiological activation evidence** and a temporal trend;
2. **withdrawal tremor evidence** as a specifically measurable withdrawal component;
3. **alcohol-cue autonomic reactivity evidence** when alcohol-cue context is known;
4. **AUD-relevant stress-reactivity evidence** when a stress challenge/context is known and the interpretation is anchored to AUD-specific response history;
5. **alcohol-associated intoxication/impairment pattern evidence**, explicitly distinct from BAC estimation;
6. **chronic autonomic dysregulation burden**, primarily from longitudinal cardiac autonomic measures;
7. **autonomic recovery / abstinence progression**, primarily as within-person longitudinal change;
8. **treatment-associated physiological change**, only when compared against a defined pre-treatment state or challenge paradigm;
9. **relapse-risk-associated physiological deviation**, not a relapse prediction;
10. **nocturnal physiological recovery deterioration / alcohol-associated nocturnal autonomic suppression**.

The evidence is strongest for: (a) alcohol-related effects on HR/HRV, including dose- and time-dependent acute effects and longitudinal recovery; (b) objective withdrawal tremor; and (c) experimental/pilot individual-level intoxication classification using motion plus autonomic signals [1-8]. Cue-reactivity and relapse-risk physiology are scientifically important but substantially less ready for unrestricted ambulatory inference [9-17].

A critical negative result for system design is the 100-day ambulatory AUD study by van Lier et al.: physiology-to-craving and physiology-to-lapse relationships varied markedly between individuals, and no one-to-one mapping supported a generic imminent-lapse detector [16]. A separate 14-day wearable feasibility study similarly found no significant HRV or EDA association with alcohol-related craving [17]. Therefore, **physiology-alone craving detection and relapse alarms should not be production outputs**.

---

# 1. Layer boundary and design contract

## 1.1 Input contract

This layer assumes upstream processing has already produced context-corrected physiological features such as:

- resting / context-conditioned heart rate;
- ECG-derived RR intervals and validated HRV features;
- PPG-derived pulse rate / inter-beat intervals when reliable;
- tonic and phasic electrodermal features;
- skin-temperature level and slope;
- activity state and tremor/gait features from IMU;
- sleep/rest state;
- generic autonomic interpretation such as sympathetic activation, reduced vagal modulation, or impaired physiological recovery;
- current contextual labels where available (rest, sleep, exercise, known alcohol cue, known stressor, treatment session, early abstinence, detoxification setting, recent drinking report, etc.);
- personal-baseline statistics and recent temporal history.

This layer does **not** re-solve sensor quality, artifact rejection, generic physiological decomposition, generic motion confounding, or context recognition.

## 1.2 Output contract

The output is an **AUD-specific physiological evidence object** with:

- target process;
- evidence score;
- evidence grade;
- runtime confidence;
- supporting and contradictory features;
- temporal classification (event/state/trend/risk marker);
- baseline relationship;
- context requirements;
- missing-data penalties;
- explicit statement of what the evidence does **not** establish.

The output must never directly be:

```text
AUD_DIAGNOSIS = TRUE
RELAPSE_WILL_OCCUR = TRUE
BAC = 0.09
MEDICATION_REQUIRED = TRUE
HOSPITALIZE = TRUE
```

Those are outside this layer.

---

# 2. Evidence hierarchy used in this design

A significant group difference is not equivalent to an individual-level monitor. Each candidate mapping is graded as follows.

### Level A — Direct individual-level inference evidence

A feature or model has been evaluated for individual-level classification, prediction, severity estimation, or longitudinal monitoring against a meaningful reference. This does **not** imply clinical validation. The document labels small internally validated studies as **A-pilot**.

### Level B — Strong physiological association

The relationship is replicated, supported by systematic review/meta-analysis, or demonstrated longitudinally, but a validated individual-level decision threshold/model is lacking.

### Level C — Preliminary association

Small studies, inconsistent findings, uncontrolled studies, single-site proof-of-concept studies, or associations with substantial between-person heterogeneity.

### Level D — Not suitable for operational inference

Scientifically interesting but too nonspecific, inconsistent, indirect, or unsupported by the available sensors for use as an AUD-specific output.

This hierarchy is intentionally operational. Formal GRADE is optimized for certainty in clinical intervention effects, whereas this layer mixes biomarkers, psychophysiology, longitudinal associations, and prediction models. Study quality is therefore tracked separately in Section 28.

---

# 3. Target states worth attempting to infer

| AUD target | What is realistically observable from physiology? | Evidence level for operational use | Recommended output |
|---|---|---:|---|
| Alcohol withdrawal | Autonomic activation, tremor component, sweating/temperature pattern, temporal stabilization | A-pilot/B | `withdrawal_related_physiological_activation` + `withdrawal_tremor_evidence` + trend |
| Withdrawal severity | Partial physiological evidence; tremor can be quantified, whole syndrome cannot be replaced by wearables | A-pilot for tremor; C/B for whole-state physiology | `withdrawal_severity_evidence`, not CIWA replacement |
| Craving | Generic ambulatory physiology is insufficiently specific | C/D for physiology alone | No standalone craving classifier |
| Alcohol-cue reactivity | Change from neutral/rest baseline during known cue challenge | B; small prospective A-like evidence for relapse association | `alcohol_cue_autonomic_reactivity` |
| Stress-related AUD response | AUD-relevant only when stress context and AUD-specific response relationship are known | B/C | `aud_relevant_stress_reactivity` |
| Acute intoxication/impairment | Multimodal motion + HR + EDA + temperature pattern can classify impaired vs non-impaired in pilot data | A-pilot | `alcohol_associated_impairment_pattern` |
| Exact BAC | Not supported by current physiology alone | D | Do not output |
| Chronic autonomic dysregulation | Reduced HRV / parasympathetic modulation at group level and within-person burden over time | B | `autonomic_dysregulation_burden` |
| Abstinence/recovery | HR and vagal HRV can improve over weeks-months; nocturnal physiology is useful | B | `autonomic_recovery_trend` |
| Treatment response | Change in HRV/cue-response dynamics can be monitored, but intervention-specific evidence is limited | B/C | `treatment_associated_physiological_change` |
| Relapse risk | Prospective HRV/cue-reactivity associations exist but no mature, externally validated wearable relapse alarm | B/C | `relapse_risk_associated_physiology` only |
| Sleep/recovery deterioration | Alcohol acutely suppresses nocturnal recovery; sleep abnormalities predict relapse in PSG research | A/B for alcohol-associated autonomic suppression; C for wearable relapse inference | `nocturnal_autonomic_recovery_state` |

---

# 4. Modality-to-AUD evidence map

## 4.1 ECG / heart rate / HRV

Cardiac autonomic measures are the best-supported physiological family for this layer.

### Features with credible AUD evidence

- resting heart rate;
- mean RR interval;
- RMSSD;
- total HRV / total power where available;
- SDNN / global RR variability where protocol permits;
- HF-HRV in controlled cue/sleep paradigms;
- pNN50 in acute alcohol studies [18,33];
- nonlinear HRV features in withdrawal (e.g., compression entropy / fractal measures), but these are not yet production-grade;
- cue-evoked change in HF-HRV;
- nocturnal HR and HRV;
- recovery slope across abstinence;
- within-person deviation from a context-matched baseline.

The 2019 systematic review/meta-analysis of 15 quantitative studies found lower parasympathetic HRV in AUD (Hedges' g = -0.4301, 95% CI -0.7601 to -0.1000), and specifically supported total variability and RMSSD; HF alone was not consistently different [5]. This argues for using **RMSSD/global variability as chronic-autonomic features** and treating LF/HF or isolated HF as context-specific rather than universal AUD biomarkers.

Acute alcohol has repeatedly been associated with increased HR and reduced vagally mediated HRV. In 35 healthy men receiving 60 g ethanol, time-domain HRV reductions persisted for up to roughly 10 hours before returning toward control values [18]. In a large real-world within-person study of 4,098 adults, alcohol intake was dose-dependently associated with increased sympathetic regulation, decreased parasympathetic regulation, and reduced physiological recovery during the first three hours of sleep; the HRV-derived recovery state decreased by 9.3, 24.0, and 39.2 percentage units in low-, moderate-, and high-dose groups, respectively [7].

**System implication:** cardiac features are useful for intoxication-associated autonomic effects, chronic dysregulation, abstinence/recovery, nocturnal recovery, and context-locked cue/stress reactivity. They are **not specific enough to label withdrawal or craving in isolation**.

## 4.2 PPG-derived measures

Direct AUD evidence for PPG waveform morphology or peripheral perfusion indices as standalone AUD-state biomarkers is much weaker than evidence for ECG/HRV. PPG is still useful in this system for:

- pulse rate;
- inter-beat interval / pulse-rate variability when validated;
- supporting acute intoxication models that use wrist optical cardiovascular signals;
- peripheral pulse amplitude/perfusion research features, if locally validated.

The 14-day AUD wearable study used wrist PPG to derive BVP, HR, HRV, and IBI, but did not establish a reliable craving mapping [17]. Therefore:

> **PPG should currently be treated mainly as an alternate measurement route for cardiac timing and peripheral response—not as an independently validated AUD-specific modality.**

Pulse morphology, perfusion index, and alcohol-specific vascular waveform features should remain research features until direct AUD validation exists.

## 4.3 Electrodermal activity (EDA/GSR)

Useful features:

- tonic skin conductance level (SCL);
- phasic SCR amplitude;
- peak count / nonspecific SCR rate;
- cue-evoked SCR difference from neutral condition;
- EDA recovery slope;
- tonic/phasic components in intoxication models.

EDA has plausible roles in withdrawal sweating/autonomic activation and cue/stress reactivity. However, cue studies show heterogeneity. In 36 detoxified alcohol-dependent patients, only 22% showed both subjective and physiological cue responses; 42% showed physiological response without subjective response [19]. In another alcohol-cue study, heart rate, skin conductance, and temperature changes were not consistently alcohol-cue specific, despite robust subjective effects [20]. A 100-day ambulatory study also found EDA-to-craving relationships weaker than cardiovascular relationships for most participants [16].

**System implication:** EDA is a supporting modality, strongest when paired with a known challenge/context and within-person baseline. **Do not implement `elevated_EDA → craving`.**

## 4.4 Skin temperature

Temperature is relevant through peripheral vasomotor effects and withdrawal-associated sweating/thermoregulatory patterns, but specificity is limited.

White et al. recorded 24-hour tremor, sweating, skin temperature, and activity in six men undergoing alcohol withdrawal and found elevated tremor/sweating/activity with characteristic diurnal patterns; sweating and skin temperature were highest around midnight–06:00 [1]. Acute alcohol can also increase peripheral skin temperature through vasodilation in controlled experiments [21]. Thus the same temperature direction can occur in very different AUD states.

**System implication:** temperature is a **modifier/supporting signal**, never a primary AUD-state discriminator. It is useful as part of multimodal intoxication, withdrawal, and nocturnal/recovery signatures.

## 4.5 IMU-derived information

IMU has two different roles:

1. upstream context/confound detection; and
2. AUD-specific phenotype measurement.

The strongest AUD-specific role is **withdrawal tremor**. Carver et al. found a linear relationship between accelerometer energy in 4.4–10 Hz and expert-rated withdrawal tremor; 75% of actual withdrawal recordings had a mean peak frequency >7 Hz versus 17% of mimicked tremors [3]. A follow-up analysis of 122 recordings from 61 patients reported smartphone-based tremor severity estimation with RMSE 0.977 on the CIWA-Ar tremor scale, with lower error for 7–10 Hz tremor [4].

However, generic actigraphy is not a withdrawal severity score. In 40 detoxification patients, actigraphy-recorded motor function did **not** correlate with clinical withdrawal symptoms, although sleep/activity patterns differed from controls [22].

For intoxication, motion features are powerful. In Kaczor et al., accelerometry carried most of the individual-level discrimination, with physiological signals providing a modest boost for BrAC-defined impairment [8].

**System implication:** implement separate feature families:

- `withdrawal_tremor_energy_4p4_10Hz`;
- `tremor_peak_frequency`;
- gait/coordination and axis RMS/spectral-power features for intoxication research/monitoring;
- do not map generic activity count to withdrawal severity.

---

# 5. Consolidated feature evidence matrix

| Physiological feature | AUD state/process | Observed relationship | Setting | Individual-level usable? | Recommended system use | Evidence | Key refs |
|---|---|---|---|---|---|---:|---|
| Resting HR | Acute alcohol | Often increases after sufficient dose | Controlled + real world | Supportive, not specific | Intoxication/nocturnal alcohol-associated pattern | B | [6-8,18] |
| RMSSD / global HRV | Chronic AUD | Lower on average in AUD | Meta-analysis | No universal threshold | Longitudinal dysregulation burden | B | [5,6] |
| Nocturnal HR/HF-HRV | Early abstinence | Higher HR and lower vagal activity; recovery over months | PSG longitudinal | Within-person trend useful | Recovery state | B | [23] |
| Cue-evoked HF-HRV | Cue reactivity / relapse association | Dynamic response associated with relapse in small treated sample | Laboratory, prospective | Potential, not externally validated | Challenge-response marker | B/C | [9] |
| Resting HRV | Craving | Accounted for 12.1% additional craving variance in N=26 | Outpatient | Insufficient alone | Vulnerability/context feature only | C | [10] |
| Ambulatory HRV/HR | Subsequent alcohol use | Higher parasympathetic HRV/lower HR associated with more abstinent days | 4-day ECG + 90-day follow-up | Risk-associated, not alarm | Relapse-risk marker | B/C | [15] |
| Tonic/phasic EDA | Intoxication | Changes between sober/intoxicated states | Controlled pilot | Yes in multimodal pilot | Multimodal impairment feature | A-pilot | [8] |
| EDA cue response | Cue/craving | Variable; some cue-related effects, weak specificity across studies | Laboratory | Not generic | Known-cue event support | C | [12,19,20] |
| EDA ambulatory | Craving | Weak/inconsistent; no one-to-one mapping | 14–100 day ambulatory | No | Do not use alone | D/C | [16,17] |
| Skin temperature | Acute alcohol | Peripheral temperature can increase | Controlled | Nonspecific | Support multimodal pattern | C/B | [8,21] |
| Skin temperature | Withdrawal | Diurnal elevation reported with sweating in small sample | Clinical 24 h | No threshold | Support only | C | [1] |
| Tremor energy 4.4–10 Hz | Withdrawal tremor | Tracks expert tremor rating | ED | Yes for tremor component | Withdrawal tremor estimator | A-pilot | [3,4] |
| Generic actigraphy | Withdrawal severity | No correlation with withdrawal symptoms in N=40 | Detox | No | Do not grade withdrawal | D | [22] |
| Accelerometry RMS/spectral features | Intoxication/impairment | Major contributors to BrAC/SFST classification | Controlled pilot | Yes, internally validated | Multimodal impairment classifier | A-pilot | [8] |
| Nocturnal HR/HRV recovery | Recent alcohol exposure | Dose-dependent suppression | Real-world N=4098 | Strong within-person effect | Alcohol-associated nocturnal recovery deterioration | A/B | [7] |
| PSG sleep latency | Relapse risk | Longer latency predicted relapse | Treatment follow-up | PSG-specific; wearable proxy not equivalent | Research context only | B | [24] |

---

# 6. Withdrawal-specific solution

## 6.1 What physiology supports

Alcohol withdrawal includes autonomic hyperactivity, but the evidence does not support replacing a validated clinical withdrawal assessment with HR/HRV/EDA alone. The strongest directly measurable component in the current hardware is **tremor**.

White et al. demonstrated objective 24-hour differences in tremor, sweating, skin temperature, and locomotor activity during withdrawal, but the withdrawal sample was only six men [1]. Bär et al. studied 20 mild and 20 moderate-to-severe withdrawal patients at peak withdrawal, one day later, and near discharge. HR was increased at admission, while most conventional HRV measures did not show the expected autonomic dysfunction; nonlinear compression entropy was more informative [2]. Separate work found reduced baroreflex sensitivity and correlations of nonlinear autonomic measures with withdrawal severity [25]. These results are important because they show that the simplistic rule `withdrawal = low HRV` is not supported.

Carver/Aarabi's tremor work is more operationally direct: accelerometer spectral energy in 4.4–10 Hz was related to expert tremor rating, and later modeling produced a tremor-score RMSE of 0.977 across 61 patients [3,4].

## 6.2 Recommended withdrawal outputs

```text
withdrawal_related_physiological_activation   # state
withdrawal_tremor_evidence                    # state/component
withdrawal_physiological_trend                # trend
withdrawal_severity_evidence                  # evidence bundle, not clinical grade
```

Do **not** output `CIWA-Ar = X` unless a dedicated device-specific model has been trained and validated against CIWA-Ar prospectively.

## 6.3 Withdrawal algorithm

### Inputs

- context: detoxification/known recent cessation or clinically established withdrawal-risk period;
- resting HR deviation from personal/context baseline;
- RMSSD/global HRV deviation, but as supporting evidence only;
- EDA tonic/phasic activation;
- temperature deviation/slope;
- tremor spectral energy 4.4–10 Hz;
- tremor dominant frequency;
- temporal persistence and trend;
- optional contemporaneous clinician withdrawal score for calibration/validation, **not required for runtime physiological evidence generation**.

### Computation

1. **Gate by AUD-withdrawal context.** Without evidence of recent cessation/detoxification context, do not label generic sympathetic activation as withdrawal.
2. Compute robust within-person deviations for resting HR, HRV, EDA, and temperature.
3. Compute tremor-band energy and dominant frequency from stationary upper-limb segments.
4. Generate a `tremor_component_score` using a device-calibrated regression trained against expert tremor ratings; the published 4.4–10 Hz band is a defensible initialization, not a transferable coefficient set [3,4].
5. Generate an `autonomic_activation_support` score from persistent HR/EDA/temperature deviations. HRV contributes only if its direction is consistent with the individual's withdrawal calibration or locally validated model.
6. Fuse tremor + autonomic support over 10-minute windows.
7. Aggregate to hourly state and slope.
8. Output evidence and uncertainty; never infer medication need or severe-withdrawal complications from physiology alone.

### Recommended interpretation rule

A rule-constrained first implementation is preferable to a black-box model:

```text
IF withdrawal_context_present
AND tremor_component_score is elevated
AND >=1 autonomic-support modality is persistently abnormal
THEN withdrawal_related_physiological_activation = supported
ELSE IF withdrawal_context_present AND tremor_component_score elevated
THEN withdrawal_tremor_evidence = supported; global withdrawal evidence = limited
ELSE autonomic abnormalities alone = nonspecific
```

### Time scale

- feature window: 30 s–5 min depending on feature;
- fused state: 10 min;
- persistence: 30–60 min;
- trend: 1–6 h;
- clinical course view: hours–days.

### Evidence grade

- tremor quantification: **A-pilot**;
- whole-withdrawal physiological state: **B/C**;
- physiology-only severity replacement: **D**.

---

# 7. Craving and alcohol-cue reactivity solution

## 7.1 Separate craving from cue reactivity

The literature strongly argues against treating these as the same target.

Quintana et al. found resting HRV accounted for an additional 12.1% of craving variance after age, anxiety, and alcohol consumption in 26 alcohol-dependent outpatients [10]. This is an association, not a craving detector.

In controlled cue exposure, alcohol-dependent patients can show HRV, temperature, EDA, HR, salivation, and subjective changes, but physiological specificity varies [11,12,19,20]. Garland et al. found autonomic indices during stress-primed alcohol cues related to attentional bias in 58 alcohol-dependent adults [11]. Another study found cue-elicited HF-HRV and attentional bias predicted occurrence and timing of relapse at six months among 47 post-treatment participants, of whom nine relapsed during follow-up [9]. Conversely, an N=80 detoxification study found alcohol cues induced physiological cue reactivity and craving, but the magnitude of baseline physiological cue reactivity did **not** predict relapse at 5 weeks or 3 months [13].

The strongest real-world caution comes from van Lier et al. (N=10, 100 days): cardiovascular activity-to-craving associations were negligible to weak overall and highly individualized; EDA performed worse; contextual variables improved some individuals but precision remained low [16]. In the N=11 14-day Empatica study, HRV and EDA features were generally not significantly associated with alcohol use or craving [17].

## 7.2 What to implement

### Implement

```text
alcohol_cue_autonomic_reactivity   # event
```

only when the system knows that an alcohol cue/challenge occurred or the user/clinical protocol labels it.

### Do not implement

```text
craving_detected_from_physiology
craving_intensity = X
```

as a generic ambulatory output.

## 7.3 Cue-reactivity algorithm

### Inputs

- known alcohol-cue context or structured cue-exposure session;
- pre-cue neutral/rest baseline;
- HR and RR/HRV features (especially HF-HRV in a controlled protocol);
- EDA SCL/SCR features;
- skin-temperature slope;
- optional self-reported craving;
- recent treatment/recovery phase.

### Event representation

For feature `f`:

```math
Δf_cue = f_cue_window - f_pre_cue_baseline
```

and, when a neutral cue is available:

```math
ΔΔf = (f_alcohol_cue - f_pre) - (f_neutral_cue - f_pre_neutral)
```

The model should learn a **multivariate response vector**, not a single expected direction, because published autonomic directions differ by paradigm and person.

### Windows

- pre-cue baseline: 2–5 min;
- cue response: first 30 s–5 min depending on paradigm;
- recovery: 5–20 min;
- repeated-cue habituation: session-level.

### Output

```yaml
alcohol_cue_autonomic_reactivity:
  present: true
  confidence: MODERATE
  evidence_grade: B
  response_vector:
    delta_hr: ...
    delta_rmssd: ...
    delta_hf_hrv: ...
    delta_scl: ...
    scr_rate_change: ...
    temperature_slope_change: ...
  self_report_available: true
  interpretation: "autonomic response temporally linked to known alcohol cue"
  not_established:
    - "exact craving intensity"
    - "imminent relapse"
```

If cue context is absent, the same physiology must remain generic autonomic arousal and **must not enter the AUD-specific cue output**.

---

# 8. Stress reactivity and AUD

Stress physiology is not inherently AUD-specific. Direct AUD studies show that recovering alcohol-dependent individuals can have altered basal and challenge responses, and stress-induced craving has prospective relapse relevance [14,26]. In Sinha et al., 28-day-abstinent alcohol-dependent individuals showed elevated basal HR/cortisol, enhanced persistent craving under stress and alcohol-cue imagery, increased anxiety/negative emotion and systolic BP responses, with blunted stress cortisol response compared with social drinkers [26]. A later 90-day prospective study found high stress- and cue-provoked craving and altered HPA-axis measures predicted shorter time to relapse; stress-induced craving hazard ratio was 1.15 (95% CI 1.01–1.31), cue-induced craving HR 1.16 (1.02–1.32), while adrenal sensitivity had stronger hazard ratios [14]. The HPA measures are not available in the current hardware, which limits direct translation.

Therefore the wearable layer should not output `AUD stress` whenever it detects sympathetic activation.

## Recommended output

```text
aud_relevant_stress_reactivity
```

only when:

1. a stressor/context is known;
2. the response is compared with that person's context-conditioned baseline;
3. the response pattern has been calibrated against the person's AUD-relevant craving/recovery history or a validated protocol; and
4. the output is framed as a **reactivity marker**, not relapse prediction.

### Operational use

- detect unusually large or slow-to-recover autonomic response during a known stress episode;
- compare with earlier treatment/recovery sessions;
- optionally request/attach self-report of craving;
- track whether stress-response recovery improves across treatment/abstinence.

### Evidence level

**B for mechanistic/prospective AUD relevance; C for wearable-only individual inference.**

---

# 9. Acute intoxication and post-intoxication recovery

## 9.1 What can be inferred

Acute alcohol reliably alters cardiovascular/autonomic physiology, but exact magnitude depends on dose, timing, sex, baseline, environment, and tolerance [6,18,21,27-29]. Wearable multimodal inference is promising.

Kaczor et al. studied 28 healthy adults who consumed alcohol in a controlled police-academy intoxication laboratory. An Empatica Embrace Plus recorded HR, skin temperature, EDA, and accelerometry. Five-minute windows were labeled using BrAC ≥0.08 g/dL and standardized field sobriety testing. Four classifiers were evaluated with **subject-wise five-fold GroupKFold**, preventing windows from the same participant from leaking across folds [8].

For BrAC-defined impairment:

- accelerometer-only: accuracy 0.79, sensitivity 0.77, specificity 0.81, AUC 0.85;
- physiological-only: accuracy 0.67, sensitivity 0.60, specificity 0.72;
- integrated: accuracy 0.80, sensitivity 0.78, specificity 0.81, AUC ~0.84 [8].

For SFST-defined impairment, accelerometer-only accuracy was 0.78; integrated accuracy was 0.74 with sensitivity 0.519 and specificity 0.847 [8]. Thus motion dominated; physiology added a small benefit for BrAC-based classification.

This is **A-pilot evidence** because it is an individual-level model with subject-wise validation, but the sample is small and no independent external cohort validation was reported.

## 9.2 What cannot be inferred

The result supports:

```text
alcohol-associated impairment pattern
```

not:

```text
exact BAC
```

and not a legal/medical intoxication determination.

## 9.3 Recommended intoxication algorithm

### Research/validation phase

1. Collect paired personal sober baseline and alcohol-exposure sessions with objective BrAC reference.
2. Segment 5-minute windows with 1-minute stride as in the pilot.
3. Extract:
   - accelerometer axis mean/SD/RMS/range/entropy/jerk/dominant-frequency/power;
   - vector-magnitude features;
   - HR mean/SD;
   - tonic/phasic EDA, peak count, normalized conductance;
   - temperature mean/SD;
   - optional personal covariates only if explicitly available and ethically appropriate.
4. Train a model using **subject-grouped validation**. Never randomly split windows across train/test because that leaks person-specific patterns.
5. Calibrate probability output and validate on a held-out external cohort before clinical deployment.

### Runtime output

```yaml
intoxication_pattern:
  evidence: MODERATE
  model_probability: 0.xx
  evidence_grade: A_PILOT
  reference_target: "alcohol-associated impairment pattern; model development label BrAC>=0.08"
  dominant_support:
    - gait_motion_pattern
    - hr_variability_change
    - eda_change
    - temperature_change
  not_established:
    - exact_BAC
    - legal_impairment
```

## 9.4 Post-intoxication recovery

Acute HRV effects can persist for hours. After 60 g ethanol, HRV effects were observed up to about 10 hours in young men [18]. Nocturnal autonomic suppression is dose dependent [7,29]. Therefore define recovery as return toward the person's sober, context-matched baseline rather than using a universal time threshold.

Recommended output:

```text
post_alcohol_autonomic_recovery_trend
```

with hourly/nightly distance-to-baseline rather than `sober = true`.

---

# 10. Chronic alcohol-related autonomic dysregulation

The strongest synthesis is the Cheng meta-analysis: across 15 quantitative studies, AUD was associated with reduced parasympathetic HRV, with total variability and RMSSD supported as useful summary indices; heterogeneity was high (I² = 83.8%) [5]. The Ralevski review similarly concluded that resting HRV is generally lower in AUD, acute alcohol reduces HRV, and recovery may require months [6].

This is strong evidence for a **population-level phenotype**, not a diagnostic threshold. Many non-AUD factors lower HRV.

## Recommended state

```text
autonomic_dysregulation_burden
```

### Features

Primary:

- RMSSD;
- global RR variability / SDNN where recording protocol is appropriate;
- nocturnal/resting HR;
- stable-rest total HRV.

Secondary/research:

- HF-HRV in controlled sleep/rest;
- nonlinear HRV metrics;
- baroreflex-related measures if available in future hardware.

### Baseline and time scale

- use stable rest/sleep periods;
- baseline: at least 5–7 valid days when feasible;
- report weekly medians and robust distributions;
- trend horizon: 2–16 weeks.

### Interpretation

```text
"Persistent cardiac autonomic profile is shifted toward lower vagal variability relative to the patient's stable baseline and/or recovery target; this is consistent with AUD-associated autonomic dysregulation but is not specific to AUD."
```

Evidence level: **B**.

---

# 11. Abstinence, treatment, and physiological recovery

## 11.1 Longitudinal recovery

De Zambotti et al. followed 15 recently detoxified alcohol-dependent adults and 13 controls during sleep. At the initial assessment (~24 days since last drink), the AUD group had higher HR and lower vagal HF activity. At four months, HR decreased and vagal modulation improved; six of 15 relapsed during follow-up, reducing later sample size [23]. Earlier work also reported RR-interval variation improving by roughly day 30 after abstinence [30].

These studies support **within-person autonomic recovery trends**, especially during sleep/rest.

## 11.2 Treatment-associated physiology

In a randomized study of 48 inpatients, six HRV-biofeedback sessions over two weeks reduced craving immediately post-intervention, while CVNN only showed a nonsignificant increasing trend; vasomotor function improved [31]. This is not evidence that an HRV increase universally equals treatment success.

In 42 alcohol-dependent participants undergoing a cue-reactivity task, high-dose baclofen altered HF-HRV cue/recovery dynamics compared with lower-dose/placebo groups, without a medication effect on subjective craving [32]. This supports using **challenge-response dynamics** as a treatment research endpoint.

## Recommended outputs

```text
autonomic_recovery_trend
treatment_associated_physiological_change
```

### Algorithm

1. Define stable rest/sleep windows and a pre-treatment/early-abstinence baseline.
2. Compute nightly/weekly median HR, RMSSD, global HRV and recovery metrics.
3. Express each feature as robust within-person deviation.
4. Fit a trend over at least 7 days; for longer recovery, compare 2–4 week blocks.
5. If a standardized cue/stress challenge is repeated, compute response and recovery vectors pre/post treatment.
6. Output direction and confidence; avoid universal thresholds.

### Time scales

- detoxification/early recovery: days;
- short treatment response: 2–6 weeks;
- autonomic recovery: months.

Evidence: **B for longitudinal autonomic recovery; C/B for treatment-specific challenge response.**

---

# 12. Relapse-risk-related physiology

This is the section where overclaiming would be most damaging.

## 12.1 Prospective evidence

### Eddie et al. 2023

Forty-two adults in the first year of an AUD recovery attempt wore ambulatory ECG for four days and were followed for 90 days using timeline follow-back. Higher parasympathetically mediated HRV and lower HR were associated with a greater percentage of abstinent days during follow-up [15]. This is prospective, naturalistic, and directly relevant, but N=42 is small and it did not establish a deployable threshold or independently validated relapse classifier.

### Garland et al. 2012

Among 47 participants with post-treatment physiological data, nine relapsed over six months. Cue-elicited HF-HRV and attentional bias predicted occurrence/timing of relapse after controlling for dependence severity and treatment condition [9]. This is provocative individual-level prospective evidence, but it is a small laboratory paradigm and includes a cognitive attentional-bias variable unavailable from passive physiology.

### Witteman et al. 2015

In 80 detoxification patients, alcohol advertisements induced HRV cue reactivity and craving, but baseline cue-reactivity magnitude did **not** predict relapse at five weeks or three months [13]. This is an important contradictory finding.

### Sinha et al. 2011

Stress/cue-induced craving and HPA-axis measures predicted time to relapse over 90 days [14]. The strongest physiological predictor involved cortisol/corticotropin ratio, which the current hardware does not measure.

### Brower et al. 1998

In 74 treated alcohol-dependent patients, 36 relapsed over an average five-month follow-up. Longer PSG sleep latency and other sleep abnormalities improved relapse prediction beyond nonsleep variables; PSG sleep latency was the strongest sleep predictor [24]. A wearable IMU cannot be assumed to reproduce PSG stage/latency metrics with equivalent validity.

## 12.2 Recommended output

Never:

```text
relapse_probability = 0.82
relapse_expected_in_48_hours
```

unless a future, prospectively validated and calibrated model directly supports that claim.

Use:

```text
relapse_risk_associated_physiological_deviation
```

and treat it as a **risk marker**.

### Inputs

- multiday resting/nocturnal parasympathetic HRV;
- HR;
- change from personal recovery baseline;
- known cue-reactivity response if measured;
- nocturnal recovery deterioration;
- recent trajectory over days;
- optional self-report/context, which substantially improves interpretability.

### Runtime policy

- require at least 3 valid days for a short-term deviation;
- prefer 7-day baseline and 3–7-day change windows;
- do not trigger from a single HRV window;
- if risk markers disagree, output `AMBIGUOUS`;
- never make this output actionable without the downstream clinical layer.

Evidence level: **B/C**.

---

# 13. Sleep and recovery deterioration

Sleep is relevant in two different ways.

## 13.1 Acute alcohol-associated nocturnal autonomic suppression

The N=4098 within-person study found dose-dependent impairment of autonomic recovery during the first three hours of sleep even at low reported alcohol intake [7]. Controlled laboratory studies also show dose-dependent higher nocturnal HR and lower vagal HRV after alcohol [29,34]. This is strong evidence that nocturnal HR/HRV can identify **alcohol-associated physiological recovery suppression**, especially relative to the person's alcohol-free nights.

Recommended output:

```text
nocturnal_autonomic_recovery_state
```

not `intoxicated during sleep`.

## 13.2 Sleep as relapse-related physiology

PSG studies show that sleep abnormalities can predict relapse [24], but consumer/wearable sleep fragmentation and IMU-derived sleep are not interchangeable with PSG architecture. The detox actigraphy study found an association between self-reported and actigraphy-recorded sleep, but actigraphy did not track withdrawal severity [22].

Therefore:

- use IMU for sleep/wake/restlessness and continuity;
- use HR/HRV for nocturnal autonomic recovery;
- do not infer REM/stage-4 abnormalities from IMU alone;
- do not generate a relapse alarm from wearable sleep deterioration alone.

---

# 14. Temporal dynamics model

| Target | Feature window | Baseline horizon | Response/persistence | Trend horizon | Output class |
|---|---:|---:|---:|---:|---|
| Withdrawal tremor | 10–30 s spectral windows | pre-withdrawal/resting motor baseline if available | repeated across 5–10 min | 1–6 h / days | state + trend |
| Withdrawal autonomic activation | 1–5 min | 1–7 days or earliest stable detox baseline | ≥30 min preferred | hours–days | state + trend |
| Cue reactivity | 30 s–5 min | 2–5 min pre-cue / neutral cue | minutes | repeated sessions | event |
| AUD stress reactivity | 1–5 min | context-matched rest | minutes to ~75 min in lab paradigms | sessions/weeks | event + recovery |
| Intoxication pattern | 5 min | personal sober baseline | consecutive windows | hours | state |
| Post-intoxication recovery | 5–30 min | sober context baseline | hours | 6–12 h / overnight | trend |
| Chronic autonomic dysregulation | nightly/rest 5 min+ | ≥5–7 valid days | persistent | weeks–months | state/trend |
| Abstinence recovery | nightly/weekly | early abstinence baseline | repeated nights | 2–16 weeks | trend |
| Relapse-associated deviation | nightly/daily | ≥7 days preferred | ≥3 days | 3–14 days | risk marker |
| Nocturnal recovery | first 3 h sleep + whole night | alcohol-free personal nights | one night | 3–7 nights | state/trend |

---

# 15. Absolute values versus personal baseline

The evidence favors **within-person/context-conditioned interpretation** wherever possible. This is particularly important because HR, HRV, EDA, temperature, and cue responses vary substantially between people [16,17].

## 15.1 Robust deviation

For feature `f` in context `c`:

```math
z^*_{f,c}(t) = \frac{f(t)-median(B_{f,c})}{1.4826\,MAD(B_{f,c}) + \epsilon}
```

where `B_f,c` is the personal baseline distribution for the same context (e.g., quiet rest, first three hours of sleep).

For interpretable percent changes:

```math
Δ%_{f,c}(t)=100\times\frac{f(t)-median(B_{f,c})}{median(B_{f,c})}
```

Use percent change only for strictly positive, non-near-zero metrics.

## 15.2 Baseline types

Maintain separate baselines:

- `stable_rest_baseline`;
- `sleep_baseline`;
- `abstinence_recovery_baseline`;
- `sober_reference_baseline` for intoxication research;
- `neutral_cue_baseline` for cue paradigms;
- optional `early_detox_baseline` for within-course withdrawal trends.

## 15.3 Baseline update policy

Do not indiscriminately update baselines during suspected events. A practical rule:

- update only during windows marked physiologically stable and outside known alcohol exposure, acute withdrawal, cue challenge, or high-stress event;
- use exponentially weighted or rolling robust statistics;
- retain a long-term reference so gradual pathological drift is not normalized away.

Maintain both:

```text
short_baseline: recent stable 7–14 days
long_reference: clinically stable/recovery reference period
```

---

# 16. Multimodal AUD physiological signatures

Multimodal fusion is useful only when the target and ground truth justify it.

## 16.1 Withdrawal

Recommended combination:

```text
IMU tremor component
+
resting HR deviation
+
EDA/sweating-related activation
+
temperature support
+
withdrawal context
```

HRV is a secondary feature because conventional HRV results during acute withdrawal are inconsistent [2,25].

## 16.2 Cue reactivity

```text
ΔHR/ΔHF-HRV
+
ΔEDA
+
temperature slope
+
known alcohol cue
+
(optional) self-report craving
```

The context is part of the signature; without it, the same physiology is nonspecific.

## 16.3 Intoxication

```text
accelerometer/gait dynamics
+
HR/HR variability
+
EDA tonic/phasic
+
skin temperature
+
personal sober baseline
```

This combination has direct A-pilot evidence [8].

## 16.4 Recovery

```text
nocturnal HR
+
RMSSD/global HRV
+
night-to-night stability
+
sleep/rest context
+
weeks of trajectory
```

This is more defensible than fusing every available sensor.

---

# 17. AUD-specific physiological state representation

Use two distinct confidence concepts:

1. **evidence_grade** — strength of research supporting the mapping;
2. **runtime_confidence** — strength/completeness/consistency of the current patient's evidence.

Recommended schema:

```yaml
aud_physiological_interpretation:
  timestamp: "..."
  model_version: "aud-phys-meaning-v1"

  withdrawal:
    type: STATE
    evidence_score: 0.72
    evidence_grade: B
    runtime_confidence: MODERATE
    interpretation: "pattern consistent with withdrawal-related physiological activation"
    supporting_features:
      - feature: tremor_energy_4p4_10Hz
        deviation: +2.4_robust_z
      - feature: resting_hr
        deviation: +1.8_robust_z
      - feature: tonic_eda
        deviation: +1.5_robust_z
    contradictory_features:
      - feature: rmssd
        status: no_material_change
    temporal_trend: WORSENING
    context_requirements:
      withdrawal_context: SATISFIED
    limitations:
      - "does not estimate CIWA-Ar total score"
      - "does not determine medication need"

  cue_reactivity:
    type: EVENT
    evidence_score: null
    status: NOT_EVALUATED
    reason: "no known alcohol-cue context"

  intoxication_pattern:
    type: STATE
    status: INSUFFICIENT_EVIDENCE

  autonomic_dysregulation:
    type: TREND
    evidence_score: 0.66
    evidence_grade: B
    runtime_confidence: MODERATE
    trend: IMPROVING
    horizon: "28d"

  relapse_associated_physiology:
    type: RISK_MARKER
    status: WEAK_EVIDENCE
    evidence_grade: C
    runtime_confidence: LOW
    interpretation: "multiday physiology has shifted toward a profile associated with poorer recovery outcomes"
    limitations:
      - "not a relapse prediction"
```

---

# 18. Evidence fusion

## 18.1 Why not one global model

The targets have different ground truths, time scales, and evidence quality. A single neural network predicting every AUD state would mix incompatible labels and make uncertainty difficult to audit.

Use **target-specific models** with a shared feature store.

## 18.2 First implementation: research-constrained evidence accumulation

For target `y`, define normalized feature-evidence terms `e_i ∈ [-1,1]`, where positive values support the target, negative values contradict it, and zero is neutral.

```math
E_y(t) = \frac{\sum_{i \in A_t} w_{y,i} e_i(t)}{\sum_{i \in A_t}|w_{y,i}|}
```

where `A_t` is the set of available features.

The weights are **not literature effect sizes**. They should initially reflect ordinal evidence strength and then be learned/calibrated from local labeled data. Published studies usually do not provide transferable patient-level likelihood ratios for these multimodal combinations.

Then apply temporal persistence:

```math
\tilde{E}_y(t) = \alpha E_y(t) + (1-\alpha)\tilde{E}_y(t-1)
```

for state-like targets, or event-specific pre/post deltas for cue reactivity.

## 18.3 Model by target

| Target | Preferred practical method | Why |
|---|---|---|
| Withdrawal tremor | supervised regression against expert tremor rating | direct measurable component and prior precedent |
| Withdrawal state | constrained weighted evidence + temporal state model | heterogeneous physiology; limited validated classifier data |
| Cue reactivity | within-person multivariate change score; mixed-effects/logistic model for research | event-locked, person-specific response |
| Intoxication pattern | gradient-boosted trees / XGBoost with subject-grouped validation | direct pilot evidence and nonlinear multimodal features [8] |
| Chronic dysregulation | robust longitudinal score / hierarchical mixed model | phenotype is continuous and person-specific |
| Recovery | state-space / mixed-effects trend model | gradual change over weeks/months |
| Relapse-associated physiology | survival model only after prospective validation | outcome is time-to-event; evidence currently insufficient for deployment |
| Sleep recovery | within-person dose/recovery deviation model | strong within-person evidence [7] |

## 18.4 Bayesian interpretation

A future clinically validated model can formalize:

```math
P(Y_t | X_{1:t}, C_t, B) \propto P(X_t | Y_t, C_t, B)P(Y_t|Y_{t-1})
```

where:

- `Y_t` = latent AUD-specific physiological state;
- `X_t` = current physiological feature vector;
- `C_t` = context;
- `B` = personal baseline parameters;
- `P(Y_t|Y_{t-1})` = temporal transition model.

This is particularly appropriate for withdrawal/recovery states, but the probabilities must be trained/calibrated on a target population. They cannot be populated from group-level literature alone.

---

# 19. State vs event vs trend vs risk marker

| Output | Class | Reason |
|---|---|---|
| withdrawal-related autonomic activation | State | persists over tens of minutes/hours |
| withdrawal tremor | State/component | measurable continuously during period |
| alcohol-cue autonomic response | Event | time-locked to cue |
| stress-reactivity response | Event + recovery | challenge-locked |
| alcohol-associated impairment pattern | State | present across consecutive windows |
| post-alcohol recovery | Trend | movement toward baseline over hours |
| chronic autonomic dysregulation | State + trend | persistent baseline phenotype |
| abstinence recovery | Trend | days–months |
| relapse-associated physiology | Risk marker | probabilistic association, not state |
| nocturnal autonomic recovery | State/night + trend | night-specific and longitudinal |

---

# 20. Confidence and uncertainty

Runtime confidence should not be the model probability alone.

Define:

```math
C_y = C_{model} \times C_{coverage} \times C_{consistency} \times C_{temporal} \times C_{context}
```

with each component in `[0,1]`.

- `C_model`: calibration/validation quality for the target model;
- `C_coverage`: fraction of required/supporting modalities available;
- `C_consistency`: agreement among independent evidence families;
- `C_temporal`: persistence/replication across expected windows;
- `C_context`: completeness of target-specific context.

Map to labels conservatively:

```text
>=0.80 STRONG
0.60–0.79 MODERATE
0.40–0.59 WEAK
<0.40 INSUFFICIENT
```

These numerical boundaries are **engineering defaults**, not clinically validated thresholds, and should be calibrated during validation.

If high-quality features contradict one another, cap confidence at `WEAK` or `AMBIGUOUS` even when the weighted score is numerically high.

---

# 21. Missing-modality handling

| Target | Minimum practical signals/context | Helpful optional signals | Missing-data behavior |
|---|---|---|---|
| Withdrawal tremor | IMU + withdrawal context | HR, EDA, temp | can output tremor evidence alone; do not infer global withdrawal strongly |
| Withdrawal activation | withdrawal context + HR + one of EDA/IMU | HRV, temp | lower confidence if tremor unavailable |
| Cue reactivity | known cue + cardiac or EDA + pre-cue baseline | temp, self-report | without cue context: do not evaluate |
| Stress reactivity | known stress context + cardiac/EDA | self-report craving | without context: generic stress only |
| Intoxication pattern | IMU + at least one autonomic channel + sober baseline | HR, EDA, temp | model must be trained for each allowed feature subset; otherwise `NOT_EVALUABLE` |
| Chronic dysregulation | ECG/validated IBI + stable rest/sleep | PPG backup | if no cardiac timing: not evaluable |
| Recovery | cardiac timing + repeated nights/rest periods | temp/IMU sleep | require multiple valid days |
| Relapse risk marker | repeated cardiac data + recovery context | cue response, sleep | absent baseline => insufficient |
| Nocturnal recovery | HR/IBI + sleep/rest segmentation | temp, IMU | no sleep context => use resting recovery only |

Do not impute an absent physiological modality with population averages for individual-level AUD interpretation.

---

# 22. Contradictory physiological evidence

Example:

```text
HR +2.0 z
RMSSD unchanged
EDA baseline
no tremor
```

In a withdrawal context, this should become:

```text
withdrawal evidence = WEAK / AMBIGUOUS
support = isolated cardiovascular activation
contradiction = absent corroborating autonomic/tremor response
```

not `withdrawal = true`.

A useful contradiction metric is:

```math
D_y = \frac{\sum |w_i| I(sign(e_i) \neq sign(E_y))}{\sum |w_i|}
```

If `D_y` exceeds a calibrated threshold, reduce confidence or return `CONFLICTING_EVIDENCE`.

---

# 23. Personalization requirements

Physiological AUD inference requires substantial personalization because baseline autonomic tone and reactivity vary with age, sex, fitness, medication, chronicity of alcohol exposure, abstinence duration, cardiovascular disease, psychiatric comorbidity, smoking, sleep, and many other variables. The layer should **not infer those variables** unless they are supplied.

Minimum personalization:

1. context-conditioned resting/sleep baseline;
2. stable within-person HR/HRV distribution;
3. EDA and temperature distributions by context;
4. optional sober baseline for intoxication research;
5. optional neutral and alcohol-cue response profile;
6. recovery-stage metadata when clinically available.

The van Lier 100-day study is particularly important here: inter- and intra-individual variability was large enough that generic physiology-to-craving mappings were not viable [16].

---

# 24. Clinical-scale relationships

| Physiological measure | Clinical/reference outcome | Relationship | Operational consequence |
|---|---|---|---|
| Accelerometer tremor energy 4.4–10 Hz | expert withdrawal tremor rating / CIWA-Ar tremor component | linear relationship reported [3] | train tremor component estimator |
| Smartphone tremor model | expert CIWA-Ar tremor ratings | RMSE 0.977 across 122 recordings; lower error at 7–10 Hz [4] | supports device-based tremor quantification |
| HRV/nonlinear autonomic measures | alcohol withdrawal score | some nonlinear/baroreflex measures correlated; conventional HRV inconsistent [2,25] | do not derive total withdrawal score from HRV |
| Resting HRV | OCDS craving | additional 12.1% variance in N=26 [10] | use as vulnerability feature only |
| Cue-evoked HF-HRV | 6-month relapse | predicted occurrence/timing in small treated cohort [9] | challenge-response risk marker, not universal threshold |
| PSG sleep latency | relapse over ~5 months | strongest sleep predictor in N=74 [24] | sleep physiology is clinically relevant, but wearable proxy needs validation |
| BrAC ≥0.08 classifier | BrAC reference | integrated wearable model accuracy 0.80, sens 0.78, spec 0.81 [8] | supports pilot impairment-pattern classifier |

---

# 25. AUD versus generic SUD evidence

This design prioritizes **direct AUD evidence**. The core algorithms above are based on alcohol dependence/AUD, alcohol withdrawal, controlled alcohol administration, or alcohol-recovery cohorts.

Labels used internally should include:

```text
DIRECT_AUD
GENERAL_SUD
RELATED_MECHANISTIC
```

Do not silently import a cocaine/opioid stress biomarker into the alcohol-specific model. General SUD evidence may motivate a research feature but must not increase AUD runtime confidence until alcohol-specific validation exists.

---

# 26. Laboratory versus clinical versus ambulatory evidence

| Finding | Evidence environment | Translation constraint |
|---|---|---|
| Withdrawal tremor spectral model | ED/clinical | promising for clinical monitoring; still device/protocol-specific |
| HR/HRV during withdrawal | detox/clinical | medications and severity alter physiology |
| Cue-reactivity HRV/EDA | laboratory | requires known cue and controlled baseline; weaker in daily life |
| Stress/cue relapse predictors | laboratory + prospective follow-up | predictors include self-report/HPA not fully available on wearable |
| Intoxication model | controlled alcohol lab | needs external real-world validation |
| 100-day craving physiology | ambulatory | strongest warning against generic craving inference |
| 14-day E4 AUD study | ambulatory | feasibility demonstrated; craving mapping weak |
| 4-day ECG + 90-day drinking outcome | ambulatory + prospective | clinically relevant association, small N |
| nocturnal HRV/alcohol | real-world N=4098 | strong within-person alcohol-associated recovery evidence |

Ambulatory evidence receives greater weight for passive monitoring claims, but controlled studies remain essential for challenge-response targets.

---

# 27. Performance metrics and validation requirements

For any model promoted beyond evidence accumulation, record:

- subject-level train/validation/test partitioning;
- AUROC and AUPRC when class balance warrants;
- sensitivity/specificity;
- precision/recall/F1;
- calibration curve and Brier score;
- confidence intervals;
- subgroup performance;
- prediction horizon;
- external validation;
- decision-curve utility if used clinically.

### Mandatory anti-leakage rule

For sliding wearable windows, **all windows from a participant must remain in the same fold**. Kaczor et al.'s GroupKFold approach is the correct pattern [8]. Random window-level splitting will produce deceptively high performance by learning person identity/baseline.

### Minimum validation by target

- withdrawal tremor: prospective device-specific comparison to blinded expert rating;
- withdrawal global state: prospective comparison to clinical assessments with medication timing recorded;
- cue reactivity: repeated sessions + neutral-cue control + test-retest reliability;
- intoxication: external cohort with objective BrAC/TAC and real-world validation;
- recovery: repeated-measures longitudinal analysis;
- relapse risk: prospective survival model with preregistered horizon and external validation.

---

# 28. Study-quality assessment of major implementation papers

| Ref | Population / N | Setting | Sensors/features | Target | Design / validation | Main implementation value | Major limitations |
|---|---|---|---|---|---|---|---|
| [1] White 1994 | 6 male withdrawal + age-matched controls | 24-h clinical ambulatory | tremor, sweating, skin temp, activity | withdrawal pattern | observational | demonstrates multimodal temporal withdrawal physiology | extremely small N |
| [2] Bär 2006 | 20 mild + 20 moderate/severe male AWS | clinical detox | HRV, SSR, BP, HR | withdrawal severity/course | repeated at peak, +1 day, discharge | shows HR elevation and limits of conventional HRV | male only; treatment in severe group; no deployable classifier |
| [3] Carver 2014 | 84 recordings, 61 subjects (49 patients, 12 nurses) | ED | accelerometry | tremor severity / factitious distinction | comparison with experts | 4.4–10 Hz tremor band; >7 Hz observation | preliminary, recording-level data |
| [4] Aarabi 2015 | 122 recordings, 61 patients | ED | smartphone accelerometry | CIWA-Ar tremor component | model vs expert ratings | RMSE 0.977; direct component quantification | no external validation; not full CIWA |
| [5] Cheng 2019 | 15 studies in meta-analysis | mixed | HRV | chronic AUD autonomic phenotype | random-effects meta-analysis | supports RMSSD/global HRV | high heterogeneity; group-level |
| [7] Pietilä 2018 | 4,098 adults | free living | beat-to-beat RR | nocturnal alcohol effect | within-person with/without alcohol days | dose-dependent real-world recovery suppression | self-reported alcohol dose; occupational sample |
| [8] Kaczor 2026 | 28 healthy adults | controlled intoxication lab | HR, temp, EDA, accelerometry | BrAC/SFST impairment | subject-wise 5-fold GroupKFold | direct multimodal classifier; detailed metrics | small N; controlled setting; no external validation |
| [9] Garland 2012 | 47 post-treatment analyzed; 9 relapsed by 6 mo | residential + lab + follow-up | HF-HRV + attentional bias | relapse occurrence/timing | logistic/survival-type prospective analysis | challenge-response prognostic signal | small relapse count; includes cognitive feature; no external validation |
| [10] Quintana 2013 | 26 AUD outpatients | outpatient | resting HRV | craving | cross-sectional regression | supports HRV as vulnerability correlate | small N; only 12.1% added variance; not classifier |
| [11] Garland 2012 | 58 alcohol-dependent adults | lab cue/stress | HRV, finger temp | stress-primed cue response | regression | context-locked autonomic cue reactivity | laboratory; association not detection accuracy |
| [13] Witteman 2015 | 80 detox patients | detox + lab + follow-up | HRV + craving | cue reactivity / relapse | 5 wk and 3 mo follow-up | important negative relapse-prediction result | controlled advertisement cues; follow-up duration limited |
| [14] Sinha 2011 | recovering AUD + controls; prospective AUD follow-up | lab + 90-day follow-up | cortisol/ACTH + craving/anxiety | relapse | survival analysis | establishes stress/cue/HPA prognostic relevance | key physiological markers not in current wearable |
| [15] Eddie 2023 | 42 early-recovery adults | 4-day ambulatory ECG + 90-day follow-up | HRV, HR | % days abstinent | prospective regression | strongest direct naturalistic cardiac-risk association | small N; no deployable threshold/external validation |
| [16] van Lier 2022 | 10 AUD participants, 100 days | ambulatory | cardiovascular + EDA + EMA | craving/lapse | idiographic MCC + lagged analysis/decision tree | shows need for personalization and limits of alarms | tiny N; case-study design |
| [17] wearable AUD study 2021 | 11 (10 female), up to 14 days | ambulatory | PPG HRV, EDA, temp, activity + EMA | stress/craving/use | correlation/feasibility | confirms wearable feasibility; negative craving specificity | tiny unbalanced sample; no multilevel powered inference |
| [22] Melby 2020 | 40 detox patients + 34 controls | clinical | actigraphy | withdrawal/activity/sleep | RCT ancillary analysis | shows generic activity does not grade withdrawal | treatment study; actigraphy not full physiology |
| [23] de Zambotti 2015 | 15 AUD + 13 controls | PSG at ~1, 2, 4 mo | ECG HRV during NREM | autonomic recovery | longitudinal | supports months-scale recovery tracking | small N; attrition/relapse; PSG setting |
| [24] Brower 1998 | 74 alcohol-dependent; 36 relapsed | treatment + ~5 mo follow-up | PSG + subjective sleep | relapse | logistic regression | sleep physiology clinically meaningful | old study; PSG features not directly wearable-equivalent |
| [31] Penzlin 2015 | 48 inpatients randomized 24/24 | rehab | CVNN, vasomotor, craving | treatment response | RCT | illustrates multi-domain treatment physiology | HRV change nonsignificant; small sample |
| [32] Logge 2020 | 42 AUD | cue challenge | skin conductance, HR, HF-HRV | treatment/cue response | controlled medication groups | shows treatment can alter response/recovery dynamics | small unequal groups; no simple wearable endpoint |

---

# 29. Final per-target algorithms

## 29.1 Withdrawal-related physiological activation

```text
INPUT
  withdrawal_context
  resting HR deviation
  RMSSD/global HRV deviation
  tonic/phasic EDA deviation
  temperature deviation
  IMU tremor spectral features
  recent 6 h history

ALGORITHM
  1. Require withdrawal-risk context.
  2. Estimate tremor component from 4.4–10 Hz energy + spectral shape.
  3. Estimate persistent autonomic activation from HR + EDA + temperature.
  4. Treat HRV as support/contradiction, not mandatory direction.
  5. Fuse over 10 min; smooth across 30–60 min.
  6. Track 1–6 h slope.
  7. Output evidence, trend, contributors, contradictions.

OUTPUT
  withdrawal_related_physiological_activation
  withdrawal_tremor_evidence
  withdrawal_physiological_trend

DO NOT OUTPUT
  CIWA total
  medication need
  seizure/delirium prediction
```

## 29.2 Alcohol-cue autonomic reactivity

```text
INPUT
  known alcohol-cue label
  2–5 min pre-cue baseline
  HR/HRV
  EDA
  temperature
  optional craving self-report

ALGORITHM
  1. Compute within-person pre/post deltas.
  2. If neutral cue exists, compute alcohol-minus-neutral differential.
  3. Form multivariate response vector.
  4. Compare to person's prior alcohol-cue response distribution.
  5. Estimate response magnitude and recovery time.
  6. Attach self-report rather than replacing it.

OUTPUT
  alcohol_cue_autonomic_reactivity

DO NOT OUTPUT
  craving intensity from physiology alone
```

## 29.3 AUD-relevant stress reactivity

```text
INPUT
  known stress context
  context-matched baseline
  HR/HRV + EDA ± temperature
  optional craving/self-report

ALGORITHM
  1. Detect magnitude of autonomic challenge response.
  2. Estimate recovery half-time / return-to-baseline.
  3. Compare with patient's historical stress-response distribution.
  4. Elevate AUD-specific interpretation only when linked to AUD-relevant
     context/history or validated challenge paradigm.

OUTPUT
  aud_relevant_stress_reactivity
```

## 29.4 Alcohol-associated impairment pattern

```text
INPUT
  5-min motion + HR + EDA + temperature features
  personal sober reference

ALGORITHM
  1. Use a locally trained multimodal classifier.
  2. Group folds by participant during validation.
  3. Require consecutive positive windows for state persistence.
  4. Calibrate probability.
  5. Report "alcohol-associated impairment pattern" only.

OUTPUT
  intoxication_pattern_evidence

DO NOT OUTPUT
  exact BAC
```

## 29.5 Chronic autonomic dysregulation burden

```text
INPUT
  stable-rest/nightly RMSSD, SDNN/global variability, HR
  >=5–7 day baseline

ALGORITHM
  1. Exclude event/challenge windows.
  2. Calculate robust nightly/rest medians.
  3. Compare with personal long reference.
  4. Estimate weekly burden and trend.
  5. Require persistence across multiple days.

OUTPUT
  autonomic_dysregulation_burden
```

## 29.6 Abstinence / autonomic recovery

```text
INPUT
  nightly/resting HR + HRV
  early-recovery reference
  weeks of data

ALGORITHM
  1. Calculate nightly HR/RMSSD/global HRV.
  2. Fit robust trend over 7–28 day blocks.
  3. Compare to early-abstinence state and stable personal target.
  4. Separate transient bad nights from persistent trajectory change.

OUTPUT
  autonomic_recovery_trend
```

## 29.7 Relapse-risk-associated physiology

```text
INPUT
  3–7 day HR/HRV deviation
  nocturnal recovery trend
  optional known cue-reactivity phenotype
  recovery-stage context

ALGORITHM
  1. Require multi-day evidence.
  2. Detect deterioration from recovery baseline.
  3. Do not calculate event probability unless prospectively validated.
  4. Output risk-associated deviation and supporting literature grade.

OUTPUT
  relapse_risk_associated_physiological_deviation
```

## 29.8 Nocturnal recovery deterioration

```text
INPUT
  sleep/rest segmentation
  first 3 h + whole-night HR/HRV
  personal alcohol-free-night baseline

ALGORITHM
  1. Compute nocturnal HR elevation and parasympathetic HRV suppression.
  2. Calculate recovery index relative to baseline.
  3. Track dose/exposure context when available.
  4. Aggregate across 3–7 nights.

OUTPUT
  nocturnal_autonomic_recovery_state
```

---

# 30. What We Should NOT Infer From Physiology Alone

The following outputs are not supported for the current system without additional validated measurements/models:

1. **AUD diagnosis.** Chronic HRV/EDA abnormalities are nonspecific.
2. **Exact craving intensity.** Ambulatory physiology-to-craving mappings are inconsistent and strongly individualized [16,17].
3. **Craving presence from elevated EDA/HR alone.** Generic arousal has many causes.
4. **Exact BAC or BrAC.** Current multimodal wearable evidence supports impairment-pattern classification, not accurate concentration estimation [8].
5. **Legal intoxication.** That requires validated legal/reference procedures.
6. **Definite alcohol withdrawal from HR/HRV/EDA alone.** Withdrawal context and clinical assessment remain essential.
7. **Full CIWA-Ar score from wearables.** Tremor component estimation is supported; the full syndrome is not.
8. **Impending withdrawal seizure or delirium from current sensors.** Not validated.
9. **Definite relapse or time-to-relapse.** Existing prospective studies are too small/heterogeneous for a deployable passive alarm.
10. **Treatment success from HRV improvement alone.** HRV is one physiological trajectory, not a complete outcome.
11. **Specific emotion.** EDA/HR/HRV indicate arousal/regulation, not unique psychological content.
12. **PSG sleep-stage pathology from IMU alone.** Wearable restlessness is not equivalent to PSG architecture.
13. **Specific SUD state for non-alcohol substances by reusing AUD mappings.** Substance-specific validation is required.

---

# 31. Recommended Implementation

## 31.1 System architecture

```text
context-corrected physiological state
↓
personal-baseline normalization
↓
target-specific AUD feature builders
↓
AUD-specific evidence models
↓
temporal persistence / event-response / trend models
↓
research evidence grade + runtime confidence
↓
AUD-specific physiological interpretation object
↓
clinical meaning/action layer
```

## 31.2 Inputs

### Cardiac

- HR mean and SD;
- RR/IBI series;
- RMSSD;
- SDNN/global variability when recording duration supports it;
- HF-HRV for controlled sleep/cue paradigms;
- optional nonlinear HRV research features.

### EDA

- tonic SCL;
- phasic activity;
- SCR peak count/rate;
- peak amplitude;
- cue-evoked delta;
- recovery slope.

### Temperature

- context-conditioned level;
- slope;
- within-person deviation;
- nocturnal behavior.

### IMU

- tremor energy 4.4–10 Hz;
- dominant tremor frequency;
- axis RMS/power/entropy/jerk;
- vector-magnitude features;
- sleep/restlessness segmentation.

### Context / history

- rest/sleep/activity state;
- withdrawal-risk/detox context;
- known alcohol cue;
- known stress challenge;
- recent alcohol-exposure label when available;
- treatment/abstinence phase;
- personal baseline distributions.

## 31.3 Targets to implement now

### Production-capable evidence layer after local validation

1. `withdrawal_tremor_evidence`
2. `withdrawal_related_physiological_activation`
3. `alcohol_cue_autonomic_reactivity` **only with known cue context**
4. `alcohol_associated_impairment_pattern` **as an investigational classifier, not BAC**
5. `autonomic_dysregulation_burden`
6. `autonomic_recovery_trend`
7. `nocturnal_autonomic_recovery_state`

### Research/guarded outputs

8. `aud_relevant_stress_reactivity`
9. `relapse_risk_associated_physiological_deviation`
10. `treatment_associated_physiological_change`

### Do not implement as standalone outputs

- physiology-only craving classifier;
- exact BAC estimator;
- relapse alarm;
- full withdrawal severity/CIWA replacement;
- AUD diagnosis.

## 31.4 Baseline comparison

Use context-conditioned robust z-scores (`median/MAD`) plus interpretable percent deviations. Maintain short and long references. Do not update baseline during suspected events.

## 31.5 Temporal modeling

- event model: cue/stress;
- state model: withdrawal/intoxication;
- trend model: recovery/chronic dysregulation;
- risk-marker model: relapse-associated physiology.

Each target has independent temporal parameters; do not apply the same rolling window to all states.

## 31.6 Evidence fusion

Start with transparent, target-specific weighted evidence and supervised component models (e.g., tremor regression). Use gradient-boosted classification for intoxication only after device-specific training. Progress to hierarchical/state-space or survival models only with sufficient prospective labeled data.

## 31.7 Confidence

Every output must include:

```text
evidence_grade: A/B/C/D
runtime_confidence: STRONG/MODERATE/WEAK/INSUFFICIENT/CONFLICTING
```

The system must retain supporting and contradictory evidence.

## 31.8 Missing data

Models may operate only on feature subsets for which they were explicitly trained/validated. Missing a primary modality reduces confidence or makes the target not evaluable; population-mean imputation is prohibited.

## 31.9 Exact output schema

```yaml
aud_physiological_state:
  generated_at: "ISO-8601"
  baseline_version: "..."

  targets:
    withdrawal:
      class: STATE
      status: SUPPORTED | WEAK | INSUFFICIENT | CONFLICTING | NOT_EVALUATED
      score: 0.0-1.0 | null
      evidence_grade: A | B | C | D
      runtime_confidence: STRONG | MODERATE | WEAK | INSUFFICIENT | CONFLICTING
      interpretation: "..."
      supporting_evidence: []
      contradictory_evidence: []
      context_basis: []
      trend:
        direction: IMPROVING | STABLE | WORSENING | UNKNOWN
        horizon: "..."
      limitations: []

    cue_reactivity: { ... }
    aud_stress_reactivity: { ... }
    intoxication_pattern: { ... }
    autonomic_dysregulation: { ... }
    recovery: { ... }
    relapse_associated_physiology: { ... }
    nocturnal_recovery: { ... }
```

## 31.10 Development order

### Phase 1 — deterministic evidence layer

- implement feature normalization/baselines;
- implement tremor spectrum and withdrawal evidence bundle;
- implement cue pre/post response vectors;
- implement longitudinal HR/HRV recovery views;
- implement nocturnal recovery metrics;
- preserve full evidence provenance.

### Phase 2 — local prospective calibration

Collect synchronized:

- clinical withdrawal scales;
- structured alcohol-cue sessions and craving ratings;
- objective BrAC/TAC during alcohol-exposure validation;
- treatment/abstinence timelines;
- 30/90/180-day alcohol outcomes.

### Phase 3 — target-specific models

- tremor regression;
- withdrawal evidence calibration;
- multimodal intoxication classifier with grouped validation;
- personalized cue-reactivity model;
- longitudinal mixed-effects/state-space recovery model.

### Phase 4 — only after external validation

- calibrated clinical probabilities;
- prospective risk models;
- downstream clinical decision integration.

## 31.11 Single consolidated design principle

The system should implement:

```text
AUD-specific physiological meaning
=
research-supported target pattern
×
known context
×
personal deviation
×
temporal behavior
×
multimodal agreement
×
validation quality
```

not:

```text
single abnormal sensor feature → AUD conclusion
```

The layer's most important engineering property is **selective inference**: it should be willing to return `INSUFFICIENT_EVIDENCE` rather than converting generic autonomic abnormalities into an AUD-specific claim.

---

# 32. Final evidence matrix

| AUD target | Physiological feature(s) | Direction/pattern | Evidence type | Individual-level utility | Recommended use | Key refs |
|---|---|---|---|---|---|---|
| Withdrawal | tremor spectrum + HR + EDA + temp | high-frequency tremor + autonomic activation in withdrawal context | clinical observational/component model | tremor: moderate-high; global state: moderate | component/state evidence + trend | [1-4,25] |
| Craving/cue reactivity | HF-HRV, HR, EDA, temp | event-locked autonomic response; direction heterogeneous | lab + small prospective | only with cue context; physiology-only craving poor | cue-reactivity event | [9-13,16,17,19,20] |
| Stress-related AUD response | HR/BP/autonomic response; HPA in literature | altered/prolonged stress response linked to craving/outcomes | lab + prospective | wearable-only limited | contextualized stress-reactivity marker | [14,26] |
| Intoxication/impairment | IMU + HR + EDA + temp | multimodal departure from sober state | controlled individual-level ML | promising pilot | impairment-pattern classifier | [8,18,21,27-29] |
| Chronic autonomic dysregulation | RMSSD/global HRV, resting HR | reduced parasympathetic variability | meta-analysis/review | longitudinal burden, not diagnosis | dysregulation trend | [5,6] |
| Abstinence/recovery | nocturnal/resting HR + HRV | HR decreases and vagal modulation improves over months | longitudinal | within-person useful | recovery trend | [23,30] |
| Treatment response | HRV, vasomotor, cue-recovery dynamics | intervention-associated changes | RCT/controlled | research endpoint | treatment-associated physiological change | [31,32] |
| Relapse-related physiology | ambulatory HRV/HR; cue-HRV; sleep | lower vagal regulation / abnormal cue response associated with poorer outcome in some cohorts | prospective | insufficient for alarm | risk-associated deviation | [9,13-15,24] |
| Sleep/recovery deterioration | nocturnal HR/HRV + sleep continuity | dose-dependent autonomic recovery suppression after alcohol; PSG sleep abnormalities linked to relapse | large real-world + PSG prospective | strong for alcohol-associated nocturnal suppression; limited for relapse | nightly recovery state | [7,22-24,29] |

---

# References

[1] White JM, Frewin CR, Kaur M, Flavel S, McGregor C. **Twenty-four hour ambulatory monitoring of tremor, sweating, skin temperature and locomotor activity in the alcohol withdrawal syndrome.** *Clinical Autonomic Research.* 1994;4(1-2):15-18. DOI: 10.1007/BF01828833. PMID: 8054832. URL: https://pubmed.ncbi.nlm.nih.gov/8054832/

[2] Bär KJ, Boettger MK, Neubauer R, et al. **Heart rate variability and sympathetic skin response in male patients suffering from acute alcohol withdrawal syndrome.** *Alcoholism: Clinical and Experimental Research.* 2006;30(9):1592-1598. DOI: 10.1111/j.1530-0277.2006.00191.x. PMID: 16930222. URL: https://pubmed.ncbi.nlm.nih.gov/16930222/

[3] Carver S, Norouzi N, Bromberg S, et al. **Time-frequency visualization of alcohol withdrawal tremors.** *2014 36th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC).* 2014:5474-5477. DOI: 10.1109/EMBC.2014.6944865. PMID: 25571233. URL: https://pubmed.ncbi.nlm.nih.gov/25571233/

[4] Aarabi P, Norouzi N, Dear T, et al. **A quantitative evaluation of alcohol withdrawal tremors.** *2015 37th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC).* 2015:6215-6218. DOI: 10.1109/EMBC.2015.7319812. PMID: 26737712. URL: https://pubmed.ncbi.nlm.nih.gov/26737712/

[5] Cheng YC, Huang YC, Huang WL. **Heart rate variability as a potential biomarker for alcohol use disorders: A systematic review and meta-analysis.** *Drug and Alcohol Dependence.* 2019;204:107502. DOI: 10.1016/j.drugalcdep.2019.05.030. PMID: 31494439. URL: https://pubmed.ncbi.nlm.nih.gov/31494439/

[6] Ralevski E, Petrakis I, Altemus M. **Heart rate variability in alcohol use: A review.** *Pharmacology Biochemistry and Behavior.* 2019;176:83-92. DOI: 10.1016/j.pbb.2018.12.003. PMID: 30529588. URL: https://pubmed.ncbi.nlm.nih.gov/30529588/

[7] Pietilä J, Helander E, Korhonen I, Myllymäki T, Kujala UM, Lindholm H. **Acute Effect of Alcohol Intake on Cardiovascular Autonomic Regulation During the First Hours of Sleep in a Large Real-World Sample of Finnish Employees: Observational Study.** *JMIR Mental Health.* 2018;5(1):e23. DOI: 10.2196/mental.9519. PMID: 29549064. PMCID: PMC5878366. URL: https://pubmed.ncbi.nlm.nih.gov/29549064/

[8] Kaczor EE, Golleru M, Carter DC, Painter OB, et al. **Detecting Ethanol Intoxication and Impairment Using Wearable Biosensors.** *Proceedings of the Annual Hawaii International Conference on System Sciences.* 2026:3532-3542. PMID: 41550855. PMCID: PMC12805382. URL: https://pubmed.ncbi.nlm.nih.gov/41550855/

[9] Garland EL, Franken IHA, Howard MO. **Cue-Elicited Heart Rate Variability and Attentional Bias Predict Alcohol Relapse Following Treatment.** *Psychopharmacology.* 2012;222(1):17-26. DOI: 10.1007/s00213-011-2618-4. PMID: 22203318. PMCID: PMC3346884. URL: https://pubmed.ncbi.nlm.nih.gov/22203318/

[10] Quintana DS, Guastella AJ, McGregor IS, Hickie IB, Kemp AH. **Heart rate variability predicts alcohol craving in alcohol dependent outpatients: further evidence for HRV as a psychophysiological marker of self-regulation.** *Drug and Alcohol Dependence.* 2013;132(1-2):395-398. DOI: 10.1016/j.drugalcdep.2013.02.025. PMID: 23601925. URL: https://pubmed.ncbi.nlm.nih.gov/23601925/

[11] Garland EL, Franken IHA, Sheetz JJ, Howard MO. **Alcohol attentional bias is associated with autonomic indices of stress-primed alcohol cue-reactivity in alcohol-dependent patients.** *Experimental and Clinical Psychopharmacology.* 2012;20(3):225-235. DOI: 10.1037/a0027199. PMID: 22329555. URL: https://pubmed.ncbi.nlm.nih.gov/22329555/

[12] Glautier S, Drummond DC. **Alcohol dependence and cue reactivity.** *Journal of Studies on Alcohol.* 1994;55(2):224-229. DOI: 10.15288/jsa.1994.55.224. PMID: 8189743. URL: https://pubmed.ncbi.nlm.nih.gov/8189743/

[13] Witteman J, Post H, Tarvainen M, et al. **Cue reactivity and its relation to craving and relapse in alcohol dependence: a combined laboratory and field study.** *Psychopharmacology.* 2015. DOI: 10.1007/s00213-015-4027-6. PMID: 26257163. PMCID: PMC4562995. URL: https://pubmed.ncbi.nlm.nih.gov/26257163/

[14] Sinha R, Fox HC, Hong KIA, Hansen J, Tuit K, Kreek MJ. **Effects of adrenal sensitivity, stress- and cue-induced craving, and anxiety on subsequent alcohol relapse and treatment outcomes.** *Archives of General Psychiatry.* 2011;68(9):942-952. DOI: 10.1001/archgenpsychiatry.2011.49. PMID: 21536969. PMCID: PMC3668981. URL: https://pubmed.ncbi.nlm.nih.gov/21536969/

[15] Eddie D, Wieman S, Pietrzak A, Zhai X. **In natura heart rate variability predicts subsequent alcohol use in individuals in early recovery from alcohol use disorder.** *Addiction Biology.* 2023;28(8):e13306. DOI: 10.1111/adb.13306. PMID: 37500488. PMCID: PMC10433643. URL: https://pubmed.ncbi.nlm.nih.gov/37500488/

[16] van Lier HG, Noordzij ML, Pieterse ME, et al. **An ideographic study into physiology, alcohol craving and lapses during one hundred days of daily life monitoring.** *Addictive Behaviors Reports.* 2022;16:100443. DOI: 10.1016/j.abrep.2022.100443. PMID: 35855973. PMCID: PMC9287639. URL: https://pubmed.ncbi.nlm.nih.gov/35855973/

[17] Alinia P, Sah RK, McDonell M, Pendry P, Parent S, Ghasemzadeh H, Cleveland MJ. **Associations Between Physiological Signals Captured Using Wearable Sensors and Self-reported Outcomes Among Adults in Alcohol Use Disorder Recovery: Development and Usability Study.** *JMIR Formative Research.* 2021;5(7):e27891. DOI: 10.2196/27891. PMID: 34287205. PMCID: PMC8339978. URL: https://pubmed.ncbi.nlm.nih.gov/34287205/

[18] Bau PFD, Moraes RS, Bau CHD, Ferlin EL, Rosito GA, Fuchs FD. **Acute ingestion of alcohol and cardiac autonomic modulation in healthy volunteers.** *Alcohol.* 2011;45(2):123-129. DOI: 10.1016/j.alcohol.2010.08.011. PMID: 21131160. URL: https://pubmed.ncbi.nlm.nih.gov/21131160/

[19] Szegedi A, Lörch B, Scheurich A, Ruppe A, Hautzinger M, Wetzel H. **Cue exposure in alcohol dependent patients: preliminary evidence for different types of cue reactivity.** *Journal of Neural Transmission.* 2000;107(6):721-730. DOI: 10.1007/s007020070073. PMID: 10943912. URL: https://pubmed.ncbi.nlm.nih.gov/10943912/

[20] Reid MS, Flammino F, Starosta A, Palamar J, Franck J. **Physiological and subjective responding to alcohol cue exposure in alcoholics and control subjects: evidence for appetitive responding.** *Journal of Neural Transmission.* 2006;113(10):1519-1535. DOI: 10.1007/s00702-005-0439-5. PMID: 16604310. URL: https://pubmed.ncbi.nlm.nih.gov/16604310/

[21] Malpas SC, Robinson BJ, Maling TJ. **Mechanism of ethanol-induced vasodilation.** *Journal of Applied Physiology.* 1990;68(2):731-734. DOI: 10.1152/jappl.1990.68.2.731. PMID: 2318781. URL: https://pubmed.ncbi.nlm.nih.gov/2318781/

[22] Melby K, Fasmer OB, Henriksen TE, et al. **Actigraphy assessment of motor activity and sleep in patients with alcohol withdrawal syndrome and the effects of intranasal oxytocin.** *PLoS ONE.* 2020;15(2):e0228700. DOI: 10.1371/journal.pone.0228700. PMID: 32053696. PMCID: PMC7018062. URL: https://pubmed.ncbi.nlm.nih.gov/32053696/

[23] de Zambotti M, Willoughby AR, Baker FC, et al. **Cardiac autonomic function during sleep: effects of alcohol dependence and evidence of partial recovery with abstinence.** *Alcohol.* 2015;49(4):409-415. DOI: 10.1016/j.alcohol.2014.07.023. PMID: 25957854. PMCID: PMC4465836. URL: https://pubmed.ncbi.nlm.nih.gov/25957854/

[24] Brower KJ, Aldrich MS, Hall JM. **Polysomnographic and subjective sleep predictors of alcoholic relapse.** *Alcoholism: Clinical and Experimental Research.* 1998;22(8):1864-1871. PMID: 9835309. URL: https://pubmed.ncbi.nlm.nih.gov/9835309/

[25] Bär KJ, Boettger MK, Boettger S, Grotelüschen M, Neubauer R, Jochum T, Baier V, Sauer H, Voss A. **Reduced baroreflex sensitivity in acute alcohol withdrawal syndrome and in abstained alcoholics.** *Drug and Alcohol Dependence.* 2006;85(1):66-74. DOI: 10.1016/j.drugalcdep.2006.03.014. PMID: 16650658. URL: https://pubmed.ncbi.nlm.nih.gov/16650658/

[26] Sinha R, Fox HC, Hong KIA, Bergquist K, Bhagwagar Z, Siedlarz KM. **Enhanced negative emotion and alcohol craving, and altered physiological responses following stress and cue exposure in alcohol dependent individuals.** *Neuropsychopharmacology.* 2009. DOI: 10.1038/npp.2008.78. PMID: 18563062. PMCID: PMC2734452. URL: https://pubmed.ncbi.nlm.nih.gov/18563062/

[27] Spaak J, Tomlinson G, McGowan CL, et al. **Dose-related effects of red wine and alcohol on heart rate variability.** *American Journal of Physiology-Heart and Circulatory Physiology.* 2010. DOI: 10.1152/ajpheart.00700.2009. PMID: 20418480. URL: https://pubmed.ncbi.nlm.nih.gov/20418480/

[28] Buckman JF, Eddie D, Vaschillo EG, Vaschillo B, Garcia A, Bates ME. **Immediate and Complex Cardiovascular Adaptation to an Acute Alcohol Dose.** *Alcoholism: Clinical and Experimental Research.* 2015;39(12):2334-2344. DOI: 10.1111/acer.12912. PMID: 26614647. PMCID: PMC4971776. URL: https://pubmed.ncbi.nlm.nih.gov/26614647/

[29] Sagawa Y, Kondo H, Matsubuchi N, et al. **Alcohol has a dose-related effect on parasympathetic nerve activity during sleep.** *Alcoholism: Clinical and Experimental Research.* 2011;35(11):2093-2100. DOI: 10.1111/j.1530-0277.2011.01558.x. PMID: 21848959. URL: https://pubmed.ncbi.nlm.nih.gov/21848959/

[30] Yokoyama A, Takagi T, Akai J, Hayashi E, Muramatsu T, Maruyama K, Eboshida A, Takagi S, Kono H. **Consecutive changes of R-R interval variation on ECG after abstinence in male alcoholics.** *Arukoru Kenkyuto Yakubutsu Ison (Japanese Journal of Alcohol Studies & Drug Dependence).* 1989;24(1):36-42. PMID: 2751485. URL: https://pubmed.ncbi.nlm.nih.gov/2751485/

[31] Penzlin AI, Siepmann T, Illigens BMW, Weidner K, Siepmann M. **Heart rate variability biofeedback in patients with alcohol dependence: a randomized controlled study.** *Neuropsychiatric Disease and Treatment.* 2015;11:2619-2627. DOI: 10.2147/NDT.S84798. PMID: 26557753. PMCID: PMC4624058. URL: https://pubmed.ncbi.nlm.nih.gov/26557753/

[32] Logge WB, Baillie AJ, Haber PS, Morley KC. **Baclofen modulates cardiovascular responses to appetitive cues in treatment-seeking alcohol use disorder individuals.** *Human Psychopharmacology.* 2020;35(2):e2722. DOI: 10.1002/hup.2722. PMID: 32045501. URL: https://pubmed.ncbi.nlm.nih.gov/32045501/

[33] Koskinen P, Virolainen J, Kupari M. **Acute alcohol intake decreases short-term heart rate variability in healthy subjects.** *Clinical Science.* 1994;87(2):225-230. DOI: 10.1042/cs0870225. PMID: 7924168. URL: https://pubmed.ncbi.nlm.nih.gov/7924168/

[34] de Zambotti M, Forouzanfar M, Javitz H, Goldstone A, Claudatos S, Alschuler V, Baker FC, Colrain IM. **Impact of evening alcohol consumption on nocturnal autonomic and cardiovascular function in adult men and women: a dose-response laboratory investigation.** *Sleep.* 2021;44(1):zsaa135. DOI: 10.1093/sleep/zsaa135. PMID: 32663278. PMCID: PMC7819834. URL: https://pubmed.ncbi.nlm.nih.gov/32663278/

---

## Research limitations of this document

This is an implementation-oriented evidence synthesis, not a PRISMA systematic review. It prioritizes direct AUD evidence, original physiological studies, prospective outcomes, systematic reviews/meta-analyses, and ambulatory wearable studies. The absence of a feature from the implementation set should not be read as proof that no relevant study exists; it means the evidence identified was insufficiently direct, specific, or operational to justify the feature in this layer.

The most important validation requirement is prospective device-specific testing in the intended clinical population. Published group-level effect sizes and pilot classifiers should be used to choose **what to measure and how to structure inference**, not copied as production thresholds.

