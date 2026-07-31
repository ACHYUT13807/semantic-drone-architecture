# Perception Pipeline

## Overview

Perception converts a downward-facing colour image into a binary road mask that the planner can treat as free space. Two segmenters coexist: a classical HSV colour threshold used for simulation and early hardware bring-up, and a learned MobileNetV2 U-Net with Gabor texture fusion that is the production network. This document describes both, the failure modes that forced a complete rewrite of the network, the loss function that finally produced usable masks, and the path to TensorRT inference on the Jetson.

## Camera Pipeline

On hardware the Intel RealSense D455 publishes RGB (and depth) via its ROS 2 driver. On simulation the Gazebo mono camera is bridged into the identical `sensor_msgs/Image` topic by `gz_camera_bridge`. Downstream nodes therefore see the same message type regardless of source. Encoding must be consistent (`bgr8` versus `rgb8`); a silent channel swap makes HSV thresholds fail mysteriously.

## HSV Colour Segmentation

HSV was chosen for the first working loop because it is fast, deterministic, needs no GPU, and is trivial to debug. Thresholds were measured on a real nadir frame rather than guessed:

```
LOWER = [100, 0, 50]
UPPER = [170, 50, 180]
```

These bounds bracket the observed road statistics (H ≈ 136–140, S ≈ 21–24, V ≈ 109–135). A diagnostic that logs the percentage of the image labelled as road every 30 frames is the fastest health check for the entire front-end: a sudden drop to near zero indicates either a lighting change or an encoding flip.

## The Original U-Net Failure

The first learned network produced near-empty masks. Three independent faults were stacked:

1. A numerical / normalisation error that collapsed activations.
2. An architectural mismatch with the memory budget of the Orin Nano.
3. A plain cross-entropy loss that was dominated by the overwhelming non-road class.

The result was a network that looked plausible on paper but emitted almost no road pixels in practice. The diagnosis required systematic ablation rather than further hyper-parameter search.

## Replacement Architecture: MobileNetV2 U-Net + Gabor

The replacement keeps the classic U-Net encoder–decoder with skip connections but substitutes a MobileNetV2 backbone pretrained on ImageNet. MobileNetV2 was selected because its inverted residual blocks fit the 8 GB memory ceiling of the Jetson while still providing strong features.

An 8-kernel Gabor bank (4 orientations × 2 frequencies) is fused into the decoder at 128×128 resolution. Texture is a powerful additional cue for asphalt and concrete under varying illumination; the fusion measurably improved boundary precision.

## Loss Function

The decisive change was the switch to a hybrid Dice + cross-entropy loss. Dice directly optimises the intersection-over-union of the road class and therefore counters the extreme class imbalance. Combined with a modest cross-entropy term it produced a validation road IoU of approximately 0.80 on the AeroScapes hold-out set — a qualitative jump from the previous near-empty masks.

## Deployment on the Jetson

Because TensorFlow 2.15 (required by the Jetson ecosystem) cannot load a Keras-3 `.keras` archive, the architecture is rebuilt in code on the device and the weights are loaded from an `.h5` file. This “architecture-plus-weights” pattern is the only reliable way to move the model between the training workstation and the aircraft.

TensorRT conversion remains on the critical path for production inference rates. Until that conversion is complete the network runs under TensorFlow; the measured latency is acceptable for the current 10–15 Hz camera rate but leaves little margin.

## Domain Gap and Fine-Tuning

The network was trained on AeroScapes. Real D455 frames differ in colour balance, resolution and lens distortion. After the 180° camera-rotation fix discovered on the first outdoor flight, a short domain fine-tuning pass on real imagery is required before the next autonomous centreline-tracking flight. That pass, followed by TensorRT conversion, closes the perception track of Prototype 1-α.

## Status

HSV remains the reliable fallback and the segmenter used for most simulation work. The MobileNetV2 + Gabor network is the production path; its validation metrics are solid and the remaining work is domain adaptation and acceleration, not architectural redesign.
