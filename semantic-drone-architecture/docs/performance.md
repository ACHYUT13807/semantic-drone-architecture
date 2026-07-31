# Performance, Rates and Distributed SITL

## Overview

This document records the measured rates of the pipeline, the completed TensorRT FP16 acceleration, the diagnosis of the earlier 4.5 Hz bottleneck in distributed simulation, and the current camera-limited operating point.

## TensorRT FP16 — The 10× Step

Prior to conversion the v7 perception path ran under TensorFlow eager execution at approximately 309 ms end-to-end (~3.2 Hz). After TensorRT FP16 conversion of the MobileNetV2 + Gabor U-Net the v8 numbers are:

| Stage                        | v7 (TF eager)     | v8 (TensorRT FP16)      |
|------------------------------|-------------------|-------------------------|
| Segmentation inference       | ≈ 266 ms          | **6.86 ms**             |
| Gabor preprocessing (CPU)    | ≈ 8 ms            | 19.6 ms (now dominant)  |
| Cost-map + skeleton + goal   | ≈ 35 ms           | ≈ 4 ms                  |
| **End-to-end**               | ~309 ms (3.2 Hz)  | **31.2 ms (≈32 FPS)**   |

Inference improved by nearly two orders of magnitude. The bottleneck moved from the GPU network to the CPU-side eight-kernel Gabor bank. At 32 FPS the perception loop is faster than the RealSense colour stream (30 fps) and is therefore camera-limited rather than compute-limited.

This acceleration was a prerequisite for the first outdoor autonomous flight. At 3.2 Hz the 300 ms pipeline latency corresponded to roughly 37 cm of position uncertainty at 1.5 m/s; at 32 FPS perception latency is comparable to a pilot’s reaction time and ceases to be the limiting factor.

## Nominal Operating Point

- Perception loop: ~32 FPS (camera-limited).
- Control setpoint streamer: 20 Hz (dedicated asyncio task, independent of perception).
- Pure-Pursuit and waypoint advance run on the perception cadence or faster.
- The 20 Hz streamer continues to emit the last valid (or zero) command even if the perception or planning node stalls, preventing PX4 Offboard timeouts.

## Distributed SITL Topology

For algorithm development the Gazebo world and PX4 SITL run on a separate simulation host while the real ROS 2 pipeline runs on the Jetson. The two machines are joined by ROS 2 DDS discovery (`ROS_DOMAIN_ID` shared, `ROS_LOCALHOST_ONLY=0`) and MAVLink over UDP.

An early 4.5 Hz collapse in this configuration was isolated past the bridge, network, USB and protobuf layers to Gazebo’s software rendering on the VM (~20 % real time). Once the simulation host received adequate GPU resources the pipeline recovered.

## HITL and Timing

Hardware-in-the-loop that places the real Pixhawk in the loop with a simulated world requires low-latency deterministic timing that the present virtual-machine environment cannot guarantee. The project therefore uses pure SITL for algorithm work and moves directly to the real aircraft for final validation.

## Remaining Headroom

Further inference speed-ups (e.g. INT8) have limited effect until Gabor preprocessing is also accelerated or moved to the GPU. Domain fine-tuning of the network on real D455 frames remains the open correctness task; it does not affect the measured rates above.
