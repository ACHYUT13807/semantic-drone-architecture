# Perception Pipeline

## Overview

Perception is the foundation of the Semantic Drone navigation framework. Every downstream decision—including graph construction, path planning, and autonomous flight—depends on the quality and consistency of the semantic representation generated from the onboard camera.

Unlike conventional UAV navigation systems that rely on handcrafted computer vision or geometric feature extraction, Semantic Drone performs dense semantic scene understanding to infer traversable road structures directly from monocular RGB imagery.

The perception subsystem is designed not merely to classify pixels, but to produce a navigation-ready representation that remains robust under changing illumination, shadows, partial occlusions, and varying road textures.

---

# Design Objectives

The perception system was developed around five primary objectives:

* Robust semantic understanding of aerial imagery
* Reliable road extraction under real-world conditions
* Real-time inference on embedded hardware
* Generation of planner-friendly traversability maps
* High modularity for future model replacement

Rather than maximizing segmentation accuracy alone, the architecture prioritizes **navigation quality**, recognizing that a visually perfect segmentation may still produce poor navigation behavior if road continuity or topology is not preserved.

---

# Perception Architecture

The perception pipeline transforms raw camera imagery into a refined traversability representation through multiple sequential processing stages.

```text
RGB Camera
      │
      ▼
Frame Acquisition
      │
      ▼
Image Preprocessing
      │
      ▼
Semantic Segmentation Network
      │
      ▼
Raw Road Probability Map
      │
      ▼
Morphological Refinement
      │
      ▼
Connected Component Analysis
      │
      ▼
Occlusion Recovery
      │
      ▼
Road Continuity Enhancement
      │
      ▼
Navigation Costmap
```

Each stage progressively increases the structural quality of the environment representation before it reaches the planning subsystem.

---

# Image Acquisition

The system operates using a forward-facing monocular RGB camera mounted on the UAV.

Compared to stereo cameras or LiDAR systems, a monocular camera offers several practical advantages:

* lower payload weight
* reduced power consumption
* lower computational requirements
* inexpensive hardware
* simpler calibration

The camera continuously streams synchronized RGB frames into the ROS 2 perception pipeline, where each frame is timestamped and processed independently.

---

# Image Preprocessing

Raw camera images undergo a standardized preprocessing stage prior to neural network inference.

Typical preprocessing operations include:

* image resizing
* normalization
* channel standardization
* tensor conversion

Maintaining a consistent input representation reduces distribution shifts between training and deployment while ensuring deterministic inference behavior across simulation and embedded hardware.

---

# Semantic Segmentation Network

At the core of the perception subsystem lies a lightweight encoder-decoder semantic segmentation architecture optimized for edge deployment.

The network combines the representational efficiency of a MobileNetV2 encoder with a U-Net-inspired decoder capable of recovering fine spatial detail through multi-scale feature fusion.

Compared to heavier encoder backbones, MobileNetV2 provides an effective balance between:

* computational efficiency
* inference speed
* parameter count
* deployment feasibility on embedded platforms

The decoder progressively reconstructs high-resolution semantic predictions using skip connections that preserve spatial information lost during feature extraction.

This design enables dense pixel-wise classification while maintaining real-time performance constraints.

---

# Texture-Aware Feature Enhancement

Road surfaces frequently exhibit subtle texture variations that are difficult to distinguish using conventional convolutional features alone.

To improve robustness under varying surface materials and illumination conditions, the perception architecture incorporates a texture-aware feature extraction strategy inspired by Gabor filtering.

Texture-oriented features assist the network in identifying:

* road boundaries
* lane-like structures
* asphalt textures
* concrete surfaces
* low-contrast road regions

Rather than replacing learned convolutional features, these complementary descriptors enrich the semantic representation and improve discrimination between traversable and non-traversable regions.

---

# Embedded Inference Optimization

The perception subsystem is intended for deployment on resource-constrained embedded hardware.

To satisfy real-time execution requirements, the trained segmentation network is deployed using an optimized inference pipeline based on ONNX Runtime, with support for hardware acceleration on NVIDIA Jetson platforms.

Optimization focuses on reducing:

* inference latency
* memory footprint
* CPU utilization
* energy consumption

This deployment strategy enables high-throughput semantic inference while preserving segmentation quality, making the system suitable for onboard autonomous flight.

---

# Semantic Road Representation

The segmentation network produces a dense semantic probability map representing the likelihood of each pixel belonging to a traversable road surface.

Unlike binary thresholding approaches, semantic probabilities retain richer structural information that supports subsequent refinement and topology reconstruction.

The road representation forms the primary interface between perception and planning.

---

# Morphological Refinement

Raw neural network predictions frequently contain small artifacts arising from sensor noise, image compression, or uncertain classifications.

To improve structural consistency, the perception pipeline applies a sequence of morphology-based refinement operations.

The objectives include:

* removing isolated false detections
* filling small discontinuities
* smoothing road boundaries
* improving topological consistency

Morphological processing significantly improves planner stability by reducing fragmented traversable regions.

---

# Connected Component Analysis

Road segmentation occasionally produces multiple disconnected traversable regions.

Some correspond to the actual road network, while others arise from false positives or isolated image artifacts.

Connected component analysis identifies coherent traversable structures and suppresses isolated regions unlikely to contribute to successful navigation.

This filtering stage reduces planning ambiguity and improves graph quality.

---

# Occlusion Recovery

Real-world roads are frequently interrupted by temporary visual occlusions caused by vehicles, trees, pedestrians, or shadows.

If these discontinuities are passed directly to the planner, navigation graphs become fragmented, leading to unnecessary path failures.

The perception pipeline therefore incorporates an occlusion recovery stage that reconstructs interrupted road connectivity using structural reasoning over neighboring traversable regions.

This improves:

* graph continuity
* planner robustness
* navigation smoothness
* mission completion reliability

Importantly, occlusion recovery operates independently of the neural network, allowing improvements in navigation quality without retraining the segmentation model.

---

# Traversability Costmap Generation

The refined semantic representation is converted into a planner-oriented traversability map.

Rather than treating every road pixel equally, the costmap encodes spatial relationships that influence downstream planning behavior.

This abstraction enables the planner to reason about:

* traversable regions
* safe navigation corridors
* obstacle proximity
* connectivity

The resulting representation bridges the gap between computer vision and robotic navigation.

---

# Robustness Considerations

Design decisions throughout the perception pipeline emphasize reliability under challenging operating conditions.

Examples include:

* varying illumination
* changing weather
* shadows
* texture variation
* partial occlusions
* sensor noise
* image compression artifacts

By combining deep semantic understanding with geometric post-processing, the perception subsystem maintains consistent navigation performance across diverse environments.

---

# Computational Pipeline

From a systems perspective, the perception subsystem performs a progressive abstraction of visual information:

```text
Raw RGB Image
      │
      ▼
Normalized Input Tensor
      │
      ▼
Deep Feature Extraction
      │
      ▼
Semantic Probability Map
      │
      ▼
Road Mask Refinement
      │
      ▼
Topology Enhancement
      │
      ▼
Traversability Representation
      │
      ▼
Navigation Costmap
```

Each stage reduces uncertainty while increasing the usefulness of the representation for downstream planning.

---

# Interface with the Planning System

The perception subsystem does not generate flight trajectories directly.

Instead, it exports a refined traversability representation that serves as the input to the navigation planner.

This strict separation of responsibilities offers several advantages:

* modular development
* independent testing
* interchangeable planners
* simpler debugging
* improved software maintainability

Future planning algorithms can therefore be integrated without requiring changes to the perception architecture.

---

# Summary

The Semantic Drone perception pipeline extends beyond conventional semantic segmentation by combining deep visual understanding with topology-aware post-processing and embedded deployment optimization. Through texture-enhanced feature extraction, structural refinement, occlusion recovery, and navigation-oriented costmap generation, the subsystem transforms monocular RGB imagery into a robust representation suitable for autonomous aerial navigation.

Rather than treating perception as an isolated computer vision task, the design views it as the first stage of a complete autonomy stack, ensuring that every output contributes directly to reliable planning and flight execution.
