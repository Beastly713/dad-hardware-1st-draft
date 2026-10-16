# Temporal Physiological Event Modeling for an AUD Physiological Monitoring System

**Design question:**  
> Given already-valid, decomposed, confound-corrected, context-aware, multimodal and patient-referenced physiological observations, how should the system determine whether the observed change is a transient fluctuation, a sustained episode, a recurring event, a recovery process, an escalation/de-escalation pattern, or part of a larger physiological trajectory?

**Scope:** temporal structuring of physiological observations. This layer does **not** independently diagnose withdrawal, craving, intoxication, relapse, or treatment need.

**Status of recommendations:** This document separates (a) methods directly supported by published temporal/physiological studies, (b) AUD/SUD-specific temporal evidence, and (c) explicitly labelled **engineering starting points** that must be calibrated on the target dataset.

---

## Executive design decision

The strongest practical design is **not one monolithic time-series model**. It is a hierarchical temporal engine with distinct online and retrospective responsibilities:

```text
already-valid personalized physiological observations
                |
                v
      modality/component event trackers
                |
        lag-aware association
                |
                v
       multimodal episode tracker
                |
      +---------+----------+
      |                    |
      v                    v
online causal state     retrospective reconciliation
(candidate/active/      (boundary refinement,
recovering/resolved)    merge/split, change points)
      |                    |
      +---------+----------+
                |
                v
       finalized episode objects
                |
      +---------+----------+
      |                    |
      v                    v
recurrence/templates     daily burden
      |                    |
      +---------+----------+
                |
                v
       multi-day trajectory
                |
                v
      AUD-specific interpretation
                |
                v
        clinical meaning/action
```

The central implementation recommendation is:

1. **Represent physiology as patient- and context-referenced deviation streams**, not raw absolute values.
2. Use **causal change detection plus threshold/state evidence** to open a *candidate* event.
3. Use an explicit **hysteretic event lifecycle** to avoid chatter and premature closure.
4. Make **duration, integrated burden, recovery and missingness first-class variables**.
5. Associate modality-level events into a **lag-aware multimodal episode**, rather than requiring simultaneous peaks.
6. Keep an **online event record** that can later be reconciled by an offline segmentation pass.
7. Build recurrence, burden and trajectory from **finalized episodes**, not from every sensor-level event.
8. Do not let generic temporal machinery assign AUD meaning. The temporal layer may say *“three similar sustained nighttime autonomic episodes occurred in 72 h with progressively slower recovery”*; the downstream AUD layer decides what that means.

This architecture follows the broad lessons of ambulatory stress episode mining [1,2], physiologically informed drug-event localization and recovery modeling [3], change-point analysis of ambulatory physiology [4], online Bayesian change-point detection [5], explicit-duration temporal models [8], and AUD/SUD longitudinal physiology [10-23].

---

# 1. Where temporal modeling should occur

Temporal reasoning should occur **at multiple levels**, not only once after multimodal fusion.

| Level | Input | Temporal object | Why it exists | Output consumed by |
|---|---|---|---|---|
| Signal/component temporal metadata | already-clean feature/component stream | feature event | preserves when an individual physiological component changed | modality tracker / fusion |
| Modality physiological event | HR, HRV, EDA, respiration, temperature, etc. | modality event | captures modality-specific onset/offset and latency | multimodal association |
| Multimodal autonomic episode | fused state + modality events | episode | represents coherent cross-modal physiology | recurrence/burden/AUD layer |
| Context-linked episode | episode + context | contextualized episode | allows “same physiology, different situation” to remain distinguishable | AUD layer |
| Daily/nightly aggregation | finalized episodes | burden state | summarizes occupancy, count, intensity and recovery | longitudinal model |
| Multi-day/week aggregation | burden history | trajectory | detects worsening/improving temporal pattern | AUD/treatment layer |
| AUD-specific episode | generic temporal evidence + AUD-specific evidence | AUD-labelled episode | belongs downstream because meaning is diagnosis-specific | clinical layer |

A temporal engine that runs only after fusion would lose useful information about **which modality changed first**, how long each modality stayed abnormal, and whether apparent disagreement was actually a normal latency difference. Conversely, counting every modality event as a separate clinical episode would double-count one underlying autonomic event. The solution is therefore **hierarchical events with parent-child links**.

---

# 2. Temporal object model

The system should use the following operational hierarchy.

| Object | Operational definition | Typical timescale | Detection / construction | Output |
|---|---|---:|---|---|
| **Sample** | one timestamped observation or upstream state estimate | milliseconds-seconds | acquisition/upstream | value + quality |
| **Window** | short causal aggregation interval used for stable features/state inference | seconds-minutes | fixed or adaptive window | feature vector |
| **Candidate event** | possible departure from current physiological regime not yet persistent enough to accept | seconds-minutes | threshold/change detector | provisional start + probability |
| **Event** | bounded modality or latent-state deviation with an estimated onset and offset | seconds-minutes | event lifecycle | event object |
| **Episode** | temporally contiguous or gap-tolerant collection of related events representing one underlying physiological process | minutes-hours | association/merge logic | episode object |
| **State** | physiological regime that persists for a non-trivial residence time | minutes-hours | lifecycle/HSMM-like logic | BASELINE/ACTIVATED/etc. |
| **Transition** | change between regimes | instant with uncertainty interval | change detector | from/to + confidence |
| **Recovery period** | post-peak/post-activation interval moving toward expected physiology | minutes-hours | recovery tracker | slope, half-recovery, completion |
| **Recurrence** | later episode sufficiently similar to a prior episode/template | hours-days | event similarity | recurrence group |
| **Cluster/burst** | unusually dense set of related episodes within a horizon | hours-days | inter-event interval/frequency model | burst object |
| **Burden** | cumulative occupancy/intensity/recovery cost over a fixed horizon | hours-days | aggregation | minutes/day, AUC/day, episodes/day |
| **Trend** | directional change in a summary metric | days-weeks | robust trend model | slope + uncertainty |
| **Trajectory** | ordered sequence of burden/states/trends over a clinically meaningful horizon | days-months | trajectory model | improving/stable/worsening + evidence |

**Important:** an *event* is not synonymous with an AUD event. A detected event can remain `GENERIC_AUTONOMIC_EVENT` until downstream evidence is sufficient.

---

# 3. Mathematical representation

Let the temporal layer receive, at each update time \(t\):

\[
O_t = \{z_{j,t}, q_{j,t}, p_{k,t}, c_t, m_t\}
\]

where:

- \(z_{j,t}\) = patient/context/circadian-referenced deviation for physiological feature \(j\);
- \(q_{j,t} \in [0,1]\) = confidence/reliability for feature or modality \(j\);
- \(p_{k,t}\) = probability or score for upstream fused physiological state \(k\);
- \(c_t\) = context state and context confidence;
- \(m_t\) = missing-data mask.

The temporal layer should not re-learn the physiological baseline. It consumes the **expected-vs-observed deviation** supplied by personalization/context layers:

\[
z_{j,t} = \frac{x_{j,t}-\mu_{j,\mathrm{expected}}(t,c_t)}{\sigma_{j,\mathrm{expected}}(t,c_t)+\epsilon}
\]

A finalized event \(E_i\) is represented as:

\[
E_i =
\{
t^{start}_i,\,
t^{peak}_i,\,
t^{end}_i,\,
D_i,\,
S_i,\,
B_i,\,
T^{recovery}_i,\,
M_i,\,
C_i,\,
Q_i
\}
\]

with:

\[
D_i = t^{end}_i - t^{start}_i
\]

and an integrated burden:

\[
B_i = \int_{t^{start}_i}^{t^{end}_i} \max(0,s(t)-s_0)\,dt
\]

where \(s(t)\) is an episode severity/activation score and \(s_0\) is the expected-state threshold.

Useful within-event descriptors are:

\[
S_{\max} = \max_t s(t)
\]

\[
S_{\mathrm{mean}} = \frac{1}{D_i}\int s(t)\,dt
\]

\[
r_{\mathrm{rise}} = \frac{S_{\mathrm{peak}}-S_{\mathrm{start}}}{t_{\mathrm{peak}}-t_{\mathrm{start}}}
\]

\[
r_{\mathrm{fall}} = \frac{S_{\mathrm{end}}-S_{\mathrm{peak}}}{t_{\mathrm{end}}-t_{\mathrm{peak}}}
\]

Recurrence over horizon \(T\):

\[
R(T)=\sum_i \mathbf{1}[E_i \sim E_{\mathrm{query}} \land t_i \in (t-T,t]]
\]

where \(E_i \sim E_j\) is an event-similarity relation, not merely “same threshold crossed.”

Daily burden can be represented by a vector, not one scalar:

\[
\mathbf{B}_{day}=
[
N_{episodes},
T_{abnormal},
AUC_{deviation},
\bar{S},
\bar{T}_{recovery},
T_{night},
N_{incomplete\ recovery}
]
\]

This preserves distinctions between one long episode and many short episodes.

---

# 4. Evidence classes and how they are used

The literature is heterogeneous. This document therefore assigns evidence to three categories.

### A. Direct temporal-method evidence
Examples: change-point detection, explicit-duration state models, event segmentation and ambulatory stress episodes [1,4-9].

**Use:** justify algorithmic structure.

### B. SUD/AUD physiological-event evidence
Examples: cocaine event/recovery dynamics [3], alcohol withdrawal trajectories [10-13], alcohol cue reactivity [16,17,22], sleep/autonomic recovery [19-21].

**Use:** justify which temporal properties matter in addiction-related physiology.

### C. Engineering starting points
Examples: a 2-minute provisional-confirmation default, a 3-minute recovery stability rule, event-similarity cutoffs.

**Use:** make the system implementable before a target dataset exists. These are **not clinical thresholds** and must be tuned using patient-level validation.

The Sarker stress work is especially instructive: it used a 1-minute cStress inference window and generated overlapping stress likelihood estimates every 5 s, then used MACD-derived increasing/decreasing intervals to construct episodes. “Significant” stress used both a high likelihood criterion and a duration criterion; a 7.3-minute duration threshold produced approximately one significant episode/day in their specific opioid-dependent sample [1]. This is evidence for **magnitude + duration + episode construction**, but **not** evidence that every autonomic event in AUD should use 7.3 minutes.

---

# 5. Time base and multi-timescale channels

Do not force all temporal phenomena into one window.

The production design should maintain four synchronized channels:

```text
FAST CHANNEL
seconds -> a few minutes
modality transitions, abrupt activation, brief transients

EPISODE CHANNEL
minutes -> hours
sustained multimodal activation, recovery, episode morphology

DAILY/NIGHTLY CHANNEL
hours -> days
occupancy, episode count, recovery burden, nighttime instability

LONGITUDINAL CHANNEL
days -> months
recurrence evolution, burden trend, abstinence/treatment recovery trajectory
```

### Engineering starting configuration

For the existing streaming architecture, a reasonable starting point is:

- base update cadence: **1 s**;
- short feature/state window: **10-30 s** where the upstream physiology permits it;
- HRV-specific windows: use the duration required by the chosen HRV metric rather than pretending 10 s and 5 min HRV are interchangeable;
- event-evidence smoothing: **30-60 s** causal;
- episode state updates: every **5-10 s**;
- daily/nightly aggregation: fixed local-day and sleep-episode summaries;
- longitudinal update: once per day plus retrospective reprocessing.

These are engineering defaults, not disease thresholds.

cStress provides a precedent for one-minute physiological inference and 5-s overlapping updates [1,2]. The exact windows in this system should remain modality-specific.

---

# 6. Candidate onset detection

A candidate event should be opened by **either a strong state excursion or a statistically credible regime change**, but confirmation requires persistence.

Use two causal detectors in parallel:

## 6.1 Score/deviation gate

Construct a reliability-weighted severity score:

\[
s_t = \frac{\sum_j q_{j,t} w_j g_j(z_{j,t})}
           {\sum_j q_{j,t} w_j + \epsilon}
\]

where \(g_j\) maps each feature’s deviation into the physiologically relevant direction (e.g. HR elevation, HRV suppression, EDA increase).

If an upstream multimodal activation probability \(p_t\) already exists, do **not** duplicate fusion; use that score directly and retain modality contributions for provenance.

## 6.2 Change-point gate

Maintain a causal detector on the multivariate deviation/state stream.

Recommended production choices:

- **Bayesian online change-point detection (BOCPD)** when a probabilistic run-length estimate is useful [5];
- **Page-Hinkley/CUSUM-like detector** when compute and interpretability dominate;
- multivariate Bayesian change-point detection is especially attractive for personalized unsupervised physiological transitions; recent stress work reported successful alignment of multimodal physiological change points with stress periods [9].

For retrospective event reconciliation:

- use **PELT** or another penalized offline multiple-change-point method [6,7].

### Candidate start rule

A candidate starts at the earliest causal timestamp satisfying either:

1. `state_score >= enter_threshold`, **or**
2. change-point posterior / CUSUM exceeds its calibrated alarm threshold,

**and** at least one trusted physiological contributor has directionally consistent evidence.

A single isolated sample is never enough to call a sustained episode.

---

# 7. Event confirmation and minimum persistence

The event lifecycle distinguishes evidence accumulation from final event semantics.

```text
BASELINE
   |
   | candidate trigger
   v
CANDIDATE
   |
   | persistence + evidence
   v
ACTIVE / CONFIRMED
   |
   | sustained criterion
   v
SUSTAINED
   |
   | recovery evidence
   v
RECOVERING
   |
   | stable recovery
   v
RESOLVED
```

Additional states:

```text
UNKNOWN / DATA_GAP
CENSORED_LEFT
CENSORED_RIGHT
INVALIDATED
MERGED
SPLIT
```

### Engineering starting rule for a generic multimodal autonomic episode

Until target-specific data are available:

- `CANDIDATE`: immediate on change/entry evidence;
- `CONFIRMED`: qualifying evidence occupies **>=70% of the next 2 minutes** and no upstream invalidation occurs;
- `SUSTAINED`: qualifying evidence persists for **>=5 minutes**;
- `<2 min`: keep as `TRANSIENT_EVENT` unless a target-specific model says short events are meaningful.

These are explicitly **engineering defaults**. They are intentionally more conservative than sample-by-sample thresholding and much shorter than Sarker’s use-case-specific “significant stress” criterion [1].

For a target with known rapid events (e.g. arrhythmia), these values must be replaced. For longer AUD phenomena such as withdrawal or overnight autonomic dysregulation, the event horizon is hours to days.

---

# 8. Hysteresis and debounce

Threshold chatter is unacceptable.

Use separate entry and exit criteria:

```text
enter when:
    score >= theta_enter
    OR strong change-point evidence

remain active while:
    score > theta_exit
    OR recovery has not stabilized

exit only when:
    score <= theta_exit
    AND recovery criterion is satisfied
    AND minimum stable recovery time has elapsed
```

with:

\[
\theta_{exit} < \theta_{enter}
\]

**Engineering starting point:** if the normalized fused score is scaled to `[0,1]`, begin calibration around `theta_enter=0.65-0.70` and `theta_exit=0.35-0.45`. These values are **not literature-derived clinical cutoffs**.

A short refractory/debounce interval after closure should stop the same recovery tail from immediately reopening as a new event.

**Starting point:** 1-2 minutes for generic autonomic episodes, unless a clear independent change point indicates a genuinely new event.

---

# 9. Offset and event closure

An event should not end because of one normal sample.

Define recovery completion using a **stable-return rule**:

1. fused episode score falls below `theta_exit`;
2. the majority of contributing modalities return to their expected/personalized band;
3. no strong new change point indicates a different abnormal state;
4. the conditions persist for a minimum stable interval.

### Engineering starting point

Use **3 minutes of stable recovery** for generic sustained autonomic episodes, with `>=80%` of windows in the recovery band.

If the episode is shorter than the recovery-stability duration, require at least a stable interval comparable to the event’s duration rather than forcing a 3-minute rule.

The stored end timestamp should be:

- the estimated physiological return point, plus
- an `offset_uncertainty` interval if exact closure is ambiguous.

Do not set the end timestamp equal to “time the software finally became certain.”

---

# 10. Persistence and severity

Persistence should be represented explicitly by:

- continuous duration;
- qualifying-state occupancy;
- number of qualifying windows;
- longest continuous qualifying run;
- integrated deviation;
- time spent at high severity.

For an episode interval \(I\):

\[
Occupancy(I) =
\frac{\sum_t \mathbf{1}[s_t > \theta] \Delta t}
     {\sum_t \mathbf{1}[data\ usable] \Delta t}
\]

This is preferable to raw duration when intermittent missingness exists.

## Severity vector

Every event should store:

```text
peak_severity
mean_severity
median_severity
integrated_burden
duration
time_to_peak
rise_slope
fall_slope
within_event_variability
high_severity_occupancy
```

A downstream model may later learn a composite score, but the raw descriptors must remain available.

This is essential because:

```text
large deviation for 10 s
```

and

```text
moderate deviation for 45 min
```

carry different temporal information even if their peaks are similar.

---

# 11. Gap tolerance and missing data

Missing data must be distinguished from physiological normalization.

A gap state is:

```text
UNKNOWN
```

not:

```text
NORMAL
```

### Short data loss

If a modality becomes unavailable while an episode is active:

- continue the **parent episode** if other modalities still support it;
- reduce confidence in proportion to missing evidence;
- retain a per-modality gap interval;
- do not infer recovery from missingness.

### Engineering starting policy

For generic multimodal episodes:

- gap `<=30 s`: bridge if pre-gap and post-gap evidence are consistent;
- `30 s-2 min`: bridge only if at least one independent trusted modality remains supportive;
- `2-5 min`: parent episode may remain open as `ACTIVE_WITH_GAP`, but boundaries are uncertain;
- `>5 min`: close as **right-censored at last known evidence** and begin a new candidate on return unless retrospective reconciliation strongly supports continuity.

These values must be calibrated to device dropout patterns and event timescale.

---

# 12. Event merging and splitting

## 12.1 Merge nearby events when

All are true:

1. gap is short relative to episode timescale;
2. physiology never achieved stable recovery;
3. event morphology is similar;
4. context did not undergo a meaningfully incompatible transition;
5. the later event looks like re-escalation of the same process rather than an independent onset.

### Engineering starting rule

Auto-merge if:

- gap `<=2 min`, and
- no stable-recovery criterion was met.

For gaps between `2 and 10 min`, merge only if:

- event similarity `>=0.8` (calibration value),
- residual deviation remained above a low recovery band, and
- no strong intervening change point/context transition occurred.

## 12.2 Split one long interval when

Any of these occurs:

- sustained return to baseline;
- strong change point into a qualitatively different physiological regime;
- a context transition plus a new physiological onset;
- distinct peaks separated by verified recovery;
- modality composition changes enough to indicate a different event class.

A long episode with several peaks can additionally retain `subevent_ids` without forcing complete splitting.

---

# 13. Censoring

Monitoring often starts or ends in the middle of an event.

Store:

```text
left_censored: true/false
right_censored: true/false
known_duration_lower_bound
start_interval
end_interval
```

A left-censored event has an unknown true onset. A right-censored event has an unknown true offset.

Do **not** use the recording boundary as if it were the physiological event boundary.

Censored events may contribute to occupancy burden but should be excluded or specially handled when estimating typical event duration or recovery-time distributions.

---

# 14. Duration-aware state modeling

A standard HMM implies a geometric state-duration distribution. That is often a poor conceptual fit when event residence time itself matters.

HSMMs explicitly model state duration and provide filtered/smoothed probabilities and state-sequence inference [8].

### Recommendation

Do **not** make an HSMM the first production dependency.

Use:

- explicit deterministic/semi-probabilistic lifecycle logic in v1;
- empirical duration distributions for each event type after sufficient history;
- HSMM or explicit-duration HMM only after enough labelled/validated episodes exist.

Why:

1. event labels in ambulatory AUD physiology are scarce;
2. the lifecycle rules need to be auditable;
3. duration distributions are likely patient- and event-specific;
4. HSMMs become much more valuable once the event taxonomy is stable.

---

# 15. Temporal-model comparison

| Method | Online? | Duration-aware? | Missing data | Interpretability | Data need | Recommended role |
|---|---:|---:|---|---|---|---|
| Fixed threshold | yes | only by added rules | simple | very high | very low | contributor, never sole engine |
| Hysteretic state machine | yes | yes, explicit rules | very good | very high | low | **core v1 lifecycle** |
| CUSUM/Page-Hinkley | yes | indirectly | good | high | low | **online change candidate** |
| BOCPD [5] | yes | run-length posterior | model-dependent | high | low-moderate | **preferred probabilistic candidate detector** |
| MACD-style trend segmentation [1,3] | yes/near-online | implicit trend duration | moderate | high | low | useful for broad rise/fall morphology |
| PELT [6] | no (retrospective) | segmentation only | requires preprocessing | high | low | **offline boundary reconciliation** |
| Binary segmentation | retrospective/stream variants | segmentation only | requires preprocessing | high | low | secondary offline option |
| HMM | yes | geometric duration | moderate-good | medium-high | moderate | only if state model validated |
| HSMM [8] | yes with suitable inference | **yes** | model-dependent | medium-high | moderate-high | v2 after labelled episode corpus |
| Continuous state-space/Kalman | yes | continuous dynamics | very good | medium-high | moderate | latent smooth state / gap prediction |
| Switching state-space model | yes/offline | yes-ish | good | medium | high | later research |
| LSTM/GRU/TCN | yes if causal | learned | model-dependent | low-medium | high | not first choice |
| Transformer/TFT | possible | learned | model-dependent | low-medium | very high | not justified for current evidence base |

Deep models should only be introduced if a large event-labelled corpus exists and evaluation is strictly patient-separated and chronological.

---

# 16. State-transition model

The temporal engine should use **descriptive physiological states**, not invented clinical states.

Recommended generic states:

```text
BASELINE_EXPECTED
EARLY_ACTIVATION
CONFIRMED_ACTIVATION
SUSTAINED_ACTIVATION
RECOVERING
RESOLVED
UNKNOWN
```

Optional direction-specific states may exist for a target stream:

```text
ELEVATED
SUPPRESSED
MIXED
```

The transition probability can conceptually be represented as:

\[
P(S_t | S_{t-1}, O_t, D_{state}, c_t, H_t)
\]

where \(D_{state}\) is state residence time and \(H_t\) recent history.

In v1, this probability need not come from a learned Markov model. It can be produced by calibrated logistic/state evidence while the **transition graph remains explicit**.

Forbidden transitions should be encoded where physiologically/logically appropriate. For example, `BASELINE -> RESOLVED` makes no sense; `RECOVERING -> SUSTAINED_ACTIVATION` is allowed as re-escalation.

---

# 17. Recovery as a first-class process

Recovery is not equivalent to “the score became normal.”

For every substantial episode track:

```text
recovery_start
recovery_slope
time_to_half_recovery
time_to_80_percent_recovery
time_to_expected_band
residual_deviation
interrupted_recovery_count
```

Hossain et al. demonstrated the value of modeling the **shape of physiological recovery** rather than classifying isolated ECG cycles. Their cocaine model separated natural post-activity parasympathetic recovery from cocaine-dampened recovery and used candidate windows containing activation and recovery phases [3]. Cocaine responses in their data could last >30 minutes, with reported heart-rate recovery after intravenous cocaine commonly taking 20-60 minutes [3]. The exact kinetics are cocaine-specific, but the methodological lesson transfers strongly.

## 17.1 Recovery start

Recovery begins when all of the following are observed:

- episode severity has passed a local peak;
- smoothed severity derivative is negative for a minimum interval;
- at least one principal contributing modality is returning toward expected physiology;
- no new independent activation change point appears.

## 17.2 Recovery curve

Do not force every episode to an exponential.

Fit candidate models only where data density allows:

\[
d(t)=A e^{-(t-t_p)/\tau}+b
\]

or:

- linear robust slope;
- biexponential/nonlinear curve;
- patient-specific empirical recovery template.

Choose the simplest model whose residuals are acceptable.

For each event store the non-parametric recovery measurements even when curve fitting fails.

---

# 18. Recovery categories

Labels such as `FAST_RECOVERY`, `NORMAL_RECOVERY`, `DELAYED_RECOVERY`, `INCOMPLETE_RECOVERY` should be **patient-referenced**, not globally hard-coded.

Recommended derivation:

```text
expected_recovery_distribution
    = patient's historical recovery times
      for same event phenotype/context
```

Then:

```text
FAST:
    below lower reference quantile

TYPICAL:
    within expected band

DELAYED:
    above upper reference quantile

INCOMPLETE:
    expected band not reached before
    censoring / next event / horizon
```

Until at least several comparable events exist, report continuous recovery metrics with wide uncertainty rather than categorical claims.

AUD studies support recovery as a longitudinal construct: HR/HRV abnormalities can improve with abstinence over weeks to months [12,14,20], but these papers do not supply a universal minute-level “normal recovery” cutoff for generic autonomic episodes.

---

# 19. Escalation and de-escalation within episodes

Use three complementary indicators:

1. robust slope of severity over a recent causal window;
2. successive change points into higher/lower regimes;
3. cross-modal directional agreement.

Example:

```text
HR deviation:  +10 -> +15 -> +24 bpm
HRV deviation: -10% -> -20% -> -35%
EDA:            moderate -> high
```

should be represented as a **multivariate escalation**, provided upstream confound/context handling still validates the evidence.

Define:

\[
E_t = \text{TheilSenSlope}(s_{t-W:t})
\]

or a robust linear slope.

### Engineering starting point

Call `ESCALATING` only if:

- slope sign is positive over at least 3 consecutive evaluation windows;
- change exceeds a minimum effect-size threshold;
- confidence is not falling because of missingness.

`DE_ESCALATING` mirrors this logic.

Do not infer clinical worsening solely from within-episode escalation.

---

# 20. Recurrence is event similarity + reappearance

Recurrence must not mean “the same scalar threshold crossed twice.”

Represent each finalized episode by an embedding/vector:

```text
[
  modality_support_vector,
  peak_severity,
  mean_severity,
  duration,
  integrated_burden,
  rise_time,
  time_to_peak,
  recovery_time,
  recovery_shape,
  context_class,
  circadian_phase,
  sleep_wake_state
]
```

Normalize dimensions by personal historical variability where possible.

Compute a composite distance:

\[
d(E_a,E_b)
=
\lambda_1 d_{physiology}
+\lambda_2 d_{shape}
+\lambda_3 d_{duration}
+\lambda_4 d_{recovery}
+\lambda_5 d_{context}
\]

For waveform/morphology comparison, dynamic time warping can be used selectively. For explainability, store the component distances.

### Engineering starting point

Two events can enter the same recurrence group if:

- physiological support/morphology is strongly similar;
- duration/recovery are not grossly incompatible;
- context is compatible or explicitly allowed to differ.

Start with an empirically tuned similarity threshold around `0.75-0.80`, but expose it as configuration and recalibrate per dataset.

---

# 21. Event templates and event-level personalization

Once enough comparable episodes exist, learn **event-level baselines** in addition to physiological baselines.

Example:

```text
Patient template A:
  rapid EDA rise
  HR +18 bpm relative to expected
  HRV suppression
  duration median 19 min
  recovery median 11 min
  mostly nighttime/rest
```

A future event can then be described as:

```text
severity: +1.8 SD vs patient's similar events
duration: +2.2 SD
recovery: +2.7 SD slower
morphology: highly similar
```

This is often more useful than comparing the event only with a population norm.

### Sparse history policy

- `<3 comparable episodes`: no template; report raw descriptors;
- `3-9`: provisional template with broad uncertainty;
- `>=10`: robust median/MAD or hierarchical estimate can be used;
- never hide sample count.

These are engineering governance rules, not physiological truths.

---

# 22. Temporal clustering and burst detection

Higher-order temporal structure matters.

Track inter-event intervals:

\[
IEI_i=t^{start}_{i+1}-t^{end}_{i}
\]

and event rate over multiple horizons:

```text
1 h
6 h
24 h
72 h
7 d
```

A burst is not defined only by count. Compare the observed count/spacing with that patient’s expected rate for the same circadian/context stratum.

Useful metrics:

```text
episodes_per_6h
episodes_per_24h
median_inter_event_interval
minimum_inter_event_interval
coefficient_of_variation_IEI
nightly_episode_count
```

For an initial system without enough personal history, report “5 episodes/6 h” rather than labeling it pathologic.

---

# 23. Cumulative event burden

Per horizon \(H\), calculate:

```text
episode_count
total_episode_minutes
fraction_observed_time_in_episode
integrated_deviation_AUC
high_severity_minutes
mean_episode_severity
max_episode_severity
mean_recovery_time
incomplete_recovery_count
nighttime_episode_minutes
```

Use observed-time denominators:

\[
BurdenOccupancy_H =
\frac{T_{episode,H}}{T_{usable,H}}
\]

rather than dividing by 24 hours when the device was worn only 8 hours.

For multimodal parent episodes, burden is counted **once**. Modality child events are supporting evidence, not additional clinical burden.

---

# 24. Circadian-aware reasoning

Repeated patterns at the same clock time may reflect expected circadian physiology.

Event detection should operate on:

\[
x_{observed}(t) - x_{expected}(circadian\ phase, context, sleep/wake)
\]

not raw values alone.

Use the personalized layer’s expected circadian/context model as input. The temporal engine then asks whether a **residual deviation** persists or recurs.

This matters because Sarker et al. observed systematic time-of-day differences in ambulatory stress density [1], and alcohol withdrawal monitoring has also shown time-of-day structure in sweating and skin temperature [11].

### Night/day recurrence

Store both:

```text
clock_time
circadian_phase (if available)
sleep/wake state
time_since_sleep_onset
```

A pattern recurring “90 minutes after sleep onset” is potentially more coherent than one recurring at a fixed wall-clock time.

---

# 25. Event-context linkage

Each episode stores the context present:

- at onset;
- at peak;
- during the majority of the episode;
- during recovery.

Do not compress context to one label if it changed.

Example:

```yaml
context:
  onset:
    sleep_state: awake
    activity: rest
    location_class: home
  peak:
    sleep_state: N/A
    activity: rest
  recovery:
    activity: light_movement
```

Context should influence **episode comparison** and downstream interpretation, but context changes should not automatically split an event unless accompanied by physiological evidence.

---

# 26. Lag-aware multimodal event association

Different modalities need not change simultaneously.

Represent each modality event as:

```text
modality
onset_interval
peak_time
offset_interval
direction
confidence
```

Then associate events using a **lag window** learned per modality pair/event phenotype.

A parent autonomic episode may contain:

```text
ECG/HR change       t0
respiration change  t0 + 8 s
EDA rise            t0 + 20 s
skin temperature    later/slower
```

without treating these as unrelated.

### Recommended matching logic

1. use onset uncertainty intervals;
2. allow modality-specific expected delays;
3. require temporal overlap or plausible causal adjacency;
4. score morphology/context compatibility;
5. create one parent episode;
6. retain original child timestamps.

Do not hard-code a universal cross-modal lag. Estimate lag distributions from synchronized labeled data.

---

# 27. Temporal precedence and sequence reasoning

The system should preserve event order:

```text
EDA onset -> HR rise -> HRV suppression -> recovery
```

is not identical to:

```text
HR rise -> EDA onset -> recovery
```

At v1, do not assign mechanistic meaning unless supported. Simply retain:

```text
ordered_child_events
pairwise_lags
```

At a later stage, sequence mining or a dynamic Bayesian model can assess whether certain ordered patterns predict AUD-specific states.

---

# 28. Event hierarchy and avoiding double-counting

Use a parent-child graph:

```text
HR_ELEVATION_EVENT ─┐
HRV_SUPPRESSION_EVENT ├─> AUTONOMIC_EPISODE_241
EDA_ACTIVATION_EVENT ┘

AUTONOMIC_EPISODE_241
      |
      +-- may later support --> AUD_SPECIFIC_EPISODE_X
```

Rules:

- child modality events retain separate evidence;
- only the parent multimodal episode contributes to parent-level burden;
- if a modality event exists without parent support, it remains a modality event;
- overlapping parent episodes of the same phenotype are reconciled before burden aggregation;
- AUD-specific episodes reference the generic physiological episode rather than copying it.

---

# 29. Event provenance

Every high-level event must be explainable.

Example:

```yaml
event_id: AUTONOMIC_EPISODE_241
state: SUSTAINED
start_estimate: 2026-10-16T21:14:20+05:30
start_uncertainty_s: 25
support:
  - HR_ELEVATION_881
  - HRV_SUPPRESSION_482
  - EDA_ACTIVATION_127
excluded_or_low_confidence:
  - skin_temperature: slow_response
context:
  activity: REST
  sleep_wake: AWAKE
confounds:
  motion: LOW
evidence:
  change_point: true
  persistence: true
  multimodal_agreement: true
confidence: 0.89
reason_codes:
  - PERSONAL_DEVIATION
  - MULTIMODAL_AGREEMENT
  - PERSISTENT
```

Provenance is mandatory because a later AUD inference should be able to trace back to the actual physiological evidence.

---

# 30. Event confidence

Confidence is a property of evidence quality, not severity.

Use separate components:

```text
Q_signal
Q_multimodal
Q_persistence
Q_context
Q_baseline
Q_missing
Q_boundary
```

A conservative aggregate is a weighted geometric mean:

\[
Q_E =
\exp\left(
\frac{\sum_k w_k \log(\epsilon+Q_k)}
     {\sum_k w_k}
\right)
\]

This prevents one extremely poor component from being completely hidden by several high scores.

Do not let persistence inflate confidence without bound. Persistence can move an event from possible to probable, but long duration cannot compensate for invalid signal quality.

### Confidence evolution

```text
first 30 s:
    candidate, low/moderate boundary confidence

after 2 min:
    confirmed if persistence/multimodal criteria met

after 5+ min:
    sustained; duration confidence rises

during data gap:
    evidence confidence falls

after stable recovery:
    end-boundary confidence rises

retrospective reconciliation:
    finalized confidence + boundary intervals
```

---

# 31. Online versus retrospective event records

Maintain two views:

## Online / causal

Only data up to \(t\) may be used.

Purpose:

- live state;
- provisional onset;
- current duration;
- escalation;
- early recovery;
- alerts or downstream streaming.

## Finalized retrospective

May use future samples.

Purpose:

- refine onset/offset;
- split/merge;
- PELT/offline CP;
- correct provisional events;
- compute accurate burden;
- research labels.

The database should preserve event revisions:

```text
event_version
online_start
final_start
online_end
final_end
revision_reason
```

This makes the system scientifically auditable.

---

# 32. Event reconciliation

The online system is allowed to revise its interpretation.

Example:

```text
t = 40 s:
CANDIDATE_EVENT

t = 3 min:
CONFIRMED_EVENT

t = 20 min:
SUSTAINED_EPISODE

t = 35 min:
RECOVERING

t = 42 min:
RESOLVED

offline:
onset shifted 22 s earlier
two short internal gaps merged
```

An event should never be silently rewritten. Keep provenance of revisions.

---

# 33. Change-point strategy

## Online

Preferred:

- BOCPD on a low-dimensional latent/fused state plus selected robust deviations [5];
- or Page-Hinkley/CUSUM for lower compute.

BOCPD is attractive because it maintains a posterior over **run length**—the time since the most recent change [5].

## Retrospective

Use PELT for multiple changes with a penalty chosen on validation data [6,7].

## Why not change points alone?

A statistical regime change can be:

- physiologically tiny;
- caused by a context transition;
- a new stable normal;
- noise from a baseline model.

Therefore a change point is a **boundary proposal**, not automatically an event.

---

# 34. Sequential probability testing

Sequential probability tests are appropriate when the target is a well-defined change with known pre/post distributions.

For a generic autonomic engine, the distributions will initially be poorly specified. Therefore:

- do not make SPRT the primary v1 detector;
- use CUSUM/BOCPD plus lifecycle persistence;
- consider SPRT later for a validated target such as a specific known post-intake or withdrawal pattern.

The main advantage would be explicit false-alarm/decision-delay trade-offs.

---

# 35. State-space models

Continuous latent-state models can help when physiology evolves smoothly.

Potential latent variable:

\[
a_t = \text{latent autonomic activation}
\]

with:

\[
a_t = f(a_{t-1}) + \eta_t
\]

\[
O_t = h(a_t) + \epsilon_t
\]

A Kalman/extended Kalman/particle filter can:

- smooth the latent state causally;
- propagate uncertainty through short gaps;
- separate measurement noise from state dynamics.

Recommendation: use this only if the fusion layer does not already supply an adequate latent-state probability. Do not create two competing “autonomic activation” models in adjacent layers.

---

# 36. Why deep temporal models are not the first choice

LSTM/GRU/TCN/Transformer models can learn sequence structure, but current AUD physiology evidence does not justify making them the core temporal layer.

Requirements before introducing them:

- enough subjects, not only enough windows;
- event-level labels;
- patient-independent validation;
- chronological split;
- no windows from one episode in both train and test;
- clear handling of missingness;
- comparison against the hybrid interpretable baseline;
- calibration of event-level false alarms/day, not only window accuracy.

Window-level random splits are particularly dangerous because adjacent windows are highly autocorrelated.

---

# 37. Temporal leakage and validation rules

Mandatory evaluation rules:

1. **patient-level separation** for claims of generalization;
2. **chronological separation** for personalization/forecasting;
3. no future-centered features in the online detector;
4. no retrospective boundary information in causal features;
5. no random window split that places parts of one episode into train and test;
6. evaluate **event detection**, not only point classification;
7. measure boundary error separately from event detection;
8. report false events/day or false events/hour;
9. report performance stratified by usable-data fraction;
10. report performance by event duration and severity.

Recommended event metrics:

```text
event sensitivity
event precision
false events / day
median onset delay
median offset error
intersection-over-union (episode interval)
duration error
burden error
recurrence-group precision
```

---

# 38. Temporal uncertainty

Boundaries are estimates.

Store:

```text
start_estimate
start_earliest
start_latest
end_estimate
end_earliest
end_latest
```

or posterior intervals from a probabilistic model.

If a detector becomes confident at 21:16 that the change probably began at 21:14:40, the physiological onset and detection time must remain separate fields.

---

# 39. Physiological target strategy

Temporal machinery should operate primarily on **derived physiological features/states**, but modality-level events remain useful for provenance.

| Target | Useful temporal object | Onset evidence | Persistence representation | Offset/recovery | Notes |
|---|---|---|---|---|---|
| HR / RR | elevation/suppression event | personal deviation + CP | duration/AUC | stable return; recovery slope | rapid, responsive; strong confound dependence upstream |
| HRV | suppression/elevation state | valid HRV estimate deviates from personal/context norm | state occupancy across valid HRV windows | stable normalization | window length depends on HRV metric |
| EDA | activation burst / sustained elevation | phasic rate/amplitude or tonic deviation + CP | SCR density/tonic occupancy/AUC | return toward expected tonic/phasic state | lag relative to cardiac changes expected |
| Respiration | rate/depth/variability event | deviation + morphology/state change | duration + occupancy | stable expected breathing | context/activity crucial upstream |
| Skin temperature | slow thermal deviation/state | slow change point | minutes-hours | gradual return | should not use same fast thresholds as HR |
| PPG-derived HR/PRV | cardiac event | same temporal logic as cardiac feature, quality permitting | duration/AUC | recovery | do not double-count ECG and PPG as independent physiology when measuring same construct |
| Tremor/IMU-derived withdrawal-relevant motion | repeated/sustained tremor event | spectral/motion feature deviation | duty cycle/burden | return to personal state | differentiate voluntary activity upstream |
| Multimodal autonomic score | parent episode | fused score + CP + child events | episode duration/AUC | multimodal recovery | primary generic episode |
| Sleep autonomic recovery | nightly state/trajectory | sleep onset-relative deviation | first-hours/night occupancy | overnight normalization | use sleep-stage/context stratification |
| Longitudinal autonomic burden | trajectory | daily summary CP/trend | days-weeks | sustained return toward personal expected | not an acute event |

---

# 40. Event-detection table

The literature does **not** provide universal onset/minimum-duration/offset thresholds for these generic states. Therefore the table separates evidence-backed structure from engineering defaults.

| Physiological/AUD target | Onset | Minimum persistence | Offset | Recovery criterion | Key references |
|---|---|---|---|---|---|
| Generic multimodal autonomic episode | fused-state excursion or CP + trusted modality | **2 min engineering default** for confirmation; 5 min sustained label | hysteretic lower threshold | 3 min stable recovery **engineering default** | [1,4,5,9] |
| Ambulatory stress-like episode | increasing trend in stress likelihood; Sarker used MACD boundaries | magnitude + duration; 7.3 min gave ~1 SSE/day in their dataset, not universal | following decreasing trend | episode ends with trend reversal/decline | [1,2] |
| Drug-like acute cardiac event methodology | broad HR/RR response window | full activation/recovery window; cocaine responses >30 min in study | recovery-window boundary | compare recovery dynamics | [3] |
| Withdrawal autonomic trajectory | time-linked sustained autonomic signs after cessation/reduction | hours, not seconds; early syndrome often begins around 6 h | resolution/stabilization over days | decreasing sustained autonomic burden | [10-13] |
| Cue-associated autonomic response | change during known cue exposure | no universal naturalistic duration supported | post-cue recovery | return toward pre-cue state | [16,17,22] |
| Alcohol/post-alcohol sleep-autonomic disturbance | within-person nocturnal HR/HRV deviation after drinking | first sleep hours/night | overnight/next-day recovery | return toward alcohol-free personal night | [18-21] |
| Treatment/recovery trajectory | daily/nightly burden shift | days-weeks | new stable level | longitudinal normalization | [14,20] |
| Relapse-risk physiological state | no validated event-level onset | multi-day summary only | not established | not established | [15-17] |

---

# 41. AUD-specific temporal evidence: withdrawal

Withdrawal is the AUD process with the clearest established clinical temporal structure.

A major review describes early withdrawal autonomic manifestations beginning around **6 h** after cessation/reduction and lasting roughly **4-48 h**, seizures commonly within 6-48 h, and delirium tremens typically beginning 48-72 h after cessation [10]. These timings are clinical syndrome timings, not wearable-event thresholds.

White et al. recorded 24-h tremor, sweating, skin temperature and locomotor activity in six withdrawing men; sweating and skin temperature were highest around 00:00-06:00, while tremor/activity showed different temporal behavior, supporting continuous objective temporal monitoring and warning against collapsing withdrawal into one scalar snapshot [11].

Weise et al. found higher HR and lower momentary HRV in withdrawing alcohol-dependent participants compared with later measurements, supporting the concept of **physiological change across withdrawal/abstinence**, while Bär et al. cautioned that classical HRV and sympathetic skin-response measures did not cleanly reflect acute withdrawal autonomic change in their sample [12,13].

## What to implement because of this

The temporal layer should maintain a **withdrawal-relevant physiological trajectory**, but not diagnose withdrawal.

If a downstream AUD layer supplies a known/estimated `time_since_last_drink`, the temporal engine should expose:

```text
autonomic_episode_burden_by_hour_since_last_drink
resting_HR_deviation_trajectory
EDA/sweating-related burden
temperature_deviation_trajectory
tremor_event_burden
respiratory_rate_deviation
nighttime_pattern
recovery/stabilization slope
```

Do **not** use HRV alone as an acute-withdrawal severity detector because the AUD evidence is inconsistent [13].

The temporal engine may output:

```text
persistent resting autonomic activation increased over 8 h,
peaked during the second monitoring block,
and declined over the subsequent 12 h
```

It must not output:

```text
severe withdrawal
```

without the downstream clinical/AUD model.

---

# 42. AUD-specific temporal evidence: craving and cue reactivity

Alcohol cue-reactivity studies demonstrate that physiological responses can occur during cue exposure, but the signal is variable.

Reid et al. found robust subjective cue effects but weaker/non-specific HR, skin-conductance and temperature effects; salivation was more alcohol-cue specific [17]. Witteman et al. found physiological cue reactivity and craving in 80 alcohol-dependent patients during detoxification, but baseline cue reactivity and craving did **not** predict relapse at follow-up [16]. HRV review evidence also shows complexity in resting vs reactive HRV [23].

## What to implement

Do not create a generic:

```text
CRAVING_EVENT = EDA up + HR up
```

rule.

Instead represent:

```text
CUE_ASSOCIATED_AUTONOMIC_EPISODE
```

only when cue context is independently known or strongly inferred.

Store:

```text
cue_onset_to_phys_onset_latency
time_to_peak
response_duration
recovery_time
modality_order
subjective_craving_alignment (if available)
```

Naturalistic craving duration should remain learned/uncertain. Laboratory exposure block duration is not a universal craving-event duration.

---

# 43. AUD-specific temporal evidence: intoxication and post-intoxication physiology

Acute alcohol affects autonomic physiology over time.

Fairbairn et al. showed that continuous transdermal alcohol data can be transformed into estimated BrAC time courses, but TAC may lag BAC by roughly **1-3 h**, with individual/situational variation [18]. This is important evidence for **lag-aware temporal interpretation**.

Pietilä et al. used the first 3 h of sleep in a large real-world sample and found dose-dependent increased sympathetic regulation, decreased parasympathetic regulation and reduced physiological recovery after alcohol [19].

## What to implement

If the system lacks a direct alcohol sensor or reliable intake timestamp, the temporal layer should not label intoxication.

It should expose:

```text
nocturnal_RHR_deviation
nocturnal_HRV_deviation
first_3h_sleep_recovery_burden
overnight_recovery_fraction
next_morning_residual_deviation
```

If a future alcohol-specific sensor is added, treat its pharmacokinetic event as a separate temporal stream and learn modality-specific lag to autonomic effects.

---

# 44. Sleep/nighttime temporal patterns in AUD

Recently detoxified alcohol-dependent individuals have shown elevated HR and reduced HRV during sleep, with abnormalities especially apparent early in the night [21]. de Zambotti et al. also reported partial recovery in HR and high-frequency HRV across approximately four months of abstinence [20].

## What to implement

Nighttime temporal reasoning should be sleep-relative:

```text
time_since_sleep_onset
sleep_stage (if reliable)
first_3h burden
nightly episode count
nighttime autonomic occupancy
overnight HR slope
overnight HRV recovery slope
number of awakenings/context transitions
```

Compare stable sleep periods with that patient’s own expected night profile.

Do not interpret predictable sleep-stage changes as pathological recurrence.

---

# 45. Treatment response and longer recovery trajectory

AUD recovery studies support longitudinal—not minute-level—temporal interpretation.

Eddie et al. found higher HRV with longer time since last drink among treatment-engaged adults in the first year of AUD recovery [14]. de Zambotti et al. observed recovery in HR and vagal modulation across four months of abstinence [20].

## What to implement

Maintain weekly/monthly trajectory variables:

```text
resting_HR expected deviation
resting/sleep HRV deviation
nighttime burden
episode frequency
mean episode duration
mean recovery time
incomplete recovery fraction
usable-data-normalized autonomic burden
```

A treatment-response temporal output can be:

```text
nighttime autonomic burden decreased on 9 of the last 12
comparable nights; median recovery time decreased from 28 min
to 14 min
```

without claiming causal treatment efficacy.

---

# 46. Relapse-risk temporal structure: conservative boundary

This area requires strict restraint.

Eddie et al. monitored 42 adults in early AUD recovery for 4 days and then followed alcohol use for 90 days; higher parasympathetic HRV and lower HR were associated with greater percent days abstinent [15]. That supports **prospective association of aggregate physiology with later alcohol-use outcome**, not an event detector for imminent relapse.

Witteman et al. found alcohol cue reactivity and craving but did not find that baseline cue response predicted relapse in their study [16].

Therefore the temporal layer must **not** output:

```text
RELAPSE_IMMINENT
```

from rising episode burden.

It may output:

```text
7-day autonomic burden is increasing
nighttime recovery is worsening
```

and allow a separately validated AUD risk model to determine whether this has predictive meaning.

---

# 47. Event burden trajectory

For each day/night, create a summary vector \(b_d\). Track trend using robust methods:

- Theil-Sen slope over a rolling horizon;
- EWMA/CUSUM for online shift detection;
- PELT for retrospective regime changes;
- bootstrap/confidence interval where data volume allows.

### Engineering starting labels

For a metric \(y_d\):

```text
WORSENING
    if robust slope is directionally adverse
    and the change exceeds the patient's expected day-to-day variability
    for >=3 comparable observations

IMPROVING
    analogous favorable direction

STABLE
    no meaningful directional change

INSUFFICIENT_DATA
    otherwise
```

The “>=3 observations” rule is a minimum engineering guardrail, not a clinical threshold. Longer confirmation should be used for noisy metrics.

---

# 48. Recovery deterioration

Repeated similar episodes should be compared on:

```text
time_to_peak
time_to_half_recovery
time_to_80_percent_recovery
time_to_expected_band
residual_deviation_at_fixed_horizon
```

Define:

\[
\Delta T_{recovery,i}
=
T_{recovery,i}
-
\mathrm{median}(T_{recovery,similar\ prior})
\]

A progressive positive \(\Delta T_{recovery}\) can be reported as **slower-than-usual recovery**.

This is exactly the type of temporal phenotype that can be useful downstream without making an AUD diagnosis.

---

# 49. Escalating recurrence

Track four independent axes:

```text
frequency escalation
duration escalation
severity escalation
recovery deterioration
```

Example output:

```text
frequency:
  1 -> 3 -> 6 episodes/day

median duration:
  12 -> 17 -> 26 min

median recovery:
  9 -> 15 -> 31 min
```

Do not collapse this into a single “risk score” until validated.

---

# 50. Novel-event detection

Detect novelty relative to the patient’s historical **episode distribution**, not only sample distribution.

Candidate novelty score:

\[
N(E)=
\alpha d(E,\mathrm{nearest\ template})
+
\beta \, surprisal(E)
\]

Inputs can include:

- modality composition;
- severity;
- duration;
- rise/fall shape;
- recovery;
- context/circadian phase.

Use one-class or clustering methods only after enough event history exists.

A novel event should initially be labelled:

```text
NOVEL_TEMPORAL_PHENOTYPE
```

not a new clinical syndrome.

---

# 51. Multi-resolution summaries

Recommended emitted summaries:

### Live (every update)
```text
current_state
current_episode_id
current_duration
current_severity
trend
confidence
```

### Hourly
```text
episode_count
episode_minutes
AUC
peak
recovery_status
```

### Daily/nightly
```text
burden
recurrence
mean/max duration
mean recovery
nighttime fraction
usable time
```

### 3-7 day
```text
frequency trend
duration trend
recovery trend
burden trend
new recurrence groups
```

### Multi-week
```text
stable/improving/worsening trajectory
change points in daily burden
recovery toward a new physiological regime
```

---

# 52. Event database schema

Recommended schema:

```yaml
event_id:
event_version:
event_level:                  # modality | multimodal | AUD-specific
event_type:
lifecycle_state:

start:
  estimate:
  earliest:
  latest:
  detection_time:
  left_censored:

peak:
  time:
  severity:

end:
  estimate:
  earliest:
  latest:
  detection_time:
  right_censored:

duration:
usable_observation_fraction:

severity:
  peak:
  mean:
  median:
  integrated_burden:
  high_severity_minutes:
  rise_slope:
  fall_slope:
  within_event_variability:

recovery:
  start_time:
  time_to_half:
  time_to_80pct:
  time_to_expected:
  residual_at_horizon:
  curve_model:
  curve_parameters:
  interrupted_count:

support:
  child_event_ids:
  modalities:
  modality_weights:
  multimodal_agreement:

context:
  onset:
  peak:
  majority:
  recovery:

personal_reference:
  baseline_model_version:
  circadian_model_version:
  deviation_summary:

recurrence:
  recurrence_group_id:
  similarity_to_template:
  prior_similar_24h:
  prior_similar_72h:
  prior_similar_7d:

confidence:
  overall:
  signal:
  multimodal:
  persistence:
  context:
  baseline:
  missingness:
  boundary:

reason_codes: []

provenance:
  upstream_result_ids: []
  change_detector:
  parameter_version:
  temporal_model_version:

reconciliation:
  online_start:
  online_end:
  final_start:
  final_end:
  revision_reason:
```

---

# 53. Reason codes

Use machine-readable reason codes:

```text
THRESHOLD_CROSSED
CHANGE_POINT
PERSONAL_DEVIATION
MULTIMODAL_AGREEMENT
PERSISTENT
SUSTAINED
RECURRENT
ESCALATING
DE_ESCALATING
RECOVERING
SLOW_RECOVERY
INCOMPLETE_RECOVERY
TEMPORAL_CONFLICT
SHORT_GAP_BRIDGED
MISSING_DATA
LEFT_CENSORED
RIGHT_CENSORED
CONTEXT_TRANSITION
MERGED
SPLIT
NOVEL_EVENT
```

Reason codes must supplement, not replace, numeric evidence.

---

# 54. Output representation

Example live output:

```yaml
current_temporal_state:
  state: SUSTAINED_ACTIVATION
  episode_id: AUTONOMIC_EPISODE_241
  duration_s: 2280
  confidence: 0.89

episode:
  start_estimate: ...
  peak_severity: 0.82
  current_severity: 0.66
  trend: DE_ESCALATING
  integrated_burden: ...

recovery:
  state: NOT_YET_STABLE
  current_slope: -0.0041

recurrence:
  recurrence_group_id: RG_17
  similar_events_last_24h: 2
  similar_events_last_7d: 8

burden:
  episode_minutes_last_24h: 94
  high_severity_minutes_last_24h: 37

trajectory:
  3_day_episode_frequency: INCREASING
  3_day_recovery_time: WORSENING

confidence_components:
  signal: 0.96
  multimodal: 0.91
  persistence: 0.98
  context: 0.83
  baseline: 0.88
  missingness: 0.94
```

No clinical conclusion appears here.

---

# 55. Central example

## Case A

```text
HR ↑ for 25 sec
then returns to expected range
```

Output:

```text
TRANSIENT_HR_EVENT
duration = 25 s
low integrated burden
no multimodal parent episode unless another modality supports it
resolved rapidly
```

## Case B

```text
HR ↑
HRV ↓
EDA ↑
motion low
persists 40 min
then slowly recovers
```

Output:

```text
SUSTAINED_MULTIMODAL_AUTONOMIC_EPISODE
duration ≈ 40 min
multimodal support = high
integrated burden = moderate/high
recovery = delayed relative to personal episode history (if established)
```

## Case C

```text
Case-B-like pattern
every night for 3 consecutive days
recovery becomes progressively longer
```

Output:

```text
RECURRENT_MULTIMODAL_PATTERN
night-linked recurrence
episode burden increasing
recovery time worsening
3-day trajectory = adverse
```

Still **not**:

```text
withdrawal severe
relapse imminent
hospitalization needed
```

Those are downstream interpretations.

---

# 56. Final per-target temporal algorithms

The following algorithms are intended to be directly implementable. All thresholds labelled **STARTING** are calibration parameters, not medical cutoffs.

## 56.1 Generic modality event

**INPUT**

```text
feature deviation z_j(t)
feature confidence q_j(t)
context/confound-valid flag
missingness
```

**ALGORITHM**

```text
1. Reject temporal inference if upstream validity = false.
2. Update causal smooth estimate for the feature.
3. Run a feature-specific change detector.
4. If change evidence OR deviation crosses entry gate:
       open CANDIDATE_MODALITY_EVENT.
5. Accumulate:
       duration,
       occupancy,
       peak,
       AUC,
       slope,
       confidence.
6. If evidence persists beyond modality-specific D_confirm:
       mark CONFIRMED.
7. Track peak and de-escalation.
8. Enter RECOVERING only after direction reverses toward expected physiology.
9. Close after hysteretic exit + stable-return interval.
10. Finalize boundary uncertainty and missingness.
11. Link to overlapping/lag-compatible parent multimodal episode.
```

**Output**

```text
modality event, never direct AUD meaning
```

---

## 56.2 Multimodal autonomic episode

**INPUT**

```text
upstream fused physiological score/probability
patient-referenced feature deviations
modality child events
modality confidence
context
history
```

**ALGORITHM**

```text
1. Run online change detector on fused state/deviation vector.
2. Open CANDIDATE when:
       fused score crosses entry gate
       OR strong multivariate change point occurs.
3. Require trusted physiological support.
4. Associate modality events using lag-aware matching.
5. STARTING confirmation:
       >=70% qualifying occupancy over 2 min.
6. If <2 min and resolves:
       finalize TRANSIENT_MULTIMODAL_EVENT.
7. If confirmed:
       ACTIVE.
8. STARTING sustained rule:
       >=5 min -> SUSTAINED.
9. Track:
       current severity,
       peak,
       integrated burden,
       modality support,
       escalation.
10. If severity reverses and contributors return toward expected:
       RECOVERING.
11. STARTING closure:
       lower exit gate +
       >=3 min stable recovery.
12. Bridge short gaps under gap rules.
13. Finalize episode.
14. Run retrospective PELT/change-point reconciliation.
15. Compare against prior episode templates.
16. Update recurrence groups and daily burden.
```

---

## 56.3 Withdrawal-relevant autonomic trajectory

This is **not a withdrawal diagnostic algorithm**.

**INPUT**

```text
generic autonomic episodes
resting HR deviation
HRV summaries
EDA/sweating-related measures
temperature
tremor/motion features
respiration
sleep/night summaries
time since last drink if supplied downstream
```

**ALGORITHM**

```text
1. Partition observations into clinically interpretable time bins:
       hourly during suspected/known early abstinence,
       then daily as appropriate.
2. Compute usable-time-normalized autonomic burden.
3. Track modality-specific trajectories rather than forcing one score.
4. Detect sustained regime changes in:
       resting HR,
       sweating/EDA burden,
       tremor burden,
       temperature,
       respiration,
       multimodal activation.
5. Record escalation/stabilization/recovery.
6. If time-since-last-drink is known, retain it as temporal metadata.
7. Emit descriptive trajectory only.
```

**Important:** HRV/SSR should not be used alone to infer acute withdrawal severity because published findings are not uniformly supportive [13].

---

## 56.4 Cue-associated physiological episode

**INPUT**

```text
known cue/context interval
physiological deviations
subjective craving if available
```

**ALGORITHM**

```text
1. Mark cue-context onset independently from physiology.
2. Search for post-cue physiological change points.
3. Estimate latency cue -> modality response.
4. Link temporally plausible modality events.
5. Track peak, duration, and recovery.
6. Compare with control/non-cue context when available.
7. Store subjective craving alignment separately.
8. Emit CUE_ASSOCIATED_AUTONOMIC_RESPONSE,
   not CRAVING, unless downstream evidence validates that interpretation.
```

---

## 56.5 Nocturnal/post-alcohol autonomic recovery pattern

**INPUT**

```text
sleep interval
sleep-relative time
nighttime HR/HRV
other valid autonomic streams
known alcohol exposure if independently available
```

**ALGORITHM**

```text
1. Align to sleep onset.
2. Compare each segment with patient's comparable alcohol-free /
   expected sleep physiology.
3. Compute first-3-h and whole-night:
       HR burden,
       HRV suppression burden,
       autonomic episode occupancy,
       recovery fraction.
4. Detect overnight recovery or failure to recover.
5. Compare with previous nights.
6. Emit NIGHTTIME_AUTONOMIC_RECOVERY_PATTERN.
7. Attach alcohol exposure only if independently known.
```

---

## 56.6 Treatment/recovery trajectory

**INPUT**

```text
daily/nightly episode summaries
baseline/reference model
treatment-phase metadata where available
```

**ALGORITHM**

```text
1. Form daily feature vector:
       burden,
       frequency,
       mean duration,
       recovery time,
       nighttime instability,
       resting HR/HRV deviation.
2. Normalize by usable monitoring time.
3. Use robust rolling slope + uncertainty.
4. Run retrospective change-point detection on daily summaries.
5. Require repeated observations before calling a trajectory shift.
6. Emit:
       IMPROVING / STABLE / WORSENING / INSUFFICIENT_DATA
   with metric-level reason codes.
7. Do not claim treatment causality without study design supporting it.
```

---

# 57. Recurrence / burden synthesis table

| Target | Recurrence metric | Burden metric | Relevant horizon | Interpretation at this layer |
|---|---|---|---|---|
| Generic autonomic episode | similar episodes / 6 h, 24 h, 72 h, 7 d | episode minutes, AUC, high-severity minutes | hours-days | recurrence/burden only |
| Nighttime autonomic episode | episodes/night; repeated sleep-relative phase | fraction of sleep in episode; first-3-h burden | night, 3-7 nights | nighttime instability |
| HR elevation events | count and IEI by context | time/AUC above personal expected band | hours-days | persistent/recurrent HR elevation |
| HRV suppression states | nights/days with suppression | valid-window occupancy below expected | days-weeks | reduced autonomic variability state |
| EDA activation | burst count, recurring morphology | SCR/activation density, AUC | minutes-days | sympathetic arousal burden, not AUD-specific |
| Withdrawal-relevant physiology | recurrence less important than sustained trajectory | hourly multimodal/autonomic burden | 6-72 h+ | escalating/stabilizing/recovering physiology |
| Cue-associated response | responses per known cue exposure | response AUC per exposure | exposure + recovery | cue-linked physiology |
| Post-alcohol sleep disturbance | disturbed nights / drinking occasions | first-3-h and nightly recovery deficit | night-days | post-exposure physiological disturbance |
| Treatment/recovery | repeated daily phenotype | weekly median burden/recovery | weeks-months | improving/stable/worsening physiology |

---

# 58. AUD temporal evidence synthesis

| AUD process | Temporal physiological pattern supported by literature | Timescale | Evidence type | Individual-level utility | Key references |
|---|---|---:|---|---|---|
| Acute withdrawal | autonomic manifestations begin after cessation/reduction; early syndrome evolves over hours; severe manifestations have later windows | hours-days | clinical reviews + small physiology studies | timing useful; wearable-specific inference not fully validated | [10-13] |
| Withdrawal monitoring | sweating, skin temperature, tremor and activity show distinct 24-h patterns | hours/day | small ambulatory study | proof of temporal monitoring concept; small N | [11] |
| Acute drug/SUD event methodology | activation + recovery morphology can distinguish drug event from activity in free living | tens of minutes | large field/lab SUD physiology study (cocaine) | strong methodological transfer, not alcohol-specific | [3] |
| Cue/craving physiology | cue-linked autonomic changes exist but vary; subjective effects often stronger | seconds-minutes during exposure | laboratory + clinical cue studies | do not infer craving from physiology alone | [16,17,23] |
| Acute alcohol/post-alcohol autonomic effect | HR/HRV sleep recovery disturbed after drinking; dose relationship reported | first sleep hours/night | observational/experimental | strong for within-person nocturnal temporal summaries | [19] |
| AUD sleep autonomic dysfunction | elevated HR/reduced HRV during sleep, especially early night in recently detoxified AUD | night | case-control physiology | useful as descriptive nighttime phenotype | [21] |
| Abstinence/recovery | HRV/HR can recover over weeks-months; time since last drink associated with HRV | weeks-months | longitudinal/cross-sectional recovery studies | suitable for trajectory, not acute event | [12,14,20] |
| Future alcohol use association | 4-day ambulatory HRV/HR associated with subsequent 90-day abstinence | days -> months | prospective observational | promising risk feature, not imminent-relapse detector | [15] |
| Cue reactivity and relapse | cue reactivity present but did not predict relapse in one clinical study | cue exposure -> 5 wk/3 mo | prospective follow-up | argues for conservative interpretation | [16] |

---

# 59. Major research-paper extraction

`NR` means not reported or not confidently extracted from the accessible publication information used for this design. It is intentionally not guessed.

## 59.1 Sarker et al. — significant stress episodes [1]

| Field | Extraction |
|---|---|
| Population | 38 opioid-dependent polydrug users receiving opioid agonist maintenance; natural environment |
| Modalities | ECG, respiration, accelerometry; GPS/context used in analyses |
| Sampling | ECG 128 Hz; respiration 64 Hz; accelerometer 32 Hz/axis |
| Monitoring duration | ~4 weeks |
| Temporal target | significant vs momentary stress episodes |
| Base inference | cStress; 1-min physiological window; modified to output every 5 s using overlapping windows |
| Event representation | increasing trend followed by decreasing trend in stress likelihood |
| Boundary logic | MACD crossover/trend logic |
| Significance | high stress likelihood plus duration |
| Example duration criterion | 7.3 min yielded ~1 significant stress episode/day in their dataset; 13.5 min ~0.5/day; 2.4 min ~2/day |
| Missing/discontinuous data | central motivation of study; stress density and episode mining designed for discontinuous ambulatory data |
| Online/offline | method is stream-compatible; parameter tuning/analysis retrospective |
| Validation/finding | 4 weeks naturalistic data; prior episode duration predicted next episode duration; time-of-day differences |
| Limitation for this project | stress episode thresholds are use-case/sample specific; not an AUD autonomic cutoff |
| Implementation lesson | use trend boundaries + magnitude + duration; keep recurrence/history and circadian context |

## 59.2 Hovsepian et al. — cStress [2]

| Field | Extraction |
|---|---|
| Population | training lab data 21 participants; independent lab and field validation cohorts |
| Modalities | ECG + respiration; accelerometry used to identify physical activity |
| Sampling | ECG 128 Hz; respiration 21.3 Hz; accelerometer 16 Hz/axis in reported setup |
| Temporal target | continuous stress probability |
| Window | 1 min |
| Output cadence | minute-level in original model; Sarker later used overlapping updates |
| Model | feature-based stress classifier with careful quality/confound processing |
| Performance | reported 89% recall and 5% false positive rate on lab testing; field validation also reported |
| Online | yes in principle |
| Limitation | stress != AUD physiology; minute-level state classification does not itself solve episode segmentation |
| Implementation lesson | temporal event modeling should consume a validated continuous physiological state rather than raw sensors |

## 59.3 Hossain et al. — cocaine event detection [3]

| Field | Extraction |
|---|---|
| Population | 9 active drug users across 89 residential-lab days; 42 active drug users across 922 field days |
| Modalities | ECG + accelerometry |
| Monitoring | total 11,283 h reported |
| Temporal target | cocaine-use event from acute physiological response |
| Candidate segmentation | smoothed RR interval + MACD |
| Smoothing | preceding 10-min simple moving average in reported model development |
| Learned MACD windows | EMA slow ~35 min; fast ~4 min; signal ~3.67 min in their data |
| Event morphology | activation followed by recovery |
| Recovery | model separates natural PNS recovery from cocaine-dampened recovery |
| Candidate screening | insufficient magnitude/width discarded; early physical activity used for screening |
| Event duration | cocaine response can last >30 min; HR recovery described as ~20-60 min after IV cocaine |
| Personalization | person-specific recovery parameter estimated |
| Performance | 100% TPR operating point with ~0.87 false positive/day lab and ~1.13/day field |
| Online/offline | designed for event localization and potential real-time scientific use; parameters trained offline |
| Limitation | cocaine-specific physiology and supervised administrations; cannot transfer drug thresholds to AUD |
| Implementation lesson | model complete response morphology and recovery; personalized recovery can be highly informative |

## 59.4 Rosenfield et al. — physiological change points [4]

| Field | Extraction |
|---|---|
| Population | ambulatory panic-disorder study |
| Modalities | peripheral cardiorespiratory physiology |
| Temporal target | unknown physiological changes preceding naturally occurring panic attacks |
| Model | change-point analysis |
| Event definition | statistically distinct longitudinal changes, not fixed pre-labelled windows |
| Validation | changes detected in pre-panic intervals and not matched non-panic control intervals |
| Online/offline | reported method primarily retrospective |
| Performance | inferential separation rather than wearable real-time false-alarm metric |
| Limitation | panic physiology, not AUD |
| Implementation lesson | use CP methods to propose onset/transition boundaries when timing is not known a priori |

## 59.5 Adams & MacKay — BOCPD [5]

| Field | Extraction |
|---|---|
| Population | methodological paper; multiple demonstration datasets |
| Temporal target | online most-recent change point |
| Model | Bayesian online change-point detection |
| State | posterior over current run length |
| Online | yes, exact online inference for specified model |
| Duration information | run length directly represented |
| Missing data | dependent on chosen observation model/implementation |
| Limitation | not physiology-specific; hazard/observation model calibration required |
| Implementation lesson | useful causal candidate-boundary detector with explicit uncertainty |

## 59.6 Killick et al. — PELT [6]

| Field | Extraction |
|---|---|
| Target | multiple offline change points |
| Model | penalized-cost exact segmentation with pruning |
| Online | no; retrospective |
| Computational property | under stated conditions expected linear cost in number of observations |
| Limitation | cost/penalty selection determines what constitutes a change |
| Implementation lesson | excellent finalized-event boundary reconciliation after enough future data exist |

## 59.7 Yu — HSMM [8]

| Field | Extraction |
|---|---|
| Target | hidden states with explicit state durations |
| Model | hidden semi-Markov model |
| Duration | explicit/general duration distributions |
| Inference | forward-backward, filtered/smoothed probabilities, best state sequence |
| Online | possible filtered inference; computational design dependent |
| Missing data | observation-model dependent |
| Limitation | requires sufficiently stable state definitions and data to estimate durations/emissions |
| Implementation lesson | v2 option after event taxonomy and labelled corpus mature |

## 59.8 Hamidi Shishavan et al. — multimodal Bayesian CP stress [9]

| Field | Extraction |
|---|---|
| Population/data | WESAD public laboratory stress dataset |
| Modalities | multimodal physiological indices |
| Temporal target | stress-response and recovery changes |
| Model | unsupervised Bayesian change-point detection + dynamic time warping |
| Performance | reported 91% stress-period detection accuracy |
| Online/offline | method emphasizes temporal change detection; evaluate causal form separately before deployment |
| Limitation | lab stress dataset; accuracy is not event false-alarm/day; not AUD |
| Implementation lesson | personalized unsupervised CP across multiple physiological indices is feasible and interpretable |

## 59.9 White et al. — 24-h alcohol withdrawal physiology [11]

| Field | Extraction |
|---|---|
| Population | six male withdrawal subjects + age-matched controls |
| Modalities | tremor, sweating, skin temperature, locomotor activity |
| Monitoring | 24 h |
| Temporal target | temporal pattern during alcohol withdrawal |
| Findings | sweating and skin temperature greatest ~00:00-06:00; tremor/activity followed different pattern and remained higher than controls |
| Model | descriptive ambulatory monitoring |
| Limitation | very small sample; older instrumentation; not an automated event detector |
| Implementation lesson | withdrawal physiology is multimodal and time-structured; preserve hourly/circadian trajectories |

## 59.10 Bär et al. — acute withdrawal autonomic measures [13]

| Field | Extraction |
|---|---|
| Population | 20 mild AWS + 20 moderate/severe AWS male patients |
| Modalities | HRV, sympathetic skin response, ambulatory BP/HR |
| Monitoring points | withdrawal climax, one day after climax, shortly before discharge |
| Temporal target | autonomic change across acute withdrawal course |
| Finding | classical HRV and SSR did not consistently show expected autonomic dysfunction; HR was elevated at admission |
| Limitation | clinical treatment and specific cohort; not continuous wearable episode detection |
| Implementation lesson | do not treat HRV/EDA-like measures as universally monotonic withdrawal severity markers |

## 59.11 Eddie et al. — time since last drink and HRV [14]

| Field | Extraction |
|---|---|
| Population | 42 treatment-engaged adults in first year of AUD recovery |
| Modalities | HR/HRV |
| Temporal target | psychophysiological recovery with time since last drink |
| Finding | HRV increased as a function of time since last drink; strongest effects for parasympathetically mediated indices |
| Design | observational association |
| Limitation | not continuous event segmentation; does not provide minute-level recovery thresholds |
| Implementation lesson | HRV belongs in longitudinal recovery trajectory, not only acute episodes |

## 59.12 Eddie et al. — in-natura HRV and later alcohol use [15]

| Field | Extraction |
|---|---|
| Population | 42 adults in first year of AUD recovery attempt |
| Modalities | ambulatory ECG/HRV |
| Monitoring | 4 days physiology; 90-day alcohol-use follow-up |
| Temporal target | prospective association with subsequent alcohol use |
| Finding | higher parasympathetic HRV and lower HR associated with greater percent days abstinent |
| Prospective? | yes for physiology -> later drinking outcome |
| Individual event inference? | no |
| Limitation | small sample; aggregate physiology; no imminent-relapse episode detector |
| Implementation lesson | multi-day physiology can feed future risk models, but temporal engine should only provide descriptive features |

## 59.13 Witteman et al. — cue reactivity and relapse [16]

| Field | Extraction |
|---|---|
| Population | 80 alcohol-dependent patients in detoxification |
| Modalities | HRV + subjective craving during alcohol/soft-drink advertisement exposure |
| Follow-up | cue exposure diary 5 weeks; relapse assessed at 5 weeks and 3 months |
| Temporal target | cue-linked physiological/craving response and future drinking |
| Finding | alcohol cues induced physiological cue reactivity/craving; baseline magnitude did not predict relapse |
| Limitation | laboratory cue paradigm; does not establish naturalistic craving-event duration |
| Implementation lesson | link physiology to known cue context; avoid treating cue response as validated relapse predictor |

## 59.14 Pietilä et al. — alcohol and early-sleep autonomic recovery [19]

| Field | Extraction |
|---|---|
| Population | large real-world employee sample |
| Modalities | HR/HRV-derived autonomic/recovery indices |
| Window | first 3 h of sleep |
| Temporal target | acute alcohol-related autonomic disturbance during sleep |
| Finding | dose-dependent higher sympathetic / lower parasympathetic regulation and poorer recovery |
| Design | within-subject observational repeated measures |
| Limitation | not necessarily AUD population; alcohol intake self-report/context |
| Implementation lesson | early-sleep temporal summaries and within-person comparison are high-value |

## 59.15 de Zambotti et al. — abstinence recovery during sleep [20]

| Field | Extraction |
|---|---|
| Population | 15 alcohol-dependent adults; 13 controls |
| Follow-up | within a month post-detox, ~2 months, ~4 months |
| Modalities | sleep ECG/HRV + respiration-related HRV information |
| Target | recovery of autonomic function across abstinence |
| Finding | evidence of recovery in HR and high-frequency vagal modulation by ~4 months |
| Limitation | small sample; selected stable NREM windows; relapse in some participants |
| Implementation lesson | longitudinal night physiology should be tracked over months, not only event minutes |

---

# 60. Data requirements and temporal personalization

Different temporal quantities require different amounts of history.

| Quantity | Minimal use | Better estimate | Sparse-history behavior |
|---|---:|---:|---|
| onset/offset | no history if threshold/CP calibrated | several labelled episodes | retain broad boundary uncertainty |
| generic persistence | no history for engineering default | 5-10 validated episodes/type | avoid patient-specific duration category |
| recovery distribution | 3 comparable episodes only as provisional | 10+ comparable episodes | continuous metric, no FAST/SLOW label |
| recurrence rate | several days | 2-4 weeks | report raw count/IEI |
| circadian event distribution | ~7 days minimum | several weeks | avoid strong circadian abnormality claim |
| event template | 3 provisional | 10+ robust | no clustering if too sparse |
| daily burden trend | >=3 comparable days for weak direction | 7-14+ days | `INSUFFICIENT_DATA` if coverage poor |
| treatment trajectory | weeks | months | metric-level trend only |

If many patients are available, hierarchical population priors can reduce cold-start uncertainty while preserving patient-specific adaptation.

---

# 61. Event-level statistical comparison to personal history

Use robust statistics because episode descriptors are often skewed.

For event feature \(f\):

\[
z^{event}_f =
\frac{f(E_i)-median(f(E_{similar}))}
     {1.4826\cdot MAD(f(E_{similar}))+\epsilon}
\]

Report:

```text
more severe than usual
longer than usual
slower recovery than usual
more frequent than usual
```

only if:

- enough comparable events exist;
- the difference exceeds both statistical uncertainty and an engineering/clinical relevance floor.

Do not let a tiny change become “abnormal” merely because variance is numerically small.

---

# 62. Temporal logic versus probabilistic models

Temporal logic is ideal for auditable statements:

```text
IF
    multimodal_activation persists >= D_confirm
AND
    motion_confounded = false
AND
    confidence >= Q_min
THEN
    CONFIRMED_AUTONOMIC_EPISODE
```

or:

```text
IF
    similar_episode_count >= N
WITHIN
    horizon T
THEN
    RECURRING_PATTERN
```

Probabilistic models are better when:

- observations are uncertain;
- boundaries are uncertain;
- duration varies;
- multiple weak cues must be accumulated.

Recommended hybrid:

```text
probabilistic evidence generation
        +
explicit temporal lifecycle / logic
        +
probabilistic confidence
```

This is more interpretable than an end-to-end black-box sequence classifier.

---

# 63. False-alarm control

Do not optimize only for event sensitivity.

Controls:

- upstream quality gating;
- personal/context deviation rather than raw thresholds;
- persistence confirmation;
- hysteresis;
- debounce/refractory interval;
- change-point penalty/hazard calibration;
- multimodal support for parent episodes;
- confidence threshold;
- merge logic;
- usable-time normalization;
- alert-level suppression independent from event storage.

Important distinction:

```text
store event
```

does not imply:

```text
alert patient/clinician
```

The clinical alerting policy belongs downstream.

Primary engineering metric should include:

```text
false parent episodes / monitored day
```

similar to the event-oriented evaluation emphasis demonstrated by free-living drug-event work [3].

---

# 64. Refractory periods

A refractory interval is useful when the detector would otherwise rediscover the same episode tail.

Use it only at the **detector/lifecycle** level:

- after resolution, suppress low-grade retriggers for a short interval;
- allow override by a strong change point or clearly different event morphology.

Do not use long refractory windows that can hide genuine recurrent events.

Starting generic value: **1-2 min** after stable closure, calibrated by event timescale.

---

# 65. Fast versus slow physiology

The same temporal parameters must not be applied to all modalities.

### Fast
- HR/RR;
- respiration rate;
- EDA phasic response;
- motion/tremor.

### Medium
- HRV state estimates depending on metric/window;
- sustained EDA tonic state;
- multimodal autonomic activation.

### Slow
- skin temperature;
- sleep-night recovery;
- daily burden;
- abstinence/treatment trajectory.

The event association layer should therefore compare **intervals and expected lags**, not exact timestamps.

---

# 66. Temperature-specific temporal handling

Temperature should be handled as a slow channel.

Do not:

```text
HR onset at 21:00:05
temperature unchanged at 21:00:10
=> disagreement
```

Instead:

- use slow change points;
- compare multi-minute trends;
- associate with parent episodes if change occurs within learned lag;
- preserve environmental/context correction from upstream.

Withdrawal research shows skin-temperature time-of-day structure [11], reinforcing the need for slow/circadian treatment.

---

# 67. HRV-specific temporal handling

HRV values depend heavily on the estimation window and metric.

Rules:

1. store the HRV metric name and effective window with every state estimate;
2. do not compare ultra-short-term HRV with conventional longer-window estimates as if identical;
3. event onset timestamp should reflect the uncertainty introduced by the HRV window;
4. use HR/IBI events for sharper onset when appropriate;
5. use HRV as a slower supporting state.

For AUD-specific meaning, HRV evidence is strongest for broader autonomic regulation/recovery and some cue/reactivity associations rather than universally precise acute-event timing [13-16,20,23].

---

# 68. EDA-specific temporal handling

The temporal layer should receive already-decomposed tonic/phasic EDA features.

Possible event descriptors:

```text
phasic response density
response amplitude burden
tonic level deviation
rise time
decay time
activation occupancy
```

Do not re-run decomposition here.

Because EDA response timing differs from cardiac timing, lag-aware parent-event matching is mandatory.

Cue-reactivity literature supports autonomic/skin-conductance changes in some contexts but also documents substantial variability [17]; therefore EDA alone should not become an AUD semantic label.

---

# 69. Respiration-specific temporal handling

Respiration may change abruptly or persistently.

Track:

```text
respiratory_rate deviation
inspiratory/expiratory timing deviations
respiratory variability
sustained irregularity
```

Use:

- context/activity-corrected data;
- change points for sustained regime shifts;
- short event logic only where the upstream respiration feature is sufficiently stable.

cStress provides a practical example of respiration combined with ECG in minute-level stress inference [2].

---

# 70. PPG/ECG duplication rule

If ECG and PPG both measure cardiac rate/pulse timing, they are partially redundant measurements of a shared cardiovascular process.

Therefore:

```text
ECG HR event + PPG pulse-rate event
```

should increase **measurement confidence**, not automatically count as two independent physiological systems.

The fusion layer should distinguish:

- redundant sensor agreement;
- genuinely independent modality agreement (e.g. cardiac + EDA + respiration).

---

# 71. Sequence mining: later-stage capability

After an event corpus is accumulated, sequence mining can investigate patterns such as:

```text
sleep disruption
-> nighttime autonomic activation
-> incomplete recovery
-> elevated next-day resting HR
```

Candidate methods:

- frequent sequential pattern mining;
- n-gram/event transition models;
- dynamic Bayesian networks;
- HSMMs;
- temporal point-process models.

Do not deploy sequence-derived AUD interpretations until independently validated.

---

# 72. Event morphology clustering

Historical events can be clustered to discover phenotypes.

Use robust standardized features and consider HDBSCAN/GMM/hierarchical clustering before deep embeddings because:

- event count may be small;
- interpretability matters;
- cluster count may be unknown.

Possible discovered clusters:

```text
short sympathetic burst
long nighttime activation
slow-recovery episode
```

These are data-driven labels only.

Cluster stability must be tested by bootstrap/resampling before a phenotype becomes a persistent template.

---

# 73. Episode-to-trajectory aggregation

Aggregation should be hierarchical:

```text
event
  |
  v
episode descriptors
  |
  v
daily/nightly burden vector
  |
  v
rolling 3-7 d summary
  |
  v
multi-week trajectory
```

Do not average raw second-level samples directly into a weekly risk score.

Episode aggregation preserves:

- frequency;
- duration;
- severity;
- recovery;
- recurrence phenotype.

---

# 74. Long-term change-point detection

Daily burden itself can undergo regime changes:

```text
low/stable burden
      |
      | CP
      v
persistently higher burden
```

Run retrospective PELT and/or online CUSUM/BOCPD on daily/nightly summary metrics.

Important safeguard:

A personalized baseline model that adapts too quickly could **absorb persistent deterioration** and make the deviation disappear. Therefore retain:

- a slow reference baseline;
- a recent adaptive baseline;
- raw daily summary history.

Temporal trajectory analysis should compare both.

---

# 75. Failure modes and mitigation

| Failure mode | What goes wrong | Mitigation |
|---|---|---|
| Threshold chatter | many tiny enter/exit events | hysteresis + persistence + debounce |
| Episode fragmentation | one event split by brief normalization | gap tolerance + stable-recovery requirement |
| Episode over-merging | distinct events become one long episode | change points + stable baseline + context transitions |
| Missing data = normal | dropout falsely ends event | explicit UNKNOWN state |
| Missing data = infinite event | episode never closes | censoring and maximum evidence gap |
| Circadian recurrence mistaken for pathology | normal nightly rhythm labelled recurring episode | expected circadian residuals |
| Multimodal latency mistaken for disagreement | EDA/temperature lag cardiac signal | learned lag-aware association |
| Redundant sensors double-counted | ECG + PPG treated as two independent systems | sensor-redundancy metadata |
| Recovery misclassified as new event | oscillatory fall crosses detector | lifecycle state + refractory logic |
| Re-escalation hidden | recovery tail merged indefinitely | new strong CP overrides refractory |
| Baseline adapts away deterioration | persistent shift becomes “new normal” too early | slow + fast baselines; trajectory on absolute summaries |
| Future leakage | retrospective info boosts online model | causal feature contract |
| Window leakage | same episode in train/test | patient/event-level split |
| Long wear time inflates counts | more events because more observed time | usable-time normalization |
| Severity conflated with confidence | large event with poor sensor looks “certain” | separate severity and confidence |
| One modality dominates parent event | parent inherits noisy feature | reliability-weighted support + modality caps |
| Cue response called craving | physiological arousal overinterpreted | context-linked descriptive label |
| HRV called withdrawal severity | literature inconsistency ignored | multimodal trajectory + clinical downstream layer |
| Burden double-counting | child and parent events both summed | parent-child accounting |
| Clock-time recurrence confused with sleep-phase recurrence | sleep timing shifts | sleep-relative phase |
| Unknown event forced into existing cluster | novelty hidden | explicit novelty score / unassigned state |
| Censored duration treated exact | biased duration model | censoring flags/survival-aware analysis |

---

# 76. Implementation data structures

Suggested runtime structures:

```python
TemporalObservation:
    timestamp
    stream_id
    value
    deviation
    confidence
    context
    missing

ChangeEvidence:
    timestamp
    detector
    probability_or_score
    direction
    affected_dimensions

ActiveEvent:
    event_id
    lifecycle_state
    candidate_start
    support
    severity_accumulator
    gap_state
    recovery_state

FinalizedEvent:
    immutable_versioned_event_record

RecurrenceGroup:
    template
    event_ids
    feature_distribution
    inter_event_distribution

BurdenSummary:
    horizon
    usable_time
    counts
    occupancy
    AUC
    recovery_stats

TrajectoryState:
    metric
    slope
    change_points
    uncertainty
    label
```

This separation prevents the active tracker from becoming a large mutable object containing longitudinal logic.

---

# 77. Processing contracts between layers

## Upstream must provide

```text
timestamp
validity/confidence
patient-referenced deviation
context/circadian expectation
confound-corrected status
multimodal fused state if available
```

## Temporal layer provides downstream

```text
current event state
finalized episodes
episode morphology
recovery descriptors
recurrence groups
burden
trajectory
confidence
provenance
```

## AUD-specific layer adds

```text
AUD hypothesis
AUD-specific state probability
clinical meaning
```

The temporal layer must not overwrite upstream confidence or invent AUD labels.

---

# 78. Recommended implementation parameter registry

All temporal parameters should be versioned, never scattered through code.

Example:

```yaml
temporal_model_version: 1.0

generic_autonomic:
  enter_threshold: 0.68          # STARTING, calibration required
  exit_threshold: 0.40           # STARTING
  confirm_window_s: 120          # STARTING
  qualifying_fraction: 0.70      # STARTING
  sustained_duration_s: 300      # STARTING
  recovery_stable_s: 180         # STARTING
  refractory_s: 90               # STARTING

gaps:
  auto_bridge_s: 30
  bridge_with_other_support_s: 120
  max_active_unknown_s: 300

merge:
  auto_merge_gap_s: 120
  conditional_merge_max_gap_s: 600
  conditional_similarity: 0.80   # STARTING

recurrence:
  similarity_threshold: 0.78     # STARTING
  horizons_s: [21600, 86400, 259200, 604800]

trajectory:
  minimum_comparable_days: 3
  preferred_window_days: 7
```

These values exist to make v1 runnable. They must be tuned with an event-labelled development set and frozen before test evaluation.

---

# 79. Calibration strategy

Do not tune thresholds to maximize window accuracy.

Tune jointly for:

1. event recall;
2. false parent episodes/day;
3. onset delay;
4. duration error;
5. burden error.

Suggested process:

```text
development subjects
    |
grid/Bayesian calibration
    |
choose operating region
    |
freeze thresholds
    |
patient-separated test subjects
```

For personalized parameters, simulate cold-start:

```text
day 1 -> day N
```

without using future history.

---

# 80. Acceptance tests

The engine should pass unit/integration scenarios before patient data.

### Scenario 1: 25-s spike
Expected:
`TRANSIENT`, no sustained parent episode.

### Scenario 2: 40-min coherent multimodal activation
Expected:
candidate -> confirmed -> sustained -> recovering -> resolved.

### Scenario 3: 10-s internal normalization
Expected:
one episode, gap bridged.

### Scenario 4: 8-min full recovery then new activation
Expected:
two episodes.

### Scenario 5: EDA unavailable for 90 s, HR/HRV remain supportive
Expected:
same parent episode with lower confidence.

### Scenario 6: all sensors unavailable for 10 min
Expected:
right-censored event; new event on return unless reconciled offline.

### Scenario 7: ECG and PPG agree but EDA/respiration neutral
Expected:
stronger cardiac measurement confidence, not “two-modality autonomic evidence.”

### Scenario 8: same episode 3 nights, progressively longer recovery
Expected:
one recurrence group + worsening recovery trajectory.

### Scenario 9: regular circadian HR rise predicted by personal baseline
Expected:
no abnormal recurrent episode.

### Scenario 10: offline boundary refinement
Expected:
event version updated, online timestamps preserved.

---

# 81. Final synthesis: why this architecture is stronger than alternatives

A fixed-window classifier answers:

```text
what class is this window?
```

The required system must answer:

```text
when did the process begin?
did it persist?
how severe was the full episode?
did it recover?
how long did recovery take?
did it recur?
is recurrence becoming more frequent?
are similar episodes lasting longer?
is daily burden changing?
```

A pure HMM/HSMM can model some of these but is difficult to justify before the state taxonomy and duration distributions are validated.

A pure change-point model identifies regime boundaries but does not by itself define physiological significance, recovery, recurrence or burden.

A pure rule system is auditable but weak at uncertainty and unknown boundaries.

The recommended hybrid assigns each method the job it is best at.

---

# Recommended Implementation

This section is the **single consolidated temporal physiological event-modeling system to build**.

## Temporal hierarchy

Implement exactly these primary levels:

```text
1. WINDOW
2. MODALITY EVENT
3. MULTIMODAL EPISODE
4. RECURRENCE GROUP
5. DAILY/NIGHTLY BURDEN
6. LONGITUDINAL TRAJECTORY
```

AUD-specific episodes remain downstream.

## Event onset

At every update:

1. consume patient/context-referenced deviations and confidence;
2. update fused physiological state;
3. run BOCPD or calibrated CUSUM on the fused/deviation stream;
4. open `CANDIDATE` at the earliest credible change/entry boundary;
5. retain onset uncertainty.

Do not require all modalities to cross simultaneously.

## Event confirmation

A candidate becomes confirmed when it satisfies target-specific persistence.

**v1 generic starting rule:**

```text
>=70% qualifying occupancy over 2 min
```

plus adequate upstream confidence.

For a multimodal parent episode, require either:

- at least two meaningfully independent physiological systems, or
- a high-confidence fused state whose model already accounts for modality reliability.

## Minimum persistence

Use target-specific persistence.

Generic v1:

```text
<2 min          -> transient unless target model says otherwise
>=2 min         -> confirmed
>=5 min         -> sustained
```

Never call these clinical cutoffs.

## Event offset

Enter recovery when:

- peak has passed;
- severity trend is moving toward expected physiology;
- no independent reactivation change point exists.

Close after:

```text
score below exit gate
+
majority of contributing modalities in expected band
+
3 min stable recovery (v1 starting value)
```

## Hysteresis

Use separate entry/exit gates.

Starting normalized gates:

```text
enter ~0.65-0.70
exit  ~0.35-0.45
```

Calibrate on event-level performance.

## Gap handling

```text
<=30 s:
    bridge automatically if evidence matches

30 s-2 min:
    bridge if another trusted modality supports continuity

2-5 min:
    keep ACTIVE_WITH_GAP; reduce confidence; boundaries uncertain

>5 min:
    right-censor and require new candidate after return
    unless offline reconciliation later merges
```

## Episode merging/splitting

Merge automatically when:

```text
gap <=2 min
AND no stable recovery occurred
```

For 2-10 min gaps, merge only with high morphology similarity, residual deviation and no incompatible context/change point.

Split when:

- stable baseline is reached;
- a strong distinct change point appears;
- context + physiology jointly indicate a new process;
- a clearly separate morphology appears.

## Severity

Store vector:

```text
peak
mean
median
AUC
high-severity occupancy
rise rate
fall rate
time-to-peak
within-event variability
```

Do not rely on peak alone.

## Persistence

Store:

```text
duration
qualifying occupancy
longest continuous qualifying run
AUC
usable-data fraction
```

## Escalation/de-escalation

Use robust recent slope + change points + cross-modal direction.

Emit:

```text
ESCALATING
STABLE_WITHIN_EPISODE
DE_ESCALATING
```

with numeric slope/evidence.

## Recovery

Track:

```text
recovery_start
time_to_half
time_to_80%
time_to_expected
residual deviation
recovery slope
interrupted recovery
```

Fit an exponential only if data support it. Always retain non-parametric metrics.

## Recurrence

Represent episodes by morphology/context vectors.

Group only if similarity is high enough.

Track:

```text
count/6h
count/24h
count/72h
count/7d
inter-event intervals
night-linked recurrence
```

## Burden

For each horizon store:

```text
episode count
episode minutes
fraction usable time in episode
AUC
high-severity minutes
mean/max severity
mean recovery
incomplete recovery count
nighttime burden
```

Count the multimodal parent once.

## Multi-timescale state

Maintain separate:

```text
fast change channel
episode channel
daily/nightly burden channel
longitudinal trajectory channel
```

Never force seconds and weeks into one model state.

## Personalization

Personalize:

- entry/exit expected deviation distribution;
- typical event duration;
- typical recovery;
- event morphology/template;
- recurrence rate;
- circadian event distribution;
- modality lag distributions.

Do not personalize before enough data exist; expose uncertainty.

## Multimodal timing

Link child modality events with:

- onset uncertainty;
- expected lag;
- overlap;
- morphology;
- context.

Do not demand exact simultaneity.

## Missing data

Missing = `UNKNOWN`.

Never treat it as baseline.

Keep parent episode if independent modalities support it; otherwise use censoring.

## Confidence

Compute separate evidence-confidence components and aggregate conservatively.

Confidence can rise with persistence but must fall with:

- signal loss;
- modality disagreement;
- uncertain context;
- poor baseline certainty;
- large unknown intervals.

## Online/offline dual path

Online:
- causal BOCPD/CUSUM + lifecycle.

Offline:
- PELT/change-point reconciliation;
- merge/split;
- final boundaries;
- burden correction.

Store both event versions.

## Output schema

Emit:

```text
current temporal state
event object
recovery state
recurrence state
burden summaries
trajectory summaries
confidence
reason codes
provenance
```

No AUD diagnosis is emitted from this layer.

---

# 83. Concrete software flow

```text
for each timestamp t:

    observations = read_upstream_state(t)

    if observations.invalid:
        update_missingness()
        continue

    update_modality_trackers(observations)

    fused = read_or_update_fused_state(observations)

    cp = change_detector.update(fused)

    active_event.update(
        score=fused.score,
        change_evidence=cp,
        child_events=modality_events,
        context=observations.context,
        confidence=observations.confidence
    )

    if active_event.closed:
        finalized = reconcile_recent_event(active_event)
        event_store.write(finalized)

        recurrence_model.update(finalized)
        burden_model.update(finalized)

periodically:
    burden_model.finalize_hour_day_night()
    trajectory_model.update()

retrospectively:
    run_offline_change_points()
    reconcile_boundaries()
    recompute_burden_if_event_versions_changed()
```

---

# 84. Research/design boundaries

This system **can** conclude:

```text
sustained multimodal autonomic activation lasting 42 minutes
```

```text
three similar nighttime episodes within 72 hours
```

```text
episode duration and recovery time progressively increased
across three comparable nights
```

```text
7-day physiological episode burden is worsening
```

It **cannot**, by itself, conclude:

```text
withdrawal is severe
craving is present
relapse is imminent
medication is required
hospitalization is necessary
```

Those are AUD-specific or clinical inferences.

---

# 85. Implementation roadmap

## Phase 1 — deterministic/interpretable core

Implement:

- event schema;
- state lifecycle;
- hysteresis;
- persistence;
- gap handling;
- recovery descriptors;
- parent-child hierarchy;
- burden summaries;
- offline reconciliation hooks.

## Phase 2 — calibrated change detection

Add:

- BOCPD/CUSUM candidate boundary detector;
- PELT retrospective pass;
- event-level threshold calibration.

## Phase 3 — personalization

Add:

- event templates;
- recovery distributions;
- personalized recurrence rate;
- circadian recurrence correction;
- learned lag distributions.

## Phase 4 — validated AUD temporal phenotypes

Only after labelled studies:

- withdrawal trajectory model;
- cue-associated episode model;
- sleep/post-alcohol phenotype;
- treatment trajectory;
- risk-model features.

## Phase 5 — learned explicit-duration model if justified

Evaluate HSMM/switching state-space models against the interpretable baseline.

Adopt only if they materially improve:

- event recall;
- false events/day;
- boundary accuracy;
- duration/recovery estimation;
- calibration.

---

# 86. Requirements coverage checklist

The design above covers the requested problem categories as follows:

| Requested topic | Where addressed |
|---|---|
| temporal objects | Sections 2-3 |
| onset/offset | 6-9 |
| persistence/severity | 7, 10 |
| transient vs sustained | 7 |
| hysteresis | 8 |
| gap tolerance | 11 |
| segmentation / CP | 12, 33 |
| duration-aware models | 14-15 |
| state transitions | 16 |
| recovery / curves | 17-18 |
| escalation/de-escalation | 19 |
| recurrence / interval | 20-22 |
| clustering / burden | 22-23, 72 |
| repeated night/day | 24, 44 |
| circadian-aware | 24 |
| event-context linkage | 25 |
| event similarity/templates | 20-21 |
| sequence / precedence | 27, 71 |
| lag-aware multimodal | 26 |
| multi-timescale | 5, 51 |
| confidence | 30 |
| provisional/confirmed states | 7, 30 |
| missing data / censoring | 11, 13 |
| merging/splitting | 12 |
| deterioration / recurrence escalation | 47-49 |
| recovery deterioration | 48 |
| event burden trajectory | 47 |
| AUD processes | 41-46 |
| temporal logic | 62 |
| change-point methods | 33 |
| sequential tests | 34 |
| hidden-state models | 14-16 |
| state-space models | 35 |
| deep learning | 36 |
| leakage | 37 |
| online vs offline | 31 |
| event reconciliation | 32 |
| database/schema | 52 |
| lifecycle | 7 |
| reason codes | 53 |
| output | 54 |
| multi-resolution summaries | 51 |
| history comparison | 61 |
| novel events | 50 |
| event clustering | 72 |
| episode-to-trajectory | 73 |
| double-counting | 28 |
| hierarchy | 28 |
| provenance | 29 |
| uncertainty | 38 |
| false alarm / refractory | 63-64 |
| temporal personalization | 21, 60-61 |
| data requirements | 60 |
| paper extraction | 59 |
| AUD-specific extraction | 59 + 58 |
| mathematical formulation | 3 |
| failure modes | 75 |
| final algorithms | 56 |
| synthesis tables | 2, 15, 40, 57, 58 |
| final consolidated system | Recommended Implementation |

---

# References

[1] Sarker H, Tyburski M, Rahman MM, Hovsepian K, Sharmin M, Epstein DH, Preston KL, Furr-Holden CD, Milam A, Nahum-Shani I, al'Absi M, Kumar S.  
**“Finding Significant Stress Episodes in a Discontinuous Time Series of Rapidly Varying Mobile Sensor Data.”**  
Proceedings of the 2016 CHI Conference on Human Factors in Computing Systems, 2016:4489-4501.  
DOI: 10.1145/2858036.2858218. PMID: 28058409. PMCID: PMC5207658.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC5207658/

[2] Hovsepian K, al'Absi M, Ertin E, Kamarck T, Nakajima M, Kumar S.  
**“cStress: Towards a Gold Standard for Continuous Stress Assessment in the Mobile Environment.”**  
Proceedings of ACM UbiComp, 2015:493-504.  
DOI: 10.1145/2750858.2807526. PMID: 26543926. PMCID: PMC4631393.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC4631393/

[3] Hossain SM, Ali AA, Rahman MM, Ertin E, Epstein DH, Kennedy A, Preston K, Umbricht A, Chen Y, Kumar S.  
**“Identifying Drug (Cocaine) Intake Events from Acute Physiological Response in the Presence of Free-living Physical Activity.”**  
Proceedings of the 13th International Symposium on Information Processing in Sensor Networks (IPSN), 2014:71-82.  
DOI: 10.1109/IPSN.2014.6846742. PMID: 25531010. PMCID: PMC4269159.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC4269159/

[4] Rosenfield D, Zhou E, Wilhelm FH, Conrad A, Roth WT, Meuret AE.  
**“Change Point Analysis for Longitudinal Physiological Data: Detection of Cardio-respiratory Changes Preceding Panic Attacks.”**  
Biological Psychology. 2010;84(1):112-120.  
DOI: 10.1016/j.biopsycho.2010.01.020. PMID: 20144682. PMCID: PMC2978268.  
URL: https://pubmed.ncbi.nlm.nih.gov/20144682/

[5] Adams RP, MacKay DJC.  
**“Bayesian Online Changepoint Detection.”**  
arXiv:0710.3742, 2007.  
DOI: 10.48550/arXiv.0710.3742.  
URL: https://arxiv.org/abs/0710.3742

[6] Killick R, Fearnhead P, Eckley IA.  
**“Optimal Detection of Changepoints With a Linear Computational Cost.”**  
Journal of the American Statistical Association. 2012;107(500):1590-1598.  
DOI: 10.1080/01621459.2012.737745.  
URL: https://doi.org/10.1080/01621459.2012.737745

[7] Truong C, Oudre L, Vayatis N.  
**“Selective Review of Offline Change Point Detection Methods.”**  
Signal Processing. 2020;167:107299.  
DOI: 10.1016/j.sigpro.2019.107299.  
URL: https://www.sciencedirect.com/science/article/pii/S0165168419303494

[8] Yu SZ.  
**“Hidden Semi-Markov Models.”**  
Artificial Intelligence. 2010;174(2):215-243.  
DOI: 10.1016/j.artint.2009.11.011.  
URL: https://www.sciencedirect.com/science/article/pii/S0004370209001416

[9] Hamidi Shishavan H, Gossett E, Bi J, Henning R, Cherniack M, Kim I.  
**“Unsupervised Bayesian Change Point Detection Model to Track Acute Stress Responses.”**  
Biomedical Signal Processing and Control. 2024;95:106415.  
DOI: 10.1016/j.bspc.2024.106415.  
URL: https://www.sciencedirect.com/science/article/abs/pii/S1746809424004737

[10] Jesse S, Bråthen G, Ferrara M, Keindl M, Ben-Menachem E, Tanasescu R, Brodtkorb E, Hillbom M, Leone MA, Ludolph AC.  
**“Alcohol Withdrawal Syndrome: Mechanisms, Manifestations, and Management.”**  
Acta Neurologica Scandinavica. 2017;135(1):4-16.  
DOI: 10.1111/ane.12671. PMID: 27586815. PMCID: PMC6084325.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC6084325/

[11] White JM, Frewin CR, Kaur M, Flavel S, McGregor C.  
**“Twenty-four Hour Ambulatory Monitoring of Tremor, Sweating, Skin Temperature and Locomotor Activity in the Alcohol Withdrawal Syndrome.”**  
Clinical Autonomic Research. 1994;4(1-2):15-18.  
DOI: 10.1007/BF01828833. PMID: 8054832.  
URL: https://pubmed.ncbi.nlm.nih.gov/8054832/

[12] Weise F, Müller D, Krell D, Kielstein V, Koch RD.  
**“Heart Rate Variability of Chronic Alcoholics in Withdrawal and Abstinence.”**  
Clinical Neurology and Neurosurgery. 1985;87(2):95-98.  
DOI: 10.1016/0303-8467(85)90104-0. PMID: 4028594.  
URL: https://pubmed.ncbi.nlm.nih.gov/4028594/

[13] Bär KJ, Boettger MK, Neubauer R, Grotelüschen M, Jochum T, Baier V, Sauer H, Voss A.  
**“Heart Rate Variability and Sympathetic Skin Response in Male Patients Suffering from Acute Alcohol Withdrawal Syndrome.”**  
Alcoholism: Clinical and Experimental Research. 2006;30(9):1592-1598.  
DOI: 10.1111/j.1530-0277.2006.00191.x. PMID: 16930222.  
URL: https://pubmed.ncbi.nlm.nih.gov/16930222/

[14] Eddie D, Pietrzak A, Ham J.  
**“Time Since Last Drink is Positively Associated with Heart Rate Variability in Outpatients with Alcohol Use Disorder: Further Evidence of Psychophysiological Recovery in Early Alcohol Use Disorder Recovery.”**  
Applied Psychophysiology and Biofeedback. 2023;48(4):433-437.  
DOI: 10.1007/s10484-023-09597-z. PMID: 37436518.  
URL: https://pubmed.ncbi.nlm.nih.gov/37436518/

[15] Eddie D, Wieman S, Pietrzak A, Zhai X.  
**“In Natura Heart Rate Variability Predicts Subsequent Alcohol Use in Individuals in Early Recovery from Alcohol Use Disorder.”**  
Addiction Biology. 2023;28(8):e13306.  
DOI: 10.1111/adb.13306. PMID: 37500488. PMCID: PMC10433643.  
URL: https://pubmed.ncbi.nlm.nih.gov/37500488/

[16] Witteman J, Post H, Tarvainen M, de Bruijn A, De Sousa Fernandes Perna E, Ramaekers JG, Wiers RW.  
**“Cue Reactivity and its Relation to Craving and Relapse in Alcohol Dependence: A Combined Laboratory and Field Study.”**  
Psychopharmacology. 2015;232(20):3685-3696.  
DOI: 10.1007/s00213-015-4027-6. PMID: 26257163. PMCID: PMC4562995.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC4562995/

[17] Reid MS, Flammino F, Starosta A, Palamar J, Franck J.  
**“Physiological and Subjective Responding to Alcohol Cue Exposure in Alcoholics and Control Subjects: Evidence for Appetitive Responding.”**  
Journal of Neural Transmission. 2006;113(10):1519-1535.  
DOI: 10.1007/s00702-005-0439-5. PMID: 16604310.  
URL: https://pubmed.ncbi.nlm.nih.gov/16604310/

[18] Fairbairn CE, Rosen IG, Luczak SE, Venerable WJ.  
**“Estimating the Quantity and Time Course of Alcohol Consumption from Transdermal Alcohol Sensor Data: A Combined Laboratory-Ambulatory Study.”**  
Alcohol. 2019;81:111-116.  
DOI: 10.1016/j.alcohol.2018.08.015. PMID: 30179707. PMCID: PMC6395533.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC6395533/

[19] Pietilä J, Helander E, Korhonen I, Myllymäki T, Kujala UM, Lindholm H.  
**“Acute Effect of Alcohol Intake on Cardiovascular Autonomic Regulation During the First Hours of Sleep in a Large Real-World Sample of Finnish Employees: Observational Study.”**  
JMIR Mental Health. 2018;5(1):e23.  
DOI: 10.2196/mental.9519. PMID: 29549064.  
URL: https://pubmed.ncbi.nlm.nih.gov/29549064/

[20] de Zambotti M, Willoughby AR, Baker FC, Sugarbaker DS, Colrain IM.  
**“Cardiac Autonomic Function During Sleep: Effects of Alcohol Dependence and Evidence of Partial Recovery with Abstinence.”**  
Alcohol. 2015;49(4):409-415.  
DOI: 10.1016/j.alcohol.2014.07.023. PMID: 25957854. PMCID: PMC4465836.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC4465836/

[21] de Zambotti M, Baker FC, Sugarbaker DS, Nicholas CL, Trinder J, Colrain IM.  
**“Poor Autonomic Nervous System Functioning During Sleep in Recently Detoxified Alcohol-Dependent Men and Women.”**  
Alcoholism: Clinical and Experimental Research. 2014.  
PMID: 24575956. PMCID: PMC3999208.  
URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC3999208/

[22] Quintana DS, Guastella AJ, McGregor IS, Hickie IB, Kemp AH.  
**“Heart Rate Variability Predicts Alcohol Craving in Alcohol Dependent Outpatients: Further Evidence for HRV as a Psychophysiological Marker of Self-Regulation.”**  
Drug and Alcohol Dependence. 2013;132(1-2):395-398.  
DOI: 10.1016/j.drugalcdep.2013.02.025. PMID: 23601925.  
URL: https://pubmed.ncbi.nlm.nih.gov/23601925/

[23] Ralevski E, Petrakis I, Altemus M.  
**“Heart Rate Variability in Alcohol Use: A Review.”**  
Pharmacology Biochemistry and Behavior. 2019;176:83-92.  
DOI: 10.1016/j.pbb.2018.12.003. PMID: 30529588.  
URL: https://pubmed.ncbi.nlm.nih.gov/30529588/

[24] Schmidt P, Reiss A, Duerichen R, Marberger C, Van Laerhoven K.  
**“Introducing WESAD, a Multimodal Dataset for Wearable Stress and Affect Detection.”**  
Proceedings of the 20th ACM International Conference on Multimodal Interaction (ICMI), 2018:400-408.  
DOI: 10.1145/3242969.3242985.  
Dataset: https://archive.ics.uci.edu/dataset/465/wesad+wearable+stress+and+affect+detection

[25] Day E, Daly C.  
**“Clinical Management of the Alcohol Withdrawal Syndrome.”**  
Addiction. 2022;117(3):804-814.  
DOI: 10.1111/add.15647. PMID: 34288186.  
URL: https://pubmed.ncbi.nlm.nih.gov/34288186/

---

## Final conclusion

The key object this system should produce is **not a smoothed physiological value**. It is a versioned, evidence-linked temporal episode:

```text
what changed
when it began
how confidently it began
how long it persisted
how severe it became
which modalities supported it
whether it escalated
how it recovered
whether recovery was complete
whether a similar episode recurred
how total burden is changing
```

That is the correct bridge between trustworthy multimodal physiology and the later AUD-specific question:

> **“What does this evolving physiological pattern mean for this patient’s AUD/SUD condition?”**
