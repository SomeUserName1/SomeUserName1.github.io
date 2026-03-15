---
title: "Data-Driven Biomarkers for Motor Subthalamic Nucleus in Parkinson's Disease During DBS Electrode Implantation"
date: 2026-03-15T17:04:00+01:00
summary: "Using machine learning and SHAP to identify electrophysiological biomarkers for precise DBS electrode placement in Parkinson's disease surgery."
---

## The Challenge: Finding the Right Spot in the Brain

Parkinson's disease (PD) affects millions worldwide with progressive motor symptoms -- tremors, rigidity, slowed movement, and impaired postural reflexes. When medication alone is no longer sufficient, Deep Brain Stimulation (DBS) offers a powerful therapeutic alternative. But its success hinges on one critical factor: precise electrode placement within the subthalamic nucleus (STN).

During DBS surgery, electrodes are implanted into the brain to deliver high-frequency electrical stimulation that inhibits pathological activity. The motor portion of the STN is the target -- but distinguishing it from surrounding tissue and non-motor STN subregions during surgery remains a challenge. This project tackles that challenge with a data-driven approach: using machine learning and explainability methods to identify electrophysiological biomarkers that reliably indicate whether a DBS electrode is inside or outside the motor STN.

## Background: Why DBS and Why the STN?

Parkinson's disease arises from the loss of dopamine-producing neurons in the substantia nigra pars compacta. This degradation disrupts the nigrostriatal dopamine pathway, impairing the basal ganglia circuit and ultimately leading to excessive inhibition of voluntary movement.

The basal ganglia circuit operates through three pathways -- direct, indirect, and hyperdirect -- all of which are dysregulated in PD. The result is increased inhibition of the thalamus and decreased cortical activation. While dopaminergic medication (L-DOPA) is the first-line treatment, DBS targeting the STN has become a well-established intervention for advanced cases, effectively modulating the pathological circuit activity.

## The Data Pipeline

This project leverages a rich multimodal dataset collected at the University Hospital Tubingen across 42 patients (5,122 samples, over 42 hours of recording time). The data spans three phases:

- **Pre-operative**: CT, MRI (including DTI), and clinical questionnaires (MDS-UPDRS, PDQ-39, BDI, MoCA)
- **Intra-operative**: EEG recordings (39-channel montage), micro-electrode recordings (MER), and DBS electrode local field potentials (LFP)
- **Post-operative**: Follow-up imaging and repeated clinical assessments

During surgery, the DBS electrode is advanced millimeter by millimeter along a stereotactically planned trajectory. At each position, signals are recorded for 30 seconds and the power spectral density (PSD) is calculated. Electrode positions are then localized in MNI space through coregistration of pre-operative MRI and post-operative CT, enabling ground-truth labeling of each recording as inside or outside the motor STN.

## The Approach: Classification Meets Explainability

Rather than simply building a black-box classifier, this project pairs prediction with interpretation. The core idea is straightforward:

1. **Featurize** the intra-operative signals using normalized PSD values, binned into 5 Hz frequency bands (with aperiodic components removed via FOOOF)
2. **Classify** each recording as motor-STN or non-motor-STN using a Decision Tree classifier from scikit-learn
3. **Explain** the classifier's decisions using SHAP (SHapley Additive exPlanations) values to quantify feature importance

SHAP values, rooted in cooperative game theory, answer a fundamental question: *How much did each feature contribute to the prediction?* By treating the set of frequency-band features as players in a coalition, SHAP assigns each feature a contribution score that is both locally faithful (explains individual predictions) and globally consistent (aggregates into meaningful feature rankings).

## The Software: `ddbm` -- A Reusable Feature Analysis Framework

The project produced `ddbm` (data-driven biomarker), a Python library built around a `FeatureAnalyzer` class that wraps the full analysis workflow. It supports multiple model types (simple models, tree-based, deep learning, and clustering) and provides:

- **SHAP-based visualization**: bar plots, beeswarm plots, heatmaps, waterfall plots, decision plots, force plots, scatter plots, and partial dependence plots
- **Feature selection methods**: forward/backward sequential selection, LASSO-based recursive feature elimination, logistic regression RFE, variance thresholding, and SHAP-based selection
- **Correlation analysis**: feature correlation matrices with heatmap visualization

The framework is designed to be reusable beyond this specific application -- the example scripts demonstrate its use on standard datasets like Iris, as well as regression and clustering tasks.

## Key Results: Beta Band Dominance

The SHAP analysis reveals a clear hierarchy of biomarker importance. The **28-32 Hz frequency band** (high beta) dominates all other features by a wide margin, with a mean absolute SHAP value of 0.16 -- roughly four times the contribution of the next most important feature. This aligns with decades of neuroscience literature identifying beta-band oscillations (13-30 Hz) as a hallmark of STN activity in Parkinson's disease.

The top features by mean SHAP value:

| Frequency Band | Mean |SHAP| |
|---|---|
| 28-32 Hz | 0.16 |
| 68-72 Hz | 0.04 |
| 73-77 Hz | 0.03 |
| 43-47 Hz | 0.03 |
| 78-82 Hz | 0.02 |

The beeswarm plot adds nuance: high power in the 28-32 Hz band strongly pushes predictions toward motor STN classification (positive SHAP values), while low power pushes predictions away -- exactly what would be expected from the known electrophysiology of the parkinsonian STN.

Interestingly, higher gamma-range frequencies (68-82 Hz) also contribute meaningfully, suggesting that biomarker signatures extend beyond the classical beta band.

## Outlook

This work opens several avenues for further investigation:

- **Extended frequency range**: Exploring finer or broader spectral features beyond the current binning
- **Sub-STN classification**: Distinguishing motor, associative, and limbic subregions of the STN
- **Advanced classifiers**: Moving beyond Decision Trees to models capable of learning non-linear feature transformations
- **Multi-label classification**: Simultaneously predicting multiple target regions
- **Feature-guided clustering**: Using the highest-SHAP features to discover patient subgroups or recording phenotypes

## Conclusion

By combining straightforward machine learning with rigorous explainability, this project demonstrates that data-driven methods can extract clinically meaningful biomarkers from intra-operative recordings. The dominance of the beta-band signal validates the approach against established neuroscience, while the identification of additional contributing frequency bands points toward a richer, multi-dimensional biomarker profile for surgical targeting. The open-source `ddbm` framework makes this methodology accessible for other electrophysiological biomarker discovery tasks.

---

*This work was conducted at the Institute for Neuromodulation and -technology, University Hospital Tubingen.*
