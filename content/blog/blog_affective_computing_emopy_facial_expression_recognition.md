---
title: "Affective Computing for Subjective Video Quality Assessment: Building EmoPy"
date: 2018-03-29T00:00:00+01:00
tags: ["AI Project Summary"]
summary: "A collaboration between the University of Konstanz and the Technical University of Berlin on using facial expression recognition for video quality assessment."
---
> *This post was AI-generated from the project's source code, thesis, and documentation. It is an automated summary, not original writing.*

## Can a Computer Tell How a Video Makes You Feel?

When you watch a sad scene from *The Lion King*, your face drops, your heart rate shifts, and your skin conductance changes. What if a machine could read all of that in real time? That's the question behind this project -- a university research effort that combines **facial expression recognition (FER)**, **bio-sensor data**, and **deep learning** to objectively measure subjective emotional responses to video content.

The broader field is called **Affective Computing**: teaching machines to recognize, interpret, and respond to human emotions. Its applications range from multimedia quality assessment and social media analytics to clinical tools that aid people with autism in reading emotions. This project focuses on one specific angle -- can we assess video quality not just by pixel metrics, but by how the content actually makes viewers *feel*?

---

## The Experimental Setup

### Video Stimuli with Arousal-Valence Mapping

The study uses a 20-minute experimental session built around the **arousal-valence model** of emotion. Participants watch two 8-minute blocks of video clips, separated by a 3-minute preparation phase and a 1-minute cool-down:

- **Positive valence / high arousal**: clips from *Tarzan*, *Kevin Hart* stand-up, *The Lion King*, *Whose Line Is It Anyway?*
- **Negative valence / high arousal**: clips from *Jackass*, *Cry Freedom*, *The Ring*, *Sinister*, *Walking Death*, *Hooligans*

A key modification to the standard arousal-valence model was extending it with **unipolar valence axes** -- separate positive and negative scales -- rather than a single bipolar dimension. The stimuli were manually labelled for higher arousal to ensure clear emotional activation.

### Bio-Sensors: ECG and EDA

Alongside the camera-based FER system, participants were fitted with:

- **ECG (Electrocardiogram)** -- three channels measuring cardiac activity. The analysis focused on peak detection, with peaks remaining relatively constant across stimuli but showing potential for heart-rate variability analysis.
- **EDA (Electrodermal Activity)** -- two channels measuring skin conductance on the palm. EDA proved particularly effective at capturing mood shifts: the signal evolution was clearly viewable even in small time frames, with distinct patterns emerging between positive and negative valence video blocks.

### Does It Work?

The answer: **sometimes.** The results revealed an interesting complementarity between modalities:

- For emotions like **sadness**, bio-sensor data strongly mirrored the FER predictions -- both channels agreed.
- For **disgust**, the face told the story clearly, but bio-sensors showed no significant reaction.
- For **anger**, neither modality was consistently reliable -- bio-sensors often stayed flat even when subjects visually appeared engaged.

This inconsistency raises a compelling question the team posed for future work: *Can the bio-sensor and FER approaches benefit from each other?*

---

## EmoPy: The Facial Expression Recognition Engine

The software backbone of this project is **EmoPy** -- an emotion recognition system built with Dlib, Keras, and TensorFlow. It classifies facial expressions into **7 emotions**: neutral, sad, happy, angry, disgust, surprise, and fear.

### Architecture Overview

EmoPy is designed as a modular pipeline with pluggable neural network architectures and preprocessing stages.

**Data Collection** supports two datasets:
- **CK/CK+** (Cohn-Kanade) -- 327 directly labelled images with FACS (Facial Action Coding System) data
- **FER2013** (Kaggle) -- a larger but noisier dataset of 48x48 grayscale facial images

**Preprocessing** is handled by a hierarchy of preprocessor classes, each tailored to a specific model type:

| Preprocessor | Input Type | Description |
|---|---|---|
| `ImagePreprocessor` | Raw images | Grayscale face images resized to input dimensions |
| `DlibPreprocessor` | Dlib landmarks | Extracts 68 facial landmark points, distances from centroid, and vector angles |
| `MultiPreprocessor` | Both | Combines image pixels with Dlib geometric features |
| `CapsPreprocessor` | Raw images | Specialized preprocessing for Capsule Networks |

### The Neural Network Zoo

The project explored a striking variety of architectures -- "maybe too many," as the team candidly noted in their presentation. Here's what they built:

#### 1. Image Input CNN (`imagenn`)

A straightforward convolutional neural network processing raw grayscale face images:

```
Input (48x48x1)
  -> BatchNorm -> Conv2D(32, 3x3) -> PReLU -> BatchNorm -> MaxPool(2x2)
  -> Conv2D(32, 1x1) -> PReLU -> BatchNorm
  -> Conv2D(64, 3x3) -> PReLU -> BatchNorm
  -> Conv2D(128, 3x3) -> PReLU -> BatchNorm -> MaxPool(2x2)
  -> Conv2D(256, 3x3) -> PReLU -> BatchNorm
  -> Flatten -> Dense(256) -> Dense(1024) -> PReLU
  -> Dense(2048) -> PReLU -> Dropout(0.25)
  -> Dense(512) -> PReLU -> Dense(7, softmax)
```

Uses PReLU activations throughout and Glorot normal initialization -- a solid baseline architecture.

#### 2. Dlib Landmark CNN (`dlibnn`)

Instead of raw pixels, this network operates on **geometric facial features** extracted by Dlib's 68-point face landmark detector. It processes three parallel input streams:

- **Landmark coordinates** (68 x 2): raw (x, y) positions of facial keypoints
- **Distances from centroid** (68 x 1): how far each landmark sits from the face center
- **Angles relative to centroid** (68 x 1): the angular position of each landmark

Each stream passes through its own convolutional tower, then all three are concatenated and fed through dense layers. This is a clever approach -- it encodes facial geometry explicitly, making the model invariant to lighting and texture.

#### 3. Multi-Input CNN (`minn`)

The most sophisticated CNN architecture, combining both raw image features and Dlib geometric features in a single model:

```
Image branch:  Conv2D stack (32->64->128->256) with strides -> Flatten -> 1024 features
Dlib branch:   Flatten all 3 streams -> Concatenate -> Dense(256) features
                        |
                Concatenate (1024 + 256 = 1280)
                        |
                Dropout(0.2) -> Dense(1024) -> Dense(7, softmax)
```

This architecture includes TensorBoard logging, model checkpointing, and learning rate decay -- the most production-ready of the set.

#### 4. Capsule Network (`caps`)

An implementation of **Capsule Networks** (Sabour et al., 2017) for emotion recognition -- a particularly forward-thinking choice for 2018. The architecture features:

- A standard conv layer followed by **PrimaryCaps** (8-dimensional capsules)
- A **CapsuleLayer** with dynamic routing producing one 16-dimensional capsule per emotion class
- A decoder network (Dense 512 -> 1024 -> reconstruct input) for regularization
- Custom **margin loss** with the classic CapsNet formulation: `m+ = 0.9`, `m- = 0.1`

#### 5. Additional Architectures

The codebase also includes **Inception-ResNet** and **VGGFace** transfer learning models, as well as commented-out **LSTM** variants for sequential/video-based emotion recognition -- evidence of ambitious experimentation.

### Results

The best model achieved **~60% accuracy** on the FER2013 dataset's 7-class problem. While this may sound modest, it's worth noting that FER2013 is notoriously noisy -- even human annotators achieve only about 65% agreement on this dataset. The team experimented extensively with architectural choices:

- Replacing max pooling with strided convolutions **slowed learning and increased overfitting**
- Adding dropout layers **increased training time** but helped generalization
- Large learning rates combined with dropout after every conv layer **hindered learning** entirely

### The Runner System

EmoPy is orchestrated through a clean runner script (`runners.py`) that supports three modes via command-line arguments:

- `init_data` -- preprocesses and structures the CK and FER2013 datasets
- `train` -- trains the selected architecture with configurable hyperparameters
- `predict` -- runs inference on images, video files, or live webcam feeds
- `visualize` -- generates neuron activation visualizations inspired by the Deep Visualization Toolbox

---

## Integration: FER Meets ROS

For real-time deployment, the FER model was integrated into a **ROS (Robot Operating System) people model**. The pipeline chains together:

1. **USB camera input** -> raw video frames
2. **Dlib face detection and shape prediction** -> facial landmarks
3. **People model** -> tracking individuals with distance, position, and attributes
4. **Emotion classification** -> real-time per-face emotion labels
5. **Debug visualization** -> overlay of predictions on the video feed

This ROS integration allowed the FER system to run alongside the bio-sensor data collection during the video stimuli experiments, providing synchronized multi-modal emotion readings.

---

## Challenges and Lessons Learned

The team identified several fundamental challenges in affective computing:

1. **Capturing "true" emotions** -- lab settings induce self-consciousness; posed vs. spontaneous expressions differ significantly.
2. **Data quality trade-offs** -- there's an inherent tension between data clarity (controlled conditions) and ecological validity (natural behavior).
3. **Aesthetics and emotion are entangled** -- visual quality of the stimulus itself influences emotional response, making it difficult to isolate content-driven affect.
4. **Bio-sensor/FER disagreement** -- different modalities capture different aspects of emotion, and they don't always align.

---

## Looking Forward

The project outlined several directions for future work:

- **Standardizing affective terminology** -- the field needs clearer, shared definitions for each research aim
- **Crowdsourcing and wearables** -- scaling data collection beyond the lab using consumer devices
- **Stronger machine learning integration** -- leveraging advances in deep learning to fuse multiple modalities more effectively
- **FACS-to-continuous-dimension mapping** -- converting Facial Action Coding System data into continuous arousal-valence dimensions, as proposed by Egon van den Broek's work on Affective Signal Processing

---

## Technical Stack

| Component | Technology |
|---|---|
| Face detection & landmarks | Dlib (68-point predictor) |
| Deep learning framework | Keras + TensorFlow |
| Transfer learning | VGGFace, Inception-ResNet |
| Computer vision | OpenCV |
| Real-time integration | ROS (Robot Operating System) |
| Bio-sensors | ECG (3-channel), EDA (2-channel) |
| Camera | GoPro (via goprocam) + USB webcam |
| Data analysis | NumPy, SciPy, scikit-learn, Pandas |

---

*This project was presented on March 29, 2018, by Jan Laun-Cieslik and Fabian Klopfer (5th Semester, BSc Computer Science) as part of a collaboration between the University of Konstanz and TU Berlin. Contributors: Nida Cilasun, Mengzhu Deng, Franz Hahn, Yazan Kittaneh, Fabian Klopfer, Anlin Liang, Zhou Long, Jan Laun-Cieslik, Daniel Matzkuhn, and Andrei Sacuiu.*
