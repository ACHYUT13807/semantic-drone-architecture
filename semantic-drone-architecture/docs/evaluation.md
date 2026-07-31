# Evaluation and Verification Evidence

## Overview

This document summarises the verification evidence accumulated at Prototype 1-α. It is a condensed form of Appendix H of the original progress report and is intended to answer the question “what has actually been shown to work on real hardware?”

## Flight Controller and Firmware

- Pixhawk 6C Mini reflashed from ArduPilot 4.6.3 to PX4 1.17.0.
- Accelerometer, gyroscope, compass and RC calibrated in QGroundControl.
- Flight modes and hardware kill switch assigned.
- Individual motors driven from the Actuators panel (props off).

## Companion ↔ Flight Controller

- MAVSDK 2.8.4 connects to the real Pixhawk over serial.
- Telemetry pose stream verified.
- Arm and takeoff commands exercised on the bench (and in the first outdoor flight).

## Perception

- HSV segmentation produces usable road masks on both simulated and real imagery.
- MobileNetV2 U-Net + Gabor fusion reaches validation road IoU ≈ 0.80 on AeroScapes.
- RealSense D455 publishes live RGB (and depth) on the Jetson; intrinsics measured.

## Planning

- A* over the semantic costmap produces paths against live frames.
- Centreline-guided planner v7 (skeleton + double-BFS + waypoint chain) implemented and exercised.
- Costmap protections (largest-road-component, road-protected inflation) verified in simulation and against real imagery.

## First Outdoor Autonomous Flight (16 July 2026)

The full pipeline ran under real GPS lock. An oscillation was observed, diagnosed from the flight log as a 180° camera mounting rotation, and corrected together with Pure-Pursuit and deceleration patches. The flight constitutes the primary end-to-end verification that photons arriving at the camera can become motor commands on a real aircraft.

## Items Not Yet Verified

- Final ESC spin-direction mapping on the complete airframe.
- Sustained centreline tracking after the camera-rotation and Pure-Pursuit patches.
- TensorRT inference rates on the fine-tuned network.
- Recovery from deliberate road occlusions (research track).

## Interpretation

Prototype 1-α has crossed the hard boundary from “works in simulation” to “flies outdoors under autonomy.” The residual verification items are short and concrete; none of them reopen architectural questions. The evidence register is therefore sufficient to declare the current configuration a true Prototype 1-α rather than a late-stage simulation exercise.
