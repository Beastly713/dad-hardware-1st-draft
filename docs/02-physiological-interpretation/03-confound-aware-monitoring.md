# Confound-Aware Physiological Monitoring Solution

**Document purpose:** engineering specification for the **Confound-Aware Monitoring** layer in the physiological-monitoring pipeline.

**Processing position:**

```text
body
  ↓
sensing
  ↓
acquisition
  ↓
signal trustworthiness
  ↓
physiological signal decomposition
  ↓
CONFOUND-AWARE MONITORING   ← this document only
  ↓
context-aware interpretation
  ↓
AUD-specific physiological meaning/action
```

**Core question**

> Given a physiological change that is already technically trustworthy and physiologically decomposed, is the change reasonably attributable to unexplained/autonomic physiology, or is it substantially explained by physical activity, recent exertion, posture, respiration, thermoregulation, or another measured confound?

This document deliberately does **not** address raw signal quality, generic motion-artifact rejection, tonic/phasic decomposition except as an upstream input, behavioral/contextual meaning, or AUD-specific interpretation.

---

## 1. System contract and design principles

### 1.1 Input modalities assumed

The hardware stack considered here contains:

- **ECG** (AD8232) → R-R intervals, heart rate (HR), ECG-derived HRV.
- **PPG** (MAX30101) → pulse rate, pulse-to-pulse intervals, pulse-rate variability (PRV), pulse amplitude/morphology, peripheral perfusion-related features. Any SpO₂ use is outside the core recommendation unless a separately validated calibration pipeline exists.
- **EDA/GSR** (tinyGSR) → already decomposed tonic/phasic electrodermal variables.
- **Skin temperature** (TMP117) → local skin temperature.
- **IMU** (MPU-6050) → tri-axial acceleration and gyroscope data used primarily as a confound-observation stream.
- **Derived respiration**, where sufficiently reliable, estimated from ECG and/or PPG; no direct respiratory belt is assumed.

### 1.2 Fundamental distinction: artifact vs confound

This subsystem must preserve a strict separation between three cases.

#### A. Measurement artifact

The sensor measurement itself is unreliable.

Examples:

```text
PPG waveform corrupted by sensor motion
EDA electrode loses contact
ECG R peaks are unreliable
TMP117 attachment changes abruptly and measurement is biased
```

These are primarily upstream **signal-trustworthiness** problems. They may enter this layer only as quality metadata, not as physiological confound estimates.

#### B. Genuine physiology caused by a confound

The measurement is accurate, but the physiological change is caused by something other than the latent process this downstream pipeline hopes to isolate.

Examples:

```text
HR rises because the subject climbed stairs
EDA rises because exercise and heat produce thermoregulatory sweating
PPG amplitude falls because cold-induced peripheral vasoconstriction reduces superficial blood volume
HRV changes because breathing pattern changes
```

This is the primary target of this layer.

#### C. Genuine mixed physiology

Target physiology and confound physiology coexist.

```text
observed HR change
    = autonomic response of interest
    + exercise response
    + orthostatic response
    + thermal response
```

This case is often **not identifiable** from wearable measurements alone. The system must therefore be allowed to emit **AMBIGUOUS** rather than forcing a corrected value.

### 1.3 Conservative identifiability rule

A confound contribution may be mathematically removed only when all of the following hold:

1. the confound is measured independently enough to be useful;
2. the feature–confound relationship has been validated or individually calibrated;
3. the current state lies inside the model's calibration domain;
4. correction is not known to erase the same physiological pathway the downstream system is trying to preserve;
5. prediction uncertainty is low enough to support correction.

Otherwise the preferred action is **NORMALIZE, DOWN-WEIGHT, ANNOTATE, GATE, or MARK AMBIGUOUS**, rather than aggressive subtraction.

This point is particularly important for HRV and EDA, where exercise, posture, respiration, thermoregulation, and psychologically driven autonomic changes can all operate through overlapping autonomic pathways [4-7,12,17-19].

---

# 2. Mathematical formulation

For physiological feature `y_m(t)` from modality `m`, represent the observation as:

```text
y_m(t)
  = b_m(state_t)
  + g_activity,m(A_t, activity_type_t, posture_t, H_A(t))
  + g_thermal,m(Tskin_t, Tambient_t, H_T(t))
  + g_resp,m(R_t)
  + u_m(t)
  + ε_m(t)
```

where:

- `b_m(state_t)` = baseline conditional on relevant physical state;
- `A_t` = continuous activity intensity;
- `activity_type_t` = walking/running/stairs/etc. when reliably classifiable;
- `posture_t` = lying/sitting/standing/unknown;
- `H_A(t)` = retained activity history and recovery state;
- `Tskin_t` = local skin temperature;
- `Tambient_t` = ambient temperature if an independent ambient sensor exists;
- `H_T(t)` = recent thermal history;
- `R_t` = respiratory state where measurable;
- `u_m(t)` = residual physiology not explained by the measured confounds;
- `ε_m(t)` = model and measurement uncertainty.

For features where correction is defensible, define:

```text
predicted_confound_change = E[y_m | measured confounds] - conditional_baseline
residual_change           = observed_change - predicted_confound_change
```

The residual is **not** declared to be the desired target physiology. It is only the portion **not explained by the confounds currently measured and modeled**.

A confound-explanation score can be computed for compatible directions:

```text
explain_fraction = clamp(
    abs(predicted_confound_change) / (abs(observed_change) + ε),
    0, 1
)
```

but this scalar must never replace prediction intervals and reason codes. If predicted and observed changes have opposite signs, the confound does not “explain” the observed change and the event should be flagged as a possible interaction or unmodeled physiology.

---

# 3. Shared confound-observation engine

## 3.1 Why the IMU should produce both continuous and discrete outputs

Movement magnitude alone is insufficient. Equal acceleration can correspond to different metabolic loads, and posture can produce major cardiovascular changes with little movement. van de Ven et al. showed that adding **posture, activity type, and lagged movement intensity** significantly improved prediction of movement-related HR over movement magnitude alone; their best model increased average adjusted R² from 0.56 to 0.76 and reduced RMSE from 15.78 to 11.84 bpm [1].

Therefore the recommended IMU representation is **hybrid**:

1. **Continuous movement/intensity features** for regression and activity-dose history.
2. **Discrete physical state** for posture- and activity-specific effects.
3. **Transition/recovery state** for delayed physiology.

## 3.2 IMU preprocessing

### Research-supported starting pipeline

van de Ven et al. used a triaxial accelerometer, low-pass filtered acceleration at 5 Hz, high-pass filtered a copy at 0.1 Hz to remove gravity for movement-intensity calculation, processed 6 s epochs with 50% overlap, and aggregated predictions into 30 s physiological windows [1].

For MPU-6050 at ~100 Hz, implement:

```text
raw accel @ 100 Hz
    ↓
calibrate scale/bias
    ↓
5-Hz low-pass for human-body movement
    ├─ gravity/orientation channel: low-frequency component
    └─ dynamic channel: 0.1-Hz high-pass or gravity-removed acceleration
```

Do not copy device-specific cut-points without validation. Raw-acceleration thresholds vary strongly with **placement**; Hildebrand et al. showed systematic differences between hip and wrist output [20].

## 3.3 IMU features

For each short IMU epoch, compute:

### Vector magnitude / dynamic RMS

After gravity removal:

```text
VM_RMS = sqrt(mean(ax² + ay² + az²))
```

Input: dynamic acceleration.

Recommended window: **6 s**, 50% overlap, following [1].

Interpretation: continuous movement magnitude. Limitation: poor proxy for workload in activities with little body translation (e.g., cycling, isometric work) [1].

### ENMO

For raw acceleration in `g`:

```text
ENMO = max(sqrt(ax² + ay² + az²) - 1, 0)
```

Average over the epoch. ENMO is widely used for raw accelerometry. Published thresholds exist, but they are sensor- and placement-dependent; for example, Hildebrand-derived adult thresholds differ between non-dominant wrist and hip [20,21].

**Implementation decision:** use ENMO as a continuous score and only use published thresholds as external validation references. Learn MPU-6050/form-factor thresholds from the device's own calibration data.

### Signal Magnitude Area

```text
SMA = (1/N) Σ (|ax| + |ay| + |az|)
```

Use dynamic acceleration. Useful as a simple activity-intensity correlate, but not sufficient alone to estimate workload.

### Acceleration variance / standard deviation

Per axis and for vector magnitude:

```text
Var(a_x), Var(a_y), Var(a_z), Var(VM)
```

These are part of the feature families successfully used for static/dynamic and activity classification [1].

### Jerk RMS

```text
jerk_x[n] = (ax[n] - ax[n-1]) / Δt
jerk_RMS  = sqrt(mean(jerk_x² + jerk_y² + jerk_z²))
```

Useful for detecting abrupt transitions and differentiating smooth from highly dynamic movement. Treat as an engineering feature rather than a standalone physiological threshold.

### Orientation / posture features

Estimate the gravity vector from low-frequency acceleration. Compute sensor inclination and orientation features. van de Ven et al. included inclination and axis statistics and classified static periods into lying/sitting/standing [1].

**Critical placement constraint:** posture classification is only trustworthy if the IMU location contains body-orientation information. A trunk/waist/hip/thigh placement is much more suitable than a freely moving wrist. van de Ven et al. explicitly caution that activity-recognition models are placement-specific [1].

If the final hardware location cannot reliably classify posture, emit:

```text
posture = UNKNOWN
posture_confidence = low
```

rather than inferring posture from weak evidence.

### Step/cadence features

For ambulatory locomotion, estimate periodicity/cadence from peaks or autocorrelation of the dynamic acceleration magnitude. Use it to support walking/running state, not as a universal activity-intensity measure.

## 3.4 Physical-state representation

Recommended state machine:

```text
STATIC_REST
STATIC_NONREST
LIGHT_MOVEMENT
WALKING
MODERATE_DYNAMIC_ACTIVITY
VIGOROUS_DYNAMIC_ACTIVITY
LOW_MOTION_HIGH_LOAD_SUSPECTED
RECOVERY
UNKNOWN
```

with orthogonal posture:

```text
LYING | SITTING | STANDING | TRANSITION | UNKNOWN
```

Why not only `REST/LIGHT/MODERATE/VIGOROUS`? Because activity type and posture improve confound modeling beyond scalar movement intensity [1]. Why not only classes? Because continuous intensity preserves dose information and avoids threshold discontinuities.

## 3.5 Activity history

Maintain three causal histories:

```text
short_history:    last 3 min     # directly supported in [1]
recovery_history: last 10 min    # captures immediate post-exercise autonomic recovery [17,18]
long_dose:        last 30 min    # engineering state for cumulative recent activity
```

The **3-min lag** is research-supported by van de Ven et al., who used six preceding 30 s movement windows [1]. They explicitly note that physiological activity effects can persist longer and that longer lags may be appropriate in ambulatory use [1]. HRV literature shows immediate post-exercise recovery commonly occupies at least the first 0-10 min and can remain altered much longer after intense exercise [17].

Therefore movement ending does **not** end the confound.

---

# 4. Modality-specific confound definitions

## 4.1 ECG-derived heart rate

| Confound | Mechanism | Observable effect | Time scale | Auxiliary observation | Correctable? |
|---|---|---|---|---|---|
| Physical activity/exertion | increased metabolic demand, cardiac output and autonomic drive | HR rises with workload | seconds to minutes; persists into recovery | IMU intensity, activity type, activity history | **Partially** |
| Recent activity/recovery | parasympathetic reactivation and continued cardiovascular recovery after exercise | HR remains elevated after motion stops | minutes; potentially much longer after intense exercise | lagged IMU + HR recovery trajectory | **Partially** |
| Posture/orthostasis | venous pooling and baroreflex adjustment with upright posture | transient and sustained HR shift | immediate peak within seconds; stabilization over tens of seconds/minutes | IMU posture/transition | **Yes, conditionally** |
| Heat/thermal load | increased skin blood flow and cardiovascular strain; sympathetic activation | HR may increase; HRV may fall | minutes and sustained exposure | skin temp; ideally ambient temp | **Usually normalize/annotate** |
| Respiration | respiratory sinus arrhythmia modulates instantaneous beat intervals | beat-to-beat oscillation, strongest for HRV | breath-by-breath | ECG/PPG-derived respiration | **Do not correct mean HR routinely** |
| Low-motion high-load activity | cycling/isometric effort can raise HR without large body acceleration | HR elevated despite modest VM | activity-dependent | activity-type model; cadence/gyro; possibly unobserved | **Often ambiguous** |

Physical activity, posture, and recent activity each improve prediction of movement-related HR [1]. Active standing itself produces a rapid HR response: the initial response can last about 20 s, and reviews of upright posture describe a prompt HR rise peaking roughly 8-15 s after standing [15,16].

### Recommended action for mean HR

**CORRECT + NORMALIZE + ANNOTATE**, but only with individualized/confound-only calibration.

Use an “additional HR”-style model inspired by van de Ven et al. [1]:

```text
predicted_HR = β0
             + β1 * movement_intensity
             + activity/posture coefficients
             + βlag * recent_activity
             + optional validated interactions
```

Their published model used:

```text
pHR_i = β0
      + β1 VM_i
      + Σ βj ActivityClass_i,j
      + β8 * mean(VM over previous six 30-s windows)
      + ε_i
```

and:

```text
aHR = observed_HR - predicted_HR
```

[1].

### Engineering improvement over direct reproduction

The published model struggled with cycling because movement intensity did not track workload well [1]. Therefore:

- include **activity-type × movement-intensity interactions** only if enough calibration data exist;
- if activity type is unrecognized or known to have poor IMU–workload coupling, do not trust residualization;
- emit `LOW_MOTION_HIGH_LOAD_SUSPECTED` / `AMBIGUOUS` when HR rises strongly but IMU intensity remains low and no validated posture/thermal explanation exists.

### Avoiding over-correction

Do **not** train the physical-confound HR model indiscriminately on field data, because real autonomic events may co-occur with movement and become absorbed into the “movement” coefficient. Train/update only on calibration or high-confidence physical-only windows. van de Ven et al. found that additional-HR modeling improved separation of physical and mental loads but remained limited for within-person discrimination, which is a warning against treating the residual as ground truth [1].

---

## 4.2 ECG-derived HRV

Important HRV features include RMSSD, SDNN and frequency-domain components. HRV is highly state-sensitive: exercise and recovery change autonomic balance; posture shifts HRV; respiration substantially changes respiratory sinus arrhythmia (RSA) and HF-HRV [12,17-19,30].

| Confound | Mechanism | Effect on HRV | Time scale | Detection | Recommended handling |
|---|---|---|---|---|---|
| Physical activity | vagal withdrawal + sympathetic/metabolic response | often lower RMSSD/HF and altered spectrum | during activity | IMU | **GATE/DOWN-WEIGHT** |
| Post-exercise recovery | gradual parasympathetic reactivation | time-dependent recovery of RMSSD/HF | first 0-10 min prominent; may last longer | activity history + HR trajectory | **DOWN-WEIGHT/ANNOTATE** |
| Posture | baroreflex/autonomic shift | standing changes R-R and spectral HRV | immediate + sustained | posture | **NORMALIZE by posture** |
| Respiration rate/depth | directly modulates RSA/HF amplitude and frequency | can change HF/RSA without equivalent tonic vagal change | breath-by-breath | derived respiration | **CONDITION/ANNOTATE** |
| Heat | thermal autonomic load | HR rises, vagal indices can fall | minutes | temperature + activity | **DOWN-WEIGHT/ANNOTATE** |

Alikhani et al. reported substantial exercise-related influence on HRV and estimated average effects of ~39% during cycling and ~32% during walking in their cohort; motion trajectory could explain 20-40% of HF energy at different running workloads [4]. This paper includes both physiological and motion-related effects; in this document, signal corruption is assumed upstream, but the result reinforces that **off-rest HRV is not directly comparable to resting HRV**.

### Recommended action for HRV

**NORMALIZE + GATE + DOWN-WEIGHT + MARK AMBIGUOUS. Do not generally subtract a fitted confound value from HRV.**

Reason: exercise, posture, thermal stress, breathing and the autonomic phenomenon of interest all alter the same cardiac autonomic pathways. A numeric regression correction can remove meaningful autonomic physiology.

Recommended rules:

1. During **moderate/vigorous activity**, do not use resting-autonomic HRV features for downstream inference → `GATE`.
2. During **light movement**, retain only if downstream method has been validated for that state → otherwise `DOWNWEIGHT`.
3. During the **first 3 min after moderate/vigorous activity**, default `GATE` for resting-HRV interpretation (engineering starting rule grounded in the 3-min lag used by [1]).
4. From **3-10 min post activity**, `DOWNWEIGHT` and keep `RECOVERY` reason code; exercise-recovery literature shows this interval is still physiologically dynamic [17,18].
5. After 10 min, accept only if HR/activity/posture have stabilized relative to the appropriate baseline. Intense exercise can produce longer recovery, so “10 min elapsed” is not itself sufficient [17].
6. Use **posture-conditioned baselines**, not a universal HRV baseline.
7. For HF-HRV/RSA, require a usable respiration estimate. Respiratory pattern can substantially alter RSA/HF amplitude without a corresponding change in tonic vagal activity [12].
8. If respiratory rate is outside the conventional HF band (0.15-0.40 Hz) or respiration is irregular, mark HF-HRV as `RESPIRATION_SENSITIVE/AMBIGUOUS` rather than pretending a fixed spectral band has the same physiological meaning [12,30].

Standard short-term HRV is typically analyzed over ~5 min; ultra-short values are not interchangeable with standard short-term measurements [30]. For real-time use, RMSSD may be produced on shorter windows if the upstream HRV implementation has already been validated, but the confound engine must preserve the window-length metadata.

---

## 4.3 PPG-derived pulse rate and PRV

PPG contains two different kinds of information that need different confound treatment:

1. **pulse timing** (pulse rate / PRV), and
2. **peripheral pulse amplitude/morphology**, which depends strongly on the peripheral vascular bed.

### PRV is not HRV

PRV should not be treated as an interchangeable surrogate for ECG HRV. Respiratory, postural, pulse-transit-time, vascular and measurement-site effects can alter PRV differently from HRV [13,14]. Since ECG is available in this system, **ECG-derived HRV should be the primary cardiac-variability stream**.

Recommended action:

```text
PPG pulse rate → may supplement HR, especially for cross-checking
PPG PRV      → ANNOTATE as PRV; do not silently rename as HRV
ECG HRV      → preferred autonomic variability source
```

### PPG amplitude/perfusion confounds

| Confound | Mechanism | Observable PPG effect | Auxiliary signal | Correct? |
|---|---|---|---|---|
| Temperature/cold | peripheral vasoconstriction reduces superficial perfusion | lower PPG amplitude; morphology change | skin temp; ambient temp desirable | **Normalize/annotate** |
| Heat | vasodilation increases superficial flow but also changes cardiovascular state | amplitude/morphology shifts | temperature | **Normalize/annotate** |
| Posture | hydrostatic pressure, venous pooling and vascular adjustment | pulse amplitude, PPI and morphology can change | posture | **Posture-condition** |
| Respiration | baseline, amplitude and frequency modulation of PPG | respiratory oscillations in PPG amplitude/interval | derived respiration | **Annotate/model only for RR** |
| Exercise | cardiac + vascular response | pulse rate rises; amplitude/perfusion shifts | IMU + history | **Gate/down-weight for resting vascular interpretation** |

PPG amplitude is sensitive to superficial blood flow. Experimental cooling decreases PPG amplitude during temperature-induced peripheral vasoconstriction [25]. Skin temperature and PPG-measured fingertip blood-flow oscillations are coupled [24]. Posture also changes PPG-derived vascular features [26].

### Recommended action for PPG amplitude/morphology

**NORMALIZE + DOWN-WEIGHT + ANNOTATE; rarely direct correction.**

Why conservative? Peripheral vascular tone is itself autonomically regulated. Temperature and posture may be “confounds” for one downstream objective but part of meaningful physiology for another. Subtracting them aggressively risks deleting genuine vasomotor information.

Implementation:

- maintain posture-specific PPG amplitude baselines;
- maintain temperature-conditioned percentiles if ambient/local thermal state is stable;
- during posture transitions, activity and early recovery, do not compare PPG amplitude directly with resting baseline;
- if local temperature is changing materially, set `THERMAL_INFLUENCE` and down-weight peripheral-amplitude inference;
- if ECG is valid, do not use PRV as a replacement for HRV.

---

## 4.4 Electrodermal activity (EDA/GSR)

EDA is particularly difficult because both psychologically mediated sympathetic sudomotor activity and thermoregulatory sweating can change skin conductance. Ambient temperature and physical activity measurably influence ambulatory EDA [5], and established reporting recommendations explicitly recognize temperature, clothing, humidity and demographic/internal variables as relevant influences [29].

Thermoregulatory sweating is physiologically driven by skin/core temperature and exercise [28]. Recent multisite work further shows that EDA at wrist/forearm/upper arm can track physical sweat output while finger responses are more sensitive to mental activity, and simultaneous sites can help decouple physical and mental activity [6].

### EDA confound table

| Confound | Mechanism | EDA effect | Time scale | Observable? | Recommended action |
|---|---|---|---|---|---|
| Physical exertion | heat production + sympathetic sudomotor drive | tonic conductance rises; phasic behavior may change | minutes; may persist into cool-down | IMU + temp | **GATE/AMBIGUOUS** |
| Ambient heat | thermoregulatory sweating | conductance rises | minutes; sustained | ambient sensor ideally | **GATE/DOWN-WEIGHT** |
| Local skin temperature | affects sweating/skin electrical properties | SCL and reactivity can shift | slow-to-moderate | TMP117 | **ANNOTATE/NORMALIZE** |
| Hydration/sweat accumulation | changes skin conductivity and dynamic range | higher conductance / altered responsiveness | minutes-hours | not directly measured | **Individual baseline + ambiguity** |
| Post-activity sweating | sweat secretion continues after locomotion stops | elevated SCL despite quiet IMU | minutes | activity history + temperature + EDA slope | **RECOVERY flag** |
| Motion/contact | electrode pressure/contact changes | abrupt artifacts | immediate | quality layer | **Not primary scope** |

### Do not subtract exercise from EDA by default

EDA correction is **not** analogous to HR residualization. Physical and psychological sympathetic sudomotor activity share the same final effector system. With one EDA site, there is generally no uniquely identifiable decomposition:

```text
observed EDA rise
    = psychological sympathetic sudomotor drive
    + thermoregulatory sudomotor drive
    + individual sweat/skin factors
```

Therefore the recommended action is:

- **light activity:** `DOWNWEIGHT + ANNOTATE`;
- **moderate/vigorous activity:** `GATE` for interpretations that assume non-thermal EDA;
- **post-activity thermal recovery:** `AMBIGUOUS/DOWNWEIGHT` until activity, skin-temperature trend and EDA tonic trajectory stabilize;
- **ambient heat without independent ambient measurement:** `THERMAL_STATE_UNKNOWN`, not “no thermal confound.”

The 2025 multisite EDA approach is promising [6], but it requires additional measurement sites and specialized permeable-electrode design. It should be treated as an optional future hardware enhancement, not assumed available in the current single-site tinyGSR implementation.

---

## 4.5 Skin temperature

Skin temperature is not a simple core-temperature proxy. It reflects local perfusion, vasomotor control and heat transfer to/from the environment. Contact thermometry is also affected by attachment, pressure, environmental temperature and sensor setup [9].

The previous quality layer should handle abrupt contact/setup failures, but the confound layer must address **real skin-temperature changes caused by environment and activity**.

| Confound | Mechanism | Skin-temperature effect | Detectable? | Recommended handling |
|---|---|---|---|---|
| Ambient temperature | changes conductive/convective/radiative heat exchange | local Tskin drifts toward environmental conditions | not independently with one on-skin TMP117 | **AMBIGUOUS unless ambient reference** |
| Airflow/clothing | changes local heat transfer | raises/lowers local Tskin | generally unavailable | **Annotate uncertainty** |
| Physical activity | metabolic heat + redistribution of skin blood flow/sweating | dynamic Tskin response | IMU + history | **Activity-condition** |
| Peripheral vasoconstriction/vasodilation | changes cutaneous blood flow | lower/higher local Tskin | partly reflected by PPG | **Do not subtract blindly** |
| Sensor pressure/attachment | measurement/setup bias | can shift measured temperature | quality metadata | **Upstream quality issue** |

MacRae et al. found attachment-related temperature differences on the order of 0.1-1.4°C in reviewed studies and pressure-related differences up to ~1.33°C in available comparisons; these are measurement/setup effects, not autonomic confounds, and belong mainly to quality control [9]. The same review shows environmental conditions materially affect contact temperature measurement [9].

### Recommended action

If only one on-skin TMP117 exists:

**ANNOTATE + NORMALIZE + MARK AMBIGUOUS** for skin-temperature changes when environmental contribution cannot be independently estimated.

Do **not** fit a model that claims to separate ambient heat from vasomotor physiology using the same Tskin channel alone; the problem is under-observed.

Preferred hardware enhancement:

- retain the current on-skin TMP117;
- add a **thermally isolated ambient-temperature channel** if skin-temperature physiology will be an important downstream feature.

With ambient temperature available, model the skin-to-ambient gradient and its recent history instead of absolute Tskin alone.

---

# 5. Respiration as a confound

Respiration matters mainly for cardiac variability and PPG variability, not as a universal correction for every modality.

## 5.1 Why respiration must be tracked for HRV

RSA/HF-HRV is strongly shaped by respiratory rate and pattern; within-person changes in RSA/HF cannot be interpreted conclusively without considering breathing pattern [12]. Slow or deep breathing can substantially increase respiratory modulation even without an equivalent increase in tonic cardiac vagal activity [12].

## 5.2 Can respiration be estimated from existing ECG/PPG?

Yes, with limitations. Respiration modulates ECG and PPG through baseline wander, amplitude modulation and frequency modulation [10,11]. Charlton et al. compared hundreds of algorithm combinations and found that high-performing approaches fuse multiple respiratory modulations; their evaluation used **32 s windows** [11]. Under their evaluated conditions, the best ECG-based algorithms achieved 95% limits of agreement of roughly ±4.7 breaths/min; PPG-based performance was somewhat worse [11].

Therefore derived respiration is useful as a **confound indicator**, but it is not equivalent to a direct respiratory belt.

## 5.3 Recommended respiratory estimator

Online:

```text
ECG + PPG
  ↓
extract respiratory surrogate signals:
    ECG amplitude modulation
    ECG frequency modulation / RRI modulation
    PPG amplitude modulation
    PPG baseline modulation
    PPG frequency modulation
  ↓
estimate RR from each usable surrogate over ~32 s
  ↓
robust fusion / agreement check
  ↓
resp_rate + resp_confidence + regularity
```

Use a 32 s causal buffer as the research-supported default [11]. An engineering implementation may update every 4-8 s while retaining the most recent 32 s window.

Do **not** infer respiratory depth confidently from ECG/PPG alone unless separately validated; rate does not fully capture the respiratory effect on RSA.

## 5.4 HRV handling

For HF-HRV/RSA:

- if respiration estimate is reliable and stable: annotate HRV with respiratory rate;
- if respiratory rate shifts substantially from the individual's matched baseline: use respiration-conditioned comparison or `DOWNWEIGHT`;
- if respiration lies outside the standard HF frequency band or is irregular: `MARK AMBIGUOUS` for HF-HRV interpretation;
- if respiratory estimate is unreliable: do not assume normal breathing; reduce confidence.

Mean HR does not require routine respiration subtraction.

---

# 6. Temperature and thermoregulation model

## 6.1 Thermal influence on cardiac variables

Heat exposure can raise HR and reduce vagally mediated HRV. In controlled heat studies, HR increased and HF/RMSSD-type indices fell during high thermal load [22,23]. These studies demonstrate that thermal state can mimic an autonomic activation pattern even without locomotion.

Therefore IMU quietness is not sufficient evidence that elevated HR/low HRV is unconfounded.

## 6.2 Thermal influence on EDA

Thermoregulatory sweating is driven by thermal stress and exercise [28]. Ambulatory skin conductance varies with ambient temperature and physical activity [5]. Thus EDA requires a thermal confound state.

## 6.3 Thermal influence on PPG

Cold-induced peripheral vasoconstriction can reduce PPG amplitude without equivalent central cardiac change [25]. PPG amplitude therefore needs thermal annotation and local-temperature conditioning.

## 6.4 Recommended thermal state

With current hardware only:

```text
thermal_state:
    STABLE
    TSKIN_RISING
    TSKIN_FALLING
    EXERCISE_HEATING_SUSPECTED
    THERMAL_RECOVERY
    AMBIENT_UNKNOWN
```

If an ambient channel is added:

```text
ΔT_skin_ambient = Tskin - Tambient
```

and model both absolute temperature and gradient history.

### Features

- `Tskin_mean_30s`
- `Tskin_mean_2min`
- `dTskin/dt` over 60-120 s
- `Tskin deviation from subject's stable-rest baseline`
- optional `Tskin - Tambient`
- optional `d(Tskin - Tambient)/dt`
- recent activity dose to discriminate exercise-associated thermal rise from quiet environmental exposure.

No universal skin-temperature threshold is recommended. Learn subject- and placement-specific distributions because peripheral temperature is highly variable and setup/environment dependent [9].

---

# 7. Posture and orthostatic effects

A posture change is a real physiological intervention. Upright posture produces venous pooling, altered vascular resistance and baroreflex responses. Reviews report a prompt HR rise peaking around 8-15 s after assuming upright posture, while active standing can produce a ~20 s transient response involving muscle and baroreceptor reflexes [15,16]. HRV also changes between supine and standing states [19].

## 7.1 Posture detector

When IMU placement supports it:

```text
STATIC
  ↓
orientation/gravity vector
  ↓
LYING / SITTING / STANDING
```

Add `POSTURE_TRANSITION` when orientation changes between stable states.

## 7.2 Recommended temporal handling

**Research-supported facts:** immediate orthostatic effects occur over the first tens of seconds [15,16].

**Engineering starting rules requiring validation:**

- `POSTURE_TRANSITION` for **30 s** after a detected state change;
- for mean HR: do not compare to a different-posture baseline during this period;
- for HRV and PPG amplitude: `GATE` first 30 s and `DOWNWEIGHT` for the next 60-90 s if the feature has not stabilized;
- after stabilization, select the baseline for the current posture.

If posture cannot be reliably observed, mark the corresponding confound as **UNKNOWN** rather than assuming sitting/rest.

---

# 8. Baseline-conditioned interpretation

A universal baseline is inappropriate for features that differ by posture/activity state. van de Ven et al. explicitly used posture-specific HR baselines [1]. HRV and PPG vascular variables are also posture-sensitive [13,15,19,26].

Maintain baselines such as:

```text
HR:
  quiet_lying
  quiet_sitting
  quiet_standing
  light_walking

HRV:
  stable_lying
  stable_sitting
  stable_standing

PPG amplitude:
  posture × thermal_stable state

EDA:
  stationary_thermal_stable state

Tskin:
  posture/activity-conditioned stable distributions
```

## 8.1 Baseline learning rule

Update a baseline only from windows satisfying:

```text
signal_quality = PASS
activity_confidence high
activity intensity within allowed state
no posture transition
not in recovery
thermal state stable or known
no missing critical auxiliary sensor
```

Use robust statistics (median and MAD or trimmed mean/variance) rather than a plain mean so transient physiology does not rapidly redefine “normal.”

Do **not** adapt the baseline during unexplained physiological excursions. That would cause the system to learn away the phenomenon it is supposed to preserve.

---

# 9. Individual calibration

The relationship between HR, acceleration and physical-activity intensity varies substantially between people; individual calibration improves estimation [21]. Fitness, resting HR and habitual exercise also affect activity and orthostatic responses [19,21]. EDA has wide interindividual variability related to age, skin properties and other factors [29].

Therefore the confound model should be **person-specific where correction is performed**.

## 9.1 Recommended one-time calibration protocol

The following is an **engineering starting protocol**, not a published mandatory standard:

```text
1. seated quiet rest            5 min
2. quiet standing               2 min
3. comfortable walking          3 min
4. brisk walking                3 min
5. seated recovery              5-10 min
```

Optional if form factor supports it:

```text
6. lying rest                   3-5 min
7. stairs                       2-3 min
```

Collect ECG, PPG, EDA, temperature and IMU continuously.

Purpose:

- learn posture-specific HR/HRV/PPG baselines;
- learn subject-specific HR vs activity response;
- learn recovery dynamics;
- learn device/placement-specific IMU activity ranges;
- estimate typical EDA and Tskin behavior in non-confounded stationary states.

Do not use unrestricted field data to fit activity correction until a window is confidently labeled as confound-dominated.

## 9.2 Population prior + personalization

Practical deployment:

```text
population model
   ↓
initial device use
   ↓
controlled calibration
   ↓
personal coefficients / baselines
   ↓
slow adaptation from high-confidence windows only
```

This avoids requiring long laboratory calibration while preserving individual differences.

---

# 10. Temporal confound state and hysteresis

The subsystem must retain history because confound physiology has inertia.

## 10.1 State machine

```text
REST/STABLE
   ↓ activity onset
ACTIVE
   ↓ activity ends
POST_ACTIVITY_RECOVERY
   ↓ physiological + thermal stabilization
REST/STABLE
```

Orthogonal flags can coexist:

```text
POSTURE_TRANSITION
THERMAL_INFLUENCE
RESPIRATORY_INFLUENCE
AUX_SENSOR_UNKNOWN
```

## 10.2 Recovery should not be time-only

A fixed timer is insufficient. Use both elapsed time and convergence criteria.

Recommended recovery exit:

```text
activity below light threshold
AND no posture transition
AND short-lag activity dose decayed
AND HR trend consistent with current posture baseline
AND, for EDA/PPG temperature-sensitive interpretation,
    skin-temperature trend is stable enough for that subject
```

The exact stabilization tolerances should be learned during calibration. If they are not learned, remain conservative and keep `RECOVERY` longer rather than silently declaring “unconfounded.”

---

# 11. Modality-specific confound models

## 11.1 ECG heart-rate model

### Selected method

**Individualized linear/GAM residualization with posture/activity class and lagged activity, plus conservative out-of-domain detection.**

Start with the interpretable OLS structure directly supported by [1]. Upgrade to a generalized additive model (GAM) only when calibration data show clear nonlinearity and validation improves.

Input every 30 s:

```text
HR_mean
VM/ENMO_mean
activity_class
posture
mean movement in previous 3 min
recent activity dose (10/30 min)
thermal flags
```

Output:

```text
predicted_physical_HR
prediction_interval
additional_HR = observed - predicted
model_domain_score
```

Decision:

- inside calibrated domain + narrow PI → `CORRECT`;
- active but unsupported activity class → `AMBIGUOUS`;
- temperature confound present without ambient reference → retain residual but `DOWNWEIGHT/ANNOTATE`;
- posture unknown → reduce confidence.

## 11.2 ECG HRV model

### Selected method

**State-conditioned interpretation, not subtraction.**

```text
if moderate_or_vigorous_activity:
    GATE
elif early_recovery:
    GATE or strong DOWNWEIGHT
elif recovery:
    DOWNWEIGHT
elif posture_transition:
    GATE
elif respiration_unreliable and feature is HF/RSA:
    AMBIGUOUS
else:
    compare with posture-matched stable baseline
```

Optional offline research analysis may fit mixed-effects models to quantify physical-state effects, but the real-time production path should remain conservative.

## 11.3 PPG model

### Pulse rate

Use the same physical-state model as HR, but ECG is the preferred timing reference when available.

### PRV

`ANNOTATE + NORMALIZE + DOWNWEIGHT`; prefer ECG HRV for autonomic variability [13,14].

### PPG amplitude/morphology

Use matched-baseline normalization:

```text
normalized_PPG_feature =
    (feature - median_baseline(posture, thermal_bin)) /
    robust_scale(posture, thermal_bin)
```

Do not subtract a thermal “effect” unless an ambient/reference model has been separately validated.

## 11.4 EDA model

### Selected method

**Rule-based confound gating + uncertainty annotation; no routine mathematical subtraction.**

```text
if moderate/vigorous activity:
    GATE for non-thermal EDA inference
elif thermal_state strongly active:
    AMBIGUOUS or GATE
elif post_activity_recovery:
    DOWNWEIGHT / AMBIGUOUS
elif light movement:
    DOWNWEIGHT
else:
    ACCEPT with current thermal annotation
```

Use current activity + recent activity + Tskin trajectory together. This is intentionally conservative because physical and psychological sweating are not uniquely separable from one EDA site.

## 11.5 Skin-temperature model

### Selected method

**Conditional normalization + ambiguity, not subtraction, unless independent ambient temperature is added.**

With ambient sensor:

```text
Tskin_expected = f(Tambient, posture, activity, recent_activity, subject)
residual_Tskin = Tskin - Tskin_expected
```

but validate carefully because autonomic vasomotor changes can correlate with the same variables.

Without ambient sensor:

```text
thermal_confidence <= moderate
```

for downstream interpretations that require separating environmental from vasomotor temperature effects.

---

# 12. Avoiding removal of real physiology

This is the central safety property of the subsystem.

## 12.1 Correlated target and confound

If movement and autonomic activation occur together, an unconstrained field-trained regression can learn:

```text
movement → HR increase
```

and subtract genuine target-related HR that happened during movement.

### Safeguards

1. **Train physical correction on controlled/confound-only periods**, not all field data.
2. Preserve the raw physiological feature alongside corrected/residualized output.
3. Store prediction uncertainty and the confound-only model estimate separately.
4. Never replace observed physiology in storage; correction is an additional channel.
5. Require out-of-distribution detection for unseen activity types/intensities.
6. Do not use EDA subtraction with a single site.
7. Do not use regression-corrected HRV as the primary real-time output.
8. If two mechanisms are physiologically inseparable, output `AMBIGUOUS`.

## 12.2 Interaction terms

Confounds need not add linearly. For example:

```text
activity × heat
posture × activity
activity_type × acceleration
respiration × posture
```

may alter physiology. Only add interaction terms when calibration/validation data support them. Otherwise widening uncertainty is preferable to fitting a high-capacity black box that appears precise.

---

# 13. Multimodal confound fusion

Use an **interpretable hierarchical fusion**, not arbitrary feature concatenation.

## 13.1 Stage 1: observable confound states

```text
IMU ─────────────→ activity_intensity, activity_type, posture, transitions
TMP117 ──────────→ local thermal trend
ECG/PPG ─────────→ respiratory estimate + confidence
activity history ─→ recovery dose/state
```

## 13.2 Stage 2: modality influence map

Map each confound to affected features:

```text
activity:
  HR HIGH
  HRV HIGH
  PPG pulse rate HIGH
  PPG amplitude MEDIUM/HIGH
  EDA HIGH
  skin temp MEDIUM/HIGH

posture:
  HR HIGH
  HRV HIGH
  PPG amplitude HIGH
  EDA LOW/MEDIUM
  skin temp MEDIUM

thermal:
  HR MEDIUM
  HRV MEDIUM
  PPG amplitude HIGH
  EDA HIGH
  skin temp HIGH

respiration:
  mean HR LOW/MEDIUM
  HRV-HF HIGH
  PRV HIGH
  PPG amplitude modulation MEDIUM
  EDA LOW
```

## 13.3 Stage 3: per-modality action

No single confound score should automatically “explain” all physiology. Each modality receives its own correction/action based on identifiability.

---

# 14. Confound-state output representation

Recommended production structure:

```yaml
confound_state:
  activity:
    state: WALKING
    intensity_score: 0.63
    confidence: 0.94
    recent_dose_3m: 0.48
    recent_dose_10m: 0.31
    recovery: false

  posture:
    state: STANDING
    confidence: 0.88
    seconds_since_transition: 142

  thermal:
    state: STABLE
    skin_temp_c: 32.4
    skin_temp_slope_c_per_min: 0.03
    ambient_available: false
    confidence: 0.55

  respiration:
    rate_bpm: 15.8
    regularity: 0.86
    confidence: 0.78

per_modality:
  ecg_hr:
    action: CORRECT
    predicted_confound_change: 14.2
    residual_change: 4.1
    residual_confidence: 0.72
    reason_codes:
      - PHYSICAL_ACTIVITY
      - POSTURE_STANDING

  ecg_hrv_rmssd:
    action: DOWNWEIGHT
    confidence: 0.42
    reason_codes:
      - ACTIVE_STATE

  ppg_amplitude:
    action: NORMALIZE
    confidence: 0.58
    reason_codes:
      - POSTURE_STANDING
      - AMBIENT_TEMPERATURE_UNKNOWN

  eda_phasic:
    action: AMBIGUOUS
    confidence: 0.31
    reason_codes:
      - PHYSICAL_ACTIVITY
      - THERMOREGULATORY_SWEATING_POSSIBLE

interpretation_status: ACCEPT_WITH_CAUTION
```

## 14.1 Standard actions

```text
ACCEPT
CORRECT
NORMALIZE
DOWNWEIGHT
ANNOTATE
AMBIGUOUS
GATE
UNKNOWN
```

`GATE` means “do not use this feature for the downstream inference that assumes an unconfounded state”; it does not mean discard the stored physiological data.

---

# 15. Confidence model

For modality `m`:

```text
C_m = C_aux × C_state × C_model × C_domain × C_identifiability
```

where each factor is `[0,1]`:

- `C_aux`: required auxiliary sensors available/reliable;
- `C_state`: confidence in activity/posture/thermal/respiratory state;
- `C_model`: cross-validated/calibrated confound model quality;
- `C_domain`: similarity to calibration domain;
- `C_identifiability`: whether confound and target are separable for this modality.

The EDA `C_identifiability` should be deliberately low during exercise/thermal sweating. HR residualization can have higher identifiability when activity/posture are well measured and the state is calibrated.

---

# 16. Missing confound data

Missing auxiliary data must produce **unknown**, never “confound absent.”

| Missing input | Consequence | Required fallback |
|---|---|---|
| IMU unavailable | activity/posture/recovery cannot be established | HR correction disabled; HRV/EDA/PPG confidence reduced; `ACTIVITY_UNKNOWN` |
| TMP117 unavailable | thermal confound poorly observed | EDA and PPG-amplitude confidence reduced; skin-temp feature unavailable |
| ambient temperature absent | environmental vs vasomotor Tskin not separable | skin-temp interpretation `AMBIGUOUS`; EDA thermal correction not attempted |
| respiration estimate unreliable | RSA/HF confound unresolved | HF-HRV/PRV `DOWNWEIGHT/AMBIGUOUS`; mean HR may continue |
| ECG unavailable | cardiac timing reference lost | PPG pulse rate usable; PRV must not be silently promoted to HRV |
| posture unreliable because of IMU placement | orthostatic confound unresolved | use `POSTURE_UNKNOWN`; avoid posture-specific correction |

---

# 17. Multiple simultaneous confounds

Simultaneous confounds are expected:

```text
walking + heat
walking + posture transition
exercise + altered breathing
recovery + cooling
```

## 17.1 Combination rule

Do not use:

```text
if activity: physiology_explained = true
```

Instead evaluate:

```text
predicted contribution / influence from each measured confound
+ interactions that have been validated
+ uncertainty from unmeasured confounds
```

When several confounds are strongly correlated and their individual contributions cannot be separated, report a grouped reason:

```text
PHYSICAL_THERMAL_CONFOUND_CLUSTER
```

and reduce confidence.

## 17.2 Dominant-confound policy

A dominant confound can justify gating (e.g., vigorous exercise for resting HRV) even if other confounds are also present. It should not be interpreted as explaining the full observed physiological change.

---

# 18. Per-modality confound map

| Physiological feature | Major confound | Typical effect | Detection signal | Correction possible? | Recommended action | Key refs |
|---|---|---|---|---|---|---|
| ECG HR | physical activity | ↑ HR | IMU intensity/type | partial | CORRECT + ANNOTATE | [1,21] |
| ECG HR | recent activity | HR remains elevated after movement | lagged IMU + HR recovery | partial | CORRECT/DOWNWEIGHT | [1,17,18] |
| ECG HR | posture | transient/sustained HR shift | IMU posture | yes, conditioned | NORMALIZE + transition gate | [1,15,16] |
| ECG HR | heat | ↑ HR | Tskin; ambient preferred | limited | DOWNWEIGHT/ANNOTATE | [22,23] |
| ECG RMSSD/HF | activity | vagal indices fall/change | IMU | unsafe to subtract | GATE/DOWNWEIGHT | [4,17] |
| ECG RMSSD/HF | recovery | delayed restoration | activity history | unsafe to subtract | DOWNWEIGHT | [17,18] |
| ECG HF/RSA | respiration | amplitude/frequency changes | ECG/PPG-derived RR | condition only | NORMALIZE/AMBIGUOUS | [10-12] |
| ECG HRV | posture | spectral/time-domain shift | posture | conditional | posture baseline | [19,30] |
| PPG pulse rate | activity | ↑ pulse rate | IMU | partial | follow HR policy | [1] |
| PPG PRV | respiration/posture/vascular timing | differs from ECG HRV | respiration/posture/ECG | not reliably “correct to HRV” | ANNOTATE; prefer ECG | [13,14] |
| PPG amplitude | cold/temperature | lower amplitude with vasoconstriction | Tskin/ambient | partial | NORMALIZE/DOWNWEIGHT | [24,25] |
| PPG morphology | posture | amplitude/morphology change | posture | conditional | posture baseline | [26] |
| PPG amplitude | exercise/recovery | vascular + cardiac shifts | IMU/history | limited | DOWNWEIGHT/ANNOTATE | [1,25] |
| EDA tonic/phasic | physical exertion | increased thermoregulatory sweating | IMU + temp | not uniquely separable | GATE/AMBIGUOUS | [5,6,28] |
| EDA | ambient/local heat | ↑ conductance/sweat | ambient + Tskin | limited | GATE/DOWNWEIGHT | [5,28,29] |
| EDA | post-exercise sweating | remains elevated after movement | history + Tskin + EDA trend | limited | RECOVERY/AMBIGUOUS | [5,28] |
| Skin temperature | ambient environment | local Tskin follows heat exchange | ambient sensor | with ambient only | NORMALIZE/AMBIGUOUS | [9] |
| Skin temperature | activity | metabolic/vascular/evaporative response | IMU/history | limited | activity-condition | [9] |
| Skin temperature | local perfusion | vasoconstriction/dilation changes Tskin | PPG + thermal state | overlaps target | ANNOTATE/AMBIGUOUS | [24,25] |

---

# 19. Thresholds and parameters

The table explicitly separates literature-derived values from engineering starting values.

| Parameter | Value | Type | Source / note |
|---|---:|---|---|
| IMU body-motion low-pass | 5 Hz | **Research-supported** | van de Ven et al. [1] |
| IMU gravity-removal high-pass for VM | 0.1 Hz | **Research-supported** | [1] |
| IMU short epoch | 6 s | **Research-supported** | [1] |
| IMU epoch overlap | 50% | **Research-supported** | [1] |
| HR modeling window | 30 s | **Research-supported** | [1] |
| HR lag history | six × 30 s = 3 min | **Research-supported** | [1] |
| Derived respiration window | 32 s | **Research-supported** | Charlton et al. [11] |
| Orthostatic immediate HR response | ~first 20 s; HR peak about 8-15 s in upright response literature | **Research-supported** | [15,16] |
| Immediate post-exercise autonomic recovery analysis | 0-10 min commonly studied | **Research-supported** | [17] |
| Standard short-term HRV | ~5 min | **Research-supported** | [30] |
| Hildebrand adult ENMO moderate threshold | ~93-101 mg wrist; ~69 mg hip depending device | **Research-supported but NOT portable to MPU-6050 without validation** | [20] |
| Posture transition gate | 30 s | **Engineering starting parameter** | chosen conservatively from [15,16] |
| HRV early recovery gate | first 3 min after MVPA | **Engineering starting parameter** | anchored to [1]; validate |
| HRV recovery down-weight | 3-10 min after MVPA | **Engineering starting parameter** | anchored to [17,18] |
| Respiration estimator update | every 4-8 s using 32 s buffer | **Engineering starting parameter** | buffer supported by [11] |
| Thermal trend window | 60-120 s | **Engineering starting parameter** | validate per form factor |
| Long activity history | 30 min | **Engineering starting parameter** | retained for dose/context; not a fixed gate |

Do not use literature thresholds from a different accelerometer placement/device as if they were universal.

---

# 20. Windowing and online temporal state

Recommended online scheduling:

```text
IMU raw:              ~100 Hz
IMU feature window:   6 s, 50% overlap
fast state update:    every 1-3 s
HR confound model:    30 s causal window, update every 5-10 s
respiration:          32 s causal window, update every 4-8 s
thermal trend:        60-120 s causal trend, update every 5-10 s
EDA confound state:   update every 5-10 s from activity + thermal history
recovery history:     continuously retained
```

The exact physiological-feature windows are upstream responsibilities; this layer consumes their timestamps and support intervals.

---

# 21. Real-time implementation architecture

```text
              trustworthy/decomposed physiology
               /        |        |        \
             ECG       PPG      EDA      Tskin
               \        |        |        /
                \       |        |       /
                       fusion clock
                           +
                    IMU confound engine
                           +
                 ECG/PPG respiration estimator
                           ↓
                 temporal confound state
                           ↓
        ┌──────────────────┼────────────────────┐
        ↓                  ↓                    ↓
   HR residualizer   state-conditioned     EDA/thermal
                    HRV/PPG normalizer       gating
        └──────────────────┼────────────────────┘
                           ↓
              per-modality confound action
                           ↓
        raw feature + adjusted/normalized feature
        + confidence + prediction interval
        + confound scores + reason codes
```

## 21.1 Computational complexity

The recommended online methods are lightweight:

- IMU filtering/features: `O(N)` per window;
- activity classifier: small tree/linear model or compact model; placement-specific;
- HR OLS/GAM inference: constant-time per update;
- derived respiration: modest spectral/time-domain computation on ~32 s buffers;
- rule/state machine: constant-time;
- robust baseline updates: constant-time streaming statistics.

These are suitable for continuous execution on the backend and many can run on an ESP32 if needed, but model training and high-fidelity offline analysis should remain backend-side.

## 21.2 Online vs offline

### Online production

- causal filters only;
- IMU states;
- 3/10/30 min activity histories;
- HR residualization;
- posture/activity-conditioned normalization;
- respiration confidence;
- conservative gating/ambiguity logic.

### Optional offline/high-fidelity

- mixed-effects analysis across subjects;
- richer GAMs/state-space models;
- long-horizon exercise recovery modeling;
- retrospective direct comparison of ECG-HRV and PPG-PRV;
- recalibration of activity thresholds and interaction terms.

---

# 22. Per-modality final algorithms

## 22.1 ECG heart rate

```text
INPUT:
    HR from valid ECG
    IMU features/state
    posture
    Tskin/thermal state
    activity history
    personal calibration model

1. Aggregate HR to a 30-s causal estimate.
2. Read continuous movement intensity and activity class.
3. Read current posture and posture-transition state.
4. Compute 3-min lagged activity dose.
5. Determine whether current state is inside calibration domain.
6. Predict physical/postural HR contribution using the individual model.
7. Generate prediction interval.
8. If calibrated and identifiable:
       residual_HR = observed_HR - predicted_physical_HR
       action = CORRECT
   Else:
       keep observed HR
       action = DOWNWEIGHT or AMBIGUOUS
9. Add thermal reason codes; do not claim thermal correction without an adequate model.
10. Emit observed HR, predicted confound HR, residual HR, confidence and reason codes.
```

## 22.2 ECG HRV

```text
INPUT:
    validated HRV features
    IMU/activity/recovery
    posture
    derived respiration + confidence
    thermal state
    posture-specific baselines

1. Determine active/recovery state.
2. If moderate/vigorous activity → GATE resting-HRV inference.
3. If early recovery → GATE or strong DOWNWEIGHT.
4. If later recovery → DOWNWEIGHT until stabilized.
5. If posture transition → GATE transient interval.
6. Select posture-matched HRV baseline.
7. For HF/RSA:
       require usable respiration estimate;
       if breathing outside expected HF interpretation or irregular → AMBIGUOUS.
8. Do not subtract a fitted activity value from RMSSD/HF in the production path.
9. Emit normalized deviation, state, confidence, and reason codes.
```

## 22.3 PPG

```text
INPUT:
    valid PPG pulse timing and amplitude/morphology
    ECG availability
    IMU/activity/recovery
    posture
    Tskin/thermal state
    respiration estimate

1. If ECG available, use ECG as primary HR/HRV timing source.
2. Compute PPG pulse rate; apply HR-like activity annotation.
3. Label pulse variability explicitly as PRV.
4. Select posture/thermal-matched PPG amplitude baseline.
5. During posture transition/activity/recovery → DOWNWEIGHT amplitude/morphology.
6. During material thermal change → THERMAL_INFLUENCE.
7. Do not convert PRV into HRV by correction.
8. Emit pulse rate, PRV, normalized PPG amplitude/morphology, confound state and confidence.
```

## 22.4 EDA

```text
INPUT:
    decomposed trustworthy EDA
    activity state/intensity
    activity history
    Tskin and thermal trend
    optional ambient temperature

1. Determine physical-activity state.
2. Determine thermal state and recent exercise heating.
3. If moderate/vigorous activity:
       GATE non-thermal EDA inference.
4. Else if thermal influence strong:
       AMBIGUOUS or DOWNWEIGHT.
5. Else if post-activity recovery:
       DOWNWEIGHT until activity + thermal + tonic EDA trajectory stabilize.
6. Else if light movement:
       DOWNWEIGHT and annotate.
7. Else:
       ACCEPT relative to stationary thermal-stable personal baseline.
8. Do not subtract a modeled exercise waveform from single-site EDA.
9. Emit original decomposed EDA + confound status/confidence.
```

## 22.5 Skin temperature

```text
INPUT:
    trustworthy Tskin
    IMU activity/history
    posture
    optional Tambient
    optional PPG perfusion features

1. Compute Tskin mean and 60-120 s trend.
2. Determine activity/recovery state.
3. If Tambient exists:
       compute skin-ambient gradient and matched baseline.
   Else:
       mark environmental contribution incompletely observed.
4. During activity/recovery, use activity-conditioned baseline.
5. If ambient contribution cannot be separated from vasomotor change:
       MARK AMBIGUOUS rather than correcting.
6. Emit Tskin, normalized Tskin, thermal confound state and confidence.
```

---

# 23. Final synthesis table

| Modality | Physiological feature | Major confounds | Confound measurement | Selected model | Temporal handling | Production action | Key refs |
|---|---|---|---|---|---|---|---|
| ECG | mean HR | activity, activity type, posture, recent exertion, heat | IMU + history + Tskin | individual OLS/GAM physical-HR model | 30 s HR; 3 min lag; recovery retained | **CORRECT + NORMALIZE + ANNOTATE** | [1,15-18,21-23] |
| ECG | RMSSD/SDNN | activity, recovery, posture | IMU + history + posture | state-conditioned baseline | gate active; down-weight recovery | **NORMALIZE + GATE + DOWN-WEIGHT** | [4,17-19,30] |
| ECG | HF-HRV/RSA | respiration, activity, posture | derived respiration + IMU | respiration-conditioned interpretation | ~32 s RR; HRV feature window metadata | **DOWN-WEIGHT + AMBIGUOUS when RR unreliable** | [10-12,30] |
| PPG | pulse rate | activity/posture/recovery | IMU/history | follow HR model; ECG preferred | same activity history | **ANNOTATE/CORRECT cautiously** | [1] |
| PPG | PRV | respiration, posture, vascular transit | ECG + respiration + posture | no conversion to HRV | state-conditioned | **ANNOTATE + NORMALIZE; prefer ECG HRV** | [13,14] |
| PPG | amplitude/morphology | temperature, posture, exercise/perfusion | Tskin + posture + IMU | matched baseline | thermal/activity recovery | **NORMALIZE + DOWN-WEIGHT + ANNOTATE** | [24-26] |
| EDA | tonic/phasic outputs | physical exertion, heat, thermoregulatory sweating, recovery | IMU + history + Tskin (+ambient ideally) | rule/state fusion | thermal/activity persistence | **GATE + DOWN-WEIGHT + AMBIGUOUS** | [5,6,28,29] |
| Skin temp | local Tskin | ambient environment, activity, perfusion | IMU; ambient optional | conditional normalization | slow thermal history | **NORMALIZE + ANNOTATE + AMBIGUOUS** | [9,24,25] |
| IMU | activity/posture | placement/device limitations | self | hybrid continuous + classes | 6 s epochs + multi-minute history | auxiliary confound stream | [1,20,21] |

---

# 24. References

[1] Sjors R. B. van de Ven, Martin J. Gevonden, Matthijs L. Noordzij, Eco J. C. de Geus.  
“Accelerometer-based heart rate adjustment for ambulatory stress research.”  
*Psychophysiology*, 62(1), e14721, 2025.  
DOI: https://doi.org/10.1111/psyp.14721  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC11775876/

[2] Karen Hovsepian, Mustafa al'Absi, Emre Ertin, Thomas Kamarck, Motohiro Nakajima, Santosh Kumar.  
“cStress: Towards a Gold Standard for Continuous Stress Assessment in the Mobile Environment.”  
*Proceedings of the ACM International Joint Conference on Pervasive and Ubiquitous Computing (UbiComp)*, 2015, pp. 493-504.  
DOI: https://doi.org/10.1145/2750858.2807526  
PubMed: https://pubmed.ncbi.nlm.nih.gov/26543926/

[3] Martin Gjoreski, Mitja Luštrek, Matjaž Gams, Hristijan Gjoreski.  
“Monitoring stress with a wrist device using context.”  
*Journal of Biomedical Informatics*, 73, 159-170, 2017.  
DOI: https://doi.org/10.1016/j.jbi.2017.08.006  
PubMed: https://pubmed.ncbi.nlm.nih.gov/28803947/

[4] Iman Alikhani, Kai Noponen, Arto Hautala, Tapio Seppänen.  
“Characterization and reduction of exercise-based motion influence on heart rate variability using accelerator signals and channel decoding in the time-frequency domain.”  
*Physiological Measurement*, 39(11), 115002, 2018.  
DOI: https://doi.org/10.1088/1361-6579/aadeff  
PubMed: https://pubmed.ncbi.nlm.nih.gov/30183678/

[5] Sigrun Doberenz, Walton T. Roth, Eileen Wollburg, Nina I. Maslowski, Sunyoung Kim.  
“Methodological considerations in ambulatory skin conductance monitoring.”  
*International Journal of Psychophysiology*, 80(2), 87-95, 2011.  
DOI: https://doi.org/10.1016/j.ijpsycho.2011.02.002  
PubMed: https://pubmed.ncbi.nlm.nih.gov/21320551/

[6] Seung-Rok Kim, Yifei Zhan, Noelle Davis, et al.  
“Electrodermal activity as a proxy for sweat rate monitoring during physical and mental activities.”  
*Nature Electronics*, 8, 353-361, 2025.  
DOI: https://doi.org/10.1038/s41928-025-01365-7  
URL: https://www.nature.com/articles/s41928-025-01365-7

[7] Md-Billal Hossain, Youngsun Kong, Hugo F. Posada-Quintero, Ki H. Chon.  
“Comparison of Electrodermal Activity from Multiple Body Locations Based on Standard EDA Indices’ Quality and Robustness against Motion Artifact.”  
*Sensors*, 22(9), 3177, 2022.  
DOI: https://doi.org/10.3390/s22093177  
PubMed: https://pubmed.ncbi.nlm.nih.gov/35590866/

[8] Sara Taylor, Natasha Jaques, Weixuan Chen, Szymon Fedor, Akane Sano, Rosalind Picard.  
“Automatic Identification of Artifacts in Electrodermal Activity Data.”  
*IEEE EMBC*, 2015, pp. 1934-1937.  
DOI: https://doi.org/10.1109/EMBC.2015.7318762  
PubMed: https://pubmed.ncbi.nlm.nih.gov/26736662/

[9] Braid A. MacRae, Simon Annaheim, Christina M. Spengler, René M. Rossi.  
“Skin Temperature Measurement Using Contact Thermometry: A Systematic Review of Setup Variables and Their Effects on Measured Values.”  
*Frontiers in Physiology*, 9:29, 2018.  
DOI: https://doi.org/10.3389/fphys.2018.00029  
PubMed: https://pubmed.ncbi.nlm.nih.gov/29441024/

[10] Peter H. Charlton, Drew A. Birrenkott, Timothy Bonnici, et al.  
“Breathing Rate Estimation From the Electrocardiogram and Photoplethysmogram: A Review.”  
*IEEE Reviews in Biomedical Engineering*, 11, 2-20, 2018.  
DOI: https://doi.org/10.1109/RBME.2017.2763681  
PubMed: https://pubmed.ncbi.nlm.nih.gov/29990026/

[11] Peter H. Charlton, Timothy Bonnici, Lionel Tarassenko, David A. Clifton, Richard Beale, Peter J. Watkinson.  
“An assessment of algorithms to estimate respiratory rate from the electrocardiogram and photoplethysmogram.”  
*Physiological Measurement*, 37(4), 610-626, 2016.  
DOI: https://doi.org/10.1088/0967-3334/37/4/610  
PubMed: https://pubmed.ncbi.nlm.nih.gov/27027672/

[12] Thomas Ritz.  
“Putting back respiration into respiratory sinus arrhythmia or high-frequency heart rate variability: Implications for interpretation, respiratory rhythmicity, and health.”  
*Biological Psychology*, 185, 108728, 2024.  
DOI: https://doi.org/10.1016/j.biopsycho.2023.108728  
PubMed: https://pubmed.ncbi.nlm.nih.gov/38092221/

[13] Isabelle Constant, Dominique Laude, Isabelle Murat, Jean-Luc Elghozi.  
“Pulse rate variability is not a surrogate for heart rate variability.”  
*Clinical Science*, 97(4), 391-397, 1999.  
PubMed: https://pubmed.ncbi.nlm.nih.gov/10491338/

[14] Emi Yuda, Muneichi Shibata, Yuki Ogata, Norihiro Ueda, Tomoyuki Yambe, Makoto Yoshizawa, Junichiro Hayano.  
“Pulse rate variability: a new biomarker, not a surrogate for heart rate variability.”  
*Journal of Physiological Anthropology*, 39, 21, 2020.  
DOI: https://doi.org/10.1186/s40101-020-00233-x  
PubMed: https://pubmed.ncbi.nlm.nih.gov/32811571/

[15] J. J. Smith, C. M. Porth, M. Erickson.  
“Hemodynamic response to the upright posture.”  
*Journal of Clinical Pharmacology*, 34(5), 375-386, 1994.  
DOI: https://doi.org/10.1002/j.1552-4604.1994.tb04977.x  
PubMed: https://pubmed.ncbi.nlm.nih.gov/8089249/

[16] C. Borst, W. Wieling, J. F. van Brederode, A. Hond, L. G. de Rijk, A. J. Dunning.  
“Mechanisms of initial heart rate response to postural change.”  
*American Journal of Physiology*, 243(5), H676-H681, 1982.  
DOI: https://doi.org/10.1152/ajpheart.1982.243.5.H676  
PubMed: https://pubmed.ncbi.nlm.nih.gov/7137360/

[17] Scott Michael, Kenneth S. Graham, Glen M. Davis.  
“Cardiac Autonomic Responses during Exercise and Post-exercise Recovery Using Heart Rate Variability and Systolic Time Intervals—A Review.”  
*Frontiers in Physiology*, 8:301, 2017.  
DOI: https://doi.org/10.3389/fphys.2017.00301  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC5447093/

[18] Tiago Peçanha, Rhenan Bartels, Leandro C. Brito, Marcelle Paula-Ribeiro, Ricardo S. Oliveira, Jeffrey J. Goldberger.  
“Methods of assessment of the post-exercise cardiac autonomic recovery: A methodological review.”  
*International Journal of Cardiology*, 227, 795-802, 2017.  
DOI: https://doi.org/10.1016/j.ijcard.2016.10.057  
PubMed: https://pubmed.ncbi.nlm.nih.gov/27836300/

[19] Mercedes R. Carnethon, Duanping Liao, Gregory W. Evans, Wayne E. Cascio, Lloyd E. Chambless, Gerardo Heiss.  
“Correlates of the shift in heart rate variability with an active postural change in a healthy population sample: The Atherosclerosis Risk In Communities study.”  
*American Heart Journal*, 143(5), 808-813, 2002.  
DOI: https://doi.org/10.1067/mhj.2002.121928  
PubMed: https://pubmed.ncbi.nlm.nih.gov/12040341/

[20] Maria Hildebrand, Vincent T. van Hees, Bjørge H. Hansen, Ulf Ekelund.  
“Age Group Comparability of Raw Accelerometer Output from Wrist- and Hip-Worn Monitors.”  
*Medicine & Science in Sports & Exercise*, 46(9), 1816-1824, 2014.  
DOI: https://doi.org/10.1249/MSS.0000000000000289  
PubMed: https://pubmed.ncbi.nlm.nih.gov/24887173/

[21] Søren Brage, Ulf Ekelund, Niels Brage, Mark A. Hennings, Karsten Froberg, Paul W. Franks, Nicholas J. Wareham.  
“Hierarchy of individual calibration levels for heart rate and accelerometry to measure physical activity.”  
*Journal of Applied Physiology*, 103(2), 682-692, 2007.  
DOI: https://doi.org/10.1152/japplphysiol.00092.2006  
PubMed: https://pubmed.ncbi.nlm.nih.gov/17463305/

[22] Stewart S. Bruce-Low, David Cotterrell, Gareth E. Jones.  
“Heart rate variability during high ambient heat exposure.”  
*Aviation, Space, and Environmental Medicine*, 77(9), 915-920, 2006.  
PubMed: https://pubmed.ncbi.nlm.nih.gov/16964740/

[23] Shinji Yamamoto, Mieko Iwamoto, Masaiwa Inoue, Noriaki Harada.  
“Evaluation of the effect of heat exposure on the autonomic nervous system by heart rate variability and urinary catecholamines.”  
*Journal of Occupational Health*, 49(3), 199-204, 2007.  
DOI: https://doi.org/10.1539/joh.49.199  
PubMed: https://pubmed.ncbi.nlm.nih.gov/17575400/

[24] A. A. Sagaidachnyi, A. V. Skripal, A. V. Fomin, D. A. Usanov.  
“Determination of the amplitude and phase relationships between oscillations in skin temperature and photoplethysmography-measured blood flow in fingertips.”  
*Physiological Measurement*, 35(2), 153-166, 2014.  
DOI: https://doi.org/10.1088/0967-3334/35/2/153  
PubMed: https://pubmed.ncbi.nlm.nih.gov/24399251/

[25] S. Jung, S. Thomson, A. Pantelopoulos, L. Sunden, P. Richards, S. Patel, S. Sheng.  
“Differential sensitivity of impedance plethysmography and photoplethysmography sensors to temperature-induced peripheral vasoconstriction.”  
*Scientific Reports*, 16, 6828, 2026.  
DOI: https://doi.org/10.1038/s41598-026-36563-6  
PubMed: https://pubmed.ncbi.nlm.nih.gov/41620497/

[26] Apakrita Tayade, Saurav Kumar, Amber Shrivastava, Ravi Bhallamudi.  
“Effect of posture on photoplethysmography signals from the posterior tibial artery in adults with and without type 2 diabetes.”  
*Scientific Reports*, 15, 43737, 2025.  
DOI: https://doi.org/10.1038/s41598-025-28223-y  
PubMed: https://pubmed.ncbi.nlm.nih.gov/41387484/

[27] A. Y. M. Jones, C. Kam, K. W. Lai, H. Y. Lee, H. T. Chow, S. F. Lau, L. M. Wong, J. He.  
“Changes in heart rate and R-wave amplitude with posture.”  
*Chinese Journal of Physiology*, 46(2), 63-69, 2003.  
PubMed: https://pubmed.ncbi.nlm.nih.gov/12974297/

[28] Manabu Shibasaki, Craig G. Crandall.  
“Neural control and mechanisms of eccrine sweating during heat stress and exercise.”  
*Journal of Applied Physiology*, 100(5), 1692-1701, 2006.  
DOI: https://doi.org/10.1152/japplphysiol.01124.2005  
PubMed: https://pubmed.ncbi.nlm.nih.gov/16614366/

[29] Wolfram Boucsein, Don C. Fowles, Sverre Grimnes, Gershon Ben-Shakhar, Walton T. Roth, Michael E. Dawson, Diane L. Filion.  
“Publication recommendations for electrodermal measurements.”  
*Psychophysiology*, 49(8), 1017-1034, 2012.  
DOI: https://doi.org/10.1111/j.1469-8986.2012.01384.x  
PubMed: https://pubmed.ncbi.nlm.nih.gov/22680988/

[30] Fred Shaffer, J. P. Ginsberg.  
“An Overview of Heart Rate Variability Metrics and Norms.”  
*Frontiers in Public Health*, 5:258, 2017.  
DOI: https://doi.org/10.3389/fpubh.2017.00258  
PubMed: https://pubmed.ncbi.nlm.nih.gov/29034226/

[31] Task Force of the European Society of Cardiology and the North American Society of Pacing and Electrophysiology.  
“Heart rate variability: standards of measurement, physiological interpretation and clinical use.”  
*Circulation*, 93(5), 1043-1065, 1996.  
PubMed: https://pubmed.ncbi.nlm.nih.gov/8598068/

[32] Che-Chang Yang, Yeh-Liang Hsu.  
“A Review of Accelerometry-Based Wearable Motion Detectors for Physical Activity Monitoring.”  
*Sensors*, 10(8), 7772-7788, 2010.  
DOI: https://doi.org/10.3390/s100807772  
PubMed: https://pubmed.ncbi.nlm.nih.gov/22163626/

---

# Recommended Implementation

Build **one interpretable confound engine with modality-specific actions**, rather than one universal “confound-corrected physiology” model.

## A. Exact inputs

```text
ECG:
  HR, RR intervals, validated HRV features

PPG:
  pulse rate, pulse intervals/PRV, amplitude/morphology features

EDA:
  upstream-decomposed tonic/phasic features

TMP117:
  local skin temperature

MPU-6050:
  accel xyz @ ~100 Hz
  gyro xyz

Derived:
  respiration rate/confidence from ECG+PPG
  activity/posture state
  activity history/recovery state
```

Keep raw/upstream features unchanged; produce confound-aware features as additional channels.

## B. IMU-derived activity representation

Use **both** continuous intensity and discrete state.

### Preprocessing

```text
100 Hz accel
→ 5 Hz low-pass
→ gravity/orientation stream
→ 0.1 Hz high-pass dynamic stream
```

### 6 s window, 50% overlap

Compute:

```text
VM_RMS
ENMO
SMA
axis variance/SD
jerk RMS
inclination/orientation
axis correlations
cadence/periodicity when locomotion suspected
```

### State output

```text
STATIC_REST
LIGHT_MOVEMENT
WALKING
MODERATE_DYNAMIC_ACTIVITY
VIGOROUS_DYNAMIC_ACTIVITY
LOW_MOTION_HIGH_LOAD_SUSPECTED
RECOVERY
UNKNOWN
```

plus:

```text
LYING | SITTING | STANDING | TRANSITION | UNKNOWN
```

If IMU placement is not suitable for posture, disable posture correction and emit `UNKNOWN`.

## C. Activity/recovery history

Retain:

```text
3 min   → research-supported short lag for HR model
10 min  → immediate autonomic recovery state
30 min  → cumulative activity dose / out-of-domain protection
```

Activity cessation never immediately clears the confound.

## D. Posture handling

- classify posture only when device placement supports it;
- create posture-specific baselines;
- mark 30 s after a posture change as `POSTURE_TRANSITION` (**engineering starting value**);
- do not compare HRV/PPG vascular features across postures without normalization.

## E. Thermal handling

With one on-skin TMP117:

- compute 30 s mean, 2 min mean, 60-120 s slope;
- detect stable/rising/falling/physical-heating/recovery thermal states;
- use Tskin to **flag** thermal influence on EDA/PPG/HR;
- do **not** claim ambient correction.

Strongly preferred future hardware addition if skin-temperature physiology matters:

```text
second thermally isolated ambient-temperature channel
```

Then model `Tskin - Tambient` and its history.

## F. Respiration

Estimate respiratory rate from multiple ECG/PPG modulations over a **32 s causal buffer**, fuse estimates and produce a confidence score [10,11].

Use respiration primarily to qualify **HF-HRV/RSA and PRV**.

If respiratory estimate is unreliable:

```text
HF-HRV → DOWNWEIGHT / AMBIGUOUS
mean HR → continue if other confounds are adequately observed
```

Do not infer respiratory depth unless separately validated.

## G. Per-modality model/actions

### ECG mean HR

**Strategy: `CORRECT + NORMALIZE + ANNOTATE`**

Use an individually calibrated interpretable model:

```text
pHR = f(
    current VM/ENMO,
    activity class,
    posture,
    previous 3-min activity,
    validated interactions
)

additional_HR = observed_HR - pHR
```

Use 30 s model windows. Produce a prediction interval. If activity is outside calibration domain (cycling/isometric/unknown high-load state), disable correction and mark `AMBIGUOUS`.

### ECG HRV

**Strategy: `NORMALIZE + DOWN-WEIGHT + GATE + MARK AMBIGUOUS`**

Do not mathematically subtract an “exercise HRV component” in production.

```text
active MVPA          → GATE
0-3 min recovery     → GATE / strong DOWNWEIGHT
3-10 min recovery    → DOWNWEIGHT
posture transition   → GATE transiently
stable state         → compare to posture-matched baseline
HF/RSA               → require respiration qualification
```

### PPG pulse rate

**Strategy: `ANNOTATE`; use HR correction logic only if ECG is unavailable.**

ECG remains primary for cardiac timing.

### PPG PRV

**Strategy: `ANNOTATE + NORMALIZE + DOWN-WEIGHT`**

Never silently treat PRV as ECG HRV. Prefer ECG HRV whenever available.

### PPG amplitude/morphology

**Strategy: `NORMALIZE + DOWN-WEIGHT + ANNOTATE`**

Normalize against posture/thermal-stable personal baselines. Do not aggressively correct temperature because peripheral vasomotor physiology may itself be meaningful.

### EDA

**Strategy: `GATE + DOWN-WEIGHT + ANNOTATE + MARK AMBIGUOUS`**

Do not perform single-site exercise subtraction.

```text
moderate/vigorous activity      → GATE
strong thermal influence         → GATE/AMBIGUOUS
post-exercise thermal recovery   → DOWNWEIGHT/AMBIGUOUS
light movement                   → DOWNWEIGHT
stationary + thermal stable      → ACCEPT
```

### Skin temperature

**Strategy: `NORMALIZE + ANNOTATE + MARK AMBIGUOUS`**

Without ambient temperature, do not claim separation of environment from vasomotor physiology. If ambient channel is later added and validated, conditional residualization becomes possible.

## H. Calibration

Perform one-time subject calibration:

```text
5 min seated rest
2 min standing
3 min normal walking
3 min brisk walking
5-10 min seated recovery
optional lying + stairs
```

Use it to learn:

- IMU intensity ranges for the actual placement/device;
- posture baselines;
- HR physical-response coefficients;
- recovery dynamics;
- stable EDA/PPG/Tskin distributions.

Adapt later only from high-confidence confound-only windows.

## I. Confidence and gating

For each modality output:

```text
observed_feature
adjusted_or_normalized_feature (if allowed)
confound_state(s)
confound_strength(s)
action
prediction_interval / uncertainty
confidence
reason_codes
```

Recommended top-level statuses:

```text
ACCEPT
ACCEPT_WITH_CORRECTION
ACCEPT_WITH_CAUTION
DOWNWEIGHT
AMBIGUOUS
GATE
UNKNOWN
```

## J. Missing auxiliary sensors

Never map missing data to “confound absent.”

```text
IMU missing        → disable physical correction; activity/posture UNKNOWN
TMP117 missing     → EDA/PPG thermal confidence reduced
respiration poor   → HF-HRV/PRV confidence reduced
ambient missing    → skin-temperature environmental separation unavailable
ECG missing        → PPG usable, but PRV remains PRV
```

## K. Final system behavior

The system should answer the intended question in this form:

```text
Observed HR increase: +18 bpm
Estimated activity/posture contribution: +13 ± 4 bpm
Residual: +5 bpm
State: walking, standing, recent activity present
Thermal influence: low/unknown
Action: ACCEPT_WITH_CORRECTION
Residual confidence: 0.72
```

or, where separation is not physiologically defensible:

```text
Observed EDA increase: substantial
State: vigorous activity + rising skin temperature
Physical/thermoregulatory sweating is plausible and shares the same
sudomotor pathway as the physiology of interest.
Action: AMBIGUOUS / GATE
No numerical EDA subtraction performed.
```

That distinction is the core engineering principle of the layer:

> **Correct only when the confound contribution is independently measurable and identifiable; otherwise condition, down-weight, annotate, gate, or explicitly mark ambiguity.**
