# Context-Aware Physiological Interpretation Solution

**Engineering design document**  
**Subsystem:** Context-Aware Interpretation  
**Position in pipeline:** after signal trustworthiness, physiological decomposition, and confound-aware monitoring; before AUD-specific physiological meaning/action.

---

## Executive summary

This subsystem answers one question:

> **Given a physiological change that is already technically trustworthy, physiologically meaningful, and not sufficiently explained by known confounds, what does the surrounding situation change about how that physiology should be interpreted?**

The literature strongly argues against interpreting ambulatory autonomic signals in isolation. Autonomic responses are flexible, context-sensitive, and frequently non-specific: similar changes in heart rate (HR), heart-rate variability (HRV), electrodermal activity (EDA), peripheral vascular state, and skin temperature can occur during different behavioral, cognitive, emotional, social, sleep/wake, and environmental conditions [1–6]. A useful context-aware subsystem therefore must **not** map a physiological feature directly to a psychological label. It should instead estimate **expected physiology for this person in this context**, compare the observed response with that expectation, and emit a **generic, context-relative physiological interpretation with uncertainty**.

The recommended implementation is a **factorized temporal context model** rather than one monolithic context label. The core dimensions are:

- wake state;
- activity/mobility state;
- posture;
- task/cognitive setting when observable;
- social-interaction setting when observable;
- coarse environment/location category when available;
- time-of-day / circadian context;
- event markers;
- recent context transitions and time since transition.

Each dimension carries:

- value or `UNKNOWN`;
- probability/confidence;
- source/provenance;
- timestamp/freshness;
- transition status.

Each physiological feature has a **relevant-context specification**. For example, HRV requires wake/sleep, posture, stable-state duration, time-of-day, and—when interpreting vagal/HF components—respiration; phasic EDA requires event/task/social context much more strongly than precise location; PPG amplitude and morphology require posture and thermal state because peripheral vascular responses are strongly context-dependent [8,11,18–21].

The recommended inference strategy is:

1. estimate context dimensions independently;
2. temporally smooth them;
3. retrieve a personalized context-conditioned baseline or prediction model;
4. compute a context-relative deviation;
5. account for recent events and context transitions;
6. propagate context and baseline uncertainty;
7. emit a generic physiological description such as:
   - `CARDIOVASCULAR_ACTIVATION_ELEVATED_FOR_CONTEXT`;
   - `CARDIAC_VAGAL_VARIABILITY_LOWER_THAN_EXPECTED_FOR_CONTEXT`;
   - `ELECTRODERMAL_ACTIVATION_EVENT_ASSOCIATED`;
   - `PERIPHERAL_VASCULAR_RESPONSE_UNUSUAL_FOR_CONTEXT`;
   - `PHYSIOLOGY_REAL_BUT_CONTEXT_INSUFFICIENT`.

It must **not** emit craving, withdrawal, relapse risk, intoxication, anxiety disorder, or any other AUD-specific or diagnostic conclusion.

---

# 1. Scope and hard boundary

The processing chain is:

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
confound-aware monitoring
  ↓
CONTEXT-AWARE INTERPRETATION        ← this document
  ↓
AUD-specific physiological meaning/action
```

This document assumes its physiological inputs have already passed upstream checks.

Therefore this layer does **not** decide:

- whether ECG/PPG/EDA contact is good;
- whether motion artifact corrupted a signal;
- whether decomposition into tonic/phasic or baseline/event components was correct;
- whether a physiological change is primarily explained by a confound already handled upstream;
- whether the user is craving alcohol;
- whether withdrawal or intoxication is present;
- whether relapse risk is high;
- whether a diagnosis or treatment action is warranted.

The output remains **generic physiological meaning conditioned on context**.

---

# 2. Operational definition of context

For this system:

> **Context is information about the person's behavioral, temporal, social, task, and environmental situation that changes the expected physiological state or changes what can legitimately be inferred from a real physiological response.**

A contextual variable is relevant only if at least one of the following is true:

1. it changes the expected distribution of a physiological feature;
2. it changes the likely generic physiological interpretation of a response;
3. it changes which baseline or comparison state is scientifically appropriate;
4. it changes interpretation confidence;
5. it identifies an event or transition to which a physiological response can be temporally related.

This definition deliberately excludes a variable that merely explains away bad signal. That belongs upstream.

## 2.1 Context is not the same as a confound

```text
Confound question:
"Did another physical/physiological factor substantially cause or distort
the measured change?"

Context question:
"Given that the change is real, which situation should it be interpreted within?"
```

Example:

```text
HR rises because the person begins running
→ upstream confound/activity-cause handling

HR rises while seated during a demanding conversation
versus while seated quietly and alone
→ context-aware interpretation
```

A second example is more subtle:

```text
thermoregulatory sweating explains an EDA rise
→ confound

a valid phasic EDA response occurs during an emotionally salient event
versus during a routine neutral task
→ context
```

The context subsystem should not re-run confound rejection. It receives the upstream result and answers a different question.

---

# 3. Why context is necessary

## 3.1 Autonomic physiology is not one-to-one with psychological states

Reviews and meta-analyses show that autonomic response patterns vary substantially within nominal emotion categories and do not support a simple one-feature-to-one-state mapping [2,3]. The 2026 critical review of real-world wearable ANS monitoring makes the same point specifically for ambulatory sensing: behavioral state and social/environmental situation shape the meaning of wearable autonomic signals, and isolated interpretation is unsafe [1].

This has a direct engineering consequence:

> **Never encode rules such as `EDA↑ = stress`, `HR↑ = anxiety`, or `HRV↓ = craving`.**

The strongest context-aware output usually remains physiological:

```text
"electrodermal sympathetic activation increased"
"cardiovascular activation is higher than expected for this context"
"cardiac vagal variability is lower than this person's context-specific norm"
"peripheral vascular response changed after a known event"
```

The exact psychological cause may remain unresolved.

## 3.2 Context-aware systems can improve inference

Gjoreski et al. explicitly used contextual activity information with wrist physiology to improve real-life stress inference [4]. cStress used physical activity as an admission-control condition rather than treating all physiological windows equivalently [5]. Smets et al. collected free-living HR, skin conductance, skin temperature, acceleration, location, sleep/wake annotations, activity, and EMA information in more than 1,000 participants and emphasized heterogeneity and the value of multimodal contextual measurement [7].

Hoemann et al. provide an especially useful methodological precedent: ambulatory ECG and accelerometry were processed in real time, and large heart-period changes triggered experience sampling only when movement/posture had not changed; the event-triggered labels then supplied situational context around physiology [6].

For this system, these studies motivate a **context state + expected physiology + context-relative deviation** architecture rather than a direct physiology classifier.

---

# 4. Physiological modalities covered

The current sensing architecture is treated as exposing the following *trusted upstream physiological streams/features*:

1. **ECG-derived cardiovascular features**
   - heart rate;
   - RR/NN intervals;
   - time-domain HRV (e.g., RMSSD, SDNN where valid);
   - frequency-domain HRV where window duration and respiratory assumptions support it.

2. **PPG-derived features**
   - pulse rate;
   - pulse-rate variability / pulse-interval variability where validated;
   - PPG pulse amplitude;
   - selected pulse morphology / timing features;
   - optional validated SpO₂ if the upstream PPG pipeline has a calibrated implementation.

3. **EDA/GSR-derived features**
   - tonic skin conductance level / slow tonic state;
   - phasic response magnitude/rate/event density.

4. **Skin temperature**
   - absolute peripheral/contact skin temperature where the mounting site is known;
   - temperature trend;
   - deviation from personal time-of-day/context baseline.

5. **Optional derived respiration**
   - only if an upstream ECG/PPG-derived respiratory-rate estimator has already declared the estimate trustworthy.

6. **IMU**
   - not treated as a physiological output;
   - used primarily as a context source for activity, posture, transitions, and sleep/wake support.

---

# 5. Context dimensions actually worth representing

A flat list of dozens of labels is not recommended. The implementation should use a **factorized context state**.

## 5.1 Mandatory core dimensions

### A. Wake state

```text
AWAKE
SLEEP
WAKE_TRANSITION
SLEEP_TRANSITION
UNKNOWN
```

Why it matters:
- HR/HRV differ strongly between wake and sleep;
- sleep stage and circadian phase alter autonomic regulation [16,17];
- wrist temperature changes around sleep onset and awakening [14];
- nocturnal respiratory rate and SpO₂ have different expected ranges than daytime measurements.

### B. Activity context

```text
STILL
LIGHT_MOVEMENT
WALKING
RUNNING_OR_VIGOROUS
STRUCTURED_EXERCISE
OTHER_ACTIVE
UNKNOWN
```

Activity state is useful even after upstream confound handling because:
- context-conditioned expectations differ;
- recovery immediately after activity is not equivalent to stable rest;
- respiration, HR, PPG morphology, and vascular tone remain context-dependent.

### C. Posture

```text
LYING
SITTING
STANDING
POSTURE_TRANSITION
UNKNOWN
```

Posture materially alters HRV and PPG characteristics [18,20,21]. Therefore a valid response in supine rest should not automatically be compared with a standing baseline.

### D. Temporal context

Represent continuously rather than as one categorical label:

```text
local_clock_time
time_since_wake
time_since_sleep_onset
day_phase: MORNING / DAY / EVENING / NIGHT
circadian_phase_proxy: optional
```

Time-of-day is important for HRV, EDA tonic level, skin temperature, and sleep-related features [14–16,23].

### E. Event context

Events are timestamped objects, not permanent labels.

```text
event_type
event_start
event_end
event_source
event_confidence
```

Examples:
- user event button;
- phone call start/end;
- task start/end;
- conversation label;
- wake-up;
- location/environment transition;
- meal when explicitly logged;
- self-reported meaningful event.

### F. Transition context

Every relevant state dimension should expose:

```text
previous_value
current_value
transition_time
time_since_transition
transition_confidence
```

Physiology during a transition should not immediately be compared to a stable-context baseline.

---

## 5.2 Optional dimensions

These should be included only when the necessary source is available and consented.

### Task/cognitive setting

```text
PASSIVE_REST
ROUTINE_TASK
FOCUSED_SCREEN_TASK
HIGH_DEMAND_TASK
CONVERSATION
OTHER_TASK
UNKNOWN
```

Cognitive demand can reduce HRV even when HR itself changes little [9,10]. Therefore task context can materially improve cardiovascular interpretation.

However, cognitive load should **not be inferred from HRV/EDA themselves** in this subsystem. That would use the response being interpreted as evidence for its own context.

### Social setting

```text
ALONE
INTERACTING_ONE_TO_ONE
INTERACTING_GROUP
PASSIVE_CROWD
UNKNOWN
```

Social interactions can alter HR, HRV, and EDA; social context can also alter the direction or magnitude of psychophysiological responses [24–27]. But current hardware alone cannot reliably determine rich social context. When no direct or user-provided evidence exists, `UNKNOWN` is the correct state.

### Coarse environment/location category

```text
HOME
WORK_OR_STUDY
COMMUTE
PUBLIC_INDOOR
OUTDOOR
OTHER_KNOWN_PLACE
UNKNOWN
```

Do not store exact GPS coordinates for this layer unless another system explicitly requires them. A coarse semantic category is enough for baseline conditioning.

### Environmental state

Only when separately sensed:

```text
ambient_temperature
noise_level
light_level
indoor_outdoor
```

Do **not** use body-worn skin temperature as a surrogate for ambient temperature. It is itself a physiological measurement and is influenced by thermoregulation and circadian state.

---

# 6. Recommended context representation

A single label such as `RESTING_AT_HOME` is too brittle because:

- dimensions may have different confidence;
- one dimension may be missing while others are known;
- source disagreement must be visible;
- physiological features require different subsets of context.

Use a **factorized probabilistic context object**.

```yaml
context_state:
  timestamp: 2026-10-16T21:30:05+05:30

  wake:
    value: AWAKE
    probability: 0.98
    source: sleep_wake_engine
    age_s: 5

  activity:
    value: STILL
    probability: 0.94
    source: imu_activity_classifier
    age_s: 1

  posture:
    value: SITTING
    probability: 0.87
    source: imu_posture_classifier
    age_s: 1

  task:
    value: FOCUSED_SCREEN_TASK
    probability: 0.71
    source: smartphone_context
    age_s: 12

  social:
    value: UNKNOWN
    probability: null
    source: none
    age_s: null

  location:
    value: WORK_OR_STUDY
    probability: 0.84
    source: smartphone_semantic_location
    age_s: 40

  time:
    local_hour: 21.50
    day_phase: EVENING
    time_since_wake_min: 820

  active_events:
    - type: SCREEN_TASK
      start_time: ...
      confidence: 0.78

  transitions:
    posture:
      previous: STANDING
      current: SITTING
      seconds_since_transition: 42

context_provenance:
  ...
```

The model must support `UNKNOWN` natively. Unknown is not a low-probability known class; it means evidence is insufficient.

---

# 7. Why a factorized hierarchy is preferable to a flat context label

A strict tree such as:

```text
REST
 ├─ alone
 ├─ social
 └─ task
```

is useful conceptually but poor as the storage model because real life violates a single tree: a person can be sitting, on a phone call, at work, after walking, in the evening, while transitioning from one task to another.

The practical representation should therefore be:

```text
macro state
  ↓
independent context dimensions
  ↓
feature-specific relevance selection
  ↓
context-conditioned physiology model
```

The *hierarchy* is used in baseline fallback:

```text
person + exact relevant context
        ↓ if sparse
person + macro behavioral context
        ↓ if sparse
person + wake state + time block
        ↓ if sparse
person global baseline
        ↓ if sparse
population prior
```

This avoids the combinatorial explosion of a Cartesian product of all context labels.

---

# 8. Context acquisition

## 8.1 Immediately available from current hardware

### IMU

Use for:

- still / walking / vigorous movement;
- posture or coarse body orientation when mounting permits;
- state transitions;
- activity intensity;
- prolonged inactivity;
- sleep/wake support when the mounting location is compatible.

### System clock

Use for:

- time-of-day;
- day/night;
- time since wake;
- time since context transition;
- event temporal distance.

### Device state

Use for:

- worn/not-worn state if available from upstream;
- charging/disconnected state;
- sensor streaming state.

### Physiological state as *cross-modal support*

Physiology may be used to improve a context estimate only when it does not create circular interpretation.

Safe example:

```text
IMU strongly suggests sleep period
+ long-duration low movement
+ time-of-day
+ stable low HR
→ sleep probability increases
```

Unsafe example:

```text
EDA is high
→ infer "stressful task"
→ use "stressful task" to explain high EDA
```

The latter is circular and must be prohibited.

---

## 8.2 Smartphone-derived context

Where available and consented, the phone can add:

- coarse semantic location;
- phone movement/activity state;
- screen on/off;
- foreground application *category* rather than content;
- phone-call active state;
- calendar event category if explicitly enabled;
- Bluetooth proximity;
- Wi-Fi/environment transition;
- optional conversation-presence inference.

StudentLife demonstrated that smartphones can continuously infer sleep, physical activity, conversation/sociability, and location-related behaviors [28]. For this subsystem, however, only context that materially changes physiological interpretation should be retained.

### Privacy-minimizing recommendation

Prefer:

```text
location = HOME
```

over raw GPS.

Prefer:

```text
call_active = true
```

over phone number/contact identity.

Prefer:

```text
speech_interaction_present = true
```

over raw microphone recordings.

---

## 8.3 User-provided context

High-value user inputs:

- event button;
- one-tap activity label;
- one-tap social label;
- short EMA;
- “meaningful event just occurred” marker.

User input is especially useful for dimensions current sensors cannot reliably infer, such as:

- emotionally salient event;
- difficult conversation;
- cognitively demanding task;
- unusual situation.

Do not burden the user continuously. Use event-triggered or sparse EMA. Event-triggered ambulatory sampling around genuine physiological changes is supported by Hoemann et al. [6].

---

## 8.4 Optional future context sensors

Potential additions only if future validation shows benefit:

- dedicated ambient temperature;
- ambient light;
- noise level without audio storage;
- barometric altitude;
- proximity sensors.

Do not add a context sensor merely because it is technically possible.

---

# 9. Context inference algorithms

The implementation should favor interpretable, low-latency modules with probabilities.

## 9.1 Activity and posture

### Inputs

- tri-axial acceleration;
- gyroscope if available;
- mounting orientation metadata;
- previous state.

### Window

Recommended engineering default:

```text
window: 5 s
step: 1 s
```

For vigorous activity, 2–5 s is sufficient. For posture, use gravity/orientation features and longer hysteresis.

### Features

- acceleration magnitude mean/std;
- vector-magnitude area;
- jerk;
- dominant frequency;
- spectral energy bands;
- gravity vector;
- axis variance;
- gyroscope energy;
- orientation change.

### Model

**Selected implementation:** small gradient-boosted-tree or random-forest classifier, followed by a causal temporal state filter.

Why:
- nonlinear enough for coarse activity;
- interpretable feature importance;
- cheap enough for continuous use;
- outputs probabilities;
- easier to validate than a deep sequence model.

### Temporal smoothing

Use a small state transition matrix:

```text
P(state_t | state_t-1)
```

and online forward filtering or hysteresis.

Do not jump immediately from `SITTING` to `RUNNING` based on one 1-s output unless confidence is very high.

### Output

```yaml
activity:
  posterior:
    STILL: 0.94
    WALKING: 0.04
    OTHER: 0.02
```

The research literature supports exploiting temporal structure and hierarchical activity categories; hierarchical HMM approaches are a precedent [29]. A lightweight probabilistic smoother is recommended here rather than a large HMM over the entire combined context state.

---

# 10. Sleep/wake inference

Sleep/wake is one of the most important context dimensions.

Classic wrist-actigraphy algorithms such as Cole-Kripke have been validated against polysomnography, but actigraphy typically has high sleep sensitivity and weaker wake specificity [30–32].

## Recommended implementation

### If the IMU is wrist-mounted

Use:

1. actigraphy-based sleep/wake probability in 30-s or 60-s epochs;
2. local time and habitual sleep schedule as weak priors;
3. optional PPG/HR support;
4. temporal smoothing;
5. explicit `WAKE_TRANSITION` and `SLEEP_TRANSITION` states.

### If the IMU is not wrist-mounted

Do **not** reuse Cole-Kripke coefficients blindly.

Instead:

- collect device-specific labeled sleep/wake data;
- train a simple classifier using the actual mounting location;
- retain the same output interface and uncertainty handling.

### Limitation

Actigraphy should not be treated as reliable sleep-stage classification. For this subsystem, wake/sleep plus transition states are sufficient for the first version.

---

# 11. Task/cognitive context

Cognitive load changes cardiovascular autonomic behavior. Studies show HRV can decrease with mental workload or sustained attentional demand, sometimes without a corresponding HR increase [9,10].

Therefore:

```text
low HRV during focused work
```

cannot automatically be interpreted against a passive-rest baseline.

## Recommended sources

Priority order:

1. explicit user task/event;
2. calendar/task timer;
3. screen/app-category context;
4. heuristic:
   - low movement;
   - screen active;
   - sustained device interaction.

Do not infer cognitive load from HRV/EDA in this layer.

## Output categories

Keep coarse:

```text
PASSIVE_REST
ROUTINE_TASK
FOCUSED_TASK
CONVERSATION
UNKNOWN
```

Do not attempt detailed workload grading until validated.

---

# 12. Social context

Social context can alter cardiovascular and electrodermal responses. Experimental and ambulatory studies demonstrate that interaction quality, social presence, and social task context can change HR/HRV/EDA responses [24–27].

## What can realistically be sensed now

From current hardware alone:

- essentially nothing reliable beyond movement patterns.

From smartphone:

- active call;
- Bluetooth proximity;
- optional speech-presence;
- coarse social-location pattern.

From user:

- interaction start/end;
- one-tap “alone / social.”

## Recommendation

Social context is **optional but high-value** for HR/HRV and EDA.

Do not block generic interpretation when it is absent. Instead:

```text
social_context = UNKNOWN
context_sufficiency = PARTIAL
```

and reduce psychological specificity.

Example:

```text
Allowed:
"electrodermal activation elevated relative to quiet-rest baseline;
social/task context is unavailable."

Not allowed:
"social anxiety detected."
```

---

# 13. Environmental context

## 13.1 Thermal context

Thermal state is particularly important for EDA, PPG amplitude/morphology, and skin temperature.

Qasim et al. found ambient temperature significantly affected tonic EDA, while phasic EDA was more robust [11]. Peripheral vascular responses to mental or arousing stimuli can even change magnitude or direction depending on thermoregulatory state [19]. Stress and thermoregulation can both activate sympathetic physiology but produce different patterns [20].

Therefore, if dedicated ambient temperature is available:

```text
ambient_temperature
thermal_transition
time_since_thermal_transition
```

should be included for:
- tonic EDA;
- skin temperature;
- PPG amplitude / vascular features.

Without a dedicated ambient sensor, the system should explicitly mark thermal context as unavailable instead of pretending worn skin temperature equals ambient temperature.

## 13.2 Indoor/outdoor and location category

Useful mainly for:
- baseline selection;
- thermal/environmental transition;
- routine vs unusual context.

Not every physiological feature needs it.

---

# 14. Temporal context and history

Context is a sequence, not a snapshot.

Use:

```text
current context
+ previous context
+ time since transition
+ duration in current state
+ recent events
+ pre-event baseline
+ post-event recovery
```

## 14.1 Online history buffers

Recommended practical buffers:

```text
high-resolution context history: 60 min
event history: 24 h
context summary history: multi-day
baseline sufficient statistics: persistent
```

The 60-min buffer is an engineering choice that comfortably covers:
- activity recovery;
- wake transitions;
- task transitions;
- EDA response/recovery;
- thermal drift.

## 14.2 Feature-specific transition guards

These are initial engineering defaults, not universal physiological constants; validate them on the target population.

| Feature | Transition sensitivity | Initial stabilization guard |
|---|---:|---:|
| HR | high after activity/posture | 1–3 min |
| RMSSD / short HRV | high | 2–5 min |
| 5-min HRV | very high | require stable context during most/all 5-min window |
| phasic EDA | event-sensitive | no blanket guard; align to event |
| tonic EDA | moderate/slow | 3–5 min after major thermal/activity transition |
| PPG amplitude/morphology | high after posture/thermal change | 1–3 min |
| skin temperature | slow thermal dynamics | 5–15+ min |
| SpO₂ | posture/sleep/altitude dependent | 1–3 min after major transition |
| derived respiratory rate | activity/recovery dependent | 1–3 min |

Rather than discard all transition data, the output may be:

```text
TRANSITION_ASSOCIATED
```

with lower confidence.

---

# 15. Event-aware interpretation

Physiology should often be interpreted relative to an event instead of arbitrary rolling windows.

For a known event at time \(t_e\):

```text
pre-event reference
event response
post-event recovery
```

## 15.1 Event template

```yaml
event:
  type: CONVERSATION_START
  time: t_e
  confidence: 0.90

physiology:
  pre_event:
    window: -120 s to -30 s
  response:
    window: 0 to +60 s
  recovery:
    window: +60 s to +300 s
```

The exact windows are feature-dependent and should be configurable.

### For HR

Track:
- pre-event median HR;
- response peak/mean;
- time to peak;
- recovery half-time.

### For phasic EDA

Track:
- event-linked response count;
- maximum phasic amplitude;
- integrated phasic activity;
- latency from event.

### For PPG vascular features

Track:
- pulse amplitude/morphology change;
- recovery to pre-event state.

### For skin temperature

Use longer windows due to thermal lag.

Event labels should not determine psychological meaning by themselves. An event called `CONVERSATION` does not imply stress.

---

# 16. Context-conditioned baselines

A single global baseline is inadequate for many features.

HRV differs by posture, sleep/wake, and time-of-day [8,16–18,23]. Wrist temperature has a circadian and sleep-related rhythm [14,15]. Tonic EDA can vary with time-of-day and temperature [11,33]. PPG morphology changes with posture and autonomic/vascular state [20,21].

The recommended baseline is therefore **personalized and context-conditioned**.

## 16.1 Do not create a baseline for every Cartesian combination

This would produce sparse cells such as:

```text
person × evening × sitting × social × work × indoors × conversation
```

Instead, use a hierarchical backoff model.

## 16.2 Baseline hierarchy

For physiological feature \(y\):

```text
Level 1: person + feature-specific exact context
Level 2: person + feature-specific macro-context
Level 3: person + wake state + posture/activity
Level 4: person + time-of-day
Level 5: person global
Level 6: population prior
```

Example for RMSSD:

```text
person + awake + sitting + focused_task + evening
        ↓ sparse
person + awake + sitting + focused_task
        ↓ sparse
person + awake + sitting
        ↓ sparse
person + awake
        ↓ sparse
person global
```

Example for phasic EDA:

```text
person + awake + event class + task/social state
        ↓ sparse
person + awake + event/no-event
        ↓ sparse
person awake baseline
```

## 16.3 Robust baseline statistics

Maintain:

- median;
- MAD;
- 10th/25th/75th/90th percentiles;
- sample count;
- number of distinct sessions/days;
- last update;
- slow EWMA mean/variance where appropriate.

Prefer robust statistics because free-living physiological distributions are non-Gaussian.

## 16.4 Context-relative z-score

Use a robust score:

\[
z_{ctx} =
\frac{x_t - \mathrm{median}(X_{person,ctx})}
     {1.4826 \cdot MAD(X_{person,ctx}) + \epsilon}
\]

Only calculate this when baseline support is adequate.

Alternatively use empirical percentiles:

```text
p_ctx = percentile of observed x within personal context distribution
```

Percentiles are often easier to explain and less sensitive to non-normality.

---

# 17. Context-conditioned expected physiology model

For features with sufficient data, a regression model is preferable to a large number of hard context cells.

Conceptually:

\[
E[y_t \mid person, context_t, history_t] =
f_i(context_t, history_t)
\]

A practical model is:

\[
y_t =
\mu_i
+ \beta_{wake}
+ \beta_{activity}
+ \beta_{posture}
+ \beta_{time}
+ \beta_{task}
+ \beta_{social}
+ \beta_{transition}
+ \epsilon_t
\]

where:
- \(\mu_i\) is the personal intercept;
- context terms are included only when relevant to that feature;
- interactions are added sparingly and only with evidence/data;
- uncertainty is estimated from the residual distribution.

## Selected implementation

**Version 1:** hierarchical robust baseline + backoff.  
**Version 2 after sufficient data:** regularized mixed-effects / generalized additive model per feature.

Why not a black-box deep model first:
- context is partially observed;
- interpretability matters;
- target data will initially be sparse;
- the output must explain *why* context changed interpretation;
- a simpler model provides better failure behavior.

---

# 18. Personalization

Personalization is required because ambulatory physiological responses vary substantially across people [1,7]. Personalized wearable models frequently outperform purely population-level models [34–36].

This subsystem should personalize:

- baseline level;
- within-context variability;
- response magnitude to common transitions/events;
- recovery time;
- time-of-day profile.

It should **not** personalize an AUD diagnosis here.

## 18.1 Baseline bootstrap

Suggested engineering minimums before a context-specific baseline is considered mature:

| Feature | Initial support target |
|---|---|
| HR / pulse rate | ≥20 min valid data across ≥3 separate sessions in that context |
| RMSSD / short HRV | ≥10 valid stable windows across ≥3 days |
| 5-min HRV | ≥8–10 valid 5-min windows across ≥3 days |
| tonic EDA | ≥20 stable 1-min windows across ≥3 sessions |
| phasic EDA | ≥20 neutral/event-free reference windows plus event data when event-relative modeling is used |
| PPG amplitude/morphology | ≥20 stable 30–60 s windows across ≥3 sessions |
| skin temperature | ≥2–3 days for coarse time-of-day pattern; preferably ≥7 days |
| SpO₂ | ≥20 stable windows per major wake/posture context if used |

These are **implementation defaults**, not universal standards. The system should expose baseline support so thresholds can be revised during validation.

## 18.2 Baseline adaptation

Update a baseline only when:

```text
input physiology quality = trusted
confound-aware status = accepted
context confidence = high enough
context stable = true
no exceptional event active
window not flagged unusual
```

Use slow adaptation so an acute physiological state does not redefine “normal.”

---

# 19. Context relevance filtering

Every feature must declare which context dimensions matter.

This prevents the system from feeding all available context into every interpretation.

## 19.1 Relevance matrix

Legend:

- **R** = required for high-confidence interpretation;
- **I** = important but interpretation can continue with lower confidence;
- **O** = optional/refining;
- **L** = usually low-value.

| Feature | Wake | Activity | Posture | Time | Task | Social | Event | Thermal/env | Transition |
|---|---|---|---|---|---|---|---|---|---|
| HR | R | R | I | I | I | I | I | O | R |
| RMSSD / HRV | R | I | R | R | R/I | I | I | O | R |
| HF-HRV / RSA-like | R | I | R | R | I | I | I | O | R + respiration context |
| tonic EDA | R | I | O | I | I | I | I | R if available | R |
| phasic EDA | R | O | O | O | I | I | R when event-based | O | I |
| PPG pulse rate | R | R | I | I | I | I | I | O | R |
| PPG amplitude/morphology | R | I | R | O/I | I | O/I | I | R | R |
| skin temperature | R | I | I | R | O | O | O/I | R if available | R |
| SpO₂ | R | I | R/I | I | L | L | O | altitude/thermal important | R |
| derived respiratory rate | R | R | I | I | I | O | I | O | R |

This matrix is the implementation’s `relevant_context_lookup`.

---

# 20. Per-modality context map

## 20.1 Heart rate

| Context variable | Why interpretation changes | Source | Evidence |
|---|---|---|---|
| wake/sleep | autonomic set-point differs across vigilance state | sleep/wake engine | [16,17] |
| activity | expected HR differs greatly by behavioral state and recovery | IMU | [4,5] |
| posture | orthostatic autonomic regulation changes HR | IMU/posture | [18] |
| cognitive task | task demand can alter cardiovascular activation | phone/user task | [9,10] |
| social interaction | social setting can modify HR responses | phone/user/EMA | [24–27] |
| time-of-day | circadian/autonomic rhythms change expected HR | clock | [16,23] |
| recent transition | postural/activity recovery changes expected HR | context history | [18] |

### Same HR change, different interpretation

```text
HR +12 bpm while:
A) stable seated passive rest
B) focused mental task
C) immediately after standing
D) during conversation
E) awakening
```

The physiological event is real in all cases. Its significance relative to expectation differs.

### Allowed output

```text
CARDIOVASCULAR_ACTIVATION_ELEVATED_FOR_CONTEXT
```

### Not allowed

```text
ANXIETY
CRAVING
STRESS_DISORDER
```

---

## 20.2 HRV

HRV is particularly context-sensitive. Methodological guidance emphasizes posture, respiration, measurement conditions, and standardization [8]. Sleep and circadian state strongly alter HRV [16,17]. Cognitive demand can reduce HRV [9,10].

| Context variable | Importance |
|---|---|
| wake/sleep | required |
| posture | required |
| stable-state duration | required |
| time-of-day | required for longitudinal comparison |
| cognitive/task context | important |
| respiration for HF/RSA interpretation | required or confidence-limiting |
| recent transition | required |
| social context | useful when interpreting interaction-linked changes |

### Respiratory caveat

HF-HRV / respiratory sinus arrhythmia can change because respiratory pattern changes even without equivalent change in tonic vagal activity [22]. Therefore:

```text
if respiratory pattern is unavailable:
    do not make strong vagal-specific claims from HF-HRV
```

RMSSD may still be used, but context should be documented.

---

## 20.3 Tonic EDA

Tonic EDA should be interpreted slowly and contextually.

Important context:

- wake/sleep;
- time-of-day;
- ambient/thermal state;
- prolonged task/social state;
- recent environment transition;
- personal baseline.

Ambient temperature significantly shifts tonic EDA [11]. Time-of-day effects have also been reported [33].

### Allowed output

```text
TONIC_ELECTRODERMAL_LEVEL_HIGH_FOR_CONTEXT
```

If thermal context is unavailable:

```text
TONIC_ELECTRODERMAL_LEVEL_ELEVATED
interpretation_confidence = MEDIUM
note = "ambient thermal context unavailable"
```

Do not infer a psychological state directly.

---

## 20.4 Phasic EDA

Phasic EDA reflects sympathetic sudomotor activation but is psychologically non-specific. It can occur with novelty, attention, emotionally salient stimuli, social events, cognitive demands, and other arousing events [2,3,12].

Important context:

- known event;
- task;
- social interaction;
- wake state;
- recent transition;
- event timing.

Thermal context is less dominant for phasic EDA than for tonic EDA in Qasim et al. [11], though environmental information remains useful.

### Preferred interpretation style

```text
ELECTRODERMAL_ACTIVATION_INCREASED
event_association = PRESENT / ABSENT / UNKNOWN
```

If event exists:

```text
"phasic electrodermal activation increased within 5 s of event onset"
```

The system must stop there.

---

## 20.5 PPG pulse rate / PRV

Pulse rate is contextually similar to ECG HR.

If pulse-rate variability is used as a surrogate for HRV, the system should preserve the distinction:

```text
PRV ≠ ECG HRV under all conditions
```

Context required:
- wake/sleep;
- activity;
- posture;
- time;
- transition.

Do not silently apply ECG-derived HRV reference expectations to PPG-derived PRV without validation.

---

## 20.6 PPG amplitude and morphology

This is one of the most context-dependent PPG outputs.

Posture produces changes in pulse amplitude and morphology [20,21]. Mental stress can alter PPG waveform features [37]. Most importantly, the direction of cutaneous vascular response to the same arousing/mental stimulus can depend on thermoregulatory state [19].

Therefore required context includes:

- posture;
- thermal state if available;
- body/site mounting;
- activity;
- recent posture transition;
- task/event.

### Allowed output

```text
PERIPHERAL_VASCULAR_TONE_SHIFT_UNUSUAL_FOR_CONTEXT
```

or

```text
PPG_PULSE_AMPLITUDE_LOWER_THAN_EXPECTED_FOR_CONTEXT
```

Do not automatically label it sympathetic stress.

---

## 20.7 Skin temperature

Peripheral skin temperature is not core body temperature. It reflects:

- local blood flow;
- thermoregulation;
- circadian phase;
- sleep/wake;
- environment;
- measurement site;
- behavioral state.

Wrist temperature has a strong daily rhythm, rises around bed time, and drops after awakening [14]. Multisensor ambulatory monitoring can estimate circadian phase from skin temperature, activity, posture, and light [15]. Psychosocial stress can alter distal skin temperature, but effects depend on measurement site [13].

Important context:

- sensor site;
- time-of-day;
- wake/sleep;
- activity;
- posture;
- environment/ambient temperature if separately sensed;
- transition history.

### Allowed output

```text
PERIPHERAL_SKIN_TEMPERATURE_LOWER_THAN_EXPECTED_FOR_TIME_AND_STATE
```

Not:

```text
STRESS
FEVER
INTOXICATION
```

unless a downstream validated layer explicitly makes such an inference.

---

## 20.8 Optional SpO₂

Only include SpO₂ if the upstream PPG subsystem provides a validated, calibrated, high-quality value.

Context that matters:

- altitude;
- sleep/wake;
- posture;
- activity;
- peripheral thermal/perfusion state;
- recent transition.

Normal saturation itself shifts at altitude, and measurement becomes harder in cold/perfusion-limited settings [38]. Body position can produce small but measurable SpO₂ differences [39].

### Output

```text
OXYGEN_SATURATION_LOWER_THAN_PERSONAL_CONTEXT_EXPECTATION
```

If altitude is unknown:

```text
context_sufficiency = PARTIAL
```

---

## 20.9 Optional derived respiratory rate

If respiratory rate is inferred from ECG/PPG and already validated upstream, interpretation must be conditioned on activity. A context-conditioned classification-regression approach has been demonstrated for wearable respiration inference, where IMU-derived activity determined which regression relationship to use [40].

Relevant context:

- wake/sleep;
- activity;
- posture;
- recent activity transition;
- task;
- time-of-day.

### Output

```text
RESPIRATORY_RATE_ELEVATED_FOR_CURRENT_ACTIVITY_STATE
```

not a diagnosis.

---

# 21. Same physiology, different meaning: required ambiguity examples

## HR elevation

```text
same approximate HR increase
├─ seated quiet rest
├─ sustained focused task
├─ conversation
├─ postural transition
├─ wake transition
└─ post-activity recovery
```

Interpretation changes from “unexpected cardiovascular activation” to “expected task/transition-associated cardiovascular response.”

## HRV reduction

```text
same RMSSD reduction
├─ stable passive rest
├─ focused cognitive work
├─ standing
├─ early post-wake period
├─ negative social interaction
└─ changed breathing pattern
```

The system should never apply one fixed threshold across these contexts.

## EDA response

```text
similar phasic EDA increase
├─ salient notification/event
├─ conversation onset
├─ cognitive task
├─ emotionally salient stimulus
├─ novelty
└─ unexplained rest
```

Only the last case might be described as an unexplained context-relative autonomic event, and even then not as a specific emotion.

## PPG amplitude decrease

```text
similar pulse-amplitude decrease
├─ posture change
├─ warm thermal state + arousing event
├─ cold thermal state
├─ mental task
└─ stable rest
```

Thermal context can alter cutaneous vasomotor response direction [19].

## Skin-temperature decrease

```text
similar distal temperature decrease
├─ awakening
├─ colder environment
├─ peripheral vasoconstriction
├─ activity transition
└─ psychosocial challenge
```

Without contextual information the physiological meaning remains broad.

---

# 22. Context fusion

Do not feed every source into one opaque classifier.

Use a **dimension-wise evidence fusion** design.

## 22.1 Source reliability

For each context dimension \(d\), source \(s\) yields:

\[
P_s(C_d = c)
\]

Combine sources with calibrated weights and freshness:

\[
\log P(C_d=c \mid sources)
\propto
\log P_0(c)
+
\sum_s w_s \cdot g(age_s)\cdot\log P_s(c)
\]

where:
- \(w_s\) = source reliability;
- \(g(age_s)\) = staleness decay;
- \(P_0\) = prior from recent state/time.

Normalize probabilities after fusion.

## 22.2 Precedence rules

Use source-specific precedence rather than one global priority.

Example:

**Activity**
```text
IMU direct classifier > phone activity classifier > user stale label
```

**Phone call**
```text
OS call-state event > user label > heuristic
```

**Emotionally meaningful event**
```text
user event label > everything else
```

**Location category**
```text
phone semantic location > calendar assumption
```

Explicit user labels are not automatically superior for sensor-observable facts if they are stale or contradictory.

## 22.3 Conflict handling

If sources disagree:

```yaml
value: UNKNOWN_OR_MIXED
posterior:
  SITTING: 0.48
  STANDING: 0.45
confidence: 0.48
conflict: true
```

Do not collapse this to a false certainty.

---

# 23. Context confidence and sufficiency

Two different concepts are needed.

## 23.1 Per-dimension context confidence

Example:

```yaml
wake_confidence: 0.98
activity_confidence: 0.93
posture_confidence: 0.79
social_confidence: null
```

## 23.2 Feature-specific context sufficiency

Each physiological feature \(f\) has a required set \(R_f\).

For example:

```text
R_HRV = {wake, posture, stable_duration, time_context}
```

A practical sufficiency score:

\[
S_f =
\frac{\sum_{d\in R_f} w_{f,d} \cdot q_d}
{\sum_{d\in R_f} w_{f,d}}
\]

where:
- \(q_d=0\) when missing;
- \(q_d\in[0,1]\) is confidence otherwise.

Then classify:

```text
S_f >= 0.85 → CONTEXT_SUFFICIENT
0.55 <= S_f < 0.85 → CONTEXT_PARTIAL
S_f < 0.55 → CONTEXT_INSUFFICIENT
```

The thresholds are engineering defaults and must be calibrated.

## 23.3 Interpretation confidence

Combine:

\[
C_{interp} =
C_{physiology}
\times
C_{context}
\times
C_{baseline}
\times
C_{model}
\]

The upstream layer may already guarantee high \(C_{physiology}\), but retaining its score helps downstream calibration.

---

# 24. Missing context behavior

This is a hard requirement.

## Known context

```text
"HR is elevated relative to the person's stable seated-evening baseline."
```

## Partially known context

```text
"HR is elevated relative to seated-awake baseline.
Task/social context is unavailable, so cause-level interpretation is limited."
```

## Ambiguous context

```text
"Real cardiovascular activation detected during an unresolved posture transition.
Context-relative abnormality cannot be determined confidently."
```

## Unknown context

```text
"Cardiovascular activation detected.
Context insufficient for comparison with an appropriate expected state."
```

Never replace missing context with the most likely psychological label.

---

# 25. Interpretation state vocabulary

Recommended context-relative categories:

```text
EXPECTED_FOR_CONTEXT
ELEVATED_FOR_CONTEXT
SUPPRESSED_FOR_CONTEXT
UNUSUAL_FOR_CONTEXT
EVENT_ASSOCIATED
TRANSITION_ASSOCIATED
RECOVERY_DELAYED_FOR_CONTEXT
AMBIGUOUS_IN_CONTEXT
INSUFFICIENT_CONTEXT
INSUFFICIENT_BASELINE
```

“Abnormal” should be reserved for a validated clinical layer. `UNUSUAL_FOR_CONTEXT` is safer and more precise here.

---

# 26. Mathematical formulation

Let:

- \(Y_t\): trusted physiological feature/event;
- \(C_t\): factorized current context;
- \(H_t\): recent context/event history;
- \(i\): individual;
- \(M_f\): feature-specific expected physiology model.

The system estimates:

\[
p(Y_t \mid C_t, H_t, i)
\]

or:

\[
\hat{Y}_t = E[Y_t \mid C_t, H_t, i]
\]

Then:

\[
D_t = Y_t - \hat{Y}_t
\]

and a normalized deviation:

\[
Z_t =
\frac{Y_t-\hat{Y}_t}
{\hat{\sigma}(Y\mid C_t,H_t,i)+\epsilon}
\]

For empirical robust baselines:

\[
Z^{robust}_t =
\frac{Y_t - median(Y_{i,C})}
{1.4826\cdot MAD(Y_{i,C})+\epsilon}
\]

The layer does **not** estimate:

\[
P(\text{AUD clinical state} \mid Y,C)
\]

That belongs downstream.

---

# 27. Feature-specific baseline selection

Pseudo-interface:

```python
def choose_baseline(feature, context, person):
    relevant = RELEVANT_CONTEXT[feature]

    context_key = project(context, relevant)

    candidates = [
        exact_person_context(context_key),
        person_macro_context(backoff(context_key)),
        person_wake_posture(context),
        person_time_block(context),
        person_global(),
        population_prior()
    ]

    for b in candidates:
        if b.support >= required_support(feature) and b.confidence >= threshold:
            return b

    return None
```

Baseline selection must record which level was used.

```yaml
baseline:
  level: PERSON_WAKE_POSTURE
  support_windows: 37
  support_days: 5
  confidence: 0.86
```

A result based on a global fallback should receive lower interpretation confidence than a mature exact-context baseline.

---

# 28. Per-modality algorithms

## 28.1 Heart rate algorithm

```text
INPUT:
  trusted HR feature/event
  wake state
  activity state
  posture
  time context
  recent transitions
  optional task/social/event context
  personal baseline models

1. Reject no data here; upstream already decided physiological validity.
2. Obtain relevant context.
3. If recent vigorous activity/posture transition:
      mark TRANSITION_ASSOCIATED
      use transition/recovery model rather than rest baseline.
4. Otherwise retrieve person-context HR expectation.
5. Compare observed HR with expected distribution.
6. If task/social context is known, annotate interpretation.
7. Emit:
      EXPECTED / ELEVATED / SUPPRESSED / UNUSUAL / AMBIGUOUS.
8. Report context sufficiency and baseline level.
```

Output example:

```yaml
generic_meaning: CARDIOVASCULAR_ACTIVATION
context_relative_state: ELEVATED_FOR_CONTEXT
confidence: 0.82
notes:
  - "stable sitting for 7 min"
  - "evening personal sitting baseline used"
  - "social context unavailable"
```

---

## 28.2 HRV algorithm

```text
INPUT:
  validated HRV feature
  HRV window duration
  wake/sleep
  posture
  context stability over entire HRV window
  time-of-day
  task state
  respiration context when required
  personal context HRV baseline

1. Require stable context over most/all analysis window.
2. If posture or wake state changes inside window:
      state = AMBIGUOUS_IN_CONTEXT
      do not compare with stable baseline.
3. If HF-HRV/RSA-like metric and respiration unavailable:
      reduce physiological-specificity confidence.
4. Retrieve context-specific HRV expectation.
5. Compute robust percentile/z-score.
6. Emit generic cardiac-autonomic interpretation only.
```

Example:

```text
"CARDIAC_VAGAL_VARIABILITY_LOWER_THAN_EXPECTED_FOR_STABLE_SEATED_CONTEXT"
```

If respiration is unknown:

```text
"HRV lower than expected; vagal-specific interpretation limited because respiratory pattern is unavailable."
```

---

## 28.3 Tonic EDA algorithm

```text
INPUT:
  tonic EDA level/trend
  wake
  time-of-day
  stable task/social state
  ambient thermal context if available
  transition history
  personal tonic EDA baseline

1. Detect major recent thermal/environment transition.
2. Select time/wake-conditioned personal baseline.
3. If ambient thermal state known:
      include thermal term.
   Else:
      mark thermal_context_missing = true.
4. Compare tonic level/trend with expected distribution.
5. Emit:
      TONIC_ELECTRODERMAL_LEVEL_EXPECTED/ELEVATED/SUPPRESSED.
6. Never emit a psychological cause.
```

---

## 28.4 Phasic EDA algorithm

```text
INPUT:
  phasic EDA event(s)
  event/task/social context
  wake
  recent transition
  personal event-free phasic baseline

1. Determine whether a known event occurred in an event-association window.
2. Quantify phasic magnitude/event density relative to context baseline.
3. If event is known:
      emit EVENT_ASSOCIATED electrodermal activation.
4. If no event known but context sufficient:
      emit UNEXPLAINED electrodermal activation relative to context.
5. If task/social context missing:
      emit generic activation and lower interpretation confidence.
```

---

## 28.5 PPG amplitude/morphology algorithm

```text
INPUT:
  trusted pulse amplitude/morphology feature
  posture
  thermal state
  wake
  activity
  task/event
  transition history
  personal PPG context baseline

1. Require known posture for high-confidence amplitude interpretation.
2. Check recent posture transition.
3. Use thermal-state-conditioned baseline when available.
4. Compare feature with expected context distribution.
5. If thermal state missing:
      lower confidence for vascular interpretation.
6. Emit:
      PERIPHERAL_VASCULAR_CHANGE + context-relative state.
```

---

## 28.6 Skin-temperature algorithm

```text
INPUT:
  trusted skin temperature/trend
  sensor site
  wake/sleep
  time-of-day
  activity/posture
  environment if available
  transition history
  personal circadian/context baseline

1. Never compare raw temperature with a single all-day threshold.
2. Retrieve personal time-of-day + wake baseline.
3. Require stable wear/site metadata.
4. If environment changed recently:
      mark TRANSITION_ASSOCIATED.
5. Compare temperature and trend with expected trajectory.
6. Emit:
      PERIPHERAL_TEMPERATURE_EXPECTED/ELEVATED/LOWER/UNUSUAL.
```

---

## 28.7 SpO₂ algorithm, if enabled

```text
INPUT:
  validated SpO2
  wake/sleep
  posture
  altitude
  peripheral perfusion/thermal context
  activity
  personal baseline

1. Require upstream high-quality oximetry.
2. Select altitude- and state-appropriate expectation.
3. If altitude unknown and value differs modestly:
      mark context partial.
4. Compare against personal stable baseline.
5. Emit generic saturation deviation only.
```

---

## 28.8 Derived respiratory-rate algorithm, if enabled

```text
INPUT:
  trusted respiratory rate
  wake/sleep
  activity
  posture
  recent transition
  task
  personal activity-conditioned baseline

1. Identify activity context.
2. Select activity-conditioned respiratory model.
3. If post-activity transition, use recovery model.
4. Compare observed rate with expected.
5. Emit:
      RESPIRATORY_RATE_ELEVATED/LOWER/EXPECTED_FOR_CONTEXT.
```

This structure follows the context-conditioned modeling principle demonstrated by Alam et al. [40].

---

# 29. Context transitions

A transition is a first-class state.

Examples:

```text
SLEEP → AWAKE
LYING → STANDING
STILL → WALKING
WALKING → STILL
ALONE → INTERACTION
HOME → COMMUTE
PASSIVE → FOCUSED_TASK
```

## 29.1 Transition representation

```yaml
transition:
  dimension: posture
  from: LYING
  to: STANDING
  t0: ...
  confidence: 0.96
  elapsed_s: 32
  expected_stabilization_s: 120
```

## 29.2 Interpretation rules

During a transition:

- do not use the stable destination baseline immediately;
- compare against a transition/recovery trajectory when enough data exists;
- otherwise emit `TRANSITION_ASSOCIATED`;
- reduce baseline confidence.

## 29.3 Learning transition models

After sufficient personal data, estimate:

```text
expected response(t | transition_type, person)
```

For example:

```text
expected HR recovery after WALKING → STILL
```

This is more informative than treating the first minute of rest as ordinary rest.

---

# 30. Circadian/time-of-day conditioning

Time-of-day is not merely a nuisance variable. It changes expected physiology.

Examples:

- HRV has circadian rhythms [16,23].
- wrist temperature has robust daily structure [14,15].
- tonic EDA has reported diurnal variation [33].

## Recommended implementation

Use:

```text
time_since_wake
sin(2π * clock_time / 24h)
cos(2π * clock_time / 24h)
```

rather than only four coarse bins when enough data exists.

For sparse baseline storage, use:

```text
MORNING
DAY
EVENING
NIGHT
```

as the fallback.

Once at least several days of data are available, fit a smooth personal time-of-day effect:

\[
y(t) = \beta_0 + s_{24h}(t) + context\ terms
\]

where \(s_{24h}\) is a cyclic spline or harmonic term.

---

# 31. Real-time implementation

## 31.1 Processing rates

Recommended:

| Module | Update |
|---|---|
| IMU activity/posture | 1 Hz |
| context temporal filter | 1 Hz |
| sleep/wake | 30–60 s |
| phone events | event-driven |
| location category | on significant transition / low frequency |
| context state snapshot | 1 Hz |
| physiological interpretation | whenever upstream feature/event updates |
| baseline statistics | background incremental update after eligible windows |

## 31.2 Causality

Online interpretation should be causal.

Allowed online:
- past/current context;
- forward filtering;
- rolling baselines;
- event history.

Optional retrospective analysis may use:
- bidirectional smoothing;
- full-day context segmentation;
- re-estimated sleep/wake;
- final corrected event boundaries.

Do not use future data in a result presented as real-time.

## 31.3 State retention

In memory:

```text
last 60 min high-resolution context
current context posteriors
active transitions
last 24 h event timestamps
baseline sufficient statistics
```

Persistent:

```text
personal baseline summaries
context model calibration
circadian profile
transition response summaries
source reliability calibration
```

Raw smartphone context should follow privacy and retention policies independent of this physiological design.

## 31.4 Latency

The context layer itself should add little computational latency:

- tree inference: milliseconds;
- dimension-wise fusion: milliseconds;
- baseline lookup: O(1) / hash-map lookup;
- output generation: milliseconds.

The dominant latency is dictated by the physiological feature window (for example, a 5-min HRV estimate), not by context interpretation.

---

# 32. Recommended output schema

```yaml
context_aware_interpretation:
  timestamp: ...

  physiology:
    modality: ECG
    feature: RMSSD
    value: 24.8
    units: ms

  generic_physiology:
    domain: CARDIAC_AUTONOMIC_VARIABILITY
    direction: LOWER

  context:
    wake:
      value: AWAKE
      confidence: 0.98
    activity:
      value: STILL
      confidence: 0.95
    posture:
      value: SITTING
      confidence: 0.89
    task:
      value: FOCUSED_TASK
      confidence: 0.72
    social:
      value: UNKNOWN
      confidence: null
    day_phase: EVENING
    transition_active: false

  context_sufficiency:
    state: CONTEXT_PARTIAL
    score: 0.81
    missing:
      - social
      - respiration

  baseline:
    type: PERSONAL_CONTEXT_CONDITIONED
    level: PERSON_AWAKE_SITTING_TASK
    support_windows: 18
    support_days: 4
    expected_median: 36.2
    expected_mad: 7.1
    confidence: 0.83

  deviation:
    robust_z: -1.09
    percentile: 14

  context_relative_state: LOWER_THAN_EXPECTED_FOR_CONTEXT

  interpretation:
    code: CARDIAC_VARIABILITY_LOWER_FOR_CONTEXT
    text: >
      Cardiac variability is lower than this person's expected range
      for a stable seated evening task context. Respiration is unavailable,
      so vagal-specific interpretation is limited.

  interpretation_confidence: 0.67

  downstream_eligibility:
    allowed: true
    caveats:
      - RESPIRATION_CONTEXT_MISSING
```

The downstream AUD-specific layer can consume this object without the context-aware layer itself making an AUD claim.

---

# 33. Downstream eligibility

A context-aware result should include whether it is suitable for higher-level interpretation.

Recommended:

```text
ELIGIBLE
ELIGIBLE_WITH_CAVEATS
NOT_ELIGIBLE_CONTEXT_INSUFFICIENT
NOT_ELIGIBLE_BASELINE_INSUFFICIENT
```

Examples:

```text
trusted EDA + known wake + known event + mature baseline
→ ELIGIBLE
```

```text
trusted HRV + unknown posture + posture changed within window
→ NOT_ELIGIBLE_CONTEXT_INSUFFICIENT
```

This is not signal rejection; it is interpretation gating.

---

# 34. Context-specific interpretation versus signal validity

The context layer should **never** silently convert:

```text
real physiology + unexpected context
```

into:

```text
invalid physiology
```

Instead:

```text
physiology_valid = true
context_relative_state = AMBIGUOUS
```

This preserves the pipeline boundary.

---

# 35. Final per-modality synthesis

| Modality / feature | Required/high-value context | Primary context source | Context model | Baseline | Generic output |
|---|---|---|---|---|---|
| HR | wake, activity, posture, transition, time; task/social helpful | IMU + clock + phone/user | factorized state + transition model | personal context baseline | cardiovascular activation relative to context |
| HRV/RMSSD | wake, posture, stable window, time; task; respiration caveat | IMU + sleep engine + clock | stable-state context + time model | person/context HRV distribution | cardiac variability higher/lower than expected |
| HF-HRV/RSA | above + respiration | IMU + derived/direct respiration | context + respiration-conditioned | person/context | vagal-related variability only with respiratory caveat satisfied |
| tonic EDA | wake, time, thermal context, long task/social state | sleep + clock + ambient sensor/phone | slow context state | time/thermal conditioned personal baseline | tonic electrodermal level relative to context |
| phasic EDA | event, task/social, wake | event/user/phone + clock | event-aware state | event-free and event-class baseline | electrodermal activation, event-associated or unexplained |
| PPG pulse rate | similar to HR | IMU + sleep + clock | HR-like context model | personal context | pulse/cardiovascular activation |
| PPG amplitude/morphology | posture, thermal state, transition, task/event | IMU + ambient context | context-conditioned vascular model | site/context personal baseline | peripheral vascular change relative to context |
| skin temperature | wake, time, site, environment, transition | clock + sleep + optional ambient sensor | circadian + state model | personal time-of-day/state curve | peripheral temperature deviation |
| SpO₂ (optional) | wake, posture, altitude, perfusion/thermal | sleep + IMU + phone/barometer | state + environment | context distribution | saturation deviation |
| respiratory rate (optional) | wake, activity, transition, posture | IMU + sleep | activity-conditioned model | activity-specific baseline | respiratory rate relative to context |

---

# 36. What context should **not** be collected by default

To avoid an over-complex and privacy-heavy subsystem, do not collect by default:

- exact continuous GPS if semantic location is enough;
- raw microphone audio;
- message content;
- phone contact identities;
- calendar text/content;
- every Bluetooth device identity;
- detailed environmental sensing unrelated to a feature;
- psychological labels inferred from the physiology being interpreted.

The test for adding a context signal is:

> **Does this context materially change the expected physiology, the appropriate baseline, or interpretation confidence for at least one target feature?**

If not, exclude it.

---

# 37. Validation plan

The context-aware layer should be validated independently from the later AUD interpretation layer.

## 37.1 Context inference validation

Measure:

- activity confusion matrix;
- posture confusion matrix;
- sleep/wake sensitivity/specificity;
- event timing error;
- calibration of posterior probabilities;
- missing/unknown behavior.

## 37.2 Context-conditioned baseline validation

For each feature compare:

```text
global personal baseline
vs
context-conditioned personal baseline
```

Metrics:

- residual MAE;
- residual variance;
- false “unusual” rate;
- calibration of prediction intervals.

A useful result is:

```text
Var(y - E[y|context]) < Var(y - E[y])
```

## 37.3 Interpretation calibration

If the system says:

```text
90% expected interval
```

approximately 90% of stable, representative observations should fall within it.

## 37.4 Ablation tests

Remove one context dimension at a time:

```text
without wake
without posture
without time
without task
without social
without thermal context
```

Measure whether context-relative error increases.

This produces direct evidence for which dimensions are worth retaining.

---

# 38. Practical implementation interfaces

## 38.1 Upstream input

```yaml
physio_event:
  timestamp_start: ...
  timestamp_end: ...
  modality: EDA
  feature: PHASIC_RESPONSE_RATE
  value: ...
  units: ...
  physiology_confidence: 0.97
  confound_status: CLEARED
  metadata:
    window_s: 30
```

## 38.2 Context engine API

```python
get_context(
    start_time,
    end_time,
    dimensions=[...]
) -> ContextWindow
```

The context result should include:
- posterior over states;
- fraction of window stable;
- transitions;
- events;
- freshness.

## 38.3 Baseline engine API

```python
get_expected_physiology(
    person_id,
    feature,
    context_window
) -> ExpectedDistribution
```

## 38.4 Interpretation API

```python
interpret(
    physio_event,
    context_window,
    expected_distribution
) -> ContextAwareInterpretation
```

---

# 39. Pseudocode for the whole subsystem

```python
def context_aware_interpret(physio_event):

    feature = physio_event.feature

    # 1. Determine only context that matters for this feature.
    required, optional = CONTEXT_SPEC[feature]

    # 2. Query context across the physiological event window.
    ctx = context_engine.get_window(
        physio_event.timestamp_start,
        physio_event.timestamp_end,
        dimensions=required | optional
    )

    # 3. Calculate feature-specific context sufficiency.
    ctx_score, ctx_state = context_sufficiency(feature, ctx)

    # 4. Detect event or transition regime.
    regime = determine_regime(feature, ctx)

    # 5. Choose the expected physiology model.
    expectation = baseline_engine.get_expectation(
        feature=feature,
        context=ctx,
        regime=regime
    )

    # 6. If context cannot support a meaningful comparison,
    #    preserve the physiological fact but reduce interpretation.
    if ctx_state == "CONTEXT_INSUFFICIENT":
        return emit_generic_only(
            physio_event,
            ctx,
            reason="required context missing"
        )

    # 7. If the appropriate baseline is unavailable, back off.
    if expectation is None:
        return emit_generic_only(
            physio_event,
            ctx,
            reason="context baseline insufficient"
        )

    # 8. Compare observed physiology with expectation.
    deviation = compare(
        observed=physio_event.value,
        expected=expectation
    )

    # 9. Generate a generic physiological interpretation.
    state = map_deviation_to_context_state(
        feature,
        deviation,
        regime
    )

    # 10. Propagate uncertainty.
    confidence = combine_confidence(
        physiology=physio_event.physiology_confidence,
        context=ctx_score,
        baseline=expectation.confidence,
        model=expectation.model_confidence
    )

    # 11. Emit no AUD-specific or diagnostic meaning.
    return ContextAwareInterpretation(
        physiology=physio_event,
        context=ctx,
        baseline=expectation,
        deviation=deviation,
        context_relative_state=state,
        interpretation_confidence=confidence,
        generic_text=generic_physiology_text(feature, state, ctx)
    )
```

---

# 40. Single recommended architecture

```text
                         ┌─────────────────────────────┐
                         │ confound-aware physiology  │
                         │ HR / HRV / EDA / PPG / ST │
                         └──────────────┬──────────────┘
                                        │
                                        v
                         ┌─────────────────────────────┐
                         │ feature-context requirement │
                         │          lookup             │
                         └──────────────┬──────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              v                         v                         v
     ┌────────────────┐       ┌────────────────┐        ┌────────────────┐
     │ IMU context    │       │ temporal state │        │ optional phone │
     │ activity       │       │ time/sleep     │        │ task/social/   │
     │ posture        │       │ transitions    │        │ coarse place   │
     └───────┬────────┘       └───────┬────────┘        └───────┬────────┘
             │                        │                         │
             └──────────────┬─────────┴─────────────┬───────────┘
                            v                       v
                    ┌────────────────────────────────────┐
                    │ factorized context state           │
                    │ probabilities + provenance + age   │
                    └─────────────────┬──────────────────┘
                                      │
                                      v
                    ┌────────────────────────────────────┐
                    │ event/transition regime selector   │
                    └─────────────────┬──────────────────┘
                                      │
                                      v
                    ┌────────────────────────────────────┐
                    │ personalized expected physiology   │
                    │ exact context → hierarchical       │
                    │ fallback → population prior        │
                    └─────────────────┬──────────────────┘
                                      │
                                      v
                    ┌────────────────────────────────────┐
                    │ context-relative deviation         │
                    │ percentile / robust z / interval   │
                    └─────────────────┬──────────────────┘
                                      │
                                      v
                    ┌────────────────────────────────────┐
                    │ uncertainty propagation            │
                    └─────────────────┬──────────────────┘
                                      │
                                      v
                    ┌────────────────────────────────────┐
                    │ generic context-aware physiology   │
                    │ NOT AUD-specific meaning           │
                    └─────────────────┬──────────────────┘
                                      │
                                      v
                    ┌────────────────────────────────────┐
                    │ AUD-specific downstream layer      │
                    └────────────────────────────────────┘
```

---

# Recommended Implementation

Build **one factorized, temporal Context Engine** shared by all physiological modalities, but do **not** build one universal physiological classifier.

## A. Context variables to collect now

Always collect:

1. **activity state** from IMU;
2. **posture** from IMU where mounting supports it;
3. **activity/posture transitions** and time since transition;
4. **wake/sleep** with actigraphy-compatible logic;
5. **clock time**, time since wake, and day phase;
6. **event timestamps**;
7. **device/wear state**.

These variables have broad utility and low privacy cost.

## B. Context variables to add through smartphone integration

Recommended, but optional:

1. coarse semantic location;
2. screen/task state;
3. active phone call;
4. environment transitions through Wi-Fi/Bluetooth;
5. optional conversation-present flag.

Do not collect content.

## C. Context variables requiring user input

Use sparse/event-driven input for:

1. meaningful event;
2. cognitively demanding task when not otherwise detectable;
3. social interaction type;
4. unusual situation.

## D. Context representation

Use independent dimensions with probability/confidence, provenance, freshness, and transition history.

Never force an unknown dimension to a value.

## E. Context inference

Use:

- IMU tree classifier + temporal filter for activity/posture;
- validated actigraphy-style sleep/wake if sensor position allows;
- event-driven phone context;
- direct user labels for subjective/situational events;
- feature-specific context-sufficiency scoring.

Do not build a giant end-to-end black-box context classifier.

## F. Context history

Retain:

- 60 min high-resolution context online;
- 24 h event history;
- persistent context-conditioned baseline summaries.

## G. Baselines

Use personalized hierarchical robust baselines:

```text
exact personal context
→ personal macro-context
→ personal wake/posture/time
→ personal global
→ population prior
```

Move to a regularized mixed-effects/GAM expectation model only after enough longitudinal data exist.

## H. Per-feature requirements

### Heart rate

Required:
- wake;
- activity;
- posture or transition state.

Strongly useful:
- time;
- task/social/event.

Without required context:
- emit `CARDIOVASCULAR_ACTIVATION_DETECTED`;
- do not classify as context-elevated.

### HRV

Required:
- wake;
- posture;
- stable context over the HRV window;
- time-of-day.

For HF-HRV/RSA:
- respiration is required for strong vagal-specific interpretation.

Without required context:
- emit `HRV_CHANGE_DETECTED_CONTEXT_INSUFFICIENT`.

### Tonic EDA

Required:
- wake;
- time;
- thermal context if available for high-confidence longitudinal comparison.

Without ambient thermal context:
- continue at lower confidence.

### Phasic EDA

Required for event-specific interpretation:
- event timing.

Useful:
- task/social context.

Without event/social/task context:
- emit generic electrodermal activation only.

### PPG amplitude/morphology

Required:
- posture;
- recent transition.

Strongly useful:
- thermal state;
- task/event.

Without posture:
- avoid vascular-context claims.

### Skin temperature

Required:
- sensor site;
- time-of-day;
- wake/sleep.

Strongly useful:
- environment;
- activity transition.

Without environment:
- retain context-relative trend interpretation, but lower causal specificity.

### SpO₂, if enabled

Required:
- trusted calibrated upstream SpO₂;
- wake/posture.

Strongly useful:
- altitude;
- perfusion/thermal state.

If altitude unknown:
- avoid strong significance from small deviations.

### Derived respiratory rate, if enabled

Required:
- wake;
- activity;
- transition state.

Use activity-conditioned expected values.

## I. Missing context

The final behavior is:

```text
good physiology + good context
→ context-relative physiological interpretation

good physiology + partial context
→ broader physiological interpretation + caveat

good physiology + insufficient context
→ physiological change reported, meaning unresolved
```

Never fabricate context.

## J. Output

The layer should emit:

```text
physiological event
+ factorized context
+ context sufficiency
+ baseline provenance
+ context-relative deviation
+ generic physiological meaning
+ interpretation confidence
+ downstream eligibility
```

This gives the next AUD-specific layer a clean input without contaminating it with unsupported psychological or clinical assumptions.

---

# References

[1] Norman GJ, Morgan E, Raja S, Berntson GG.  
“Wearable ANS monitoring in real life: A critical review of context-sensitive interpretation and implications for psychophysiology.”  
*Autonomic Neuroscience*, 2026;263:103364.  
DOI: 10.1016/j.autneu.2025.103364  
URL: https://pubmed.ncbi.nlm.nih.gov/41506124/

[2] Siegel EH, Sands MK, Van den Noortgate W, Condon P, Chang Y, Dy J, Quigley KS, Barrett LF.  
“Emotion fingerprints or emotion populations? A meta-analytic investigation of autonomic features of emotion categories.”  
*Psychological Bulletin*, 2018;144(4):343–393.  
DOI: 10.1037/bul0000128  
URL: https://pubmed.ncbi.nlm.nih.gov/29389177/

[3] Kreibig SD.  
“Autonomic nervous system activity in emotion: a review.”  
*Biological Psychology*, 2010;84(3):394–421.  
DOI: 10.1016/j.biopsycho.2010.03.010  
URL: https://pubmed.ncbi.nlm.nih.gov/20371374/

[4] Gjoreski M, Luštrek M, Gams M, Gjoreski H.  
“Monitoring stress with a wrist device using context.”  
*Journal of Biomedical Informatics*, 2017;73:159–170.  
DOI: 10.1016/j.jbi.2017.08.006  
URL: https://pubmed.ncbi.nlm.nih.gov/28803947/

[5] Hovsepian K, al’Absi M, Ertin E, Kamarck T, Nakajima M, Kumar S.  
“cStress: Towards a Gold Standard for Continuous Stress Assessment in the Mobile Environment.”  
*Proceedings of the ACM International Joint Conference on Pervasive and Ubiquitous Computing (UbiComp)*, 2015:493–504.  
DOI: 10.1145/2750858.2807526  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC4631393/

[6] Hoemann K, Khan Z, Feldman MJ, Nielson C, Devlin M, Dy J, Barrett LF, Wormwood JB, Quigley KS.  
“Context-aware experience sampling reveals the scale of variation in affective experience.”  
*Scientific Reports*, 2020;10:12459.  
DOI: 10.1038/s41598-020-69180-y  
URL: https://pubmed.ncbi.nlm.nih.gov/32719368/

[7] Smets E, Rios Velazquez E, Schiavone G, Chakroun I, D’Hondt E, De Raedt W, Cornelis J, Janssens O, Van Hoecke S, Claes S, Van Diest I, Van Hoof C.  
“Large-scale wearable data reveal digital phenotypes for daily-life stress detection.”  
*npj Digital Medicine*, 2018;1:67.  
DOI: 10.1038/s41746-018-0074-9  
URL: https://pubmed.ncbi.nlm.nih.gov/31304344/

[8] Laborde S, Mosley E, Thayer JF.  
“Heart Rate Variability and Cardiac Vagal Tone in Psychophysiological Research – Recommendations for Experiment Planning, Data Analysis, and Data Reporting.”  
*Frontiers in Psychology*, 2017;8:213.  
DOI: 10.3389/fpsyg.2017.00213  
URL: https://pubmed.ncbi.nlm.nih.gov/28265249/

[9] Luque-Casado A, Perales JC, Cárdenas D, Sanabria D.  
“Heart rate variability and cognitive processing: The autonomic response to task demands.”  
*Biological Psychology*, 2016;113:83–90.  
DOI: 10.1016/j.biopsycho.2015.11.013  
URL: https://pubmed.ncbi.nlm.nih.gov/26638762/

[10] Delliaux S, Delaforge A, Deharo JC, Chaumet G.  
“Mental Workload Alters Heart Rate Variability, Lowering Non-linear Dynamics.”  
*Frontiers in Physiology*, 2019.  
URL: https://pubmed.ncbi.nlm.nih.gov/31156454/

[11] Qasim MS, Bari DS, Martinsen ØG.  
“Influence of ambient temperature on tonic and phasic electrodermal activity components.”  
*Physiological Measurement*, 2022;43(6):065001.  
DOI: 10.1088/1361-6579/ac72f4  
URL: https://pubmed.ncbi.nlm.nih.gov/35609614/

[12] Anusha AS, Joy J, Preejith SP, Joseph J, Sivaprakasam M.  
“Differential effects of physical and psychological stressors on electrodermal activity.”  
*IEEE Engineering in Medicine and Biology Society Annual Conference*, 2017:4549–4552.  
DOI: 10.1109/EMBC.2017.8037868  
URL: https://pubmed.ncbi.nlm.nih.gov/29060909/

[13] Vinkers CH, Penning R, Hellhammer J, Verster JC, Klaessens JHGM, Olivier B, Kalkman CJ.  
“The effect of stress on core and peripheral body temperature in humans.”  
*Stress*, 2013;16(5):520–530.  
DOI: 10.3109/10253890.2013.807243  
URL: https://pubmed.ncbi.nlm.nih.gov/23790072/

[14] Sarabia JA, Rol MA, Mendiola P, Madrid JA.  
“Circadian rhythm of wrist temperature in normal-living subjects: A candidate of new index of the circadian system.”  
*Physiology & Behavior*, 2008;95(4):570–580.  
DOI: 10.1016/j.physbeh.2008.08.005  
URL: https://pubmed.ncbi.nlm.nih.gov/18761026/

[15] Bonmati-Carrion MA, Middleton B, Revell V, Skene DJ, Rol MA, Madrid JA.  
“Circadian phase assessment by ambulatory monitoring in humans: correlation with dim light melatonin onset.”  
*Chronobiology International*, 2014;31(1):37–51.  
DOI: 10.3109/07420528.2013.820740  
URL: https://pubmed.ncbi.nlm.nih.gov/24164100/

[16] Boudreau P, Yeh WH, Dumont GA, Boivin DB.  
“Circadian variation of heart rate variability across sleep stages.”  
*Sleep*, 2013;36(12):1919–1928.  
DOI: 10.5665/sleep.3230  
URL: https://pubmed.ncbi.nlm.nih.gov/24293767/

[17] Stein PK, Pu Y.  
“Heart rate variability, sleep and sleep disorders.”  
*Sleep Medicine Reviews*, 2012;16(1):47–66.  
DOI: 10.1016/j.smrv.2011.02.005  
URL: https://pubmed.ncbi.nlm.nih.gov/21658979/

[18] Sipinková I, Hahn G, Meyer M, Tadlánek M, Hájek J.  
“Effect of respiration and posture on heart rate variability.”  
*Physiological Research*, 1997;46(3):173–179.  
URL: https://pubmed.ncbi.nlm.nih.gov/9728503/

[19] Oberle J, Elam M, Karlsson T, Wallin BG.  
“Temperature-dependent interaction between vasoconstrictor and vasodilator mechanisms in human skin.”  
*Acta Physiologica Scandinavica*, 1988;132(4):459–469.  
DOI: 10.1111/j.1748-1716.1988.tb08353.x  
URL: https://pubmed.ncbi.nlm.nih.gov/3227886/

[20] Fechir M, Schlereth T, Kritzmann S, et al.  
“Stress and thermoregulation: different sympathetic responses and different effects on experimental pain.”  
*European Journal of Pain*, 2009;13(9):935–941.  
DOI: 10.1016/j.ejpain.2008.11.002  
URL: https://pubmed.ncbi.nlm.nih.gov/19136286/

[21] Linder SP, Wendelken SM, Wei E, McGrath SP.  
“Using the morphology of photoplethysmogram peaks to detect changes in posture.”  
*Journal of Clinical Monitoring and Computing*, 2006;20(3):151–158.  
DOI: 10.1007/s10877-006-9015-2  
URL: https://pubmed.ncbi.nlm.nih.gov/16688391/

[22] Ritz T, et al.  
“Putting back respiration into respiratory sinus arrhythmia or high-frequency heart rate variability: Implications for interpretation, respiratory rhythmicity, and health.”  
*Biological Psychology*, 2023.  
DOI: 10.1016/j.biopsycho.2023.108728  
URL: https://pubmed.ncbi.nlm.nih.gov/38092221/

[23] Bilan A, Witczak A, Palusiński R, Myśliński W, Hanzlik J.  
“Circadian rhythm of spectral indices of heart rate variability in healthy subjects.”  
*Journal of Electrocardiology*, 2005.  
DOI: 10.1016/j.jelectrocard.2005.01.012  
URL: https://pubmed.ncbi.nlm.nih.gov/16003709/

[24] Schwerdtfeger A, Friedrich-Mai P.  
“Social interaction moderates the relationship between depressive mood and heart rate variability: evidence from an ambulatory monitoring study.”  
*Health Psychology*, 2009.  
DOI: 10.1037/a0014664  
URL: https://pubmed.ncbi.nlm.nih.gov/19594275/

[25] Hawkley LC, Burleson MH, Berntson GG, Cacioppo JT.  
“Loneliness in everyday life: cardiovascular activity, psychosocial context, and health behaviors.”  
*Journal of Personality and Social Psychology*, 2003;85(1):105–120.  
DOI: 10.1037/0022-3514.85.1.105  
URL: https://pubmed.ncbi.nlm.nih.gov/12872887/

[26] Han SC, Schacter HL, Timmons AC, Kim Y, Sichko S, Pettit C, Chaspari T, Narayanan S, Margolin G.  
“Romantic partner presence and physiological responses in daily life: Attachment style as a moderator.”  
*Biological Psychology*, 2021;161:108082.  
DOI: 10.1016/j.biopsycho.2021.108082  
URL: https://pubmed.ncbi.nlm.nih.gov/33753190/

[27] Smeijers D, et al.  
“Psychophysiological Reactivity in Couples During a Marital Interaction Task.”  
*Applied Psychophysiology and Biofeedback*, 2017.  
URL: https://pubmed.ncbi.nlm.nih.gov/28866813/

[28] Wang R, Chen F, Chen Z, Li T, Harari G, Tignor S, Zhou X, Ben-Zeev D, Campbell AT.  
“StudentLife: Assessing Mental Health, Academic Performance and Behavioral Trends of College Students Using Smartphones.”  
*ACM UbiComp*, 2014.  
DOI: 10.1145/2632048.2632054  
URL: https://www.cs.dartmouth.edu/~xia/publication/ubicomp14-studentlife/

[29] Ronao CA, Cho SB.  
“Recognizing human activities from smartphone sensors using hierarchical continuous hidden Markov models.”  
*International Journal of Distributed Sensor Networks*, 2017.  
DOI: 10.1177/1550147716683687  
URL: https://journals.sagepub.com/doi/10.1177/1550147716683687

[30] Cole RJ, Kripke DF, Gruen W, Mullaney DJ, Gillin JC.  
“Automatic sleep/wake identification from wrist activity.”  
*Sleep*, 1992;15(5):461–469.  
DOI: 10.1093/sleep/15.5.461  
URL: https://pubmed.ncbi.nlm.nih.gov/1455130/

[31] Kripke DF, Hahn EK, Grizas AP, et al.  
“Wrist actigraphic scoring for sleep laboratory patients: algorithm development.”  
*Journal of Sleep Research*, 2010.  
DOI: 10.1111/j.1365-2869.2010.00835.x  
URL: https://pubmed.ncbi.nlm.nih.gov/20408923/

[32] Plekhanova T, et al.  
“Validation of an automated sleep detection algorithm using data from multiple accelerometer brands.”  
*Journal of Sleep Research*, 2023.  
URL: https://pubmed.ncbi.nlm.nih.gov/36317222/

[33] Hot P, Naveteur J, Leconte P, Sequeira H.  
“Diurnal variations of tonic electrodermal activity.”  
*International Journal of Psychophysiology*, 1999;33(3):223–230.  
DOI: 10.1016/S0167-8760(99)00060-4  
URL: https://pubmed.ncbi.nlm.nih.gov/10533838/

[34] Xu Q, Nwe TL, Guan C.  
“Cluster-based analysis for personalized stress evaluation using physiological signals.”  
*IEEE Journal of Biomedical and Health Informatics*, 2015;19(1):275–281.  
DOI: 10.1109/JBHI.2014.2311044  
URL: https://pubmed.ncbi.nlm.nih.gov/25561450/

[35] Tazarv A, Labbaf S, Reich SM, Dutt N, Rahmani AM, Levorato M.  
“Personalized Stress Monitoring using Wearable Sensors in Everyday Settings.”  
*IEEE EMBC*, 2021:7332–7335.  
DOI: 10.1109/EMBC46164.2021.9630224  
URL: https://pubmed.ncbi.nlm.nih.gov/34892791/

[36] Bolpagni M, Pardini S, Dianti M, Gabrielli S.  
“Personalized Stress Detection Using Biosignals from Wearables: A Scoping Review.”  
*Sensors*, 2024;24(10):3221.  
DOI: 10.3390/s24103221  
URL: https://pubmed.ncbi.nlm.nih.gov/38794074/

[37] Celka P, Charlton PH, Farukh B, Chowienczyk P, Alastruey J.  
“Influence of mental stress on the pulse wave features of photoplethysmograms.”  
*Healthcare Technology Letters*, 2020;7(1):7–12.  
DOI: 10.1049/htl.2019.0001  
URL: https://pubmed.ncbi.nlm.nih.gov/32190335/

[38] Luks AM, Swenson ER.  
“Pulse oximetry at high altitude.”  
*High Altitude Medicine & Biology*, 2011;12(2):109–119.  
DOI: 10.1089/ham.2011.0013  
URL: https://pubmed.ncbi.nlm.nih.gov/21718156/

[39] Ceylan B, Khorshid L, Güneş ÜY, Zaybak A.  
“Evaluation of oxygen saturation values in different body positions in healthy individuals.”  
*Journal of Clinical Nursing*, 2016.  
URL: https://pubmed.ncbi.nlm.nih.gov/26879626/

[40] Alam R, Peden DB, Lach JC.  
“Wearable Respiration Monitoring: Interpretable Inference with Context and Sensor Biomarkers.”  
*IEEE Journal of Biomedical and Health Informatics*, 2021;25(6):1938–1948.  
DOI: 10.1109/JBHI.2020.3035776  
URL: https://pubmed.ncbi.nlm.nih.gov/33147151/

---

## Final implementation principle

The subsystem should behave according to one rule:

> **Context does not decide whether the physiology is real. Context decides which expectation and which level of interpretation are justified for that real physiology.**

That preserves the architecture:

```text
trusted physiology
→ generic context-conditioned meaning
→ downstream AUD-specific meaning/action
```

without turning non-specific wearable physiology into unsupported psychological or clinical labels.
