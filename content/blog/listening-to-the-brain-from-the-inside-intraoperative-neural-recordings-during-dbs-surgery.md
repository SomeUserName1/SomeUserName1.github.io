---
title: "Listening to the Brain from the Inside: What Happens When You Record Neural Signals During Brain Surgery"
date: 2023-06-01T00:01:00+01:00
tags: ["AI Project Summary"]
summary: "What do brain signals actually sound like when you lower an electrode into a living human brain — and what can they teach us about treating Parkinson's disease?"
---
> *This post was AI-generated from the project's source code, thesis, and documentation. It is an automated summary, not original writing.*

## The Operating Room

The patient is awake. That detail tends to surprise people, but it's essential. Somewhere beneath sterile drapes and a titanium frame that holds the head perfectly still, a person is conscious, sometimes talking, sometimes asked to tap their fingers or extend their wrist. The neurosurgeon is guiding something impossibly thin — a microelectrode, barely wider than a human hair — through a small opening in the skull, aiming for a target deep inside the brain: the subthalamic nucleus, or STN.

The STN is tiny. Almond-shaped, roughly the size of a small lentil, it sits nestled in the base of the brain, part of an ancient circuit that helps orchestrate movement. In Parkinson's disease, this circuit misfires. The neurons in and around the STN fall into abnormal rhythmic patterns — synchronized oscillations that, instead of facilitating movement, actively suppress it. The result is the tremor, the rigidity, the slowness that defines the disease.

Deep brain stimulation, or DBS, is one of the most effective treatments available. A thin electrode is permanently implanted in the STN, delivering precisely calibrated electrical pulses that disrupt those pathological rhythms and restore something closer to normal motor function. For many patients, the effect is dramatic — a tremor that has plagued them for years can vanish within seconds of the stimulator being switched on.

But here's the thing: we still don't fully understand *why* it works so well, or how to make it work better. To answer those questions, you need data — not from a lab bench, not from a computer simulation, but from the living human brain, recorded in the very moment the electrode descends toward its target. That is exactly what happens at the University Clinics Tübingen's Neuromodulation and -technology lab. And what they've built is not just a collection of recordings. It's a dataset, a set of tools, and an invitation to the research community.

## The Descent

Picture the electrode's journey. It enters from the top of the skull and advances downward along a precisely planned trajectory, millimeter by millimeter, toward the STN. At each depth — each stop along the way — the surgical team pauses to record. What they capture is extraordinary in its richness.

The DBS electrode itself has eight contacts, arranged in rings and segments along its tip. Each contact picks up **local field potentials**, or LFPs — the summed electrical activity of thousands of neurons in the immediate vicinity of the electrode. These signals are intimate. They reflect what the brain is doing *right here, right now*, in a volume of tissue smaller than a pea.

But the team isn't just listening from the inside. Simultaneously, **39 EEG electrodes** on the patient's scalp are recording the brain's electrical activity from the surface — the same technology used in sleep studies and epilepsy monitoring, but here captured at the same moment as the depth recordings. Surface and depth. Macro and micro. Two perspectives on the same brain, recorded at the same time.

The numbers are striking. Every channel is sampled at **5,500 times per second** — fast enough to capture oscillations well into the high-gamma range, fast enough to see the fine temporal structure of neural communication. At each depth, multiple trials are recorded. Both hemispheres — left and right — are implanted in many patients. The result is a densely layered dataset: hemisphere by hemisphere, depth by depth, trial by trial, channel by channel.

The directory structure of the dataset mirrors this physical reality. Each patient's data is organized by hemisphere (`Hemis_L`, `Hemis_R`), then by depth (`Depth_01`, `Depth_02`, ...), then by trial. It's not just data storage — it's a map of the electrode's journey through the brain.

## Making Sense of Electrical Chaos

Raw brain signals, straight off the electrode, are a mess. They're beautiful in their complexity, but they're also full of things you don't want: the 50 Hz hum of hospital power lines and its harmonics bleeding into every channel, slow drifts from electrode impedance changes, eye blinks sending electrical storms across the scalp electrodes, muscle artifacts from a patient shifting slightly in discomfort.

Before any science can happen, you have to clean the telescope.

The first pass is **notch filtering** — surgically removing the power-line frequency and its harmonics (50 Hz, 100 Hz, 150 Hz) without disturbing the neural signals that live between them. Then comes bandpass filtering, keeping only the frequency range of interest and discarding everything outside it.

But the subtler artifacts require subtler tools. Eye blinks and eye movements produce large, stereotyped electrical signatures that propagate across the EEG. The dataset's processing pipeline uses **Independent Component Analysis (ICA)** to identify and remove these artifacts — essentially decomposing the mixed signals into independent sources, identifying which sources look like eye movements (using a virtual EOG channel constructed from frontal EEG electrodes), and reconstructing the data without them.

Bad channels — electrodes that have lost contact or are producing noise — are detected, flagged, and interpolated from their neighbors. The continuous recording is then segmented into **epochs**: overlapping three-second windows, stepping forward one second at a time, giving the analysis overlapping snapshots of neural activity. Each epoch is individually assessed for quality using algorithms like the Riemannian Potato method, which detects outlier epochs based on their statistical distance from the rest of the data in a geometric space defined by covariance matrices.

What emerges from this pipeline is clean, validated, structured data — ready to reveal the brain's hidden patterns. All of this is implemented in Python, built on the **MNE** neurophysiology framework, and designed to be reproducible: the same preprocessing steps, applied the same way, across every patient in the dataset.

## The Brain's Hidden Rhythms

Now the real science begins. Take a cleaned LFP signal from an electrode contact sitting inside the STN, and decompose it into its constituent frequencies — a power spectral density, or PSD. What you see is not flat. The brain's electrical activity has structure.

First, there's the **aperiodic background**: a broad, roughly 1/f-shaped decline in power as frequency increases. This isn't noise — it reflects the balance of excitation and inhibition in local neural circuits, and its slope (the "aperiodic exponent") is itself a biomarker that differs between healthy and diseased tissue.

Riding on top of this background are **peaks** — narrowband bumps in the spectrum where the brain is producing rhythmic activity. In the STN of a Parkinson's patient, the most prominent of these is typically in the **beta band** (roughly 13–30 Hz). This exaggerated beta synchronization is one of the hallmarks of Parkinson's pathophysiology. It's what DBS is trying to disrupt.

The dataset's analysis tools use **FOOOF** (Fitting Oscillations and One-Over-f) to disentangle these components — separating the aperiodic background from the periodic peaks, quantifying each peak's frequency, power, and bandwidth. This matters because raw power in a frequency band conflates the two: a change in beta power could mean a change in the oscillation, or a change in the underlying aperiodic slope. FOOOF tells you which.

But the brain's rhythms interact with each other in ways that go beyond simple power. **Phase-amplitude coupling** (PAC) measures whether the phase of a slow oscillation (say, theta at 4–8 Hz) modulates the amplitude of a faster one (say, high gamma at 50–150 Hz). This cross-frequency coupling is thought to reflect hierarchical neural communication — slow rhythms coordinating the timing of fast local processing. The dataset includes six different PAC estimation methods with surrogate-based statistical testing, allowing researchers to probe these interactions rigorously.

And then there are the **complexity measures**: spectral entropy, Lempel-Ziv complexity, singular value decomposition entropy. These capture something different from spectral analysis — they quantify how predictable or random the signal is, how much information it carries, how "complex" the neural dynamics are at a given recording site. A signal that is perfectly rhythmic has low complexity. A signal that is purely random has high complexity. The brain, characteristically, lives somewhere in between — and where exactly it falls on that spectrum changes with depth, with disease state, and with stimulation.

## Bridging Surface and Depth

Perhaps the most compelling aspect of this dataset is that it captures two worlds simultaneously. The EEG on the scalp and the LFP deep in the brain are not independent signals — they're different windows into the same underlying neural activity. The question is: how are they related?

**Spectral connectivity analysis** answers this by quantifying the statistical relationship between pairs of channels across frequencies. Coherence measures whether two signals share the same frequency content at the same time. Phase-locking value (PLV) asks whether the phase relationship between two signals is consistent across trials. Imaginary coherence filters out spurious connections caused by volume conduction — the instantaneous spread of electrical fields through tissue that can make distant electrodes appear falsely correlated.

The dataset's tools compute an extensive suite of these metrics: coherence, imaginary coherence, PLV, phase lag index, weighted phase lag index, pairwise phase consistency. Each captures a slightly different aspect of neural communication, and each has different strengths and vulnerabilities to artifacts.

What emerges is a picture of how the STN talks to the cortex. Beta-band coherence between the STN and motor cortex, for instance, is elevated in Parkinson's disease — the same pathological synchronization seen locally in the STN extends upward to the cortical surface. Visualized as circular connectivity plots, these relationships form intricate networks, with connections strengthening and weakening as the electrode advances through different tissue on its way to the target.

This is where the simultaneous recording becomes irreplaceable. You cannot get this information from EEG alone, or from LFP alone. You need both, recorded at the same moment, in the same brain.

## From One Patient to Many

A single patient's recordings are illuminating. But the real power of a dataset lies in what you can learn across patients — and that's where the hardest problem lives: **every brain is different**.

The STN doesn't sit in exactly the same place in every person. It varies in size, in shape, in its position relative to surrounding structures. An electrode contact that sits at the dorsal border of the STN in one patient might be two millimeters above it in another. If you simply average across patients by depth number ("Depth 5"), you're mixing signals from different anatomical structures. The result would be meaningless.

The dataset solves this with **spatial alignment**. Using anatomical landmarks identified during surgery — the point where the electrode enters the STN and the point where it exits — each patient's depth recordings are mapped onto a standardized coordinate system. The STN is normalized to a standard length (4 mm), and all depths are expressed relative to the STN boundaries.

For even greater anatomical precision, the pipeline integrates with **Lead-DBS**, a widely used software package for electrode localization and trajectory reconstruction. Lead-DBS uses postoperative imaging to determine exactly where each electrode contact ended up in the brain, in standardized stereotactic coordinates. The dataset's `DBSElectrode` class loads these reconstructions and maps them onto the recording data, supporting multiple commercial electrode models from Abbott and Medtronic.

With spatial alignment in place, group-level analysis becomes possible. The tools aggregate features — power spectra, connectivity metrics, entropy measures — across patients, aligned to a common anatomical framework. Cluster-based permutation statistics handle the multiple-comparison problem inherent in testing effects across many channels and frequencies. The result is population-level maps of neural activity relative to the surgical target: how beta power peaks at the dorsal STN border, how connectivity patterns shift as you enter and exit the nucleus, how aperiodic slopes change with depth.

## An Invitation

This dataset is more than an archive of brain recordings. It's a research platform — built in Python, grounded in the MNE neurophysiology ecosystem, designed to be extended. The processing pipeline is modular: swap in a different artifact rejection method, try a new connectivity metric, add a novel complexity measure. The data structures are standardized: MNE Raw and Epochs objects, familiar to anyone who has worked with EEG or MEG data. The analysis tools are there, from preprocessing to group statistics.

What could you do with it? Map the electrophysiological signature of the STN with higher resolution than ever before. Discover biomarkers that predict which patients will respond best to DBS. Develop algorithms for **adaptive stimulation** — devices that listen to the brain's own signals and adjust their output in real time, closing the loop between recording and therapy. Understand, at a fundamental level, how pathological neural circuits organize themselves across spatial scales, from the local field of a few thousand neurons to the global rhythms of the cortex.

The bigger picture is this: DBS works, but it could work much better. Current devices deliver fixed stimulation patterns, indifferent to the brain's moment-to-moment state. The path to smarter, more personalized neuromodulation runs through exactly this kind of data — simultaneous, multi-scale, intraoperative recordings from the human brain, made available with the tools to analyze them.

Back in the operating room, the electrode has reached its target. The neurophysiologist confirms the characteristic STN signal — a dense, rhythmic crackle that experienced teams recognize by ear. The permanent electrode is placed. The stimulator is switched on.

The patient's hand, which has been trembling for years, goes still.

---

**Full documents:** [Report (PDF)](/files/dbs-report.pdf) · [Presentation (PDF)](/files/dbs-presentation.pdf)

*The Intraoperative Neurophysiology Dataset is developed and maintained by the Neuromodulation and -technology lab at the University Clinics Tübingen. The codebase is built on Python and the MNE framework, with tools for preprocessing, spectral analysis, connectivity, complexity measures, and group-level spatial alignment.*
