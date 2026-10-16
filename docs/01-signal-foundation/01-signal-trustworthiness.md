# Physiological Signal Trustworthiness Solution
## Implementation design for continuous “is this acquired signal good enough to use?” monitoring

**Document purpose:** Define an implementation-ready Signal Trustworthiness Layer that continuously determines whether each acquired physiological signal is sufficiently reliable for downstream processing.

**Scope:** ECG, red/IR photoplethysmography (PPG), electrodermal activity (EDA/GSR), skin temperature, and IMU data used strictly as a quality-verification sensor. Derived quantities such as heart rate (HR), heart-rate variability (HRV), pulse rate, and SpO₂ are discussed only insofar as their use must be gated by raw-signal quality.

**Current hardware assumed:** single-lead ECG through AD8232; red/IR PPG through MAX30101; EDA/GSR through the current GSR front end; skin temperature through TMP117; 6-axis motion through MPU-6050; synchronized acquisition on the embedded system. The current project rates are approximately ECG 250 Hz, PPG 100 Hz, EDA 128 Hz, IMU 100 Hz, and skin temperature 2 Hz. The implementation must nevertheless read the configured sampling rate at runtime rather than hard-code these values.

---

# 1. Design objective

The Signal Trustworthiness Layer answers one question:

> **Before a physiological measurement is used, is the underlying acquired signal trustworthy enough for that use?**

This is deliberately different from asking whether the physiology is normal.

A high-quality ECG can contain an abnormal rhythm. A high-quality PPG can contain an unusual pulse rate. A genuine EDA response can change quickly. Therefore, a quality algorithm must avoid turning “unexpected physiology” into “artifact” unless there is supporting acquisition or waveform evidence.

The system should make four distinct judgments:

1. **Did acquisition work?**  
   Were samples actually delivered at the expected cadence, without transport loss, sensor errors, FIFO overflow, ADC saturation, or stale data?

2. **Is the sensor physically coupled to the subject?**  
   Are electrodes/optics/contact surfaces actually connected well enough to measure the intended signal?

3. **Does the acquired waveform/data possess the structure expected of that measurement?**  
   Examples: reproducible QRS complexes for ECG, pulsatile morphology for PPG, plausible conductance continuity for EDA, slow thermal behavior for skin temperature.

4. **Is there enough evidence to authorize downstream use?**  
   The result should be a state, confidence/score, and explicit rejection/downgrade reasons.

This document recommends a **layered, veto-first design**:

```text
raw sensor sample/packet
        ↓
acquisition integrity
        ↓
sensor/contact integrity
        ↓
basic waveform sanity
        ↓
modality-specific SQIs
        ↓
physiological plausibility as supporting evidence
        ↓
quality state + score + reasons
        ↓
only authorized data/features passed downstream
```

Cheap, definitive failures are evaluated first. More expensive SQIs are evaluated only on data that survives the earlier stages.

---

# 2. Quality-state semantics

Use the following states across modalities.

| State | Meaning | Downstream policy |
|---|---|---|
| `GOOD` | No hard acquisition/contact failure and the modality-specific waveform evidence is strong. | Signal may be used for the downstream feature(s) whose quality requirements are satisfied. |
| `DEGRADED` | Signal is present and may support limited/robust uses, but one or more quality indicators are weak, conflicting, or marginal. | Do not silently treat as fully valid. Pass only to downstream consumers that explicitly accept degraded input. |
| `BAD` | Acquisition/contact failure or waveform corruption is strong enough that physiological use is unsafe/unreliable. | Reject the affected interval for physiological processing. |
| `NOT_READY` | Sensor has just started/reconnected or insufficient history exists to compute the required SQIs. | Hold data until minimum history/settling requirements are met. |

`BAD` should mean **untrustworthy**, not “physiologically abnormal.”

## 2.1 Quality score

Emit a modality-specific `quality_score` in `[0,1]`, but do **not** make this the only acceptance criterion.

The initial implementation should use:

- **hard vetoes** for unequivocal acquisition/contact failures;
- **rule-based state classification** for the first production version;
- a score as a monotonic summary of surviving SQIs;
- later calibration of the score against manually labeled data from the actual hardware.

An uncalibrated score must not be interpreted as a probability.

Suggested initial state mapping after all hard vetoes:

```text
score >= 0.80        -> GOOD
0.50 <= score < 0.80 -> DEGRADED
score < 0.50         -> BAD
```

These numeric state cut-points are **engineering starting thresholds**, not literature-derived universal clinical boundaries. The actual state logic below remains authoritative until a labeled calibration set is available.

---

# 3. Threshold taxonomy

Every threshold in code should carry metadata describing where it came from.

```text
RESEARCH_SUPPORTED
    Explicitly reported/validated in a cited paper or device specification.

ENGINEERING_START
    Reasonable initial operating value chosen for this implementation.
    Must be calibrated/validated on the actual device, placement, population,
    ADC configuration, and intended downstream use.

CALIBRATED
    Re-estimated from labeled recordings collected with this hardware.
```

Do not bury these distinctions in comments. Store them in the quality configuration.

Example:

```yaml
ecg:
  template_corr_min:
    value: 0.66
    provenance: RESEARCH_SUPPORTED
    reference: 1
  clipping_fraction_bad:
    value: 0.01
    provenance: ENGINEERING_START
```

---

# 4. Common acquisition-integrity layer

These checks apply to every data stream before modality-specific processing.

## 4.1 Timestamp and sequence continuity

For a stream with nominal sampling frequency \(f_s\),

\[
\Delta t_{expected} = \frac{1}{f_s}.
\]

For each arriving sample/packet, retain:

- monotonic timestamp;
- packet/sample sequence number;
- expected samples per packet;
- actual samples per packet;
- hardware error/status flags;
- configured sampling rate.

Compute over every quality window:

\[
missing\_fraction =
1-\frac{N_{received}}{N_{expected}}
\]

and

\[
jitter_i = \Delta t_i-\Delta t_{expected}.
\]

Record:

- fraction missing;
- maximum contiguous gap;
- median absolute timing error;
- 95th/99th percentile timing error;
- duplicated timestamps/samples;
- out-of-order packets.

### Recommended starting policy

These are **engineering starting values**, not biomedical constants:

- any transport/FIFO overflow flag: add a reason code immediately;
- `missing_fraction > 0.01`: at least `DEGRADED`;
- `missing_fraction > 0.05`: normally `BAD` for ECG/PPG windows;
- any contiguous gap large enough to remove or materially distort a beat/pulse: `BAD` for beat-timing uses;
- stale repeated samples: `BAD` if they persist beyond the modality’s normal dynamics.

For EDA and temperature, short gaps may sometimes be tolerated for slow-trend use, but missing raw samples must still be represented explicitly rather than silently interpolated and forgotten.

## 4.2 Numeric validity

Reject/flag:

- NaN/Inf;
- impossible integer codes;
- sensor reset values;
- unchanged repeated register value caused by failed reads;
- I²C/SPI error;
- checksum/CRC failure if implemented.

## 4.3 Rail/clipping detection

For an ADC with lower/upper usable rails \(x_{min},x_{max}\), define a rail margin \(m\):

\[
clipped_i =
(x_i \le x_{min}+m)\;\lor\;(x_i \ge x_{max}-m)
\]

and

\[
clip\_fraction=\frac{1}{N}\sum_i \mathbf{1}(clipped_i).
\]

Do not choose \(m\) solely from “percent of full scale” if the analog front end does not use the entire ADC range. Determine it from the actual ADC/front-end configuration.

Recommended initial policy:

- any continuous rail run longer than a physiologically meaningful waveform feature is a strong failure;
- `clip_fraction >= 1%` within a 10-s ECG/PPG window is an **engineering starting BAD threshold** pending calibration;
- isolated one-sample rail hits should still be counted and reported.

## 4.4 Flatline / stale-data detection

A flatline can indicate disconnection, dead electronics, a stuck bus/register, or severe saturation.

Useful statistics over a window \(x_1,\ldots,x_N\):

\[
\sigma^2 = \frac{1}{N-1}\sum_i(x_i-\bar{x})^2
\]

and

\[
range = \max(x)-\min(x).
\]

Absolute thresholds must be learned in physical units or ADC counts from the actual hardware. A generic variance threshold is not portable across gain settings.

Use two checks:

1. **exact/stale run:** same digital code repeated for implausibly long;
2. **near-flat window:** variance/range below a calibrated noise floor.

---

# 5. ECG trustworthiness

## 5.1 What “good enough” means for ECG

### `GOOD`

A good ECG window has:

- intact acquisition with no meaningful data gaps;
- no lead-off indication;
- no meaningful clipping/saturation;
- sufficient amplitude/variance to represent a cardiac waveform;
- repeatable QRS morphology;
- high agreement between independent QRS detectors;
- manageable baseline wander/high-frequency contamination;
- beat timing that is internally coherent.

Orphanidou et al. developed a 10-s ECG SQI specifically to determine whether a reliable heart rate could be derived from ambulatory ECG. Their method combines physiologic feasibility rules with adaptive beat-template correlation and achieved 94% sensitivity and 97% specificity in their validation data [1].

### `DEGRADED`

A window is degraded when the ECG is recognizably present but one of the following occurs:

- weak detector agreement;
- morphology correlation is marginal;
- elevated baseline wander;
- elevated high-frequency/EMG contamination;
- intermittent motion artifact;
- one published “physiological plausibility” rule fails while morphology and acquisition quality remain strong.

The last case is important: a genuine arrhythmia or unusual rate can violate an HR/IBI rule without the ECG being an acquisition failure.

### `BAD`

A window is bad when:

- AD8232 lead-off is asserted;
- signal is clipped/railed for a material fraction of the window;
- transport loss removes waveform continuity;
- ECG is flat/stale;
- QRS complexes cannot be detected reliably;
- independent detectors substantially disagree;
- beat morphology is inconsistent enough that the signal no longer supports dependable beat detection;
- corruption dominates the waveform.

---

## 5.2 ECG failure modes

| Failure mode | How it appears | Detection |
|---|---|---|
| Electrode detachment | lead-off flag, abrupt rail/settling transient, loss of QRS | AD8232 `LOD+`/`LOD-` + waveform checks |
| Poor electrode contact / impedance changes | baseline wander, intermittent large transients, motion-correlated distortion | morphology, baseline SQI, IMU coincidence |
| Saturation | samples near ADC rail; recovery tail | rail-hit detector |
| Flatline/stuck acquisition | near-zero variance or repeated ADC code | variance/range/stale-run |
| Packet/sample loss | timestamp/sequence gaps | acquisition layer |
| Baseline wander | excessive low-frequency power | `basSQI` |
| High-frequency/EMG/power-line interference | excess power outside expected ECG/QRS band | `pSQI`, spectrum |
| Motion artifact | morphology deformation, false/missed QRS, baseline shifts | template correlation, bSQI, IMU evidence |
| QRS detector failure | independent algorithms disagree | bSQI |
| Abnormally weak signal | very low amplitude relative to device noise floor | calibrated amplitude/SNR floor |

The AD8232 supports AC/DC lead-off detection. In AC lead-off mode it can determine that one of the two measurement electrodes has lost connection; in DC mode it can identify which monitored input is disconnected [14]. Lead-off is a direct hardware observation and should be treated as stronger evidence than an inferred SQI.

---

## 5.3 ECG SQIs to implement

### 5.3.1 AD8232 lead-off status — primary hard gate

**Input:** `LOD+`, `LOD-` sampled with the ECG acquisition state.

**Decision:**  
If lead-off is asserted, mark ECG `BAD` with `ELECTRODE_OFF`.

The AD8232 documentation explicitly states that lead-off indicates loss of the required conduction/contact path [14].

**Real-time suitability:** excellent; \(O(1)\) per sample/interrupt.

**Failure case:** lead-off detects disconnection, not every form of poor contact. A partially attached electrode can still produce low-quality ECG while lead-off remains inactive. Therefore, it cannot replace waveform SQIs.

---

### 5.3.2 Clipping fraction

\[
q_{clip}=1-clip\_fraction.
\]

**Input/window:** raw ADC samples, continuously; summarize over 1 s and the main 10-s ECG window.

**Strength:** cheap and unequivocal for saturation.

**Failure case:** a signal can be badly corrupted without reaching the rails.

---

### 5.3.3 Orphanidou feasibility + template-matching SQI

Orphanidou et al. use 10-s ECG segments [1]. After R-peak detection, the original algorithm applies:

1. extrapolated HR between **40 and 180 bpm**;
2. maximum R-R gap \(\le 3\) s;
3. ratio \(RR_{max}/RR_{min}<2.2\);
4. average beat-to-template correlation at least **0.66** for ECG [1].

The template is constructed by extracting beat windows around detected peaks, averaging them, and calculating the Pearson correlation of each beat with the template. Let \(b_i\) be a normalized beat and \(t\) the template:

\[
r_i =
\frac{\sum_k(b_i[k]-\bar b_i)(t[k]-\bar t)}
{\sqrt{\sum_k(b_i[k]-\bar b_i)^2}\sqrt{\sum_k(t[k]-\bar t)^2}}
\]

and

\[
SQI_{template}=\frac{1}{M}\sum_{i=1}^{M}r_i.
\]

**Research-supported threshold:** \(SQI_{template}\ge0.66\) for ECG [1].

**Why it works:** cardiac cycles should retain repeated morphology; noise, motion, false QRS detections, and missed beats reduce repeatability.

**Strengths:** validated for wearable ECG; computationally modest; interpretable.

**Known failure cases:**

- ectopy/arrhythmia can produce genuine morphology variation;
- a consistently corrupted periodic artifact can appear repeatable;
- quality depends on the R-peak detector;
- the 40–180 bpm rule can reject real bradycardia/tachycardia.

**Implementation decision:** keep the 0.66 morphology threshold as a primary SQI. Treat the HR/IBI feasibility rules as strong supporting evidence for reliable beat-rate extraction, but do not let an isolated plausibility failure override otherwise strong acquisition/morphology evidence. Such a case becomes `DEGRADED` with `PHYSIOLOGY_RULE_OUTSIDE_VALIDATION_RANGE`, not automatically `BAD`.

---

### 5.3.4 Beat detector agreement (bSQI)

Use two QRS detectors with different noise responses, e.g. a Pan-Tompkins/Hamilton-family detector and a second independent detector.

For detected beat sets \(A\) and \(B\), count a beat as matched when the detections are within tolerance \(\tau\) (commonly 100–150 ms in implementations). A Jaccard-like bSQI is:

\[
bSQI =
\frac{N_{matched}}
{N_A+N_B-N_{matched}}.
\]

Detector agreement is a major ECG quality signal because noise affects algorithms differently [2]. A threshold around **0.8** is commonly used in later ECG studies for high-quality windows, while some work uses >0.9 as a stricter “optimal” level [21].

**Recommended initial interpretation:**

- `bSQI >= 0.90`: strong detector agreement;
- `0.80 <= bSQI < 0.90`: acceptable;
- `0.60 <= bSQI < 0.80`: degraded;
- `bSQI < 0.60`: strong bad-quality evidence.

The first three boundaries are literature-informed but not universal. The exact detector pair and tolerance must be fixed during calibration.

**Complexity:** \(O(M)\) beat matching per window after detection.

---

### 5.3.5 Spectral QRS-power SQI (pSQI)

A common ECG spectral SQI is:

\[
pSQI =
\frac{\int_{5}^{15} P(f)\,df}
{\int_{5}^{40} P(f)\,df}.
\]

It measures how much 5–40 Hz ECG-band power is concentrated in the typical QRS-dominant 5–15 Hz region [2].

A commonly reported “optimal” adult range is approximately **0.5–0.8**, with values outside that range indicating suspicious spectral composition in the specific methods/datasets from which those cut-points were derived [21].

**Use in this system:** secondary evidence only. Do not hard-reject solely on pSQI because heart rate, lead configuration, filtering, morphology, and pathology change the spectrum.

---

### 5.3.6 Baseline-wander SQI (basSQI)

Use:

\[
basSQI =
\frac{\int_{1}^{40} P(f)\,df}
{\int_{0}^{40} P(f)\,df}
=
1-\frac{\int_{0}^{1}P(f)\,df}
{\int_{0}^{40}P(f)\,df}.
\]

Low values indicate excessive low-frequency baseline energy. Clifford et al. include baseline and spectral information in automated ECG quality assessment [2].

Later implementations have used:

- `>= 0.95`: optimal;
- `0.90–0.95`: suspicious;
- `< 0.90`: poor [21].

Treat those as **research-supported in that implementation, not universal AD8232 thresholds**. They are useful initial categories pending calibration.

---

### 5.3.7 Skewness/kurtosis — diagnostic secondary SQIs

For samples \(x_i\),

\[
skewness =
\frac{1}{N}\sum_i\left(\frac{x_i-\mu}{\sigma}\right)^3
\]

\[
kurtosis =
\frac{1}{N}\sum_i\left(\frac{x_i-\mu}{\sigma}\right)^4.
\]

These detect changes in distribution shape caused by noise and are included in established ECG SQI work [2]. They should be logged and can later enter a trained classifier, but there is not a sufficiently portable universal threshold for this hardware.

---

## 5.4 Recommended ECG decision mechanism

Use **hard hardware gates + beat/morphology validation + detector agreement + spectral diagnostics**.

### Hard veto

Set `BAD` immediately if any of the following applies:

- lead-off asserted;
- acquisition window missing or invalid;
- major packet/FIFO loss;
- persistent flatline/stale data;
- substantial ADC clipping/rail behavior;
- too little data to detect any plausible QRS structure after `NOT_READY` history is satisfied.

### Main waveform decision

On a rolling **10-s** window, updated every **1 s**:

1. detect R peaks with two independent detectors;
2. compute bSQI;
3. compute Orphanidou template correlation;
4. compute published feasibility rules;
5. compute pSQI and basSQI;
6. compute motion evidence from aligned IMU data.

### Initial rule set

```text
if hard_veto:
    BAD
elif template_corr >= 0.66 and bSQI >= 0.80
     and no severe spectral failure:
    GOOD
elif recognizable_QRS
     and (template_corr marginal OR 0.60 <= bSQI < 0.80
          OR spectral quality suspicious
          OR only physiological-plausibility rule failed):
    DEGRADED
else:
    BAD
```

`template_corr < 0.66` is research-supported bad-quality evidence [1], but in the combined system it should be interpreted together with bSQI because genuine morphology changes can reduce template correlation.

### Hysteresis/recovery

Engineering starting behavior:

- enter `BAD` immediately on lead-off, major clipping, or acquisition loss;
- enter `BAD` from waveform evidence after 2 consecutive bad 10-s evaluations;
- recover `BAD -> DEGRADED` after 2 consecutive non-bad evaluations;
- recover `DEGRADED -> GOOD` after 3 consecutive good evaluations.

This avoids rapid state flapping while remaining responsive to hardware disconnection.

---

## 5.5 ECG pseudocode

```python
def assess_ecg(window_10s, hw_status, imu_window, state):
    reasons = []

    # hard acquisition/contact gates
    if hw_status.lead_off:
        return BAD(0.0, ["ELECTRODE_OFF"])

    acq = acquisition_quality(window_10s)
    if acq.major_gap:
        return BAD(0.0, ["DATA_GAP"])
    if acq.flatline:
        return BAD(0.0, ["FLATLINE"])
    if acq.severe_clipping:
        return BAD(0.0, ["CLIPPING"])

    ecg = quality_filter_ecg(window_10s)  # only filtering needed for SQIs

    r1 = qrs_detector_A(ecg)
    r2 = qrs_detector_B(ecg)

    bsqi = beat_agreement(r1, r2, tolerance_ms=150)
    template_corr = orphanidou_template_corr(ecg, r1)
    rules = orphanidou_feasibility_rules(r1, window_seconds=10)

    psqi = qrs_band_power_ratio(ecg)
    bassqi = baseline_power_ratio(ecg)

    if template_corr >= 0.66 and bsqi >= 0.80 \
       and not severe_spectral_failure(psqi, bassqi):
        raw_state = "GOOD"
    elif recognizable_qrs(r1, r2) and (
         template_corr_is_marginal(template_corr)
         or bsqi >= 0.60
         or rules.only_physio_rule_failed
         or spectral_suspicious(psqi, bassqi)):
        raw_state = "DEGRADED"
    else:
        raw_state = "BAD"

    if high_motion(imu_window) and raw_state != "GOOD":
        reasons.append("MOTION_ASSOCIATED_CORRUPTION")

    return apply_hysteresis(raw_state, score_ecg(...), reasons, state)
```

---

# 6. PPG trustworthiness — red and infrared channels

PPG must be assessed **per optical channel first**, then jointly when a downstream quantity needs both red and IR.

## 6.1 What “good enough” means for PPG

### `GOOD`

A usable PPG channel has:

- valid MAX30101 acquisition with no FIFO loss or ambient-light-cancellation overflow;
- no ADC saturation;
- sufficient DC optical signal and pulsatile AC component;
- repeated pulse morphology;
- reliable pulse detection;
- reasonable beat-to-beat timing consistency;
- low enough random/motion contamination that a pulse waveform is still identifiable.

Orphanidou et al. validated a PPG SQI with 91% sensitivity and 95% specificity and found an average template-correlation threshold of **0.86** [1].

### `DEGRADED`

Pulse structure remains detectable but:

- perfusion/pulsatile amplitude is weak;
- morphology is marginal;
- motion is present;
- spectral randomness is elevated;
- red and IR differ in quality;
- occasional optical/contact artifacts occur.

### `BAD`

- FIFO overflow/data loss;
- `ALC_OVF` ambient-light-cancellation overflow;
- channel saturation/clipping;
- no pulsatile structure;
- optical sensor off-body/poorly coupled;
- pulse morphology fails;
- motion/ambient light dominates the signal.

---

## 6.2 PPG hardware failure modes

The MAX30101 provides especially valuable native quality information:

- **FIFO overflow counter:** when full, samples are lost [15];
- **`ALC_OVF`:** the ambient-light cancellation circuit has reached its maximum limit and ambient light is affecting ADC output [15];
- **data-ready/FIFO pointers:** support acquisition continuity checks [15];
- programmable ADC full-scale range and LED current [15].

These hardware flags should be captured with every affected PPG window.

Other key PPG failure modes:

| Failure mode | Observable effect |
|---|---|
| Poor optical contact | weak/unstable AC pulse, morphology failure |
| Excessive or insufficient contact pressure | altered AC/DC ratio and waveform morphology |
| Sensor displacement | abrupt DC shift, morphology loss |
| Ambient light leakage | `ALC_OVF`, baseline/DC shifts, nonphysiological waveform |
| Motion | periodic/aperiodic deformation, false pulse peaks |
| Low perfusion | low pulsatile amplitude/PI |
| ADC saturation | rail hits, truncated pulses |
| Red/IR imbalance | one wavelength usable, the other poor |
| FIFO/transport loss | missing pulse segments |

Contact pressure measurably affects reflective PPG waveform amplitude and morphology [12]. Therefore, “a pulse exists” is not sufficient for morphology-sensitive uses.

---

## 6.3 PPG SQIs to implement

### 6.3.1 MAX30101 native status — hard gates

**Immediately `BAD` for the affected channel/window when:**

- FIFO overflow counter increments;
- `ALC_OVF` is asserted and persists into the data interval;
- power/reset/brownout occurred without required reinitialization;
- raw counts saturate the configured ADC range.

The MAX30101 explicitly states that `ALC_OVF` means ambient light is affecting ADC output [15].

---

### 6.3.2 Perfusion / AC-to-DC index

Elgendi defines the PPG perfusion SQI as [3]:

\[
P_{SQI}
=
\frac{y_{max}-y_{min}}
{|\bar{x}|}
\times 100,
\]

where \(x\) is the raw PPG signal and \(y\) is a filtered pulsatile component.

This approximates:

\[
PI \approx \frac{AC}{DC}\times100.
\]

**Interpretation:** the pulse-related optical variation must be sufficiently large relative to the static optical level.

**Strength:** simple, interpretable, useful for contact/perfusion problems.

**Failure cases:**

- no universal threshold transfers across wavelength, anatomical site, LED current, photodiode gain, skin properties, and contact pressure;
- motion can artificially increase peak-to-peak amplitude and therefore produce a misleadingly high PI.

**Implementation:** compute per red and IR channel. Use as a relative/adaptive quality feature, not a single hard threshold initially.

**Engineering starting strategy:** learn the distribution of PI during a device-specific “known-good contact” calibration and define low-PI degradation relative to that baseline (e.g. below the 5th percentile of validated clean windows).

---

### 6.3.3 PPG template correlation

Use the Orphanidou procedure on a 10-s pulse window:

1. detect pulse peaks;
2. estimate median pulse interval;
3. extract individual pulse waveforms;
4. time-normalize if necessary;
5. average them to form a local template;
6. correlate each pulse with the template;
7. average the correlations.

**Research-supported acceptance threshold:** **0.86** for PPG [1].

This is the primary morphology SQI for the recommended implementation.

A related real-time literature line also uses local pulse-template comparison because motion/artifact modifies pulse morphology [4,22].

**Failure case:** a periodic artifact can sometimes look repeatable. Therefore, combine with IMU and spectral/randomness evidence.

---

### 6.3.4 Skewness

Elgendi evaluated eight PPG SQIs and found skewness to be the best-performing single SQI among those tested for separating expert-rated quality categories [3].

\[
S_{SQI} =
\frac{1}{N}
\sum_{i=1}^{N}
\left(\frac{x_i-\mu}{\sigma}\right)^3.
\]

**Use:** secondary statistical quality feature.

**Do not hard-code a universal skewness cut-point.** Its distribution depends on sensor type, site, preprocessing, wavelength, and population.

---

### 6.3.5 Spectral entropy

Normalize the power spectral density in a target band:

\[
p_k = \frac{P_k}{\sum_j P_j}
\]

and calculate normalized spectral entropy:

\[
H =
-\frac{\sum_k p_k\log p_k}{\log K}.
\]

\(H \to 0\) for energy concentrated into a small number of spectral components; \(H \to 1\) for a flat/random spectrum.

Böttcher et al. used PPG/BVP spectral entropy over **4-s windows**, in approximately **0.1–5 Hz**, with an empirically determined poor-quality threshold **>0.8** for the Empatica E4 dataset [9].

**Research-supported threshold:** `H > 0.8` in that specific E4 implementation [9].

**For MAX30101:** treat 0.8 as an **engineering starting threshold only**, because sensor placement and preprocessing differ.

---

### 6.3.6 Orphanidou timing feasibility

The same 10-s feasibility rules used for ECG were applied to PPG [1]:

- HR 40–180 bpm;
- max successive pulse gap \(\le3\) s;
- max/min beat interval ratio <2.2.

Use them as supporting evidence for pulse-rate reliability.

As for ECG, do not confuse a genuine irregular rhythm with optical corruption. If morphology and acquisition are strong, a timing-rule failure should usually downgrade to `DEGRADED` rather than automatically mark raw PPG `BAD`.

---

### 6.3.7 Motion coincidence from IMU

Pflugradt and Orglmeister showed that accelerometer information can improve PPG quality indication because motion is a primary PPG disturbance source [13].

Compute a motion indicator from synchronized IMU:

\[
a_{mag}(t)=\sqrt{a_x^2+a_y^2+a_z^2}
\]

\[
a_{dyn}(t)=a_{mag}(t)-g
\]

with rolling RMS or robust dispersion.

**Important policy:** high motion is **not itself a PPG failure**. A PPG that retains strong morphology during motion remains usable. IMU should strengthen an artifact diagnosis when PPG quality simultaneously falls.

---

## 6.4 Red/IR pair consistency for SpO₂ authorization

SpO₂ is derived from both optical channels. Therefore:

```text
SpO2_quality <= min(red_quality, ir_quality)
```

SpO₂ use must be rejected if either channel has:

- `ALC_OVF`;
- clipping/saturation;
- insufficient pulsatility;
- poor morphology;
- missing samples;
- mismatched pulse timing.

Also require detected red and IR pulses to align in time within a calibration-defined tolerance because the two wavelengths represent the same cardiac pulses.

This does not calculate or interpret SpO₂. It only prevents a derived value from acquiring more trust than its source signals deserve.

---

## 6.5 Recommended PPG decision mechanism

Evaluate red and IR independently on rolling **10-s windows**, updated every **1 s**. Also calculate 4-s spectral entropy for fast noise detection.

### Per-channel decision

```text
if FIFO overflow OR ALC_OVF OR major data gap OR severe clipping:
    BAD
elif template_corr >= 0.86
     AND pulse detection stable
     AND pulsatile amplitude/PI above calibrated floor
     AND spectral entropy not strongly noise-like:
    GOOD
elif pulse morphology recognizable
     but one/more SQIs are marginal:
    DEGRADED
else:
    BAD
```

### Contact-pressure limitation

Without an actual pressure/contact-force sensor, the system can infer **poor coupling** from low PI, DC instability, morphology loss, and movement, but it cannot uniquely determine whether the cause is “too loose,” “too tight,” low perfusion, displacement, or another optical issue.

Research shows contact pressure changes PPG SNR and morphology [12]. If the final wearable enclosure permits it, direct contact-force/proximity sensing would improve diagnosability, but the quality pipeline must work without it.

---

## 6.6 PPG pseudocode

```python
def assess_ppg(channel, window_10s, max30101_status, imu_window, state):
    reasons = []

    if max30101_status.fifo_overflow:
        return BAD(0.0, ["FIFO_OVERFLOW"])
    if max30101_status.alc_overflow:
        return BAD(0.0, ["AMBIENT_LIGHT_OVERFLOW"])

    acq = acquisition_quality(window_10s)
    if acq.major_gap:
        return BAD(0.0, ["DATA_GAP"])
    if acq.severe_clipping:
        return BAD(0.0, ["CLIPPING"])
    if acq.flatline:
        return BAD(0.0, ["FLATLINE"])

    raw = window_10s.raw
    filtered = ppg_quality_bandpass(raw)

    peaks = detect_ppg_pulses(filtered)
    template_corr = pulse_template_correlation(filtered, peaks)
    pi = 100.0 * peak_to_peak(filtered) / abs(mean(raw))
    skew = skewness(raw)
    entropy = spectral_entropy(filtered, band=(0.1, 5.0))
    timing_rules = orphanidou_timing_rules(peaks)

    low_pi = below_calibrated_pi_floor(channel, pi)
    high_entropy = entropy > configured_entropy_threshold(channel)

    if template_corr >= 0.86 and not low_pi and not high_entropy \
       and stable_pulse_detection(peaks):
        raw_state = "GOOD"
    elif recognizable_pulses(peaks) and (
         template_corr_is_marginal(template_corr)
         or low_pi
         or high_entropy
         or timing_rules.only_physio_rule_failed):
        raw_state = "DEGRADED"
    else:
        raw_state = "BAD"

    if high_motion(imu_window) and raw_state != "GOOD":
        reasons.append("MOTION_ASSOCIATED_CORRUPTION")

    return apply_hysteresis(raw_state, score_ppg(...), reasons, state)
```

---

# 7. Electrodermal activity (EDA/GSR) trustworthiness

EDA differs from ECG and PPG: there is no repeating beat morphology that must always be present. Quality assessment should therefore focus on contact, range, impossible slopes/jumps, hardware saturation, and artifact shape.

## 7.1 What “good enough” means for EDA

### `GOOD`

- acquisition is continuous;
- electrodes are in contact;
- signal is within the usable sensor range;
- no floor/ceiling saturation;
- changes are compatible with EDA dynamics;
- no large abrupt contact/motion artifacts;
- signal remains usable after simple quality-only filtering.

Kleckner et al. proposed a transparent automated EDA quality procedure and reported 91% sensitivity, 99% specificity, and 92% accuracy against expert labels [5].

### `DEGRADED`

- EDA remains measurable but contains short suspicious jumps;
- contact is intermittent but not fully lost;
- movement coincides with a questionable transition;
- signal approaches device range limits;
- only a small fraction of the window is invalid.

### `BAD`

- contact loss/near-zero line;
- sensor overload/ceiling;
- impossible jump/slope;
- long data gap;
- rail saturation;
- a large fraction of the window is within an artifact-transition exclusion zone.

---

## 7.2 Research-supported EDA rules

Kleckner et al. used four rules [5]:

1. EDA within **0.05–60 µS**;
2. EDA change not faster than **±10 µS/s**;
3. skin temperature within **30–40 °C** as a worn/contact-support criterion;
4. invalidate data within **5 s** of detected artifact transitions.

These values were explicitly presented as adjustable for different hardware, populations, and contexts [5].

### Important portability note

The 60 µS ceiling was influenced by the Q Sensor hardware, and the 0.05 µS floor was selected as an accepted minimum SCR-related level/contact discriminator in that implementation [5]. For the project’s GSR front end, the exact high-end range must be determined from the analog circuit and ADC mapping.

Therefore:

- **0.05 µS floor:** research-supported starting point; validate with the current electrodes/site;
- **60 µS ceiling:** do not assume universal; replace with the actual calibrated hardware ceiling if different;
- **±10 µS/s:** strong starting artifact rule;
- **±5-s transition mask:** directly reusable and conservative.

---

## 7.3 EDA SQIs/features

### 7.3.1 Range validity

For each sample:

\[
valid_{range}(i)=
(EDA_i \ge L)\land(EDA_i\le U).
\]

Start with \(L=0.05\,\mu S\) and \(U=60\,\mu S\) only if the current hardware calibration confirms the same usable range [5].

Track:

\[
range\_valid\_fraction =
\frac{N_{valid}}{N}.
\]

A persistent near-zero value strongly suggests electrode loss in common dry-electrode wearable configurations [5,9].

---

### 7.3.2 Slope/jump detector

\[
s_i =
\frac{EDA_i-EDA_{i-1}}
{t_i-t_{i-1}}.
\]

Flag:

\[
|s_i| > 10\,\mu S/s
\]

using the Kleckner research-supported threshold [5].

This is particularly valuable because electrode movement/contact changes produce abrupt transitions that are much faster than typical underlying EDA evolution.

---

### 7.3.3 Artifact transition expansion

Whenever a range/slope artifact is detected, mark:

```text
[t_artifact - 5 s, t_artifact + 5 s]
```

invalid, following Kleckner’s transition rule [5].

This prevents the system from accepting the settling tail immediately adjacent to a contact artifact.

---

### 7.3.4 Temperature-supported contact

Kleckner used **30–40 °C** as a temperature criterion supporting the conclusion that the wearable was on-body [5].

Böttcher et al. used a broader **25–40 °C** wearable skin-temperature range for on-body/quality logic [9].

Recommended policy:

- temperature `<25 °C` or `>40 °C`: strong off-body/thermal-contact suspicion for this use case;
- 25–30 °C: do not automatically reject EDA, but lower contact confidence;
- 30–40 °C: strong positive contact evidence.

This use of temperature is only for measurement trustworthiness.

---

### 7.3.5 Derivative/wavelet artifact classifier — optional phase 2

Taylor et al. labeled **5-s EDA epochs** and trained artifact classifiers from:

- raw and filtered amplitude statistics;
- first derivative;
- second derivative;
- Haar wavelet coefficients at several scales.

Their binary SVM achieved **95.67% test accuracy** in their dataset [6].

This is a useful second-stage algorithm, but should **not** be the initial hard gate because:

- it was trained on a specific device and dataset;
- classifier generalization to this GSR front end is unknown;
- deterministic range/slope rules are easier to audit and calibrate.

Recommended use: after collecting labeled project data, train/validate a classifier to refine `GOOD` vs `DEGRADED`, while retaining deterministic hard gates.

---

## 7.4 Recommended EDA temporal behavior

Use two time scales:

### Sample-level / event-level

Continuously evaluate:

- hardware read status;
- range;
- slope;
- rail/stale detection.

### 5-s rolling quality epoch

Taylor’s artifact work used 5-s epochs [6]. Use a **5-s rolling window updated every 1 s** for aggregated EDA quality.

Within each 5-s window, retain:

- invalid-sample fraction;
- max absolute slope;
- contact confidence;
- number/duration of artifact masks;
- motion coincidence.

### Transition handling

A detected hard artifact expands ±5 s [5].

### Recovery

Do not return directly from `BAD` to `GOOD` at the first clean sample.

Engineering starting rule:

- require at least 5 s with no range/slope/contact failure after the artifact mask expires;
- recover `BAD -> DEGRADED`;
- require a second clean 5-s epoch for `DEGRADED -> GOOD`.

---

## 7.5 EDA decision logic

```text
if acquisition failure OR hardware saturation:
    BAD
elif EDA outside calibrated hardware/physiological-contact range:
    BAD
elif abs(dEDA/dt) > 10 µS/s:
    BAD for transition mask ±5 s
elif contact evidence weak OR small artifact fraction:
    DEGRADED
else:
    GOOD
```

Do **not** reject merely because the EDA level is unusually high/low within the validated sensor range.

---

## 7.6 EDA pseudocode

```python
def assess_eda(window_5s, temp_quality, imu_window, state):
    reasons = []

    acq = acquisition_quality(window_5s)
    if acq.major_gap:
        return BAD(0.0, ["DATA_GAP"])
    if acq.severe_clipping:
        return BAD(0.0, ["CLIPPING"])
    if acq.flatline_at_sensor_floor:
        return BAD(0.0, ["CONTACT_LOSS", "FLATLINE"])

    eda_q = eda_quality_filter(window_5s)

    range_bad = (eda_q < cfg.eda_low_uS) | (eda_q > cfg.eda_high_uS)
    slope = derivative_uS_per_sec(eda_q)
    jump_bad = abs(slope) > 10.0  # research-supported starting rule [5]

    if any(range_bad):
        add_artifact_mask(timestamp_of(range_bad), pre=5, post=5)
    if any(jump_bad):
        add_artifact_mask(timestamp_of(jump_bad), pre=5, post=5)

    invalid_fraction = artifact_mask_fraction(window_5s)

    if invalid_fraction >= cfg.eda_bad_fraction:
        raw_state = "BAD"
    elif invalid_fraction > 0 or temp_quality.contact_confidence == "LOW":
        raw_state = "DEGRADED"
    else:
        raw_state = "GOOD"

    if high_motion(imu_window) and any(jump_bad):
        reasons.append("MOTION_ASSOCIATED_CONTACT_ARTIFACT")

    return apply_eda_recovery_hysteresis(raw_state, ..., state)
```

`eda_bad_fraction` should initially be an engineering threshold (for example 0.20–0.30 of a 5-s epoch) and calibrated against expert labels. Individual Kleckner rule violations still create point/transition invalidity regardless of the aggregate threshold.

---

# 8. Skin temperature trustworthiness

Skin temperature requires a different quality model because it is not a repetitive waveform. Quality is primarily about:

1. whether the digital sensor is functioning;
2. whether the sensor is physically coupled to skin;
3. whether the reading is dynamically plausible;
4. whether thermal settling has occurred;
5. whether environmental/microclimate bias is likely to be material.

## 8.1 What “good enough” means

### `GOOD`

- TMP117 reports fresh conversions;
- acquisition timing is correct;
- reading lies in a plausible on-body skin-temperature range;
- no abrupt nonthermal jumps;
- temperature is thermally settled after attachment;
- contact is corroborated by at least one independent on-body indicator where available.

### `DEGRADED`

- likely on-body, but recent attachment/removal or rapid change means thermal equilibrium is uncertain;
- value is in a marginal on-body range;
- temperature is plausible but environmental bias/contact confidence is uncertain.

### `BAD`

- stale/no conversion;
- transport failure;
- reading near ambient/off-body state with supporting evidence;
- abrupt impossible jump;
- out-of-range sensor behavior;
- sensor detached.

---

## 8.2 Sensor-native integrity

TMP117 provides:

- a **Data Ready** flag indicating conversion completion;
- programmable conversion timing and averaging;
- high/low alerts;
- 16-bit output with 0.0078 °C resolution;
- specified accuracy up to ±0.1 °C over −20 to 50 °C [16].

Use `Data_Ready` to distinguish a fresh measurement from a stale register read [16].

### Hard failures

- no fresh conversion by expected deadline;
- I²C error;
- repeated stale sample despite expected conversions;
- reset/configuration mismatch;
- impossible sensor-code value.

---

## 8.3 On-body/contact plausibility

Böttcher et al. used **25–40 °C** for wearable skin-temperature quality/on-body assessment [9]. Kleckner et al. used **30–40 °C** as supporting evidence that an EDA wearable was being worn [5].

Recommended hierarchy:

```text
25–40 °C    broad research-supported on-body plausibility band
30–40 °C    stronger positive contact evidence in the Kleckner wearable rule
<25 °C      strong off-body/contact-loss suspicion
>40 °C      abnormal for ordinary skin-contact monitoring; inspect environment/device/contact
```

These are not diagnostic body-temperature thresholds.

---

## 8.4 Rapid-change quality check

Böttcher et al. calculated a rate-of-amplitude-change measure in 2-s windows and considered >20% change poor quality for temperature/EDA in their wearable dataset [9].

Use this as a **gross discontinuity detector**, not as the only temperature quality rule.

A more physically interpretable engineering metric should also be logged:

\[
s_T = \frac{dT}{dt}.
\]

Because there is no universal published derivative threshold for this exact TMP117 mounting, use a calibrated limit.

**Engineering starting threshold:** flag `TEMP_RAPID_CHANGE` if a robust 2-s slope exceeds roughly **1 °C/s**, and mark `DEGRADED` unless concurrent evidence clearly indicates detachment/reattachment, in which case mark the transition `BAD/NOT_READY`.

The 1 °C/s value is intentionally labeled `ENGINEERING_START` and must be replaced after bench/on-body step tests.

---

## 8.5 Thermal settling after attachment

A temperature sensor can be electrically healthy but not yet representative of skin because it needs time to equilibrate.

No universal settling time transfers across:

- package/enclosure;
- adhesive/strap pressure;
- sensor thermal mass;
- skin-to-ambient gradient;
- airflow;
- body site.

MacRae et al. show that contact-sensor bias depends on attachment and skin-to-environment/microclimate gradients [10,11].

Recommended implementation:

```text
on suspected new attachment:
    state = NOT_READY
    collect temperature history

exit NOT_READY only when:
    value is in broad on-body range
    AND recent linear slope has stabilized
    AND no detachment evidence
```

**Engineering starting values:**

- minimum post-attachment observation: 30 s;
- slope stability: `abs(robust_slope_20s) < 0.02 °C/s`;
- require 10 continuous seconds satisfying both before `GOOD`.

These values require empirical characterization of the actual enclosure.

---

## 8.6 Environmental-gradient limitation

A critical limitation is that a contact temperature sensor can be well attached and still be biased by the skin-to-environment gradient. MacRae et al. found contact-sensor bias changed with environmental/microclimate conditions and attachment [10,11].

Therefore the quality layer should distinguish:

```text
sensor functioning
skin contact likely
thermally settled
metrological environmental confidence
```

With the current hardware, the first three can be assessed reasonably. The fourth cannot be fully proven without an ambient/microclimate reference or a validated thermal model.

Do not claim `GOOD` means “unbiased core/body temperature.” It means **the skin-contact temperature stream is trustworthy as a skin-surface measurement within the validated operating conditions**.

---

## 8.7 Temperature pseudocode

```python
def assess_skin_temp(samples, tmp117_status, eda_contact, ppg_contact, state):
    reasons = []

    if not tmp117_status.fresh_conversion:
        return BAD(0.0, ["STALE_SENSOR_DATA"])

    acq = acquisition_quality(samples)
    if acq.major_gap:
        return BAD(0.0, ["DATA_GAP"])

    T = latest(samples)
    plausible = (25.0 <= T <= 40.0)  # literature-supported wearable band [9]

    rac_bad = rate_amplitude_change_2s(samples) > 0.20  # [9]
    slope = robust_linear_slope(samples.last_seconds(20))

    contact_support = combine_contact_evidence(
        temperature_plausible=plausible,
        eda_contact=eda_contact,
        ppg_contact=ppg_contact
    )

    if contact_support == "OFF_BODY":
        raw_state = "BAD"
    elif state.just_attached:
        raw_state = "NOT_READY"
    elif rac_bad:
        raw_state = "DEGRADED"
    elif plausible and abs(slope) < cfg.temp_stable_slope:
        raw_state = "GOOD"
    else:
        raw_state = "DEGRADED"

    return apply_temp_settling_and_hysteresis(raw_state, ..., state)
```

---

# 9. IMU trustworthiness and its role as a quality witness

The IMU is not itself a physiological waveform here. It has two roles:

1. provide reliable motion data;
2. help determine whether corruption in ECG/PPG/EDA coincides with movement.

It must first pass its own integrity checks before it can be trusted as a witness.

## 9.1 IMU failure modes

- FIFO overflow;
- missing `DATA_RDY`;
- I²C error;
- axis stuck at a fixed code;
- clipping at configured accelerometer/gyroscope full-scale range;
- incorrect scale/configuration register;
- sensor self-test failure;
- gross bias/calibration failure;
- sample timing irregularity.

The MPU-6050 provides programmable accelerometer ranges ±2/4/8/16 g and gyroscope ranges ±250/500/1000/2000°/s and includes self-test and interrupt/status mechanisms [17].

## 9.2 IMU integrity checks

### Startup / periodic maintenance

Run manufacturer self-test during initialization or controlled maintenance periods, not during live physiological recording if it disturbs output.

### Per sample

- finite integer decode;
- timestamp monotonic;
- no I²C error;
- compare each axis against configured full-scale range.

### Per 1-s window

- clipping fraction per axis;
- stale/repeated code runs;
- sample-count/timing integrity;
- robust variance.

### Stationary sanity check

When the system independently detects a stationary interval from low gyro/acceleration variance, the accelerometer norm should be near gravity:

\[
\|a\|=\sqrt{a_x^2+a_y^2+a_z^2}\approx g.
\]

A tolerance such as ±0.15 g is a reasonable **engineering starting check**, not a universal specification. Use the actual calibrated MPU-6050 bias/scale error and enclosure orientation tests to set the bound.

---

## 9.3 Motion score for cross-sensor quality

Compute:

\[
a_{dyn}(t)=\|a(t)\|-g
\]

and robust RMS:

\[
M_a =
\sqrt{\frac{1}{N}\sum a_{dyn}(t)^2}.
\]

Also compute gyroscope magnitude:

\[
M_\omega =
\sqrt{\frac{1}{N}\sum (\omega_x^2+\omega_y^2+\omega_z^2)}.
\]

Rather than assuming one universal activity threshold, calibrate a `motion_score` from clean resting data using median and MAD:

\[
z_M =
\frac{M_a-\operatorname{median}(M_a)}
{1.4826\,MAD(M_a)}.
\]

Use a high robust z-score or device-specific percentile to mark `MOTION_HIGH`.

**Key rule:**

> IMU motion alone must never invalidate ECG/PPG/EDA.

Use it to assign `MOTION_ASSOCIATED_CORRUPTION` only when the physiological signal’s own SQIs simultaneously deteriorate.

This follows the quality-assistance role demonstrated for PPG/accelerometer combinations [13] and avoids discarding valid biosignals obtained during movement.

---

# 10. Cross-sensor quality verification

Cross-sensor evidence is allowed only to answer “is this measurement trustworthy?”

## 10.1 PPG + IMU

```text
PPG morphology ↓ + spectral entropy ↑ + motion ↑
    -> strong MOTION_ARTIFACT evidence

PPG morphology ↓ + motion low
    -> suspect optical contact / ambient light / perfusion / displacement

motion high + PPG morphology remains strong
    -> keep PPG GOOD
```

## 10.2 ECG + IMU

```text
ECG template correlation ↓ + bSQI ↓ + motion ↑
    -> MOTION_ASSOCIATED_CORRUPTION

ECG lead-off asserted
    -> ELECTRODE_OFF regardless of motion

motion high but ECG template/bSQI good
    -> do not reject ECG
```

## 10.3 EDA + IMU

```text
EDA abrupt |dS/dt| violation + motion spike
    -> likely contact/motion artifact

EDA smooth response + motion spike
    -> do not reject merely because subject moved
```

## 10.4 EDA + temperature for contact

Temperature can support the conclusion that an EDA sensor is on-body, as in Kleckner’s quality rules [5].

Do not require temperature 30–40 °C as the only condition; cold peripheral skin and environmental exposure can produce lower values. Use it as corroborative contact evidence.

## 10.5 PPG/EDA/temperature for device-on-body confidence

A small fused `on_body_confidence` is useful:

```text
strong evidence:
    valid PPG pulsatility
    OR EDA above contact floor
    OR skin temperature in plausible on-body band

strong off-body evidence:
    EDA near zero/contact floor
    AND temperature trends toward ambient/off-body range
    AND PPG has no physiologic pulse morphology
```

Do not use IMU inactivity as off-body proof: a worn device can be stationary.

---

# 11. Reason-code vocabulary

Use explicit bit flags or an enum set.

```text
NO_DATA
DATA_GAP
TIMESTAMP_JITTER
PACKET_LOSS
FIFO_OVERFLOW
STALE_SENSOR_DATA
I2C_ERROR
SENSOR_RESET

CLIPPING
ADC_RAIL
FLATLINE
LOW_AMPLITUDE
RANGE_FAILURE

ELECTRODE_OFF
LOW_CONTACT
CONTACT_UNCERTAIN
OPTICAL_CONTACT_POOR
AMBIENT_LIGHT_OVERFLOW
OFF_BODY
THERMAL_NOT_SETTLED

MORPHOLOGY_FAILURE
LOW_TEMPLATE_CORRELATION
LOW_DETECTOR_AGREEMENT
BASELINE_WANDER
HIGH_FREQUENCY_NOISE
HIGH_SPECTRAL_ENTROPY
LOW_PERFUSION_INDEX
PULSE_DETECTION_FAILURE
QRS_DETECTION_FAILURE

IMPLAUSIBLE_RATE_FOR_SQI_MODEL
IBI_GAP_RULE_FAILURE
IBI_RATIO_RULE_FAILURE

EDA_RANGE_FAILURE
EDA_SLOPE_ARTIFACT
EDA_TRANSITION_MASK

TEMP_OUTSIDE_ON_BODY_RANGE
TEMP_RAPID_CHANGE
TEMP_ENVIRONMENTAL_CONFIDENCE_LOW

IMU_CLIPPING
IMU_SELF_TEST_FAILURE
IMU_STUCK_AXIS
IMU_CALIBRATION_SUSPECT
MOTION_ASSOCIATED_CORRUPTION
```

Reason codes should be additive. A window may have several simultaneous causes.

---

# 12. Output data structure

Recommended schema:

```python
class SignalQualityRecord:
    modality: str                 # ECG, PPG_RED, PPG_IR, EDA, TEMP, IMU
    window_start_ns: int
    window_end_ns: int

    state: str                    # NOT_READY, GOOD, DEGRADED, BAD
    quality_score: float          # 0..1, modality-calibrated
    hard_veto: bool

    reasons: list[str]

    acquisition:
        expected_samples: int
        received_samples: int
        missing_fraction: float
        max_gap_ms: float
        jitter_p95_ms: float
        clipping_fraction: float

    metrics: dict[str, float]     # modality-specific SQIs
    motion_score: float | None
    contact_confidence: float | None

    # Which downstream categories are currently authorized.
    # These are measurement-use categories, not clinical interpretations.
    permissions: set[str]
```

Example ECG record:

```json
{
  "modality": "ECG",
  "state": "DEGRADED",
  "quality_score": 0.74,
  "hard_veto": false,
  "reasons": ["BASELINE_WANDER"],
  "metrics": {
    "template_corr": 0.83,
    "bsqi": 0.91,
    "psqi": 0.62,
    "bassqi": 0.91
  },
  "permissions": ["RATE"],
  "motion_score": 0.18
}
```

Example PPG record:

```json
{
  "modality": "PPG_IR",
  "state": "BAD",
  "quality_score": 0.11,
  "hard_veto": true,
  "reasons": ["AMBIENT_LIGHT_OVERFLOW"],
  "permissions": []
}
```

---

# 13. Feature-specific authorization

A single raw-signal state is useful but not always sufficient because different downstream measurements tolerate different errors.

Examples:

- HR/pulse rate can tolerate some waveform-shape degradation if beat timing remains reliable.
- HRV is much more sensitive to missed/extra beats; beat timing must be exceptionally clean.
- SpO₂ requires both red and IR channels.
- morphology-derived PPG features require higher waveform fidelity than simple pulse rate.

Therefore, downstream permission should be explicit.

Suggested starting policy:

## ECG

```text
RATE:
    GOOD
    or DEGRADED only if bSQI >= 0.80 and beat detection stable

HRV:
    GOOD only
    bSQI >= 0.90 preferred
    no data gaps
    no missed/extra-beat evidence

MORPHOLOGY:
    GOOD only
    template correlation strong
    spectral/baseline quality acceptable
```

## PPG

```text
PULSE_RATE:
    GOOD
    or DEGRADED if pulse timing stable

PULSE_VARIABILITY:
    GOOD only
    strict pulse detector agreement / morphology

SpO2_SOURCE:
    RED GOOD + IR GOOD
    no ALC overflow
    no clipping
    pulse alignment stable

PPG_MORPHOLOGY:
    GOOD only
```

The HRV caution is supported by work showing that PPG/beat-detection error propagates nonlinearly into HRV measures [20].

---

# 14. Continuous windowing plan

| Modality | Fast checks | Main SQI window | Update | Retained state |
|---|---|---:|---:|---|
| ECG | per sample/packet | 10 s | 1 s | last peaks, template, SQIs, hysteresis |
| PPG red/IR | per sample/packet + MAX status | 10 s morphology; 4 s entropy | 1 s | pulse template, PI baseline, SQIs |
| EDA | per sample slope/range | 5 s | 1 s | artifact mask ±5 s, contact state |
| Skin temp | each conversion | 2 s jump + 20 s slope + settling | 2 s | contact/settling state |
| IMU | each sample/status | 1 s | 1 s | bias/stationary baseline, motion score |

This preserves the literature’s 10-s Orphanidou ECG/PPG segments [1], Taylor’s 5-s EDA artifact epochs [6], Böttcher’s 2-s TEMP/EDA rapid-change check and 4-s PPG entropy window [9], while allowing a 1-s system update cadence.

---

# 15. Quality-score construction

The recommended production gate is rule-based, but a numeric score is still useful for logs, UI, retrospective analysis, and future calibration.

Do not average unrelated SQIs blindly.

## 15.1 Stage scores

Calculate separate normalized scores:

```text
q_acquisition
q_contact
q_waveform
q_timing
q_motion_robustness
```

A hard veto sets the final score to 0.

For surviving data use a conservative fusion such as a weighted geometric mean:

\[
Q =
\prod_j q_j^{w_j},
\qquad
\sum_jw_j=1.
\]

The geometric mean is preferable to an arithmetic mean because one weak essential dimension cannot be completely hidden by several strong dimensions.

**Weights are engineering parameters** and must be calibrated. Until calibration, keep the rule-based state as authoritative.

## 15.2 Suggested initial components

### ECG

```text
q_acquisition    = completeness/clipping score
q_contact        = 1 if lead-on; 0 if lead-off
q_waveform       = normalized template-correlation + spectral quality
q_timing         = bSQI + detector stability
q_motion         = penalty only when motion coincides with waveform degradation
```

### PPG

```text
q_acquisition    = FIFO/ALC/continuity/clipping
q_contact        = PI/DC stability/contact confidence
q_waveform       = template correlation + skewness/entropy evidence
q_timing         = pulse-detection consistency
q_motion         = conditional penalty
```

### EDA

```text
q_acquisition    = continuity/clipping
q_contact        = range + temperature-supported contact
q_waveform       = fraction not violating slope/range/artifact mask
```

### Temperature

```text
q_acquisition    = fresh-conversion/completeness
q_contact        = on-body plausibility
q_waveform       = dynamic continuity/stability
q_settled        = thermal-settling confidence
```

### IMU

```text
q_acquisition    = completeness/FIFO
q_sensor         = self-test + no clipping/stuck axes
q_calibration    = stationary gravity/bias sanity
```

---

# 16. Recommended Implementation

This is the single consolidated implementation recommended for the project.

## 16.1 Processing sequence

### Stage 0 — configuration integrity

At boot and after every sensor reset:

1. read back sensor configuration;
2. verify expected sampling rates/gain/ranges;
3. record firmware/configuration version;
4. run IMU self-test when allowed;
5. verify AD8232 lead-off lines;
6. verify MAX30101 interrupt/FIFO status;
7. verify TMP117 conversion mode/data-ready behavior.

No modality reaches `GOOD` until configuration checks succeed.

---

### Stage 1 — per-sample/per-packet acquisition validation

For every stream:

1. validate sequence/timestamp;
2. count lost/duplicate samples;
3. validate numeric decode;
4. detect stale repeated values;
5. detect rail/clipping;
6. ingest sensor-native error flags.

Emit immediate hard reasons such as:

```text
FIFO_OVERFLOW
AMBIENT_LIGHT_OVERFLOW
ELECTRODE_OFF
I2C_ERROR
DATA_GAP
```

---

### Stage 2 — contact/sensor sanity

**ECG:** AD8232 lead-off + amplitude/flatline.

**PPG:** DC level, PI/pulsatility, ALC overflow, optical stability.

**EDA:** conductance floor/range + temperature support.

**Temperature:** on-body range + settling.

**IMU:** self-test/config/range + stationary sanity.

---

### Stage 3 — modality-specific SQIs

**ECG**
- two QRS detectors;
- bSQI;
- 10-s template correlation;
- Orphanidou feasibility rules;
- pSQI;
- basSQI.

**PPG red + IR independently**
- pulse detection;
- 10-s template correlation;
- PI;
- skewness;
- 4-s spectral entropy;
- Orphanidou timing rules.

**EDA**
- calibrated range validity;
- \(|dEDA/dt|\);
- ±5-s transition mask;
- optional later Taylor-style artifact classifier.

**Temperature**
- fresh-conversion check;
- on-body range;
- 2-s rapid-change metric;
- 20-s robust slope;
- thermal settling state.

**IMU**
- clipping/stuck-axis;
- sample integrity;
- motion magnitude;
- robust motion score.

---

### Stage 4 — cross-sensor quality evidence

Only after each sensor’s own integrity is known:

```text
if biosignal SQI poor AND IMU motion high:
    add MOTION_ASSOCIATED_CORRUPTION

if EDA contact weak AND TEMP off-body:
    strengthen OFF_BODY/CONTACT_LOSS

if PPG has no pulse morphology AND TEMP/EDA indicate off-body:
    strengthen OPTICAL_CONTACT_POOR/OFF_BODY
```

Cross-sensor evidence may strengthen or explain a poor-quality decision. It should not convert an otherwise high-quality waveform to `BAD` merely because another sensor changed.

---

### Stage 5 — state decision

Apply hard vetoes first.

Then apply modality rules.

Then apply hysteresis.

Never let a numeric composite score override:

- electrode-off;
- FIFO overflow causing lost data;
- ambient-light-cancellation overflow;
- severe clipping;
- major data gap;
- known stale sensor data.

---

### Stage 6 — downstream authorization

Every emitted sample/window is accompanied by the latest quality record.

Downstream code must require explicit permission:

```python
if quality.permissions.contains("HRV"):
    process_hrv(ecg)
else:
    skip_hrv()
```

Do not rely on downstream modules remembering their own ad hoc quality rules.

---

# 17. Initial threshold registry

The table below separates published values from project-specific starting values.

| Modality | Metric | Initial threshold | Status | Source |
|---|---|---:|---|---|
| ECG | SQI window | 10 s | Research-supported | Orphanidou [1] |
| ECG | HR feasibility | 40–180 bpm | Research-supported for their wearable HR SQI | [1] |
| ECG | max beat gap | ≤3 s | Research-supported | [1] |
| ECG | RR max/min | <2.2 | Research-supported | [1] |
| ECG | template correlation | ≥0.66 | Research-supported | [1] |
| ECG | bSQI acceptable | ≥0.80 | Literature-supported common operating value | [21] |
| ECG | bSQI strong | ≥0.90 | Literature-supported strict category | [21] |
| ECG | pSQI | ~0.5–0.8 optimal in cited implementation | Literature-supported but nonportable | [21] |
| ECG | basSQI | ≥0.95 optimal; 0.90–0.95 suspicious | Literature-supported but nonportable | [21] |
| ECG/PPG | severe clip fraction | ≥1% / 10 s | Engineering start | project calibration required |
| PPG | template correlation | ≥0.86 | Research-supported | [1] |
| PPG | spectral entropy poor | >0.8 | Research-supported for Empatica E4; engineering start here | [9] |
| PPG | PI floor | clean-data 5th percentile initially | Engineering start | calibrate; PI method [3] |
| EDA | lower level | 0.05 µS | Research-supported starting point | [5] |
| EDA | upper level | 60 µS | Research-supported for Q Sensor; hardware-specific here | [5] |
| EDA | max slope | ±10 µS/s | Research-supported | [5] |
| EDA | artifact margin | ±5 s | Research-supported | [5] |
| EDA | epoch | 5 s | Research-supported artifact-classifier epoch | [6] |
| TEMP | broad on-body range | 25–40 °C | Research-supported wearable rule | [9] |
| TEMP | strong contact-support range | 30–40 °C | Research-supported in EDA QA procedure | [5] |
| TEMP | >20% change / 2 s | poor in cited wearable method | Research-supported for that dataset | [9] |
| TEMP | stable slope | <0.02 °C/s over 20 s | Engineering start | calibrate enclosure |
| TEMP | minimum reattach settling | 30 s | Engineering start | calibrate enclosure |
| IMU | stationary \(|a|-g\) tolerance | ±0.15 g | Engineering start | calibrate device |
| all | missing fraction degraded | >1% | Engineering start | validate |
| ECG/PPG | missing fraction bad | >5% | Engineering start | validate |

---

# 18. Final per-modality algorithms

## 18.1 ECG

```text
INPUT:
    raw ECG samples
    timestamps/sequence
    AD8232 lead-off status
    synchronized IMU

1. Verify packet/sample continuity.
2. If lead-off -> BAD immediately.
3. Detect rail/clipping and stale/flat data.
4. Apply only the filter required for SQI/QRS detection.
5. Run two independent QRS detectors.
6. Compute bSQI.
7. Compute 10-s adaptive beat-template correlation.
8. Compute Orphanidou HR/gap/IBI-ratio rules.
9. Compute pSQI and basSQI.
10. Check whether poor SQIs coincide with motion.
11. Apply ECG state rules.
12. Apply hysteresis.
13. Emit quality state/score/reasons + permissions.
```

## 18.2 PPG red and IR

```text
INPUT:
    raw red/IR samples
    MAX30101 status/FIFO
    timestamps
    synchronized IMU

1. Verify MAX30101 FIFO integrity.
2. If ALC_OVF -> BAD affected interval.
3. Detect clipping/saturation and data gaps.
4. Compute raw DC level and filtered pulsatile component.
5. Compute perfusion index.
6. Detect pulse peaks.
7. Compute 10-s pulse-template correlation.
8. Compute skewness.
9. Compute 4-s spectral entropy.
10. Evaluate pulse timing feasibility.
11. Use IMU only as supporting motion evidence.
12. Classify red and IR separately.
13. For dual-channel use, require both channels to satisfy the
    downstream quality policy.
14. Apply hysteresis and emit reasons/permissions.
```

## 18.3 EDA

```text
INPUT:
    calibrated EDA in µS
    timestamps
    temperature contact evidence
    synchronized IMU

1. Verify acquisition continuity.
2. Detect ADC/sensor saturation and stale floor.
3. Check calibrated EDA range.
4. Compute dEDA/dt continuously.
5. If |dEDA/dt| > 10 µS/s, create artifact event.
6. Expand invalid interval ±5 s.
7. Add temperature-supported contact confidence.
8. If abrupt artifact coincides with motion, add motion/contact reason.
9. Aggregate validity in rolling 5-s window.
10. Apply recovery hysteresis.
11. Emit state/score/reasons.
```

## 18.4 Skin temperature

```text
INPUT:
    TMP117 samples/status
    timestamps
    EDA/PPG contact evidence

1. Require fresh conversion (Data_Ready behavior).
2. Verify timing/no stale reads.
3. Check broad on-body plausibility (25–40 °C).
4. Compute 2-s rapid-change measure.
5. Compute robust 20-s temperature slope.
6. Detect new attachment/re-attachment.
7. Hold NOT_READY during thermal settling.
8. Fuse contact evidence only for on-body confidence.
9. Classify GOOD/DEGRADED/BAD/NOT_READY.
10. Emit thermal/contact reasons.
```

## 18.5 IMU

```text
INPUT:
    accel + gyro
    sensor configuration/status
    timestamps

1. Verify configured FSR/sample rate.
2. Run startup self-test.
3. Verify data-ready/FIFO and timing.
4. Detect axis clipping.
5. Detect stuck axes/stale data.
6. During stationary intervals, check |a| ≈ g and gyro bias.
7. Compute motion score every 1 s.
8. Emit IMU quality.
9. Supply motion score to other SQI modules only if IMU quality
   itself is acceptable.
```

---

# 19. Integrated Signal Trustworthiness Layer

```text
                    ┌───────────────────────────┐
                    │  raw sensor acquisition   │
                    └─────────────┬─────────────┘
                                  │
                    per sample / packet
                                  │
                    ┌─────────────▼─────────────┐
                    │ acquisition integrity      │
                    │ timestamps / seq / FIFO    │
                    │ bus errors / stale / rails │
                    └─────────────┬─────────────┘
                                  │
                         hard failure veto?
                         │                 │
                        yes               no
                         │                 │
                 ┌───────▼──────┐   ┌────▼──────────────────┐
                 │ BAD + reason │   │ contact/sensor sanity │
                 └──────────────┘   └────┬──────────────────┘
                                         │
                              ┌──────────▼───────────┐
                              │ modality-specific    │
                              │ SQIs                 │
                              │ ECG morphology/bSQI │
                              │ PPG morphology/PI   │
                              │ EDA range/slope     │
                              │ TEMP stability      │
                              └──────────┬───────────┘
                                         │
                              ┌──────────▼───────────┐
                              │ supporting evidence │
                              │ plausibility + IMU  │
                              │ quality assistance  │
                              └──────────┬───────────┘
                                         │
                              ┌──────────▼────────────┐
                              │ rule-based decision   │
                              │ + hysteresis          │
                              └──────────┬────────────┘
                                         │
                              ┌──────────▼────────────┐
                              │ QualityRecord         │
                              │ state / score         │
                              │ reasons / permissions │
                              └──────────┬────────────┘
                                         │
                                  explicit gate
                                         │
                              ┌──────────▼────────────┐
                              │ downstream processing │
                              │ trusted data only     │
                              └───────────────────────┘
```

---

# 20. State that must be retained

The layer is stateful. Retain at minimum:

## Common

- expected/configured sampling rate;
- previous sequence/timestamp;
- sample-loss counters;
- previous sensor reset/configuration version;
- rolling clipping/flatline statistics;
- current quality state;
- consecutive good/bad evaluation counts;
- last reason codes.

## ECG

- recent QRS detections from both detectors;
- local beat template;
- bSQI/template/pSQI/basSQI history.

## PPG

- recent pulse detections;
- per-channel pulse template;
- PI baseline distribution;
- spectral entropy history;
- MAX30101 FIFO/ALC events.

## EDA

- last EDA sample/time;
- active artifact-mask intervals;
- contact confidence;
- recent slope/range history.

## Temperature

- last fresh conversion time;
- recent 20–60 s temperature buffer;
- attach/detach/settling state.

## IMU

- configuration/full-scale range;
- self-test result;
- stationary calibration baseline;
- recent motion score.

---

# 21. Validation plan before trusting the quality layer

The literature provides the algorithms; the final thresholds still require device-specific verification.

Create a labeled quality dataset containing controlled examples of:

### ECG

- good stationary ECG;
- electrode partially peeled;
- full lead-off;
- motion with intact electrodes;
- deliberate baseline movement;
- induced ADC saturation;
- packet loss.

### PPG

- good contact;
- loose contact;
- excessive strap/contact force if mechanically possible;
- finger/wrist displacement;
- strong ambient light;
- controlled motion;
- low perfusion conditions that are ethically/operationally reasonable;
- FIFO overflow simulation.

### EDA

- good contact;
- one electrode lifted;
- movement/pressure shifts;
- sensor floor;
- injected step/jump;
- high-end saturation.

### Temperature

- sensor off-body to on-body transition;
- on-body to off-body;
- different ambient temperatures;
- airflow/covering;
- controlled thermal step.

### IMU

- each FSR tested near limits;
- stationary orientations;
- known rotation/motion;
- injected sample loss/stale register;
- startup self-test.

Label each relevant interval as:

```text
GOOD
DEGRADED
BAD
```

and annotate causes.

Then:

1. measure sensitivity to bad signal and specificity for good signal;
2. optimize only `ENGINEERING_START` thresholds;
3. keep published thresholds unless project data provides a defensible reason to change them;
4. test separately under motion and rest;
5. test per placement/site;
6. preserve a held-out validation set;
7. version the quality configuration with firmware/software.

The objective is **not** to train a large black-box model first. The first system should be interpretable, deterministic, and traceable to sensor evidence. Machine learning can later refine ambiguous EDA/PPG quality once enough labeled project data exists.

---

# 22. Concise synthesis

| Modality | Main failure modes | Primary SQIs/checks | Secondary SQIs | Window | Decision method | Output | Key references |
|---|---|---|---|---|---|---|---|
| ECG | lead-off, motion, baseline wander, high-frequency noise, clipping, gaps, flatline | AD8232 lead-off, Orphanidou template corr, bSQI | pSQI, basSQI, skew/kurtosis, IMU coincidence | 10 s, update 1 s | hard gates + morphology + detector agreement | state/score/reasons + rate/HRV permissions | [1], [2], [14], [21] |
| PPG red | poor contact, motion, low pulsatility, ambient light, saturation, FIFO loss | MAX ALC/FIFO, template corr, PI | skewness, spectral entropy, timing rules, IMU | 10 s; entropy 4 s | hard gates + pulse morphology + optical strength | per-channel state/score/reasons | [1], [3], [9], [12], [13], [15] |
| PPG IR | same as red | same as red | same as red | same | same | per-channel state/score/reasons | same |
| EDA/GSR | electrode loss, jumps, motion/contact artifact, floor/ceiling, saturation, gaps | range, ±10 µS/s slope, ±5-s transition mask | temperature contact support, optional derivative/wavelet classifier, IMU | 5 s, update 1 s | deterministic rule hierarchy | state/valid fraction/reasons | [5], [6], [8], [9] |
| Skin temperature | off-body, stale read, thermal settling, environmental bias, abrupt jumps | TMP117 freshness, 25–40 °C plausibility, settling state | 2-s RAC, robust slope, EDA/PPG contact corroboration | 2 s + 20 s slope | sensor sanity + contact + temporal stability | state/settling/contact reasons | [5], [9], [10], [11], [16] |
| IMU | clipping, stale axes, FIFO/timing errors, bias/config/self-test failure | sensor status, FSR clipping, self-test, timing | stationary \(|a|\approx g\), robust motion score | 1 s | self-quality first; then quality witness | IMU quality + motion score | [13], [17], [18], [19] |

---

# 23. Practical implementation order

Implement in this order:

1. **Common acquisition-integrity library**
   - timestamps, sequence numbers, sample completeness;
   - clipping/flatline/stale detection;
   - sensor status events.

2. **ECG quality**
   - AD8232 lead-off;
   - dual QRS detectors + bSQI;
   - Orphanidou template SQI;
   - spectral diagnostics.

3. **PPG quality**
   - MAX30101 FIFO/ALC handling;
   - per-channel clipping/DC/PI;
   - pulse template correlation;
   - spectral entropy;
   - red/IR pair authorization.

4. **EDA quality**
   - Kleckner rules;
   - transition masks;
   - temperature/contact corroboration.

5. **Temperature quality**
   - Data Ready/freshness;
   - on-body/contact plausibility;
   - settling state.

6. **IMU quality + motion assistance**
   - self-quality;
   - robust motion score;
   - conditional reason attribution.

7. **Unified QualityRecord + permissions + hysteresis**

8. **Controlled quality-validation dataset**
   - calibrate only hardware-specific thresholds;
   - quantify confusion matrix per modality.

This sequence produces a useful trustworthiness layer early while preserving a path toward more sophisticated, device-trained SQI classifiers.

---

# 24. References

[1] C. Orphanidou, T. Bonnici, P. H. Charlton, D. A. Clifton, D. Vallance, and L. Tarassenko.  
“Signal-Quality Indices for the Electrocardiogram and Photoplethysmogram: Derivation and Applications to Wireless Monitoring.”  
*IEEE Journal of Biomedical and Health Informatics*, 19(3):832–838, 2015.  
DOI: 10.1109/JBHI.2014.2338351  
PubMed: https://pubmed.ncbi.nlm.nih.gov/25069129/  
Author-hosted paper: https://www.robots.ox.ac.uk/~davidc/pubs/jbhi_sqi2015.pdf

[2] G. D. Clifford, J. Behar, Q. Li, and I. Rezek.  
“Signal quality indices and data fusion for determining clinical acceptability of electrocardiograms.”  
*Physiological Measurement*, 33(9):1419–1433, 2012.  
DOI: 10.1088/0967-3334/33/9/1419  
PubMed: https://pubmed.ncbi.nlm.nih.gov/22902749/

[3] M. Elgendi.  
“Optimal Signal Quality Index for Photoplethysmogram Signals.”  
*Bioengineering*, 3(4):21, 2016.  
DOI: 10.3390/bioengineering3040021  
PubMed: https://pubmed.ncbi.nlm.nih.gov/28952584/  
Full text: https://pmc.ncbi.nlm.nih.gov/articles/PMC5597264/

[4] W. Karlen, K. Kobayashi, J. M. Ansermino, and G. A. Dumont.  
“Photoplethysmogram signal quality estimation using repeated Gaussian filters and cross-correlation.”  
*Physiological Measurement*, 33(10):1617–1629, 2012.  
DOI: 10.1088/0967-3334/33/10/1617  
PubMed: https://pubmed.ncbi.nlm.nih.gov/22986287/

[5] I. R. Kleckner, R. M. Jones, O. Wilder-Smith, J. B. Wormwood, M. Akcakaya, K. S. Quigley, C. Lord, and M. S. Goodwin.  
“Simple, Transparent, and Flexible Automated Quality Assessment Procedures for Ambulatory Electrodermal Activity Data.”  
*IEEE Transactions on Biomedical Engineering*, 65(7):1460–1467, 2018.  
DOI: 10.1109/TBME.2017.2758643  
PubMed: https://pubmed.ncbi.nlm.nih.gov/28976309/  
Full text: https://pmc.ncbi.nlm.nih.gov/articles/PMC5880745/

[6] S. Taylor, N. Jaques, W. Chen, S. Fedor, A. Sano, and R. W. Picard.  
“Automatic Identification of Artifacts in Electrodermal Activity Data.”  
*37th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC)*, 1934–1937, 2015.  
DOI: 10.1109/EMBC.2015.7318762  
PubMed: https://pubmed.ncbi.nlm.nih.gov/26736662/  
MIT repository: https://dspace.mit.edu/handle/1721.1/103781

[7] W. Chen, N. Jaques, S. Taylor, A. Sano, S. Fedor, and R. W. Picard.  
“Wavelet-Based Motion Artifact Removal for Electrodermal Activity.”  
*37th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC)*, 6223–6226, 2015.  
DOI: 10.1109/EMBC.2015.7319814  
PubMed: https://pubmed.ncbi.nlm.nih.gov/26737714/

[8] W. Boucsein, D. C. Fowles, S. Grimnes, G. Ben-Shakhar, W. T. Roth, M. E. Dawson, and D. L. Filion.  
“Publication recommendations for electrodermal measurements.”  
*Psychophysiology*, 49(8):1017–1034, 2012.  
DOI: 10.1111/j.1469-8986.2012.01384.x  
PubMed: https://pubmed.ncbi.nlm.nih.gov/22680988/

[9] S. Böttcher, S. Vieluf, E. Bruno, et al.  
“Data quality evaluation in wearable monitoring.”  
*Scientific Reports*, 12:21412, 2022.  
DOI: 10.1038/s41598-022-25949-x  
PubMed: https://pubmed.ncbi.nlm.nih.gov/36496546/  
Publisher: https://www.nature.com/articles/s41598-022-25949-x

[10] B. A. MacRae, S. Annaheim, R. Stämpfli, C. M. Spengler, and R. M. Rossi.  
“Validity of contact skin temperature sensors under different environmental conditions with and without fabric coverage: characterisation and correction.”  
*International Journal of Biometeorology*, 62(10):1861–1872, 2018.  
DOI: 10.1007/s00484-018-1589-0  
PubMed: https://pubmed.ncbi.nlm.nih.gov/30062610/

[11] B. A. MacRae, C. M. Spengler, A. Psikuta, R. M. Rossi, and S. Annaheim.  
“A Thermal Skin Model for Comparing Contact Skin Temperature Sensors and Assessing Measurement Errors.”  
*Sensors*, 21(14):4906, 2021.  
DOI: 10.3390/s21144906  
PubMed: https://pubmed.ncbi.nlm.nih.gov/34300649/

[12] X. F. Teng and Y. T. Zhang.  
“The effect of contacting force on photoplethysmographic signals.”  
*Physiological Measurement*, 25(5):1323–1335, 2004.  
DOI: 10.1088/0967-3334/25/5/020  
PubMed: https://pubmed.ncbi.nlm.nih.gov/15535195/

[13] M. Pflugradt and R. Orglmeister.  
“Improved signal quality indication for photoplethysmographic signals incorporating motion artifact detection.”  
*36th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC)*, 1872–1875, 2014.  
DOI: 10.1109/EMBC.2014.6943975  
PubMed: https://pubmed.ncbi.nlm.nih.gov/25570343/

[14] Analog Devices.  
“AD8232: Single-Lead, Heart Rate Monitor Front End — Data Sheet, Rev. D.”  
Official data sheet.  
https://www.analog.com/media/en/technical-documentation/data-sheets/AD8232.pdf

[15] Analog Devices / Maxim Integrated.  
“MAX30101: High-Sensitivity Pulse Oximeter and Heart-Rate Sensor for Wearable Health — Data Sheet, Rev. 3.”  
Official data sheet.  
https://www.analog.com/media/en/technical-documentation/data-sheets/max30101.pdf

[16] Texas Instruments.  
“TMP117 High-Accuracy, Low-Power, Digital Temperature Sensor With SMBus and I2C-Compatible Interface — Data Sheet.”  
Official data sheet.  
https://www.ti.com/lit/ds/symlink/tmp117.pdf

[17] InvenSense / TDK.  
“MPU-6000 and MPU-6050 Register Map and Descriptions, Rev. 4.2.”  
Official register map, 2013.  
https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map.pdf  
TDK product page: https://product.tdk.com/en/search/sensor/mortion-inertial/imu/info?part_no=MPU-6050

[18] C. Auepanwiriyakul, S. Waibel, et al.  
“Accuracy and Acceptability of Wearable Motion Tracking for Inpatient Monitoring Using Smartwatches.”  
*Sensors*, 20(24):7313, 2020.  
DOI: 10.3390/s20247313  
Full text: https://pmc.ncbi.nlm.nih.gov/articles/PMC7766923/

[19] L. Wang, S. Liu, G. Li, and R. Zhu.  
“Interface sensors with skin piezo-thermic transduction enable motion artifact removal for wearable physiological monitoring.”  
*Biosensors and Bioelectronics*, 188:113325, 2021.  
DOI: 10.1016/j.bios.2021.113325  
PubMed: https://pubmed.ncbi.nlm.nih.gov/34030098/

[20] D. Morelli, et al.  
“Profiling the propagation of error from PPG to HRV features in a wearable physiological-monitoring device.”  
*Healthcare Technology Letters*, 5(2):59–64, 2018.  
DOI: 10.1049/htl.2017.0039  
Publisher: https://ietresearch.onlinelibrary.wiley.com/doi/10.1049/htl.2017.0039

[21] Supporting ECG SQI implementation categories and formulas:  
“A pediatric ECG database with disease diagnosis covering 11643 children.”  
*Scientific Data*, 2025.  
Publisher/full text: https://www.nature.com/articles/s41597-025-05225-z  
This reference is used here only for the explicit pSQI/basSQI/bSQI implementation categories; the primary ECG quality-method source remains Clifford et al. [2].

[22] G. B. Papini, P. Fonseca, X. L. Aubert, S. Overeem, J. W. M. Bergmans, and R. Vullings.  
“Photoplethysmography beat detection and pulse morphology quality assessment for signal reliability estimation.”  
*39th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC)*, 117–120, 2017.  
DOI: 10.1109/EMBC.2017.8036776  
PubMed: https://pubmed.ncbi.nlm.nih.gov/29059824/

---

# 25. Bottom-line recommendation

Build the quality layer as an **interpretable staged gate**, not one opaque “SQI model”:

```text
1. acquisition/status vetoes
2. contact evidence
3. modality-specific waveform SQIs
4. supporting physiological plausibility
5. IMU-assisted artifact attribution
6. state + score + reason codes
7. explicit downstream permission
8. hysteresis/recovery
```

Use the most defensible published methods directly:

- **ECG:** AD8232 lead-off + Orphanidou 10-s morphology SQI + dual-detector bSQI + baseline/spectral SQIs.
- **PPG:** MAX30101 FIFO/ALC status + per-channel PI + Orphanidou 0.86 morphology SQI + spectral entropy + conditional IMU evidence.
- **EDA:** Kleckner range/slope/temperature/transition rules as the first-line quality engine; add a trained artifact classifier only after device-specific labeled data exists.
- **Skin temperature:** TMP117 freshness + on-body range + dynamic stability + explicit thermal-settling state; do not overclaim metrological accuracy without environmental-gradient validation.
- **IMU:** verify the IMU itself first, then use synchronized motion as supporting evidence rather than a blanket rejection rule.

The core engineering principle is:

> **No downstream physiological computation receives authority merely because a sensor returned a number. Authority is granted only after the acquisition path, physical coupling, and signal structure provide enough evidence that the measurement is trustworthy for that use.**
