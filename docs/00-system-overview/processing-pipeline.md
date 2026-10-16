# Processing Pipeline

This page is an architecture and navigation summary of the documentation layers. The numbered order expresses conceptual dependencies, not a mandatory runtime sequence.

Multimodal Fusion, Personalized Longitudinal Modeling, and Temporal Event Modeling are cross-cutting reasoning mechanisms. They may interact or be applied at multiple stages in the eventual implementation; the documentation numbering does not require a strict `multimodal → personalization → temporal` runtime order.

## 01 — Signal Trustworthiness

- **Core question:** Is the acquired physiological signal good enough to use?
- **Expected input:** Raw sensor samples, acquisition metadata, timing, and contact or hardware status.
- **Responsibility:** Assess acquisition integrity, sensor coupling, waveform plausibility, and signal-quality indicators.
- **Expected output:** Quality state, confidence, usability, and reasons for rejecting or downgrading each signal.
- **Does not decide:** Whether the physiology is abnormal, what caused it, or what it means clinically.
- **Detailed research:** [01 — Signal Trustworthiness](../01-signal-foundation/01-signal-trustworthiness.md)

## 02 — Physiological Signal Decomposition

- **Core question:** Which signal components represent meaningful physiology versus baseline, drift, or residual noise?
- **Expected input:** Signals that passed trustworthiness checks.
- **Responsibility:** Separate slow/tonic, dynamic/phasic/pulsatile, event or beat, and residual components as appropriate to each modality.
- **Expected output:** Physiological components and derived features with provenance and uncertainty.
- **Does not decide:** Whether a change is caused by a confound, its surrounding context, or its AUD/clinical meaning.
- **Detailed research:** [02 — Physiological Signal Decomposition](../01-signal-foundation/02-physiological-signal-decomposition.md)

## 03 — Confound-Aware Monitoring

- **Core question:** Is a physiological change genuine, or substantially explained by a known confound?
- **Expected input:** Decomposed physiological features plus motion, activity, posture, respiration, temperature, and related confound evidence.
- **Responsibility:** Identify, model, and qualify confound-related changes.
- **Expected output:** Confound-aware physiological changes, explanations, and residual uncertainty.
- **Does not decide:** The patient’s AUD/SUD state, a diagnosis, or a treatment action.
- **Detailed research:** [03 — Confound-Aware Monitoring](../02-physiological-interpretation/03-confound-aware-monitoring.md)

## 04 — Context-Aware Interpretation

- **Core question:** How does the surrounding situation change the interpretation of a real physiological change?
- **Expected input:** Trustworthy, physiologically meaningful, confound-aware changes plus available context and time information.
- **Responsibility:** Compare physiology with context-conditioned expectations and preserve uncertainty when context is missing.
- **Expected output:** Generic context-relative physiological interpretations and context-linked evidence.
- **Does not decide:** A specific psychological or AUD diagnosis, causality, or intervention.
- **Detailed research:** [04 — Context-Aware Interpretation](../02-physiological-interpretation/04-context-aware-interpretation.md)

## 05 — Reliability-Aware Multimodal Physiological Fusion

- **Core question:** How should modalities be combined when they agree, disagree, disappear, or have different reliability?
- **Expected input:** Time-stamped modality features, branch evidence, reliability, confidence, and missingness.
- **Responsibility:** Fuse redundant and complementary physiological evidence while retaining disagreement, uncertainty, and asynchronous timing.
- **Expected output:** Multimodal physiological evidence and state estimates with provenance and confidence.
- **Does not decide:** AUD-specific meaning, diagnosis, or clinical authority.
- **Detailed research:** [05 — Reliability-Aware Multimodal Physiological Fusion](../03-cross-modal-reasoning/05-reliability-aware-multimodal-fusion.md)

## 06 — Personalized Longitudinal Physiological Modeling

- **Core question:** Is the current state abnormal for this patient relative to their own baseline, context, and trajectory?
- **Expected input:** Trustworthy physiological evidence, context, patient history, and population/reference constraints.
- **Responsibility:** Model patient-specific baselines, circadian patterns, recent history, deviations, and longer-term trajectories.
- **Expected output:** Patient-relative residuals, change points, trends, and longitudinal uncertainty.
- **Does not decide:** Whether a deviation has a particular AUD cause or warrants a treatment action.
- **Detailed research:** [06 — Personalized Longitudinal Physiological Modeling](../03-cross-modal-reasoning/06-personalized-longitudinal-modeling.md)

## 07 — Temporal Physiological Event Modeling

- **Core question:** Is a change transient, sustained, recurring, recovering, escalating, or part of a larger trajectory?
- **Expected input:** Time-stamped physiological evidence, multimodal states, patient-relative deviations, and data-availability information.
- **Responsibility:** Detect, confirm, close, associate, and summarize physiological events and trajectories across timescales.
- **Expected output:** Event objects, episodes, recovery patterns, recurrence, burden, and temporal confidence.
- **Does not decide:** A definitive AUD/SUD diagnosis, relapse, withdrawal, or treatment need.
- **Detailed research:** [07 — Temporal Physiological Event Modeling](../03-cross-modal-reasoning/07-temporal-physiological-event-modeling.md)

## 08 — AUD-Specific Physiological Meaning

- **Core question:** What might trustworthy, context-corrected physiological evidence indicate about the patient’s AUD/SUD condition?
- **Expected input:** Upstream physiological evidence, events, patient-relative patterns, context, and relevant AUD/SUD knowledge.
- **Responsibility:** Form conservative AUD/SUD-relevant physiological interpretations such as withdrawal-related progression, cue response, recovery, or treatment-response patterns.
- **Expected output:** AUD/SUD-specific evidence statements or hypotheses with uncertainty and provenance.
- **Does not decide:** A definitive diagnosis, causal attribution from physiology alone, or a prescribed intervention.
- **Detailed research:** [08 — AUD-Specific Physiological Meaning](../04-aud-specific-inference/08-aud-specific-physiological-meaning.md)

## 09 — Evidence/Authority-Aware Clinical Interpretation

- **Core question:** What is the strongest clinician-facing statement the available evidence justifies?
- **Expected input:** The upstream evidence graph, claim-specific support, uncertainty, applicability, and validation or authority information.
- **Responsibility:** Assess evidence strength, specificity, maturity, authority boundaries, abstention conditions, and appropriate clinical wording.
- **Expected output:** Calibrated clinician-facing observations or decision support with explicit limitations.
- **Does not decide:** A stronger diagnostic or treatment claim than the evidence and clinical authority support, or replace clinician judgment.
- **Detailed research:** [09 — Evidence/Authority-Aware Clinical Interpretation](../05-clinical-boundary/09-evidence-authority-aware-clinical-interpretation.md)
