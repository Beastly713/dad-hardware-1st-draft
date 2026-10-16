# Final End-to-End Physiological Monitoring System for AUD/SUD

**Status:** authoritative system design  
**Evidence cut-off:** 17 October 2026  
**Repository:** `Beastly713/dad-hardware-1st-draft`

---

## Executive System Definition

This system is a **stateful, provenance-preserving physiological evidence graph** for continuous AUD/SUD monitoring. It begins with synchronized multimodal measurements, determines which measurements are technically usable, extracts physiologically meaningful components, separates physical confounds from residual physiology where the data permit that separation, interprets observations against context and the individual patient's own expected physiology, reconciles redundant and complementary modalities without double-counting, structures changes into temporal episodes and longitudinal trajectories, applies target-specific AUD semantics only when the required context and scientific evidence exist, and finally limits clinician-facing language to the strongest statement that the evidence is authorized to support.

The design is deliberately not a single classifier and not a strictly linear software pipeline. Its epistemic order is fixed even though its computational dependencies form a directed, stateful graph:

```text
BODY
  ↓
SENSING / ACQUISITION
  ↓
ACQUISITION INTEGRITY
  ↓
SIGNAL TRUSTWORTHINESS ──────────────┐
  ↓                                  │
PHYSIOLOGICAL DECOMPOSITION          │
  ↓                                  │
PHYSICAL-CONFOUND RESOLUTION         │
  ↓                                  │
CONTEXT-DEFINED EXPECTATION          │
  ↓                                  │
PATIENT-RELATIVE INTERPRETATION      │
  ↓                                  │
DEPENDENCY-AWARE MULTIMODAL EVIDENCE │
  ↓                                  │
TEMPORAL EPISODES / TRAJECTORIES     │
  ↓                                  │
TARGET-SPECIFIC AUD EVIDENCE         │
  ↓                                  │
EVIDENCE / AUTHORITY CEILING         │
  ↓                                  │
CLINICIAN-FACING INTERPRETATION      │
                                     │
      provenance + validity +        │
      missingness + conflicts ───────┘
```

Two guarded feedback relationships are part of the finished system:

1. the **personal expectation model** supplies context- and patient-relative residuals to event detection, while active or unresolved abnormal episodes supply an *exclusion mask* that prevents those intervals from silently redefining the stable baseline; and
2. the **temporal engine** may retrospectively refine episode boundaries, but it may not retroactively make invalid measurements valid or raise clinical authority beyond the evidence available for the claim.

The central invariant is:

```text
measurement trustworthy
≠ physiology meaningful
≠ physiology patient-abnormal
≠ physiology AUD-specific
≠ clinical conclusion justified
≠ clinical action beneficial
```

Each `≠` is implemented as a separate evidence boundary.

---

# 1. Scope and Intended Inference

The system is designed to support the following classes of output:

- technically valid physiological measurements and derived features;
- context-conditioned, patient-relative physiological deviations;
- generic physiological states such as sustained autonomic activation, recovery, or tremor;
- longitudinal physiological burden and recovery trajectories;
- conservative, target-specific AUD compatibility statements where the required context is independently known;
- claim-specific abstention when evidence is missing, contradictory, out of domain, weakly validated, or clinically nonspecific.

The system is **not** authorized by its present evidence base to infer, from physiology alone:

- an AUD diagnosis;
- subjective craving presence or intensity;
- exact BAC/BrAC;
- legal impairment;
- definitive alcohol withdrawal;
- a full CIWA-Ar score;
- impending seizure or delirium tremens;
- imminent relapse;
- treatment success;
- medication need or treatment action.

Those restrictions are not unfinished research items. They are part of the final inference behavior unless and until the evidence registry for a precisely defined claim is upgraded by appropriate validation.

---

# 2. System-Wide Invariants

The following rules are hard system invariants.

1. **Invalid measurement cannot regain authority downstream.** A later model cannot make clipped, unsynchronized, disconnected, or otherwise unusable data valid.
2. **Missing evidence is not negative evidence.** Absence of ECG, EDA, context, or a clinical fact cannot be silently replaced with a normal value.
3. **Abnormal physiology is not signal failure.** A physiologically unusual waveform is not rejected merely because it is unusual.
4. **Slow physiology is not automatically noise.** Tonic EDA, temperature drift, circadian cardiac variation, and long recovery components are retained when biologically meaningful.
5. **Measurement artifact and confound-driven physiology are distinct.** Motion-corrupted PPG is a measurement problem; a genuine exercise-induced heart-rate rise is real physiology explained by a confound.
6. **Physical confounds and interpretive context have different owners.** Activity, posture, respiratory state, and thermal environment can physically alter physiology; sleep/wake, circadian phase, known cue exposure, treatment state, and abstinence interval select the expectation or interpretation.
7. **Context must not be inferred circularly from the same physiology it is then used to explain**, unless a formally defined joint model explicitly represents that dependence.
8. **Redundant sensors are not independent votes.** ECG heart rate and PPG pulse rate can corroborate cardiac timing quality but do not count as two independent autonomic pathways.
9. **High-quality disagreement is evidence.** It is stored as conflict and is never averaged away merely to produce a cleaner state.
10. **Patient-relative abnormality does not imply pathology.** A large personal deviation is an observation, not a diagnosis.
11. **Generic autonomic activation does not imply AUD.** Strong cardiac plus electrodermal activation can remain A3 physiological evidence with no AUD attribution.
12. **AUD association does not imply diagnosis.** Group or longitudinal associations cap language at compatibility/association unless individual diagnostic validity exists.
13. **High signal quality cannot overcome weak target specificity.** Excellent ECG does not make nonspecific tachycardia diagnostic of withdrawal.
14. **Population evidence cannot silently become calibrated individual probability.** P-values, effect sizes, correlations, AUROC, and feature importance are not patient probabilities.
15. **A sustained abnormal interval cannot silently redefine the stable personal baseline.** Active episodes and unresolved regime shifts are excluded from stable-baseline adaptation.
16. **Device-regime changes cannot silently become patient physiology.** Electrode placement, firmware, sampling, sensor replacement, gain, calibration, and mechanical changes create explicit measurement-regime boundaries.
17. **Public/reference datasets cannot establish target-hardware equivalence.** They establish independent physiology and algorithmic behavior; the target device must be characterized on its own data.
18. **Target-hardware data cannot by itself establish clinical validity.** Device-specific reproducibility does not prove AUD specificity or treatment utility.
19. **Synthetic data cannot establish physiological or clinical truth.** It may test software or mathematical behavior, not biological validity.
20. **A probability is prohibited unless its target, horizon, population, reference standard, external validity, and calibration are defined.**
21. **Abstention is a valid final result.** `INSUFFICIENT_EVIDENCE`, `AMBIGUOUS`, `MIXED_EVIDENCE`, `OUT_OF_DOMAIN`, and `PROBABILITY_NOT_VALIDATED` are completed inference states.
22. **Retrospective refinement cannot leak into live decisions.** Online processing is causal; retrospective processing is separately versioned.
23. **Episode burden is counted at the parent physiological-process level.** Child modality events are evidence, not additional copies of the same burden.
24. **A hardware-specific observable may be used when it solves a real inference problem, even if public datasets lack it; its meaning must first be established on the target device.**

---

# 3. Complete End-to-End Architecture

The computational architecture consists of nine interacting scientific subsystems.

| Subsystem | Owns | Does not own |
|---|---|---|
| Acquisition integrity | clocks, sequence, packet/storage integrity, sensor state, configuration provenance | physiological plausibility |
| Signal trustworthiness | feature-specific technical usability and uncertainty | cause or clinical meaning |
| Physiological decomposition | beat/pulse/tonic/phasic/thermal/motion representations | confound attribution |
| Expected-physiology model | population prior, stable personal reference, circadian/context expectation, recent reference | event boundaries or AUD meaning |
| Confound resolver | physical explanation, identifiability, residualization/gating/ambiguity | contextual or AUD semantics |
| Multimodal evidence graph | redundancy, dependence, complementary pathways, conflict, latency | clinical authority |
| Temporal engine | onset, persistence, recovery, recurrence, burden, trajectory, censoring | diagnosis |
| AUD target engines | target-specific evidence semantics | clinical authority beyond evidence |
| Authority engine | claim ceiling, wording, abstention, surfacing eligibility | sensor processing |

## 3.1 The key reconciliation: one expected-physiology model

The repository research could otherwise produce three overlapping implementations of “correction” for activity, context, and personalization. The final design avoids that duplication.

The **longitudinal expected-physiology model** is the sole owner of conditional physiological expectation. It provides different views to different consumers:

- the **confound resolver** queries how much change is expected from independently measured physical variables such as activity, posture, post-exercise recovery, thermal environment, and respiration;
- the **context interpreter** queries the distribution expected for the current sleep/wake state, circadian phase, task state, or independently known event;
- the **patient-relative layer** queries the stable and recent distributions for this individual under matched conditions.

Therefore activity is not subtracted once in a “confound model” and again in a “personalization model.” The same expectation function is used with clearly defined counterfactual inputs.

## 3.2 The key reconciliation: fusion and temporal modeling

Fusion and temporal modeling are interleaved but have distinct ownership:

- redundant measurements of the **same physiological construct** are reconciled before parent episode construction;
- each physiological pathway retains its own time series and can create a modality/pathway event with its own onset uncertainty;
- the temporal engine associates lag-compatible child events into one parent physiological episode;
- recurrence, recovery, burden, and trajectory are properties of finalized or active parent episodes, not duplicate sensor events;
- AUD target engines consume both current pathway evidence and temporal objects.

## 3.3 The key reconciliation: baseline regimes and pathological episodes

The longitudinal layer owns expected physiology and candidate baseline regimes. The temporal engine owns abnormal episodes. A shared **regime adjudication rule** prevents a prolonged episode from becoming a new normal merely because it lasts a long time.

A candidate new stable regime is adopted only when:

1. a measurement-regime change has been excluded or explicitly bridged;
2. the interval is not part of an active or unresolved abnormal episode;
3. adequate context and circadian coverage exist;
4. the new distribution is stable over repeated valid observations;
5. the change is not being used to erase an adverse longitudinal trajectory.

The recent reference may adapt faster for short-term comparison, but the stable reference remains anchored until those conditions are met.

---

# 4. Sensing and Acquisition Model

## 4.1 Final hardware configuration

The candidate hardware is retained with two targeted additions required by the complete inference architecture.

| Function | Final component/configuration | Operating specification |
|---|---|---|
| ECG AFE | Analog Devices **MAX30005** | 512 samples/s; single-lead ECG; AC/DC lead-off and right-leg-drive functions enabled where appropriate |
| PPG AFE | Analog Devices **MAX86173** | two simultaneous optical readout channels; 200 physiological macroframes/s |
| PPG emitter | ams OSRAM **SFH 7016** | green ~530 nm, red ~655 nm, IR ~940 nm |
| PPG detectors | **2 × Vishay VEMD8080** | simultaneous near/far reflective geometry |
| EDA / contact impedance | Analog Devices **AD5940** | continuous EDA at 32 samples/s; time-multiplexed AC contact diagnostics |
| Skin temperature | Texas Instruments **TMP117** | 1 sample/s with 8-conversion averaging on a low-mass skin-contact island |
| Wrist IMU | TDK InvenSense **ICM-42688-P** | 200 Hz accelerometer + gyroscope, rigidly co-located with PPG |
| **Trunk IMU — added** | TDK InvenSense **ICM-42688-P** | 200 Hz accelerometer + gyroscope at chest pod |
| **Ambient T/RH — added** | Sensirion **SHT45** | outward-facing, thermally isolated from skin/electronics; low-rate environmental reference |
| Main controller | **ESP32-S3-WROOM-1-N16R8** | 16 MB flash + 8 MB PSRAM; acquisition master |
| Chest node controller | **ESP32-S3-WROOM-1-N16R8** | local timestamping/buffering and deterministic synchronization with master |
| Local recording | **16 GB eMMC 5.x** at master | raw native streams + acquisition metadata; SDMMC 4-bit at 20 MHz is sufficient |

The MAX86173 supports two simultaneous optical channels, four photodiode inputs, multiple LED outputs, 20-bit integrating ADCs, and frame rates far above the chosen physiological rate; its evaluation system itself uses VEMD8080 photodiodes and an inertial sensor [2]. The MAX30005 provides a single-lead ECG AFE with lead-off, right-leg-drive, EMI filtering, calibration/self-test support, and internal/external clock options [1]. The AD5940 is explicitly designed for precision skin/body impedance and electrochemical measurements [5]. The remaining components' manufacturer specifications support the stated operating envelope [3,4,6-9].

### 4.1.1 Why the trunk IMU is required

The wrist IMU must remain rigidly co-located with the optical head because it is needed to characterize PPG motion contamination and wrist tremor. A wrist-only IMU is not a reliable substitute for trunk orientation/posture: hand and forearm orientation can change independently of body posture. Yet posture is a major cardiovascular confound and a context variable used by HR/HRV interpretation.

The second chest IMU therefore provides independent trunk posture, whole-body movement, transition timing, and a respiratory-motion proxy at rest. This prevents the same wrist sensor from being forced to solve two incompatible spatial inference problems.

### 4.1.2 Why an ambient sensor is required

One skin-contact temperature sensor cannot distinguish peripheral vasomotor change from environmental heating/cooling. The SHT45 provides an independent ambient temperature and humidity reference with very small size and low power [9]. The final system does not automatically “correct” skin temperature by ambient temperature; it uses environmental measurements to determine whether a physiological thermal interpretation is identifiable and to model device-specific thermal transfer.

## 4.2 Physical architecture

The system uses **two battery-powered local acquisition pods** connected by a low-voltage digital tether. Sensitive analog/optical paths remain local; no long analog physiological trace crosses the body.

### Chest pod

- MAX30005 adjacent to the ECG connector and short electrode leads;
- chest ICM-42688-P rigidly fixed to the pod;
- local ESP32-S3 for deterministic sample timestamping, buffering, acquisition health, and sync capture.

### Wrist/forearm hub

- MAX86173 + SFH7016 + two VEMD8080 detectors;
- wrist ICM-42688-P on the same rigid optical PCB/mechanical reference;
- TMP117 on a low-mass skin-contact thermal island, thermally isolated from MCU, LED drivers, battery, and power regulators;
- SHT45 exposed to ambient air on the external side of the enclosure and isolated from the skin-temperature island;
- AD5940 close to the EDA connector;
- main ESP32-S3, eMMC, battery, and synchronization master.

The tether carries battery power/ground, framed digital data, and a dedicated hardware synchronization pulse. It does **not** carry raw analog ECG/PPG/EDA signals.

## 4.3 Electrode and sensor placement

### ECG

Use a three-electrode, Lead-II-like torso geometry:

- negative electrode high on the right torso/infraclavicular region;
- positive electrode on the lower left torso/rib region;
- right-leg-drive/reference electrode on the lower right torso.

This orientation maximizes robust R-wave timing for continuous cardiac monitoring. The system is not a 12-lead diagnostic ECG and must not make ischemia/localization claims from this lead.

### PPG

Place the reflective optical head on the distal volar forearm/high-wrist region with controlled strap pressure and optical shielding. The two photodiodes are arranged as one nearer and one farther emitter-detector path. The exact center-to-center separations are a **device-identified engineering parameter**, selected from prototype measurements to maximize all of:

- pulse SNR across subjects;
- useful correlation of the common pulsatile component;
- distinguishable path/coupling behavior between detectors;
- motion/contact diagnostic value;
- absence of detector saturation across LED-current range.

That selection rule is the final specification; inventing a universal millimetre separation without measurements would be fake precision.

Each 5 ms optical macroframe contains time-separated green, red, IR, and ambient/dark observations, with the two detector channels sampled simultaneously for each illuminated slot. The effective sampling rate of each wavelength × detector physiological channel is 200 Hz. LED current/integration time is controlled to maintain headroom rather than maximize DC level.

Green is the primary pulse-timing channel at the wrist. Red/IR and the second detector provide additional pulse, coupling, motion, and peripheral optical information. **No SpO2 value is emitted** unless the complete optical geometry is separately calibrated and validated against an appropriate oxygen-saturation reference.

### EDA

Use Ag/AgCl electrodes on the distal volar/palmar index and middle fingers, with short leads to the wrist hub. Physiological EDA and contact impedance remain distinct signals.

### Temperature

TMP117 is pressed reproducibly against skin through the low-mass island. Ambient SHT45 is exposed away from the skin interface and heat-generating electronics.

## 4.4 Cross-node time synchronization

Cross-modal timing is a physiological requirement, not transport metadata. The master emits a hardware synchronization pulse captured by hardware timers on both nodes. Each sample is timestamped against the node's monotonic hardware clock. The host timebase is produced by an affine clock model fit from the sequence of sync captures:

\[
t_{master}=a\,t_{node}+b.
\]

The synchronization acceptance criterion is established by loopback characterization, with a design requirement that the 99.9th-percentile cross-node timing error remain **≤1 ms** over a recording block. If that criterion is not met, ECG–PPG beat correspondence, pulse-arrival features, and any cross-node lag analysis are marked unavailable; slower multimodal features may remain usable.

Every stored record carries:

- node clock value;
- mapped master timestamp;
- synchronization model/version;
- residual synchronization error estimate;
- sequence number;
- sensor configuration and acquisition regime ID.

## 4.5 Safety boundary

All body-connected acquisition is battery powered. While electrodes are attached:

- no galvanic connection to a mains-powered USB host, desktop, charger, or debugger is allowed;
- charging is disabled by a physical/logic interlock that also disables body-connected acquisition;
- wired service/debug operation occurs only after body electrodes are disconnected;
- wireless transfer may occur while worn because it does not create a galvanic mains path.

This is a system-level operating invariant, not a user instruction that can be ignored by software.

---

# 5. Canonical Data and Evidence Representation

The system uses typed evidence objects rather than passing anonymous arrays between layers.

## 5.1 Core objects

### `AcquisitionSegment`

Represents native data and acquisition truth.

```yaml
segment_id:
modality:
node_id:
start_time:
end_time:
native_sample_rate:
configuration_id:
clock_model_id:
sequence_integrity:
packet_loss:
saturation_count:
raw_data_reference:
```

### `MeasurementAuthority`

Feature-specific permission to use a signal.

```yaml
source_segment_id:
feature_family:
state: VALID | DEGRADED | UNUSABLE | MISSING
measurement_uncertainty:
coverage:
reason_codes: []
contact_state:
artifact_state:
```

A stream can be valid for one purpose and unusable for another. For example, a PPG segment may support pulse rate while its amplitude/perfusion interpretation is degraded.

### `PhysiologicalFeature`

```yaml
feature_id:
construct:
effective_time_interval:
value:
units:
uncertainty:
source_ids: []
quality_authority:
processing_provenance:
physiological_component:
```

### `ExpectedPhysiology`

```yaml
feature:
context_signature:
patient_reference_version:
population_prior_version:
expected_location:
expected_scale:
expected_distribution:
baseline_maturity:
device_regime:
```

### `ConfoundAssessment`

```yaml
feature_id:
physical_explanations: []
identifiability: IDENTIFIABLE | PARTIAL | NOT_IDENTIFIABLE
residual_value:
residual_uncertainty:
state: RESIDUAL_CHANGE | EXPLAINED | PARTLY_EXPLAINED | AMBIGUOUS
```

### `ContextState`

Each dimension is separate:

```yaml
activity: {state, confidence, provenance, freshness}
posture: {state, confidence, provenance, freshness}
sleep_wake: {state, confidence, provenance, freshness}
circadian_phase: {value, uncertainty, provenance}
ambient: {temperature, humidity, confidence}
known_event: {type, source, start, end}
aud_context: {last_drink_time, withdrawal_risk, cue, treatment_state, provenance}
```

Unknown dimensions remain unknown; they are not replaced by a composite label.

### `PathwayEvidence`

Represents a physiological pathway rather than a sensor vote.

```yaml
pathway: CARDIAC | ELECTRODERMAL | PERIPHERAL_VASCULAR_THERMAL | MOTOR_TREMOR | RESPIRATORY
supporting_features: []
contradictory_features: []
missing_features: []
state_vector:
uncertainty:
reliability:
dependencies: []
```

### `TemporalEpisode`

```yaml
episode_id:
version:
state: CANDIDATE | CONFIRMED | SUSTAINED | RECOVERING | RESOLVED | UNKNOWN
start_estimate:
start_interval:
peak_time:
end_estimate:
end_interval:
left_censored:
right_censored:
child_events: []
pathway_support: []
conflicts: []
duration:
usable_observation_fraction:
severity_descriptors:
recovery_descriptors:
recurrence_group:
context_trace:
```

### `AUDTargetEvidence`

```yaml
target_id:
required_context:
context_status:
primary_evidence: []
supporting_evidence: []
contradictory_evidence: []
missing_required_evidence: []
temporal_support:
evidence_maturity:
target_specificity:
status: SUPPORTED | WEAK | AMBIGUOUS | INSUFFICIENT_EVIDENCE | NOT_EVALUATED
```

### `AuthorityDecision`

```yaml
claim_id:
claim_class:
measurement_authority:
physiology_authority:
target_specificity:
claim_evidence_maturity:
reference_standard_quality:
applicability:
calibration_status:
clinical_corroboration:
action_utility:
authority_level:
authority_ceiling:
limiting_reason_codes: []
```

## 5.2 Availability semantics

Every evidence item is one of:

- `OBSERVED_VALID` — present and permitted for the stated construct;
- `OBSERVED_DEGRADED` — usable with increased uncertainty or limited feature authority;
- `UNUSABLE` — observed but not permitted for the target feature;
- `MISSING` — no observation;
- `NOT_APPLICABLE` — construct does not apply to the current context;
- `UNSUPPORTED` — the requested inference lacks a valid scientific mapping.

These states are not interchangeable. In particular, `MISSING` and `UNSUPPORTED` are never treated as negative evidence.

---

# 6. Signal Trustworthiness

Signal trustworthiness answers only: **is this measurement adequate for the downstream construct being requested?** It does not decide whether the physiology is normal or what caused it. Orphanidou et al. provide an established ECG/PPG SQI framework for wearable monitoring, but thresholds are treated as device- and use-specific rather than copied blindly [10].

## 6.1 Acquisition integrity common to all modalities

Before waveform-specific analysis, each block is checked for:

- sequence discontinuity;
- timestamp monotonicity;
- cross-node synchronization status;
- sample-count consistency with configured rate;
- packet/storage corruption;
- sensor reset or configuration change;
- ADC/integer clipping;
- frozen/flat repeated values;
- impossible or stale metadata;
- warm-up/stabilization state.

A corrupted acquisition interval is marked `UNUSABLE` or `MISSING` before physiological algorithms see it.

## 6.2 ECG trustworthiness

The ECG quality engine combines:

1. MAX30005 lead-off/lead-on and AFE status;
2. saturation and flatline tests;
3. broadband noise and power-line/interference measures;
4. QRS detectability and detector consistency;
5. beat-template correlation and morphology stability;
6. RR plausibility without rejecting genuine arrhythmia merely for being irregular;
7. local baseline/noise estimates around candidate R waves.

Feature authority is separated:

- R-peak timing / heart rate;
- NN interval eligibility for HRV;
- waveform morphology.

An irregular but clearly measured rhythm can remain valid for heart-rate reporting while HRV windows are marked ineligible for normal NN-based interpretation.

## 6.3 PPG trustworthiness

Each detector × wavelength channel is assessed using:

- ADC and ambient-light headroom;
- AC/DC pulse ratio and pulsatile energy;
- pulse-template consistency;
- spectral concentration near plausible pulse frequency;
- inter-beat interval stability appropriate to the measured state;
- wrist-IMU motion coupling;
- simultaneous near/far detector coherence;
- wavelength consistency;
- agreement with ECG beat timing when ECG is available;
- contact/optical-coupling transitions.

ECG–PPG agreement is supporting evidence, not an absolute veto. A real pulse deficit or unusual electromechanical pattern must be representable as high-quality disagreement.

The dual-detector architecture is used explicitly: common pulsatile structure supports genuine optical pulse information; detector-specific saturation, coupling changes, or motion responses support a quality/coupling diagnosis. The exact statistical boundaries are learned from target-device recordings with controlled contact and motion variation.

## 6.4 EDA and impedance trustworthiness

EDA quality is based on:

- electrode contact/open/short state;
- ADC saturation;
- abrupt nonphysiological slew or step artifacts;
- extended flatline;
- attachment-change signatures;
- AD5940 contact-impedance diagnostic;
- motion/lead disturbance.

The AC contact diagnostic is **never treated as EDA physiology**. During a diagnostic burst, continuous EDA samples are marked `DIAGNOSTIC_BLANK`. Stable-condition diagnostics are scheduled sparsely enough that blanking remains a negligible fraction of physiological observation time; additional bursts are triggered by suspected attachment/contact changes. Exact excitation and cadence are determined from target-electrode characterization.

## 6.5 Temperature trustworthiness

Temperature quality combines:

- TMP117 status and numerical integrity;
- skin-contact stability inferred from attachment and thermal response;
- abrupt impossible steps;
- local heating events from electronics/LED duty cycle;
- ambient temperature/humidity and their rate of change;
- thermal-island device model.

A clean TMP117 value can still have **low physiological authority** if skin contact or ambient contamination is uncertain. Measurement quality and physiological specificity remain separate.

## 6.6 IMU trustworthiness

For both IMUs check:

- sensor self/status flags;
- timing and FIFO integrity;
- frozen axes;
- clipping at configured range;
- impossible discontinuities.

Motion itself is not an artifact. High movement is often exactly the valid observation needed to interpret other physiology.

---

# 7. Physiological Signal Decomposition and Feature Construction

## 7.1 ECG

The raw 512 Hz ECG is preserved. Two derived branches are maintained:

### Morphology/physiology branch

- low-frequency baseline wander is estimated robustly and removed without eliminating QRS/P/T morphology needed for timing and gross morphology;
- a practical implementation uses the repository's two-scale robust median baseline approach, with window lengths scaled to the native sample rate;
- a morphology-preserving band is retained separately from the narrow detection branch.

### R-peak detection branch

Use a Pan–Tompkins-style causal QRS detector—band-limited QRS emphasis, derivative/energy transformation, integration and adaptive thresholding—followed by R-peak refinement on the less-filtered morphology signal [11]. The method is deterministic, interpretable, and can be benchmarked on expert-annotated ECG such as MIT-BIH [16].

For every beat store:

- R-peak timestamp and uncertainty;
- detector evidence;
- morphology-template distance;
- preceding/following RR;
- NN eligibility.

### Cardiac features

- beat-to-beat RR and HR;
- robust short-window HR;
- explicitly named ultra-short RMSSD where used;
- conventional 5-minute HRV summaries when enough valid NN data exist;
- SDNN and frequency-domain HRV only with the required window duration and stationarity semantics;
- HF-HRV interpretation is downgraded when respiratory information is absent or unstable.

Ultra-short HRV and conventional 5-minute HRV are not stored under one ambiguous feature name. The 1996 Task Force standards remain a useful reference for standard HRV semantics [13]. LF/HF is **not** used as a literal “sympathovagal balance” score.

## 7.2 PPG

For each wavelength × detector channel:

1. remove dark/ambient contribution using the scheduled ambient slot;
2. retain the slowly varying optical DC component;
3. derive a pulsatile AC component;
4. maintain a cardiac-band branch for pulse detection;
5. detect pulse foot/peak and morphology only when channel authority permits;
6. maintain beat amplitude, rise time, width, area, AC/DC and template features.

The decomposition deliberately preserves multiple time scales:

- very slow optical/DC behavior;
- low-frequency vascular variation;
- cardiac pulsatility.

ECG is the preferred source of cardiac timing/HRV when available. PPG supplies pulse rate, redundant cardiac timing evidence, peripheral pulse morphology, and—after target-device characterization—limited pulse-arrival/peripheral vascular features. PPG-derived variability is labeled `PRV`, never silently substituted for ECG HRV.

Pulse-arrival timing from ECG R wave to PPG pulse foot may be retained as a research-quality peripheral timing feature when synchronization and pulse-foot quality meet the required bound. It is **not** converted into cuffless blood pressure without separate validation.

## 7.3 EDA

EDA is decomposed into tonic, phasic, and residual components using **cvxEDA** [12]. The target-device implementation uses the published Bateman-response formulation and convex decomposition as the scientific model; operating thresholds on phasic events are calibrated on the device and population rather than copied from another sensor.

The 32 Hz native signal is retained. A lower-rate analysis stream is permitted because electrodermal dynamics are slow; resampling occurs only after anti-alias filtering and quality masking.

Outputs include:

- tonic skin-conductance level/trend;
- phasic sudomotor driver;
- SCR-like event timing and amplitude where identifiable;
- residual/decomposition fit;
- valid observation fraction.

A slow tonic component is physiology, not a nuisance baseline to be discarded.

## 7.4 Skin temperature

The temperature branch produces:

- calibrated skin-contact temperature level;
- robust short slope;
- multi-minute slow trend;
- device thermal-state estimate;
- skin–ambient relationship with uncertainty.

A first-order device thermal model is identified from target hardware to account for the thermal island's time constant. The system does not high-pass away slow changes simply because they are slow.

## 7.5 IMU

Each IMU undergoes offset/scale calibration. Accelerometer + gyroscope orientation is estimated without pretending a magnetometer exists; roll/pitch relative to gravity are reliable enough for posture while absolute yaw remains uncertain.

### Chest IMU outputs

- trunk orientation/posture class;
- postural transitions;
- whole-body activity intensity;
- locomotor transition timing;
- low-motion respiratory-movement proxy when quality permits.

### Wrist IMU outputs

- local wrist motion intensity;
- PPG motion covariates;
- movement periodicity;
- stationary upper-limb intervals;
- tremor spectrum and dominant frequency.

For withdrawal tremor, spectral energy in approximately **4.4–10 Hz** and dominant-frequency structure are retained because published alcohol-withdrawal tremor work supports that band as informative [19,20]. The mapping from those features to a tremor grade is retrained/calibrated against expert reference ratings on the target device; the published coefficients are not copied.

## 7.6 Respiratory proxy

The system does not contain a dedicated respiratory belt in normal operation. Instead it may produce a **quality-qualified respiratory proxy** from independent candidate channels:

- ECG-derived respiratory modulation;
- PPG amplitude/interval modulation;
- chest-IMU respiratory motion during low movement.

A respiratory proxy is emitted only when at least one source is valid and source agreement is acceptable. It is used primarily to qualify HRV interpretation and physiological context. It is not elevated to clinical respiratory measurement authority without reference validation. The BIDMC ECG/PPG/impedance-respiration dataset provides an independent benchmark for respiratory-estimation methods [15].

---

# 8. Confound-Aware Physiological Reasoning

The confound layer asks:

> Is the observed change measurement corruption, genuine physiology plausibly produced by an independently measured physical cause, mixed physiology, or a residual change not explained by those causes?

It uses four operations only when justified: **gating, stratification, calibrated residualization, and explicit ambiguity**.

## 8.1 Cardiac activity/posture model

For heart rate and selected cardiac features, the expected-physiology model is a robust, patient-adaptive generalized additive model with population shrinkage:

\[
\mu_f(t)=\beta_{0,f}+s_{24,f}(\text{circadian phase})+
 g_f(\text{activity})+
 \beta_{posture,f}+g_{rec,f}(\text{time since activity})+
 \beta_{sleep,f}+u_{patient,f}.
\]

Only terms relevant to a feature are included. Interactions are added only when supported by data.

For HR, a physical-demand residual is computed only when:

- chest activity/posture are valid;
- the current activity is inside the calibrated range;
- the model has sufficient personal or population support;
- the activity transition/recovery phase is represented.

Otherwise the state is `AMBIGUOUS` rather than an overconfident residual.

## 8.2 HRV

HRV is not broadly “corrected” by subtracting an activity model. Instead:

- vigorous activity and unstable postural transitions gate many HRV interpretations;
- rest/sleep/posture-matched HRV is compared to the appropriate expected distribution;
- respiration proxy availability is recorded for HF-HRV interpretation;
- ultra-short and 5-minute features remain separate.

## 8.3 PPG

Wrist motion is used primarily to assess measurement contamination and pulse-feature authority. Adaptive waveform subtraction is not used unless target-device data demonstrate that the contaminating transfer function is identifiable and that subtraction improves reference performance without distorting true pulse physiology.

Near/far and multi-wavelength relationships strengthen motion/coupling assessment. They are not automatically assigned physiological meaning.

PPG-DaLiA, which contains synchronized wrist PPG/accelerometer and chest ECG in 15 subjects performing daily activities, is an appropriate independent benchmark for motion-robust pulse-rate estimation; the target geometry still requires native-device characterization [17,18].

## 8.4 EDA

Physical activity, temperature, humidity and thermoregulatory sweating can produce genuine electrodermal changes. A single palmar EDA channel cannot always distinguish psychological/autonomic activation from thermoregulatory effects.

The system therefore:

- removes obvious measurement corruption in the quality layer;
- conditions expectation on activity/ambient state;
- uses EDA residualization only where repeated target-device data show identifiability;
- otherwise reports `EDA_ACTIVATION_WITH_THERMAL_ACTIVITY_AMBIGUITY`.

## 8.5 Skin temperature

Skin temperature is interpreted through a device-specific thermal model incorporating ambient temperature/humidity, activity, contact stability, and recent local LED/electronics heating. If the observed thermal trajectory can be explained by environmental change within model uncertainty, it is tagged accordingly. If environmental and vasomotor explanations cannot be separated, the result is `AMBIGUOUS`, not a manufactured autonomic residual.

## 8.6 Tremor

Withdrawal-relevant tremor analysis is permitted only during segments where:

- wrist IMU is valid;
- gross voluntary movement is low enough to resolve tremor;
- the spectral pattern is distinct from locomotor/voluntary periodic motion;
- the AUD target layer independently has withdrawal-risk context if it wants to call the tremor “withdrawal-related.”

Without that context, the output remains `TREMOR_PHENOTYPE`.

---

# 9. Context-Aware Interpretation

Context sets the correct expectation for genuine physiology. It does not create physiological truth.

## 9.1 Context dimensions

The final context model includes:

- activity intensity and activity class;
- trunk posture;
- postural/activity transition and recovery phase;
- sleep/wake state;
- local circadian phase / time of day;
- ambient temperature and humidity;
- independently known cue/stressor/task events;
- independently supplied alcohol-use/cessation timing;
- treatment/detoxification state and clinically supplied medication context when available.

Every dimension has provenance, freshness, validity and uncertainty.

## 9.2 Anti-circularity rules

- Sleep/wake is inferred primarily from actigraphy/time and validated behavioral signals; cardiac physiology that will later be interpreted inside sleep is not allowed to become the sole definition of sleep.
- “Stress” is never inferred from EDA and then used as context to explain the same EDA.
- “Alcohol cue” comes from protocol/user/clinical/event metadata, not autonomic arousal.
- “Withdrawal risk” comes from alcohol-use history/clinical setting, not tachycardia.
- “Recent drinking” comes from an independent report/reference measurement, not nocturnal HRV suppression.

## 9.3 Context sufficiency is claim specific

There is no universal `context_confidence` scalar. For example:

- HR-at-rest interpretation requires activity/posture confidence;
- cue-reactivity interpretation requires independently known cue timing;
- nocturnal recovery requires a sufficiently reliable sleep interval;
- withdrawal attribution requires a recent reduction/cessation or equivalent clinical context;
- generic physiological-state detection may proceed without any AUD context.

---

# 10. Personalized Longitudinal Physiology

## 10.1 Reference hierarchy

For each feature the system maintains four distinct references:

1. **population prior** — independent normative/benchmark prior used during cold start;
2. **stable personal reference** — guarded long-term definition of expected physiology;
3. **context-conditioned personal reference** — stable reference conditioned on circadian/context state;
4. **recent adaptive reference** — short-term state used to quantify current drift and recovery without redefining stable normal.

A historical regime archive stores previous stable references and device regimes.

## 10.2 Robust expected-physiology model

The central estimate is a robust hierarchical GAM or equivalent robust additive model, with:

- cyclic 24-hour spline for circadian behavior;
- categorical posture/sleep states;
- smooth activity/recovery terms where physically relevant;
- ambient terms for thermal/EDA features where identifiable;
- patient-specific intercepts/slopes shrunk toward population priors while data are sparse;
- heavy-tailed/robust residual handling.

For a current observation:

\[
d_f(t)=x_f(t)-\hat\mu_f(t,c_t)
\]

and a robust standardized deviation is:

\[
z_f^*(t)=\frac{d_f(t)}{1.4826\,MAD_f(c_t)+\epsilon}.
\]

The system stores the raw delta, robust z, empirical percentile, expected distribution and reference maturity rather than collapsing them into one anomaly score.

## 10.3 Baseline maturity

Baseline maturity is a vector, not “seven days have passed.” It includes:

- number and duration of valid observations;
- independent days/nights represented;
- circadian-bin coverage;
- required context coverage;
- residual stability;
- recency;
- device-regime continuity;
- fraction of history excluded because of episodes/confounds.

A patient-relative statement is authorized only when the specific feature/context cell has enough effective data or a hierarchical posterior with sufficiently narrow uncertainty. Otherwise the system reports population/context-relative evidence and marks personal-reference maturity as insufficient.

## 10.4 Baseline protection

Stable-reference updates are allowed only from intervals that are:

- technically valid;
- outside active/unresolved physiological episodes;
- outside known acute alcohol exposure, cue challenge, major stress protocol, or withdrawal-risk interval when the feature would be biased;
- inside a known measurement regime;
- sufficiently context characterized.

Persistent deterioration therefore remains visible in the stable-reference residual and longitudinal trajectory even if the recent reference gradually follows it.

## 10.5 Device changes

A new electrode placement, optical head revision, detector spacing, LED configuration, sensor replacement, sampling-rate change, firmware algorithm change, gain/filter setting, or enclosure/contact change starts a new `measurement_regime_id`.

The stable patient model is frozen across the boundary until a device-bridge analysis demonstrates which features are comparable. Unbridged features start a new reference; the old history remains archived rather than being numerically concatenated.

---

# 11. Reliability-Aware Multimodal Physiological Fusion

## 11.1 Dependency-aware hierarchy

Fusion occurs in three levels.

### Level 1 — redundant observation fusion

Measurements of the same construct are reconciled first.

Example: ECG HR and PPG pulse rate.

If their difference is within the calibrated consistency distribution, a precision-weighted estimate can be formed:

\[
\hat x=\frac{\sum_i x_i/\sigma_i^2}{\sum_i1/\sigma_i^2},
\qquad
\operatorname{Var}(\hat x)=\left(\sum_i1/\sigma_i^2\right)^{-1}.
\]

The observation variances are learned from reference/target-device performance and inflated under degraded quality.

If two **high-quality** measurements disagree beyond the calibrated range, the system does not fuse them. It emits `HIGH_QUALITY_CONFLICT` and preserves both observations.

### Level 2 — pathway construction

Independent/complementary constructs are grouped into pathway evidence:

- cardiac timing/autonomic regulation;
- electrodermal activation;
- peripheral vascular/thermal response;
- motor/tremor;
- respiratory support.

Multiple features derived from the same RR sequence remain one cardiac dependency family.

### Level 3 — target-specific complementary reasoning

AUD target engines consume only pathways relevant to that target. There is no universal “all-sensors confidence.” A target-specific probabilistic model is allowed only after its exact target has adequate subject-independent validation and calibration.

## 11.2 Reliability as observation uncertainty

Where mathematically meaningful, technical reliability modifies **observation variance** rather than multiplying arbitrary confidences. Where quantities are conceptually incomparable—signal quality, target specificity, baseline maturity, clinical authority—they remain separate fields.

## 11.3 Latency-aware evidence

Each pathway observation includes an effective time interval and latency uncertainty. Cardiac timing may change before EDA; skin temperature may respond much more slowly. Parent episode association therefore uses interval overlap and learned pairwise lag distributions, not simultaneous peaks.

## 11.4 Conflict semantics

The fusion layer distinguishes:

- `MISSING` — not observed;
- `UNRELIABLE` — observed but low technical authority;
- `NEUTRAL` — valid observation provides neither support nor contradiction;
- `CONTRADICTORY` — valid evidence opposes a target/pathway expectation;
- `UNSUPPORTED` — no valid mapping exists.

Averaging these states into zero would destroy their semantics and is prohibited.

---

# 12. Temporal Physiological Event and Trajectory Modeling

The temporal layer converts valid patient/context-referenced physiology into **versioned episodes**, not endless isolated windows.

## 12.1 Hierarchy

```text
WINDOW
  ↓
PATHWAY / MODALITY EVENT
  ↓
MULTIMODAL PHYSIOLOGICAL EPISODE
  ↓
RECURRENCE GROUP
  ↓
DAILY / NIGHTLY BURDEN
  ↓
LONGITUDINAL TRAJECTORY
```

AUD target episodes reference these objects; they do not duplicate them.

## 12.2 Online candidate boundaries

For each pathway residual stream, use **Bayesian online change-point detection (BOCPD)** with a robust predictive observation model [21]. BOCPD supplies a posterior over run length and a causal boundary proposal. A boundary proposal alone is not a physiological event.

A candidate event opens when either:

- a patient/context-referenced deviation crosses a calibrated entry region; or
- BOCPD produces a sufficiently strong change posterior;

and at least one valid physiological pathway supports the direction.

The entry operating point is selected on held-out subjects to meet a pre-specified false-parent-episodes-per-observed-day constraint while minimizing onset delay. It is not a universal biological constant.

## 12.3 Explicit lifecycle

```text
EXPECTED
   ↓ credible change
CANDIDATE
   ↓ persistence / evidence
CONFIRMED
   ↓ sustained duration
SUSTAINED
   ↓ return trajectory
RECOVERING
   ↓ stable return
RESOLVED
```

Additional states:

- `UNKNOWN_DATA_GAP`;
- `LEFT_CENSORED`;
- `RIGHT_CENSORED`;
- `INVALIDATED`;
- `MERGED`;
- `SPLIT`.

Confirmation, sustained-duration, hysteresis and recovery-stability parameters are selected from event-level calibration data. The internal research corpus's two-minute/five-minute/three-minute generic autonomic values are useful initialization points for calibration, **not physiological laws** and therefore are not hard-coded as final clinical constants.

## 12.4 Severity is a vector

Every episode stores:

- peak deviation;
- mean/median deviation;
- integrated abnormality/burden;
- qualifying occupancy;
- duration;
- time to peak;
- rise/fall slopes;
- within-episode variability;
- high-severity occupancy;
- usable-observation fraction.

A 10-second large excursion and a 45-minute moderate excursion are therefore not equivalent.

## 12.5 Recovery is first class

Track:

- recovery start;
- time to half-recovery;
- time to 80% recovery where meaningful;
- time to expected personal/context band;
- recovery slope;
- residual deviation at fixed horizons;
- interrupted-recovery count;
- complete vs censored recovery.

No exponential model is forced. If a simple exponential or other curve adequately fits repeated comparable episodes, its parameters may be retained in addition to nonparametric descriptors.

## 12.6 Missingness and censoring

Missing data is `UNKNOWN`, never `NORMAL`.

During an active episode:

- if one modality disappears but independent pathways remain supportive, the parent episode can stay open with reduced evidence coverage;
- if all decisive evidence disappears, the episode becomes `ACTIVE_WITH_GAP` and eventually right-censored according to the calibrated gap policy;
- recovery cannot be inferred inside a complete data gap.

Gap-bridging parameters are calibrated against the actual device's dropout behavior and event timescale.

## 12.7 Merge and split

Merge adjacent event fragments only when:

- stable recovery was not achieved;
- physiology/morphology is compatible;
- context did not undergo an incompatible transition;
- the intervening gap is within the calibrated continuity range.

Split when:

- a stable return occurs;
- a strong change point enters a qualitatively different regime;
- a new context transition plus new physiological onset supports a distinct process;
- distinct morphology is sustained.

## 12.8 Retrospective reconciliation

After enough future data exist, use **PELT** penalized change-point detection to refine episode boundaries and merge/split decisions [22]. Retrospective output stores both the original online boundaries and the revised boundaries plus revision reason.

## 12.9 Burden and trajectory

Daily/nightly burden is a vector:

```text
episode count
observed-time-normalized episode occupancy
integrated deviation
high-severity minutes
median episode duration
median recovery time
incomplete-recovery count
nighttime burden
```

Longitudinal trends use robust estimators such as Theil–Sen slope plus change-point analysis. They are normalized by usable observation time. A trajectory may be `IMPROVING`, `STABLE`, `WORSENING`, or `INSUFFICIENT_DATA`, but the label is attached to a named physiological metric or burden vector—not a generic disease score.

---

# 13. AUD-Specific Physiological Inference

AUD semantics enter only after upstream physiological evidence has passed its technical, confound, context, patient-reference and temporal requirements. There is **no universal AUD classifier**.

## 13.1 Target registry

| Target | Required primary evidence | Essential independent context/reference | Present output | Present authority ceiling |
|---|---|---|---|---|
| Withdrawal-related physiological activation | persistent autonomic episode ± tremor | recent reduction/cessation, detox/withdrawal-risk context | compatibility/state evidence | A4 |
| Withdrawal tremor | validated tremor phenotype | withdrawal-risk context for AUD attribution | component evidence | A4; tremor itself A3 |
| Alcohol-cue autonomic reactivity | event-locked autonomic change | independently known alcohol-cue interval; neutral/pre-cue baseline | cue-associated response | A4 |
| AUD-relevant stress reactivity | stress-response/recovery physiology | independently known stressor + AUD context/history | reactivity marker | A4 |
| Alcohol-associated impairment pattern | IMU + autonomic multimodal pattern | device-specific model, sober reference; objective alcohol reference for validation | compatibility only under current evidence | A4; no clinical probability |
| Chronic autonomic dysregulation | repeated stable-rest/sleep HR/HRV | mature longitudinal baseline | patient-relative burden/trend | A3 + A4 literature association |
| Autonomic recovery/abstinence progression | repeated HR/HRV/nightly recovery | abstinence/treatment chronology | physiological recovery trend | A3/A4 |
| Nocturnal autonomic recovery disturbance | sleep-relative cardiac recovery | reliable sleep interval; alcohol exposure only if independently known | nightly state/trend | A3; A4 if exposure known |
| Treatment-associated physiological change | longitudinal/challenge response | treatment timeline and suitable baseline | association, not efficacy | A3/A4 |
| Relapse-risk-associated physiology | multi-day recovery/HRV/cue features | defined recovery population and outcome horizon | descriptive risk-associated deviation | A4 association only; no probability |

## 13.2 Withdrawal-related activation

Alcohol withdrawal has clinically recognized autonomic hyperactivity and tremor, but wearable autonomic physiology is nonspecific and cannot replace clinical assessment [31]. Conventional HRV findings during acute withdrawal are also inconsistent across studies.

The target engine therefore requires **withdrawal-risk context** before applying AUD semantics.

### Inputs

- generic sustained autonomic episodes at rest or after physical-confound handling;
- resting HR deviation;
- EDA activation with thermal/activity interpretation;
- skin-temperature support where interpretable;
- wrist tremor phenotype;
- HRV as supporting/contradictory evidence, not a mandatory monotonic marker;
- time since reduction/cessation if known;
- clinical assessment if available.

### Logic

- tremor phenotype + persistent autonomic activation + withdrawal context = stronger compatibility evidence;
- isolated tremor = `withdrawal_tremor_evidence` only when context supports attribution;
- isolated tachycardia/EDA = nonspecific autonomic activation;
- contradictory valid evidence is retained;
- no wearable-only CIWA total, seizure prediction, delirium prediction, medication recommendation, or withdrawal diagnosis is produced.

Permitted clinician-facing wording at A4:

> Sustained multimodal autonomic activation is present relative to the patient's expected resting physiology. In the documented withdrawal-risk context, the pattern is compatible with withdrawal-related autonomic hyperactivity, but is not diagnostically specific.

## 13.3 Withdrawal tremor

The motor pathway calculates tremor-band energy, dominant frequency, spectral width, intermittency, and stationary-segment quality. Published work identified withdrawal-related tremor information in the roughly 4.4–10 Hz range and showed feasibility of quantitative tremor scoring [19,20].

The target device does not reuse another device's regression coefficients. It fits a device-specific ordinal/regression mapping against blinded expert tremor ratings. Until that mapping meets the evidence threshold, the system reports the physical tremor phenotype and its compatibility with withdrawal context rather than a clinical scale score.

## 13.4 Alcohol-cue autonomic reactivity

This target exists **only** for an independently known cue interval.

For each feature:

\[
\Delta f=f_{cue}-f_{pre-cue}
\]

and, where a neutral cue is available:

\[
\Delta\Delta f=(f_{alcohol\ cue}-f_{pre})-(f_{neutral}-f_{pre,neutral}).
\]

The response object stores cardiac, EDA, thermal, timing and recovery components separately. Known cue timing may support `ALCOHOL_CUE_AUTONOMIC_REACTIVITY`; physiology does **not** establish subjective craving. Naturalistic craving remains dependent on self-report/EMA because ambulatory studies show strong inter-individual heterogeneity and weak generic mappings [26,27].

## 13.5 AUD-relevant stress reactivity

Generic stress physiology is not AUD-specific. The target is evaluated only when a stressor is independently known and the response is compared with that person's historical context-matched response/recovery profile. The output is a reactivity phenotype, never “relapse imminent.”

## 13.6 Alcohol-associated impairment pattern

A 2026 controlled study of 28 participants found that wrist motion plus autonomic signals could classify BrAC/SFST-defined impairment with promising within-study subject-grouped performance, with movement carrying much of the signal [25]. This justifies the **structure** of an impairment-pattern engine, not deployment of a universal intoxication detector.

The final engine therefore:

- uses 5-minute-scale multimodal windows from wrist/chest motion, HR, EDA and skin temperature;
- compares with an individual's sober reference where available;
- requires a **target-device, participant-separated model** for any classifier output;
- stores the model's exact development label and applicability domain;
- currently caps clinician-facing output at A4 compatibility;
- suppresses numeric probability because independent external calibration for the target device/population is absent;
- never estimates exact BAC or legal impairment.

Objective BrAC/BAC/TAC, when available, is treated as the higher-specificity exposure/reference measurement rather than being overwritten by wearable inference.

## 13.7 Chronic autonomic dysregulation

The strongest established AUD physiology is at the **group and longitudinal cardiac-autonomic level**, not as an individual diagnostic test. A meta-analysis found lower parasympathetic HRV in AUD with substantial heterogeneity [23].

The system tracks:

- context-matched resting/nightly HR;
- RMSSD and other duration-appropriate HRV summaries;
- persistent deviation from stable personal reference;
- weekly/monthly trend and recovery.

The output is `AUTONOMIC_DYSREGULATION_BURDEN` or a named HR/HRV trend. Literature association can be attached at A4, but the feature is not an AUD diagnostic biomarker.

## 13.8 Autonomic recovery / abstinence progression

Longitudinal sleep/rest HR and HRV can improve across abstinence in published cohorts [29]. The system therefore compares repeated comparable nights/rest periods using the stable reference and early-recovery regime, reporting direction and uncertainty.

Permitted wording:

> Nighttime cardiac autonomic physiology has moved toward the patient's stable recovery reference over the observed interval; this direction is compatible with recovery patterns reported during abstinence but is not a validated measure of treatment efficacy.

## 13.9 Nocturnal autonomic recovery

A large real-world study found dose-dependent autonomic recovery suppression during the first hours of sleep after alcohol intake [28]. The system computes:

- HR elevation relative to alcohol-free/comparable-night reference;
- HRV suppression burden;
- first-three-hours episode occupancy;
- overnight recovery fraction;
- next-morning residual deviation.

If alcohol exposure is not independently known, the result is `NOCTURNAL_AUTONOMIC_RECOVERY_DISTURBANCE` at A3. If exposure is independently documented, the episode may be described as **alcohol-associated** at A4; the physiology alone is not used to infer drinking.

## 13.10 Treatment-associated change

Treatment timing can be used as independently supplied context. Physiological trends or standardized challenge-response changes may be associated with the treatment interval, but no causal efficacy claim is made without a design capable of supporting causality.

## 13.11 Relapse-risk-associated physiology

Prospective cardiac and cue-reactivity studies provide interesting associations, but naturalistic evidence does not support a mature passive imminent-relapse detector [26,30]. The system may expose a multi-day adverse physiological trajectory and annotate that similar constructs have been associated with poorer outcomes. It does not emit a relapse probability, time-to-relapse prediction, or interruptive relapse alarm from physiology alone.

---

# 14. Evidence-Authority-Aware Clinical Interpretation

The terminal layer answers:

> What is the strongest statement this exact evidence is justified in making for this exact claim, patient, setting, sensor configuration and time horizon?

## 14.1 Separate dimensions

The authority engine keeps these distinct:

- measurement validity;
- feature validity;
- confound resolution;
- context sufficiency;
- baseline maturity;
- patient-reference uncertainty;
- multimodal agreement/conflict;
- temporal support;
- target specificity;
- claim evidence maturity;
- reference-standard quality;
- applicability domain;
- calibration quality;
- clinical corroboration;
- clinical-action utility.

No “overall confidence = 0.91” is required or preferred.

## 14.2 Claim Evidence Maturity (CEM)

The evidence registry uses a claim-specific maturity ladder:

| Grade | Evidence achieved | Maximum default use |
|---|---|---|
| CEM-0 | mechanistic plausibility/hypothesis | no AUD patient-level claim |
| CEM-1 | replicated group association | literature association only |
| CEM-2 | within-person/longitudinal association | cautious compatibility/monitoring |
| CEM-3 | individual prediction/discrimination with proper subject-independent validation | constrained model-based interpretation |
| CEM-4 | independent external validation + calibration in intended population | calibrated A5 probability may be eligible |
| CEM-5 | prospective workflow evaluation with demonstrated clinical benefit/safety | A6/A7 may be eligible within exact protocol |

CEM is always paired with risk of bias, reference-standard quality, replication, target definition, prediction horizon and applicability.

## 14.3 Authority levels

| Authority | Permitted claim |
|---|---|
| **A0 — ABSTAIN** | higher-level interpretation unsupported |
| **A1 — MEASUREMENT** | reliable direct measurement |
| **A2 — PATIENT-RELATIVE FEATURE** | validated personal/context deviation |
| **A3 — PHYSIOLOGICAL STATE** | supported physiological state/episode |
| **A4 — AUD COMPATIBILITY / ASSOCIATION** | cautious target relationship, explicitly nonspecific |
| **A5 — CALIBRATED AUD PROBABILITY/RISK** | exact validated target + horizon + external calibration |
| **A6 — CLINICIAN-REVIEW CDS** | validated criterion supporting clinician reassessment |
| **A7 — ACTION RECOMMENDATION** | prospectively validated action protocol with demonstrated utility |

The current physiological evidence base leaves most AUD targets at A3/A4. A5–A7 are **locked per claim** until the evidence registry satisfies their requirements.

## 14.4 Hybrid ceiling + cumulative support

Authority is governed by a hard ceiling:

\[
A_{max}=\min\{A_{upstream},A_{specificity},A_{CEM},A_{reference},A_{domain},A_{calibration},A_{corroboration},A_{utility}\}.
\]

The expression is conceptual; the fields are categorical constraints, not assumed commensurate numbers.

Within `Amax`, persistence, independent pathways, a mature personal baseline and clinical corroboration may strengthen the statement. They can never break the ceiling.

Example:

```text
excellent ECG quality
strong HR deviation
strong EDA deviation
45-minute persistence
low motion
weak withdrawal specificity
```

remains representable as:

```text
measurement authority: HIGH
physiological state: STRONG SUSTAINED AUTONOMIC ACTIVATION
AUD specificity: LIMITED
withdrawal authority: A4 maximum
numeric probability: PROHIBITED
```

## 14.5 Hard gates

A higher claim is blocked/capped by:

- invalid required upstream evidence;
- unresolved dominant confound;
- missing target-essential context;
- no established clinical association for the target;
- critical population/setting/sensor/horizon mismatch;
- numeric probability without external validation/calibration;
- action recommendation without evidence of clinical utility.

The lower-level supported physiology remains visible.

## 14.6 Abstention states

The final claim can explicitly be:

- `INSUFFICIENT_EVIDENCE`;
- `AMBIGUOUS`;
- `MIXED_EVIDENCE`;
- `OUT_OF_DOMAIN`;
- `OUT_OF_DISTRIBUTION`;
- `CLINICAL_INTERPRETATION_NOT_SUPPORTED`;
- `PROBABILITY_NOT_VALIDATED`.

## 14.7 Clinical corroboration

Clinical data are separate evidence families. For withdrawal, a validated clinical assessment and cessation history provide higher-specificity corroboration than wearable autonomic physiology. ASAM guidance treats withdrawal as a clinical syndrome with multiple possible symptoms and alternative explanations [31]. A 2026 multicenter VHA cohort also found that withdrawal scales incorporating vital signs did not reduce complicated withdrawal compared with CIWA-Ar-based management and were associated with longer treatment duration, reinforcing that “more objective physiology” is not synonymous with clinical utility [32].

## 14.8 Probability privilege

A numeric probability is emitted only when all are true:

1. the exact target is defined;
2. the horizon is defined;
3. the reference label is appropriate;
4. development/testing separate subjects and avoid episode/window leakage;
5. independent external validation exists;
6. calibration is acceptable in the intended population/setting;
7. the current patient is in domain;
8. uncertainty and applicable prevalence/prior are represented.

A raw classifier score is never relabeled “probability.” Calibration and discrimination are distinct properties [35].

## 14.9 Clinician-facing wording

Language is locked to authority:

- A1: “Resting HR is 92 bpm.”
- A2: “Resting HR is elevated relative to the patient's established baseline.”
- A3: “Multimodal evidence supports sustained autonomic activation.”
- A4: “In the documented withdrawal-risk context, the pattern is compatible with withdrawal-related autonomic hyperactivity, but is not diagnostically specific.”
- A5: exact calibrated probability with target, horizon, domain and uncertainty.
- A6: “The validated review criterion is met; clinical reassessment may be appropriate.”
- A7: only wording authorized by the prospectively validated clinical protocol.

FDA's 2026 CDS guidance emphasizes the importance of allowing clinicians to independently review the basis of recommendations; this system exposes the supporting, contradictory, missing and applicability evidence for every surfaced claim [33].

---

# 15. Uncertainty, Conflict, Missingness and Abstention

## 15.1 Uncertainty is a vector

The system keeps at least:

```yaml
acquisition_integrity:
signal_quality:
measurement_uncertainty:
feature_validity:
coverage:
confound_uncertainty:
context_certainty:
baseline_maturity:
patient_reference_uncertainty:
model_epistemic_uncertainty:
modality_agreement_or_conflict:
temporal_support:
target_specificity:
aud_evidence_maturity:
calibration_quality:
applicability_domain:
clinical_authority:
```

## 15.2 Aleatoric vs epistemic uncertainty

**Aleatoric/target ambiguity** remains when excellent physiology has multiple plausible causes. More sensor precision may not solve it.

**Epistemic uncertainty** comes from limited or mismatched validation, sparse personal history, sensor-regime changes or poor model knowledge. It can potentially be reduced by better evidence.

## 15.3 Conflict policy

High-quality conflict causes one of three outcomes:

- keep pathway evidence separate and continue lower-level inference;
- downgrade a target that requires concordance;
- return `MIXED_EVIDENCE` if the disagreement materially affects the claim.

Conflict never disappears through arithmetic averaging.

## 15.4 Missing-required-modality policy

Each target declares primary and optional evidence. If a required primary modality is missing, the target becomes `UNEVALUABLE`/`INSUFFICIENT_EVIDENCE`; the system does not impute a population-normal value.

---

# 16. Online and Retrospective Processing Semantics

The system supports two scientifically consistent views.

## 16.1 Online causal view

Uses only data available at time `t`:

- acquisition integrity;
- causal filtering;
- signal-quality gating;
- live decomposition;
- current expected physiology;
- confound/context interpretation;
- pathway evidence;
- BOCPD candidate boundaries;
- active episode lifecycle;
- permitted AUD/authority claims.

## 16.2 Retrospective refined view

May use future samples for:

- zero-phase/refined filtering where appropriate;
- longer-window EDA decomposition;
- refined quality decisions near boundaries;
- PELT change-point reconciliation;
- merge/split decisions;
- final recovery and burden;
- clock-drift correction using the full recording.

A retrospective revision stores:

```text
online value/boundary
final value/boundary
revision timestamp
algorithm/version
revision reason
```

The interpretation is never silently overwritten.

---

# 17. Data and Evidence Foundation

No single dataset validates this complete system. The scientific basis deliberately combines **independent public/reference data** with **target-hardware data** because they answer different questions.

## 17.1 Evidence-source matrix

| Component | Independent evidence/reference | Target-hardware evidence | What this establishes | What it does not establish |
|---|---|---|---|---|
| ECG R peaks / beat quality | MIT-BIH expert beat annotations [16]; QRS literature [11] | simultaneous target ECG vs reference ECG | detector robustness + device timing | AUD validity |
| PPG pulse timing under motion | PPG-DaLiA ECG+PPG+accel [17,18] | target dual-PD/multi-wavelength PPG + ECG + wrist IMU | pulse-rate robustness and native SQI | exact equivalence to public sensors |
| ECG/PPG SQI principles | Orphanidou et al. [10] | controlled motion/contact degradation | general QC structure + device thresholds | clinical specificity |
| Respiration proxy | BIDMC ECG/PPG with impedance respiration and breath annotations [15] | target ECG/PPG/chest IMU vs respiratory belt subset | method/reference behavior | clinical respiration diagnosis |
| EDA decomposition | cvxEDA [12], controlled EDA literature | native AD5940 recordings + contact diagnostics | decomposition behavior and device noise/contact effects | AUD-specific meaning |
| Context/activity confounds | WESAD [14], PPG-DaLiA [17] | controlled activity/posture/environment protocol | general relationships + native calibration | clinical AUD target validity |
| Personal baselines | longitudinal public cohorts where suitable | repeated target-device subject recordings | within-person distributions | diagnosis |
| Tremor | Carver/Aarabi withdrawal tremor studies [19,20] | target IMU vs blinded expert tremor ratings | target-device tremor component validity | full withdrawal severity |
| Alcohol impairment pattern | Kaczor et al. 2026 [25] | target device + objective BrAC/TAC/structured impairment reference | model transferability only after new validation | exact BAC/legal impairment |
| Withdrawal compatibility | ASAM/clinical withdrawal literature [31] | simultaneous wearable + clinician assessment/reference scale | clinical association and device relationship | autonomous diagnosis/utility |
| Cue reactivity | controlled AUD cue studies [27,30] | independently timed cue protocol + self-report | event-linked response | generic naturalistic craving |
| Nocturnal alcohol effect | Pietilä et al. [28] and related sleep HRV studies | repeated known-exposure/alcohol-free nights | within-person nocturnal pattern | alcohol exposure if not independently known |
| Recovery trajectory | de Zambotti/Eddie longitudinal work [29,30] | repeated longitudinal target-device recordings | patient trajectory | treatment causality |
| Relapse outcome | prospective AUD cohorts [30] | confirmed drinking outcome with fixed horizon | risk-model evidence if adequately powered | imminent relapse from generic physiology |
| Clinical authority | IMDRF/FDA/calibration/clinical AI frameworks [33-38] | prospective workflow evaluation for any higher authority | claim governance | AUD biology by itself |

## 17.2 Public datasets retained as permanent evidence

Public/reference datasets are not disposable scaffolding. They provide:

- independent physiology;
- reproducibility across platforms;
- expert/reference labels difficult to recreate locally;
- subject-independent tests;
- protection against overfitting conclusions to one prototype.

Important permanent resources include:

- **MIT-BIH Arrhythmia Database:** expert-reviewed ambulatory ECG beat annotations [16];
- **BIDMC PPG and Respiration Dataset:** synchronized ECG, PPG, impedance respiration and manual breath annotations [15];
- **PPG-DaLiA:** wrist PPG/accelerometer plus chest ECG/respiration across daily activities [17,18];
- **WESAD:** wrist/chest multimodal physiology and motion with controlled affect/stress protocol [14].

## 17.3 Target-hardware data is a first-class input

Native device data is required to estimate or characterize:

- sensor noise and dynamic range;
- clipping/saturation distributions;
- timestamp jitter and clock drift;
- cross-node sync residuals;
- LED current/integration behavior;
- near/far detector relationships;
- wavelength-specific pulse/coupling behavior;
- optical–IMU motion coupling;
- electrode impedance ↔ EDA quality relationship;
- ECG lead/contact behavior;
- temperature island time constant and ambient transfer;
- modality-specific observation variance;
- contact-state thresholds;
- device-specific signal-quality distributions;
- feature translation from independent datasets to the actual device;
- patient-specific baselines and recovery distributions.

Target-device data may reveal useful relationships absent from public datasets. Such relationships can become part of the final inference graph after their information content is established; they are not discarded merely because a benchmark lacks them.

## 17.4 Deterministic native replay

A stored target-device recording must be replayable through the same scientific inference graph used live:

```text
acquisition integrity
→ signal trustworthiness
→ decomposition
→ confound/context interpretation
→ patient expectation
→ multimodal pathways
→ temporal episodes
→ AUD target evidence
→ authority decision
```

Replay may select the retrospective algorithm mode, but the semantic contracts and provenance remain identical.

---

# 18. Ground Truth and Reference Standards

Every learned or calibrated component declares what “truth” means.

| Target | Required reference truth |
|---|---|
| ECG beat timing | expert/reference ECG R-peak annotation |
| ECG/PPG signal quality | expert waveform annotation and/or controlled corruption/contact protocol |
| PPG pulse rate | ECG-derived cardiac timing under synchronized conditions |
| respiration proxy | respiratory belt/impedance/validated respiratory reference with annotated breaths |
| activity/posture | protocol/video/manual activity/posture annotation |
| sleep/wake | validated actigraphy reference; PSG when claims require sleep architecture |
| EDA contact state | controlled electrode/contact state + electrical impedance behavior |
| skin temperature | calibrated contact temperature reference + controlled ambient condition |
| tremor | blinded expert tremor assessment/reference motion measurement |
| alcohol exposure | objective BrAC/BAC/TAC or clearly documented administration, depending target |
| withdrawal | clinician assessment / validated clinical scale + structured clinical context; target precisely defined |
| craving | contemporaneous self-report/EMA for the subjective construct |
| alcohol cue | independently controlled/recorded cue timing |
| relapse/drinking outcome | clearly defined confirmed drinking outcome and pre-specified horizon |
| treatment response | pre-specified clinical outcome; physiology alone is not the reference for efficacy |

A proxy label may be useful, but the resulting model inherits the proxy's semantics. A model trained against BrAC ≥0.08 is not automatically a model of “unsafe to drive.” A model trained against self-reported craving is not automatically a relapse predictor.

---

# 19. Calibration and Evaluation Semantics

Calibration/evaluation is intrinsic to deciding what the final system is allowed to compute; it is not a separate “validation plan.”

## 19.1 Leakage rules

For any learned model:

- subjects are separated between train/test when generalization to new patients is claimed;
- overlapping windows from one episode never cross train/test boundaries;
- longitudinal personalization is evaluated chronologically;
- future data are never used in online features;
- retrospective boundaries are not leaked into causal detection;
- probability calibration uses held-out subjects/data;
- event claims are evaluated at event level, not only window level.

## 19.2 Metrics by layer

### Signal/feature layer

- sensitivity/precision for usable vs unusable segments;
- timing/feature error against reference;
- coverage;
- error stratified by motion/contact/context.

### Temporal layer

- event sensitivity and precision;
- false parent episodes per observed day;
- onset delay;
- boundary error / interval IoU;
- duration error;
- recovery-time error;
- burden error;
- performance versus usable-data fraction.

### Predictive/AUD model layer

- subject-independent AUROC/AUPRC where appropriate;
- sensitivity/specificity and predictive values under intended prevalence;
- calibration curve, intercept/slope and Brier score for any probability model;
- subgroup/applicability performance;
- prediction-horizon correctness;
- external validation before A5.

### Clinical surfacing layer

A6/A7 require evidence that the surfacing/action strategy has acceptable benefit, false-alert burden and safety relative to existing care. Accuracy alone is insufficient; decision-curve or prospective workflow evidence may be appropriate [36,38].

---

# 20. Consolidated Parameter and Provenance Registry

Important parameters are not anonymous constants.

| Parameter family | Examples | Provenance class | Rule |
|---|---|---|---|
| Sensor technical | ECG 512 Hz; EDA 32 Hz; TMP117 1 Hz; IMU 200 Hz | hardware/engineering | version with acquisition config |
| Physiological method | cvxEDA response formulation; tremor 4.4–10 Hz; 5-min conventional HRV semantics | literature-supported | cite source; change only deliberately |
| Device-specific | SQI thresholds, LED currents, detector spacing, contact-impedance boundaries, thermal time constant | target-device empirical | estimate on native device data |
| Observation uncertainty | per-feature error variance vs quality/context | public + target empirical | calibrate against reference |
| Confound model | activity→HR response, posture effects, thermal transfer | population + patient empirical | update only on eligible data |
| Patient adaptive | stable baseline, context residual scale, circadian curve, recovery distribution | longitudinal patient data | guarded adaptation |
| Temporal operating | BOCPD hazard, entry/exit, persistence, gap/merge thresholds | event-level empirical | choose on held-out subjects under false-events/day constraint |
| AUD target | model coefficients/thresholds | target-specific clinical dataset | exact target/reference/horizon required |
| Authority | CEM, reference standard, applicability, max authority | evidence registry | literature/governance, not learned from sensor confidence |

A parameter registry entry includes `value`, `units`, `scope`, `source`, `population/device`, `estimation method`, `version`, and `effective date`.

---

# 21. Hardware-Change Justification

| Current configuration | Final change | Requirement | Gain | Cost/complexity | Status |
|---|---|---|---|---|---|
| One wrist IMU co-located with PPG | add second ICM-42688-P at chest pod | posture/body motion must be observed independently while wrist IMU remains local to PPG/tremor | reliable trunk posture/transitions, whole-body motion, respiratory-motion proxy, cleaner confound ownership | one low-cost IMU + data/power | **essential for robust posture-aware inference** |
| Skin TMP117 only | add outward-facing SHT45 ambient T/RH | skin temperature/EDA are environmentally confounded and one skin sensor cannot identify ambient effects | environmental observability, thermal-transfer characterization, better ambiguity handling | very small, low-power sensor | **materially beneficial; included** |
| Candidate dual-PD PPG geometry unspecified | retain dual VEMD8080, select near/far spacing by native characterization criterion | spatial redundancy should solve real coupling/motion-quality problems | device-specific spatial consistency and contact/motion evidence | mechanical characterization | **included** |
| One centralized conceptual acquisition node | use local chest + wrist acquisition pods with shared hardware timebase | analog integrity + cross-modal timing | short analog paths and deterministic synchronization | second MCU/digital tether | **essential to complete architecture** |

No other candidate component is replaced. A direct alcohol sensor, respiratory belt, PSG system, or research-grade reference sensor is not made part of normal wearable operation; these may be used as **reference standards during evidence collection** when the target requires them.

---

# 22. End-to-End Worked Scenarios

The following traces demonstrate actual inference semantics.

## Case A — genuine autonomic change at rest

**Observed:** clean ECG/PPG/EDA, minimal wrist/chest motion, stable seated posture, HR elevated relative to context/patient baseline, RMSSD suppressed, EDA phasic/tonic activation, no large ambient change.

1. Acquisition integrity passes; clocks synchronized.
2. ECG, PPG and EDA feature authorities are valid.
3. ECG yields valid RR/NN intervals; PPG pulse agrees with ECG; EDA decomposition is stable.
4. Activity/posture model cannot explain the cardiac change; thermal/activity evidence does not explain EDA.
5. Context = awake, seated/rest; circadian expectation is known.
6. Personal model yields substantial cardiac and electrodermal residuals.
7. ECG/PPG are treated as redundant cardiac support; EDA is an independent pathway.
8. Temporal engine confirms a sustained parent autonomic episode and tracks recovery.
9. No independent AUD-specific context is available.
10. Final authority: **A3 physiological state**. AUD target claims abstain.

**Clinician-facing output:**

> Sustained cardiac and electrodermal autonomic activation is present relative to the patient's expected resting physiology. Physical activity and posture do not explain the observed change. AUD-specific attribution is not supported by the available context.

## Case B — activity-driven physiology

**Observed:** HR/EDA resemble Case A, but chest/wrist IMUs show sustained moderate-to-high physical activity followed by recovery.

1. Signal quality is valid; this is not artifact merely because motion exists.
2. HR rise is inside the calibrated activity-response distribution; post-exercise recovery behaves as expected.
3. EDA is real but physical/thermal sweating is a plausible contributor; the system does not subtract it into a fabricated “stress residual.”
4. Parent state is classified as activity-associated physiology or `AMBIGUOUS` for EDA-specific autonomic attribution.
5. No AUD episode is created.

**Output:**

> Cardiovascular activation is consistent with the measured physical activity and recovery state. Electrodermal activation is present but cannot be separated confidently from activity/thermoregulatory effects.

## Case C — corrupted PPG but clean ECG

**Observed:** optical saturation/motion contamination causes poor PPG; chest ECG is excellent.

1. PPG pulse/amplitude authorities become `UNUSABLE`/`DEGRADED` with motion/coupling reason codes.
2. ECG R peaks and NN intervals remain valid.
3. Cardiac pathway uses ECG; PPG contributes no fake neutral vote.
4. HR/HRV continue if ECG requirements are met.
5. Pulse-amplitude/peripheral vascular features become unavailable.
6. Multimodal targets that do not require PPG may continue; targets needing PPG-specific evidence downgrade/abstain.

**Output:** the system degrades gracefully and explicitly reports reduced optical coverage; it does not mark the patient's pulse “normal.”

## Case D — strong high-quality modality disagreement

**Observed:** high-quality ECG indicates stable cardiac rhythm while high-quality PPG shows a persistent beat/pulse discrepancy beyond the known sync/arrival distribution; EDA is elevated.

1. Both signals pass technical QC.
2. Synchronization residual is verified within specification.
3. The cardiac redundancy layer refuses precision fusion and emits `HIGH_QUALITY_ECG_PPG_CONFLICT`.
4. Possible physiological or mechanical explanations are preserved rather than selected without evidence.
5. Any target requiring a coherent cardiac construct is downgraded.
6. EDA evidence remains separately valid.

**Output:**

> High-quality ECG and optical pulse measurements are discordant beyond the device's expected agreement range. Cardiac fusion is withheld; electrodermal activation remains independently supported. Higher-level interpretation is limited pending resolution of the conflict.

## Case E — immature personal baseline

**Observed:** valid HR/EDA deviations relative to population/context expectation, but only sparse patient history exists.

1. Technical and physiological processing proceeds normally.
2. Baseline maturity for this context is insufficient for a patient-specific z-score with acceptable uncertainty.
3. The system uses the population/context prior and reports a wide patient-reference uncertainty.
4. It may form a generic physiological episode if the absolute/context evidence is strong.
5. It does **not** claim “unusual for this patient.”

**Output:**

> The observed physiology is elevated relative to the current population/context reference. The personal baseline is not yet mature enough to support a strong patient-relative abnormality statement.

## Case F — strong physiological episode with no AUD-specific context

**Observed:** an excellent 45-minute resting autonomic episode with cardiac + EDA support and clear slow recovery. No cue, recent drinking, cessation, withdrawal-risk, or treatment event is known.

1. A3 physiological evidence is strong.
2. Persistence raises temporal support but does not create target specificity.
3. Withdrawal, cue, intoxication and relapse target engines lack essential context.
4. Authority ceiling prevents conversion of the episode into an AUD claim.

**Output:**

> A sustained multimodal autonomic episode occurred and recovered over 45 minutes. The episode is strongly supported physiologically; its AUD-specific meaning is indeterminate from the available evidence.

## Case G — withdrawal-risk context + persistent activation + tremor evidence

**Observed:** independently documented recent alcohol cessation, sustained resting HR/EDA activation, low activity, valid wrist tremor phenotype in the withdrawal-relevant band, persistent episode over hours.

1. All primary measurements pass QC.
2. Activity/posture do not explain the episode.
3. Tremor and autonomic activation are independent physiological families.
4. Temporal engine shows persistence and a worsening then stabilizing trajectory.
5. Withdrawal target context requirement is satisfied.
6. Published evidence supports compatibility but wearable physiology lacks sufficient specificity for autonomous diagnosis.
7. Authority ceiling = **A4** unless stronger, claim-specific clinical corroboration/protocol exists.

**Output:**

> Persistent multimodal autonomic activation and a compatible tremor phenotype are present in the documented withdrawal-risk interval. The pattern is compatible with withdrawal-related physiological activation, but it does not establish withdrawal diagnosis or severity. Clinical withdrawal assessment remains the appropriate higher-authority reference.

## Case H — known alcohol cue with autonomic response

**Observed:** protocol logs an alcohol cue; pre-cue baseline is valid; HR/HRV and EDA change after the cue and recover; neutral-cue response is smaller.

1. Cue timing is independent of physiology, so there is no circularity.
2. Target engine computes pre/post and alcohol-minus-neutral response vectors.
3. Temporal engine estimates cue-to-response latency, peak and recovery.
4. Output may reach A4 `ALCOHOL_CUE_AUTONOMIC_REACTIVITY`.
5. Subjective craving is not inferred; self-report, if available, is attached as a distinct reference.

**Output:**

> An autonomic response occurred after the independently recorded alcohol cue and exceeded the patient's neutral-cue response. This supports alcohol-cue autonomic reactivity; physiology alone does not establish subjective craving or relapse risk.

## Case I — nocturnal autonomic recovery disturbance

**Observed:** mature sleep baseline; reliable sleep interval; first three hours show higher HR and lower RMSSD than comparable nights, with incomplete overnight recovery.

1. Sleep context is derived independently enough for cardiac interpretation.
2. Patient-relative nocturnal deviation is strong.
3. Temporal model shows incomplete recovery and elevated nightly burden.
4. If no alcohol exposure is independently known: A3 `NOCTURNAL_AUTONOMIC_RECOVERY_DISTURBANCE`.
5. If recent alcohol exposure is independently documented: A4 wording may describe the pattern as alcohol-associated, still not proof of intoxication or dose.

**Output without exposure context:**

> Nocturnal cardiac autonomic recovery is reduced relative to the patient's comparable-night baseline, with incomplete recovery by the end of the monitored sleep interval. Cause is not established.

## Case J — missing/partially available modalities

**Observed:** ECG unavailable from lead-off; PPG is high quality at rest; EDA valid; IMUs valid; temperature missing.

1. ECG-specific HRV becomes `MISSING`, not normal.
2. PPG supports pulse rate and, if validated, PRV under its own semantics; it does not silently manufacture ECG HRV.
3. Cardiac timing pathway continues with reduced construct coverage.
4. EDA/motion pathways remain valid.
5. A generic autonomic episode may still be evaluable.
6. Any target whose primary evidence requires ECG HRV or temperature becomes insufficient/partially observed according to its registry.

**Output:**

> Cardiac pulse rate and electrodermal evidence remain available, but ECG-derived HRV and skin-temperature evidence are unavailable. Higher-level claims requiring those primary measurements are not evaluated; no missing channel is treated as negative evidence.

---

# 23. Complete Clinician-Facing Output Semantics

The system emits claim objects, not a dashboard-only number.

```yaml
claim:
  id: withdrawal_related_autonomic_activation
  class: AUD_COMPATIBILITY
  effective_interval: ...

observation:
  physiological_state: SUSTAINED_MULTIMODAL_AUTONOMIC_ACTIVATION
  patient_relative_deviation: ...
  temporal_episode_id: ...

measurement_evidence:
  ECG: VALID
  PPG: DEGRADED
  EDA: VALID
  wrist_IMU: VALID
  chest_IMU: VALID

context:
  withdrawal_risk: PRESENT
  activity: REST
  posture: STABLE
  time_since_last_drink: ...

supporting_evidence:
  - cardiac_activation
  - electrodermal_activation
  - tremor_phenotype
  - persistence

contradictory_evidence:
  - ...

missing_evidence:
  - clinical_withdrawal_assessment

uncertainty:
  measurement: LOW
  confound: LOW
  patient_reference: MODERATE
  target_ambiguity: HIGH
  model_epistemic: MODERATE

clinical_evidence:
  CEM: 2
  reference_standard_quality: ...
  applicability: MATCHED_FOR_ASSOCIATION
  calibration: NOT_APPLICABLE

authority:
  level: A4
  ceiling: A4
  limiting_reasons:
    - AUD_SPECIFICITY_LIMITED
    - DIAGNOSTIC_VALIDATION_ABSENT
    - CLINICAL_CORROBORATION_MISSING

allowed_statement: >
  Sustained multimodal autonomic activation and a compatible tremor phenotype
  are present in the documented withdrawal-risk interval. The pattern is
  compatible with withdrawal-related physiological activation, but is not
  diagnostically specific.

abstention_for_higher_claims:
  - PROBABILITY_NOT_VALIDATED
  - DIAGNOSIS_NOT_SUPPORTED

audit:
  acquisition_regime: ...
  patient_reference_version: ...
  temporal_model_version: ...
  target_registry_version: ...
  authority_policy_version: ...
```

The clinician can trace every statement backward to the exact episodes, pathway evidence, features, quality decisions, sensor configuration and raw recording references that support it.

---

# 24. Internal Scientific Stress Test

The completed architecture has been checked against the major failure modes identified across the source corpus:

- **Circular reasoning:** AUD/stress/cue contexts are independent inputs; physiology does not create the context used to explain itself.
- **Evidence double-counting:** correlated cardiac/EDA features are grouped by dependency; ECG/PPG cardiac timing is fused as redundancy.
- **Duplicate responsibility:** one expected-physiology model serves confound, context and personalization queries.
- **Later-layer bypass:** invalid quality authority is inherited downstream and cannot be restored.
- **Window leakage:** event/subject split rules are explicit.
- **Uncalibrated probabilities:** locked behind A5 criteria.
- **Independence assumptions:** dependencies are explicit; high-quality conflicts are preserved.
- **Hidden imputation:** missing is a first-class state.
- **Future leakage:** causal and retrospective versions are separate.
- **Event/baseline confusion:** episode masks guard stable baseline updates.
- **Missing gaps as recovery:** explicitly prohibited.
- **Artifact vs real confound physiology:** owned by different layers.
- **Slow physiology as noise:** explicitly preserved.
- **ECG HRV vs PPG PRV equivalence:** explicitly prohibited.
- **Activity mistaken for autonomic target:** conditional expectation/confound layer prevents this where identifiable.
- **Generic stress interpreted as AUD:** target context required.
- **Craving inferred without evidence:** physiology-only craving output prohibited.
- **Relapse overclaim:** no current numeric relapse prediction.
- **Withdrawal overclaim:** A4 compatibility ceiling under present evidence.
- **Exact BAC from physiology:** prohibited.
- **Clinical action from measurement strength:** separate utility boundary.
- **One dataset validating everything:** evidence matrix assigns a reference per construct.
- **Target hardware mistaken for independent validation:** explicitly separated.
- **Proxy labels mistaken for ground truth:** target semantics inherit the reference standard.

No unresolved scientific gap is converted into fake certainty. Where evidence cannot distinguish competing explanations, the final system behavior is ambiguity or abstention.

---

# 25. References

## Hardware and sensing

[1] Analog Devices. **MAX86176/MAX30005: Ultra-Low-Power, Optical PPG and Single-Lead ECG AFE**, Rev. 4. 2024. https://www.analog.com/media/en/technical-documentation/data-sheets/max86176-max30005.pdf

[2] Analog Devices. **MAX86173 Dual-Channel Optical AFE with High Signal-to-Noise Ratio for Wearable Health.** https://www.analog.com/en/products/MAX86173.html

[3] ams OSRAM. **SFH 7016 Multi Chip LED — Green 530 nm, Red 655 nm, Infrared 940 nm.** https://ams-osram.com/products/leds/multi-color-leds/osram-multi-chip-led-sfh-7016

[4] Vishay Semiconductors. **VEMD8080 Silicon PIN Photodiode.** https://www.vishay.com/docs/84565/vemd8080.pdf

[5] Analog Devices. **AD5940/AD5941 High-Precision, Impedance & Electrochemical Front End**, Rev. G. 2026. https://www.analog.com/en/products/ad5940.html

[6] Texas Instruments. **TMP117 High-Accuracy, Low-Power, Digital Temperature Sensor.** https://www.ti.com/product/TMP117

[7] TDK InvenSense. **ICM-42688-P 6-Axis MotionTracking Device.** https://product.tdk.com/en/search/sensor/mortion-inertial/imu/info?part_no=ICM-42688-P

[8] Espressif Systems. **ESP32-S3-WROOM-1 & ESP32-S3-WROOM-1U Datasheet.** https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf

[9] Sensirion. **SHT45 Digital Humidity and Temperature Sensor.** https://sensirion.com/products/catalog/SHT45

## Signal processing, quality, temporal methods and datasets

[10] Orphanidou C, Bonnici T, Charlton P, Clifton D, Vallance D, Tarassenko L. **Signal-quality indices for the electrocardiogram and photoplethysmogram: derivation and applications to wireless monitoring.** *IEEE J Biomed Health Inform.* 2015;19(3):832-838. doi:10.1109/JBHI.2014.2338351.

[11] Pan J, Tompkins WJ. **A real-time QRS detection algorithm.** *IEEE Trans Biomed Eng.* 1985;32(3):230-236. doi:10.1109/TBME.1985.325532.

[12] Greco A, Valenza G, Lanata A, Scilingo EP, Citi L. **cvxEDA: A Convex Optimization Approach to Electrodermal Activity Processing.** *IEEE Trans Biomed Eng.* 2016;63(4):797-804. doi:10.1109/TBME.2015.2474131.

[13] Task Force of the European Society of Cardiology and the North American Society of Pacing and Electrophysiology. **Heart rate variability: standards of measurement, physiological interpretation and clinical use.** *Circulation.* 1996;93(5):1043-1065. PMID:8598068.

[14] Schmidt P, Reiss A, Dürichen R, Marberger C, Van Laerhoven K. **Introducing WESAD, a Multimodal Dataset for Wearable Stress and Affect Detection.** *ICMI 2018.* doi:10.1145/3242969.3242985. Dataset DOI:10.24432/C57K5T.

[15] Pimentel MAF, Johnson AEW, Charlton PH, Clifton DA. **BIDMC PPG and Respiration Dataset.** PhysioNet v1.0.0, 2018. doi:10.13026/C2208R. Associated method: doi:10.1109/TBME.2016.2613124.

[16] Moody GB, Mark RG. **MIT-BIH Arrhythmia Database.** PhysioNet v1.0.0; original overview *IEEE Eng Med Biol Mag.* 2001;20(3):45-50. Dataset doi:10.13026/C2F305.

[17] Reiss A, Indlekofer I, Schmidt P, Van Laerhoven K. **PPG-DaLiA: PPG Dataset for Motion Compensation and Heart Rate Estimation in Daily Life Activities.** UCI Machine Learning Repository. 2019. doi:10.24432/C53890.

[18] Reiss A, Indlekofer I, Schmidt P, Van Laerhoven K. **Deep PPG: Large-Scale Heart Rate Estimation with Convolutional Neural Networks.** *Sensors.* 2019;19(14):3079. doi:10.3390/s19143079.

[19] Carver S, Norouzi N, Bromberg S, et al. **Time-frequency visualization of alcohol withdrawal tremors.** *IEEE EMBC.* 2014:5474-5477. doi:10.1109/EMBC.2014.6944865.

[20] Aarabi P, Norouzi N, Dear T, et al. **A quantitative evaluation of alcohol withdrawal tremors.** *IEEE EMBC.* 2015:6215-6218. doi:10.1109/EMBC.2015.7319812.

[21] Adams RP, MacKay DJC. **Bayesian Online Changepoint Detection.** arXiv:0710.3742. doi:10.48550/arXiv.0710.3742.

[22] Killick R, Fearnhead P, Eckley IA. **Optimal Detection of Changepoints With a Linear Computational Cost.** *J Am Stat Assoc.* 2012;107(500):1590-1598. doi:10.1080/01621459.2012.737745.

## AUD-specific physiology

[23] Cheng YC, Huang YC, Huang WL. **Heart rate variability as a potential biomarker for alcohol use disorders: A systematic review and meta-analysis.** *Drug Alcohol Depend.* 2019;204:107502. doi:10.1016/j.drugalcdep.2019.05.030.

[24] Ralevski E, Petrakis I, Altemus M. **Heart Rate Variability in Alcohol Use: A Review.** *Pharmacol Biochem Behav.* 2019;176:83-92. doi:10.1016/j.pbb.2018.12.003.

[25] Kaczor EE, Golleru M, Carter DC, Painter OB, et al. **Detecting Ethanol Intoxication and Impairment Using Wearable Biosensors.** *HICSS Proceedings.* 2026:3532-3542. PMID:41550855; PMCID:PMC12805382.

[26] van Lier HG, Noordzij ML, Pieterse ME, et al. **An ideographic study into physiology, alcohol craving and lapses during one hundred days of daily life monitoring.** *Addict Behav Rep.* 2022;16:100443. doi:10.1016/j.abrep.2022.100443.

[27] Alinia P, Sah RK, McDonell M, et al. **Associations Between Physiological Signals Captured Using Wearable Sensors and Self-reported Outcomes Among Adults in Alcohol Use Disorder Recovery.** *JMIR Form Res.* 2021;5(7):e27891. doi:10.2196/27891.

[28] Pietilä J, Helander E, Korhonen I, Myllymäki T, Kujala UM, Lindholm H. **Acute Effect of Alcohol Intake on Cardiovascular Autonomic Regulation During the First Hours of Sleep in a Large Real-World Sample.** *JMIR Ment Health.* 2018;5(1):e23. doi:10.2196/mental.9519.

[29] de Zambotti M, Willoughby AR, Baker FC, Sugarbaker DS, Colrain IM. **Cardiac autonomic function during sleep: effects of alcohol dependence and evidence of partial recovery with abstinence.** *Alcohol.* 2015;49(4):409-415. doi:10.1016/j.alcohol.2014.07.023.

[30] Eddie D, Wieman S, Pietrzak A, Zhai X. **In natura heart rate variability predicts subsequent alcohol use in individuals in early recovery from alcohol use disorder.** *Addict Biol.* 2023;28(8):e13306. doi:10.1111/adb.13306. See also Witteman J, et al. *Psychopharmacology.* 2015;232(20):3685-3696. doi:10.1007/s00213-015-4027-6.

## Clinical evidence, authority and decision support

[31] American Society of Addiction Medicine. **The ASAM Clinical Practice Guideline on Alcohol Withdrawal Management.** 2020. https://www.asam.org/quality-care/clinical-guidelines/alcohol-withdrawal-management-guideline

[32] Roberts AE, Yarbrough P, Vargas J, et al. **The Association of Alcohol Withdrawal Severity Scales that Incorporate Vital Signs and Withdrawal Outcomes: A Multicenter Cohort Study.** *J Addict Med.* 2026. doi:10.1097/ADM.0000000000001698. PMID:42360115.

[33] U.S. Food and Drug Administration. **Clinical Decision Support Software: Guidance for Industry and FDA Staff.** Final Guidance, January 2026. https://www.fda.gov/regulatory-information/search-fda-guidance-documents/clinical-decision-support-software

[34] International Medical Device Regulators Forum. **Software as a Medical Device (SaMD): Clinical Evaluation.** IMDRF/SaMD WG/N41FINAL:2017. https://www.imdrf.org/documents/software-medical-device-samd-clinical-evaluation

[35] Van Calster B, McLernon DJ, van Smeden M, Wynants L, Steyerberg EW, et al. **Calibration: the Achilles heel of predictive analytics.** *BMC Med.* 2019;17:230. doi:10.1186/s12916-019-1466-7.

[36] Vickers AJ, Elkin EB. **Decision curve analysis: a novel method for evaluating prediction models.** *Med Decis Making.* 2006;26(6):565-574. doi:10.1177/0272989X06295361.

[37] Collins GS, Moons KGM, Dhiman P, Riley RD, Beam AL, et al. **TRIPOD+AI statement: updated guidance for reporting clinical prediction models that use regression or machine learning methods.** *BMJ.* 2024;385:e078378. doi:10.1136/bmj-2023-078378. See also Moons KGM, et al. **PROBAST+AI.** *BMJ.* 2025;388:e082505.

[38] Vasey B, Nagendran M, Campbell B, et al. **DECIDE-AI: reporting guideline for the early-stage clinical evaluation of AI-driven decision support systems.** *Nat Med.* 2022;28:924-933. doi:10.1038/s41591-022-01772-9.

---

# 26. Internal Repository Corpus Synthesized

This document is a synthesis—not a concatenation—of the complete current research corpus in this repository:

- `docs/README.md`
- `docs/00-system-overview/processing-pipeline.md`
- `docs/01-signal-foundation/01-signal-trustworthiness.md`
- `docs/01-signal-foundation/02-physiological-signal-decomposition.md`
- `docs/02-physiological-interpretation/03-confound-aware-monitoring.md`
- `docs/02-physiological-interpretation/04-context-aware-interpretation.md`
- `docs/03-cross-modal-reasoning/05-reliability-aware-multimodal-fusion.md`
- `docs/03-cross-modal-reasoning/06-personalized-longitudinal-modeling.md`
- `docs/03-cross-modal-reasoning/07-temporal-physiological-event-modeling.md`
- `docs/04-aud-specific-inference/08-aud-specific-physiological-meaning.md`
- `docs/05-clinical-boundary/09-evidence-authority-aware-clinical-interpretation.md`

The principal cross-document conflicts have been resolved by this final architecture: a single owner for expected physiology; explicit separation of physical confounds from interpretive context; dependency-aware fusion rather than sensor voting; temporal ownership of episodes and burden; baseline protection against pathological adaptation; target-specific AUD inference rather than a universal AUD classifier; and an authority ceiling that prevents strong physiology from exceeding weak clinical specificity.

---

## Final System Position

The system to build is therefore not a wearable that asks every sensor to vote for an AUD label. It is a **hierarchical physiological evidence system** in which each inferential step must inherit the limits of the evidence beneath it.

It preserves raw observability, makes feature use conditional on technical authority, distinguishes artifact from real confound-driven physiology, compares real physiology with context- and patient-specific expectation, combines modalities according to physiological dependence, treats time and recovery as first-class evidence, introduces AUD semantics only for precisely defined targets with independently satisfied context, and finally caps the clinician-facing statement at the weakest non-compensable evidentiary link.

That produces the desired end-to-end behavior:

```text
This measurement is trustworthy.
This is the physiological component it supports.
This portion is explained by physical context; this portion is not.
This is how unusual the remaining physiology is for this patient here and now.
These modalities agree, conflict, or are missing for known reasons.
This is the episode that occurred, how long it persisted, and how it recovered.
This is the AUD-specific interpretation the evidence can support—and no stronger.
This is the clinical authority of that statement, why it is capped there, and what evidence is missing.
```

Where the science cannot distinguish competing explanations, the completed system says so. Where a target lacks individual clinical validity, the completed system abstains. That restraint is not a limitation of the architecture; it is one of its core technical properties.
