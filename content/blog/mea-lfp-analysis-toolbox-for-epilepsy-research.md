---
title: "Building a Toolbox to Decode the Brain's Electrical Whispers"
date: 2023-03-01T00:00:00+01:00
tags: ["AI Project Summary"]
summary: "How a custom Python application is helping neuroscientists study epilepsy and migraine through multi-electrode array recordings."
---
> *This post was AI-generated from the project's source code, thesis, and documentation. It is an automated summary, not original writing.*

### The Problem: Ion Channels Gone Wrong

Mutations in ion channels — the molecular gates that control electrical signaling in neurons — can cause devastating neurological conditions. A single mutation in the voltage-gated sodium channel NaV1.1 (SCN1A) can produce both epileptic seizures and migraine. Understanding how these "channelopathies" alter network-level brain activity is key to developing better treatments.

Researchers study this by placing thin slices of mouse brain onto **multi-electrode arrays (MEAs)** — chips studded with 256 tiny electrodes arranged in a 16x16 grid, each spaced 200 micrometers apart. These electrodes simultaneously record **local field potentials (LFPs)**: the aggregate electrical activity of neural populations. The result is a massive, multi-channel time series that captures how seizure-like activity emerges, propagates, and subsides across the tissue.

The problem? No existing software was up to the task of analyzing it properly.

### The Software Gap

Existing tools each fell short in their own way:

- **FieldTrip** (MATLAB) and **MNE** (Python) target EEG/MEG data at spatial resolutions orders of magnitude coarser than MEAs.
- **Spike-sorting toolboxes** like Brainstorm focus on binary spike events, ignoring the waveform structure that carries crucial information in LFP recordings — such as the large, slow oscillations characteristic of epileptiform bursts.
- **Multi Channel Systems' own proprietary software** applies a single threshold across all 256 electrodes, blind to the heterogeneity of signals across different brain regions on the slice.
- **Xenon's LFP Analysis Platform** targets high-density CMOS arrays with sub-cellular resolution — a different regime entirely — and only reads its own data format.

What was needed was a unified, open, interactive tool that respects the specific spatial scale of 256-electrode MEAs and the unique characteristics of LFP signals.

### The Solution: A Web-Based Analysis Pipeline

The MEA Analysis Toolbox is a Python web application built on **Dash** with Bootstrap styling, giving researchers an interactive browser-based interface to walk through the entire analysis pipeline — from raw data import to publication-ready visualizations — without writing a single line of code.

The workflow follows three stages:

**1. Import and Select.** Raw recordings from MultiChannel Systems H5 files are loaded, and the user is presented with an interactive electrode grid. A microscopy image of the brain slice can be overlaid to see which electrodes sit over which brain regions. Electrodes can be selected individually or via lasso, and a time window of interest can be specified.

**2. Preprocess.** Signals are filtered using Butterworth bandpass/bandstop filters, power-line noise at 50 Hz is removed, and data can be downsampled to speed up downstream computation — critical when a 2-minute recording at 25 kHz produces 3 GiB of uncompressed data across 252 active channels.

**3. Analyze and Visualize.** This is where the toolbox shines. Capabilities include:

- **Spectral analysis** — Power spectral density via Welch's method, spectrograms, and decomposition of signals into periodic and aperiodic components using FOOOF
- **Peak detection** — Two adaptive algorithms that avoid the pitfall of global thresholds by computing per-electrode baselines using the median absolute deviation (MAD)
- **Event/burst detection** — Identification of seizure-like events using moving standard deviation, with automatic merging of adjacent events and filtering by duration
- **Event quantification** — Statistics including duration, spike rate, RMS power, max amplitude, mean inter-spike interval, and power decomposition across frequency bands (delta through high-gamma)
- **Network analysis** — Cross-correlation, mutual information, transfer entropy, Granger causality, coherence, and current source density estimation
- **Channel characterization** — SNR, RMS energy, sample entropy, and peak frequency per electrode
- **Visualization** — Interactive electrode grids, time series grid plots arranged like the physical electrode layout, spectrograms, raster plots with color-coded amplitude, and exportable CSV results

### Clever Algorithms for Tricky Signals

One of the most interesting design decisions is in the peak detection algorithms. In MEA recordings, signal amplitude varies wildly across electrodes depending on the tissue lying over each one. A fixed threshold would miss quiet channels entirely while drowning in false positives on loud ones.

The toolbox implements two complementary approaches:

**Amplitude-based detection** computes the standard deviation from a baseline recording both globally and per-electrode, then uses a weighted combination as an adaptive threshold for SciPy's `find_peaks`.

**Moving-average detection** smooths the absolute signal with a small window (0.1% of the signal length), then applies the same adaptive threshold logic. This favors peaks that occur during periods of sustained activity — exactly the signature of epileptiform bursts — while ignoring isolated spontaneous spikes.

For burst detection, a **moving standard deviation** method exploits the fact that bursting activity increases local signal variance. The algorithm squares the deviation from the mean, computes a moving average (window size ~5% of the signal), and takes the square root. Threshold crossings on both the rising and falling edges delineate burst boundaries. Adjacent events closer than 500 ms are merged, and events shorter than 128 ms are discarded.

### Under the Hood

Performance matters when you're processing millions of data points across hundreds of channels. The toolbox uses several strategies:

- **Numba JIT compilation** with `@nb.jit(parallel=True)` accelerates inner loops for entropy calculations and other compute-heavy operations
- **Shared memory** via Python's `multiprocessing.shared_memory` allows the recording data to be accessed across processes without serialization overhead
- **SciPy's optimized signal processing** handles filtering, FFT, and decimation through battle-tested C/Fortran backends

The architecture follows a clean **Model-View-Controller** pattern: the `Recording` model holds raw data and all computed results in DataFrames; controllers handle import, selection, filtering, and analysis; views render electrode grids (Plotly), time series (PyQtGraph), and static figures (Matplotlib).

### What It Enables

With this toolbox, researchers can answer questions that were previously tedious or impossible to address systematically: *How far does a seizure-like event spread across the slice? How fast? With what magnitude? Which frequency components are involved? How does the network connectivity change between baseline and seizure conditions? And critically — how do these patterns differ between SCN1A mutant and wild-type tissue?*

By providing adaptive, electrode-aware analysis in an accessible GUI, the toolbox lets neuroscientists focus on the biology rather than wrestling with incompatible software or writing one-off scripts.

### Looking Forward

The project continues to evolve. The roadmap includes support for high-density CMOS MEAs, phase-amplitude coupling analysis, parallel processing via worker pools, and the tantalizing possibility of fusing MEA electrophysiology data with 2-photon calcium imaging for truly multi-modal analysis of neural network dynamics.

For researchers studying channelopathies and their devastating neurological consequences, tools like this one are quietly essential — turning the brain's electrical whispers into data that can be heard, understood, and acted upon.

---

*The MEA Analysis Toolbox was developed by Fabian Klopfer during a lab rotation at the Hertie Institute for Clinical Brain Research, University Hospital Tubingen, supervised by Dr. Thomas Wuttke and Dr. Ulrike Hedrich. It is built with Python, Dash, NumPy, SciPy, and a constellation of open-source neuroscience libraries.*
