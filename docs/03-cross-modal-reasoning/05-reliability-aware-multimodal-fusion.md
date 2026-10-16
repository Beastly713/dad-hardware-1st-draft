# Reliability-Aware Multimodal Physiological Fusion

**Technical research/design reference for continuous AUD/SUD physiological monitoring**  
**Version:** 1.0  
**Scope:** combining already-processed physiological evidence from ECG, PPG, EDA, temperature, respiration, and IMU/context information when modalities agree, disagree, disappear, arrive asynchronously, or have different confidence.

---

## Executive design decision

The fusion layer should **not** be a single feature-concatenation model and should **not** assign one static weight per sensor. The strongest practical architecture for this system is a **hierarchical, reliability-aware, pathway-aware probabilistic fusion system with temporal state tracking**.

The recommended architecture has three distinct fusion stages:

1. **Redundant-measurement fusion inside physiological branches.** Signals that estimate substantially the same physiological quantity (for example ECG heart rate and PPG pulse rate, or dedicated respiration and ECG/PPG-derived respiration) are compared, cross-validated, and fused with quality-dependent measurement uncertainty. This is where signal-quality-aware Kalman/state-space methods are most defensible [1–5].
2. **Complementary physiological-state fusion across branches.** Cardiac, electrodermal, respiratory, and peripheral/thermal branch states are not averaged. They provide heterogeneous evidence about a target latent state such as autonomic activation. The target model combines branch posteriors/likelihoods while explicitly retaining uncertainty, correlations, missingness, and disagreement. Multimodal stress studies show that complementary modalities can improve inference, but the gain is task- and dataset-dependent and often shrinks outside controlled laboratory settings [6–10].
3. **Temporal fusion of the target state.** A state-space/HMM-style update uses recent posterior state, event timing, and modality-specific response delays so that the system does not treat each window as independent. Fast event-level, medium state-level, and long trend-level estimates are kept separate rather than forcing seconds, minutes, and days into one score [6,11–14].

A useful name for the implementation is **Hierarchical Reliability-Aware Dynamic Evidence Fusion (HRA-DEF)**.

The key operational rule is:

> **Reliability controls how much a measurement is allowed to influence inference; physiological evidence controls what that measurement says; these are separate variables.**

A high-quality EDA increase is strong evidence that *electrodermal sympathetic activity changed*. It is not automatically high-confidence evidence that a specific AUD state is present. Conversely, a low-quality PPG stream should not veto good ECG evidence merely because its pulse estimate differs.

---

# 1. Position in the overall pipeline

The system should use **fusion at multiple levels**, not a single fusion block.

```text
body
  ↓
sensing / acquisition
  ↓
signal trustworthiness and modality quality
  ↓
physiological decomposition / feature extraction
  ↓
confound handling + context-relative interpretation
  ↓
┌──────────────────────────────────────────────────────────────┐
│ FUSION STAGE A: redundant-measurement / branch fusion       │
│ ECG ↔ PPG cardiac timing; dedicated ↔ derived respiration  │
└──────────────────────────────────────────────────────────────┘
  ↓
pathway-specific physiological states + uncertainty
  ↓
┌──────────────────────────────────────────────────────────────┐
│ FUSION STAGE B: complementary target-specific fusion        │
│ cardiac + EDA + respiratory + peripheral/thermal evidence  │
└──────────────────────────────────────────────────────────────┘
  ↓
latent physiological state(s)
  ↓
AUD-specific physiological interpretation
  ↓
┌──────────────────────────────────────────────────────────────┐
│ FUSION STAGE C: optional AUD-evidence fusion                │
│ only where AUD-specific multimodal evidence is validated    │
└──────────────────────────────────────────────────────────────┘
  ↓
temporal state / trend model
  ↓
final AUD-specific clinical interpretation layer
```

This ordering resolves an apparent ambiguity in a single linear pipeline. Some fusion belongs **before** AUD-specific interpretation because it creates a more robust estimate of generic physiology. Some fusion may also belong **after** AUD-specific interpretation when several modality-specific AUD evidence channels exist and have been independently validated. The latter should be introduced conservatively because AUD-specific multimodal datasets remain much smaller and less mature than generic wearable stress/autonomic datasets [15–23].

### What should not be fused early

Do not fuse raw ECG, PPG, EDA, temperature, and respiration samples into one common waveform. They have different units, physiology, bandwidth, morphology, delay, and failure modes. Raw/signal-level fusion is appropriate mainly for **redundant measurements of the same latent quantity** (for example, ECG and PPG contributions to cardiac timing or respiration estimation) and has good quality-aware evidence [1–5].

---

# 2. Fusion input contract

Every input to fusion is a timestamped **evidence object**, not merely a scalar feature.

```text
ModalityEvidence {
    modality                 // ECG, PPG, EDA, TEMP, RESP, IMU_CONTEXT
    physiological_pathway    // cardiac, electrodermal, respiratory, peripheral, context
    feature_name
    value
    unit
    baseline_normalized_value
    direction                // increase / decrease / neutral if interpretable
    window_start
    window_end
    effective_timestamp      // corrected for known feature latency
    update_time

    signal_quality           // upstream measurement quality
    feature_validity         // confidence that feature extraction is valid
    confound_validity        // confidence remaining after motion/activity/etc correction
    context_validity         // confidence in contextual interpretation

    predictive_variance      // preferred reliability representation after calibration
    availability             // observed / stale / missing
    source                   // directly measured / derived / reconstructed
    missingness_reason       // if known
}
```

The fusion engine should not interpret `signal_quality = 0.8` from two modalities as equivalent unless those scores have been calibrated against modality-specific error. Section 6 defines the reliability transformation.

## 2.1 ECG inputs

| Input | Meaning | Unit | Recommended evidence window | Update | Role |
|---|---|---:|---|---|---|
| HR | beat-to-beat cardiac rate | bpm | 5–15 s robust aggregate | 1 s | derived physiology |
| RR/NN interval statistics | cardiac timing | ms | beat-level / 10–60 s | 1 s | derived physiology |
| RMSSD | short-term vagal-sensitive variability | ms | preferably 30–120 s for dynamic monitoring; 5 min for standardized short-term HRV | 5–10 s | derived physiology |
| SDNN | variability | ms | preferably ≥30–120 s; 5 min standardized | 5–10 s | derived physiology |
| frequency-domain HRV | LF, HF etc. | ms² / normalized | standard 5 min where feasible | 30–60 s | derived physiology |
| cardiac activation evidence | normalized target-specific branch score | z/log-likelihood | target-specific | 1–10 s | state evidence |
| ECG predictive variance | expected feature error given current quality | feature-unit² | same as feature | same | reliability |

Five-minute windows remain the conventional standard for short-term frequency-domain HRV; ultra-short RMSSD/SDNN can be useful for engineering applications but should not be silently treated as equivalent to standardized 5-min measures [24–26].

## 2.2 PPG inputs

| Input | Meaning | Unit | Recommended evidence window | Update | Role |
|---|---|---:|---|---|---|
| pulse rate | peripheral pulse timing | bpm | 5–15 s | 1 s | redundant cardiac evidence |
| pulse interval variability | peripheral interval variability | ms | 30–120 s+ | 5–10 s | conditionally redundant cardiac evidence |
| pulse amplitude | peripheral vascular pulse amplitude | device/normalized | 10–60 s | 1–5 s | peripheral evidence |
| perfusion/waveform features | peripheral vascular state | device-specific | 10–60 s | 5 s | peripheral evidence |
| PPG-derived respiration, if validated | respiratory surrogate | breaths/min | 30–60 s | 5–10 s | redundant respiratory evidence |
| PPG predictive variance | feature error conditional on quality | feature-unit² | matched | matched | reliability |

PPG pulse rate should generally be considered **redundant** with ECG heart rate for the purpose of detecting a common cardiac rate change; pulse amplitude/perfusion is **complementary** because it carries peripheral vascular information not present in ECG.

## 2.3 EDA inputs

| Input | Meaning | Unit | Recommended evidence window | Update | Role |
|---|---|---:|---|---|---|
| tonic SCL | slower electrodermal level | µS / baseline-normalized | 30 s–5 min | 5–10 s | electrodermal state |
| phasic driver/activity | fast sympathetic sudomotor activity | normalized | seconds–30 s | 1 s | electrodermal event evidence |
| SCR count/frequency | response frequency | events/min | 30–60 s | 5–10 s | electrodermal state |
| SCR amplitude | phasic response magnitude | µS | event-level | event | electrodermal event evidence |
| EDA activation state | modality-specific state posterior/evidence | probability/log-odds | 10–60 s | 1–5 s | state evidence |
| EDA predictive variance | expected evidence error | calibrated | same | same | reliability |

Stimulus-elicited EDA responses have a characteristic onset latency commonly evaluated in roughly the **1–4 s** interval after a stimulus, so event fusion should allow EDA to lag cardiac/context events rather than requiring simultaneous peaks [27].

## 2.4 Temperature inputs

| Input | Meaning | Unit | Recommended evidence window | Update | Role |
|---|---|---:|---|---|---|
| skin temperature | peripheral thermal state | °C | 30–120 s | 2–10 s | peripheral state |
| deviation from personal baseline | subject-relative thermal change | °C or z | 1–10 min | 10–30 s | peripheral evidence |
| trend/slope | slow vasomotor/environmental trend | °C/min | 1–10 min | 10–30 s | trend evidence |
| temperature predictive variance | confidence after ambient/contact correction | calibrated | matched | matched | reliability |

Temperature should not be used as a fast event detector. It is a slower supporting channel and is highly context-sensitive.

## 2.5 Respiration inputs

| Input | Meaning | Unit | Recommended evidence window | Update | Role |
|---|---|---:|---|---|---|
| respiratory rate | breathing rate | breaths/min | 20–60 s | 5–10 s | respiratory state |
| variability | breath-to-breath variability | calibrated | 30–120 s | 10 s | respiratory state |
| inspiration/expiration features | respiratory pattern | ratio/duration | 30–60 s | 5–10 s | respiratory evidence |
| respiratory activation score | target-specific state evidence | probability/log-odds | 30–60 s | 5–10 s | state evidence |
| respiratory predictive variance | feature error conditional on quality | calibrated | matched | matched | reliability |

If respiration is derived from ECG/PPG rather than directly measured, the fusion object must be marked `source=derived` and its error covariance must reflect dependence on the parent signal. It must **not** be counted as an independent modality.

## 2.6 IMU/context inputs

IMU/context does not usually vote directly for an autonomic state. It changes whether physiological evidence is interpretable.

```text
activity_level
posture
recent_activity_decay
motion_artifact_risk
sleep/rest/wake context
ambient/thermal context if available
known cue/event context if available
context confidence
```

IMU is therefore primarily a **conditioning/gating variable**. Activity-aware stress studies show that adding activity/context changes real-world performance materially [8,9].

---

# 3. Correct fusion level

| Fusion level | Strength | Main weakness | Missing modality | Interpretability | Training burden | Recommendation |
|---|---|---|---|---|---|---|
| Raw/signal-level | Can improve a shared waveform/estimate | Invalid across heterogeneous physiology | poor unless custom | medium | low–medium | **Use only for redundant quantities** |
| Feature-level concatenation | Simple and often strong in controlled data | Double-counts correlation; brittle missingness; hard to explain weights | poor by default | medium | medium | **Not primary architecture** |
| Learned representation fusion | Captures nonlinear interactions | data hungry; calibration/generalization problems | variable | low | high | research/phase-2 only |
| State-level fusion | physiology-aligned; heterogeneous modalities comparable | requires good per-branch states | good | high | medium | **Primary cross-pathway fusion** |
| Decision/evidence-level | robust modularity; easy missingness | loses some low-level interactions | very good | very high | low–medium | **Use for AUD evidence where validated** |
| Hierarchical fusion | balances redundancy, complementarity, time | engineering complexity | very good | high | medium | **Recommended** |

The recommendation is therefore **hierarchical fusion**: quality-aware redundant fusion within branches, followed by target-specific state/evidence fusion, followed by temporal filtering. The 2026 wearable-fusion review similarly emphasizes that fusion level and signal quality must be matched to the application rather than assuming more sensors or earlier fusion is always better [5].

---

# 4. Physiological pathway-aware organization

Use the following branch structure.

```text
CARDIAC BRANCH
    ECG HR / RR / HRV
    PPG pulse rate / PPI
        ↓ redundant-consistency logic
    cardiac state posterior + covariance

ELECTRODERMAL BRANCH
    tonic EDA
    phasic EDA
        ↓
    electrodermal state posterior + covariance

RESPIRATORY BRANCH
    dedicated respiration
    ECG-derived respiration (optional)
    PPG-derived respiration (optional)
        ↓ redundancy + dependence-aware fusion
    respiratory state posterior + covariance

PERIPHERAL / THERMAL BRANCH
    skin temperature
    PPG pulse amplitude / perfusion
        ↓
    peripheral state posterior + covariance

CONTEXT / CONFOUND BRANCH
    IMU/activity/posture/rest/sleep/event context
        ↓
    conditioning variables + confidence
```

This organization prevents the target model from seeing ECG HR and PPG pulse rate as two fully independent votes for the same cardiac activation. It also makes explainability clinically and physiologically meaningful: the final system can say that **cardiac and electrodermal pathways agree while the peripheral branch is neutral**, rather than listing dozens of raw features.

---

# 5. Redundancy, complementarity, and dependence

## 5.1 Redundant pairs

| Pair | Relationship | Fusion behavior |
|---|---|---|
| ECG HR ↔ PPG pulse rate | strongly overlapping cardiac timing | cross-validate; do not count twice |
| ECG RR ↔ PPG pulse interval | overlapping timing with pulse-transit effects | compare after lag tolerance; ECG usually timing primary |
| dedicated respiration ↔ ECG-derived respiration | same target, different source | quality-aware redundant fusion |
| dedicated respiration ↔ PPG-derived respiration | same target | quality-aware redundant fusion |
| ECG-derived respiration ↔ PPG-derived respiration | conditionally redundant and correlated with parent cardiac signals | track source dependence; avoid independent-vote assumption |

## 5.2 Complementary pairs

| Pair | Why complementary |
|---|---|
| cardiac ↔ EDA | cardiac autonomic response and sudomotor sympathetic response are different physiological manifestations |
| cardiac ↔ respiration | coupled but non-identical autonomic/respiratory response |
| EDA ↔ respiration | different autonomic pathways and response kinetics |
| temperature/perfusion ↔ cardiac | peripheral vasomotor/thermal information adds information not contained in ECG timing |
| EDA ↔ temperature/perfusion | sympathetic sudomotor vs peripheral vasomotor effects |

## 5.3 Conditional redundancy

Correlation depends on feature and context. PPG pulse rate may be highly redundant with ECG HR at rest while PPG amplitude remains complementary. ECG-derived respiration is not an independent respiratory sensor because it is calculated from ECG morphology/beat variation. Similarly, HRV and respiration are physiologically coupled, so a model using both must learn or explicitly represent that dependence rather than multiplying naive independent likelihoods.

---

# 6. Reliability as a first-class variable

## 6.1 Do not multiply arbitrary confidence scores

A tempting formula is:

```text
reliability = signal_quality × decomposition_confidence × confound_confidence × context_confidence
```

Do **not** make this the production rule. These upstream scores have different calibration and may be statistically dependent. Multiplication can collapse confidence excessively and gives a false probabilistic interpretation.

## 6.2 Preferred representation: predictive error distribution

For every modality-feature pair, learn a reliability model from validation data:

```text
q_m(t) = [signal_quality,
          feature_validity,
          confound_validity,
          context_validity,
          activity,
          staleness,
          sensor_state]

σ²_m(t) = g_m(q_m(t))
```

where `σ²_m(t)` is the **expected squared error / predictive variance** of that feature or evidence estimate under the current quality conditions.

Equivalently, for bounded validity tasks:

```text
r_m(t) = P(|error_m| ≤ clinically/engineering acceptable tolerance | q_m(t))
```

This is much more useful than an arbitrary 0–1 quality number because state-space and Bayesian algorithms can consume variance directly. Li et al. and Nemati et al. demonstrate the same principle: signal-quality information modifies the measurement/update uncertainty so poor observations influence the fused estimate less [1,2]. Modern quality-aware ECG/PPG fusion uses instantaneous quality as an adaptive fusion weight [4].

## 6.3 Cold-start before calibration data exist

Until enough labeled quality/error data are available:

1. Apply **hard invalidity gates** for known unusable windows.
2. Map each upstream QC state to conservative categories: `HIGH`, `MEDIUM`, `LOW`, `INVALID`.
3. Assign deliberately broad measurement variance to `LOW`, not a fabricated precise probability.
4. Store the raw upstream quality vector so the mapping can later be recalibrated.

Do not expose numerical confidence to users until reliability calibration has been evaluated.

---

# 7. Dynamic modality weighting

For **redundant scalar measurements** of the same quantity, use a quality-aware state-space update.

Let the latent quantity be `x_t`, with measurement from modality `m`:

```text
z_m,t = H_m x_t + ε_m,t
ε_m,t ~ N(0, R_m,t)
```

`R_m,t` is the calibrated measurement-error covariance derived from current quality. The Kalman gain automatically reduces the influence of a noisy source:

```text
K_t = P^-_t Hᵀ (H P^-_t Hᵀ + R_t)^-1
```

A lower `R_m,t` means a more trusted measurement; a larger `R_m,t` means a weaker update. If the modality is missing, omit that measurement from the update. This directly implements the quality-aware logic demonstrated in multi-source heart-rate and respiration work [1,2].

For **complementary branches**, do not normalize qualities and average branch scores. Instead, quality is carried into each branch posterior/variance and the target model consumes the uncertain branch state.

---

# 8. Cross-validation between redundant sensors

ECG and PPG should validate one another, but should not be blindly averaged.

For two redundant estimates `z_i, z_j` with predictive variances `σ²_i, σ²_j`, calculate:

```text
D_ij = (z_i - z_j)^2 / (σ²_i + σ²_j + 2·cov_error_ij)
```

This is a normalized disagreement statistic. In the Gaussian case it can be tested against the appropriate chi-square distribution rather than choosing an arbitrary raw difference threshold.

### Behavior

**Both high reliability + agreement**  
→ fuse; reduce branch uncertainty; emit `REDUNDANT_SENSOR_AGREEMENT`.

**High ECG reliability + low PPG reliability + disagreement**  
→ retain ECG; strongly down-weight/omit PPG; emit `PPG_LOW_RELIABILITY_DISAGREEMENT`.

**Both high reliability + disagreement**  
→ do not force a compromise. Flag `REDUNDANT_SENSOR_CONFLICT`, inflate branch covariance, inspect timing/lag, ectopy/pulse-transit/measurement-specific causes, and withhold high-confidence branch inference until resolved.

**ECG missing + good PPG**  
→ PPG may provide fallback pulse-rate evidence; mark source substitution and increase branch uncertainty because ECG-specific HRV/morphology evidence is unavailable.

Johnson et al. showed that quality-guided selection between redundant cardiac sources can preserve beat detection when ECG is noisy or absent; their external evaluation reported 95.1% sensitivity and 89.3% positive predictive value [3].

---

# 9. Agreement between complementary modalities

For complementary branches, agreement means their **target-specific likelihood contributions point in the same direction**, not that their numeric values match.

For target `S` (e.g., autonomic activation), branch `k` contributes a calibrated log-likelihood ratio:

```text
ℓ_k(t) = log P(B_k(t) | S=1, context) / P(B_k(t) | S=0, context)
```

A simple independent-evidence model would add log-likelihoods, but physiological branches are not fully independent. Therefore the production model should use either:

- a target-specific Bayesian/factor model with dependencies; or
- an interpretable calibrated logistic model over **branch-level** states, including interaction/correlation terms.

The second is easier to deploy initially:

```text
logit P(S_t=1) = β0
                + βC · Bcardiac
                + βE · Belectrodermal
                + βR · Brespiratory
                + βP · Bperipheral
                + βCE · Bcardiac·Belectrodermal
                + βCR · Bcardiac·Brespiratory
                + context terms
```

The model should be fit with subject-independent validation and calibrated afterward. It creates increased probability when reliable complementary branches agree without pretending the channels are independent.

### Why not additive hand scoring?

A hand score such as `EDA + HR - HRV` is interpretable but gives no principled treatment of unequal uncertainty, correlations, population variability, or missingness. Use hand rules for **gating and reason codes**, not as the final probability model.

---

# 10. Disagreement and conflict-aware fusion

The system must distinguish **sensor conflict** from **physiological heterogeneity**.

## 10.1 Type A: redundant-sensor conflict

Example:

```text
ECG HR = elevated, high quality
PPG pulse = normal, high quality
```

This is a measurement-consistency problem. Use the normalized innovation/disagreement test above, temporal alignment check, and covariance inflation. Do not interpret it as a real autonomic pattern until the branch conflict is resolved.

## 10.2 Type B: complementary physiological disagreement

Example:

```text
EDA activation = high, high reliability
cardiac activation = neutral, high reliability
respiratory activation = neutral, high reliability
```

This can be a legitimate modality-specific response. Do not suppress EDA and do not force the global state to “high.” The output should retain:

```text
state_probability: moderate
confidence: moderate
status: MODALITY_SPECIFIC_RESPONSE
support: electrodermal
neutral: cardiac, respiratory
```

## 10.3 Type C: low-quality disagreement

Example:

```text
EDA high reliability → activation
ECG low reliability → no activation
```

The low-quality neutral evidence should have little veto power. Emit a reason code showing that it was down-weighted.

## 10.4 Conflict index

For explainability, define a non-probabilistic conflict diagnostic:

```text
C = 1 - |Σ c_k| / (Σ |c_k| + ε)
```

where `c_k` is the signed branch contribution to the target. `C≈0` means aligned evidence; larger values mean cancellation. **Do not use a universal clinical cutoff.** Set alert labels from validation distributions and target-specific operating points.

### Output states

Use explicit semantic states:

- `HIGH_RELIABILITY_CONSENSUS`
- `LOW_CONFIDENCE`
- `AMBIGUOUS`
- `CONFLICTING_EVIDENCE`
- `MODALITY_SPECIFIC_RESPONSE`
- `INSUFFICIENT_EVIDENCE`
- `TEMPORAL_MISALIGNMENT`
- `REDUNDANT_SENSOR_CONFLICT`

---

# 11. Missing-modality handling

Missing sensors are expected in wearable systems and should be designed into inference rather than treated as an exception [12,13].

## 11.1 Production rule

**Do not impute a missing modality and then treat the imputed value as observed evidence.**

The target model should be capable of marginalizing/omitting absent observations:

```text
P(S | observed branches)
```

rather than requiring every input.

At branch level, a Kalman/state-space filter simply skips the missing measurement update and propagates state uncertainty forward. At target level, a probabilistic model uses the available branch posteriors and increases uncertainty/lowers evidence coverage.

## 11.2 Missingness metadata

Every missing event should carry:

```text
missing_reason = {
    contact_loss,
    motion_corruption,
    sweating,
    device_failure,
    packet_loss,
    battery_saving,
    not_measured,
    unknown
}
```

Missingness can be **informative**. PPG may disappear during motion; EDA can degrade during sweating; these contexts can correlate with the physiological state. The observation process must therefore be logged and may enter the model as a covariate rather than being assumed missing completely at random [28,29].

## 11.3 Reconstruction is auxiliary, not evidence

Self-supervised missing-modality reconstruction can improve representation robustness, and 2026 work explicitly trains wearable models with modality dropout and reconstruction on WESAD/MobiAct [12]. If adopted later, reconstructed data must remain marked `source=reconstructed` and should not receive the same evidentiary weight as observed data unless reconstruction uncertainty is validated.

---

# 12. Sensor hierarchy and fallback

| Quantity | Primary | Secondary/fallback | Rule |
|---|---|---|---|
| beat timing / HR | ECG | PPG | ECG timing preferred when both valid; PPG fallback if ECG invalid/missing |
| HRV | ECG NN intervals | PPG PPI only if validated for the specific metric/context | do not mix metric definitions silently |
| peripheral pulse/perfusion | PPG | none | no ECG substitute |
| electrodermal activity | EDA | none | absence is true loss of this pathway |
| skin temperature | temperature sensor | none | no synthetic replacement |
| respiration | dedicated respiration if present | ECG/PPG-derived respiration | derived channels carry parent dependence and larger uncertainty |
| activity/motion context | IMU | contextual software sources | used for conditioning, not autonomic vote |

The hierarchy is target-specific. “ECG primary” means primary for cardiac timing, not universally more important than EDA or respiration for every physiological target.

---

# 13. Temporal alignment

Fusion should operate on **physiological time**, not merely packet time.

Each evidence object stores:

```text
observation_window = [start, end]
feature_timestamp = end or center
expected_latency_distribution
staleness
```

## 13.1 EDA

For discrete stimuli, event-related EDA onset is commonly evaluated around 1–4 s after stimulus onset [27]. Therefore a cardiac change at `t` and an EDA response at `t+2.5 s` can be concordant rather than asynchronous conflict.

## 13.2 Cardiac rate vs HRV

HR can update rapidly from beat timing. HRV is a statistical property of a sequence and requires a longer observation interval. Standard short-term HRV commonly uses 5-min recordings; ultra-short RMSSD can be estimated over shorter windows but carries greater uncertainty and is not equivalent to standardized HRV [24–26].

## 13.3 Respiration

Respiratory state should be estimated over multiple breaths. Use approximately 20–60 s event/state windows for rate/pattern, with a longer buffer where variability is required. cStress used one-minute ECG/respiration windows and obtained strong lab performance, then lower but still useful field performance, illustrating the value of a moderate physiological state window [6].

## 13.4 Temperature

Treat temperature as a slow peripheral trend. Do not expect its peak to coincide with EDA or HR. Use minute-scale alignment and trend features rather than sample-level synchrony.

## 13.5 Alignment algorithm

For each target inference tick:

1. Select the most recent non-stale evidence object from each branch.
2. Shift/associate event evidence using its known latency distribution.
3. Permit target-specific tolerance windows rather than exact timestamps.
4. For event-level analysis, optionally estimate local cross-correlation lag only inside physiologically plausible bounds.
5. If the best alignment exceeds the allowed bound, emit `TEMPORAL_MISALIGNMENT` instead of treating the modalities as contradictory.

---

# 14. Multi-timescale fusion

Use three concurrent state products.

## 14.1 Fast event layer

**Purpose:** detect transient physiological reactions.  
**Fusion tick:** ~1 s.  
**Evidence windows:** approximately 5–30 s depending on modality.  
**Examples:** phasic EDA response, rapid HR increase, respiratory perturbation, cue-linked event.

## 14.2 Medium state layer

**Purpose:** estimate sustained autonomic state.  
**Fusion tick:** 5–10 s.  
**Evidence windows:** approximately 30 s–5 min.  
**Examples:** sustained sympathetic activation, parasympathetic suppression, short recovery state.

## 14.3 Long trend layer

**Purpose:** estimate change relative to personal baseline over hours/days.  
**Fusion tick:** 1–5 min or event summaries.  
**Evidence horizon:** tens of minutes to days.  
**Examples:** sleep/recovery deterioration, progressive normalization during abstinence, persistent autonomic dysregulation.

The long layer should consume **summaries of calibrated medium-state posteriors**, not raw high-rate samples. AUD sleep/recovery studies demonstrate that clinically meaningful autonomic changes can operate over days to months, which is fundamentally different from seconds-scale EDA events [20,21].

---

# 15. Fusion targets: do not create one global physiology score

The fusion engine is invoked **per target**. Recommended generic targets:

1. `AUTONOMIC_ACTIVATION`
2. `SYMPATHETIC_ELECTRODERMAL_ACTIVATION`
3. `PARASYMPATHETIC_SUPPRESSION`
4. `CARDIOVASCULAR_ACTIVATION`
5. `PERIPHERAL_AROUSAL`
6. `RESPIRATORY_ACTIVATION`
7. `RECOVERY_STATE`

AUD/SUD-specific targets should be separate calibrated models:

8. `WITHDRAWAL_RELATED_PHYSIOLOGICAL_EVIDENCE`
9. `CUE_CRAVING_REACTIVITY_EVIDENCE`
10. `INTOXICATION_RECOVERY_EVIDENCE`
11. `AUTONOMIC_DYSREGULATION_EVIDENCE`
12. `SLEEP_RECOVERY_DETERIORATION_EVIDENCE`

The suffix **EVIDENCE** is deliberate. Physiology should not independently assert a diagnosis, medication need, relapse event, or hospitalization decision.

---

# 16. Target-modality map

Legend: **P = primary**, **S = secondary**, **O = optional/supporting**, **C = context/gating**, **– = generally not used directly**.

This table is a starting architecture, not a validated AUD diagnostic specification. AUD-specific assignments are deliberately conservative because individual physiological correlates can be weak or heterogeneous [17–23].

| Target | ECG | PPG | EDA | Temp | Resp | IMU/context | Minimum evidence rule |
|---|---:|---:|---:|---:|---:|---:|---|
| Autonomic activation | P | S | P | O | P | C | ≥2 independent physiological pathways, unless a target-specific model supports otherwise |
| Sympathetic electrodermal activation | O | O | P | S | O | C | EDA required; others contextual/supporting |
| Parasympathetic suppression | P | S | – | – | S | C | valid cardiac variability + stable context; respiration strongly recommended |
| Cardiovascular activation | P | S | O | O | O | C | ECG or valid PPG; ECG preferred for HRV |
| Peripheral arousal | S | P | S | P | O | C | ≥1 peripheral measure + context; complementary cardiac/EDA support |
| Respiratory activation | O | O | O | – | P | C | valid respiration source |
| Recovery state | P | S | O | O | S | C | cardiac + context; respiration desirable |
| Withdrawal-related physiological evidence | P | S | S | O | S | C | do not infer from one channel; ≥2 pathways + AUD/withdrawal context |
| Cue/craving-reactivity evidence | S | O | P | O | P | C | event context + ≥1 responsive pathway; ≥2 preferred |
| Intoxication/recovery evidence | S | P | S | P | O | C | multimodal model + alcohol exposure/context required |
| Autonomic dysregulation | P | S | O | O | S | C | longitudinal cardiac evidence; multimodal support |
| Sleep/recovery deterioration | P | S | O | O | S | C | sleep/rest context + cardiac evidence; repeated nights preferred |

### AUD-specific caution

- Acute alcohol withdrawal research shows that HR/HRV/autonomic metrics do not all move in a simple monotonic sympathetic direction; some classical HRV and sympathetic skin response measures have failed to capture expected changes [18,19]. This is a strong reason **not** to hard-code `HR↑ + HRV↓ = withdrawal`.
- A 2026 VR alcohol-cue study in 61 alcohol-dependent patients observed cue-related changes in nonspecific SCR frequency and respiration rate, while HR/HRV were not significant in that protocol [22]. This supports EDA/respiration as important cue-reactivity channels but also demonstrates target-specific heterogeneity.
- Wearable SUD monitoring in 30 treatment participants using HR, EDA, skin temperature, and accelerometry achieved AUC ≈0.82 for stress and craving discrimination, showing feasibility but not proving a universal physiological signature [15].
- A 2026 intoxication feasibility study (N=28) using HR, skin temperature, EDA, and accelerometry reported up to 0.80 accuracy, supporting multimodal intoxication inference but still requiring larger external validation [23].

---

# 17. Agreement scoring and posterior update

Agreement should increase certainty only when the agreeing evidence is both **reliable** and **not merely duplicated**.

For a binary target state `S`, a branch-level evidence model can be written as:

```text
p_t = P(S_t = 1 | B_t, C_t)
```

where `B_t` contains pathway-level branch states and `C_t` contains context. Rather than multiplying modality likelihoods under a false independence assumption, fit a calibrated logistic/Bayesian model over the branch states and their selected interactions.

A practical initial model is:

```text
η_t = β0
    + βc * cardiac_state
    + βe * electrodermal_state
    + βr * respiratory_state
    + βp * peripheral_state
    + βce * cardiac_state * electrodermal_state
    + βcr * cardiac_state * respiratory_state
    + βer * electrodermal_state * respiratory_state
    + βctx * context_features

p_t = sigmoid(η_t)
```

The branch means are accompanied by covariance/uncertainty. During inference, uncertainty can be propagated by Monte Carlo sampling from the branch posterior distributions or by an approximation suitable for the deployed model. This produces both a target probability and uncertainty due to sensor quality.

### Why this is stronger than voting

A majority vote assumes comparable detectors and often implicit independence. That is not true here: ECG HR and PPG pulse rate are correlated, HRV depends on respiration, and some modalities are less relevant to specific targets. The 2026 wearable fusion review notes that weighted voting and Dempster–Shafer are available choices, but also emphasizes quality-aware fusion and the danger that poor channels can degrade the fused result [5].

---

# 18. Conflict-aware posterior behavior

The final posterior should not be the only output. Keep a **signed contribution vector** and a **conflict summary**.

For an interpretable logistic target model, use contribution terms:

```text
contribution_k = β_k * B_k
```

and interaction contributions separately. Then classify each branch:

```text
supporting: contribution_k > positive_validation_threshold
neutral:    |contribution_k| <= neutral_band
conflicting: contribution_k < negative_validation_threshold
```

The thresholds must come from the validation distribution and operating objective; do not hard-code universal numbers.

### Example

```text
EDA branch:          +0.92 log-odds contribution
cardiac branch:      +0.71
respiratory branch:  +0.43
peripheral branch:   -0.08

result:
    strong multimodal agreement
    peripheral branch neutral
```

versus:

```text
EDA branch:          +0.88
cardiac branch:      -0.71
respiratory branch:  -0.34
peripheral branch:   +0.05

result:
    conflicting physiological evidence
    fused probability may be near 0.5
    confidence must not be reported as high merely because all signals were high quality
```

This distinction is critical: **measurement confidence** and **state certainty** are not the same thing. High-quality sensors can produce high-confidence disagreement.

---

# 19. Bayesian fusion: where it fits

Bayesian inference is well suited because it naturally represents uncertainty, priors, missing observations, and temporal state.

A generic formulation is:

```text
P(S_t | Z_1:t, R_1:t, C_1:t)
```

with:

- `S_t`: latent target physiological state;
- `Z`: observed branch/modal evidence;
- `R`: reliability/error information;
- `C`: context/confound state.

## Recommended use

Use Bayesian/state-space reasoning at **two places**:

1. **Branch state estimation**, especially for redundant quantities, where measurement variance is directly tied to reliability [1,2].
2. **Temporal target-state update**, where the prior is the previous state estimate and the likelihood is current multimodal evidence.

A minimal binary temporal model is:

```text
P(S_t | evidence_t) ∝ P(evidence_t | S_t) * P(S_t | S_{t-1})
```

The transition term discourages implausible one-window state flipping without preventing genuine rapid events.

## Conditional independence

Do not assume `ECG ⟂ PPG | S` when the features share cardiac timing, and do not assume HRV is independent of respiration. The branch architecture reduces these dependencies before cross-pathway fusion. Remaining dependencies should be represented with interaction terms or a factor graph.

---

# 20. Evidence theory / Dempster–Shafer

Dempster–Shafer theory is attractive because it can explicitly represent support, opposition, and ignorance. It is useful conceptually for outputs such as:

```text
support activation: 0.60
support no activation: 0.10
uncommitted/ignorance: 0.30
```

However, it is **not recommended as the primary production fusion engine** for this system because:

- belief-mass assignment from physiological features is itself a calibration problem;
- highly conflicting evidence can produce unintuitive normalized results;
- temporal dynamics require an additional model;
- dependency between physiological sources remains difficult;
- probabilistic calibration and clinical interpretation are easier to validate with a probabilistic state model.

Use Dempster–Shafer only as an experimental comparison or explainability layer if a specific dataset shows material benefit.

---

# 21. Probabilistic graphical / state-space models

A hierarchical dynamic factor graph maps well to the physiology:

```text
ECG/PPG observations ───────→ latent cardiac state ───┐
EDA observations ───────────→ latent EDA state ───────┤
RESP observations ──────────→ latent respiratory ─────┤
TEMP/PPG amplitude ─────────→ latent peripheral ──────┤
context ───────────────────────────────────────────────┤
                                                       ↓
                                             target state S_t
                                                       ↑
                                                   S_{t-1}
```

Advantages:

- missing observations can be omitted/marginalized;
- quality maps into observation variance;
- branch structure limits correlated double counting;
- time is explicit;
- posterior uncertainty is available;
- contributions can be traced by branch.

The practical implementation does not need a heavy general-purpose probabilistic-programming stack. A set of branch Kalman filters plus a calibrated target model and a small HMM/Bayesian temporal filter implements most of the desired behavior.

---

# 22. Machine-learning fusion: what to use now and what to defer

## 22.1 Recommended now

Use **regularized logistic regression or gradient boosting at the pathway-state level**, not raw sample-level deep fusion.

Reasons:

- interpretable contributions;
- works with modest datasets;
- supports explicit missingness features/masks;
- easier probability calibration;
- easier LOSO/cross-site validation;
- nonlinear interaction terms can be added deliberately;
- computationally light for continuous deployment.

## 22.2 Candidate learned extension

Reliability-aware attention/gating is a valid future direction. Recent work explicitly uses signal quality or reliability to gate multimodal physiological representations [13,14]. A 2026 CVPR Workshop study using ECG and respiration from 20 subjects reported up to 74.44% accuracy and 72.08% F1 under subject-independent LOSO validation with reliability-aware cross-attention [14].

However, deep multimodal fusion should **not** be the first production design because:

- many wearable datasets are small;
- controlled WESAD-like performance can be near ceiling and does not imply field generalization;
- cross-dataset stress models can lose substantial performance under distribution shift [30];
- deep classifiers may be poorly calibrated unless explicitly calibrated [31].

## 22.3 Hybrid recommendation

```text
hard quality gates
    ↓
quality→variance calibration
    ↓
branch-level state-space fusion
    ↓
interpretable target model
    ↓
temporal Bayesian/HMM filter
    ↓
probability calibration
    ↓
reason-code/explanation layer
```

This is the recommended production architecture.

---

# 23. Rule-based versus learned fusion

| Criterion | Rules only | Learned only | Recommended hybrid |
|---|---|---|---|
| Interpretability | high | low–medium | high |
| Data requirement | low | high | medium |
| Reliability handling | explicit | only if designed | explicit + learnable |
| Missing modalities | easy if coded | often brittle | explicit |
| Nonlinear interaction | limited | strong | selective |
| Calibration | weak unless engineered | possible | explicit |
| AUD data scarcity | favorable | unfavorable | favorable |
| Real-time cost | low | variable | low–medium |
| Validation/debugging | easy | difficult | manageable |

Use rules for **admission control, quality gates, sensor hierarchy, temporal plausibility, and explanation**. Use a calibrated statistical model for **how valid complementary evidence maps to a target state**.

---

# 24. Reliability calibration

Sensor-quality scores need calibration just as final model probabilities do.

`ECG_quality = 0.8` and `EDA_quality = 0.8` should not be assumed equivalent. For each feature family, build reliability curves against an error reference:

```text
quality-bin → empirical feature error distribution
```

Examples:

- ECG quality vs absolute HR error / R-peak disagreement;
- PPG quality vs pulse-rate error against clean ECG when both available;
- derived respiration quality vs direct respiration reference;
- EDA quality vs artifact-labelled feature stability;
- temperature confidence vs contact/ambient-related deviation.

Then estimate:

```text
R_m(t) = Var(error | quality/context bin)
```

or fit a monotonic regression mapping quality features to predictive variance.

Calibration should be checked per device, sensor placement, population, and software version where possible.

---

# 25. Final probability calibration

A classifier score is not automatically a meaningful probability. Calibration should be assessed with:

- reliability diagrams;
- Brier score;
- expected calibration error (ECE);
- calibration slope/intercept;
- subject-wise calibration analysis.

For logistic/boosting models, evaluate Platt scaling or isotonic regression on a held-out calibration set. For neural models, temperature scaling is a strong simple baseline [31]. Recent wearable-stress work emphasizes that cross-subject discrimination may remain acceptable while calibration degrades substantially, and that person-level recalibration can materially improve probability usefulness [32].

**No output should display `confidence = 0.87` unless 0.87 has a validated probabilistic or clearly defined reliability meaning.**

---

# 26. Missing-not-at-random handling

Missingness should become its own metadata stream.

For each modality, maintain:

```text
missing_now
missing_duration
missing_fraction_5m
missing_fraction_1h
last_good_timestamp
known_missing_reason
activity_at_dropout
quality_before_dropout
```

Use missingness in two ways:

1. **Evidence coverage:** less observed information increases final uncertainty.
2. **Observation-process feature:** if missingness is demonstrably associated with context/physiology (e.g., PPG loss during vigorous movement), the missingness pattern may be modeled as a covariate [28,29].

Do not infer physiology *from missingness alone* unless that relationship is validated prospectively.

---

# 27. Contradictory high-quality modalities

Case:

```text
ECG quality high
PPG quality high
cardiac estimates disagree
```

This is more serious than high-vs-low quality disagreement.

Procedure:

1. verify clock alignment and pulse-transit delay tolerance;
2. compare beat-level correspondences rather than only window averages;
3. compute normalized innovation/disagreement;
4. inspect arrhythmia/ectopy or missed/double detections if available;
5. inflate cardiac branch covariance;
6. retain both source estimates in the trace;
7. if unresolved, return `CARDIAC_CONFLICT` rather than selecting the larger value or averaging them.

Cross-sensor disagreement can reveal hidden sensor failure. It can also reveal real physiology (e.g., pulse deficits in certain rhythm conditions), which is another reason not to force agreement.

---

# 28. Personalized fusion

Personalization is strongly advisable at the **baseline and calibration** level before learning fully subject-specific fusion weights.

## Recommended personalization sequence

### Phase 0: population model

Use population priors and modality models with subject-normalized features.

### Phase 1: baseline adaptation

Collect stable periods to estimate:

- resting HR distribution;
- HRV baseline by context/time-of-day;
- tonic EDA baseline and responsiveness;
- temperature baseline/trend;
- respiration baseline;
- modality-specific typical quality.

### Phase 2: response-profile adaptation

As labeled/known events accumulate, estimate whether the person is a strong/weak responder in each branch. Use hierarchical shrinkage rather than unconstrained per-person weights:

```text
β_person = β_population + δ_person
δ_person ~ N(0, τ²)
```

This protects against overfitting small personal datasets.

### Phase 3: probability recalibration

If enough labeled personal data exist, recalibrate the final posterior. Personalized wearable-stress studies show that subject adaptation can reduce label requirements and improve cross-person behavior [32,33].

Do **not** rapidly adapt weights during suspected AUD deterioration without safeguards, because the system could normalize pathological changes into the baseline.

---

# 29. Population-to-person adaptation

Prefer, in order:

1. subject-relative normalization;
2. population model with hierarchical/mixed-effects subject offsets;
3. post-hoc personal calibration;
4. limited fine-tuning only after enough labels;
5. full personalized deep model only if the data volume justifies it.

This is safer than training one independent model per person from a small number of events.

---

# 30. Evidence from multimodal studies: does fusion actually help?

The answer is **sometimes, and usually less than simplistic narratives imply**.

## 30.1 cStress: ECG + respiration

Hovsepian et al. trained cStress on 21 participants and externally tested in a separate lab cohort of 26 and a field cohort of 20 [6]. In training cross-subject validation:

| Feature set | F1 | AUC | Hit rate |
|---|---:|---:|---:|
| ECG + respiration (all) | 0.81 | 0.96 | 0.93 |
| ECG features | 0.78 | 0.95 | 0.92 |
| HRV only | 0.56 | 0.78 | 0.84 |
| respiration | 0.75 | 0.93 | 0.90 |

The full multimodal set improved over ECG alone, but not by an enormous margin. The independent lab test produced median F1 ≈0.90; field self-report performance was around 0.72. This is an important pattern: **complementarity helps, but field generalization remains harder** [6].

## 30.2 WESAD

WESAD contains ECG, EDA, EMG, respiration, temperature, BVP and accelerometry from 15 subjects in a laboratory stress/amusement/baseline protocol [7]. Its benchmark reports binary stress-vs-nonstress accuracies up to roughly 93% depending on device/model. A public reproduction of chest-sensor baselines found, for example, LDA binary accuracy of about 86.5% for ECG, 84.2% for respiration, and 89.9% for the physiological multimodal set [34]. This supports a modest gain from complementary fusion while also showing that a strong single modality can already perform well in a controlled dataset.

## 30.3 ECG + EDA reproducibility evidence

A 2020 reproducibility study summarizing one 26-participant stress dataset reported LOSO F1 improvement from 0.87 with ECG alone to 0.94 with ECG+EDA in the laboratory and from 0.66 to 0.73 in field data [10]. Again, multimodal gain persists but absolute field performance is lower.

## 30.4 Signal-quality-aware AF fusion

Malone et al. fused ECG, PPG, and ABP with signal-quality weighting for AF detection [4]. At −10 dB SNR, accuracy improved by **4.51 percentage points with two inputs and 10.92 points with three inputs** relative to a single input. This is one of the clearest demonstrations that additional sensors are especially valuable **when reliability is explicitly modeled under noise**, rather than simply concatenated.

## 30.5 Quality-aware ECG+PPG respiration

John et al. fused ECG and PPG for respiratory-rate estimation with instantaneous signal quality as weights and reported a median absolute error of 0.34 breaths/min on CapnoBase across the studied noise range [35]. This strongly supports dynamic quality-aware redundant fusion for derived physiology.

### Conclusion from the evidence

The correct claim is not “multimodal is always better.” It is:

> **Multimodal fusion can improve accuracy and robustness when modalities carry complementary or redundant useful information, but gains depend on task, quality, dependency, setting, and validation. Quality-aware fusion is particularly valuable under sensor degradation.**

---

# 31. Laboratory versus ambulatory evidence

| Setting | Main advantage | Main danger |
|---|---|---|
| Laboratory | clean labels, synchronized sensors, controlled stimuli | overestimates real-world performance |
| Clinical/detox | clinically relevant state and supervision | smaller cohorts; treatment alters physiology |
| Ambulatory | realistic motion/context/dropout | noisy labels, missingness, confounding, domain shift |

The cStress lab-to-field decrease [6], context-aware stress work [8,9], and reproducibility studies [10,30] all argue against using laboratory accuracy as the expected continuous ambulatory performance.

For this system, **ambulatory robustness metrics should be first-class validation endpoints**: performance vs signal quality, sensor dropout, activity level, subject, day, and device session.

---

# 32. AUD/SUD evidence relevant specifically to fusion

Fusion methodology can transfer from stress/wearable monitoring, but AUD interpretation must be grounded separately.

## 32.1 Stress/craving in SUD treatment

A 30-participant ambulatory pilot used accelerometry, skin conductance, skin temperature, and heart rate for four days [15]. Top models achieved approximately:

- stress vs non-stress: 74.5% accuracy, AUC 0.82;
- craving vs no-craving: 75.7% accuracy, AUC 0.82;
- craving vs stress: 76.8% accuracy, AUC 0.80.

This supports feasibility of multimodal SUD-state discrimination but does not establish universal sensor weights.

## 32.2 Alcohol cue reactivity

Older and newer alcohol-cue studies show that physiological channels are heterogeneous. HRV may be sensitive in some protocols [17], while a 2026 VR study found significant cue effects for nonspecific SCR frequency and respiration rate but not HR/HRV [22]. Other cue studies report weak or inconsistent HR/skin-conductance/temperature specificity [16].

**Implementation consequence:** cue-reactivity fusion must allow `MODALITY_SPECIFIC_RESPONSE`; it should not require all modalities to agree.

## 32.3 Withdrawal/autonomic dysfunction

Withdrawal and chronic AUD affect autonomic function, but classical measures do not form a simple uniform signature. Studies report altered HRV/baroreflex function and changes over abstinence, while another acute withdrawal study found classical HRV/SSR measures insufficient for the proposed sympathetic activation [18–21].

**Implementation consequence:** withdrawal-related fusion should use a validated AUD-specific target model and longitudinal personal baseline, not generic stress rules.

## 32.4 Intoxication

A 2026 study with 28 healthy participants used HR, EDA, skin temperature and accelerometry and achieved up to ~0.80 accuracy for intoxication/impairment classification [23].

**Implementation consequence:** multimodal intoxication inference is plausible, but the current evidence is feasibility-level and population/device-specific. Keep it downstream from generic physiological fusion and require alcohol exposure/context evidence.

---

# 33. Confidence decomposition

Final confidence should have multiple components.

```text
confidence {
    state_probability: 0.84
    calibrated_probability: true

    evidence_coverage: 0.78
    measurement_reliability: 0.91
    cross_modal_consistency: 0.87
    temporal_consistency: 0.82
    model_domain_confidence: 0.70

    overall_label: HIGH | MODERATE | LOW
}
```

Do not collapse these factors into one unexplained arithmetic product. The `state_probability` comes from the probabilistic target model. The remaining values are explanation/quality diagnostics that help interpret why that posterior should or should not be trusted.

A useful distinction:

- **probability:** what the model believes about the target state;
- **coverage:** how much expected evidence is available;
- **measurement reliability:** whether observed sensors are trustworthy;
- **consistency:** whether reliable branches agree;
- **domain confidence:** whether the current scenario resembles validated operating conditions.

---

# 34. Fusion reason codes

Recommended codes:

```text
MULTIMODAL_AGREEMENT
HIGH_RELIABILITY_CONSENSUS
SINGLE_PATHWAY_SUPPORT
MODALITY_SPECIFIC_RESPONSE
CARDIAC_REDUNDANT_AGREEMENT
CARDIAC_CONFLICT
RESPIRATORY_REDUNDANT_AGREEMENT
REDUNDANT_SENSOR_CONFLICT
LOW_MODALITY_COUNT
PRIMARY_MODALITY_MISSING
SUPPORTING_MODALITY_MISSING
LOW_RELIABILITY_MODALITY_GATED
DEGRADED_MODALITY_DOWNWEIGHTED
TEMPORAL_MISALIGNMENT
POSSIBLE_INFORMATIVE_MISSINGNESS
HIGH_ACTIVITY_CONTEXT
RECENT_ACTIVITY_DECAY
CONTEXT_UNCERTAIN
OUT_OF_DOMAIN_CONTEXT
MODEL_UNCALIBRATED
INSUFFICIENT_EVIDENCE
```

Every reason code should map to a deterministic condition visible in logs/tests.

---

# 35. Output schema

Recommended JSON-like contract:

```text
FusedState {
    target: AUTONOMIC_ACTIVATION
    timestamp: ...
    timescale: FAST | STATE | TREND

    posterior {
        probability: 0.84
        calibrated: true
        class: HIGH
    }

    confidence {
        overall: HIGH
        evidence_coverage: 0.78
        measurement_reliability: 0.91
        cross_modal_consistency: 0.87
        temporal_consistency: 0.82
        domain_confidence: 0.70
    }

    branches {
        cardiac {
            state: ACTIVATED
            posterior: 0.82
            uncertainty: 0.11
            contribution: +0.31
            sources: [ECG, PPG]
        }
        electrodermal {
            state: ACTIVATED
            posterior: 0.91
            uncertainty: 0.08
            contribution: +0.28
            sources: [EDA]
        }
        respiratory {
            state: UNKNOWN
            contribution: 0
            sources: []
        }
        peripheral {
            state: NEUTRAL
            posterior: 0.53
            contribution: -0.04
            sources: [TEMP, PPG_AMPLITUDE]
        }
    }

    supporting_modalities: [ECG, EDA, PPG]
    neutral_modalities: [TEMP]
    conflicting_modalities: []
    missing_modalities: [RESP]
    gated_modalities: []

    conflict_index: 0.08
    reason_codes: [
        MULTIMODAL_AGREEMENT,
        HIGH_RELIABILITY_CONSENSUS,
        SUPPORTING_MODALITY_MISSING
    ]

    provenance {
        model_version: ...
        calibration_version: ...
        feature_versions: ...
        quality_models: ...
    }
}
```

This preserves the evidence chain and avoids the unacceptable output `0.87` with no explanation.

---

# 36. Per-target fusion algorithm

## 36.1 Generic autonomic activation

**Inputs:** cardiac branch, EDA branch, respiration branch, peripheral branch, activity/context, temporal history.

```text
1. Collect branch evidence aligned to the current physiological time.
2. Gate invalid/stale observations.
3. Update redundant cardiac/respiratory branch filters.
4. Convert branch quality to calibrated predictive uncertainty.
5. Compute branch state posteriors.
6. Apply activity/context conditioning.
7. Evaluate target model using observed branch states only.
8. Calculate branch signed contributions and conflict diagnostics.
9. Apply temporal state transition/update.
10. Calibrate probability.
11. Compute coverage/reliability/consistency diagnostics.
12. Emit posterior + contributions + missing/conflicting/gated branches + reason codes.
```

Minimum inference rule: normally require at least **two independent physiological pathways** for a `HIGH_RELIABILITY_CONSENSUS` label. A one-pathway event may still be emitted as `SINGLE_PATHWAY_SUPPORT` if the target permits it, but not as equivalent multimodal confirmation.

## 36.2 Parasympathetic suppression

Primary evidence is cardiac HRV with respiration/context qualification.

```text
1. Require valid ECG RR/NN series or validated PPG-PPI fallback.
2. Use window length appropriate to the HRV metric.
3. Require or model respiration because HRV is respiration-sensitive.
4. Exclude/qualify periods of substantial activity.
5. Compare to personal context-matched baseline.
6. Estimate cardiac vagal state posterior.
7. Smooth at the medium timescale.
8. Do not use EDA as a direct substitute for vagal evidence.
```

## 36.3 Cue/craving-reactivity evidence

```text
1. Require cue/event context and timestamp confidence.
2. Create event-relative windows rather than generic simultaneous windows.
3. Allow EDA lag of several seconds after cue onset.
4. Extract EDA phasic response, respiratory change, cardiac response, peripheral response.
5. Compare each to pre-event personal baseline.
6. Fuse with a cue-reactivity model trained specifically on AUD/SUD data.
7. Permit modality-specific response; do not require HR/HRV change.
8. Emit evidence strength and uncertainty, not 'craving=true'.
```

The 2026 alcohol VR evidence makes this asymmetry important: EDA/respiration can change without significant HR/HRV change [22].

## 36.4 Withdrawal-related physiological evidence

```text
1. Require known AUD/withdrawal monitoring context from upstream clinical layer.
2. Use medium and trend timescales; avoid one-window diagnosis.
3. Compare HR/HRV/respiration/EDA/peripheral measures to personal baseline.
4. Require at least two independent physiological pathways for strong evidence.
5. Model medication/treatment timing as context if available.
6. Preserve conflicting HRV/EDA evidence rather than forcing sympathetic consensus.
7. Fuse only with an AUD-specific model validated against withdrawal severity labels.
8. Output 'physiological evidence consistent/inconsistent with withdrawal-related autonomic change'.
```

## 36.5 Intoxication/recovery evidence

```text
1. Require exposure/context evidence; physiology alone is nonspecific.
2. Use HR/PPG, EDA, temperature and motion/context features.
3. Track trajectory over time, not only a static class.
4. Use a calibrated AUD/alcohol-specific model.
5. If sensor quality drops during movement, reduce those measurements rather than interpreting dropout as intoxication.
6. Output evidence strength + trend + uncertainty.
```

## 36.6 Sleep/recovery deterioration

```text
1. Require validated sleep/rest context.
2. Aggregate stable nocturnal HR/HRV and respiration.
3. Compare to personal sleep baseline and recent nights.
4. Keep night-level estimates separate from fast daytime activation.
5. Fuse across nights with a slow state/trend model.
6. Emit deterioration/improvement evidence and confidence.
```

AUD research shows lower parasympathetic HRV during sleep and partial autonomic recovery with abstinence, supporting the trend-level use of cardiac autonomic evidence [20,21].

---

# 37. Real-time architecture and timing

A practical online scheduler:

```text
RAW ACQUISITION                         high-rate, modality-native
    ↓
UPSTREAM FEATURE ENGINE                 modality-specific
    ↓
EVIDENCE BUS                            timestamped evidence objects
    ↓
BRANCH FILTERS                          1 s tick
    ↓
FAST TARGET FUSION                      1 s tick, 5–30 s evidence
    ↓
MEDIUM TARGET FUSION                    5–10 s tick, 30–300 s evidence
    ↓
LONG TREND AGGREGATOR                   1–5 min tick, hours/days history
```

### Suggested buffers

- beat/event buffer: 30–60 s;
- fast feature buffer: 2–5 min;
- standardized HRV buffer: 5 min;
- medium state history: 30–60 min;
- long trend summaries: days/weeks.

### Compute cost

The recommended design is light:

- branch Kalman/state filters: O(number of active observations) per tick;
- logistic/gradient-boosted target model: negligible on backend/edge CPU;
- HMM/Bayesian temporal update: small state dimension;
- no requirement for GPU inference.

Deep multimodal attention should be an optional server-side experiment, not a dependency of the first reliable system.

---

# 38. Failure modes and mitigations

| Failure mode | Why it matters | Mitigation |
|---|---|---|
| Correlated ECG/PPG counted twice | false certainty | fuse within cardiac branch first |
| Derived respiration counted as independent of ECG/PPG | false certainty | track parent-source covariance/source graph |
| Low-quality channel dominates | catastrophic fusion | quality→variance mapping + hard gates |
| High-quality redundant sensors disagree | hidden failure/real physiological difference | innovation test + covariance inflation + conflict state |
| Complementary pathways disagree | model may force false consensus | preserve signed contributions and `MODALITY_SPECIFIC_RESPONSE` |
| Clock drift | artificial conflict | monotonic synchronized timestamps + drift correction |
| EDA latency ignored | false disagreement | event-relative lag window 1–4 s where appropriate |
| HRV window too short | unstable features | metric-specific minimum windows/uncertainty |
| Temperature treated as fast signal | spurious event associations | slow trend branch |
| Missing sensor causes crash | poor ambulatory reliability | probabilistic omission/marginalization |
| Imputed modality treated as observed | false confidence | mark reconstructed and propagate uncertainty |
| Missingness assumed random | context bias | log and model observation process |
| Static modality weights | quality/context mismatch | dynamic variance and target-specific model |
| Lab-only training | real-world performance collapse | field/cross-dataset validation |
| Probability uncalibrated | misleading confidence | held-out calibration + reliability diagrams |
| Subject non-responder | systematic error | personal baseline/response profile |
| Rapid adaptation normalizes pathology | unsafe personalization | slow bounded adaptation + frozen reference baseline |
| Context wrong | physiological meaning wrong | context confidence + out-of-domain reason code |
| One huge feature magnitude dominates | numerical/model artifact | standardized branch states; regularization |

---

# 39. Validation protocol

Validation should explicitly test the conditions the fusion layer claims to handle.

## 39.1 Split strategy

At minimum:

- leave-one-subject-out cross-validation;
- held-out final subjects not used for feature/threshold selection;
- if multiple datasets/sites are available, cross-dataset or external validation;
- session/day separation to avoid temporal leakage.

## 39.2 Required ablations

For every target:

```text
full multimodal
minus ECG
minus PPG
minus EDA
minus respiration
minus temperature
minus context
cardiac only
electrodermal only
respiratory only
peripheral only
```

Also test branch fusion variants:

```text
static equal weights
quality-weighted
quality + temporal model
full hierarchical model
```

## 39.3 Robustness tests

Inject realistic degradation:

- contiguous modality dropout of varying duration;
- intermittent packet loss;
- delayed modality timestamps;
- quality score corruption;
- motion-related PPG degradation;
- contact-related EDA degradation;
- missing primary sensor;
- conflicting redundant sensors;
- one high-quality contradictory branch;
- multiple low-quality agreeing branches.

Do not test robustness only with additive Gaussian noise; Malone et al. explicitly note this as a limitation of their AF study [4].

## 39.4 Metrics

Report:

- AUROC/AUPRC/F1 where classification applies;
- sensitivity/specificity at declared operating points;
- Brier score and calibration slope/intercept;
- ECE/reliability plots;
- mean/median absolute error for continuous state estimates;
- performance stratified by activity and modality quality;
- failure rate / `INSUFFICIENT_EVIDENCE` rate;
- confidence-coverage curve;
- sensor-dropout degradation curve;
- per-subject variance;
- conflict detection precision where ground truth is available.

---

# 40. Research evidence table

| Paper | Modalities | Target / population | Fusion level & method | Reliability / missing handling | Multimodal result | Validation |
|---|---|---|---|---|---|---|
| Li et al. 2008 [1] | multi-lead ECG + ABP | HR; 437 ICU patients, >6000 h | estimate/state-level; SQI-modified Kalman filters + weighted fusion | SQI modifies update; robust to noise/missing | accurate HR under persistent artifact | large retrospective ICU data + injected noise |
| Nemati et al. 2010 [2] | ECG, respiration, peripheral tonometry-derived RR | respiration rate; 30 overnight PSG subjects + simulation | state-level modified KF | SQI modifies covariance; innovations weight estimates | automated fusion outperformed individual RR methods across studied noise/rates | real PSG + simulation |
| Johnson et al. 2015 [3] | ECG + ABP | heartbeat detection; PhysioNet Challenge data | decision/event fusion selecting higher-SQI detections | explicit per-signal SQI | external Se 95.1%, PPV 89.3% | external Challenge set |
| Malone et al. 2025 [4] | ECG + PPG + ABP | AF detection | wavelet/feature fusion + BiLSTM | quality-weighted fusion | at −10 dB, +4.51 pp with 2 inputs; +10.92 pp with 3 vs single input | MIMIC/experimental noise; no wearable-motion validation |
| John et al. 2025 [35] | ECG + PPG | respiration-rate estimation | feature/signal-derived fusion | instantaneous SQI weights | median absolute error 0.34 bpm on CapnoBase | benchmark + noise scenarios |
| Hovsepian et al. 2015 [6] | ECG + respiration + activity gating | stress; train N=21, lab test N=26, field N=20 | feature/state-level SVM | quality screening + activity exclusion | train F1 all 0.81 vs ECG 0.78, RIP 0.75; lab test ~0.90; field ~0.72 | cross-subject + independent lab + field |
| WESAD 2018 [7,34] | ECG, EDA, EMG, RESP, TEMP, BVP, ACC | stress/affect; N=15 lab | feature-level classical ML | not primarily reliability-aware | binary accuracy up to ~93%; public reproduction physiology multimodal ~89.9% LDA vs ECG ~86.5% | LOSO benchmark |
| Mishra et al. 2020 [10] | multiple studies; ECG/EDA and others | stress; 90 participants analyzed across 4 studies | reproducibility/cross-study ML | examines cross-study robustness | one dataset: ECG+EDA F1 0.94 lab vs ECG 0.87; field 0.73 vs 0.66 | cross-study evaluation |
| Gjoreski et al. 2017 [8] | BVP/HR/EDA/temp/ACC | real-life stress | hierarchical context-aware ML | activity/context qualifies stress | 70% recall, 95% precision in real-life evaluation | 55 days real-life data |
| Cisse & Ishimaru 2026 [12] | IMU, ECG, EDA etc. | stress/activity representation | self-supervised multimodal representation | modality dropout detection + reconstruction | improved robustness under simulated missing modalities | WESAD + MobiAct |
| Kumar & Ganapathy 2026 [14] | ECG + respiration | driver stress; N=20 | reliability-aware cross-attention | dynamic reliability-aware fusion | up to 74.44% accuracy / 72.08% F1 | LOSO subject-independent |
| Leonard et al. 2020 [15] | HR, EDA, temp, ACC | stress/craving in SUD; N=30 | feature-level ML | activity logged; not formal probabilistic reliability fusion | AUC 0.82 stress and craving | 4-day ambulatory pilot |
| VR alcohol cue study 2026 [22] | EDA, HR/HRV, respiration, pupil | craving in alcohol dependence; N=61 | multimodal assessment, mixed-effects analysis | not a fusion algorithm | cue effects on NS-SCR and RR; not HR(V) in protocol | clinical VR cue exposure |
| Intoxication wearable study 2026 [23] | HR, EDA, skin temp, ACC | alcohol impairment; N=28 | XGBoost feature fusion | not explicitly reliability-aware | accuracy up to 0.80 | controlled intoxication lab |

### Evidence interpretation

The strongest **methodological** support for this architecture comes from [1–5,12,14,35]: reliability should modify influence, noisy sources should be down-weighted, and missing modalities should not collapse the system. The strongest **physiological complementarity** support comes from [6–10]. AUD/SUD studies [15–23] establish that multimodal physiological patterns can be useful, but they do not yet justify one universal AUD fusion equation.

---

# 41. Modality role table

| Modality | Physiological information | Redundant with | Complementary with | Reliability input | Typical effective timing |
|---|---|---|---|---|---|
| ECG | cardiac timing, HR, HRV | PPG timing; ECG-derived resp shares source | EDA, resp, peripheral | R-peak/ECG quality, feature validity, motion/confound status | HR seconds; HRV tens sec–5 min+ |
| PPG | pulse timing, pulse amplitude, perfusion | ECG HR/PPI; PPG-derived resp | EDA, temp, resp | waveform SQI, motion/contact, perfusion quality | pulse seconds; perfusion tens sec |
| EDA | tonic/phasic sudomotor sympathetic activity | none in current core set | cardiac, resp, temp/perfusion | electrode/contact, artifact, decomposition quality | phasic onset commonly 1–4 s after stimulus; tonic slower |
| Temperature | peripheral thermal/vasomotor trend | partly with peripheral perfusion only | cardiac, EDA | contact, ambient/context, slope stability | tens sec–minutes |
| Respiration | rate/pattern/variability | derived respiration estimates | cardiac, EDA | belt/signal quality or derived-source quality | multiple breaths, ~20–60 s+ |
| IMU/context | movement/activity/posture/context | n/a | conditions all physiology | classifier/context confidence | near-immediate; activity decay can persist |

---

# 42. Fusion-method comparison

| Method | Missing data | Reliability-aware | Conflict-aware | Temporal | Interpretability | Recommendation |
|---|---|---|---|---|---|---|
| Equal average | weak | no | no | no | high | no |
| Static weighted score | moderate | weak | weak | no | high | no as final model |
| Quality-normalized weighted average | good for redundant scalar estimates | yes | limited | no | high | only within redundant branch |
| Kalman/state-space | excellent | yes via covariance | innovation exposes conflict | yes | high | **yes for branch/time fusion** |
| Naive Bayes | naturally handles missing | possible | limited | extension needed | high | no unless independence validated |
| Bayesian network/factor graph | excellent | yes | yes | DBN extension | high | **conceptually strong** |
| Dempster–Shafer | good | possible | explicit conflict/ignorance | no by default | medium | experimental only |
| Logistic/GLM branch fusion | good with masks/subset design | yes via uncertain branch inputs | contribution-based | external temporal filter | high | **yes** |
| Gradient boosting | good if designed | yes as features | moderate | external temporal | medium | optional comparator |
| Deep early fusion | poor by default | only if designed | opaque | possible | low | defer |
| Reliability-aware attention/MoE | good if trained for dropout | yes | learned | possible | low–medium | research extension |

---

# 43. Recommended implementation

## Fusion level

Implement **hierarchical multi-stage fusion**:

1. **Feature/state inputs only** for heterogeneous modalities; no global raw-signal fusion.
2. **Redundant measurement fusion** within cardiac and respiratory branches using quality-calibrated state-space estimation.
3. **Pathway-level complementary fusion** using a calibrated target-specific probabilistic model.
4. **Temporal state fusion** using an HMM/state transition or state-space prior.
5. **Optional AUD-evidence fusion** only after target-specific AUD models are validated.

## Modality-specific inputs

- ECG: HR, RR/NN series, RMSSD/SDNN and frequency HRV when window-valid, cardiac branch evidence, predictive variance.
- PPG: pulse rate/PPI, pulse amplitude/perfusion, optional derived respiration, predictive variance.
- EDA: tonic SCL, phasic activity/SCR frequency/amplitude, electrodermal state, predictive variance.
- Temperature: baseline-relative temperature and slope, peripheral state, predictive variance.
- Respiration: rate, variability/pattern and respiratory state, predictive variance.
- IMU/context: activity, motion/confound state, posture/rest/sleep/cue context and confidence; mostly gating/conditioning.

## Reliability representation

Do **not** multiply raw upstream confidence scores. Learn `quality/context → expected error variance` per modality-feature. Pass predictive variance into branch filters and target uncertainty propagation.

Cold start: hard invalidity gates + conservative categorical reliability mapped to broad variance.

## Dynamic modality weighting

For redundant observations, dynamic weight emerges from measurement covariance in the Kalman/state-space update. For complementary modalities, do not assign a universal normalized weight; use a target-specific calibrated model over branch states and uncertainties.

## Temporal alignment

- retain timestamps and feature windows;
- model EDA event lag explicitly (~1–4 s range for stimulus-elicited responses [27]);
- allow HR to update rapidly but HRV only when its statistical window is valid;
- treat temperature as slow trend;
- use event-relative alignment for cue/reactivity targets;
- emit `TEMPORAL_MISALIGNMENT` when evidence cannot be physiologically aligned.

## Redundancy handling

- cardiac timing: ECG primary, PPG validation/fallback;
- compare high-quality redundant estimates using normalized innovation;
- fuse only after consistency check;
- derived respiration retains dependence on ECG/PPG parent source;
- never count derived and parent features as independent evidence without modeling covariance.

## Conflict handling

- low-quality conflict: down-weight/gate low-quality channel;
- high-quality redundant conflict: inflate branch uncertainty and emit conflict state;
- high-quality complementary conflict: preserve modality-specific response and lower state certainty; do not force consensus;
- report signed contribution per branch.

## Missing modality handling

- skip the absent measurement update;
- infer from observed branches only;
- reduce evidence coverage and increase uncertainty;
- log reason/duration of missingness;
- treat reconstruction as auxiliary and uncertainty-bearing, never as equivalent to observation;
- include missingness pattern as a covariate only if validated.

## Temporal fusion

Maintain three outputs:

```text
FAST_EVENT_STATE     ~ seconds
MEDIUM_PHYS_STATE    ~ 30 s–5 min
LONG_TERM_TREND      ~ hours–days/weeks
```

Use a Markov/state-space prior within each target so that the posterior at `t-1` informs `t`, with transition dynamics matched to the target.

## Probability/confidence calibration

- calibrate final model on held-out subjects;
- evaluate Platt/isotonic/temperature scaling depending on model class;
- report Brier, ECE, slope/intercept and reliability plots;
- consider limited subject-level recalibration after enough labeled personal data;
- never expose a numerical confidence without calibration metadata.

## Output schema

Return:

```text
TARGET_STATE
posterior probability
confidence label
measurement reliability
coverage
cross-modal consistency
branch posteriors
branch signed contributions
supporting / neutral / conflicting branches
missing / gated modalities
reason codes
model/calibration versions
```

## Explainability

Every fused decision must answer:

1. Which physiological pathways supported it?
2. Which opposed it?
3. Which were missing?
4. Which were observed but down-weighted, and why?
5. Was agreement truly complementary or merely redundant?
6. Were modalities temporally aligned?
7. Is the probability calibrated for this population/context?

---

# 44. Concrete reference implementation pseudocode

```python
class FusionEngine:
    def update(self, evidence_batch, t):
        aligned = temporal_aligner.align(evidence_batch, t)

        # 1. Admission / quality gating
        valid = {}
        for e in aligned:
            if hard_invalid(e):
                trace.gate(e, reason="LOW_RELIABILITY_MODALITY_GATED")
                continue
            e.predictive_variance = reliability_models[e.key].variance(e)
            valid[e.key] = e

        # 2. Redundant branch fusion
        cardiac = cardiac_filter.update(
            ecg=valid.get("ECG_CARDIAC"),
            ppg=valid.get("PPG_CARDIAC")
        )

        respiratory = respiratory_filter.update(
            direct=valid.get("RESP_DIRECT"),
            ecg_derived=valid.get("RESP_FROM_ECG"),
            ppg_derived=valid.get("RESP_FROM_PPG")
        )

        electrodermal = eda_state_model.update(valid.get("EDA"))
        peripheral = peripheral_state_model.update(
            temp=valid.get("TEMP"),
            ppg_amp=valid.get("PPG_PERIPHERAL")
        )

        branches = {
            "cardiac": cardiac,
            "electrodermal": electrodermal,
            "respiratory": respiratory,
            "peripheral": peripheral,
        }

        # 3. Target-specific complementary fusion
        outputs = {}
        for target in configured_targets:
            obs = select_relevant_branches(target, branches)
            ctx = context_state(target, valid)

            raw_p, contributions = target_models[target].predict(
                obs=obs,
                context=ctx,
                uncertainty=True
            )

            # 4. Temporal state update
            temporal_p = temporal_models[target].update(raw_p, t, ctx)

            # 5. Calibration
            p = calibrators[target].transform(temporal_p)

            # 6. Explanation
            diag = diagnostics.compute(
                target=target,
                branches=obs,
                contributions=contributions,
                context=ctx,
                probability=p
            )

            outputs[target] = build_fused_state(
                target=target,
                probability=p,
                branches=obs,
                diagnostics=diag,
                provenance=current_versions()
            )

        return outputs
```

The implementation should keep branch filters and target models independently testable so a bad update in one modality cannot silently alter unrelated pathways.

---

# 45. Example behavior

## Example A — strong multimodal autonomic activation

```text
EDA ↑                  high reliability
HR ↑                   high reliability
RMSSD ↓                valid medium-window estimate
respiration ↑          moderate-high reliability
motion low             high confidence
skin temperature       stable
```

Expected output:

```text
AUTONOMIC_ACTIVATION = HIGH
state_probability = calibrated 0.xx
confidence = HIGH

support:
    electrodermal = strong
    cardiac = strong
    respiratory = moderate
neutral:
    peripheral = neutral
conflict:
    none
reason_codes:
    HIGH_RELIABILITY_CONSENSUS
    MULTIMODAL_AGREEMENT
    LOW_ACTIVITY_CONTEXT
```

Do not preassign `0.87`; the numeric value must come from calibration data.

## Example B — isolated EDA response

```text
EDA ↑                  high reliability
HR unchanged           high reliability
HRV unchanged          high reliability
resp unchanged         high reliability
```

Expected:

```text
AUTONOMIC_ACTIVATION = AMBIGUOUS/MODERATE
ELECTRODERMAL_ACTIVATION = HIGH
confidence = MODERATE
reason_codes:
    MODALITY_SPECIFIC_RESPONSE
    SINGLE_PATHWAY_SUPPORT
```

The system does not erase the EDA response, but it does not call it high-confidence whole-body autonomic activation.

## Example C — ECG degraded, PPG missing

```text
EDA ↑                  good
ECG                    low quality
PPG                    unavailable
```

Expected:

```text
cardiac branch = UNKNOWN / low confidence
electrodermal branch = activated
fused autonomic state = limited evidence
reason_codes:
    LOW_RELIABILITY_MODALITY_GATED
    PRIMARY_MODALITY_MISSING (for reliable cardiac timing)
    LOW_MODALITY_COUNT
```

## Example D — activity-related mixed physiology

```text
HR ↑
EDA ↓/neutral
skin temperature changing
recent physical activity high
```

Expected:

```text
physiological activation = activity-qualified
psychological/AUD interpretation = withheld or low confidence
reason_codes:
    HIGH_ACTIVITY_CONTEXT
    RECENT_ACTIVITY_DECAY
    MODALITY_SPECIFIC_RESPONSE
```

---

# 46. Implementation phases

## Phase 1 — deterministic/probabilistic core

Build first:

- evidence object schema;
- time alignment service;
- reliability calibration interface;
- ECG↔PPG redundant cardiac filter;
- direct/derived respiration filter;
- EDA and peripheral branch state objects;
- target-specific logistic models;
- temporal HMM/state filter;
- calibration layer;
- reason-code/explanation output.

This phase is implementable with modest data and is auditable.

## Phase 2 — dataset validation and personalization

Add:

- LOSO + external validation harness;
- realistic sensor-dropout/noise simulator;
- per-subject baseline and hierarchical adaptation;
- probability recalibration;
- missingness modeling.

## Phase 3 — learned reliability-aware representation fusion

Only after adequate multimodal labeled data:

- modality dropout training;
- uncertainty-aware attention;
- mixture-of-experts gating;
- learned cross-modal interaction;
- compare against the Phase-1 interpretable baseline.

Retain Phase-1 output semantics and explanation even if the internal model becomes more learned.

---

# 47. Final synthesis

The central design mistake to avoid is treating each sensor as an independent vote. The system contains both **redundant** and **complementary** physiology:

```text
ECG HR ─┐
        ├─→ cardiac branch ─────────────────────────────┐
PPG HR ─┘                                               │
                                                        │
EDA ─────────→ electrodermal branch ────────────────────┤
                                                        │
RESP / derived RR ─→ respiratory branch ────────────────┤
                                                        │
TEMP + PPG amplitude → peripheral branch ───────────────┤
                                                        ↓
                                             TARGET-SPECIFIC MODEL
                                                        ↓
                                              TEMPORAL STATE MODEL
                                                        ↓
                                    calibrated posterior + explanation
```

The architecture should obey six invariants:

1. **No evidence without reliability.** Every observation carries calibrated uncertainty.
2. **No double counting.** Redundant signals are fused before they vote on higher-level states.
3. **No forced consensus.** High-quality disagreement is preserved and reported.
4. **No failure on missing input.** Missing modalities reduce coverage; they do not collapse inference.
5. **No single timescale.** Event, state, and trend layers are separate.
6. **No unexplained confidence.** Every posterior is calibrated and accompanied by evidence provenance.

This makes the fusion layer suitable as a bridge between trustworthy physiological sensing and AUD-specific interpretation while staying inside the correct boundary: it produces **physiological/AUD-relevant evidence**, not an autonomous clinical treatment decision.

---

# References

[1] Q. Li, R. G. Mark, G. D. Clifford. “Robust heart rate estimation from multiple asynchronous noisy sources using signal quality indices and a Kalman filter.” *Physiological Measurement*, 29(1):15–32, 2008. DOI: 10.1088/0967-3334/29/1/002. PMID: 18175857. PMCID: PMC2259026.

[2] S. Nemati, A. Malhotra, G. D. Clifford. “Data Fusion for Improved Respiration Rate Estimation.” *EURASIP Journal on Advances in Signal Processing*, 2010:926305, 2010. DOI: 10.1155/2010/926305. PMID: 20806056. PMCID: PMC2929127.

[3] A. E. W. Johnson, J. Behar, F. Andreotti, G. D. Clifford, J. Oster. “Multimodal heart beat detection using signal quality indices.” *Physiological Measurement*, 36(8):1665–1677, 2015. DOI: 10.1088/0967-3334/36/8/1665. PMID: 26218060.

[4] S. Malone, B. Cardiff, D. John, A. John. “Signal-quality-aware multisensor fusion for atrial fibrillation detection.” *Healthcare Technology Letters*, 12(1):e12121, 2025. DOI: 10.1049/htl2.12121. PMCID: PMC11856055.

[5] A. John, A. P. James, B. Cardiff, D. John. “A review on multi-sensor data fusion for wearable health monitoring.” *Information Fusion*, 133:104319, 2026. DOI: 10.1016/j.inffus.2026.104319.

[6] K. Hovsepian, M. al’Absi, E. Ertin, T. Kamarck, M. Nakajima, S. Kumar. “cStress: Towards a Gold Standard for Continuous Stress Assessment in the Mobile Environment.” *Proceedings of ACM UbiComp*, 2015:493–504. DOI: 10.1145/2750858.2807526. PMID: 26543926. PMCID: PMC4631393.

[7] P. Schmidt, A. Reiss, R. Duerichen, C. Marberger, K. Van Laerhoven. “Introducing WESAD, a Multimodal Dataset for Wearable Stress and Affect Detection.” *Proceedings of the 20th ACM International Conference on Multimodal Interaction*, 2018:400–408. DOI: 10.1145/3242969.3242985.

[8] M. Gjoreski, M. Luštrek, M. Gams, H. Gjoreski. “Monitoring stress with a wrist device using context.” *Journal of Biomedical Informatics*, 73:159–170, 2017. DOI: 10.1016/j.jbi.2017.08.006. PMID: 28803947.

[9] F.-T. Sun, C. Kuo, H.-T. Cheng, S. Buthpitiya, P. Collins, M. L. Griss. “Activity-Aware Mental Stress Detection Using Physiological Sensors.” In *Mobile Computing, Applications, and Services: Second International ICST Conference, MobiCASE 2010, Revised Selected Papers*, Springer, 2012, pp. 211–230. DOI: 10.1007/978-3-642-29336-8_12. Study: ECG, GSR and accelerometry in 20 participants; reported 92.4% 10-fold CV accuracy and 80.9% between-subject accuracy.

[10] V. Mishra, S. Sen, G. Chen, T. Hao, J. Rogers, C.-H. Chen, D. Kotz. “Evaluating the Reproducibility of Physiological Stress Detection Models.” *Proceedings of the ACM on Interactive, Mobile, Wearable and Ubiquitous Technologies*, 4(4):147, 2020. DOI: 10.1145/3432220. PMID: 36189150. PMCID: PMC9523764.

[11] M. Gjoreski, H. Gjoreski, M. Luštrek, M. Gams. “Continuous stress detection using a wrist device: in laboratory and real life.” *UbiComp/ISWC Adjunct*, 2016:1185–1193. DOI: 10.1145/2968219.2968306.

[12] A. H. Cisse, S. Ishimaru. “Detect and Repair: Robust Self-Supervised Wearable Sensing Under Missing Modalities.” *Sensors*, 26(8):2419, 2026. DOI: 10.3390/s26082419. PMID: 42076526.

[13] S. Choi, M. Ali, C. Pack, E. Lee. “Trustworthy Multimodal Learning via Uncertainty-aware Feature Modulation and Reliability Auditing.” SSRN preprint, posted 21 Aug 2026; listed by the authors as under review at *Knowledge-Based Systems*. DOI: 10.2139/ssrn.7327625. **Non-peer-reviewed; used only as emerging methodological evidence, not as clinical validation.**

[14] K. P. Kumar, N. Ganapathy. “Trust What You Fuse: Reliability-Aware Cross-Attention for Multimodal Physiological Stress Assessment in the Wild.” *Proceedings of the IEEE/CVF CVPR Workshops*, 2026:5341–5350.

[15] N. R. Leonard et al. “Wearable sensor-based detection of stress and craving in patients during treatment for substance use disorder: A mixed methods pilot study.” *Drug and Alcohol Dependence*, 2020. PMID: 32193048. PMCID: PMC7197459.

[16] M. S. Reid, F. Flammino, A. Starosta, J. Palamar, J. Franck. “Physiological and subjective responding to alcohol cue exposure in alcoholics and control subjects: evidence for appetitive responding.” *Journal of Neural Transmission*, 113(10):1519–1535, 2006. DOI: 10.1007/s00702-005-0439-5. PMID: 16604310.

[17] I. Rajan, P. J. Murthy, A. G. Ramakrishnan, B. N. Gangadhar, N. Janakiramaiah. “Heart rate variability as an index of cue reactivity in alcoholics.” *Biological Psychiatry*, 43(7):544–546, 1998. DOI: 10.1016/S0006-3223(97)00399-5. PMID: 9547935.

[18] K.-J. Bär et al. “Heart rate variability and sympathetic skin response in male patients suffering from acute alcohol withdrawal syndrome.” *Alcoholism: Clinical and Experimental Research*, 30(9):1592–1598, 2006. DOI: 10.1111/j.1530-0277.2006.00191.x. PMID: 16930222.

[19] K.-J. Bär, M. K. Boettger, S. Boettger, M. Grotelüschen, R. Neubauer, T. Jochum, V. Baier, H. Sauer, A. Voss. “Reduced baroreflex sensitivity in acute alcohol withdrawal syndrome and in abstained alcoholics.” *Drug and Alcohol Dependence*, 85(1):66–74, 2006. DOI: 10.1016/j.drugalcdep.2006.03.014. PMID: 16650658.

[20] M. de Zambotti et al. “Cardiac autonomic function during sleep: effects of alcohol dependence and evidence of partial recovery with abstinence.” *Alcohol*, 2015. DOI: 10.1016/j.alcohol.2014.07.023. PMID: 25957854. PMCID: PMC4465836.

[21] S. E. Wemm, M. Golden, J. Martins, N. Fogelman, R. Sinha. “Patients with AUD exhibit dampened heart rate variability during sleep as compared to social drinkers.” *Alcohol and Alcoholism*, 58(6):653–661, 2023. DOI: 10.1093/alcalc/agad061. PMID: 37756494. PMCID: PMC10642609.

[22] “Multimodal Psychophysiological Assessment of Craving in Patients With Alcohol Dependence During Virtual Reality Cue Exposure: Exploratory Single-Arm Clinical Study.” *JMIR Serious Games*, 2026. PMID: 42269089. Study population: 61 patients with alcohol dependence.

[23] “Detecting Ethanol Intoxication and Impairment Using Wearable Biosensors.” 2026. PMID: 41550855. PMCID: PMC12805382. Study population: 28 participants.

[24] Task Force / ACC-AHA guidance on HRV measurement: standard short-term HRV recordings commonly use 5 minutes; frequency-domain duration must be adequate for the lowest frequency component. See ACC/AHA ambulatory electrocardiography guidance and the 1996 ESC/NASPE HRV standards.

[25] “Heart Rate Variability and Cardiac Vagal Tone in Psychophysiological Research – Recommendations for Experiment Planning, Data Analysis, and Data Reporting.” 2017. PMCID: PMC5316555.

[26] “Validity of (Ultra-)Short Recordings for Heart Rate Variability Measurements.” 2015. PMID: 26414314.

[27] W. Boucsein et al. “Publication recommendations for electrodermal measurements.” *Psychophysiology*, 49(8), 2012. DOI: 10.1111/j.1469-8986.2012.01384.x. Event-related EDA onset windows commonly around 1–4 s after stimulus change.

[28] “Informative presence and observation in routine health data: A review of methodology for clinical risk prediction.” *Journal of the American Medical Informatics Association*, 28(1):155–166, 2021. PMCID: PMC7810439.

[29] “Wearable Sensors as a Preoperative Assessment Tool: A Review.” 2024. PMCID: PMC10820534. Includes discussion of MCAR/MAR/MNAR missingness in wearable data.

[30] M. Calza-Metre, L. Borzì. “Machine learning-based automatic stress detection: Performance and generalization across datasets.” *Smart Health*, 41:100693, 2026. DOI: 10.1016/j.smhl.2026.100693. Evaluates four public wearable datasets; cross-dataset testing shows substantial degradation, especially for deep models, while transfer learning improves generalization.

[31] C. Guo, G. Pleiss, Y. Sun, K. Q. Weinberger. “On Calibration of Modern Neural Networks.” *Proceedings of the 34th International Conference on Machine Learning*, PMLR 70:1321–1330, 2017.

[32] A. Akkaya. “Calibration, not architecture, limits cross-subject wearable stress detection: a multi-dataset, decision-focused evaluation with label-efficient recalibration.” *BMC Medical Informatics and Decision Making*, 2026. DOI: 10.1186/s12911-026-03842-1.

[33] “Individualized Stress Mobile Sensing Using Self-Supervised Pre-Training.” 2023/2024. PMID: 39507765. PMCID: PMC11540419.

[34] WESAD public baseline experiments/reproduction accompanying the WESAD dataset. LOSO classical-model results reported per modality and for all/physiological modalities; used here as implementation evidence alongside the peer-reviewed WESAD paper [7].

[35] A. John, H. Wang, B. Cardiff, K. K. Parhi, D. John. “Multimodal fusion for robust respiratory rate estimation in wearable sensing.” *Information Fusion*, 123:103253, 2025. DOI: 10.1016/j.inffus.2025.103253.

---

## Final build recommendation in one sentence

> **Build a hierarchical fusion engine that first resolves redundant measurements with quality-calibrated state-space filters, then fuses independent physiological pathways with target-specific calibrated probabilistic models, then applies temporal state tracking—while explicitly preserving missingness, conflict, uncertainty, contribution provenance, and AUD-specific evidence boundaries.**


# Recommended Implementation

The system should implement **Hierarchical Reliability-Aware Dynamic Evidence Fusion (HRA-DEF)** as the production architecture. The preceding sections justify each component; this section is the normative build specification.

## Fusion level

Use **hierarchical multi-stage fusion**, not a single early-fusion block:

1. **Within-pathway redundant fusion:** reconcile measurements of substantially the same physiological variable (for example ECG HR versus PPG pulse rate, or direct versus derived respiration) using quality-calibrated state-space estimation.
2. **Pathway/state construction:** produce cardiovascular, electrodermal, respiratory, and peripheral/thermal state estimates with uncertainty.
3. **Target-specific complementary fusion:** combine those pathway states with a calibrated probabilistic model specific to the target (autonomic activation, cue-reactivity evidence, recovery state, etc.).
4. **Temporal state tracking:** apply target-specific event/state/trend dynamics rather than forcing all modalities into the same instantaneous window.
5. **AUD evidence fusion only after validated AUD mapping:** generic autonomic evidence must not automatically be renamed withdrawal, craving, intoxication, or relapse risk.

## Modality-specific inputs

- **ECG:** HR, RR intervals, RMSSD/SDNN and other adequately-windowed HRV features, cardiac activation evidence, SQI, feature uncertainty, confound/context validity.
- **PPG:** pulse rate/IBI, pulse variability when sufficiently valid, pulse amplitude/perfusion features, waveform quality, feature uncertainty, confound/context validity.
- **EDA:** tonic/phasic state, SCR count/amplitude/rate or normalized electrodermal activation evidence, EDA quality, decomposition confidence, confound/context validity.
- **Temperature:** personal-baseline deviation, slope/trend, peripheral thermal state, quality, environmental/context validity.
- **Respiration:** rate, variability/pattern features, respiratory activation state, signal/derivation quality and uncertainty.
- **IMU/context:** conditioning variables and confound/context confidence. IMU should generally **qualify physiology rather than vote as another autonomic organ system**.

Every modality-state observation must be represented as at least `(value/evidence, calibrated uncertainty, timestamp/effective time, availability, provenance)`.

## Target-specific modality map

Use the full target-modality table in this document as configuration. The key rule is that weights are **target dependent**: ECG/PPG dominate cardiac timing; EDA is primary for electrodermal sympathetic activation; respiration is primary for respiratory activation; temperature/perfusion are slower supporting peripheral signals; IMU/context gates interpretation. For AUD-specific targets, only enable mappings that have direct AUD/SUD evidence and keep generic autonomic evidence distinct from diagnosis.

## Reliability representation

Do **not** multiply arbitrary upstream confidence numbers. Convert modality quality into a calibrated predictive uncertainty:

```text
observation: z_m(t)
predictive variance: R_m(t)
availability: a_m(t)
quality provenance: {SQI, decomposition, confound, context}
```

The fusion engine should operate on `R_m(t)` (or an equivalent calibrated probability of observation error), while retaining the upstream components for explanation. Reliability scores from different modalities must be calibrated separately before they are compared.

## Dynamic modality weighting

For redundant observations, use **precision/state-space weighting**: low predictive variance gives more influence; high variance gives less; unavailable/gated observations do not update the state. This follows the quality-index-driven Kalman/state-space pattern demonstrated for asynchronous cardiac and respiration fusion [1,2,35].

For complementary modalities, use target-specific model coefficients/interactions plus reliability-derived attenuation—not normalized equal voting. Start with an interpretable calibrated logistic/GLM or similarly auditable probabilistic model; add learned gates only if they beat this baseline under subject-independent, corruption/dropout and cross-dataset testing.

## Temporal alignment

Maintain timestamps at both acquisition time and **physiological effective time**. Resample outputs onto a common inference clock only after feature generation. Use modality-specific windows and permissible lags; event-related EDA should be allowed its seconds-scale latency rather than being required to peak with cardiac changes in the same instant [27]. Do not compute slow HRV features at the cadence of beat-to-beat HR.

## Redundancy handling

Fuse redundant evidence **inside the branch before higher-level fusion**. ECG HR and PPG pulse rate must not count as two independent votes for autonomic activation. Compute cross-modal disagreement first; when they agree within uncertainty, produce one cardiac estimate whose uncertainty may shrink. When they disagree, inspect quality and preserve a conflict flag rather than averaging blindly [1,3].

## Conflict handling

Implement two explicit cases:

- **Reliability-resolvable conflict:** one modality is clearly less trustworthy → reduce/gate that modality and continue.
- **High-quality physiological conflict:** both/all modalities are trustworthy but disagree → do not force consensus. Reduce fused confidence and emit `CONFLICTING_EVIDENCE` or a pathway-specific state such as `MODALITY_SPECIFIC_RESPONSE`.

Neutral evidence must be distinguished from contradictory evidence and from missing evidence.

## Missing modality handling

Represent missingness explicitly. A missing observation contributes no likelihood/update; it is not a physiological zero. Continue if the target's minimum evidence requirements are satisfied, lower coverage/confidence, and emit reason codes such as `PRIMARY_MODALITY_MISSING` or `LOW_MODALITY_COUNT`. Record missingness indicators because wearable loss can be informative/MNAR (for example motion-associated PPG loss) [28,29]. Learned reconstruction is optional and must never be presented as if it were measured evidence [12].

## Temporal fusion

Maintain three target-dependent horizons:

- **Fast/event:** seconds to short minutes for cue/event responses and phasic activation.
- **State:** minutes to hours for sustained autonomic state, sleep/rest physiology, withdrawal monitoring.
- **Trend:** hours to days for recovery/deterioration relative to personal baseline.

Use a state-space/HMM-style persistence model at state level; aggregate calibrated state posteriors into trend features rather than repeatedly re-fusing raw signals.

## Probability/confidence calibration

Separate:

```text
state_probability       = model estimate of target state
evidence_quality        = how trustworthy/complete the available observations are
conflict_score          = degree of high-quality cross-modal contradiction
coverage                = expected target evidence currently present
```

Calibrate model probabilities on held-out subjects (temperature scaling, Platt scaling, isotonic regression, or another empirically selected method) and report reliability diagrams/Brier score/ECE where appropriate [31,32]. Do not label an arbitrary weighted score `0.87 confidence`.

## Output schema

```yaml
fused_state:
  target: AUTONOMIC_ACTIVATION
  probability: 0.84          # only after calibration
  state: HIGH

evidence_quality: 0.88
coverage: 0.80
conflict_score: 0.08

supporting:
  cardiovascular:
    state: ACTIVATED
    contribution: strong
    reliability: high
    sources: [ECG, PPG]
  electrodermal:
    state: ACTIVATED
    contribution: strong
    reliability: high
    sources: [EDA]

neutral:
  - peripheral_temperature

conflicting: []
missing:
  - respiration

gated:
  - modality: PPG
    reason: null             # omitted here because PPG was usable

reason_codes:
  - MULTIMODAL_AGREEMENT
  - LOW_MOTION
  - RESPIRATION_MISSING

time:
  inference_time: ...
  evidence_window: ...
  effective_lag_model: ...
```

`contribution` should preferably be emitted in branch space rather than for dozens of correlated raw features.

## Explainability

Every output must expose:

- which physiological pathways supported the target;
- which were neutral;
- which conflicted;
- which were missing or gated and why;
- the reliability/uncertainty used for each branch;
- whether agreement came from independent/complementary evidence or merely redundant sensors;
- temporal alignment/lag decisions where material;
- whether the target is generic physiological evidence or an AUD-specific interpretation.

The final invariant is therefore:

> **Never fuse a modality without its reliability, never count redundant physiology twice, never turn high-quality disagreement into fake consensus, never treat missingness as normal physiology, and never promote generic autonomic activation into an AUD-specific clinical claim without a validated mapping.**
