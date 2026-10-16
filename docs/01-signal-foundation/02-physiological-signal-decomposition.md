# Physiological Signal Decomposition Solution

## Purpose

This document specifies the **Physiological Signal Decomposition layer** for the current multimodal physiological monitoring system.

The layer answers one question:

> **Once a physiological signal is trustworthy, what part of that signal represents meaningful physiology, and what part represents baseline, slow drift, trend, tonic/DC activity, residual noise, or another component that should not be interpreted in the same way?**

This is an engineering design document, not a general signal-processing survey. The goal is to turn established physiological signal-processing methods into a concrete implementation for the acquired streams:

- ECG
- optical PPG (including red/IR channels where available)
- electrodermal activity / GSR
- skin temperature
- IMU acceleration and angular velocity as the kinematic auxiliary stream

The decomposition layer begins **after signal-quality/trustworthiness gating** and ends when it has emitted physiologically meaningful component signals. It does **not** determine what caused those changes, whether they are due to activity or another confound, what surrounding context means, or what any change means clinically in AUD/SUD.

---

# 1. Design principles

## 1.1 Separation is preferable to deletion

A slow component is not automatically noise.

For example:

- the slowly varying PPG component contains non-pulsatile optical attenuation, but can also contain respiration-, sympathetic-, venous-, and thermoregulatory-related variation [4,5];
- EDA tonic activity is genuine physiology rather than a nuisance baseline [1,2];
- skin temperature changes naturally on several time scales, from tens of seconds through circadian time scales [16-18];
- ECG contains diagnostically and physiologically meaningful low-frequency morphology, so aggressive high-pass filtering can create false ST displacement [9,11,12].

Therefore the architecture should prefer:

```text
signal
├── slow / tonic / DC component
├── dynamic / phasic / pulsatile component
├── event or beat series
└── residual
```

rather than:

```text
signal
↓
single aggressive filter
↓
"cleaned" signal
```

unless literature demonstrates that a removed band contains no needed information.

---

## 1.2 Two parameter classes

Every numerical parameter in the recommended implementation is explicitly classified as:

**[RS] Research-supported parameter**  
A parameter explicitly used, recommended, or physiologically justified in published literature or the reference algorithm.

**[ES] Engineering starting parameter**  
A practical initial value inferred from literature and system constraints. It must be calibrated against data from the actual hardware and placement before being treated as final.

---

## 1.3 Real-time and retrospective processing may differ

Some of the strongest published methods are zero-phase, bidirectional, or batch optimization methods.

The architecture therefore distinguishes:

- **real-time causal/fixed-lag decomposition**, used by the continuously operating system;
- **offline/high-fidelity decomposition**, used for retrospective validation, model development, and checking whether the real-time approximation is destroying useful physiology.

---

## 1.4 Never use a detection-only filtered branch as the physiological waveform

A narrow band may be excellent for peak detection but destructive for morphology.

Examples:

- Pan-Tompkins uses an approximately 5–15 Hz QRS-detection band [13];
- PPG systolic-peak algorithms often use approximately 0.5–8 Hz [7].

Those branches are useful for event timing, but they must not replace the preserved morphology branch.

---

# 2. System-level decomposition interface

For every modality, the decomposition service accepts:

```text
trusted_segment {
    samples
    timestamps
    sampling_rate
    channel_id
    continuity_id
}
```

`continuity_id` changes whenever the upstream trustworthiness layer declares a discontinuity that should not be bridged as if samples were continuous.

The decomposition service returns modality-specific components plus generic metadata:

```text
decomposition_result {
    modality
    start_time
    end_time

    components: {...}

    algorithm_id
    algorithm_version
    parameter_set_id

    state_status:
        cold_start | warming | mature | reacquired

    residual_summary: {...}
}
```

The decomposition layer should preserve the original trusted waveform in storage or make it addressable by reference. Decomposition must be reproducible from the trusted source signal.

---

# 3. Electrodermal activity / GSR

## 3.1 What the measured signal contains

Skin conductance is not one homogeneous signal. The standard physiological decomposition is:

\[
y(t) = r(t) + t(t) + \epsilon(t)
\]

where:

- \(r(t)\) = phasic skin-conductance component;
- \(t(t)\) = tonic skin-conductance component;
- \(\epsilon(t)\) = residual/model error and measurement noise.

Greco et al.'s cvxEDA explicitly models EDA this way [1]. Benedek and Kaernbach similarly separate continuous tonic and phasic activity through nonnegative deconvolution [2].

The phasic component is itself modeled as the response of sweat-gland dynamics to a sparse sudomotor neural driver. A common impulse-response representation is a Bateman-type biexponential:

\[
h(t) = \left(e^{-t/\tau_0} - e^{-t/\tau_1}\right)u(t)
\]

and:

\[
r(t) = h(t) * p(t)
\]

where \(p(t)\) is the sparse nonnegative phasic driver.

### Components

| Component | Physiological origin | Time scale | Keep? |
|---|---|---:|---|
| Tonic EDA / SCL | slowly varying conductance level | tens of seconds to minutes and longer | **keep separately** |
| Phasic EDA / SCR | transient sudomotor responses | seconds | **keep separately** |
| Sparse driver | inferred sudomotor activation producing phasic EDA | impulses / short bursts | **keep** |
| Linear drift term | slow model trend not adequately represented by tonic spline | long | keep internally; expose for audit |
| Residual | model mismatch + unmodeled measurement structure | sample scale | keep internally + summary |

Benedek and Kaernbach report that, after deconvolution, overlapping responses become compact impulses with an average duration below about 2 s [2]. cvxEDA uses a physiologically motivated biexponential impulse response and smooth spline tonic component [1].

---

## 3.2 Selected method

### Primary high-fidelity method: cvxEDA

cvxEDA is selected because it:

1. explicitly separates **tonic**, **phasic**, and **residual** components;
2. uses a physiological sweat-response model;
3. imposes sparsity/nonnegativity on the inferred driver;
4. is mature, widely reused, and has an available reference implementation [1];
5. directly matches the responsibility of this layer.

The 2026 ospEDA method is a promising newer alternative that combines tonic estimation, orthogonal subspace projection, and NNLS deconvolution and reports strong performance across simulated and real datasets [3]. It should be tracked as a future benchmark, but cvxEDA remains the safer production baseline because of its maturity, clearer reference implementation, and much larger downstream usage base.

---

## 3.3 cvxEDA parameters

Reference implementation defaults are [1]:

| Parameter | Value | Class | Meaning |
|---|---:|---|---|
| \(\tau_0\) | 2.0 s default; paper explored 2–4 s | [RS] | slow Bateman time constant |
| \(\tau_1\) | 0.7 s | [RS] | fast Bateman time constant |
| tonic spline knot spacing | 10 s | [RS] | tonic flexibility |
| \(\alpha\) | 0.0008 | [RS] | sparse driver penalty |
| \(\gamma\) | 0.01 | [RS] | tonic smoothness penalty |
| signal normalization | z-score recommended by reference code | [RS] | numerical scaling |

The Greco study fixed \(\tau_1=0.7\) s and selected \(\tau_0\) in 2–4 s per subject by residual minimization, while using \(\alpha=0.0008\) and \(\gamma=0.01\) [1].

### Sampling

The acquisition rate is substantially higher than needed by the dynamics represented in the cvxEDA model.

Recommended:

- retain original trusted EDA at native rate;
- create a decomposition copy at **8 Hz [ES]**, after anti-alias low-pass filtering;
- do not delete the native-rate signal.

8 Hz is an engineering choice that leaves ample resolution for the sub-second-to-seconds dynamics implied by \(\tau_1=0.7\) s while greatly reducing convex optimization cost. Published wearable analyses commonly use low-rate EDA after appropriate preprocessing, but 8 Hz is not a universal cvxEDA requirement.

---

## 3.4 Real-time implementation

cvxEDA is fundamentally a batch convex optimization method. It is therefore not strictly sample-causal.

### Recommended real-time approximation: fixed-lag sliding cvxEDA

**[ES]**

```text
native trusted EDA
↓
anti-alias LPF
↓
resample to 8 Hz
↓
maintain 60 s rolling block
↓
run cvxEDA every 1 s
↓
emit/commit only samples older than final 5 s of block
```

Recommended starting values:

- block length: **60 s [ES]**
- update interval: **1 s [ES]**
- uncommitted right-edge guard: **5 s [ES]**
- decomposition sampling rate: **8 Hz [ES]**

Rationale:

- 60 s contains six 10-s tonic-spline knot intervals;
- the 5-s guard is longer than the dominant phasic rise/decay time constants and reduces right-edge instability;
- computation is still small at 8 Hz.

The final 5 s should be marked `provisional`. When those samples move away from the window boundary, they are recomputed and then committed.

### Offline implementation

For retrospective analysis:

- run cvxEDA on the longest contiguous trusted segment that is computationally reasonable;
- compare with blockwise results;
- optimize \(\tau_0\) within the literature-supported 2–4 s range if data justify it.

---

## 3.5 Edge and state behavior

When EDA becomes unusable:

1. stop feeding samples into the decomposition state;
2. close the current contiguous segment;
3. never let the tonic spline bridge a substantial bad-quality gap as if data existed;
4. on reacquisition, initialize a new segment.

Recommended state flags:

- `warming`: first **20 s [ES]** after reacquisition;
- `mature`: once sufficient tonic context exists;
- `provisional`: samples inside the right-edge guard.

The first 20 s is an engineering warm-up choice, not a physiological constant.

---

## 3.6 Output

```text
eda_raw_trusted              μS, native rate
eda_tonic                    μS, 8 Hz
eda_phasic                   μS, 8 Hz
eda_phasic_driver            normalized driver units, 8 Hz
eda_linear_drift             μS-equivalent, 8 Hz
eda_residual                 μS, 8 Hz
eda_component_maturity       categorical, 1 Hz
```

Do **not** discard tonic EDA. Tonic and phasic components represent different physiological dynamics and must remain individually accessible.

---

## 3.7 Residual checks

The residual is:

\[
e(t)=y(t)-\hat{t}(t)-\hat{r}(t)
\]

Inspect only decomposition adequacy:

- residual RMS relative to input RMS;
- residual autocorrelation;
- power remaining below approximately the phasic band;
- structured repeated transients that indicate under-fitting.

Do not reinterpret this residual check as a new signal-quality classifier.

---

# 4. Photoplethysmography

## 4.1 What the waveform contains

Allen describes the PPG as a pulsatile **AC component**, primarily cardiac-synchronous blood-volume variation, superimposed on a slowly varying **DC component** containing lower-frequency contributions including respiration, sympathetic activity, and thermoregulation [4].

A useful engineering model is:

\[
x_\lambda(t)=D_\lambda(t)+R_\lambda(t)+C_\lambda(t)+\epsilon_\lambda(t)
\]

for each optical wavelength \(\lambda\), where:

- \(D_\lambda(t)\) = very-slow optical baseline / trend;
- \(R_\lambda(t)\) = respiration-related and other low-frequency hemodynamic modulation;
- \(C_\lambda(t)\) = cardiac pulsatile waveform;
- \(\epsilon_\lambda(t)\) = residual high-frequency/unmodeled content.

This is intentionally more informative than simply calling everything below 0.5 Hz "baseline noise".

PPG reviews place the dominant cardiac PPG band around 0.5–5 Hz, while respiratory baseline modulation frequently falls around 0.15–0.5 Hz [5]. Charlton et al. note that very-low-frequency removal for respiratory extraction is often performed around 0.03–0.05 Hz [6].

---

## 4.2 Component bands

The following bands are selected as a decomposition convention, not as claims that physiology has hard spectral borders.

| Component | Band / time scale | Parameter status | Action |
|---|---:|---|---|
| very slow trend | <0.05 Hz | lower boundary supported by respiration-extraction literature [6] | preserve separately |
| low-frequency / respiratory modulation | 0.05–0.5 Hz | [ES] boundaries anchored by [5,6] | preserve separately |
| cardiac pulsatile branch | 0.5–8 Hz | [RS] common published PPG processing band; Elgendi uses 0.5–8 Hz [7] | preserve |
| high-frequency residual | >8 Hz | [ES] for this decomposition branch | preserve only internally / summary |

Why use 8 Hz rather than 5 Hz for the cardiac branch? The 0.5–5 Hz interval captures the main PPG energy [5], but morphology and derivatives may benefit from harmonics above 5 Hz. Elgendi et al. use a 0.5–8 Hz second-order Butterworth branch for systolic-wave detection [7]. Therefore 8 Hz is a conservative upper limit for the morphology-supporting cardiac branch at a ~100 Hz sample rate.

---

## 4.3 Selected decomposition strategy

### Real-time filter-bank decomposition

For each red/IR channel:

```text
trusted PPG
├── very-slow trend branch        LP <0.05 Hz
├── low-frequency modulation      BP 0.05–0.5 Hz
├── cardiac pulsatile waveform    BP 0.5–8 Hz
└── residual                      original - reconstructed branches
```

The filter bank is preferable to a single 0.5-Hz high-pass because respiratory and other low-frequency physiological modulation would otherwise be deleted [4-6].

### Filter family

Use second-order Butterworth sections as the starting implementation:

- cardiac branch: **2nd-order Butterworth 0.5–8 Hz [RS]** based on Elgendi's published PPG processing [7];
- low-frequency branch: **2nd-order Butterworth 0.05–0.5 Hz [ES]**;
- very-slow branch: **2nd-order Butterworth low-pass 0.05 Hz [ES]**.

Implement all IIR filters as second-order sections (SOS).

For offline processing, apply forward-backward filtering for zero-phase output where morphology timing matters. In the live path, use causal filtering and retain a known filter-delay/phase metadata field.

---

## 4.4 Local optical DC and AC/DC

For pulse oximetry or normalized pulse amplitude, "DC" is also required in a local optical sense.

Pulse-oximetry literature defines:

\[
R=\frac{(AC/DC)_{\text{red}}}{(AC/DC)_{\text{IR}}}
\]

and the device-specific calibration maps \(R\) to oxygen saturation [8].

Therefore the decomposition service should emit:

- `ppg_dc_local_red`
- `ppg_dc_local_ir`
- `ppg_ac_amplitude_red`
- `ppg_ac_amplitude_ir`
- `ppg_ratio_of_ratios`

but should **not** claim that ratio-of-ratios is a calibrated SpO₂ value unless the hardware system has an empirically validated calibration function.

### Local DC estimator

Use a beat-synchronous robust local baseline:

\[
DC_k = \mathrm{median}\{x(t): t \in \text{one pulse neighborhood}\}
\]

or a slow local low-pass.

Recommended initial implementation:

- median across **2 s [ES]** rolling window for local optical DC;
- pulse amplitude from cardiac branch, beat by beat.

The 2 s value is an engineering choice; it is deliberately distinct from the <0.05 Hz "very slow trend" output.

---

## 4.5 Beat segmentation

Use the 0.5–8 Hz branch only for beat detection.

Elgendi's method uses a zero-phase second-order 0.5–8 Hz Butterworth filter and moving-average logic [7].

Recommended:

- live: causal 0.5–8 Hz branch + peak confirmation after local maximum;
- offline: zero-phase 0.5–8 Hz branch;
- map peak times back to the minimally altered/preserved PPG to extract morphology.

Outputs:

```text
pulse_times
pulse_intervals
pulse_amplitude
pulse_waveform_segment
```

---

## 4.6 Real-time state

At ~100 Hz:

- IIR state: a few coefficients per section;
- rolling local-DC buffer: **2 s [ES]**, ~200 samples/channel;
- beat detector state: last confirmed peak + adaptive interval estimate;
- optional 10–30 s buffer for low-frequency component diagnostics.

On reacquisition:

1. initialize SOS states to the first valid sample rather than zero;
2. invalidate component outputs for **2 s [ES]**;
3. do not bridge pulse intervals across the gap;
4. start a new pulse-sequence continuity id.

---

## 4.7 Avoid destructive preprocessing

A 0.5 Hz high-pass is appropriate when the sole purpose is cardiac pulse detection, but it removes respiratory and other low-frequency modulation [5].

Therefore:

- do not define "PPG cleaned" = 0.5–8 Hz;
- define `ppg_cardiac_component` = 0.5–8 Hz;
- preserve `ppg_low_frequency_component`;
- preserve `ppg_very_slow_component`.

This distinction is essential.

---

## 4.8 Output

Per wavelength:

```text
ppg_raw_trusted                 ADC units, native rate
ppg_very_slow_trend             ADC units, native or decimated
ppg_low_frequency_component     ADC units
ppg_cardiac_component           ADC units
ppg_local_dc                    ADC units
ppg_pulse_times                 timestamps
ppg_pulse_intervals             ms
ppg_pulse_amplitude             ADC units / normalized units
ppg_residual                    ADC units
```

Across red and IR:

```text
ppg_acdc_red
ppg_acdc_ir
ppg_ratio_of_ratios
```

---

# 5. Electrocardiography

## 5.1 What the signal contains

A trusted ECG still contains several superposed elements:

\[
x(t)=b(t)+e_{\text{cardiac}}(t)+n(t)
\]

where:

- \(b(t)\) is baseline wander / slowly varying offset;
- \(e_{\text{cardiac}}(t)\) contains P, QRS, ST, and T morphology;
- \(n(t)\) is residual high-frequency/unmodeled content.

The central danger is that low-frequency ECG content is not equivalent to baseline wander.

AHA recommendations historically use a 0.05 Hz low-frequency cutoff for diagnostic ECG because stronger high-pass filtering can distort ST and T morphology [11]. Contemporary measurements continue to demonstrate substantial ST distortion at 0.5 Hz and above [12].

Therefore baseline removal must be **morphology preserving**.

---

## 5.2 Selected real-time baseline estimator

### Two-stage median baseline estimation

A widely used baseline estimator applies:

1. a median filter of approximately **200 ms [RS]**;
2. then a median filter of approximately **600 ms [RS]**;
3. subtracts the result from the original ECG.

This structure suppresses short P/QRS complexes in the baseline estimate, then suppresses longer T-wave content, leaving a slowly varying baseline estimate.

At 250 Hz:

- 200 ms = 50 samples; use **51 samples [ES]** so the window is odd;
- 600 ms = 150 samples; use **151 samples [ES]**.

For real-time processing, use a **fixed-lag centered implementation [ES]** with approximately half the longest window as algorithmic latency rather than a purely trailing median, because the centered form is less temporally biased.

Approximate fixed lag:

- ~300 ms plus implementation overhead.

This is acceptable for decomposition and morphology extraction even if another layer has a lower-latency beat trigger.

### Why not simply high-pass at 0.5 Hz?

Because 0.5 Hz and higher high-pass filtering can create clinically significant ST deformation [11,12].

### Why not only 0.05 Hz high-pass?

It is safer for morphology but often inadequate at removing substantial baseline wander [9,11].

---

## 5.3 Offline high-fidelity baseline

Use **cubic-spline / morphology-aware baseline estimation** as an offline comparator.

Meyer and Keiser introduced cubic-spline baseline estimation for ECG [10]. Sörnmo later demonstrated adaptive/time-varying baseline filtering designed around ECG low-frequency properties and linear phase [9]. Froning et al. specifically discuss limitations and correct implementation of cubic-spline estimation under exercise/wander conditions.

Offline baseline anchors should be chosen from isoelectric portions identified relative to detected beats, then interpolated smoothly.

The offline result is used to:

- quantify how much the real-time median estimator alters ST/T morphology;
- tune the live estimator;
- provide retrospective high-fidelity data.

---

## 5.4 Cardiac event branch

Use a separate detector branch:

```text
baseline-corrected ECG
↓
5–15 Hz detector band
↓
derivative / energy / adaptive threshold
↓
R-peak timestamps
```

Pan and Tompkins use an approximately 5–15 Hz detection band and adaptive real-time logic [13].

Important:

> The 5–15 Hz signal is a **detector feature**, not the physiological ECG output.

Once an R peak is detected, morphology is extracted from the morphology-preserving ECG branch.

---

## 5.5 Beat and variability components

The decomposition layer should emit the beat series:

```text
R_peak_times
RR_intervals
heart_rate_series
```

It does not need to interpret HRV, but it must preserve the timing needed for later analysis.

For reference, standard short-term HRV spectral bands are:

- VLF: approximately 0.0033–0.04 Hz;
- LF: 0.04–0.15 Hz;
- HF: 0.15–0.4 Hz [14].

This is another reason not to confuse low-frequency cardiac modulation with baseline wander.

---

## 5.6 Real-time algorithm

```text
INPUT: trusted ECG at 250 Hz

1. Feed ECG to a 51-sample median filter.
2. Feed that result to a 151-sample median filter.
3. Treat the second median output as baseline estimate B(t).
4. Compute morphology-preserving ECG:
       E(t) = X(t) - B(t)
5. In parallel, derive a 5–15 Hz detector branch from E(t).
6. Detect R peaks with adaptive thresholds.
7. Map R-peak times back to E(t).
8. Emit baseline, morphology-preserving ECG, beat timestamps, RR series.
```

---

## 5.7 State and gap behavior

On bad-quality interruption:

- close the RR sequence;
- do not calculate an RR interval across the gap;
- clear median windows;
- on reacquisition, refill windows.

Warm-up:

- baseline estimator valid after ~600 ms of contiguous data;
- detector branch can begin earlier, but mark first **1 s [ES]** as reacquisition/transient.

If the upstream system provides only short isolated missing samples, they should normally have been resolved before this layer. The decomposition layer should not silently interpolate long gaps.

---

## 5.8 Output

```text
ecg_raw_trusted              mV or ADC units, 250 Hz
ecg_baseline                 same units, 250 Hz
ecg_morphology_component     same units, 250 Hz
ecg_r_peak_times             timestamps
ecg_rr_intervals             ms
ecg_detector_branch          internal only
ecg_residual_summary         internal diagnostic
```

---

# 6. Skin temperature

## 6.1 Why temperature needs a multi-timescale model

Skin temperature is inherently slow.

It reflects:

- local heat transfer;
- skin perfusion and vasomotor state;
- thermoregulation;
- circadian rhythm;
- environmental thermal exchange.

Fingertip sympathetic vasoconstriction can produce measurable temperature decreases with a delay of about 15 s [15]. Wrist-temperature work also shows useful information in both absolute temperature and temperature change, and one study used 10-s means/differentials to suppress instantaneous noise [16].

Distal skin temperature also has strong circadian rhythmicity that can be represented by cosinor parameters such as mesor, amplitude, and acrophase [17,18].

Therefore:

> A long-term temperature trend must not be thrown away as "baseline noise".

---

## 6.2 Decomposition model

Use:

\[
T(t)=B_{\text{slow}}(t)+D_{\text{short}}(t)+\epsilon(t)
\]

where:

- \(B_{\text{slow}}\) = slowly varying thermal state;
- \(D_{\text{short}}\) = shorter-time-scale deviation from that state;
- \(\epsilon\) = small residual/noise.

In retrospective multiday data, optionally decompose:

\[
B_{\text{slow}}(t)
=
M
+
A\cos\left(\frac{2\pi t}{24h}+\phi\right)
+
B_{\text{noncircadian}}(t)
\]

where:

- \(M\) = mesor;
- \(A\) = circadian amplitude;
- \(\phi\) = acrophase.

---

## 6.3 Selected real-time strategy

### Step 1: short smoothing

At the current temperature sampling rate, compute a causal **10-s moving average [RS]**.

Sim et al. evaluated window durations from 1 to 60 s for temperature differential and used 10 s to reduce instantaneous noise [16].

Output:

\[
T_{10}(t)=\frac{1}{N}\sum_{i=0}^{N-1}T(t-i)
\]

### Step 2: adaptive slow baseline

Maintain an exponentially weighted moving average:

\[
B_t = \alpha T_{10,t} + (1-\alpha)B_{t-1}
\]

with:

\[
\alpha = 1-e^{-\Delta t/\tau}
\]

Use:

- \(\tau = 30\) min **[ES]** as the initial live baseline time constant.

This 30-min value is deliberately an engineering parameter, not a literature-established physiological boundary. It is chosen to make the baseline much slower than the tens-of-seconds response demonstrated in peripheral skin temperature [15,16], while preserving the baseline itself as an output rather than deleting it.

Then:

\[
D_{\text{short}}(t)=T_{10}(t)-B_t
\]

### Step 3: temperature slope

Also emit:

\[
\Delta T_{10}(t) = T_{10}(t)-T_{10}(t-10s)
\]

as a compact short-term dynamic representation.

---

## 6.4 Offline circadian decomposition

When at least a substantial fraction of 24 h is available, and preferably multiple days, fit a 24-h cosinor model:

\[
T(t)=M+A\cos\left(\frac{2\pi t}{24h}+\phi\right)+e(t)
\]

Distal skin-temperature research uses mesor, amplitude, and acrophase to characterize the circadian component [17,18].

Do not subtract this circadian component permanently from stored data. Emit it as another separated component.

---

## 6.5 State and reacquisition

Temperature baseline state should be treated conservatively because the signal changes slowly.

On short gap:

- freeze baseline state;
- restart the 10-s moving average when contiguous samples return;
- do not update EWMA using nonexistent samples.

On long gap:

- preserve the previous baseline as historical metadata;
- initialize a new active baseline from the first 10-s stable average;
- mark output `reacquired`.

Warm-up:

- 10 s for short smooth/differential;
- longer baseline maturity should be represented continuously rather than with a hard claim of full certainty.

---

## 6.6 Output

```text
temp_raw_trusted             °C, native rate
temp_smoothed_10s            °C, e.g. 1 Hz update
temp_slow_baseline           °C, 1 Hz update
temp_short_dynamic           °C, 1 Hz update
temp_delta_10s               °C / 10 s, 1 Hz update
temp_circadian_fit           °C, offline
temp_circadian_mesor         °C, offline
temp_circadian_amplitude     °C, offline
temp_circadian_acrophase     time, offline
temp_residual                °C
```

---

# 7. IMU: acceleration and angular velocity

## 7.1 Scope

The IMU is not itself a physiological signal. It is included because it is an acquired data stream and because correct decomposition of acceleration into gravity and body movement is necessary before later layers can use it.

This section does not classify movement context and does not use motion to explain changes in other physiological sensors.

---

## 7.2 Acceleration components

Raw accelerometer output is approximately:

\[
\mathbf{a}_{meas}
=
\mathbf{g}_{body}
+
\mathbf{a}_{linear}
+
\mathbf{b}_{a}
+
\mathbf{n}_{a}
\]

where:

- \(\mathbf{g}_{body}\) = gravity expressed in sensor axes;
- \(\mathbf{a}_{linear}\) = dynamic body acceleration;
- \(\mathbf{b}_{a}\) = accelerometer bias;
- \(\mathbf{n}_{a}\) = residual noise.

Van Hees et al. explicitly frame raw acceleration as movement + gravity + noise and compare ENMO, high-pass, and combined methods [19]. They used a fourth-order Butterworth high-pass at 0.2 Hz for HFEN and evaluated 0.2 and 0.5 Hz in human data [19].

However, pure high-pass separation becomes unreliable during orientation changes. Since the hardware includes accelerometer and gyroscope, orientation-based gravity subtraction is preferable.

---

## 7.3 Selected orientation method

Use the Madgwick IMU orientation filter.

Madgwick et al. describe a computationally efficient quaternion orientation estimator that combines gyroscope integration with an accelerometer-derived gravity correction and performs well at low computational cost [20].

State:

\[
q_t = \text{orientation quaternion}
\]

Compute predicted gravity in sensor coordinates:

\[
\mathbf{g}_{body}
=
R(q_t)^T
\begin{bmatrix}
0\\0\\g
\end{bmatrix}
\]

Then:

\[
\mathbf{a}_{linear}
=
\mathbf{a}_{meas}-\mathbf{g}_{body}
\]

### Parameters

- sample rate: actual IMU sample rate, approximately 100 Hz in the current configuration;
- Madgwick gain \(\beta\): **0.1 [ES]** initial value, consistent with common reference implementations, but must be tuned against this hardware/placement;
- gravity constant: 9.80665 m/s² or 1 g according to chosen units.

Because there is no magnetometer in the MPU-6050 path, yaw can drift. Gravity separation depends principally on roll/pitch, so this does not prevent gravity removal.

---

## 7.4 Fallback/validation movement metrics

For validation against published accelerometry processing, calculate in parallel:

\[
ENMO = \max(\|\mathbf{a}\|-1g, 0)
\]

and optionally HFEN using a fourth-order 0.2 Hz high-pass as in van Hees et al. [19].

These should be validation/reference outputs, not the primary decomposition method.

---

## 7.5 Gyroscope decomposition

Gyroscope measurement:

\[
\boldsymbol{\omega}_{meas}
=
\boldsymbol{\omega}_{true}
+
\mathbf{b}_{g}
+
\mathbf{n}_{g}
\]

Recommended:

- maintain a slowly updated gyro-bias estimate only during periods where the orientation estimator identifies a near-static state;
- emit bias-corrected angular velocity;
- retain bias separately.

Static-state bias estimation thresholds are implementation-specific and should remain **[ES]** until validated.

---

## 7.6 State and reset

IMU decomposition is fully causal.

State:

- quaternion \(q\);
- gyro bias estimate;
- filter gain;
- previous timestamp.

On a data gap:

- if the gap is small, continue with measured \(\Delta t\);
- if the gap is large enough that integration uncertainty becomes unacceptable, preserve prior quaternion for audit but reinitialize roll/pitch from gravity once the device is sufficiently stationary.

Do not fabricate orientation through a long gap.

---

## 7.7 Output

```text
imu_accel_raw_xyz             m/s² or g, native rate
imu_gravity_xyz               same units, native rate
imu_linear_accel_xyz          same units, native rate
imu_linear_accel_magnitude    same units, native rate
imu_gyro_raw_xyz              rad/s or deg/s
imu_gyro_bias_xyz             same units
imu_gyro_corrected_xyz        same units
imu_orientation_quaternion    unit quaternion
imu_enmo                      g, optional
imu_hfen                      g, optional validation branch
```

---

# 8. Cross-modality decomposition

Cross-modality information should only be used when necessary to isolate a component, not to interpret causes.

Recommended use in this layer is intentionally limited.

Examples allowed:

- use beat timing from ECG to align PPG pulses if synchronized and if pulse segmentation is ambiguous;
- use PPG/ECG beat timing to build beat-synchronous component representations;
- use accelerometer + gyroscope together to estimate orientation and gravity.

Examples excluded from this layer:

- "PPG changed because the person moved";
- "EDA is stress rather than exercise";
- "temperature change is environmental";
- any AUD/SUD interpretation.

Those belong downstream.

---

# 9. Frequency and time-scale summary

| Stream/component | Approximate range / time scale | Preserve? | Notes |
|---|---:|---|---|
| EDA phasic response | seconds; Bateman \(\tau_1\sim0.7\) s, \(\tau_0\sim2–4\) s [1] | yes | deconvolve |
| EDA tonic | tens of seconds to minutes+ | yes | spline / slow component |
| PPG very slow trend | <0.03–0.05 Hz [6] | yes | do not call all of it noise |
| PPG respiratory/low-frequency modulation | roughly 0.15–0.5 Hz commonly observed [5] | yes | filter-bank output |
| PPG cardiac | main energy ~0.5–5 Hz; detector/morphology branch can extend to 8 Hz [5,7] | yes | pulse waveform |
| ECG baseline wander | often very low frequency | separate | avoid aggressive HPF |
| ECG morphology | spans low frequency through tens of Hz | yes | 0.5-Hz HPF can distort ST [11,12] |
| HRV LF | 0.04–0.15 Hz [14] | preserve via RR series | not baseline drift |
| HRV HF | 0.15–0.4 Hz [14] | preserve via RR series | respiration-linked modulation |
| skin-temp transient | tens of seconds+; ~15 s response lag reported in fingertip [15] | yes | use short dynamic output |
| skin-temp circadian | ~24 h [17,18] | yes | offline cosinor |
| acceleration gravity | quasi-DC in world frame; varies in sensor axes with rotation | separate | orientation-based |
| linear acceleration | broad / movement dependent | yes | gravity removed |

---

# 10. Residual analysis

The purpose of residual analysis is not to decide whether the sensor signal was good. Upstream already did that.

The residual is used only to answer:

> Did the chosen decomposition explain the signal without obviously destroying or missing structured physiology?

## EDA

\[
e = y - (tonic + phasic)
\]

Inspect:

- RMS;
- autocorrelation;
- persistent SCR-shaped residuals;
- residual drift.

## PPG

Because practical filter-bank branches overlap and causal IIR filters have phase, exact additive reconstruction is approximate.

Inspect:

- residual RMS;
- power around the cardiac fundamental;
- obvious pulse morphology remaining in the residual.

Large cardiac-periodic residual structure suggests under-separation.

## ECG

The key residual check is **morphological difference** between:

- raw-minus-baseline live output;
- offline spline/reference output.

Measure:

- beat-aligned RMS difference;
- ST/T displacement;
- QRS amplitude/width difference.

## Temperature

Inspect:

\[
e_T=T_{10}-(B_{\text{slow}}+D_{\text{short}})
\]

which should be numerically small by construction. More useful is checking whether abrupt, repeated oscillation is being hidden by the smoothing choice.

## IMU

Compare:

\[
\|\mathbf{a}_{meas}-\mathbf{g}_{body}\|
\]

against ENMO/HFEN trends as a consistency check. Do not make this a motion-quality classifier here.

---

# 11. Handling bad-quality gaps and reacquisition

A single cross-modality rule should apply:

## 11.1 When upstream says `unusable`

- stop updating the corresponding decomposition model;
- close the current continuity segment;
- do not use bad samples to update slow baselines;
- do not silently connect events across the gap.

## 11.2 When signal becomes usable again

Create a new continuity segment and apply modality-specific warm-up:

| Modality | Warm-up / recovery behavior |
|---|---|
| ECG | refill 600-ms baseline window; ~1 s [ES] transient flag |
| PPG | reset SOS state from first sample; ~2 s [ES] before mature pulse amplitude |
| EDA | new cvxEDA segment; first ~20 s [ES] `warming`; edge samples provisional |
| Temperature | refill 10-s smoothing window; reinitialize or cautiously resume long baseline |
| IMU | restore orientation state only if gap small; otherwise reinitialize roll/pitch from gravity |

This keeps decomposition state physically defensible.

---

# 12. Avoiding destructive preprocessing

## 12.1 ECG

**Risk:** high-pass cutoff ≥0.5 Hz can generate false ST displacement [11,12].

**Recommendation:** estimate and emit baseline explicitly; use median/spline methods. Use narrow 5–15 Hz only for QRS detection.

---

## 12.2 PPG

**Risk:** treating all <0.5 Hz energy as "baseline noise" destroys respiratory and slow hemodynamic modulation [4-6].

**Recommendation:** separate <0.05 Hz, 0.05–0.5 Hz, and 0.5–8 Hz branches.

---

## 12.3 EDA

**Risk:** high-pass filtering to "remove baseline" destroys tonic EDA and distorts the phasic response shape.

**Recommendation:** use model-based tonic/phasic decomposition.

---

## 12.4 Skin temperature

**Risk:** detrending can remove the very physiological dynamics temperature is intended to represent.

**Recommendation:** retain slow baseline and short dynamic components simultaneously.

---

## 12.5 IMU

**Risk:** fixed high-pass separation confuses slow real movement with gravity and performs poorly under rotation.

**Recommendation:** estimate orientation using gyro + accelerometer, then subtract the gravity vector; retain ENMO/HFEN only as supplementary references.

---

# 13. Per-modality implementation pseudocode

## 13.1 EDA

```text
function decompose_eda(trusted_segment):
    x_native = trusted_segment.samples

    x = anti_alias_and_resample(x_native, 8 Hz)     # [ES]

    append_to_contiguous_buffer(x)

    every 1 s:
        block = last 60 s                            # [ES]

        z = zscore(block)

        tonic, phasic, driver, drift, residual =
            cvxEDA(
                z,
                tau0 = 2.0 s,                       # [RS]
                tau1 = 0.7 s,                       # [RS]
                tonic_knot = 10 s,                  # [RS]
                alpha = 0.0008,                     # [RS]
                gamma = 0.01                        # [RS]
            )

        mark final 5 s provisional                   # [ES]
        commit older samples
```

---

## 13.2 PPG

```text
function decompose_ppg(x):
    trend = causal_butterworth_lowpass(
        x, fc=0.05 Hz, order=2                       # fc anchored in [6]; order [ES]
    )

    lowfreq = causal_butterworth_bandpass(
        x, 0.05 Hz, 0.5 Hz, order=2                  # [ES]
    )

    cardiac = causal_butterworth_bandpass(
        x, 0.5 Hz, 8 Hz, order=2                     # [RS] band/order from [7]
    )

    local_dc = rolling_median(x, 2 s)                # [ES]

    pulse_times = peak_detector(cardiac)
    pulse_amplitude = measure_on_preserved_branch(
        x, pulse_times, local_dc
    )

    residual = x - approximate_reconstruction(
        trend, lowfreq, cardiac
    )

    emit(...)
```

Offline:

```text
replace causal filters with forward-backward zero-phase filters
```

---

## 13.3 ECG

```text
function decompose_ecg(x_250Hz):
    b1 = centered_median(x, 51 samples)              # ~200 ms
    baseline = centered_median(b1, 151 samples)      # ~600 ms

    morphology = x - baseline

    detector = bandpass(morphology, 5 Hz, 15 Hz)
    r_peaks = adaptive_qrs_detector(detector)

    rr = diff(r_peaks)

    emit(
        baseline,
        morphology,
        r_peaks,
        rr
    )
```

The centered implementation is fixed-lag rather than zero-latency.

---

## 13.4 Skin temperature

```text
function decompose_temperature(x, fs):
    temp10 = moving_mean(x, 10 s)                    # [RS]

    alpha = 1 - exp(-dt / (30 min))                  # [ES]
    baseline = EWMA(temp10, alpha)

    dynamic = temp10 - baseline
    delta10 = temp10 - temp10.shift(10 s)

    emit(
        temp10,
        baseline,
        dynamic,
        delta10
    )
```

Offline multiday:

```text
fit 24-h cosinor to trusted temperature history
emit mesor, amplitude, acrophase, circadian curve
```

---

## 13.5 IMU

```text
function decompose_imu(accel, gyro, dt):
    q = madgwick_update(
        q_prev,
        gyro,
        accel,
        dt,
        beta = 0.1                                  # [ES]
    )

    g_body = rotate_world_gravity_into_sensor(q)

    linear_accel = accel - g_body

    enmo = max(norm(accel) - 1g, 0)

    emit(
        q,
        g_body,
        linear_accel,
        enmo
    )
```

---

# 14. Real-time computational requirements

Approximate qualitative cost:

| Modality | Algorithm | Complexity | Buffer |
|---|---|---|---|
| ECG | two median filters + QRS detector | low | <1 s |
| PPG | several biquad filters + peak detector | very low | ~2 s + filter state |
| EDA | convex QP on 60-s 8-Hz block | moderate | 60 s |
| Temperature | moving mean + EWMA | trivial | 10 s |
| IMU | quaternion orientation update | low | state only |

The EDA solver is the only component likely to merit a dedicated worker thread/process.

---

# 15. Final synthesis table

| Modality | Physiological components | Baseline / nuisance components | Selected decomposition | Key parameters | Window/state | Outputs | Key refs |
|---|---|---|---|---|---|---|---|
| EDA/GSR | tonic SCL, phasic SCR, sparse driver | drift, residual | cvxEDA | \(\tau_0=2\) s default, \(\tau_1=0.7\) s, 10-s tonic knots, \(\alpha=8e-4\), \(\gamma=1e-2\) | 60-s live block [ES], 5-s fixed-lag edge [ES] | tonic, phasic, driver, drift, residual | [1,2] |
| PPG | cardiac pulsation, low-frequency/respiratory modulation, slow optical trend | high-frequency residual | explicit filter bank + beat segmentation | <0.05, 0.05–0.5, 0.5–8 Hz | causal filter state; ~2-s local DC buffer | slow, LF, cardiac, pulse series, AC/DC | [4-8] |
| ECG | morphology, beats, RR series | baseline wander | median baseline estimator; QRS detector branch | ~200 ms + 600 ms medians; detector 5–15 Hz | ~300-ms fixed lag | baseline, morphology, R peaks, RR | [9-14] |
| Skin temperature | slow thermal state, short dynamic change, circadian component | small fast residual | 10-s smoother + slow EWMA; offline cosinor | 10-s [RS], \(\tau=30\) min [ES], 24-h cosinor | 10-s live buffer + long baseline state | slow baseline, dynamic, slope, circadian | [15-18] |
| IMU | linear acceleration, angular velocity, orientation | gravity, gyro bias | Madgwick orientation + gravity subtraction | \(\beta=0.1\) [ES] | sample-causal quaternion state | gravity, linear accel, orientation, bias | [19,20] |

---

# Recommended Implementation

This is the single consolidated decomposition subsystem that should be built.

## A. Common infrastructure

Implement a `DecompositionEngine` with one stateful processor per modality.

Every processor must:

1. accept only trusted contiguous data;
2. never bridge a declared data gap silently;
3. preserve source timestamps;
4. emit explicit components, not merely one "cleaned" waveform;
5. expose parameter/version metadata;
6. expose warm-up/provisional state;
7. retain enough information to reconstruct or compare with the trusted source.

---

## B. ECG processor

### Input
250-Hz trusted ECG.

### Sequence

```text
trusted ECG
↓
200-ms median baseline stage
↓
600-ms median baseline stage
├── baseline output
└── subtract from original
    ↓
    morphology-preserving ECG
    ├── downstream morphology output
    └── 5–15 Hz QRS detector branch
        ↓
        R peaks → RR intervals
```

### Parameters

- 51-sample first median [ES implementation of ~200 ms];
- 151-sample second median [ES implementation of ~600 ms];
- detector band 5–15 Hz [RS];
- fixed-lag centered filtering [ES].

### State

- two median windows;
- QRS adaptive threshold state;
- previous accepted R peak.

### Reset

- clear windows after discontinuity;
- no RR interval may cross discontinuity;
- mark first ~1 s after reacquisition as warming [ES].

### Offline alternative

Cubic-spline / time-varying baseline estimation [9,10], compared against live output.

---

## C. PPG processor

### Input
Trusted PPG per optical wavelength.

### Sequence

```text
trusted PPG
├── LP <0.05 Hz → very-slow trend
├── BP 0.05–0.5 Hz → low-frequency physiological modulation
├── BP 0.5–8 Hz → cardiac pulsatile component
│                    ↓
│                 pulse detector
│                    ↓
│               pulse timestamps
└── rolling local DC estimate
                     ↓
              AC/DC per pulse
```

### Parameters

- cardiac 0.5–8 Hz second-order Butterworth [RS];
- very-slow 0.05 Hz boundary [RS physiological anchor, filter design ES];
- low-frequency 0.05–0.5 Hz [ES];
- local DC 2-s rolling median [ES].

### State

- SOS filter states;
- 2-s local DC buffer;
- last pulse time;
- current pulse interval estimate.

### Reset

- initialize IIR states from first valid level;
- do not create pulse intervals across gaps;
- mark first 2 s after reacquisition warming [ES].

### Offline alternative

Forward-backward zero-phase filter bank.

---

## D. EDA processor

### Input
Trusted EDA at native rate.

### Sequence

```text
trusted EDA
↓
anti-alias
↓
8-Hz decomposition copy
↓
60-s rolling cvxEDA
├── tonic
├── phasic
├── sparse driver
├── drift
└── residual
```

### Parameters

- \(\tau_0=2.0\) s default [RS], optionally optimize in 2–4 s range offline;
- \(\tau_1=0.7\) s [RS];
- tonic knot 10 s [RS];
- \(\alpha=0.0008\) [RS];
- \(\gamma=0.01\) [RS];
- 8-Hz internal rate [ES];
- 60-s rolling block [ES];
- 5-s right-edge uncommitted guard [ES].

### State

- 60-s sample buffer;
- previous solver solution for warm start if implementation supports it;
- continuity id;
- maturity flag.

### Reset

- terminate block at bad-quality boundary;
- never spline across the missing region;
- new segment on reacquisition;
- first ~20 s warming [ES].

### Offline alternative

Run cvxEDA on long contiguous segments; benchmark against ospEDA when available [3].

---

## E. Temperature processor

### Input
Trusted skin temperature.

### Sequence

```text
trusted temperature
↓
10-s moving mean
├── short smoothed temperature
├── 10-s differential
└── 30-min EWMA baseline [ES]
      ├── slow baseline
      └── subtract
          ↓
          short dynamic component
```

Offline:

```text
multiday trusted temperature
↓
24-h cosinor
├── mesor
├── amplitude
├── acrophase
└── circadian curve
```

### Parameters

- 10-s smoothing/differential [RS];
- 30-min EWMA time constant [ES];
- 24-h cosinor period [RS].

### State

- 10-s buffer;
- EWMA baseline state;
- baseline age/maturity.

### Reset

- freeze baseline across gap;
- reinitialize from first valid 10-s average after long discontinuity;
- preserve old baseline as historical metadata.

---

## F. IMU processor

### Input
Trusted 3-axis accelerometer + gyroscope.

### Sequence

```text
accel + gyro
↓
Madgwick quaternion orientation
├── orientation
└── predicted gravity vector
      ↓
raw acceleration - gravity
      ↓
linear acceleration
```

Also compute ENMO as a published reference metric.

### Parameters

- actual IMU sampling rate;
- \(\beta=0.1\) [ES];
- gravity = 1 g / 9.80665 m/s².

### State

- quaternion;
- gyro bias;
- timestamp.

### Reset

- resume integration only over small gaps;
- otherwise reinitialize roll/pitch from gravity when sufficiently static.

---

## G. Unified component contract

The final decomposition layer should expose a schema conceptually like:

```text
PhysiologicalComponents {
    ecg?: {
        baseline
        morphology
        r_peaks
        rr_intervals
    }

    ppg?: {
        slow_trend
        low_frequency
        cardiac
        local_dc
        pulse_times
        pulse_intervals
        pulse_amplitudes
        ac_dc_ratio
        ratio_of_ratios
        residual
    }

    eda?: {
        tonic
        phasic
        phasic_driver
        drift
        residual
    }

    temperature?: {
        smoothed
        slow_baseline
        short_dynamic
        delta_10s
        circadian_component?
    }

    imu?: {
        orientation
        gravity
        linear_acceleration
        gyro_bias
        corrected_gyro
        enmo
    }
}
```

The downstream layer should consume these named components rather than re-filtering the raw signals independently.

---

# 16. What should **not** be implemented in this layer

Do not add:

- stress inference;
- craving/withdrawal/relapse inference;
- intoxication inference;
- motion-vs-autonomic causal classification;
- activity recognition;
- contextual reasoning;
- subject-state labels;
- AUD/SUD-specific thresholds;
- signal-quality acceptance/rejection logic already owned upstream.

The decomposition layer has one responsibility:

> **Represent each trustworthy measurement as the physiologically distinct components that coexist within it, preserving meaningful slow and fast dynamics rather than destroying them with generic filtering.**

---

# References

[1] A. Greco, G. Valenza, A. Lanata, E. P. Scilingo, L. Citi.  
“cvxEDA: A Convex Optimization Approach to Electrodermal Activity Processing.”  
*IEEE Transactions on Biomedical Engineering*, 63(4), 797–804, 2016.  
DOI: 10.1109/TBME.2015.2474131  
PubMed: https://pubmed.ncbi.nlm.nih.gov/26336110/  
Reference implementation: https://github.com/lciti/cvxEDA

[2] M. Benedek, C. Kaernbach.  
“A Continuous Measure of Phasic Electrodermal Activity.”  
*Journal of Neuroscience Methods*, 190(1), 80–91, 2010.  
DOI: 10.1016/j.jneumeth.2010.04.028  
PubMed: https://pubmed.ncbi.nlm.nih.gov/20451556/  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC2892750/

[3] Y. Lee, Y. Kong, K. H. Chon.  
“ospEDA: Orthogonal Subspace Projection for Electrodermal Activity Decomposition.”  
*IEEE Transactions on Biomedical Engineering*, 2026, advance publication.  
DOI: 10.1109/TBME.2026.3703619  
PubMed: https://pubmed.ncbi.nlm.nih.gov/42295969/

[4] J. Allen.  
“Photoplethysmography and its Application in Clinical Physiological Measurement.”  
*Physiological Measurement*, 28(3), R1–R39, 2007.  
DOI: 10.1088/0967-3334/28/3/R01  
PubMed: https://pubmed.ncbi.nlm.nih.gov/17322588/

[5] J. Park, H. S. Seok, S.-S. Kim, H. Shin.  
“Photoplethysmogram Analysis and Applications: An Integrative Review.”  
*Frontiers in Physiology*, 12:808451, 2022.  
DOI: 10.3389/fphys.2021.808451  
https://www.frontiersin.org/articles/10.3389/fphys.2021.808451/full

[6] P. H. Charlton, D. A. Birrenkott, T. Bonnici, et al.  
“Breathing Rate Estimation From the Electrocardiogram and Photoplethysmogram: A Review.”  
*IEEE Reviews in Biomedical Engineering*, 11, 2–20, 2018.  
DOI: 10.1109/RBME.2017.2763681  
PubMed: https://pubmed.ncbi.nlm.nih.gov/29990026/  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC7612521/

[7] M. Elgendi, I. Norton, M. Brearley, D. Abbott, D. Schuurmans.  
“Systolic Peak Detection in Acceleration Photoplethysmograms Measured from Emergency Responders in Tropical Conditions.”  
*PLOS ONE*, 8(10):e76585, 2013.  
DOI: 10.1371/journal.pone.0076585  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC3805543/

[8] M. Nitzan, A. Romem, R. Koppel.  
“Pulse Oximetry: Fundamentals and Technology Update.”  
*Medical Devices: Evidence and Research*, 7, 231–239, 2014.  
DOI: 10.2147/MDER.S47319  
PubMed: https://pubmed.ncbi.nlm.nih.gov/25031547/  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC4099100/

[9] L. Sörnmo.  
“Time-Varying Digital Filtering of ECG Baseline Wander.”  
*Medical & Biological Engineering & Computing*, 31(5), 503–508, 1993.  
DOI: 10.1007/BF02441986  
PubMed: https://pubmed.ncbi.nlm.nih.gov/8295440/

[10] C. R. Meyer, H. N. Keiser.  
“Electrocardiogram Baseline Noise Estimation and Removal Using Cubic Splines and State-Space Computation Techniques.”  
*Computers and Biomedical Research*, 10(5), 459–470, 1977.  
DOI: 10.1016/0010-4809(77)90021-0  
PubMed: https://pubmed.ncbi.nlm.nih.gov/923219/

[11] P. Kligfield, L. S. Gettes, J. J. Bailey, et al.  
“Recommendations for the Standardization and Interpretation of the Electrocardiogram.”  
*Circulation*, 2007.  
DOI: 10.1161/CIRCULATIONAHA.106.180200

[12] K. Barmpagiannos, Z. Malik, S. Whyte, S. Stavrakis.  
“Quantifying High-Pass Filter-Induced ST-Segment Distortion.”  
*Journal of Electrocardiology*, 99:154442, 2026.  
DOI: 10.1016/j.jelectrocard.2026.154442  
PubMed: https://pubmed.ncbi.nlm.nih.gov/42660012/

[13] J. Pan, W. J. Tompkins.  
“A Real-Time QRS Detection Algorithm.”  
*IEEE Transactions on Biomedical Engineering*, BME-32(3), 230–236, 1985.  
DOI: 10.1109/TBME.1985.325532  
PubMed: https://pubmed.ncbi.nlm.nih.gov/3997178/

[14] Task Force of the European Society of Cardiology and the North American Society of Pacing and Electrophysiology.  
“Heart Rate Variability: Standards of Measurement, Physiological Interpretation and Clinical Use.”  
*Circulation*, 93(5), 1043–1065, 1996.  
PubMed: https://pubmed.ncbi.nlm.nih.gov/8598068/

[15] M. Kistler, M. Mariauzouls, K. von Berlepsch.  
“Fingertip Temperature as an Indicator for Sympathetic Responses.”  
*International Journal of Psychophysiology*, 29(1), 35–41, 1998.  
DOI: 10.1016/S0167-8760(97)00087-1  
PubMed: https://pubmed.ncbi.nlm.nih.gov/9641246/

[16] S. Y. Sim, M. J. Koh, K. M. Joo, et al.  
“Estimation of Thermal Sensation Based on Wrist Skin Temperatures.”  
*Sensors*, 16(4):420, 2016.  
DOI: 10.3390/s16040420  
PubMed: https://pubmed.ncbi.nlm.nih.gov/27023538/  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC4850935/

[17] M. Dittmar, T. Stark, S. Wedell.  
“Circadian Rhythm of Distal Skin Temperature in Healthy Older and Young Women and Its Relationship with Sleep-Wake Rhythm and Environmental Factors under Natural Living Conditions.”  
*Geriatrics*, 9(4):102, 2024.  
DOI: 10.3390/geriatrics9040102  
PubMed: https://pubmed.ncbi.nlm.nih.gov/39195132/

[18] H. Batinga, A. Martinez-Nicolas, M. Zornoza-Moreno, M. Sánchez-Solis, E. Larqué, M. T. Mondéjar, M. Moreno-Casbas, F. J. García, M. Campos, M. A. Rol, J. A. Madrid.  
“Ontogeny and Aging of the Distal Skin Temperature Rhythm in Humans.”  
*Age*, 37(2):29, 2015.  
DOI: 10.1007/s11357-015-9768-y  
PubMed: https://pubmed.ncbi.nlm.nih.gov/25813804/  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC4375132/

[19] V. T. van Hees, L. Gorzelniak, E. C. Dean León, et al.  
“Separating Movement and Gravity Components in an Acceleration Signal and Implications for the Assessment of Human Daily Physical Activity.”  
*PLOS ONE*, 8(4):e61691, 2013.  
DOI: 10.1371/journal.pone.0061691  
PubMed: https://pubmed.ncbi.nlm.nih.gov/23626718/  
PLOS: https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0061691

[20] S. O. H. Madgwick, A. J. L. Harrison, R. Vaidyanathan.  
“Estimation of IMU and MARG Orientation Using a Gradient Descent Algorithm.”  
*IEEE International Conference on Rehabilitation Robotics*, 2011.  
DOI: 10.1109/ICORR.2011.5975346  
PubMed: https://pubmed.ncbi.nlm.nih.gov/22275550/

[21] A. A. K. Timimi, M. A. Mohd Ali, K. Chellappan.  
“A Novel AMARS Technique for Baseline Wander Removal Applied to Photoplethysmogram.”  
*IEEE Transactions on Biomedical Circuits and Systems*, 11(3), 627–639, 2017.  
DOI: 10.1109/TBCAS.2017.2649940  
PubMed: https://pubmed.ncbi.nlm.nih.gov/28489546/

[22] J. Li, J. Jin, X. Chen, W. Sun, P. Guo.  
“Comparison of Respiratory-Induced Variations in Photoplethysmographic Signals.”  
*Physiological Measurement*, 31(3), 415–425, 2010.  
DOI: 10.1088/0967-3334/31/3/009  
PubMed: https://pubmed.ncbi.nlm.nih.gov/20147775/

[23] A. Jubran.  
“Pulse Oximetry.”  
*Critical Care*, 19:272, 2015.  
DOI: 10.1186/s13054-015-0984-8  
PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC4504215/

[24] P. H. Charlton, T. Bonnici, L. Tarassenko, et al.  
“Extraction of Respiratory Signals from the Electrocardiogram and Photoplethysmogram: Technical and Physiological Determinants.”  
*Physiological Measurement*, 38(5), 669–690, 2017.  
DOI: 10.1088/1361-6579/aa670e  
PubMed: https://pubmed.ncbi.nlm.nih.gov/28296645/

[25] R. A. Frankel, E. W. Pottala, R. W. Bowser, J. J. Bailey.  
“A Filter to Suppress ECG Baseline Wander and Preserve ST-Segment Accuracy in a Real-Time Environment.”  
*Journal of Electrocardiology*, 24(4), 315–323, 1991.  
DOI: 10.1016/0022-0736(91)90014-D  
PubMed: https://pubmed.ncbi.nlm.nih.gov/1836004/

---

## Implementation status summary

### Ready to implement now

- ECG two-stage median baseline + separate QRS detector branch.
- PPG explicit slow/LF/cardiac filter bank + per-beat local DC.
- Temperature 10-s smoother + preserved slow EWMA baseline.
- IMU quaternion orientation + gravity subtraction.
- EDA cvxEDA offline and blockwise fixed-lag service.

### Must be calibrated on hardware data before freezing

- EDA decomposition resampling rate, live block length, and edge guard.
- PPG low-frequency filter order and local-DC window.
- temperature EWMA time constant.
- IMU Madgwick gain.
- gap-length thresholds that decide state continuation versus reset.

### Must never be collapsed into one "cleaning" stage

- EDA tonic vs phasic.
- PPG slow vs respiratory/low-frequency vs cardiac.
- ECG baseline vs morphology.
- temperature slow state vs short deviation.
- IMU gravity vs linear acceleration.

That separation is the central architectural requirement of this layer.
