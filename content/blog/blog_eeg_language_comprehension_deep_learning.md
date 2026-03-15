---
title: "Decoding Language Comprehension from Brainwaves: A Deep Learning Approach to EEG Coherence Classification"
date: 2026-03-15T17:06:00+01:00
summary: "Can a neural network tell how well you understood a sentence — just by reading your brainwaves? Classifying language comprehension from EEG coherence spectra using deep learning."
---

## The Problem

When we listen to speech or read a sentence, our brain produces measurable electrical activity. Electroencephalography (EEG) captures this activity through electrodes placed on the scalp. One particularly informative signal is **coherence** — a measure of how synchronized two brain regions are at a given frequency. High coherence between certain electrode pairs at specific frequency bands has been linked to successful language processing.

But going from raw coherence spectra to a prediction of *how well someone understood a sentence* is far from straightforward. The data is high-dimensional (many electrodes, many frequency bins, many trials), noisy, and variable across participants. Traditional statistical approaches struggle here. Enter deep learning.

## The Data

The project works with MATLAB `.mat` files containing trial-level EEG coherence data. Each file contains coherence spectra (`cohspctrm`) across electrode pairs and frequencies, along with trial metadata including comprehension scores — numerical ratings reflecting how well a participant understood the presented linguistic stimulus.

The pipeline selects a cluster of six electrodes — **AFZ, C2, C4, CP4, CP6, and F1** — a set spanning frontal, central, and centroparietal regions commonly implicated in language processing. Rather than using the full frequency spectrum, the system isolates narrow frequency bands (e.g., around 5 Hz or 10 Hz, corresponding to theta and alpha rhythms) that are known to play roles in linguistic and cognitive processing.

## The Architecture(s)

What makes this project especially interesting is its multi-pronged modeling strategy. Rather than committing to a single network design, it explores several approaches:

### 1. AutoKeras Neural Architecture Search (NAS)

The primary approach leverages **AutoKeras**, an automated machine learning library that searches for optimal neural network architectures. The `EEGLangComprehension` class extends AutoKeras's `DeepTaskSupervised` to perform classification of comprehension into six discrete categories, while `EEGLangComprehensionNAS` tackles it as a regression problem, predicting comprehension scores on a continuous scale. The system is given a 30-minute time budget to explore architectures — effectively letting the algorithm design its own brain-reading network.

### 2. VGG-based Convolutional Network

A VGG16-inspired model adapted for 1D EEG signals. VGG's deep, uniform convolutional structure is a natural candidate for learning hierarchical features from spectral data.

### 3. Inception-ResNet Hybrid

An adaptation of the Inception-ResNet-V2 architecture, using multiple dense layers with PReLU activations to map learned representations down to the final classification output. This brings the power of residual learning and multi-scale feature extraction to the EEG domain.

All models share a common base class (`AbstractNet`) that handles model persistence, logging, frequency band selection, and preprocessing — including z-score standardization of coherence values.

## Key Design Decisions

- **Frequency-band isolation**: Rather than feeding the full spectrum, the system uses a bisection-based lookup to extract a narrow band (target frequency +/- 1 Hz), focusing the model on the most neurophysiologically relevant signal.
- **Electrode clustering**: By selecting a specific subset of electrodes, the system reduces dimensionality while preserving coverage of language-relevant cortical regions.
- **Classification vs. regression**: The project explores both framings — treating comprehension as categorical (6 classes) and as continuous (regression with MSE loss) — an important design choice when the target variable is an ordinal scale.
- **Automated architecture search**: Using AutoKeras removes human bias from architecture design and allows the system to discover structures that might not be intuitively obvious.

## Why This Matters

This project represents a compelling use case for applied deep learning in cognitive neuroscience. Traditionally, EEG-based language research relies on hand-crafted features and classical statistical tests. By bringing neural architecture search to the table, this work asks: *what if we let the model discover the relevant patterns itself?*

The implications extend beyond academic curiosity. Reliable, automated decoding of language comprehension from EEG could contribute to brain-computer interfaces for non-verbal patients, real-time assessment of cognitive load in educational settings, or clinical evaluation of language processing disorders.

It's a reminder that some of the most exciting applications of deep learning aren't in generating images or chatting with users — they're in reading the electrical whispers of the human brain and making sense of what they say about the mind at work.

---

*Tech stack: Python, PyTorch, Keras, AutoKeras, scikit-learn, SciPy, NumPy*
