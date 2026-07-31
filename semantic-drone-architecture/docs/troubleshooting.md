# Troubleshooting Log — Bugs Found and Fixed

## Overview

This document is a condensed version of Appendix D of the Prototype 1-α progress report. It lists the principal bugs encountered during development, their symptoms, root causes, and the fixes that were applied. The list is ordered roughly chronologically and is intended as a quick reference when a familiar symptom reappears.

## Core Package Imports Fail under `ros2 run`

**Symptom.** Modules import cleanly when executed directly but fail under `ros2 run`.  
**Cause.** The `core` sub-package was missing from `setup.py`.  
**Fix.** Ensure `find_packages()` (or an explicit list) includes every Python package that must be installed.

## No Images Reaching the Segmenter

**Symptom.** Segmentation callbacks never fire.  
**Cause.** QoS mismatch (best-effort publisher versus reliable subscriber).  
**Fix.** Explicitly set both ends to `ReliabilityPolicy.RELIABLE`.

## HSV Segmentation “Stops Working” after a Camera Change

**Symptom.** Masks suddenly empty.  
**Cause.** Encoding swap (`bgr8` ↔ `rgb8`) or changed lighting.  
**Fix.** Re-measure thresholds on a real frame; log road-percentage continuously.

## asyncio ↔ ROS “Everything Just Stops”

**Symptom.** Pose stream freezes, callbacks stop, vehicle stops responding.  
**Cause.** A blocking call inside an asyncio coroutine starves the cooperative event loop.  
**Fix.** Async-first pattern with `spin_once` + short `await asyncio.sleep`; wrap every vehicle wait in `asyncio.wait_for`.

## gRPC / `_pose_stream` Crash

**Symptom.** Telemetry stream dies with a gRPC error.  
**Cause.** VM resource starvation collapsing the MAVSDK gRPC channel (not a logic bug).  
**Fix.** Recognise environmental root cause; design reconnect-with-exponential-backoff.

## A* Returns “No Path”

**Symptom.** Planner emits nothing.  
**Cause.** Empty mask, goal on obstacle, or disconnected free space.  
**Fix.** Mask-zero diagnostic, `_nearest_free_goal` spiral, `largest_road_component` filter.

## Goal-on-Obstacle / Edge-Hugging

**Symptom.** Aircraft tracks the shoulder instead of the centreline.  
**Cause.** Auto-goal landing on the road edge; costmap inflation pinching corridors.  
**Fix.** Centreline skeletonisation (v7) + road-protected inflation.

## MAVSDK `connect()` Hangs on the Bench

**Symptom.** Connection never completes.  
**Cause.** Default health-check waits for GPS/home that are absent indoors.  
**Fix.** Remove the health gate for bring-up; rely on PX4 pre-arm and operator.

## RealSense Streams Degraded / Driver GPG Key

**Symptom.** Low frame rate or installation failure.  
**Cause.** USB 2.1 bandwidth limit; missing GPG key for the Intel repository.  
**Fix.** Move to USB 3.x; import the correct key.

## “Fence Requires Position” Pre-arm Blocks Arming

**Symptom.** Cannot arm outdoors until GPS lock.  
**Cause.** Geofence logic requires a valid position estimate.  
**Fix.** Obtain outdoor GPS lock; documented as a remaining physical task.

## U-Net OOM / Keras Version Mismatch / Protobuf Collision

**Symptom.** Allocation failure or model will not load.  
**Cause.** Model larger than available memory; Keras-3 archive under Keras 2.15; conflicting protobuf requirements of TensorFlow and MAVSDK.  
**Fix.** MobileNetV2 encoder; rebuild architecture + load `.h5` weights; pin protobuf == 3.20.3 and MAVSDK 2.8.4.

## 4.5 Hz Camera on Distributed SITL

**Symptom.** Pipeline rate collapses.  
**Cause.** Gazebo software rendering at ~20 % real time.  
**Fix.** Provide GPU resources to the simulation host or limit camera rate.

## Duplicate ROS apt Repository / setuptools Too New

**Symptom.** Ambiguous package resolution or colcon build failures.  
**Cause.** ROS 2 repository configured twice; setuptools 83 incompatible with Humble’s ament_python.  
**Fix.** Single source list; pin setuptools == 79.x.

The full chronological log with chapter references appears in the original progress report. The list above is the practical subset needed for day-to-day debugging.
