# Perception Pipeline

## Overview

Perception converts a downward-facing colour image into a semantic mask that the planner can treat as free space. The production path is a MobileNetV2 U-Net with Gabor texture fusion, converted to a TensorRT FP16 engine. A classical HSV colour threshold remains available as the simulation-only fallback. This document describes both segmenters, the failure modes that forced the network rewrite, the loss function and architecture that produced usable masks, the completed TensorRT conversion, and the remaining domain-adaptation work.

## Camera Pipeline

On hardware the Intel RealSense D455 publishes RGB (and depth) via its ROS 2 driver. On simulation the Gazebo mono camera is bridged into the identical `sensor_msgs/Image` topic by `gz_camera_bridge`. Downstream nodes therefore see the same message type regardless of source. Encoding must be consistent (`bgr8` versus `rgb8`); a silent channel swap makes HSV thresholds fail mysteriously.

After the first outdoor flight a 180° camera-roll mounting error was diagnosed. The live camera node now applies `cv2.rotate(frame, cv2.ROTATE_180)` to both colour and depth streams and recomputes the principal-point intrinsics accordingly, preserving RGB-depth registration.

## HSV Colour Segmentation (Simulation Fallback)

HSV was chosen for the first working loop because it is fast, deterministic, needs no GPU, and is trivial to debug. Thresholds were measured on a real nadir frame rather than guessed:

```
LOWER = [100, 0, 50]
UPPER = [170, 50, 180]
```

These bounds bracket the observed road statistics. A diagnostic that logs the percentage of the image labelled as road every 30 frames is the fastest health check for the entire front-end. On real hardware the learned network is selected by default (`make_segmenter(mode="real")`); HSV is retained only for simulation (`mode="sim"`).

## The Original U-Net Failure

The first learned network produced near-empty masks. Three independent faults were stacked:

1. A numerical / normalisation error that collapsed activations.
2. An architectural mismatch with the memory budget of the Orin Nano.
3. A plain cross-entropy loss that was dominated by the overwhelming non-road class.

The result was a network that looked plausible on paper but emitted almost no road pixels in practice.

## Replacement Architecture: MobileNetV2 U-Net + Gabor

The replacement keeps the classic U-Net encoder–decoder with skip connections but substitutes a MobileNetV2 backbone pretrained on ImageNet. MobileNetV2 was selected because its inverted residual blocks fit the 8 GB memory ceiling of the Jetson while still providing strong features.

An 8-kernel Gabor bank (4 orientations × 2 frequencies) is fused into the decoder at 128×128 resolution. Texture is a powerful additional cue for asphalt and concrete under varying illumination.

The network emits a **12-class softmax head**. The road class is read at index 10; the remaining classes provide the multi-class supervision that improves feature quality even when only the binary road/not-road decision is used downstream.

## Loss Function

The decisive change was the switch to a hybrid Dice + cross-entropy loss. Dice directly optimises the intersection-over-union of the road class and therefore counters extreme class imbalance. Combined with a modest cross-entropy term it produced a validation road IoU of approximately 0.80 on the AeroScapes hold-out set.

## TensorRT FP16 Conversion — Completed

TensorRT conversion of the MobileNetV2 + Gabor network is complete (Part XVI §62). The trained Keras weights are compiled into an optimised, fused, FP16 engine specific to the Orin Nano GPU.

| Stage                        | v7 (TF eager)     | v8 (TensorRT FP16)      |
|------------------------------|-------------------|-------------------------|
| Segmentation inference       | ≈ 266 ms          | **6.86 ms**             |
| Gabor preprocessing (CPU)    | ≈ 8 ms            | 19.6 ms (now dominant)  |
| Cost-map + skeleton + goal   | ≈ 35 ms           | ≈ 4 ms                  |
| **End-to-end**               | ~309 ms (3.2 Hz)  | **31.2 ms (≈32 FPS)**   |

This is a 10× end-to-end improvement and the single largest performance step of the project. Inference dropped by nearly two orders of magnitude; the bottleneck moved from the network to the CPU-side Gabor bank. At 32 FPS the pipeline is faster than the RealSense 30 fps colour stream and is therefore camera-limited rather than compute-limited.

Nothing about the network’s output changes under TensorRT: the same 12-class softmax head is produced and validation IoU on AeroScapes stays within ~0.5 % of the unquantised model. FP16 was chosen deliberately for near-zero accuracy loss with minimal calibration effort; INT8 remains a possible future step if another factor of two is required.

Because TensorFlow 2.15 on the Jetson cannot load a Keras-3 `.keras` archive, the architecture is rebuilt in code on the device and the weights are loaded from an `.h5` file (or the TensorRT engine is loaded directly). This “architecture-plus-weights / engine” pattern is the reliable deployment path.

## Domain Gap and Remaining Work

The network was trained on AeroScapes. Real D455 frames differ in colour balance, resolution and lens distortion. Domain fine-tuning / validation on real nadir imagery is still pending; it is the correctness half of deploying the learned segmenter on out-of-distribution data. After the 180° camera-rotation fix, collecting and labelling real frames is the next perception action.

## Status at the Close of Prototype 1-α

- HSV remains the reliable simulation fallback.
- The MobileNetV2 + Gabor network is the production segmenter on real hardware.
- TensorRT FP16 conversion is complete and was a prerequisite for the first outdoor autonomous flight.
- The 12-class head is live; road class is extracted at index 10 for the binary mask used by the planner.
- Domain fine-tuning on real D455 frames is the sole open perception item.
