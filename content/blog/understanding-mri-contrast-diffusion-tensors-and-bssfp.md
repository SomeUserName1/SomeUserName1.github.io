---
title: "Understanding MRI Contrast: From Diffusion Tensors to Balanced Steady-State Free Precession"
date: 2024-03-01T00:00:00+01:00
tags: ["AI Project Summary"]
summary: "An accessible introduction to two MRI techniques — diffusion-weighted imaging and bSSFP — and why one might encode information about the other."
---

> *This post was AI-generated from the project's source code, thesis, and documentation. It is an automated summary, not original writing.*

## Why This Matters

If you've ever had a brain MRI, you might have spent 30 to 60 minutes lying perfectly still inside a loud, narrow tube. For patients with neurological conditions --- stroke, brain tumors, neurodegenerative diseases --- part of that time is often dedicated to **diffusion-weighted imaging (DWI)**, a technique that reveals the microscopic architecture of brain tissue by tracking how water molecules move through it.

DWI is extraordinarily powerful. It can show the intricate highway system of white matter tracts that connect different brain regions, detect the earliest signs of ischemic stroke (often before conventional MRI), and characterize tumors based on their cellular density. The catch? DWI sequences are slow, sensitive to patient motion, prone to geometric distortions, and require specialized gradient hardware that pushes scanners to their limits.

**What if there were a shortcut?** What if we could extract diffusion-like microstructural information from a fundamentally different, faster MRI sequence --- one that doesn't even use diffusion gradients?

This is exactly the question tackled in a master's thesis out of the Max Planck Institute for Biological Cybernetics and the University of Tubingen, titled *"Volume-based Translation of Phase-Cycle Balanced Steady State Free Precession MRI Data to Diffusion Tensors."* The work, authored by Fabian Klopfer (from Villingen-Schwenningen, Germany) under the supervision of Dr. Rahel Heule (Department of High-field Magnetic Resonance, MPI for Biological Cybernetics; Center for MR Research, Children's Hospital University of Zurich) and Prof. Dr. Klaus Scheffler (Department of High-field Magnetic Resonance, MPI for Biological Cybernetics), uses deep learning to predict full 3D diffusion tensors from balanced steady-state free precession (bSSFP) images --- a fundamentally different MRI contrast mechanism.

## The Physics: Why bSSFP Might Encode Diffusion Information

To appreciate why this project is both ambitious and plausible, we need to understand the two MRI techniques involved.

### Diffusion-Weighted Imaging (DWI)

DWI works by applying strong magnetic field gradients that sensitize the MRI signal to water molecule movement. In the Bloch-Torrey equation --- the fundamental equation governing MRI signal with diffusion --- the diffusion tensor **D** appears as an additional term:

> dM/dt = M x gamma * B_eff - M_xy/T2 - (Mz - Me)/T1 + D * nabla^2(M)

By acquiring images with gradients applied in multiple directions and at different strengths (controlled by the **b-value**, typically 1000 s/mm^2), we can reconstruct the full 3x3 symmetric diffusion tensor at every voxel. This tensor, which has six unique elements (Dxx, Dxy, Dxz, Dyy, Dyz, Dzz), encodes the magnitude and directionality of water diffusion.

From the tensor's eigendecomposition, clinicians derive several scalar maps:
- **Mean Diffusivity (MD)**: Average water mobility, elevated in edema and CSF
- **Fractional Anisotropy (FA)**: How directionally biased diffusion is --- high in organized white matter
- **Axial Diffusivity (AD)**: Diffusion along the primary fiber direction
- **Radial Diffusivity (RD)**: Diffusion perpendicular to fibers
- **RGB direction maps**: Color-coded visualization of primary fiber orientation

### Balanced Steady-State Free Precession (bSSFP)

bSSFP is a completely different beast. It's a gradient echo sequence with extremely short repetition times (TR ~ 4.8 ms in this work) where all gradient moments are fully balanced --- they integrate to zero on every axis within each TR. This creates a steady state where the magnetization oscillates predictably, producing images with T2/T1-weighted contrast.

The key feature of bSSFP is its **frequency response profile**. The signal magnitude depends on the off-resonance dephasing angle, creating characteristic "pass bands" and "stop bands" with signal nulls at certain frequencies. By acquiring images at multiple RF phase increments (called **phase cycles**), you sample different points along this frequency response.

Here's the crucial insight: previous research by Miller et al. (2010) demonstrated that the bSSFP signal frequency profile shows asymmetries that correlate with white matter fiber orientation. The complex bSSFP signal is sensitive to tissue microstructure through its dependence on T1, T2, proton density, and off-resonance effects --- and these properties are themselves influenced by the organization of cellular structures like myelin sheaths around axons.

In other words, bSSFP images may carry a latent "fingerprint" of tissue microstructure --- one that overlaps, at least partially, with the information captured by diffusion imaging. The question is whether a neural network can learn to decode it.

### Prior Work: Birk et al. (2022)

This thesis builds directly on the work of Birk et al., who demonstrated that diffusion tensor *scalar maps* (FA, MD, AD, RD) can be predicted voxel-wise from 12-phase-cycle bSSFP data using a relatively small multi-layer perceptron (MLP). Their approach fed each voxel plus its 9 neighbors into the network, using L2 loss with an uncertainty term, and applied hyperparameter optimization for layer count, neurons per layer, and batch size.

The MLP approach achieved small relative errors but with structural imprecisions --- particularly in and around the ventricles --- because voxel-wise prediction lacks the spatial context that larger receptive fields provide. This thesis takes the opposite extreme: feeding full 3D brain volumes into a UNet-based architecture, trading computational cost for structural coherence. It also predicts the full diffusion tensor (6 elements) rather than just the derived scalar maps, which is a strictly harder but more general task.

---

*Based on the master's thesis "Volume-based Translation of Phase-Cycle Balanced Steady State Free Precession MRI Data to Diffusion Tensors" by Fabian Klopfer, University of Tubingen / Max Planck Institute for Biological Cybernetics, 2024.*
