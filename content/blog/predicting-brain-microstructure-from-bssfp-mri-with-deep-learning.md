---
title: "Can We Predict Brain Microstructure Without Diffusion MRI? A Deep Learning Approach Using bSSFP Imaging"
date: 2026-03-15T17:03:00+01:00
summary: "A deep dive into a master's thesis project that uses GANs to generate diffusion tensor data from faster MRI acquisitions — bridging the gap between scan speed and microstructural insight."
---

## Why This Matters

If you've ever had a brain MRI, you might have spent 30 to 60 minutes lying perfectly still inside a loud, narrow tube. For patients with neurological conditions --- stroke, brain tumors, neurodegenerative diseases --- part of that time is often dedicated to **diffusion-weighted imaging (DWI)**, a technique that reveals the microscopic architecture of brain tissue by tracking how water molecules move through it.

DWI is extraordinarily powerful. It can show the intricate highway system of white matter tracts that connect different brain regions, detect the earliest signs of ischemic stroke (often before conventional MRI), and characterize tumors based on their cellular density. The catch? DWI sequences are slow, sensitive to patient motion, prone to geometric distortions, and require specialized gradient hardware that pushes scanners to their limits.

**What if there were a shortcut?** What if we could extract diffusion-like microstructural information from a fundamentally different, faster MRI sequence --- one that doesn't even use diffusion gradients?

This is exactly the question tackled in a master's thesis out of the Max Planck Institute for Biological Cybernetics and the University of Tubingen, titled *"Volume-based Translation of Phase-Cycle Balanced Steady State Free Precession MRI Data to Diffusion Tensors."* The work, authored by Fabian Klopfer (from Villingen-Schwenningen, Germany) under the supervision of Dr. Rahel Heule (Department of High-field Magnetic Resonance, MPI for Biological Cybernetics; Center for MR Research, Children's Hospital University of Zurich) and Prof. Dr. Klaus Scheffler (Department of High-field Magnetic Resonance, MPI for Biological Cybernetics), uses deep learning to predict full 3D diffusion tensors from balanced steady-state free precession (bSSFP) images --- a fundamentally different MRI contrast mechanism.

---

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

## The Dataset: DOVE

The research leverages the **DOVE dataset** (acquired at the Department of High-field Magnetic Resonance, Max Planck Institute for Biological Cybernetics), which is described as one of the largest collections of phase-cycled bSSFP brain images ever acquired. Key specifications:

| Parameter | bSSFP | DWI | T1w (MP2RAGE) |
|-----------|-------|-----|---------------|
| **Subjects** | 120 healthy | 120 healthy | 120 healthy |
| **Sessions** | 3 per subject | 4 per subject | 1 per subject |
| **Resolution** | 1.4 mm isotropic | 2 mm isotropic | 1 mm isotropic |
| **Phase cycles** | 12 (0-360 deg) | - | - |
| **TR / TE** | 4.8 / 2.4 ms | 3500 / 62 ms | 5000 / 2.98 ms |
| **b-value** | - | 1000 s/mm^2 | - |
| **B0 field** | 3T | 3T | 3T |
| **Scanner** | Siemens Prisma | Siemens Prisma | Siemens Prisma |

The dataset was organized following the **BIDS standard** (Brain Imaging Data Structure), converted from DICOM using BIDScoin, and all modalities were coregistered to the T1-weighted images using SPM.

By cross-pairing bSSFP and DWI sessions within each subject (3 bSSFP sessions x 4 DWI sessions = 12 pairs per subject), the authors constructed approximately **1,077 training samples** --- a creative way to multiply the effective dataset size from a limited number of scans.

---

## Preprocessing: From Raw Scans to Neural Network Input

The preprocessing pipeline is surprisingly involved, reflecting the real-world complexity of working with multi-modal MRI data.

### bSSFP Preprocessing
1. **Phase rescaling**: Scanner output [0, 4095] converted to radians [-pi, pi]
2. **Phase correction**: Each phase cycle multiplied by a complex exponential to remove global phase offsets
3. **Gibbs ringing removal**: Using DiPy to suppress truncation artifacts
4. **Resampling**: Complex data resampled to match DWI resolution (2 mm isotropic)
5. **Masking**: Brain mask derived from DWI applied
6. **Normalization**: Magnitude and phase each normalized to [0, 1] across the entire dataset
7. **Channel interleaving**: Magnitude and phase alternated, producing **24-channel** 3D volumes (12 magnitude + 12 phase)

### DWI Preprocessing
1. **Distortion correction**: FSL's TOPUP for susceptibility-induced distortions
2. **Brain extraction**: FSL's BET
3. **Eddy current correction**: FSL's eddy
4. **Tensor estimation**: FSL's DTIFIT to compute the 6 independent tensor elements
5. **Normalization**: Diagonal and off-diagonal elements normalized separately to [0, 1]

### T1-weighted Preprocessing
- Masked, resampled to DWI resolution, normalized to [0, 1], and replicated to 6 channels to match the DWI tensor input dimensionality

---

## The Model: A Conditional GAN with Modality-Specific Heads

### Generator Architecture

The generator follows an encoder-decoder (UNet) design built on MONAI's BasicUNet, adapted for 3D volumetric data with several key modifications:

**Input heads**: Each source modality gets its own ResNet-style input block --- a residual block with 3 convolutional layers, ReLU activations, and instance normalization. This maps the modality-specific channel count (24 for bSSFP, 6 for DWI/T1w) to a standardized 24-channel representation that feeds into the shared backbone.

**UNet backbone**: 5 contracting and 5 expanding blocks with feature maps progressing as 48 -> 96 -> 192 -> 384 -> 768 -> 384 -> 192 -> 96 -> 48 -> 24. Key modifications from the standard UNet:
- **PReLU** activation instead of ReLU, providing learnable negative slopes for more expressive feature extraction
- **Batch normalization** and **dropout** at every block for regularization
- **3D convolutions** throughout, operating on full volumetric data
- **Skip connections** between corresponding encoder and decoder blocks

**Output**: 6 channels representing the upper-triangular elements of the symmetric diffusion tensor (Dxx, Dxy, Dxz, Dyy, Dyz, Dzz).

The full model has approximately **51 million trainable parameters**, requiring roughly 14 GiB of GPU VRAM.

### Discriminator Architecture

A PatchGAN-style discriminator receives the concatenation of input image and tensor (real or generated):
- bSSFP input: 24 + 6 = 30 input channels
- DWI input: 6 + 6 = 12 input channels

Five progressive convolutional layers with LeakyReLU(0.2) and batch normalization downsample the input, producing a spatial map of real/fake predictions rather than a single scalar --- this encourages the generator to produce locally realistic textures.

### Loss Function Design

The total generator loss combines three complementary objectives:

1. **L1 Reconstruction Loss**: Captures low-frequency, voxel-wise accuracy. Simple but essential --- it ensures the predicted tensor values are numerically close to ground truth.

2. **SSIM Loss** (1 - SSIM): Penalizes differences in luminance, contrast, and structural patterns using a sliding-window comparison. Computed over local patches, it captures perceptual similarity that pixel-wise metrics miss.

3. **Perceptual Loss**: This is where things get interesting. Rather than using a VGG network pre-trained on ImageNet (the standard in computer vision), the authors use **MedicalNet** --- a ResNet-10 pre-trained on a large corpus of medical imaging data across different organs, modalities, and pathologies. Both the generated and ground-truth tensors are passed through MedicalNet, and the L2 distance between their deep feature representations becomes a loss term. This captures "medical image-ness" --- structural patterns characteristic of real anatomical data --- that neither L1 nor SSIM can quantify.

4. **Adversarial Loss**: BCEWithLogitsLoss from the PatchGAN discriminator, pushing the generator to produce tensors that are locally indistinguishable from real data.

These losses are weighted and summed, with the perceptual component receiving a 1000x multiplier to ensure it meaningfully influences the gradient.

---

## Training Strategy: Why Naive Training Fails

### Single-Stage Training (Direct)

The first approach was straightforward: initialize the network randomly and train end-to-end on the source-to-DT task. Using AdamW with a learning rate of 1e-4, distributed across 4 NVIDIA GeForce RTX 5000 GPUs via PyTorch Lightning's DDP strategy, with early stopping (patience of 5 epochs).

**The results were sobering.** Looking at the test loss breakdown (Figure 4.1), an interesting pattern emerges: the SSIM loss dominates the total loss across all modalities (~0.03--0.05), while the L1 and perceptual losses are comparatively tiny. The auto-encoding task (DT-to-DT) has the lowest total loss, while pc-bSSFP performs worst. But despite these seemingly reasonable loss values, the predictions contained visible artifacts across all input modalities (clearly shown in Figure 4.2, where red circles highlight checkerboard-like artifacts in brain regions). Quantitatively:
- Diagonal tensor elements (Dxx, Dyy, Dzz): 5--30% relative error --- acceptable
- Off-diagonal elements (Dxy, Dxz, Dyz): up to **3000% relative error** for pc-bSSFP --- catastrophic
- Fractional Anisotropy: **>100% error** across all modalities --- clinically useless
- Mean Diffusivity: 20--50% error --- too imprecise for clinical use

The off-diagonal elements encode the subtle directional information in diffusion --- the fiber crossings, the orientation of tracts. They have smaller absolute values than diagonal elements, making them proportionally harder to predict and more sensitive to relative error.

### Multi-Stage Training (The Breakthrough)

The key insight was that the network needs to first "understand" what a valid diffusion tensor looks like before it can learn to predict one from a foreign modality.

**Stage 1: Autoencoder Pre-training**
Train the full UNet as a DT-to-DT autoencoder. The network takes a real diffusion tensor as input and must reproduce it. This forces the decoder to learn the statistical structure and spatial patterns of valid diffusion tensors --- the correlations between elements, the typical ranges in different brain regions, the spatial smoothness properties.

**Stage 2: Input Head Training**
Swap the DT input head for the target modality's input head (e.g., bSSFP). Freeze all pre-trained weights and train only the new input head. This teaches the input block to map the source modality into the representation space that the pre-trained backbone expects, without disturbing the learned tensor generation capabilities.

**Stage 3: Fine-tuning**
Unfreeze all weights, reduce the learning rate to 1e-5, and fine-tune the entire network end-to-end. The pre-trained weights serve as a strong initialization, allowing the network to make subtle adaptations without catastrophic forgetting of what constitutes a valid diffusion tensor.

Interestingly, the multi-stage test losses (Figure 4.5) are comparable in magnitude to the direct training losses. However, the DT autoencoder now performs dramatically better (loss ~0.005), and pc-bSSFP as input (~0.02) is clearly superior to bSSFP (~0.04) and T1w (~0.007). The key takeaway: similar loss magnitudes can mask profoundly different prediction quality --- the multi-stage approach doesn't just reduce the loss, it changes *what* the network learns to represent.

---

## Results: What the Network Learned

### Quantitative Performance

Multi-stage training produced dramatically better results. Reading the actual error plots from the thesis (Figures 4.3--4.9), we can extract precise numbers:

**Normalized Tensor Element Errors (Figure 4.6):**

| Element Type | DT (autoencoder) | pc-bSSFP | bSSFP | T1w |
|-------------|-----------------|----------|-------|-----|
| Diagonal (CSF/GM/WM) | ~3/3/5% | ~7/8/10% | ~7/8/7% | ~7/7/7% |
| Off-diagonal (CSF/GM/WM) | ~5/5/5% | ~20/20/55% | ~25/35/60% | ~20/25/60% |

**Scalar Map Errors (Figures 4.7--4.9):**

| Scalar | DT | pc-bSSFP | bSSFP | T1w |
|--------|-----|---------|-------|-----|
| Radial Diffusivity | 2--5% | 15--20% | 15--20% | 15--20% |
| Axial Diffusivity | 1--3% | 8--14% | 8--12% | 7--10% |
| Mean Diffusivity | 2--5% | 10--17% | 8--15% | 8--15% |
| Fractional Anisotropy | 8--12% | 15--27% | 15--25% | 10--15% |
| Inclination | <0.05 deg | <0.5 deg | <0.5 deg | <0.5 deg |
| Azimuth | <0.25 deg | <2.0 deg | <1.75 deg | <1.75 deg |

Compare these multi-stage results to single-stage training (Figures 4.3--4.4):
- Off-diagonal elements went from **1000--3000%** (pc-bSSFP, log scale!) down to **20--55%** --- an improvement of 1--2 orders of magnitude
- Fractional Anisotropy went from **250--600%** (all modalities, clinically useless) down to **8--27%** (usable)
- Mean Diffusivity went from **20--50%** down to **2--17%** depending on modality and ROI

The angular accuracy is perhaps the most striking result. The network predicts the **direction of the primary diffusion eigenvector** --- which corresponds to the orientation of white matter fiber bundles --- with sub-degree precision. This means the major white matter tracts (corpus callosum, corticospinal tract, arcuate fasciculus, etc.) are faithfully captured in the predicted tensors.

### Modality Comparison

Four input modalities were tested:
1. **DT (autoencoder)**: Best performance, as expected --- it's reconstructing its own modality
2. **Phase-contrast bSSFP (12 phase cycles)**: Best among non-DT inputs, confirming that the complex bSSFP signal indeed encodes microstructural information
3. **bSSFP (single phase cycle repeated)**: Surprisingly competitive, suggesting that even a single bSSFP contrast carries substantial diffusion-related information
4. **T1-weighted (MP2RAGE)**: Competitive with bSSFP, despite carrying no explicit diffusion sensitivity

That pc-bSSFP outperforms single-phase bSSFP validates the hypothesis that the full frequency response profile --- sampled across 12 phase cycles --- contains richer microstructural information than any single contrast.

### Regional Analysis

Error patterns varied systematically across brain tissue types:
- **White Matter (WM)**: Off-diagonal elements remained hardest to predict, likely because WM has the most complex and heterogeneous fiber architecture
- **Gray Matter (GM)**: Generally lower errors, possibly because diffusion is more isotropic and thus easier to predict
- **Cerebrospinal Fluid (CSF)**: Largest errors for most scalars (except when using DT input), which makes sense --- CSF has very different diffusion properties (high MD, near-zero FA) and the network may have less training signal in these regions

### Qualitative Assessment

The thesis presents side-by-side comparisons (ground truth on left, prediction on right) for the pc-bSSFP input modality. These are among the most compelling figures in the work:

**What works well (Figures 4.10, 4.12, 4.14--4.15):**
- **Diagonal tensor element Dxx** (Figure 4.10): The predicted axial slice is strikingly close to the ground truth. The ventricular system, cortical gray matter boundaries, and white matter intensity patterns are all faithfully reproduced. No artifacts are visible --- a stark contrast to the single-stage results where red-circled artifacts were evident across all modalities (Figure 4.2).
- **Mean Diffusivity** (Figure 4.12): Nearly indistinguishable from ground truth. The characteristic bright CSF in the ventricles, intermediate gray matter values, and darker white matter are all correctly rendered with appropriate contrast and spatial detail.
- **RGB direction maps** (Figures 4.14--4.15): These are the "hero" figures. In both axial and coronal views, the major white matter tracts are clearly visible with correct color coding --- red for left-right fibers (corpus callosum), blue for superior-inferior fibers (corticospinal tract), and green for anterior-posterior fibers. The prediction captures the overall tract geometry, though the ground truth appears slightly crisper and more saturated.

**Where it struggles (Figures 4.11, 4.13):**
- **Off-diagonal element Dyz** (Figure 4.11): The contrast is mostly correct, but the prediction is noticeably smoother than the ground truth. The fine-grained texture visible in the ground truth --- representing subtle directional variations at small spatial scales --- is partially washed out. The high-frequency components of the off-diagonal elements appear hardest for the network to learn.
- **Fractional Anisotropy** (Figure 4.13): The overall structure is preserved, but the predicted FA map is slightly darker and lower-contrast compared to the ground truth. This directly reflects the off-diagonal element limitations, since FA depends on the variance of eigenvalues --- a quantity highly sensitive to errors in the off-diagonal tensor components.

---

## Technical Implementation

### Data Pipeline

The training pipeline uses **TorchIO** for 3D medical image I/O and augmentation:

- **Patch-based training**: 64x64x64 voxel patches uniformly sampled from full volumes
- **Queue-based loading**: TorchIO's Queue system with 16 max queue length, 8 samples per volume, 8 worker processes --- this efficiently manages the memory/compute tradeoff of 3D volumetric data
- **Data augmentation** (training only): Random bias fields (25% probability, modeling B1 inhomogeneities) and Gaussian noise (25% probability, std 0--0.01) --- intentionally minimal to avoid introducing unrealistic artifacts
- **Grid-based inference**: Full-volume predictions assembled from overlapping grid patches

### Training Infrastructure

- **Framework**: PyTorch Lightning 2.2.1 with manual optimization (separate generator/discriminator steps)
- **Hardware**: 4x NVIDIA GeForce RTX 5000 GPUs
- **Strategy**: Distributed Data Parallel (DDP) with unused parameter detection
- **Precision**: 32-bit float (no mixed precision)
- **Monitoring**: Weights & Biases for experiment tracking
- **Checkpointing**: Top-10 checkpoints saved by validation loss

### Evaluation Pipeline

Post-training evaluation is comprehensive and parallelized:

1. **Tensor denormalization**: Inverse min-max scaling using saved normalization parameters
2. **Eigendecomposition**: Full 3x3 tensor eigenanalysis at every voxel using NumPy
3. **Scalar computation**: FA, MD, AD, RD, inclination, azimuth, and RGB direction maps
4. **Error analysis**: Per-voxel relative error (and absolute angular error for directional metrics), stratified by tissue type using probabilistic segmentation maps from FSL
5. **Aggregation**: Median, 1st/25th/75th/99th percentiles, mean, and standard deviation per subject, per ROI

---

## Limitations and Honest Assessment

The thesis is refreshingly candid about the limitations of the approach:

### Statistical Power
Only one model instance was trained per modality. With random weight initialization, different training runs can converge to different solutions. Drawing definitive conclusions about which modality is "best" requires multiple training runs with different random seeds --- a standard practice in deep learning research that was omitted here, likely due to computational constraints (each training run requires significant GPU time).

### Model Complexity
At 51 million parameters and 14 GiB of VRAM, the model is large. The thesis discusses a spectrum of approaches: voxel-wise regression (as in prior work by Birk et al.) at one extreme, and full-volume processing at the other. Patch-based approaches offer a middle ground, with patch size as a tunable knob controlling the trade-off between structural context and computational cost.

### Data Representation
The current approach treats the complex bSSFP data as real-valued multi-channel images, discarding the inherent complex structure. **Complex-valued neural networks** could preserve the mathematical properties of the MRI signal (phase relationships, magnitude-phase correlations) and potentially improve prediction quality.

Similarly, the 12 phase cycles are treated as independent channels rather than as a temporal sequence sampling a frequency response. **Temporal modeling** via 1D convolutions, LSTMs, or vision transformers could explicitly capture the sequential structure of phase-cycle data.

### Segmentation Quality
The probabilistic tissue segmentations used for regional error analysis (from FSL) were acknowledged to be of poor quality. The thesis notes that improved segmentations using FreeSurfer were being generated at the time of writing.

### Perceptual Quality
While the perceptual loss and adversarial training help capture some high-frequency detail, the predictions remain smoother than ground truth. The thesis acknowledges that a PatchGAN discriminator loss (as described in Pix2Pix) might further improve detail but at the cost of more parameters and slower training.

---

## Future Directions

The thesis outlines several promising research avenues:

### Better Generative Models
**Diffusion models** (the deep learning kind, confusingly sharing terminology with diffusion MRI) and **latent diffusion models** have shown stunning results in image generation. Applying these to medical image synthesis could dramatically improve prediction quality, though at significant computational cost. **Flow matching**, using neural ODEs, offers a potentially more efficient alternative.

### Architecture Innovations
- **Complex-valued networks**: Natively handle the complex MRI signal
- **Temporal convolutions or transformers**: Model phase cycles as sequences rather than channels
- **Neural architecture search**: Systematically explore the architecture space rather than relying on manual design

### Multi-Modal Foundation Models
Instead of training separate models for each modality pair, a **foundation model** could learn general MRI-to-MRI translation, then be fine-tuned for specific tasks. Recent work on vision transformers with convolutional components (ViT, Swin Transformer) and parameter-efficient fine-tuning (adapters, LoRA) makes this increasingly feasible.

### Alternative Prediction Targets
Rather than predicting the full diffusion tensor, the network could predict:
- The T2-weighted images used to estimate the tensor (an intermediate representation)
- Scalar maps (FA, MD) directly, as in prior work by Birk et al.
- Higher-order diffusion representations (e.g., orientation distribution functions)

### Broader Context: The Image-to-Image Translation Landscape

The thesis situates this work within the broader landscape of image-to-image translation methods (surveyed extensively in Chapter 2 with a comprehensive taxonomy from Pang et al., 2021). This problem --- translating images from one domain to another --- has spawned a remarkable diversity of approaches: supervised and unsupervised, two-domain and multi-domain, using GANs, VAEs, flow-based models, and diffusion models. Prior MRI-to-MRI synthesis work has largely focused on single-channel modalities (T1 to T2, PD to T1), while this thesis tackles a many-to-many channel problem (24-channel bSSFP to 6-channel DT), which is substantially more challenging both in terms of information content and computational requirements.

---

## The Bigger Picture

This work sits at an exciting intersection of MRI physics, computational neuroscience, and deep learning. The core finding --- that bSSFP imaging carries recoverable diffusion information --- has implications beyond this specific implementation:

**Clinical impact**: If perfected, bSSFP-derived diffusion tensors could provide microstructural information from scans that are already being acquired for other purposes (T2/T1 mapping, functional MRI preparation), effectively making diffusion data available "for free" in terms of scan time.

**Physics insight**: The fact that a neural network can learn this mapping provides indirect evidence that the bSSFP signal encodes microstructural information, complementing the theoretical and experimental work by Miller et al. While the network doesn't provide an analytical formula relating bSSFP to diffusion, its success suggests that such a relationship exists and might be partially derivable.

**Methodological template**: The multi-stage training strategy --- autoencoder pre-training followed by modality-specific fine-tuning --- is a general recipe for cross-modal medical image synthesis that could be applied to many other modality pairs.

The predictions aren't perfect. The off-diagonal elements remain noisy, FA maps lack the crispness of the real thing, and the model hasn't been validated on pathological data. But the directional accuracy is remarkable, the major tracts are clearly resolved, and the multi-stage training breakthrough shows that the approach has room to grow.

For neuroscientists, radiologists, and MRI physicists, this work opens a door: one where the rich microstructural information of diffusion imaging might be accessible from faster, more robust, and more widely available MRI sequences.

---

*This project was developed as a Master of Science thesis (Tubingen, March 2024) at the Graduate Training Centre of Neuroscience, University of Tubingen (Faculty of Science and Faculty of Medicine), in collaboration with the Department of High-field Magnetic Resonance at the Max Planck Institute for Biological Cybernetics. The code is implemented in Python using PyTorch Lightning, MONAI, and TorchIO, and is available as open source.*

---

### Technical Reference

**Key Technologies**: PyTorch 2.2.1, PyTorch Lightning 2.2.1, MONAI 1.3.0, TorchIO 0.19.6, nibabel, PyBIDS, Weights & Biases

**Model**: Modified 3D BasicUNet (MONAI) + PatchGAN discriminator, ~51M parameters

**Training**: AdamW (lr=1e-4), DDP across 4x NVIDIA RTX 5000, early stopping (patience=5), multi-stage: autoencoder pre-train -> input head transfer -> full fine-tune (lr=1e-5)

**Data**: DOVE dataset, 120 subjects, 1077 paired samples, 108 unseen test samples, 64-channel receive head coils

**Evaluation**: Per-voxel relative error stratified by tissue type (CSF/GM/WM), PSNR, SSIM, FID (MedicalNet features), eigendecomposition-derived scalar maps
