# Personalized Longitudinal Physiological Modeling Solution

**Technical research/design document for patient-specific physiological baselines, deviation detection, and longitudinal trajectories in an AUD/SUD monitoring system**

**Version:** 1.0  
**Date:** 2026-10-16  
**Primary design question:** *Is this physiological state abnormal for humans generally, or abnormal for this patient relative to their own baseline, circadian pattern, recent history, and longitudinal recovery trajectory?*

---

## Executive summary

The personalized longitudinal layer should **not** be implemented as a single rolling average placed after multimodal fusion. The evidence supports a **hybrid, multi-timescale, context-conditioned patient model** that operates at several points in the pipeline:

1. **Before or during fusion:** convert trustworthy modality-specific features into patient-relative, context-relative residuals and uncertainty estimates.
2. **During fusion:** combine those normalized residuals with modality reliability and covariance information so that correlated modalities are not double counted.
3. **After fusion:** maintain short-, medium-, and long-horizon patient states; detect acute deviations, sustained deviations, candidate change points, and longitudinal trends.
4. **In parallel:** preserve population/reference abnormalities and hard safety constraints. Personalization must never make an objectively concerning value “safe” merely because it has become common for that patient.
5. **Downstream:** pass personalized deviations, trajectories, and confidence to the AUD-specific interpretation layer. This layer itself must not diagnose withdrawal, predict imminent relapse, or prescribe treatment.

Longitudinal wearable studies show why this is necessary. Daily resting heart rate is much more stable within many individuals than between individuals, and population-wide “normal” ranges can hide large patient-specific excursions [1,2]. Personalized activity- and history-normalized models have detected physiological departures before overt illness, and real-time wearable systems have demonstrated practical streaming baselines and sequential alerting [1,3,4]. Circadian variation materially changes expected resting HR, skin temperature, EDA, respiratory rate, and sleep physiology, so the expectation must be a function of **patient × context × time-of-day**, not one global patient mean [5,10,11,13–15]. HRV exhibits especially large inter-individual variation and methodological sensitivity, which argues for standardized acquisition plus within-person normalization rather than naive universal thresholds [8]. EDA also contains stable individual response phenotypes but strong sleep/state dependence [14–17]. Nocturnal respiratory rate is another strong candidate for patient-specific monitoring because inter-person variation can exceed within-person variation and longitudinal departures from a personal baseline can be informative [11,12].

The recommended implementation is therefore a **gated hierarchical model**:

```text
population/reference model + absolute safety bounds
                    │
                    ├──────────────────────────────────────┐
                    │                                      │
trusted physiological feature                     population status
+ context/confound metadata                               │
+ patient history                                         │
        │                                                  │
        ▼                                                  │
stable regime baseline                                    │
+ circadian model                                          │
+ context-conditioned offset                              │
+ slow longitudinal state                                 │
        │                                                  │
        ▼                                                  │
expected personal physiology                              │
        │                                                  │
current observation ───────► personal residual / percentile│
        │                                                  │
        ▼                                                  │
acute detector + sustained detector + change-point model   │
        │                                                  │
        ▼                                                  │
patient-relative state ────────────────────────────────────┘
        │
        ▼
reliability-aware personalized multimodal fusion
        │
        ▼
24 h / 7 d / ~28 d / regime-level trajectory
        │
        ▼
AUD-specific physiological interpretation downstream
```

The design deliberately separates four ideas that are often conflated:

- **population abnormality:** abnormal compared with reference populations or clinical limits;
- **personal abnormality:** unexpected for this patient in the present state/context;
- **persistent physiological change:** a deviation that lasts beyond an acute episode;
- **baseline-regime change:** evidence that the patient's stable physiology itself has moved to a new level.

The system should maintain all four rather than collapse them into a single score.

### Most important implementation decisions

- Use **robust distributions** (median, MAD, empirical quantiles/prediction intervals) by default rather than assuming Gaussian physiology.
- Model **sleep and awake-rest separately** for HR/HRV/EDA/temperature/respiration when data support it.
- Model circadian expectation with a **streaming harmonic/cosinor model** (24 h fundamental; add 12 h only if it demonstrably improves prediction) [5].
- Calculate both a **robust standardized residual** and an **empirical personal percentile**.
- Use **CUSUM or an equivalent sequential detector** for persistent deviation, but tune the detector to an acceptable false-alert rate from stable personal data rather than importing disease-specific thresholds [3,4,21].
- Never immediately overwrite the long-term baseline after a change point. Create a **candidate new regime**, accumulate stable evidence, and retain the previous regime.
- Maintain at least a **recent reference**, a **stable long-term regime reference**, and a **historical recovery trajectory**. A 7-night HR reference is supported for one overnight-RHR implementation, while a ~28-day individualized background has precedent in wearable anomaly detection; these horizons are **not universal constants for every feature** [3,4].
- During cold start, blend population priors with sparse personal data and explicitly emit **low baseline confidence**.
- Freeze stable-baseline updates during poor signal, unresolved confounding, acute anomalies, persistent-change detection, treatment/withdrawal transitions, suspected device changes, or other non-stable periods.
- Keep a **short-term adaptive reference** separate from the long-term stable baseline. The short-term reference can follow the patient; it must not redefine “normal.”
- Preserve **absolute safety/reference status in parallel** because purely adaptive baselines can normalize gradual deterioration [7,20].
- Treat AUD-specific recovery findings as downstream evidence. Existing longitudinal AUD literature supports autonomic recovery during abstinence in some cohorts, particularly HRV/vagal indices, but trajectories are heterogeneous and evidence for real-time relapse prediction is insufficient [25–30,38].

---

# 1. Scope, boundary, and placement in the system

## 1.1 Inputs assumed to be available

This layer receives outputs that have already passed earlier processing:

```text
trusted physiological feature
+ feature quality / measurement uncertainty
+ decomposition output where applicable
+ confound state
+ context state
+ timestamp and recent history
+ optional fused autonomic state
+ known treatment / clinical phase metadata when available
```

It should not redo raw-signal quality assessment, generic motion-artifact rejection, tonic/phasic decomposition, or broad context recognition.

## 1.2 Outputs this layer is allowed to produce

Examples of valid outputs:

- “Resting HR is at the 99th personal percentile for this patient's nighttime-rest distribution.”
- “RMSSD is 2.6 robust personal standard units below expected sleep/rest value.”
- “The deviation has persisted across three consecutive valid nightly summaries.”
- “A candidate persistent baseline shift is present; stable baseline update is frozen.”
- “Seven-day autonomic trajectory is worsening relative to the previous stable regime.”
- “The current value is personally unusual but remains inside a population reference interval.”
- “The current value is common for this patient but violates an absolute safety constraint.”

## 1.3 Outputs this layer must not produce

It must not independently emit:

```text
WITHDRAWAL_CONFIRMED
RELAPSE_IMMINENT
INTOXICATION_CONFIRMED
MEDICATION_REQUIRED
TREATMENT_CHANGE_REQUIRED
```

Those conclusions require the downstream AUD-specific/clinical layer.

## 1.4 Personalization belongs at multiple points

A strictly serial architecture that waits until after multimodal fusion to personalize wastes information. The recommended placement is:

| Pipeline location | Personalization | Why |
|---|---|---|
| Feature extraction output | patient/context expected value and residual | makes HR, HRV, EDA, temperature, RR comparable as deviations rather than incompatible raw units |
| Multimodal fusion | reliability, response phenotype, covariance | prevents a weak/nonresponsive modality from dominating and avoids double counting correlated channels |
| Longitudinal layer | multi-horizon baseline, change point, trend | distinguishes acute episodes from persistent regime shifts |
| AUD evidence layer | patient-relative evidence magnitude and uncertainty | lets AUD models reason about “large change for this patient” without the personalization layer making the diagnosis |
| Safety/reference path | **no replacement** by personalization | absolute/reference concerns remain visible in parallel [7,20] |

Personalization of raw hardware-quality thresholds can also be useful in some devices, but that is a separate signal-trustworthiness problem and is not developed here.

---

# 2. Evidence base: inter-person versus intra-person physiology

The basic justification for personalization is not that every variable needs a bespoke ML model. It is that many physiological features have large between-person differences while remaining meaningfully structured within a person, especially when acquisition context is standardized.

## 2.1 Feature-level evidence synthesis

| Feature | Inter-person variability | Within-person stability | Circadian/state dependence | Personalization value | Key evidence |
|---|---|---|---|---|---|
| Resting HR | High; individual long-term normal ranges differ substantially | Often comparatively stable, but can shift over days/weeks | Strong; daytime/nighttime and season matter | **Very high** | 92,457-person longitudinal cohort; personalized wearable studies [1,2,10] |
| ECG HRV (e.g., RMSSD, SDNN) | Very high; spectral measures especially variable | Useful when protocol/context standardized; day-to-day variability remains material | Strong effects of sleep, posture, respiration, time, stress | **Very high**, but acquisition must be standardized | systematic review of 44 studies/21,438 adults; monitoring literature [8,9] |
| PPG pulse rate | Mirrors HR when measurement trustworthy | Good at rest; context/device dependent | Same circadian/rest dependence as HR | **High** | [2,10,18] |
| PPG-derived PRV | Between-person differences are real, but measurement equivalence varies | Can be reliable at rest; agreement with ECG worsens under activity/stress | strong context dependence | **Conditional**; only after PRV-vs-HRV validity constraints | [18,19] |
| EDA tonic level (SCL) | Large person/site/device differences | individual lability/responding can be stable | strong sleep-stage, temperature and context effects | **Very high** for within-person use | [14–17] |
| EDA phasic responsiveness | clear responder/non-responder and lability differences | response phenotype shows retest stability | event- and state-dependent | **High** when event/context matched | [14–17,23,24] |
| Peripheral/wrist temperature | absolute value strongly affected by environment, site, device | personal rhythm can be stable under comparable conditions | **very strong circadian/sleep relation** | **High for residual/trend**, weak for universal absolute interpretation | [1,13] |
| Nocturnal respiratory rate | population differences meaningful | often relatively stable within person during sleep | sleep/state dependent; illness and other conditions shift it | **High** | wearable validation/longitudinal cohorts [11,12] |
| Awake respiratory rate | high behavioral/context influence | less stable than standardized nocturnal RR | posture, speech, emotion, activity, voluntary control | **Context-conditioned** | [11,12] |
| Respiratory variability | method dependent; normative interpretation less standardized | may be useful longitudinally if measurement protocol fixed | sleep stage/context dependent | **Trend-first** | [11,12] |
| Sleep/autonomic summaries | substantial individual sleep architecture and autonomic differences | nightly variability, but longitudinal trajectories measurable | inherently sleep-stage/circadian dependent | **High for longitudinal trend** | [14,15,28,32] |
| Recovery kinetics after challenge/event | large individual response phenotypes | can become repeatable with repeated matched events | depends on event/context | **Potentially high**, but requires repeated comparable events | [23,24] |

### Interpretation

The table does **not** imply “population information is useless.” It implies that for continuous monitoring, a single global threshold usually answers a different question from a personal deviation. Quer et al. found major inter-individual differences in long-term resting HR despite much smaller within-person variability for many participants [2]. Nunan et al. documented very large inter-individual dispersion in short-term HRV values and substantial methodological dependence [8]. Harrington et al. explicitly observed greater inter-patient than intra-patient variance for nocturnal respiratory rate, motivating patient-specific references [12]. EDA literature similarly shows stable responder/lability phenotypes while sleep studies show profound state dependence [14–17].

---

# 3. What should and should not be personalized

Personalization should be **feature- and purpose-specific**, not automatic.

## 3.1 Classification

### A. Population/reference threshold suitable — keep in parallel

Use absolute/reference information where clinically or physiologically meaningful and where the measurement actually corresponds to the referenced quantity. Examples include extreme HR or RR values and other recognized vital-sign limits supplied by the clinical layer. These are **safety/reference constraints**, not substitutes for the personal model.

Caution: wrist/peripheral temperature is not equivalent to core body temperature, and PPG PRV is not automatically equivalent to ECG HRV [13,18]. Do not attach a population clinical limit to the wrong measurement construct.

### B. Personal-baseline preferred

- resting HR;
- ECG HRV summaries under standardized rest/sleep conditions;
- nocturnal RR;
- stable PPG pulse features under matched conditions;
- long-term sleep/autonomic summaries.

### C. Context-conditioned personal baseline required

- HR outside tightly standardized resting periods;
- HRV across sleep/awake/posture/respiration states;
- EDA tonic and phasic features;
- peripheral temperature;
- awake RR;
- PPG amplitude/perfusion-related features.

### D. Trend-first features

- circadian amplitude/acrophase;
- recovery slope over days/weeks;
- recovery time after repeated matched events;
- some respiratory variability metrics;
- PPG amplitude/perfusion trajectories when absolute calibration is weak.

### E. Insufficient evidence for strong AUD-specific personalization rules

- patient-specific EDA threshold that predicts relapse;
- patient-specific peripheral-temperature threshold that diagnoses withdrawal;
- a universal percent change in HRV that confirms craving;
- universal cross-modal weights that identify impending relapse.

These may be research targets, but current evidence does not justify hard coding them.

---

# 4. Definitions: do not use one universal baseline

For each patient `p`, physiological feature `f`, context `c`, and time `t`, define several references.

## 4.1 Population/reference model

```text
P_population(x_f | age/sex/etc. when justified)
```

Purpose: contextualize the observation against external reference information and safety limits. It is **never overwritten** by patient adaptation.

## 4.2 Stable long-term personal regime baseline

```text
B_stable[p,f,r]
```

where `r` is a stable regime identifier. This estimates what is typical for the patient during a stable period such as a documented stable-recovery phase. It changes slowly and only through gated updates.

## 4.3 Context-conditioned personal baseline

```text
B_context[p,f,r,c]
```

Examples:

- awake resting;
- sleeping;
- standing;
- post-activity recovery;
- supervised clinical rest.

The context model should be as coarse as necessary to obtain enough samples. Do not create dozens of tiny strata that never accumulate sufficient data.

## 4.4 Circadian expectation

```text
C[p,f](phase_of_day)
```

This handles systematic time-of-day variation. RHR and peripheral temperature show strong 24-h structure [5,10,13]; EDA and respiration also vary materially across sleep/wake and sleep stages [11,14,15].

## 4.5 Recent adaptive reference

```text
B_recent[p,f]
```

A short-horizon estimator used to track the *current operating level*. This reference may adapt faster than the stable baseline, but it must **not** redefine health or normality. It is useful for detecting whether the present state is changing relative to yesterday even while the long-term baseline remains fixed.

## 4.6 Longitudinal trajectory

```text
T[p,f](day)
```

A level + slope + uncertainty model over daily/weekly robust summaries. It answers whether physiology is improving, stable, or deteriorating relative to prior stable states.

## 4.7 Recovery reference(s)

For AUD monitoring, retain multiple historical anchors rather than destructively replacing the past:

```text
initial_or_pre_treatment_reference
current_stable_regime_reference
recent_reference
best_stable_recovery_reference   # optional, only after sufficient evidence
```

Early-withdrawal physiology should never silently become the only baseline against which later recovery is judged.

---

# 5. Mathematical formulation

For a trusted feature observation `x_{p,f}(t)`:

```text
x_{p,f}(t)
= m_{p,f,r(t)}
+ C_{p,f}(φ_t)
+ β_{p,f}ᵀ c_t
+ L_{p,f}(t)
+ a_{p,f}(t)
+ ε_{p,f}(t)
```

where:

- `m_{p,f,r(t)}` = stable level for patient `p`, feature `f`, regime `r`;
- `C_{p,f}(φ_t)` = patient-specific circadian component;
- `c_t` = recognized context covariates/state indicators;
- `β` = context offsets/effects;
- `L(t)` = slow longitudinal trend;
- `a(t)` = acute/sustained physiological departure that should **not** immediately enter the baseline;
- `ε(t)` = residual measurement/biological variation.

The expected personal state is:

```text
μ̂_{p,f}(t)
= m̂_{p,f,r(t)} + Ĉ_{p,f}(φ_t) + β̂ᵀc_t + L̂_{p,f}(t)
```

and the raw personal deviation is:

```text
d_{p,f}(t) = x_{p,f}(t) - μ̂_{p,f}(t)
```

For robust standardization:

```text
s_{p,f,c} = max(1.4826 × MAD(residual_history), scale_floor_f)

z^robust_{p,f}(t) = d_{p,f}(t) / s_{p,f,c}
```

The scale floor prevents unstable scores when historical variation is artificially tiny.

Also compute the empirical residual percentile:

```text
q_{p,f}(t) = F̂_{p,f,c}(d_{p,f}(t))
```

The system should retain **all three**:

```text
absolute_delta
robust_z
personal_percentile
```

because each is useful for a different reason. A `+15 bpm` HR change is clinically interpretable; a `+3.0` robust personal score shows unusualness; the percentile communicates a distribution-free rank.

## 5.1 Population and personal abnormality are separate axes

```text
population_status = A_pop(x)
personal_status   = A_personal(x | patient, context, time, history)
```

Do not compute:

```text
one_final_abnormality = blend(population, personal)
```

unless a downstream validated model explicitly requires that blend. The hybrid-baseline literature warns that adaptive individual baselines can track gradual deterioration and thereby lose sensitivity to absolute risk [7].

---

# 6. Baseline acquisition and cold start

No single evidence-based initialization duration exists for every physiological feature. The system therefore needs **feature-specific baseline readiness** rather than one global “baseline complete” flag.

## 6.1 General cold-start policy

### First minutes

Use only:

- population/reference information;
- current trusted context;
- immediate local history;
- very wide personal uncertainty.

Output:

```text
baseline_confidence = VERY_LOW
personal_deviation_status = NOT_ESTABLISHED
```

Do not call a value “abnormal for this patient” from a few minutes of data.

### First hours

Build provisional context-specific empirical summaries where enough valid samples exist. These can support *relative-to-this-session* comparisons but should remain distinct from the stable patient baseline.

### First day

Create provisional sleep/awake-rest distributions if both contexts are observed, but retain wide uncertainty. A single atypical day must not define the patient.

### First several days

Begin feature-specific baselines. Evidence from wearable HRV monitoring suggests that three or more standardized measurements per week can approximate weekly HRV changes in athletic monitoring, but that result is methodological transfer evidence rather than an AUD clinical baseline rule [9]. For overnight RHR, Alavi et al. found that more than 80% of users reached an overnight baseline within ±2 bpm of their three-month baseline after seven nights [4]. That is a useful implementation anchor for **overnight RHR**, not a universal seven-day rule.

### One to four weeks

Circadian/context coverage and long-horizon distributions become more stable. Mishra et al. used individualized historical baselines in wearable anomaly detection, and the Stanford real-time work used longer personal backgrounds alongside streaming overnight references [3,4]. A ~28-day stable history is therefore reasonable as a **mature operational reference horizon** for continuously sampled features when available, but it remains an engineering choice that must be validated feature by feature.

## 6.2 Recommended baseline-readiness object

```yaml
baseline_readiness:
  resting_hr_night:
    status: MATURE
    valid_days: 18
    valid_nights: 16
    context_coverage: 0.92
    circadian_coverage: 0.81
    stability: HIGH
  rmssd_sleep:
    status: PROVISIONAL
    valid_days: 6
    valid_windows: 31
    context_coverage: 0.68
    stability: MEDIUM
  eda_sleep:
    status: PROVISIONAL
    valid_nights: 4
    sleep_stage_coverage: UNKNOWN
```

Avoid one global `baseline_ready=true` because the HR model may be mature while EDA or respiration remains sparse.

## 6.3 Population prior + personal posterior

A hierarchical Bayesian implementation is attractive during cold start:

```text
population prior
       ↓
patient observations
       ↓
patient posterior distribution
```

However, a full Bayesian model is not required for the first production version. The practical equivalent is:

1. initialize expected values/ranges from appropriately matched population/reference data when valid;
2. assign large uncertainty;
3. progressively shift weight toward personal estimates as valid personal observations accumulate;
4. preserve absolute/reference limits separately.

Conceptually:

```text
μ̂_personal = w_personal × μ_personal_data
            + (1 - w_personal) × μ_population_prior
```

where `w_personal` is driven by data sufficiency and stability, not simply elapsed time.

The posterior/uncertainty formulation is superior when a probabilistic stack already exists because it naturally represents sparse-history uncertainty. A robust empirical implementation is simpler and is preferable for the initial real-time system.

---

# 7. Statistical representation of personal baselines

## 7.1 Default representation

Physiological variables are frequently skewed, heavy-tailed, multimodal by context, or contaminated by occasional outliers. Therefore the default baseline record should contain:

```yaml
baseline_distribution:
  median:
  mad:
  p05:
  p10:
  p25:
  p50:
  p75:
  p90:
  p95:
  n_valid_windows:
  n_valid_days:
  last_update:
```

Add mean/SD only when the feature distribution and downstream model justify them.

## 7.2 Feature-specific transformations

### HR / resting HR

Raw bpm is interpretable and usually suitable for direct robust modeling. Retain absolute bpm difference alongside normalized residual.

### HRV

Many HRV indices, particularly RMSSD and spectral power measures, are right-skewed. Use a log transform where appropriate:

```text
y = ln(RMSSD)
```

and build the personal model in log space. Report both the normalized log-domain residual and the original-unit value. The large inter-individual variability documented in HRV literature makes protocol consistency especially important [8].

### EDA tonic level

SCL can be heavily person/device/site dependent and skewed. Prefer:

- robust median/MAD;
- empirical quantiles;
- log transform if the implementation has strictly positive values and model diagnostics support it.

### EDA phasic measures

SCR counts/frequency may be zero-inflated; amplitudes are skewed. Do not assume Gaussianity. Use empirical distributions, count models, or robust quantiles depending on the derived feature.

### Peripheral temperature

Model the residual from the patient's circadian/context curve rather than a global mean. Absolute wrist temperature is strongly affected by distal vasomotor state and environment [13].

### Respiratory rate

For nocturnal RR, robust nightly central tendency and within-night dispersion are appropriate. Population ranges remain useful context, but personal deviations are important [11,12].

## 7.3 When mixture distributions are warranted

A mixture model can be useful when the feature has genuine latent states that cannot be captured by recognized context, but explicit context stratification is preferable when the state is known. For example:

```text
awake-rest HR distribution
sleep HR distribution
```

is better than fitting an opaque two-component Gaussian mixture to a pooled HR distribution.

---

# 8. Context-conditioned baselines

A global patient average is insufficient because the same patient has physiologically different expected states while sleeping, resting awake, standing, recovering from activity, or engaged in activity. Recent ambulatory work on “additional heart rate” likewise demonstrates the usefulness of subtracting expected HR attributable to physical activity and posture before interpreting residual cardiac activation [22].

## 8.1 Baseline selection hierarchy

At each valid observation:

```text
1. exact context-specific personal baseline if mature
2. broader parent context personal baseline if exact context sparse
3. time-of-day personal baseline
4. global personal baseline
5. population/reference prior
```

Each fallback increases uncertainty.

## 8.2 Minimum recommended contexts

For the first deployable version:

```text
SLEEP
AWAKE_REST
UPRIGHT_LOW_ACTIVITY
POST_ACTIVITY_RECOVERY
OTHER_VALID_CONTEXT
```

Do not initially create separate baselines for every semantic context such as “working,” “socializing,” or “watching television.” Those belong only if the upstream context layer can identify them reliably and enough patient-specific samples accumulate.

## 8.3 Feature-context matrix

| Feature | Sleep baseline | Awake-rest | Upright/low activity | Post-activity | Time-of-day correction |
|---|---:|---:|---:|---:|---:|
| HR/RHR | **yes** | **yes** | yes | yes | **yes** |
| ECG HRV | **yes** | **yes** | conditional | conditional | yes |
| PRV | yes, if validated | yes, if validated | generally avoid unless quality proven | generally avoid | yes |
| EDA tonic | **yes** | **yes** | yes | yes | yes |
| EDA phasic | event/state specific | event/state specific | event/state specific | event/state specific | secondary |
| wrist temperature | **yes** | **yes** | yes | yes | **yes** |
| nocturnal RR | **yes** | n/a | n/a | n/a | sleep-time model sufficient initially |
| awake RR | n/a | **yes** | conditional | conditional | secondary |

The upstream confound-aware layer should already have dealt with motion/posture/other confounding where possible. The personalized layer uses the resulting context labels to choose the correct expectation.

---

# 9. Circadian modeling

## 9.1 Why it is mandatory

Resting HR changes across the 24-hour cycle [5,10]. Wrist temperature has a prominent circadian pattern, rises around bedtime, and falls after awakening [13]. EDA during sleep is not simply “low arousal”: wrist EDA peaks and storms are concentrated in particular NREM stages [14,15]. Respiratory rate is also best interpreted under standardized nocturnal conditions [11,12].

Therefore:

```text
expected_personal_value = f(patient, context, circadian_phase, history)
```

not:

```text
expected_personal_value = patient's global average
```

## 9.2 Candidate methods

| Method | Strength | Weakness | Real-time recommendation |
|---|---|---|---|
| time-of-day bins | simple and interpretable | discontinuities; sparse bins; ignores smooth rhythm | fallback only |
| single-component cosinor | interpretable MESOR/amplitude/phase | may underfit nonsinusoidal rhythms | good baseline |
| multi-harmonic regression | interpretable, cheap, streaming-friendly | requires guard against overfitting | **recommended** |
| cyclic splines | flexible | more parameters, harder online update | useful offline/advanced |
| Gaussian process | flexible + uncertainty | computationally heavier, more difficult online | research/advanced |
| neural temporal model | expressive | data hungry, opaque | not justified initially |

Morelli et al. showed that a computationally cheap multi-component circadian model improved RHR prediction while retaining interpretable MESOR/amplitude/acrophase parameters [5]. Mixed-effects cosinor methods can estimate group and subject circadian effects longitudinally and are useful for cohort/model-development analyses [6].

## 9.3 Recommended circadian model

Start with:

```text
C_f(t) = a1 sin(2πt/24h) + b1 cos(2πt/24h)
```

Add a 12-hour harmonic only if it lowers held-out error sufficiently:

```text
+ a2 sin(4πt/24h) + b2 cos(4πt/24h)
```

Do not automatically add many harmonics.

Refit/update the circadian parameters from **stable, valid** history only. A detected acute or persistent event should not immediately distort the circadian curve.

## 9.4 What to retain longitudinally

For each feature where a circadian model is supported:

```yaml
circadian_model:
  mesor:
  amplitude_24h:
  acrophase_24h:
  amplitude_12h: optional
  acrophase_12h: optional
  fit_error:
  valid_days:
  last_fit:
```

Changes in MESOR, amplitude, or acrophase can themselves become longitudinal features, but should not be interpreted as AUD-specific without downstream evidence.

---

# 10. Sleep/wake-specific personal models

Sleep is not merely another time-of-day bin. It changes autonomic regulation, HR/HRV, respiration, peripheral temperature, and EDA. Speed et al. documented systematic daytime versus nighttime resting HR differences [10]; Sano et al. showed sleep-stage-linked EDA patterns [14,15]; Sarabia et al. showed wrist temperature coupling with bedtime and awakening [13].

## 10.1 Recommended separation

Maintain at minimum:

```text
sleep model
awake-rest model
```

for:

- resting HR;
- HRV;
- EDA;
- wrist/peripheral temperature;
- respiration where available.

## 10.2 Sleep transitions

Do not update either stable baseline during uncertain transition windows around sleep onset/waking unless the context layer marks the state confidently. Transition periods can be retained as a separate context if enough data later accumulate.

## 10.3 Sleep-stage modeling

If the system has independently validated sleep-stage estimates, stage-conditioned HRV/EDA/respiration may be valuable. If sleep staging is not reliable, do **not** condition on noisy pseudo-stages. Use whole-sleep or stable nocturnal windows instead.

---

# 11. Personal deviation score

The goal is not merely “above threshold?” but:

> How surprising is this observation under the patient's expected distribution for this context and time?

## 11.1 Primary score

For robustly modeled features:

```text
z_robust = (x - μ̂_context,circadian) / max(1.4826 × MAD_residual, scale_floor)
```

## 11.2 Empirical percentile

Also compute:

```text
personal_percentile = empirical_CDF(residual)
```

This is preferable when the residual distribution is asymmetric or heavy-tailed.

## 11.3 Prediction interval status

```text
inside_personal_90_interval
inside_personal_95_interval
outside_personal_95_interval
```

Do not equate “outside 95% personal range” with clinical pathology. It is a statistical unusualness statement only.

## 11.4 Relative percentage change

Use percentage change only when the denominator is meaningful and non-negligible. It can be useful for HRV trends but can be unstable for low/near-zero features such as some EDA measures.

## 11.5 Mahalanobis/multivariate distance

For a normalized vector `z_t` of sufficiently observed, nonredundant personal residuals:

```text
D_M(t) = sqrt(z_tᵀ Σ_shrink^{-1} z_t)
```

Use shrinkage covariance because patient-specific sample sizes may be modest and modalities are correlated. If stable-data volume is insufficient, use a reliability-weighted diagonal model rather than estimating an unstable covariance matrix.

---

# 12. Acute versus sustained versus baseline shift versus trajectory

The model must maintain separate detectors at different timescales.

| Phenomenon | Typical operational timescale | Primary mechanism | Stable-baseline update? | Output |
|---|---|---|---|---|
| acute deviation | seconds–minutes | robust residual + persistence | **no** | `ACUTE_DEVIATION` |
| sustained state | minutes–hours | run-length / rolling evidence / CUSUM | **no** | `SUSTAINED_DEVIATION` |
| daily shift | hours–day | robust daily summary vs expected | usually no until stable | `DAILY_DEVIATION` |
| candidate baseline shift | days | sequential change-point evidence | **freeze old baseline** | `CHANGE_POINT_CANDIDATE` |
| confirmed new regime | multiple stable days / phase-dependent | stable post-change evidence + context/treatment metadata | create new regime; retain old | `BASELINE_SHIFT_CONFIRMED` |
| recovery trajectory | weeks–months | state-space/robust trend | separate from normality baseline | `IMPROVING/STABLE/DETERIORATING` |

These are engineering timescale categories, not AUD diagnostic windows. Exact persistence requirements should be calibrated with the target deployment dataset.

---

# 13. Acute-change detection

## 13.1 Window-level processing

For every canonical feature window:

1. require upstream trustworthiness;
2. select the most specific mature personal/context baseline;
3. compute circadian-corrected expected value;
4. compute robust z and percentile;
5. update an acute persistence state machine;
6. emit state + uncertainty;
7. **do not** update the stable baseline from extreme/active-event windows.

## 13.2 Persistence state machine

Example structure:

```text
NORMAL
  ↓ repeated personally unusual windows
WATCH
  ↓ sufficient persistence / multimodal corroboration
ACUTE_DEVIATION
  ↓ prolonged evidence
SUSTAINED_DEVIATION
  ↓ return to expected range
RECOVERING
  ↓ stable
NORMAL
```

The number of windows must be feature- and cadence-specific. Do not hard-code one threshold across HR, HRV, EDA, temperature, and respiration.

---

# 14. Persistent-change and change-point detection

Sequential methods are appropriate because the question is whether repeated residuals indicate a state shift rather than isolated noise. CUSUM has precedent in real-time wearable anomaly systems [4], and change-point analysis has been used directly on longitudinal physiological data [21].

## 14.1 CUSUM on personal residuals

For standardized residual `z_t`:

```text
S⁺_t = max(0, S⁺_{t-1} + z_t - k)
S⁻_t = min(0, S⁻_{t-1} + z_t + k)
```

Trigger a candidate persistent change when:

```text
S⁺_t > h  or  |S⁻_t| > h
```

### Critical rule

`k` and `h` should be calibrated using stable personal/history data or development cohorts to achieve an acceptable false-alert rate. Do **not** import an infection-specific or ICU-specific threshold into AUD monitoring.

## 14.2 Alternative methods

| Method | Use | Recommendation |
|---|---|---|
| EWMA control chart | smooth persistent shifts | strong simple alternative |
| Page-Hinkley | mean-shift detection | useful when implementation simplicity matters |
| Bayesian online change-point detection | posterior over run length | strong research/advanced option, especially if uncertainty modeling is mature |
| generalized likelihood ratio | model-based change detection | useful with known pre/post distributions |
| SPRT | sequential hypothesis testing | useful when explicit hypotheses exist |

The initial implementation should use **CUSUM or EWMA + explicit state machine** because they are interpretable, streaming-friendly, and easy to validate.

---

# 15. Adaptive baseline update and contamination prevention

This is the most important operational safeguard.

A baseline that indiscriminately follows the patient will eventually learn persistent pathology, withdrawal, intoxication, sensor changes, or other abnormal states as “normal.” The hybrid-baseline literature explicitly identifies this danger for adaptive individual baselines [7].

## 15.1 Two-reference rule

Maintain both:

```text
B_recent   # tracks current operating level quickly
B_stable   # represents stable expected physiology and moves slowly
```

`B_recent` may adapt while an event unfolds. `B_stable` must remain frozen.

## 15.2 Baseline update gate

Define:

```text
G_t = 1 only if all stability conditions pass
```

Recommended gate conditions:

```text
signal_quality = TRUSTED
confound_state = RESOLVED/LOW
context_confidence >= required
acute_event = FALSE
sustained_deviation = FALSE
change_point_candidate = FALSE
device_distribution_shift = FALSE
known_transition_state = FALSE
missingness <= tolerated
baseline_model_not_stale = TRUE
```

Then:

```text
if G_t == 1:
    update stable statistics with capped influence
else:
    prediction only; do not update stable baseline
```

## 15.3 Robust online update

For features requiring a smooth online estimate, use bounded-influence EW updates or robust quantile estimators rather than unrestricted EWMA. For example:

```text
clipped_residual = clip(x - μ, -c*s, +c*s)
μ_new = μ_old + α * clipped_residual
```

where `α` is small for the stable baseline and larger for the recent reference.

`α` should be selected by desired adaptation timescale and validation, not invented as a universal physiological constant.

## 15.4 When a persistent shift occurs

Do **not** “unfreeze and catch up” immediately. Instead:

```text
old stable regime
        ↓
change-point candidate
        ↓
post-change observation buffer
        ↓
stability/context review
        ↓
new candidate regime
        ↓
confirmed new stable regime
```

Retain the old regime for historical comparisons.

---

# 16. Baseline confidence and model uncertainty

A single arbitrary confidence score can conceal a critical weakness. The model should retain **component confidence**.

## 16.1 Recommended confidence components

```yaml
confidence:
  data_sufficiency: 0..1
  recency: 0..1
  context_coverage: 0..1
  circadian_coverage: 0..1
  stability: 0..1
  upstream_signal_quality: 0..1
  regime_certainty: 0..1
  model_fit: 0..1
  overall: LOW | MEDIUM | HIGH
```

## 16.2 What drives each component

### Data sufficiency

- number of valid windows;
- number of independent valid days/nights;
- effective sample size after autocorrelation/context filtering.

### Recency

Decay when the patient has not contributed valid data for prolonged periods.

### Context coverage

A patient may have 30 days of data but only two nights of sleep data. The sleep baseline confidence should remain low.

### Stability

High dispersion, repeated change points, or nonstationarity lowers confidence.

### Regime certainty

A baseline collected during a known treatment transition should not be labeled a mature stable regime.

## 16.3 Overall confidence

For safety-critical interpretation, prefer a conservative combination such as:

```text
overall = min(critical_components)
```

or a calibrated probabilistic model. Do not simply average a nearly-zero context-coverage score with several high scores and call the result “moderate.”

---

# 17. Multiple baseline horizons and data recency

The system should maintain multiple horizons because recent physiology and historical normality answer different questions.

## 17.1 Recommended horizons

```text
immediate state: minutes–hours
recent daily reference: ~1 day
recent multi-day reference: ~7 days
mature rolling historical reference: ~28 days when available
regime history: full stable-regime duration
recovery history: weeks–months
```

These are operational defaults, not universal biological constants.

Evidence anchors:

- seven nights were sufficient for >80% of users to obtain an overnight RHR baseline within ±2 bpm of a three-month baseline in Alavi et al. [4];
- individualized multi-week histories have been used in smartwatch anomaly detection [3,4];
- HRV monitoring literature supports repeated standardized measurements across the week rather than a single sporadic reading [9].

## 17.2 Comparison vector

For every feature, expose:

```yaml
comparisons:
  vs_expected_now:
  vs_24h_reference:
  vs_7d_reference:
  vs_28d_reference:
  vs_stable_regime:
  vs_best_recovery_reference: optional
```

This is more informative than allowing the 7-day reference to overwrite the 28-day or regime-level baseline.

## 17.3 Recency weighting

Within a stable regime, older observations may receive lower weight for *current expectation* but should remain available for *trajectory analysis*. This is why the system should store summary history instead of destructively applying one exponential decay to everything.

---

# 18. Trend estimation and recovery trajectories

## 18.1 Online versus retrospective objectives

### Online

Need:

- simple streaming update;
- interpretable slope/level;
- uncertainty;
- robustness to missing days.

### Retrospective research

Can use:

- mixed-effects models;
- splines;
- Gaussian processes;
- Bayesian longitudinal models;
- richer covariate adjustment.

Do not burden the real-time system with the full retrospective research stack.

## 18.2 Recommended online trend model

First aggregate valid windows into robust daily/nightly summaries. Then apply a local linear state-space model:

```text
level_d = level_{d-1} + slope_{d-1} + w_level
slope_d = slope_{d-1} + w_slope
observed_daily_summary_d = level_d + v_d
```

This gives a continuously updated level, slope, and uncertainty while handling irregular observations better than a raw moving average.

A robust regression over the most recent stable daily summaries is a simpler fallback.

## 18.3 Trend classification

Do not label `IMPROVING` merely because the point estimate slope has the desired sign. Require:

1. sufficient valid days;
2. slope magnitude larger than expected day-to-day noise;
3. confidence/credible interval sufficiently away from zero or repeated directional consistency;
4. no unresolved regime change/device shift;
5. feature-specific interpretation of direction.

Example:

```yaml
trajectory:
  rmssd:
    horizon: 28d
    slope_log_units_per_day: 0.012
    slope_ci: [0.004, 0.020]
    direction: IMPROVING
    confidence: HIGH
```

The personalized layer should describe physiological direction. Mapping increased RMSSD to AUD recovery evidence belongs downstream and must use AUD-specific evidence [25–30].

---

# 19. Baseline reset versus adaptation versus new regime

## 19.1 Continue existing baseline

Use when:

- residuals remain within expected personal variability;
- no persistent detector signal;
- context composition is stable;
- device/sensor configuration unchanged.

## 19.2 Adapt existing baseline slowly

Use when:

- small gradual change is consistent with stable physiology;
- no clinical/treatment transition is active;
- no persistent abnormality flag;
- model residuals show mild drift rather than abrupt change.

## 19.3 Create a new candidate regime

Use when one or more occur:

- persistent change-point evidence;
- known clinically meaningful phase transition (e.g., entry into supervised treatment, completion of withdrawal-management period, sustained recovery phase) supplied as metadata;
- repeated multi-day shift that cannot be explained by context/circadian changes;
- device configuration change after recalibration (but mark it as a **measurement regime** rather than physiology unless proven otherwise).

## 19.4 Promote candidate regime

Promotion requires:

- adequate stable post-change data;
- acceptable signal/context coverage;
- absence of active acute/persistent anomaly;
- no evidence that the shift is only a transient event;
- if available, treatment/context metadata consistent with a genuine new physiological phase.

The previous baseline remains stored.

---

# 20. Regime-based modeling in AUD monitoring

It is useful to support multiple labeled regimes, but the personalized layer should not infer AUD stages on its own.

Possible **externally supplied** regime labels include:

```text
PRE_TREATMENT
SUPERVISED_WITHDRAWAL
STABILIZATION
EARLY_RECOVERY
STABLE_RECOVERY
RETURN_TO_DRINKING_CONFIRMED
UNKNOWN_TRANSITION
```

These labels should come from verified treatment/context information or downstream clinical interpretation. If such labels are absent, use neutral physiological regime IDs:

```text
REGIME_001
REGIME_002
CANDIDATE_003
```

This prevents the personalization engine from turning statistical change-point detection into an unsupported diagnosis.


# 21. Patient-specific multimodal modeling

The multimodal layer should consume **patient-normalized features**, not only raw feature values.

Example:

```text
HR:       +3.2 personal robust-SD
lnRMSSD:  -2.7 personal robust-SD
EDA:      +2.3 personal robust-SD
temperature residual: +0.3 personal robust-SD
motion/confound: low
```

is more comparable across modalities than raw bpm, milliseconds, microsiemens, and degrees.

## 21.1 Do not double count correlated measurements

Examples of likely dependence:

- ECG HR and PPG pulse rate;
- ECG HRV and PPG PRV;
- multiple HRV indices derived from the same RR series;
- HR and HR-over-activity residuals;
- EDA tonic and closely related summary statistics.

The fusion layer should group correlated features or use covariance-aware modeling.

## 21.2 Recommended multivariate model

When adequate stable personal data exist:

1. create a vector of robust personal residuals;
2. remove/reduce redundant variables;
3. estimate a **shrinkage covariance matrix** from stable-history residuals;
4. compute a joint Mahalanobis-type distance;
5. retain per-feature contributions for interpretability.

When data are sparse:

```text
joint_score = sqrt( Σ w_f × z_f² )
```

where `w_f` reflects reliability and availability and is capped to prevent one modality dominating.

Do not train a patient-specific autoencoder during cold start. One-class SVMs, isolation forests, and autoencoders may be evaluated offline after enough stable data are available, but they should not replace transparent robust residuals without clear validation [24].

---

# 22. Patient-specific modality responsiveness

People differ not only in baseline levels but also in which autonomic channels respond strongly. Cross-system stress research identifies distinct response profiles, and EDA literature documents stable responder/lability differences [16,23].

The system may therefore learn, **after repeated comparable events**, the following per modality:

```yaml
response_profile:
  amplitude_distribution:
  onset_latency_distribution:
  peak_latency_distribution:
  recovery_time_distribution:
  event_to_event_reliability:
  missingness_rate:
  upstream_quality_rate:
```

## 22.1 Safe fusion-weight adaptation

Personalized fusion weight should depend on:

```text
measurement reliability
× response reproducibility
× context validity
× current signal confidence
```

not simply response magnitude.

A patient who exhibits huge but inconsistent EDA responses should not automatically receive a high EDA weight.

## 22.2 Weight safeguards

- start from modality priors learned on development data;
- require repeated matched events before personal adaptation;
- cap minimum and maximum weights;
- shrink personal weights toward population priors when data are sparse;
- lower weight when missingness or quality deterioration increases;
- do not update weights during suspected pathological/transition episodes unless the training objective explicitly requires it.

---

# 23. Missing data, irregular sampling, and model staleness

Real-world longitudinal physiology is incomplete. Missing data should increase uncertainty, not be invisibly filled.

## 23.1 Missing minutes/hours

For short gaps:

- keep the latent state prediction;
- increase uncertainty;
- do not generate anomaly evidence from imputed points;
- do not update the stable baseline from imputed points.

## 23.2 Missing nights/days

Mark the corresponding daily summary as missing. Do not substitute a population value.

Alavi et al.'s NightSignal implementation used a very limited imputation policy for isolated missing nights in its specific overnight-RHR system [4]. That does not justify broad interpolation across arbitrary physiological streams.

## 23.3 Long gaps

After a configurable feature-specific stale interval:

```text
model_status = STALE
baseline_confidence.recency ↓
```

On return:

1. compare the new distribution with the previous stable regime;
2. temporarily rely more heavily on the population prior/reference;
3. reacquire sufficient personal data;
4. test for device/configuration distribution shift;
5. do not instantly resume full-confidence personal thresholds.

## 23.4 Unequal observation density

Daily summaries should prevent a day with thousands of windows from automatically outweighing a day with fewer valid windows in long-term trend estimation. Aggregate within day/context first, then model across days.

---

# 24. Device changes and measurement distribution shifts

A new electrode placement, firmware version, wearable, sampling configuration, or sensor may change a feature distribution without a physiological change.

Recommended handling:

```text
DEVICE_CHANGE
    ↓
freeze cross-device stable baseline update
    ↓
overlap/calibration period when possible
    ↓
compare old vs new device under matched context
    ↓
measurement-regime adjustment OR new measurement regime
```

Never interpret an unexplained device discontinuity as patient deterioration or recovery.

For ECG-derived HRV and PPG-derived PRV, this is especially important because PRV is not universally interchangeable with ECG HRV, particularly under activity/stress [18,19].

---

# 25. Personalized thresholds

The preferred system should avoid a single universal formula such as `baseline ± 2 SD` for every feature.

## 25.1 Threshold families

### Robust standardized threshold

Useful when the residual distribution is approximately symmetric after transformation:

```text
|z_robust| > calibrated_limit
```

### Empirical percentile

Useful for skewed/non-Gaussian data:

```text
x < personal_p05  or  x > personal_p95
```

### Model prediction interval

Useful when circadian/context regression supplies an expected value and uncertainty:

```text
x outside 95% personal prediction interval
```

### Relative change

Useful for interpretable trajectory statements:

```text
RMSSD = 28% below patient's stable sleep baseline
```

### Absolute/reference threshold

Maintained independently for population/clinical safety.

## 25.2 Threshold calibration principle

A statistical personal threshold should be calibrated to:

- false-alert rate during known stable periods;
- persistence rules;
- feature sampling cadence;
- downstream cost of false positives/negatives;
- availability of corroborating modalities.

van Rossum et al. show that personal/situational adaptive thresholds can improve alarm behavior in continuous vital-sign monitoring, but their ward thresholds are not transferable wholesale to AUD [20].

---

# 26. Feature-specific implementation algorithms

The algorithms below assume the feature already passed signal-quality and confound handling.

## 26.1 Resting heart rate (ECG or trusted PPG HR)

**Personalization class:** context-conditioned personal baseline required.

### Model

```text
expected_RHR = stable_regime_level
             + sleep/awake-rest offset
             + circadian harmonic
             + slow trend
```

### Statistics

- raw bpm;
- median/MAD of residuals;
- empirical percentiles;
- robust z;
- daily/nightly median or another validated robust summary.

### Algorithm

1. accept only validated low-activity/rest/sleep windows;
2. select sleep versus awake-rest model;
3. subtract personalized circadian expectation;
4. compute bpm delta, robust z, percentile;
5. update acute persistence state;
6. feed residual into sequential sustained detector;
7. aggregate stable windows into daily/nightly summary;
8. update stable baseline only when update gate passes;
9. update 7-day/28-day/regime trajectory;
10. retain population/reference HR status separately.

### Evidence

Large longitudinal wearable cohorts support strong individual RHR structure and circadian/time-of-day effects [1,2,10]. Seven valid nights can support a practical overnight RHR reference in one real-time wearable implementation [4].

## 26.2 ECG HRV — RMSSD

**Personalization class:** personal baseline preferred; context conditioning required.

### Model

Prefer `ln(RMSSD)` under standardized rest/sleep windows.

### Context controls

- sleep versus awake-rest;
- posture;
- respiration when available;
- window duration/method;
- ectopic/editing protocol;
- medication/treatment metadata downstream.

### Algorithm

1. require validated ECG interval series;
2. calculate RMSSD according to fixed protocol;
3. log-transform if diagnostics support it;
4. select matched personal context;
5. calculate robust residual and percentile;
6. use repeated windows/daily summary rather than one isolated value;
7. detect persistent suppression/elevation via sequential model;
8. update long-term baseline only during stable periods;
9. track daily/weekly slope separately from current abnormality.

HRV has very large inter-individual variability and methodological sensitivity [8]. AUD literature supports group-level and longitudinal changes but does **not** justify a universal patient-specific RMSSD cutoff for withdrawal/craving/relapse [25–30,33,34,38].

## 26.3 ECG HRV — SDNN and other indices

Use the same baseline architecture but keep feature semantics separate. Do not combine dozens of highly correlated HRV indices into fusion simply because software can compute them. Select a small pre-specified set with interpretable autonomic relevance and adequate repeatability.

Frequency-domain HRV requires especially careful respiratory/context control. The personalized layer should retain the value and personal residual; downstream interpretation should not label LF/HF or any single spectral index as a simplistic “sympathovagal balance” truth.

## 26.4 PPG pulse rate

Use the same HR expectation model when signal quality is strong. If ECG HR is simultaneously available, treat PPG HR as redundant/confirmatory rather than a second independent autonomic vote.

## 26.5 PPG-derived pulse rate variability (PRV)

**Personalization class:** conditional.

1. permit PRV personalization in validated rest/sleep conditions;
2. track device/context-specific PRV baseline separately from ECG HRV;
3. never silently merge PRV history into ECG HRV history;
4. suppress or downweight PRV under activity/stress when agreement is known to degrade [18];
5. if both ECG HRV and PRV are present, prefer ECG HRV for primary HRV interpretation.

## 26.6 PPG pulse amplitude / peripheral perfusion features

**Personalization class:** context-conditioned/trend-first.

Absolute amplitude is strongly dependent on sensor contact/site, vascular tone, temperature, and device gain. Use:

- normalized within-session values;
- patient/context residuals;
- robust trends;
- change-point detection after device/contact stability is established.

Do not create population clinical thresholds unless the specific derived metric is validated for that purpose.

## 26.7 EDA tonic level

**Personalization class:** strong personal/context baseline.

### Model

```text
EDA_tonic_expected = personal_context_level
                   + time/sleep component
                   + slow trend
```

### Statistics

- median/MAD;
- empirical quantiles;
- optional log transform;
- separate sleep and awake contexts.

### Algorithm

1. consume tonic EDA from decomposition layer;
2. select context/sleep model;
3. compute personal residual and percentile;
4. avoid pooling sleep NREM-related EDA dynamics with awake EDA [14,15];
5. use acute/sustained detector;
6. learn stable person-specific lability only after sufficient history;
7. freeze baseline during high-autonomic episodes/change-point candidates.

EDA responder/lability differences are sufficiently stable to justify personalization, while sleep EDA patterns demonstrate the need for context conditioning [14–17].

## 26.8 EDA phasic responsiveness

**Personalization class:** event-conditioned personal response.

Store per comparable event/context:

```text
SCR count/frequency
amplitude distribution
latency
recovery time
non-responder rate
```

Do not compare a clinical-task SCR distribution directly with spontaneous nighttime SCR storms.

## 26.9 Wrist/peripheral skin temperature

**Personalization class:** circadian/context-conditioned trend.

### Model

Use a personal harmonic/cosinor residual:

```text
temp_residual = observed_wrist_temp - expected_personal_wrist_temp(time, sleep/wake, context)
```

The wrist temperature rhythm is strongly related to sleep/wake timing and has a different phase from oral/core temperature [13]. Therefore:

- do not interpret wrist temperature as core temperature;
- do not use one absolute “fever” threshold;
- emphasize deviations from personal circadian expectation and persistent shifts;
- retain environmental/context information from upstream layers.

## 26.10 Respiratory rate

### Nocturnal RR

**Personalization class:** high-value personal baseline.

1. use validated sleep/nocturnal windows;
2. aggregate robust nightly RR;
3. maintain personal median/quantiles and day-to-day variance;
4. compute nightly deviation and sequential trend;
5. detect persistent shift across nights;
6. retain population/reference information separately.

Wearable RR validation and high-risk longitudinal monitoring both support the value of personal nocturnal references [11,12].

### Awake RR

Require context conditioning because voluntary control, speech, emotion, posture, and activity affect the measurement. Personal awake-rest baselines are more interpretable than pooled all-day RR.

## 26.11 Respiratory variability

Treat as a trend/personal feature until its exact derived definition and validation are fixed. Do not invent a universal threshold for “abnormal respiratory variability.”

## 26.12 Sleep/autonomic summaries

Possible daily features:

- sleeping RHR;
- sleep HRV;
- nocturnal RR;
- sleep EDA summary;
- wrist-temperature phase/amplitude;
- sleep duration/fragmentation supplied by a validated sleep layer.

Normalize each against personal sleep history and model multi-day trajectories. AUD studies show that sleep/autonomic recovery can evolve over months, so these are better treated as longitudinal state features than isolated nightly alarms [28,32].

---

# 27. AUD-specific longitudinal evidence

This section intentionally separates **AUD evidence** from the general personalization methods above.

## 27.1 What is well enough supported to use

### Cardiac autonomic dysfunction and recovery

A review of 33 alcohol/HRV studies concluded that acute alcohol generally reduces resting HRV; people with AUD often have lower resting HRV than controls; and some evidence indicates recovery with sustained abstinence, while reactive-HRV findings are more heterogeneous [25].

Weise et al. repeatedly studied 11 chronic alcohol-dependent participants from early abstinence through 5, 12, and 24 weeks; the group showed significant HRV improvement with prolonged abstinence of at least approximately six months, while some participants showed persistent vagal dysfunction [26]. This directly supports **patient-specific longitudinal trajectories**, because not every patient follows the same recovery path.

Yokoyama et al. followed 51 alcohol-dependent patients at admission, day 7, and day 30. Heart-rate variation improved by day 30 overall, but a subgroup retained persistent autonomic dysfunction [27]. Again, the important lesson is heterogeneity and repeated within-person measurement, not a universal one-month normalization rule.

de Zambotti et al. studied 15 alcohol-dependent participants and 13 controls during sleep within roughly one month after detoxification, around two months, and around four months. The AUD group initially had higher HR and respiratory rate and lower vagal HRV; HR and high-frequency vagal activity showed evidence of partial recovery across four months [28]. This strongly supports a **sleep-specific longitudinal cardiac-autonomic model**.

Eddie et al. reported that greater time since last drink was associated with higher HRV in treatment-engaged people in the first year of recovery [29]. This supports recovery-related autonomic normalization but is not sufficient by itself to define a within-person real-time recovery slope threshold.

A separate prospective study by Eddie et al. collected four days of ambulatory ECG in 42 people in early AUD recovery and related HRV/HR to alcohol use over a subsequent 90-day follow-up. Higher parasympathetic HRV and lower HR were associated with more abstinent days [30]. This is useful prognostic association evidence, but it does **not** establish that an acute personal HRV drop means relapse is imminent.

A 2025 repeated-measure study of 100 alcohol-dependent patients reported HRV changes after four weeks of detoxification/treatment versus baseline, further supporting that autonomic indices can move over clinically relevant weeks [37]. Treatment, abstinence, and other concurrent factors were not cleanly separable enough to make it a universal trajectory template.

## 27.2 Withdrawal physiology

Acute withdrawal can change multiple autonomic signals, but studies also warn against simplistic one-feature interpretation.

Bär et al. found strongly reduced baroreflex sensitivity during acute withdrawal and milder abnormalities in abstained participants; some nonlinear HRV/BRS measures correlated with withdrawal severity [33]. In another study of mild versus moderate/severe withdrawal, conventional HRV and sympathetic skin response did not consistently reflect the presumed sympathetic activation even though HR was elevated [34]. This is a direct argument for **multimodal physiology + longitudinal comparison**, not a single “HRV = withdrawal severity” rule.

White et al. obtained 24-hour ambulatory tremor, sweating, skin temperature, and activity records in six men undergoing withdrawal and controls. Sweating and skin temperature had pronounced nighttime patterns, reinforcing the importance of time-of-day/context correction even during withdrawal [35]. The sample was very small, so it should not set thresholds.

Older electrodermal work also reports withdrawal-related psychophysiological changes, but sample sizes/methods are insufficient for modern wearable personalized cutoffs [36].

## 27.3 Sleep and recovery

Longitudinal sleep work shows that recovery after alcohol dependence can be slow. Drummond et al. followed abstinent primary alcohol-dependent patients across early abstinence and later months/years and reported persistent sleep abnormalities with gradual improvement [32]. Follow-up attrition was substantial, so this supports *long-horizon modeling* more than precise patient-level prediction.

The de Zambotti sleep HRV study is more directly relevant to the proposed sensors because it shows repeated autonomic sleep measurements during early abstinence and partial recovery over four months [28].

## 27.4 EDA and ambulatory multimodal feasibility

Alinia et al. used an Empatica E4 for up to two weeks in a small sample of people recovering from AUD and examined EDA/HRV in relation to ecological self-reports [31]. The study supports feasibility of longitudinal wrist-based autonomic monitoring, but the small sample and analytic limitations mean it should **not** be used to define AUD-specific personal thresholds.

## 27.5 Relapse prediction: evidence boundary

Do not implement:

```text
personal HRV ↓ → RELAPSE_IMMINENT
```

The prospective Eddie study supports an association between ambulatory autonomic function and later alcohol use at the group level [30]. But Witteman et al., in 80 alcohol-dependent patients, found that laboratory cue-induced physiological reactivity/craving did **not** predict relapse at 5 weeks or 3 months [38]. The HRV review also characterizes the literature as heterogeneous [25].

Therefore the strongest defensible personalized output is:

```text
AUTONOMIC_TRAJECTORY_WORSENING_RELATIVE_TO_PERSONAL_RECOVERY_BASELINE
```

not:

```text
RELAPSE_IMMINENT
```

---

# 28. AUD longitudinal evidence synthesis table

| AUD phase/state | Feature | Observed longitudinal/within-person pattern | Time scale | Evidence strength | Recommended use in this layer | References |
|---|---|---|---|---|---|---|
| early abstinence → prolonged abstinence | HRV/vagal function | group HRV increased with prolonged abstinence; some non-recovery | weeks–months | moderate, small early study | track personal HRV recovery trajectory; permit heterogeneous trajectories | [26] |
| admission → day 30 abstinence | R-R variation | improvement overall by day 30; persistent dysfunction subgroup | days–month | moderate | detect persistent patient-specific autonomic impairment rather than assume normalization | [27] |
| post-detox → ~4 months | sleep HR, HRV, RR | high HR/RR and low vagal activity initially; partial HR/vagal recovery | months | moderate | separate sleep baseline + multi-month trajectory | [28] |
| first year recovery | HRV | higher HRV associated with longer time since last drink | variable | moderate association | downstream recovery evidence; not threshold source | [29] |
| early recovery → 90 d follow-up | HR/HRV | higher parasympathetic HRV/lower HR associated with more abstinent days | days → 3 months | moderate prognostic association | trajectory/risk evidence downstream; no imminent-relapse label | [30] |
| detox/treatment | HRV | repeated HRV changes over 4 weeks | 4 weeks | moderate | supports treatment-phase-specific baselines | [37] |
| acute withdrawal | BRS/HRV | BRS markedly reduced; some nonlinear autonomic measures associated with severity | hours–days | moderate | emphasizes transition regime and multimodal monitoring | [33] |
| acute withdrawal | HR/HRV/skin response | HR elevated; conventional HRV/SSR not uniformly diagnostic | days | moderate | do not make one-feature withdrawal rule | [34] |
| acute withdrawal | sweating, skin temp, tremor, activity | elevations with marked 24-h structure | 24 h | low (n=6) | reinforces circadian/context correction; not thresholds | [35] |
| abstinent recovery | sleep | sleep disturbance may persist and improve slowly | months–years | low–moderate due attrition | long-horizon sleep trajectory, not acute alarm | [32] |
| recovery wearable feasibility | EDA/HRV | ambulatory autonomic data feasible and related to ecological states | up to ~14 d | low–moderate | feasibility/support; no clinical cutoffs | [31] |
| detox cue reactivity → relapse follow-up | HRV cue response | cue reactivity present but baseline cue reactivity did not predict relapse | 5 wk / 3 mo | moderate negative evidence | explicit limit on relapse claims | [38] |

### AUD-specific conclusion

The most evidence-supported personalized AUD-relevant physiological trajectory today is **cardiac autonomic recovery/dysregulation**, especially repeated HR/HRV measures under standardized rest or sleep. EDA, temperature, respiration, and sleep add important complementary information, but evidence for modality-specific personalized AUD thresholds is thinner. The architecture should therefore preserve these modalities and their trajectories while avoiding invented clinical semantics.

---

# 29. How to distinguish healthy normalization from sensor/model drift

A persistent shift can represent recovery, deterioration, medication/treatment effect, changed behavior, changed environment, or a measurement-system change.

Use a triangulation rule:

```text
physiological shift candidate
        │
        ├─ device/configuration changed? ─► measurement shift candidate
        ├─ upstream quality/contact changed? ─► measurement shift candidate
        ├─ context composition changed? ─► re-estimate context expectation
        ├─ multiple independent modalities shift coherently? ─► supports physiology
        ├─ known treatment/recovery phase changed? ─► supports new physiological regime
        └─ repeated stable post-change data? ─► candidate stable regime
```

Healthy recovery is a **downstream interpretation**. This layer should report the objective trajectory and contextual evidence.

---

# 30. Explicit failure modes and mitigations

| Failure mode | What goes wrong | Mitigation |
|---|---|---|
| abnormal state contaminates baseline | persistent abnormality becomes “normal” | dual recent/stable baselines; update gate; event exclusion; candidate regimes |
| adaptation too fast | true change disappears into baseline | cap stable update rate; freeze during anomalies; maintain old regime |
| adaptation too slow | recovered physiology still compared with obsolete state | change-point candidate + new-regime promotion after stable evidence |
| sparse history | unstable quantiles/z-scores | population prior; wide uncertainty; provisional state; minimum data sufficiency |
| circadian pattern ignored | normal night/day variation flagged | patient-specific harmonic/cosinor model [5,10,13] |
| sleep/awake pooled | multimodal distributions become artificially wide | separate sleep and awake-rest baselines |
| treatment improvement flagged as pathology | healthy shift appears “abnormal” | retain transition metadata; candidate regime; downstream interpretation |
| relapse/return-to-drinking state absorbed | deterioration ceases to look abnormal | freeze stable baseline during known/suspected transition; retain recovery reference |
| contexts mixed | baseline variance inflates, sensitivity falls | hierarchical context selection |
| device change | artificial change point | device-regime metadata; calibration/overlap; separate measurement regime |
| PRV treated as ECG HRV | biased HRV interpretation | separate histories; restrict PRV to validated contexts [18,19] |
| too many correlated features | fusion overstates evidence | feature grouping/shrinkage covariance |
| one high-amplitude modality dominates | personalized fusion unstable | reliability/reproducibility-based capped weights |
| missing data interpolated blindly | false smoothness or alarms | prediction with uncertainty; no baseline update from imputed values |
| long gap ignored | stale baseline treated as current | staleness flag; recency confidence decay; reacquisition |
| population and personal status collapsed | absolute danger or personal deviation hidden | two explicit status axes [7,20] |
| “outside personal 95%” treated as disease | statistical tail becomes diagnosis | downstream semantics boundary; persistence and corroboration |
| AUD stage inferred from physiology alone | circular/unvalidated clinical claim | neutral regime IDs unless stage supplied externally |
| early withdrawal baseline becomes “healthy normal” | future comparisons misleading | preserve initial, current stable, and best-recovery references separately |


# 31. Major-paper extraction matrix

The following table extracts the implementation-relevant information from the studies that most directly determine this design. “Sampling” is reported only where it was material and available from the paper/abstract; absence means the study should be consulted before reproducing its acquisition protocol.

| Ref | Population / N | Modality | Duration / repeated measures | Within- vs between-person | Baseline / personalization method | Circadian/context handling | Statistical/modeling method and validation | Main implementation finding | Major limitation |
|---|---|---|---|---|---|---|---|---|---|
| [1] Li et al. 2017 | up to 43 individuals; >250,000 daily wearable measurements | HR, skin temperature, SpO2, activity and others | 1–24 months across participants | longitudinal within-person + between-person | personalized activity-based normalization and personal distributions | personalized circadian patterns considered | longitudinal anomaly framework; illness/environment case analyses | normalize physiology relative to personal activity/context rather than population-only values | small heterogeneous cohort; not AUD |
| [2] Quer et al. 2020 | 92,457 adults wearing trackers ≥35 weeks | daily RHR + sleep/activity metadata | median hundreds of days; study interval up to ~2 y | **explicit inter- vs intra-person** | individual long-term RHR distribution | season/time and participant factors analyzed | retrospective longitudinal cohort | RHR has strong personal structure; personal deviation can be meaningful despite population-plausible value | consumer-device derived; not clinical event model |
| [3] Mishra et al. 2020 | ~5,300 cohort; 32 COVID-positive analyzed | smartwatch HR, steps, sleep | retrospective pre/during infection histories | patient-relative anomaly detection | individualized historical baseline; activity-aware HR signal | activity context incorporated | two-tier warning/anomaly system | demonstrates personal residual/anomaly approach at scale | disease-specific thresholds not transferable to AUD |
| [4] Alavi et al. 2022 | 3,318 prospective participants; 2,155 contributed wearable data; 84 infected | overnight RHR, activity | prospective streaming | within-person real-time | streaming personal overnight baseline; other personal algorithms incl. CUSUM | nighttime/rest selection | prospective real-time alerts; baseline convergence analysis | >80% obtained overnight RHR baseline within ±2 bpm of 3-mo baseline after 7 nights; demonstrates streaming baseline + sequential detection | infection-specific alert parameters |
| [5] Morelli et al. 2019 | wearable RHR datasets | RHR | longitudinal wearable data | patient-specific circadian model | individual circadian prediction | **explicit circadian model** | multi-component model; ~10% lower RMSE than single-component cosinor in reported comparison | use interpretable streaming harmonic/cosinor rather than time bins alone | RHR-focused; not AUD |
| [6] Hou et al. 2021 | methodological/multiple longitudinal circadian datasets | circadian outcomes | repeated longitudinal observations | population + subject effects | mixed-effects cosinor | explicit MESOR/amplitude/acrophase | mixed-effects regression package/method | useful offline for separating population, patient, covariate, and time effects | not a direct real-time wearable algorithm |
| [7] Just et al. 2026 | methodological longitudinal digital-health analysis | behavioral/longitudinal signals | longitudinal | individual adaptive + group/fixed | hybrid adaptive and absolute references | context/task dependence discussed | hybrid baseline framework | adaptive personal baseline alone can follow gradual deterioration; retain absolute/fixed bounds | behavioral focus; principle must be validated physiologically |
| [8] Nunan et al. 2010 | 44 studies; 21,438 healthy adults | short-term HRV | cross-study repeated protocols | predominantly between-person normative review | no patient adaptive baseline | methodology/context heterogeneity central | systematic quantitative review | huge inter-individual variation and protocol sensitivity justify within-person standardized HRV | not longitudinal patient monitoring |
| [9] Plews et al. 2014 | trained athletes (monitoring study) | HRV | repeated weekly measurements | within-person | weekly HRV summaries | standardized measurement context | agreement across compliance levels | repeated measurements across a week are preferable; ≥3 valid points/week can approximate weekly change in that setting | athlete evidence; not AUD/clinical baseline duration |
| [10] Speed et al. 2023 | 433 participants; 19k+ person-days/nights | PPG-derived RHR | up to ~3 months | within-person day/night + population | rest-window RHR | full 24-h analysis, day/night | large wearable observational analysis | resting HR differs systematically across 24 h; sleep/night and daytime rest should not be pooled blindly | device/algorithm specific |
| [11] Natarajan et al. 2021 | large wearable datasets + sleep-study validation | respiratory rate derived from HR/PPG | repeated nightly | population + longitudinal | personal longitudinal RR application | nocturnal/sleep restriction | validation MAE ~0.46 breaths/min, MAPE ~3% | nocturnal RR is measurable and longitudinally useful; retain personal baseline | derived RR method/device specific |
| [12] Harrington et al. 2022 | 2,000 sleep studies for reference; 34 high-risk patients, 3,403 patient-nights | nocturnal RR | long home monitoring | explicit inter- vs intra-patient | patient-specific NRR reference | nocturnal only | deviations vs baseline; hospitalization association | interpatient variance > intrapatient variance; strong rationale for personal nocturnal RR | preprint/clinical deterioration setting; not AUD |
| [13] Sarabia et al. 2008 | 99 university students | wrist temperature + diaries/oral temp | normal-living repeated recordings | within-person rhythm + group | rhythm characterization | explicit sleep/wake and circadian phase | circadian analysis | wrist temperature rises around bedtime and falls after awakening; model rhythm, not global mean | healthy young cohort; wrist != core temp |
| [14] Sano et al. 2014 | 80 nights across multiple healthy-adult protocols | wrist/palm EDA + PSG subset | multi-night | inter- and intra-night | sleep EDA distributions | **sleep stage** | automatic EDA peak/storm characterization | >80% EDA peaks occurred in NREM; sleep EDA needs separate context model | small participant numbers with uneven nights |
| [15] Sano & Picard 2011 | 7 PSG subjects; 24 multi-night; 3 long-term 30–60 nights | EDA, skin temperature, motion | up to 30–60 nights | explicit inter/intra individual | individual sleep EDA patterns | sleep context | descriptive taxonomy | supports personal sleep EDA phenotypes and long-horizon collection | exploratory/small cohorts |
| [16] Crider et al. 2004 | 345 adult twin pairs | electrodermal response lability | retest/cross-situational | individual differences | response-lability phenotype | multiple situations | retest/cross-situational analyses | EDA response lability has substantial stability; patient-specific response phenotype is plausible | lab psychophysiology, not wearable/AUD |
| [17] Schell et al. 2002 | 71 recent-onset schizophrenia patients + 36 controls | EDA variables | ~1 year | retest within-person + group | repeated personal response measures | task protocol | test-retest correlations | many EDA variables/respondership show year-scale stability | clinical population unrelated to AUD |
| [18] Schäfer & Vagedes 2013 | review of PRV vs HRV studies | PPG/other pulse intervals vs ECG RR | multiple studies | method-comparison | not a personal baseline | rest/activity/stress contrasted | comparative review | PRV may approximate HRV at rest but agreement worsens with activity/stress; keep PRV separate | heterogeneous studies |
| [19] Han et al. 2026 | 914 young adults; validation cohorts 536 + 652 | resting PPG PRV | four 15-min sessions over 2 days | test-retest | reliability by metric | resting-state | ICC; independent lifespan cohorts | many PRV metrics reliable at rest, but reliability is feature-specific | research resting sessions, not free-living AUD |
| [20] van Rossum et al. 2022 | 39 postsurgical ward patients; 3,999 h; 18 adverse events | continuous vital signs | hospitalization monitoring | personal/situational adaptation | adaptive alarm thresholds | situational factors | retrospective simulation | adaptive thresholds can improve sensitivity/alarm load; personalization can be operationally useful | small clinical cohort; no AUD transfer of thresholds |
| [21] Rosenfield et al. 2010 | ambulatory physiological panic-attack data | cardio-respiratory physiology | longitudinal before attacks | within-person | change-point analysis | ambulatory context | change-point methodology | explicit change-point detection can identify transition rather than fluctuation | panic context, not AUD |
| [23] Kupper et al. 2021 | 744 young adults | cardiovascular/autonomic/hemodynamic systems | rest + acute stress | between-person response profiles | response-profile characterization | controlled stress context | multivariate response profiles | people differ in which physiological systems respond; supports personalized modality responsiveness | lab acute stress |
| [25] Ralevski et al. 2019 | 33 alcohol/HRV articles | HRV | acute/chronic/abstinence literature | mixed | review of AUD HRV patterns | resting vs reactive HRV separated | narrative/systematic review synthesis | AUD HRV is heterogeneous; abstinence recovery exists, reactive HRV evidence less settled | heterogeneous protocols/studies |
| [26] Weise et al. 1986 | 11 chronic alcohol-dependent participants | HRV | 1–30 d after abstinence, then 5/12/24 wk; two very long follow-ups | **within-person** | early abstinence as reference | not a circadian study | repeated follow-up | HRV increased with prolonged abstinence in group; some did not recover | very small sample |
| [27] Yokoyama et al. 1991 | 51 alcohol-dependent patients | R-R interval variation | admission, day 7, day 30 | **within-person** | admission/early abstinence | not primary | repeated measures + subgrouping | overall improvement by day 30; persistent dysfunction subgroup | older measure/protocol |
| [28] de Zambotti et al. 2015 | 15 AUD + 13 controls | sleep HR, HRV, RR | within ~1 mo post-detox; ~2 mo; ~4 mo | repeated within-person + controls | post-detox repeated sleep physiology | stable NREM sleep | frequency-domain HRV, repeated assessment | partial recovery in HR and vagal HRV; sleep-specific trajectory justified | small cohort; 6 relapsed |
| [30] Eddie et al. 2023 | 42 early-recovery participants | ambulatory ECG HR/HRV | 4 d ECG + 90 d alcohol follow-up | baseline physiology predicting later outcome | ambulatory patient measures | in natura | regression against percent days abstinent | higher vagal HRV/lower HR associated with more abstinent days | not an imminent-relapse detector; limited duration |
| [31] Alinia et al. 2021 | 11 adults in AUD recovery | EDA + HRV wearable | up to 14 d | repeated ambulatory | feature associations | ecological momentary assessments | signal-quality + feature/self-report associations | demonstrates feasibility of continuous autonomic wearables in AUD recovery | very small sample; no personalized clinical threshold |
| [32] Drummond et al. 1998 | 29 early, 29 at 19 wk, 9 at 14 mo, 4 at 27 mo | polysomnographic sleep | ~16 d, 19 wk, 14 mo, 27 mo | longitudinal | abstinence-time trajectory | sleep architecture | repeated follow-up | sleep recovery can be slow and incomplete; motivates months-long trajectory model | severe attrition |
| [33] Bär et al. 2006 | 20 acute withdrawal + matched controls; 15 abstained | BRS, HRV, BPV | withdrawal peak, +2 h, +24 h; abstinent comparison | repeated acute + group | withdrawal phase | controlled clinical | linear/nonlinear autonomic analyses | acute autonomic dysregulation not reducible to one HRV feature | treatment context; small cohort |
| [34] Bär et al. 2006 | 20 mild + 20 moderate/severe male withdrawal | HRV + sympathetic skin response | climax, next day, before discharge | repeated | withdrawal trajectory | clinical | HRV/SSR + withdrawal scores | conventional HRV/SSR not consistently diagnostic despite high HR; warns against one-feature rules | male inpatient cohort |
| [35] White et al. 1994 | 6 men in withdrawal + age-matched controls | tremor, sweating, skin temp, activity | 24 h | repeated ambulatory | controls + within-day pattern | **24-h time structure** | ambulatory descriptive | withdrawal signals have strong time-of-day structure | extremely small sample |
| [38] Witteman et al. 2015 | 80 alcohol-dependent detoxification patients | HRV cue reactivity + craving | lab + 5 wk diary + 5 wk/3 mo relapse follow-up | cue response + prospective outcome | cue baseline | controlled cue exposure | prospective relapse analysis | physiological cue reactivity occurred but did **not** predict relapse | cue-specific, not continuous wearable monitoring |

---

# 32. Final synthesis table A — feature personalization

| Feature | Personalization needed? | Baseline type | Circadian model | Update horizon | Primary deviation metric | Key references |
|---|---|---|---|---|---|---|
| resting HR | **yes** | sleep + awake-rest stable regime, recent reference | 24-h harmonic; optional 12-h if validated | immediate, 7 d, ~28 d, regime | bpm delta + robust z + percentile | [1–5,10] |
| ECG RMSSD | **yes** | standardized sleep/rest; log-domain distribution | sleep/awake separation; time effect if enough data | repeated daily, 7 d, ~28 d, regime | log residual + robust z + percentile | [8,9,25–30] |
| ECG SDNN/selected HRV | yes | standardized protocol-specific | context/time dependent | daily/weekly/regime | robust z/percentile | [8,25–30] |
| PPG HR | yes | same physiological HR contexts; device-specific residual checks | same as HR | immediate/7 d/28 d | bpm residual | [2,10] |
| PPG PRV | conditional | **separate PRV baseline** in validated rest/sleep | context dependent | repeated rest/sleep | metric-specific robust residual | [18,19] |
| PPG pulse amplitude/perfusion | yes, trend-first | device/contact/context-specific | temperature/time as context | hours–weeks | relative/robust residual | methodology dependent |
| EDA tonic | **yes** | sleep + awake/context-specific | state/time model | minutes, daily, weeks | robust quantile/log residual | [14–17,24,31] |
| EDA phasic SCR | **yes, event-conditioned** | matched-event response distribution | secondary to event state | event history | count/amplitude percentile, latency | [14–17,23] |
| wrist/peripheral temp | **yes** | personal circadian/context curve | **required** | daily/weekly/regime | residual from circadian curve | [1,13,35] |
| nocturnal RR | **yes** | sleep/nightly baseline | nocturnal restriction initially | nightly, 7 d, 28 d | breaths/min delta + percentile | [11,12,28] |
| awake RR | yes if used | awake-rest/context-specific | optional after context | minutes/days | robust residual | [11,12] |
| respiratory variability | trend-first | definition-specific | sleep/context | days/weeks | distribution/rate-of-change | [11,12] |
| sleep/autonomic summary | **yes** | personal sleep regime | inherent sleep/circadian | daily → months | robust daily residual + trend slope | [14,15,28,32] |
| response latency/recovery | after repeated matched events | event-conditioned | context/event | repeated events | deviation from personal response distribution | [16,23] |

---

# 33. Final synthesis table B — timescales

| Physiological phenomenon | Operational timescale | Model | Stable baseline update allowed? | Output |
|---|---|---|---|---|
| measurement/window fluctuation | seconds–minutes | robust residual | only if stable and non-anomalous | `NORMAL_VARIATION` / tail score |
| acute deviation | seconds–minutes | residual + persistence state machine | **no** | `ACUTE_DEVIATION` |
| sustained autonomic state | minutes–hours | rolling evidence / CUSUM | **no** | `SUSTAINED_DEVIATION` |
| daily shift | hours–day | robust daily/nightly summary | candidate only if stable | `DAILY_DEVIATION` |
| recent reference movement | ~days/week | robust 7-day state | recent reference yes; stable baseline gated | `RECENT_LEVEL_SHIFT` |
| persistent baseline shift | multi-day | CUSUM/change point + post-change buffer | freeze old; create candidate regime | `CHANGE_POINT_CANDIDATE` |
| confirmed regime change | feature/phase dependent, generally multiple stable days | regime model | create new regime; keep old | `BASELINE_SHIFT_CONFIRMED` |
| recovery/deterioration trajectory | weeks–months | state-space/robust trend | trajectory updates; normality baseline separate | `IMPROVING/STABLE/DETERIORATING` |

---

# 34. Final synthesis table C — baseline architecture

| Baseline/reference | Horizon | Purpose | Update rule | Freeze conditions |
|---|---|---|---|---|
| population/reference | fixed/external | external context + safety | only when validated reference model version changes | never patient-adapted |
| session/local reference | minutes–hours | detect change within current session | fast robust estimator | poor quality/confound |
| recent adaptive reference | ~1–7 d | track current operating level | moderate gated robust update | poor quality; may continue cautiously during some transitions but never defines health |
| stable personal regime | ~28 d+ / regime | define expected personal normal | slow robust update from stable windows only | acute/sustained anomaly, change point, treatment transition, device shift, confound, poor quality |
| circadian/context model | rolling stable history | expected value “now” | daily/nightly refit on stable data | transition/anomaly/device shift |
| historical regime archive | indefinite summaries | compare current state with previous phases | append-only | not overwritten |
| best stable recovery reference | established stable recovery only | high-quality recovery anchor | update only after sustained verified stable recovery | active deterioration/transition; insufficient evidence |
| longitudinal trajectory | days–months | improvement/stability/deterioration | update from valid daily summaries | long missing gaps lower confidence rather than freeze history |

---

# 35. Real-time implementation architecture

## 35.1 Separation of real-time and offline components

### Real-time patient-state service

Responsibilities:

- ingest trusted feature windows;
- retrieve patient/context model;
- compute expected personal state;
- calculate residuals/percentiles;
- update acute/persistent state machines;
- apply baseline-update gate;
- emit personalized state;
- persist compact sufficient statistics.

### Offline/periodic longitudinal analytics

Responsibilities:

- recompute circadian models;
- estimate shrinkage covariance;
- re-evaluate regime boundaries;
- update long-horizon trend model;
- validate response phenotypes;
- generate patient-level daily/weekly summaries;
- monitor model calibration and false-alert rates.

## 35.2 Suggested update cadence

The upstream system may update every second, but the personal model should operate on physiologically meaningful feature windows.

**Engineering starting point (must be validated):**

- continuous feature residuals: every 30–60 s or at upstream feature cadence;
- acute state machine: every feature window;
- sustained CUSUM: every valid canonical window;
- sleep/night summary: after sleep episode/night;
- daily trajectory summary: once per day when enough valid coverage exists;
- circadian model: refit nightly/daily or after substantial stable data accumulation;
- stable baseline: update from accepted stable summaries, typically daily;
- covariance/personal fusion model: periodic (daily/weekly) after sufficient data;
- regime review: when change-point candidate exists and during periodic offline processing.

These cadences are implementation defaults, not paper-derived physiological constants.

## 35.3 State retained per patient/feature

```yaml
feature_model:
  model_version:
  measurement_regime_id:
  physiological_regime_id:
  population_reference_version:

  stable_baseline:
    median:
    mad:
    quantiles:
    n_valid_windows:
    n_valid_days:

  recent_reference:
    level:
    scale:
    horizon:

  circadian:
    coefficients:
    mesor:
    amplitude:
    acrophase:
    fit_error:

  context_models:
    sleep: {...}
    awake_rest: {...}
    upright_low_activity: {...}

  sequential_state:
    cusum_positive:
    cusum_negative:
    persistence_state:
    run_length:

  trajectory:
    level:
    slope:
    covariance:

  confidence:
    data_sufficiency:
    recency:
    context_coverage:
    stability:
    regime_certainty:

  last_valid_observation:
  last_stable_update:
```

## 35.4 Computational complexity

The recommended first version is cheap:

- robust residual: `O(1)` after maintained summaries;
- harmonic expectation: `O(k)` for small harmonic count;
- CUSUM/state machine: `O(1)`;
- EW/online robust update: `O(1)`;
- daily state-space update: `O(1)` for small state;
- shrinkage covariance: periodic `O(d^2 n + d^3)` for small feature dimension `d`, acceptable offline.

This is well suited to a real-time backend and does not require deep learning inference per sample.

---

# 36. Storage and data minimization

Months of raw sensor data are **not required solely for the personalized longitudinal layer** once upstream feature extraction has produced validated summaries.

Retain, at minimum:

- stable distribution sufficient statistics/quantile sketches;
- context-specific daily/nightly summaries;
- circadian coefficients and fit statistics;
- sequential detector state;
- trend state and uncertainty;
- regime transition history;
- device/measurement-regime history;
- representative event windows or feature traces when needed for audit/validation;
- model version and reason codes.

Raw physiological data may still need retention for signal-quality reprocessing, research validation, regulatory/audit requirements, or later feature extraction, but that is an upstream/system policy rather than a necessity of the personal baseline algorithm.

---

# 37. Output schema

A production response should expose current values, expectations, both abnormality dimensions, trajectory, and confidence.

```yaml
personal_state:
  patient_model_version: "1.0"
  timestamp: "..."

  physiological_regime:
    id: "REGIME_004"
    external_label: "STABLE_RECOVERY"   # only if supplied/verified externally
    certainty: 0.91

  baseline_confidence:
    overall: HIGH
    data_sufficiency: 0.94
    recency: 0.99
    context_coverage: 0.88
    circadian_coverage: 0.91
    stability: 0.86
    regime_certainty: 0.91

  features:
    resting_hr:
      current: 92.0
      unit: bpm
      context: AWAKE_REST
      expected_for_context_time: 67.1
      personal_interval_90: [61.8, 72.6]
      absolute_delta: 24.9
      robust_z: 3.15
      personal_percentile: 0.994
      population_status: REFERENCE_NOT_EXCEEDED
      personal_status: MARKEDLY_ELEVATED
      persistence:
        duration_minutes: 20
        state: SUSTAINED_DEVIATION
      baseline_source: STABLE_REGIME_CONTEXT_CIRCADIAN
      stable_update_allowed: false
      reason_codes:
        - ABOVE_PERSONAL_BASELINE
        - OUTSIDE_PERSONAL_RANGE
        - SUSTAINED_DEVIATION
        - BASELINE_UPDATE_FROZEN

    ln_rmssd:
      current_original_ms: 24.0
      expected_original_ms: 51.0
      robust_z: -2.8
      personal_percentile: 0.012
      population_status: NOT_INTERPRETED_HERE
      personal_status: SUPPRESSED
      stable_update_allowed: false
      reason_codes:
        - BELOW_PERSONAL_BASELINE
        - SUSTAINED_DEVIATION

  multimodal:
    model: SHRINKAGE_COVARIANCE
    joint_personal_distance: 3.7
    dominant_nonredundant_contributors:
      - resting_hr
      - ln_rmssd
      - eda_tonic
    correlated_features_deduplicated:
      - ppg_hr_redundant_with_ecg_hr
    state: MULTIMODAL_PERSONAL_DEVIATION

  trajectory:
    24h:
      state: WORSENING
      confidence: MEDIUM
    7d:
      state: STABLE
      confidence: HIGH
    28d:
      state: IMPROVING
      confidence: HIGH
    stable_regime:
      state: IMPROVING_SINCE_REGIME_START

  change_point:
    candidate: false
    posterior_or_score: null

  model_status:
    stale: false
    device_shift_suspected: false
    last_stable_update: "..."
```

The values above are illustrative schema examples, not clinical thresholds.

---

# 38. Reason codes

Recommended interpretable codes:

```text
ABOVE_PERSONAL_BASELINE
BELOW_PERSONAL_BASELINE
OUTSIDE_PERSONAL_90_PERCENT_RANGE
OUTSIDE_PERSONAL_95_PERCENT_RANGE
ACUTE_DEVIATION
SUSTAINED_DEVIATION
DAILY_DEVIATION
RECENT_LEVEL_SHIFT
CHANGE_POINT_CANDIDATE
BASELINE_SHIFT_CONFIRMED
CIRCADIAN_EXPECTED
CONTEXT_EXPECTED
RECOVERY_TREND_IMPROVING
RECOVERY_TREND_WORSENING
RECOVERY_TREND_STABLE
MULTIMODAL_CONCORDANCE
MULTIMODAL_DISCORDANCE
CORRELATED_FEATURES_DEDUPED
INSUFFICIENT_PERSONAL_HISTORY
BASELINE_PROVISIONAL
BASELINE_UNCERTAIN
PERSONAL_MODEL_STALE
BASELINE_UPDATE_FROZEN
DEVICE_DISTRIBUTION_SHIFT
MEASUREMENT_REGIME_CHANGED
CONTEXT_COVERAGE_INSUFFICIENT
POPULATION_REFERENCE_EXCEEDED
POPULATION_REFERENCE_NOT_EXCEEDED
```

Reason codes should explain *why* the personalized state exists without pretending to provide the AUD diagnosis.

---

# Recommended Implementation

This section specifies the **single consolidated personalized longitudinal physiological modeling system that should actually be built**.

## 1. Personal model hierarchy

Maintain the following simultaneously for each patient and feature:

```text
A. population/reference model + absolute safety constraints
B. stable personal physiological regime
C. context-conditioned personal distribution
D. personal circadian model
E. recent adaptive reference
F. sequential acute/persistent-deviation state
G. long-term trajectory model
H. historical regime archive / optional best stable recovery reference
```

Do not replace one with another.

### Data path

```text
trusted feature x_f(t)
+ quality/confound/context
        │
        ▼
select patient regime + context model
        │
        ▼
μ̂_f(t) = stable level + circadian + context + slow trend
        │
        ├────────── observed x_f(t)
        ▼
personal residual + robust z + percentile
        │
        ├─► acute persistence state
        ├─► CUSUM/persistent detector
        ├─► multimodal normalized residual vector
        └─► daily robust summary
                  │
                  ▼
          longitudinal level/slope
                  │
                  ▼
       candidate/confirmed regime change
```

In parallel:

```text
x_f(t) ─► population/reference/safety evaluation
```

Downstream receives both personal and population status.

---

## 2. Baseline initialization

### 2.1 On enrollment

Create a feature model with:

- population/reference prior where scientifically appropriate;
- `stable_baseline = UNINITIALIZED`;
- `recent_reference = SESSION_ONLY`;
- `baseline_confidence = VERY_LOW`;
- separate context buckets created lazily as valid data arrive.

### 2.2 First valid session

Calculate local robust summaries but label them `SESSION_REFERENCE`, not a patient baseline.

### 2.3 First several valid days

Build provisional sleep and awake-rest distributions. For RHR, begin overnight baseline convergence tracking. For HRV, require repeated standardized windows/days; do not equate one night's RMSSD with a stable personal HRV baseline.

### 2.4 Baseline maturity

Use feature-specific readiness rather than one elapsed-day threshold.

For overnight RHR, **seven valid nights** is an evidence-supported checkpoint because Alavi et al. observed >80% baseline convergence to within ±2 bpm of a three-month reference by that point [4]. It remains a checkpoint, not an automatic “mature” label if the patient's data are unstable or sparse.

For other features, maturity should depend on:

```text
valid independent days
+ context coverage
+ residual stability
+ recency
+ circadian coverage
```

A ~28-day reference should be maintained when enough continuous history exists because multi-week personalized histories are established in wearable anomaly detection [3,4], but no statement should claim that 28 days is biologically optimal for every modality.

---

## 3. Cold-start strategy

Use a **population-prior-to-personal transition**.

### Stage 0 — no personal history

```text
expected = population/reference/context model
personal status = NOT_ESTABLISHED
uncertainty = very high
```

### Stage 1 — sparse personal history

```text
expected = shrinkage blend(population, personal provisional)
personal status = PROVISIONAL
uncertainty = high
```

### Stage 2 — adequate context-specific history

```text
expected = predominantly personal context/circadian model
personal status = ACTIVE
uncertainty = estimated from residual distribution
```

### Stage 3 — mature stable regime

```text
population prior remains parallel
personal model drives personal-deviation interpretation
```

Do not hide cold-start uncertainty behind a numeric z-score computed from five observations.

---

## 4. Circadian modeling

Implement per feature/context:

```text
C(t) = a1 sin(2πt/24h) + b1 cos(2πt/24h)
```

Optionally add:

```text
+ a2 sin(4πt/24h) + b2 cos(4πt/24h)
```

only when cross-validation or rolling held-out prediction shows a meaningful improvement.

### Update rule

- fit on stable valid data only;
- refit daily/nightly or incrementally;
- retain fit residuals and uncertainty;
- freeze/discount data from acute or persistent anomalies;
- store MESOR/amplitude/acrophase for longitudinal analysis.

This keeps the computational and interpretability advantages demonstrated by wearable circadian RHR modeling [5].

---

## 5. Context-conditioned baselines

Start with exactly these contexts:

```text
SLEEP
AWAKE_REST
UPRIGHT_LOW_ACTIVITY
POST_ACTIVITY_RECOVERY
OTHER_VALID
```

Create a more specific context only after:

- the context classifier is reliable;
- sufficient patient-specific observations exist;
- separate modeling improves held-out prediction beyond a simpler parent context.

Fallback chain:

```text
specific context personal
→ parent context personal
→ personal time-of-day model
→ global personal
→ population prior
```

Increase uncertainty at each fallback.

---

## 6. Personal deviation score

For each trusted feature:

```text
expected = μ̂_f(t)
residual = x_f(t) - expected
scale = max(1.4826 × MAD(stable matched residuals), scale_floor_f)
z_robust = residual / scale
percentile = ECDF_stable_matched(residual)
```

Also retain:

```text
absolute_delta
relative_delta where meaningful
inside/outside personal prediction interval
```

### Transformation

- raw HR/RR: native units;
- RMSSD and other strongly right-skewed positive HRV indices: preferably log domain if validated;
- EDA amplitudes/levels: robust nonparametric or log domain when appropriate;
- wrist temperature: residual from circadian/context model.

Never rely on `z_robust` alone as a clinical statement.

---

## 7. Acute-change detection

Use a per-feature persistence state machine over personal residuals:

```text
NORMAL → WATCH → ACUTE_DEVIATION → RECOVERING → NORMAL
```

Transition based on:

- residual magnitude/percentile;
- number/duration of consecutive valid windows;
- current context confidence;
- upstream quality;
- optional multimodal corroboration.

Exact magnitude and persistence thresholds must be calibrated to stable-history false-alert behavior for the target dataset. Do not invent one universal threshold.

Stable-baseline updates are disabled while state is `WATCH`, `ACUTE_DEVIATION`, or `RECOVERING` until recovery stability criteria are met.

---

## 8. Persistent-change detection

Run two-sided CUSUM on standardized personal residuals:

```text
Splus  = max(0, Splus_prev  + z - k)
Sminus = min(0, Sminus_prev + z + k)
```

A threshold crossing creates:

```text
CHANGE_POINT_CANDIDATE
```

not a new baseline.

Tune `k` and decision threshold `h` from stable-development/patient data to a target false-alarm rate. Keep these as configurable model parameters with versioning.

Alternative EWMA/Page-Hinkley/BOCPD implementations can be benchmarked, but CUSUM is the recommended first implementation because it is transparent, streaming, and already demonstrated in wearable longitudinal anomaly systems [4].

---

## 9. Baseline-shift detection

After a persistent-change candidate:

1. freeze the previous stable baseline;
2. collect a post-change buffer of trustworthy, context-corrected data;
3. check whether the change survives circadian/context correction;
4. check for device/firmware/contact distribution shift;
5. incorporate known treatment/regime transition metadata;
6. verify the post-change distribution is itself stable;
7. create `CANDIDATE_REGIME_N`;
8. promote to `REGIME_N` only after sufficient stable evidence;
9. keep the old regime immutable for historical comparisons.

The personalized layer labels this a physiological regime change, not “recovery” or “relapse” unless that external label is supplied by the downstream clinical system.

---

## 10. Trend estimation

### Daily aggregation

For each feature/context, create one robust daily/nightly summary with coverage and uncertainty.

### Online trend model

Use a local linear state-space model:

```text
level_d = level_{d-1} + slope_{d-1} + w1
slope_d = slope_{d-1} + w2
y_d     = level_d + v_d
```

Output:

```text
current level
slope
slope uncertainty
24h / 7d / 28d directional summaries
```

A trend may be labeled `IMPROVING`/`DETERIORATING` only when direction has feature-specific meaning and the slope is sufficiently certain. Otherwise emit `INDETERMINATE` or `STABLE`.

For the personalized layer, a generic alternative is:

```text
INCREASING / STABLE / DECREASING
```

with the AUD-specific layer deciding whether that direction is clinically favorable.

---

## 11. Baseline adaptation

Maintain two rates:

```text
recent reference → moderate adaptation
stable baseline → very slow, gated adaptation
```

Use robust bounded-influence updates or quantile sketches.

### Stable update gate

Stable baseline can update only if:

```text
quality trusted
AND confound state acceptable
AND context confidence adequate
AND no acute state
AND no sustained state
AND no change-point candidate
AND no device shift
AND no known treatment/withdrawal transition
AND missingness acceptable
```

### Critical principle

A long-running deviation may influence the **recent reference** while the stable baseline stays frozen. This allows the system to say:

> current physiology has been elevated for three days relative to the patient's stable regime

instead of gradually redefining those three days as normal.

---

## 12. Baseline freeze rules

Freeze stable updates when any of the following is true:

```text
SIGNAL_UNTRUSTWORTHY
CONFOUND_UNRESOLVED
CONTEXT_UNCERTAIN
ACUTE_DEVIATION
SUSTAINED_DEVIATION
CHANGE_POINT_CANDIDATE
TREATMENT_TRANSITION
KNOWN_WITHDRAWAL_MANAGEMENT_PHASE
KNOWN_RETURN_TO_DRINKING_TRANSITION
DEVICE_OR_FIRMWARE_CHANGE
SENSOR_SITE_CHANGE
MODEL_STALE_AFTER_LONG_GAP
EXTREME_MISSINGNESS
```

A downstream clinician/system may later mark a period as a valid new stable regime; only then should the model establish a new baseline.

---

## 13. Multimodal personalization

### Step 1: normalize each feature

Convert each trustworthy modality into a patient/context residual and uncertainty.

### Step 2: deduplicate

Group/reduce correlated features:

```text
ECG HR vs PPG HR
ECG HRV vs PPG PRV
multiple strongly correlated HRV indices
related EDA summaries
```

### Step 3: reliability weighting

Weight by:

```text
current measurement confidence
× personal test-retest/reproducibility
× context coverage
× historical availability
```

### Step 4: covariance-aware joint score

Use shrinkage covariance when enough stable data exist. Otherwise use a diagonal weighted score.

### Step 5: emit both joint and component evidence

Never output only one opaque “autonomic score.” Preserve per-modality deviations and reasons.

---

## 14. Recovery trajectory

Maintain multiple immutable/comparable references:

```text
initial reference
current stable regime
recent 7-day reference
mature ~28-day reference
best stable recovery reference (only once established)
```

The trajectory service should answer:

```text
How does the current stable physiology compare with:
- the beginning of monitoring?
- the prior stable regime?
- the last week?
- the best sustained stable period observed so far?
```

AUD evidence supports using HR/HRV, especially sleep/rest autonomic measures, as candidate recovery trajectories, while emphasizing substantial patient heterogeneity [25–30,37].

---

## 15. Confidence

Do **not** emit one unexplained probability.

Retain:

```text
data sufficiency
recency
context coverage
circadian coverage
residual stability
upstream quality
regime certainty
model fit
```

Then derive:

```text
VERY_LOW / LOW / MEDIUM / HIGH
```

through a conservative calibrated rule.

When confidence is low:

- widen personal prediction intervals;
- shrink personal expectation toward the population prior;
- suppress strong personal-abnormality labels;
- retain raw observations and reference/safety evaluation.

---

## 16. Missing-data handling

### Short gap

Prediction only; uncertainty grows; no baseline update from imputation.

### Missing night/day

No fabricated daily summary. Mark missing.

### Long gap

Set:

```text
PERSONAL_MODEL_STALE
```

Decay recency confidence and reacquire a personal baseline after return.

If hardware changed during the gap, open a new measurement regime.

---

## 17. Population safety/reference bounds

Always maintain:

```text
population_status
personal_status
```

Example:

```yaml
HR:
  current: 105 bpm
  population_status: ABOVE_REFERENCE_OR_SAFETY_BOUND
  personal_status: TYPICAL_FOR_CURRENT_PERSONAL_REGIME
```

This must **not** become:

```text
NORMAL
```

The adaptive baseline describes the patient's expected distribution; it does not cancel external safety information [7,20].

---

## 18. Personal recovery reference promotion

A `BEST_STABLE_RECOVERY_REFERENCE` may be created only when:

- the patient has a sufficiently long externally verified stable-recovery period;
- signal/context coverage is high;
- no ongoing change-point evidence exists;
- distributions are stable;
- the downstream clinical system permits that label.

It should be append-only/versioned. If later deterioration occurs, do not overwrite the best reference.

---

## 19. Recommended implementation pseudocode

```python
for feature_window in incoming_trusted_features:
    f = feature_window.feature
    p = feature_window.patient

    model = load_patient_feature_model(p, f)

    # 1. External/reference path stays independent.
    population_status = evaluate_population_reference(feature_window)

    # 2. Validate personalized inference prerequisites.
    if not feature_window.trusted:
        emit_untrusted(population_status)
        continue

    context = select_context(feature_window)
    baseline, fallback_level = select_best_personal_baseline(model, context)

    # 3. Cold-start shrinkage / uncertainty.
    expected, pred_uncertainty = predict_expected_personal_state(
        model=model,
        context=context,
        timestamp=feature_window.timestamp,
        population_prior=get_population_prior(f, context),
    )

    # 4. Patient-relative deviation.
    residual = feature_window.value - expected
    z = robust_standardize(residual, model, context)
    percentile = empirical_percentile(residual, model, context)

    # 5. Acute and sustained state.
    acute_state = update_persistence_state(model, z, percentile)
    cusum_state = update_cusum(model, z)

    # 6. Detect candidate regime shift, but do not overwrite baseline.
    if cusum_state.threshold_crossed:
        model.change_point = open_or_update_candidate_regime(model)

    # 7. Decide if stable baseline may learn from this window.
    update_allowed = stable_update_gate(
        quality=feature_window.quality,
        confound=feature_window.confound_state,
        context_confidence=context.confidence,
        acute_state=acute_state,
        change_point=model.change_point,
        device_state=feature_window.device_state,
        treatment_transition=feature_window.transition_state,
    )

    update_recent_reference(model, feature_window, allowed=True_or_limited)

    if update_allowed:
        robust_update_stable_baseline(model, feature_window, context)
        update_circadian_sufficient_statistics(model, feature_window, context)

    # 8. Aggregate accepted data into daily/nightly history.
    update_daily_summary_buffer(model, feature_window, context)

    # 9. Emit transparent personal state.
    emit({
        "population_status": population_status,
        "expected_personal": expected,
        "prediction_uncertainty": pred_uncertainty,
        "residual": residual,
        "robust_z": z,
        "personal_percentile": percentile,
        "acute_state": acute_state,
        "persistent_state": cusum_state,
        "baseline_update_allowed": update_allowed,
        "baseline_confidence": model.confidence,
        "reason_codes": derive_reason_codes(...),
    })

    save_model(model)
```

Daily/periodic process:

```python
for patient in patients_with_new_daily_data:
    finalize_robust_daily_summaries(patient)
    refit_or_update_circadian_models_from_stable_data(patient)
    update_state_space_trajectories(patient)
    update_personal_covariance_if_data_sufficient(patient)
    evaluate_candidate_regime_promotions(patient)
    decay_confidence_for_stale_contexts(patient)
    generate_24h_7d_28d_regime_summary(patient)
```

---

## 20. Validation plan before clinical interpretation

The model should be evaluated separately from any AUD diagnosis model.

### A. Baseline prediction

Measure held-out prediction error for:

- global patient mean;
- context-only model;
- circadian-only model;
- context + circadian personal model.

Require the chosen personalization complexity to improve held-out prediction.

### B. Stable-period false-alert calibration

During known stable periods, measure:

```text
alerts per patient-day
fraction of windows outside personal range
CUSUM false change-points
```

Tune persistence/CUSUM thresholds accordingly.

### C. Change-point validation

Use known transitions, simulated shifts, and device-change controls to estimate:

- detection delay;
- false shifts;
- ability to distinguish measurement from physiological regime changes.

### D. Recovery-trajectory validation

For AUD cohorts, compare personal longitudinal features against independently measured clinical/treatment state. Evaluate direction and effect sizes without allowing the physiological model to define the clinical ground truth.

### E. Ablation

Compare:

```text
population only
personal global baseline
personal + context
personal + context + circadian
+ gated adaptation
+ multimodal normalized fusion
```

This is necessary to prove that each layer adds value rather than complexity.

### F. Calibration across patients

A score such as `|z_personal| > threshold` should have approximately comparable false-alert behavior across people if it is going to feed fusion. Test calibration explicitly; do not assume robust standardization guarantees it.

---

## 21. Final design statement

The system to build is therefore:

> **A hybrid population-plus-personal, robust, context- and circadian-conditioned, multi-timescale physiological state model with gated adaptation, sequential change-point detection, regime memory, uncertainty, and reliability-aware multimodal normalization.**

Its central output is not a diagnosis. It is a structured statement of:

```text
what this patient's physiology is now
vs what is expected for this patient now
vs what was expected recently
vs what was expected in prior stable regimes
how long the deviation has persisted
whether the stable baseline itself appears to be shifting
how confident the model is
and whether external population/reference concerns also exist
```

This is the correct bridge between trustworthy/context-corrected physiology and downstream AUD-specific physiological meaning.

---

# References

[1] Li X, Dunn J, Salins D, Zhou G, Zhou W, Schüssler-Fiorenza Rose SM, et al.  
**“Digital Health: Tracking Physiomes and Activity Using Wearable Biosensors Reveals Useful Health-Related Information.”**  
*PLOS Biology*. 2017;15(1):e2001402.  
DOI: 10.1371/journal.pbio.2001402  
PMID: 28081144  
URL: https://pubmed.ncbi.nlm.nih.gov/28081144/

[2] Quer G, Gouda P, Galarnyk M, Topol EJ, Steinhubl SR.  
**“Inter- and intraindividual variability in daily resting heart rate and its associations with age, sex, sleep, BMI, and time of year: Retrospective, longitudinal cohort study of 92,457 adults.”**  
*PLOS ONE*. 2020;15(2):e0227709.  
DOI: 10.1371/journal.pone.0227709  
PMID: 32023264  
URL: https://pubmed.ncbi.nlm.nih.gov/32023264/

[3] Mishra T, Wang M, Metwally AA, Bogu GK, Brooks AW, Bahmani A, et al.  
**“Pre-symptomatic detection of COVID-19 from smartwatch data.”**  
*Nature Biomedical Engineering*. 2020;4(12):1208–1220.  
DOI: 10.1038/s41551-020-00640-6  
PMID: 33208926  
URL: https://pubmed.ncbi.nlm.nih.gov/33208926/

[4] Alavi A, Bogu GK, Wang M, Rangan ES, Brooks AW, Wang Q, et al.  
**“Real-time alerting system for COVID-19 and other stress events using wearable data.”**  
*Nature Medicine*. 2022;28(1):175–184.  
DOI: 10.1038/s41591-021-01593-2  
PMID: 34845389  
URL: https://pubmed.ncbi.nlm.nih.gov/34845389/

[5] Morelli D, Bartoloni L, Rossi A, Clifton DA.  
**“A computationally efficient algorithm to obtain an accurate and interpretable model of the effect of circadian rhythm on resting heart rate.”**  
*Physiological Measurement*. 2019;40(9):095001.  
DOI: 10.1088/1361-6579/ab3dea  
PMID: 31437825  
URL: https://pubmed.ncbi.nlm.nih.gov/31437825/

[6] Hou R, Tomalin LE, Suárez-Fariñas M.  
**“cosinoRmixedeffects: an R package for mixed-effects cosinor models.”**  
*BMC Bioinformatics*. 2021;22:553.  
DOI: 10.1186/s12859-021-04463-3  
PMID: 34773978  
URL: https://pubmed.ncbi.nlm.nih.gov/34773978/

[7] Just SA, Tedeschi E, Holsbø E, et al.  
**“The challenges of implementing hybrid baselines for the interpretation of longitudinal behavioral data from individuals.”**  
*npj Digital Medicine*. 2026;9:331.  
DOI: 10.1038/s41746-026-02668-5  
PMID: 42014851  
URL: https://www.nature.com/articles/s41746-026-02668-5

[8] Nunan D, Sandercock GRH, Brodie DA.  
**“A quantitative systematic review of normal values for short-term heart rate variability in healthy adults.”**  
*Pacing and Clinical Electrophysiology*. 2010;33(11):1407–1417.  
DOI: 10.1111/j.1540-8159.2010.02841.x  
PMID: 20663071  
URL: https://pubmed.ncbi.nlm.nih.gov/20663071/

[9] Plews DJ, Laursen PB, Stanley J, Kilding AE, Buchheit M.  
**“Monitoring training with heart-rate variability: how much compliance is needed for valid assessment?”**  
*International Journal of Sports Physiology and Performance*. 2014;9(5):783–790.  
DOI: 10.1123/ijspp.2013-0455  
PMID: 24334285  
URL: https://pubmed.ncbi.nlm.nih.gov/24334285/

[10] Speed C, Arneil T, Harle R, Wilson A, Karthikesalingam A, McConnell M, Phillips J.  
**“Measure by measure: Resting heart rate across the 24-hour cycle.”**  
*PLOS Digital Health*. 2023;2(4):e0000236.  
DOI: 10.1371/journal.pdig.0000236  
PMID: 37115739  
URL: https://pubmed.ncbi.nlm.nih.gov/37115739/

[11] Natarajan A, Su H-W, Heneghan C, Blunt L, O'Connor C, Niehaus L.  
**“Measurement of respiratory rate using wearable devices and applications to COVID-19 detection.”**  
*npj Digital Medicine*. 2021;4:136.  
DOI: 10.1038/s41746-021-00493-6  
PMID: 34526602  
URL: https://pubmed.ncbi.nlm.nih.gov/34526602/

[12] Harrington N, Torres Barba D, Bui QM, Wassell A, Khurana S, Rubarth RB, et al.  
**“Nocturnal Respiratory Rate Dynamics Enable Early Recognition of Impending Hospitalizations.”**  
*medRxiv* / PMC preprint record. 2022.  
DOI: 10.1101/2022.03.10.22272238  
PMID: 35313571  
URL: https://pubmed.ncbi.nlm.nih.gov/35313571/

[13] Sarabia JA, Rol MA, Mendiola P, Madrid JA.  
**“Circadian rhythm of wrist temperature in normal-living subjects: A candidate of new index of the circadian system.”**  
*Physiology & Behavior*. 2008;95(4):570–580.  
DOI: 10.1016/j.physbeh.2008.08.005  
PMID: 18761026  
URL: https://pubmed.ncbi.nlm.nih.gov/18761026/

[14] Sano A, Picard RW, Stickgold R.  
**“Quantitative analysis of wrist electrodermal activity during sleep.”**  
*International Journal of Psychophysiology*. 2014;94(3):382–389.  
DOI: 10.1016/j.ijpsycho.2014.09.011  
PMID: 25286449  
URL: https://pubmed.ncbi.nlm.nih.gov/25286449/

[15] Sano A, Picard RW.  
**“Toward a taxonomy of autonomic sleep patterns with electrodermal activity.”**  
*Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC)*. 2011:777–780.  
DOI: 10.1109/IEMBS.2011.6090178  
PMID: 22254426  
URL: https://pubmed.ncbi.nlm.nih.gov/22254426/

[16] Crider A, Kremen WS, Xian H, Jacobson KC, Waterman B, Eisen SA, Tsuang MT, Lyons MJ.  
**“Stability, consistency, and heritability of electrodermal response lability in middle-aged male twins.”**  
*Psychophysiology*. 2004;41(4):501–509.  
DOI: 10.1111/j.1469-8986.2004.00189.x  
PMID: 15189473  
URL: https://pubmed.ncbi.nlm.nih.gov/15189473/

[17] Schell AM, Dawson ME, Nuechterlein KH, Subotnik KL, Ventura J.  
**“The temporal stability of electrodermal variables over a one-year period in patients with recent-onset schizophrenia and in normal subjects.”**  
*Psychophysiology*. 2002;39(2):124–132.  
DOI: 10.1017/S0048577202010193  
PMID: 12212661  
URL: https://pubmed.ncbi.nlm.nih.gov/12212661/

[18] Schäfer A, Vagedes J.  
**“How accurate is pulse rate variability as an estimate of heart rate variability? A review on studies comparing photoplethysmographic technology with an electrocardiogram.”**  
*International Journal of Cardiology*. 2013;166(1):15–29.  
DOI: 10.1016/j.ijcard.2012.03.119  
PMID: 22809539  
URL: https://pubmed.ncbi.nlm.nih.gov/22809539/

[19] Han X, Li XE, Volodarskiy A, Slotwiner D, Zhao Y.  
**“Reliability of resting-state photoplethysmogram-based pulse rate variability in lifespan samples: implications for cardiovascular research.”**  
*Physiological Measurement*. 2026;47(5).  
DOI: 10.1088/1361-6579/ae66c7  
PMID: 42055071  
URL: https://pubmed.ncbi.nlm.nih.gov/42055071/

[20] van Rossum MC, Vlaskamp LB, Posthuma LM, Visscher MJ, Breteler MJM, Hermens HJ, Kalkman CJ, Preckel B.  
**“Adaptive threshold-based alarm strategies for continuous vital signs monitoring.”**  
*Journal of Clinical Monitoring and Computing*. 2022;36(2):407–417.  
DOI: 10.1007/s10877-021-00666-4  
PMID: 33575922  
URL: https://pubmed.ncbi.nlm.nih.gov/33575922/

[21] Rosenfield D, et al.  
**“Change point analysis for longitudinal physiological data: detection of cardio-respiratory changes preceding panic attacks.”**  
*Biological Psychology*. 2010;84(1):112–120.  
DOI: 10.1016/j.biopsycho.2010.01.020  
PMID: 20144682  
URL: https://pubmed.ncbi.nlm.nih.gov/20144682/

[22] van de Ven SRB, Gevonden MJ, Huizinga NA, Keur-Huizinga L, van der Mee DJ, Noordzij ML, de Geus EJC.  
**“Detecting physiological stress responses in daily life using additional heart rate: Association with individual characteristics and affective states.”**  
*Biological Psychology*. 2026; article 109369.  
DOI: 10.1016/j.biopsycho.2026.109369  
PMID: 42607858  
URL: https://pubmed.ncbi.nlm.nih.gov/42607858/

[23] Kupper N, Jankovic M, Kop WJ.  
**“Individual Differences in Cross-System Physiological Activity at Rest and in Response to Acute Social Stress.”**  
*Psychosomatic Medicine*. 2021;83(2):138–148.  
DOI: 10.1097/PSY.0000000000000901  
PMID: 33395213  
URL: https://pubmed.ncbi.nlm.nih.gov/33395213/

[24] Bolpagni M, Pardini S, Dianti M, Gabrielli S.  
**“Personalized Stress Detection Using Biosignals from Wearables: A Scoping Review.”**  
*Sensors*. 2024;24(10):3221.  
DOI: 10.3390/s24103221  
PMID: 38794074  
URL: https://pubmed.ncbi.nlm.nih.gov/38794074/

[25] Ralevski E, Petrakis I, Altemus M.  
**“Heart rate variability in alcohol use: A review.”**  
*Pharmacology Biochemistry and Behavior*. 2019;176:83–92.  
DOI: 10.1016/j.pbb.2018.12.003  
PMID: 30529588  
URL: https://pubmed.ncbi.nlm.nih.gov/30529588/

[26] Weise F, Müller D, Krell D, Kielstein V, Koch RD.  
**“Heart rate variability in chronic alcoholics: a follow-up study.”**  
*Drug and Alcohol Dependence*. 1986;17(4):365–368.  
DOI: 10.1016/0376-8716(86)90086-4  
PMID: 3757771  
URL: https://pubmed.ncbi.nlm.nih.gov/3757771/

[27] Yokoyama A, Takagi T, Ishii H, Muramatsu T, Akai J, Kato S, et al.  
**“Impaired autonomic nervous system in alcoholics assessed by heart rate variation.”**  
*Alcoholism: Clinical and Experimental Research*. 1991;15(5):761–765.  
DOI: 10.1111/j.1530-0277.1991.tb00595.x  
PMID: 1661563  
URL: https://pubmed.ncbi.nlm.nih.gov/1661563/

[28] de Zambotti M, Willoughby AR, Baker FC, Sugarbaker DS, Colrain IM.  
**“Cardiac autonomic function during sleep: effects of alcohol dependence and evidence of partial recovery with abstinence.”**  
*Alcohol*. 2015;49(4):409–415.  
DOI: 10.1016/j.alcohol.2014.07.023  
PMID: 25957854  
URL: https://pubmed.ncbi.nlm.nih.gov/25957854/

[29] Eddie D, Pietrzak A, Ham J.  
**“Time Since Last Drink is Positively Associated with Heart Rate Variability in Outpatients with Alcohol Use Disorder: Further Evidence of Psychophysiological Recovery in Early Alcohol Use Disorder Recovery.”**  
*Applied Psychophysiology and Biofeedback*. 2023;48(4):433–437.  
DOI: 10.1007/s10484-023-09597-z  
PMID: 37436518  
URL: https://pubmed.ncbi.nlm.nih.gov/37436518/

[30] Eddie D, Wieman S, Pietrzak A, Zhai X.  
**“In natura heart rate variability predicts subsequent alcohol use in individuals in early recovery from alcohol use disorder.”**  
*Addiction Biology*. 2023;28(8):e13306.  
DOI: 10.1111/adb.13306  
PMID: 37500488  
URL: https://pubmed.ncbi.nlm.nih.gov/37500488/

[31] Alinia P, Sah RK, McDonell M, Pendry P, Parent S, Ghasemzadeh H, Cleveland MJ.  
**“Associations Between Physiological Signals Captured Using Wearable Sensors and Self-reported Outcomes Among Adults in Alcohol Use Disorder Recovery: Development and Usability Study.”**  
*JMIR Formative Research*. 2021;5(7):e27891.  
DOI: 10.2196/27891  
PMID: 34287205  
URL: https://pubmed.ncbi.nlm.nih.gov/34287205/

[32] Drummond SP, Gillin JC, Smith TL, DeModena A.  
**“The sleep of abstinent pure primary alcoholic patients: natural course and relationship to relapse.”**  
*Alcoholism: Clinical and Experimental Research*. 1998;22(8):1796–1802.  
PMID: 9835298  
URL: https://pubmed.ncbi.nlm.nih.gov/9835298/

[33] Bär K-J, Boettger MK, Boettger S, Grotelüschen M, Neubauer R, Jochum T, Baier V, Sauer H, Voss A.  
**“Reduced baroreflex sensitivity in acute alcohol withdrawal syndrome and in abstained alcoholics.”**  
*Drug and Alcohol Dependence*. 2006;85(1):66–74.  
DOI: 10.1016/j.drugalcdep.2006.03.014  
PMID: 16650658  
URL: https://pubmed.ncbi.nlm.nih.gov/16650658/

[34] Bär K-J, Boettger MK, Neubauer R, Grotelüschen M, Jochum T, Baier V, Sauer H, Voss A.  
**“Heart rate variability and sympathetic skin response in male patients suffering from acute alcohol withdrawal syndrome.”**  
*Alcoholism: Clinical and Experimental Research*. 2006;30(9):1592–1598.  
DOI: 10.1111/j.1530-0277.2006.00191.x  
PMID: 16930222  
URL: https://pubmed.ncbi.nlm.nih.gov/16930222/

[35] White JM, Frewin CR, Kaur M, Flavel S, McGregor C.  
**“Twenty-four hour ambulatory monitoring of tremor, sweating, skin temperature and locomotor activity in the alcohol withdrawal syndrome.”**  
*Clinical Autonomic Research*. 1994;4(1-2):15–18.  
DOI: 10.1007/BF01828833  
PMID: 8054832  
URL: https://pubmed.ncbi.nlm.nih.gov/8054832/

[36] Wang M.  
**“Changes in psychophysiological and conditioning variables during chlormethiazole-medicated and unmedicated ethanol withdrawal: preliminary report.”**  
*Acta Psychiatrica Scandinavica Supplementum*. 1986;(329):146–152.  
DOI: 10.1111/j.1600-0447.1986.tb10552.x  
PMID: 3463147  
URL: https://pubmed.ncbi.nlm.nih.gov/3463147/

[37] Timothy A, Sharma S, Sharma S.  
**“Changes in heart rate variability in patients of alcohol dependence syndrome—Do we have a biomarker for craving?”**  
*Indian Journal of Psychiatry*. 2025;67(10):969–975.  
DOI: 10.4103/indianjpsychiatry_405_24  
PMID: 41211495  
URL: https://pubmed.ncbi.nlm.nih.gov/41211495/

[38] Witteman J, Post H, Tarvainen M, de Bruijn A, De Sousa Fernandes Perna E, Ramaekers JG, Wiers RW.  
**“Cue reactivity and its relation to craving and relapse in alcohol dependence: a combined laboratory and field study.”**  
*Psychopharmacology*. 2015;232(20):3685–3696.  
DOI: 10.1007/s00213-015-4027-6  
PMID: 26257163  
URL: https://pubmed.ncbi.nlm.nih.gov/26257163/

---

## Evidence-strength note

The references deliberately combine two evidence classes:

1. **Personalization-method evidence:** wearable longitudinal monitoring, circadian modeling, adaptive alarms, HRV/EDA reliability, and change-point methods [1–24].
2. **AUD-specific longitudinal evidence:** repeated abstinence/withdrawal/recovery and prospective AUD studies [25–38].

Methods transfer across domains; **AUD-specific physiological meaning does not**. A method validated for infection detection, sports monitoring, stress, panic, or postsurgical deterioration is used here only for its longitudinal modeling principle, not for its original disease threshold.

---

## Final boundary

This implementation is designed to state, with explicit uncertainty:

> **“This physiological state is unusual for this patient in this context and relative to this point in their longitudinal trajectory.”**

It is intentionally not designed to state, by itself:

> **“This patient is in withdrawal,” “this patient will relapse,” or “this treatment should be changed.”**

Those claims require independently validated AUD-specific and clinical interpretation downstream.
