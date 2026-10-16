# Physiological Monitoring Documentation

This documentation set describes the research and design layers for an AUD/SUD physiological monitoring system, from trustworthy sensing through clinician-facing interpretation.

## Conceptual pipeline

```text
Body
  ↓
Sensing / Acquisition
  ↓
Signal Trustworthiness
  ↓
Physiological Signal Decomposition
  ↓
Confound-Aware Monitoring
  ↓
Context-Aware Interpretation
  ↓
┌───────────────────────────────────────────┐
│ Cross-Modal / Cross-Time Reasoning        │
│                                           │
│ • Multimodal Fusion                       │
│ • Personalized Longitudinal Modeling      │
│ • Temporal Event Modeling                 │
└───────────────────────────────────────────┘
  ↓
AUD-Specific Physiological Meaning
  ↓
Evidence / Authority-Aware Clinical Interpretation
  ↓
Clinician-Facing Observation / Decision Support
```

## Documentation layers

- [Processing Pipeline](00-system-overview/processing-pipeline.md)
- [01 — Signal Trustworthiness](01-signal-foundation/01-signal-trustworthiness.md)
- [02 — Physiological Signal Decomposition](01-signal-foundation/02-physiological-signal-decomposition.md)
- [03 — Confound-Aware Monitoring](02-physiological-interpretation/03-confound-aware-monitoring.md)
- [04 — Context-Aware Interpretation](02-physiological-interpretation/04-context-aware-interpretation.md)
- [05 — Reliability-Aware Multimodal Physiological Fusion](03-cross-modal-reasoning/05-reliability-aware-multimodal-fusion.md)
- [06 — Personalized Longitudinal Physiological Modeling](03-cross-modal-reasoning/06-personalized-longitudinal-modeling.md)
- [07 — Temporal Physiological Event Modeling](03-cross-modal-reasoning/07-temporal-physiological-event-modeling.md)
- [08 — AUD-Specific Physiological Meaning](04-aud-specific-inference/08-aud-specific-physiological-meaning.md)
- [09 — Evidence/Authority-Aware Clinical Interpretation](05-clinical-boundary/09-evidence-authority-aware-clinical-interpretation.md)

## Directory guide

- `00-system-overview/` — architecture and navigation.
- `01-signal-foundation/` — signal quality and physiological signal components.
- `02-physiological-interpretation/` — confound-aware and context-aware interpretation.
- `03-cross-modal-reasoning/` — reasoning across modalities and time.
- `04-aud-specific-inference/` — AUD/SUD-specific physiological meaning.
- `05-clinical-boundary/` — evidence strength, authority, uncertainty, and clinical wording.

The numbering represents documentation and conceptual dependency order, not necessarily a strict runtime execution order.
