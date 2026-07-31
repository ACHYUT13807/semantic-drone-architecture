# Evaluation and Verification Evidence

## Overview

This document summarises the verification evidence at the close of Prototype 1-α, including the post-α improvements recorded in Part XVI of the progress report and the first outdoor autonomous flight of 16 July 2026.

## Flight Controller and Firmware

- Pixhawk 6C Mini reflashed to PX4 1.17.0; sensors and RC calibrated.
- Flight modes and hardware kill switch assigned.
- Motors driven from the Actuators panel (props off) and subsequently under real flight.

## Companion ↔ Flight Controller

- MAVSDK 2.8.4 connects over serial; telemetry pose stream verified.
- 20 Hz setpoint streamer keeps Offboard setpoints flowing continuously after handoff.
- Arm, takeoff and Offboard mode transitions exercised on the real aircraft.

## Perception

- HSV segmentation usable on simulated imagery (simulation fallback).
- MobileNetV2 U-Net + Gabor fusion reaches val_road_iou ≈ 0.80 on AeroScapes.
- **12-class softmax head** live; road class extracted at index 10 for the binary mask.
- **TensorRT FP16 conversion complete**: end-to-end 31.2 ms (~32 FPS), inference 6.86 ms.
- RealSense D455 publishes live RGB/depth; 180° roll corrected in software after the first flight.

## Planning & Control

- Centreline-guided planner (skeleton + double-BFS + waypoint chain) implemented.
- Costmap protections (largest-road-component, road-protected inflation) in place.
- Pure-Pursuit with corrected lookahead and deceleration law patched post-flight.

## First Outdoor Autonomous Flight (16 July 2026)

- Full v8 pipeline ran on the Jetson under real GPS lock (eph ≈ 0.22 m).
- Two Offboard windows totalled 26 s of accepted companion setpoints; every exit was pilot-commanded; no failsafes fired.
- Oscillation observed, diagnosed from the log as a 180° camera-roll mounting error (positive-feedback limit cycle), and corrected in software together with Pure-Pursuit and deceleration patches.
- The flight demonstrates successful engagement of autonomous Offboard control; it does **not** yet demonstrate successful centreline tracking (that is the purpose of the post-fix flight).

## Items Still Open

- Domain fine-tuning / scoring of the network on real D455 frames.
- Confirmatory centreline-tracking outdoor flight after the camera-rotation and control-law patches.
- USB 3.x port for the RealSense, mount-plate CAD tweak, final harness serial-link check.
- Road-occlusion recovery implementation (research track).

## Interpretation

Prototype 1-α has crossed the boundary from “works in simulation” to “flies outdoors under autonomy with a fully accelerated perception stack.” The residual items are concrete and short; none reopen architectural questions. The evidence register supports both the engineering claim (full pipeline on real hardware at 32 FPS) and the scientific claim (well-diagnosed first-flight failure analysis).
