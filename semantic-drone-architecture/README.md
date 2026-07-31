# Semantic Drone Architecture

**Autonomous Semantic Drone Road-Following System**  
Prototype 1-α Progress Documentation Repository

This repository contains the complete technical documentation for the Autonomous Semantic Drone project — a quadrotor that performs real-time semantic road following using only onboard vision, without pre-loaded maps or GPS waypoint lists. The system classifies every pixel of a downward-facing camera image into one of 12 semantic classes (road extracted at index 10), builds an occupancy grid, plans a path with a centreline-guided A* chain, and flies the resulting path via offboard control on PX4.

The documentation is organised to mirror the physical and software architecture of the aircraft. It is intended both as a durable record of the working configuration at the close of Prototype 1-α (including the first outdoor autonomous flight of 16 July 2026) and as the companion reference for the project website.

## Project Snapshot

| Component | Technology |
|-----------|------------|
| Flight controller | Pixhawk 6C Mini running PX4 1.17.0 |
| Companion computer | NVIDIA Jetson Orin Nano 8GB (JetPack 6.x / Ubuntu 22.04) |
| Camera | Intel RealSense D455 (RGB + depth) |
| Middleware | ROS 2 Humble |
| Flight interface | MAVSDK-Python 2.8.4 over MAVLink |
| Segmentation | MobileNetV2 U-Net + Gabor texture fusion → **TensorRT FP16** (12-class head, road @ index 10) |
| Classical fallback | HSV colour thresholding (simulation only) |
| Planning | A* over semantic costmap with centreline skeletonisation (v7/v8) |
| Perception rate | **~32 FPS** (31.2 ms end-to-end after TensorRT) |
| Simulation | Gazebo + PX4 SITL (distributed topology supported) |
| Airframe | Quad-X carbon fibre, 6S 12000 mAh |

## Key Results at the Close of Prototype 1-α

- TensorRT FP16 conversion completed: 10× speedup (3.2 Hz → 32 FPS; inference 6.86 ms).
- First outdoor autonomous Offboard flight performed 16 July 2026 (26 s of accepted setpoints across two windows).
- Oscillation diagnosed from the flight log as a 180° camera-roll mounting error; corrective patches (rotation, intrinsics, Pure-Pursuit lookahead, deceleration law) are in the codebase.
- Many earlier “remaining” items (actuator mapping, outdoor GPS lock, TensorRT, texture fusion) were resolved before or by the flight.
- Domain fine-tuning of the network on real D455 frames and a confirmatory post-fix centreline-tracking flight remain open.

## Repository Layout

```
.
├── README.md                 # This file
├── LICENSE
├── docs/
│   ├── architecture.md       # System architecture & data flow
│   ├── ros2.md               # ROS 2 Humble usage, nodes, asyncio integration
│   ├── hardware.md           # Pixhawk, Jetson, RealSense, power, mounts
│   ├── perception.md         # Camera, HSV, U-Net rewrite, TensorRT FP16, 12-class
│   ├── planning.md           # Occupancy grids, A*, auto-goal, centreline v7
│   ├── control.md            # Control node, Pure Pursuit, 20 Hz streamer, offboard
│   ├── safety.md             # Pilot gate, layered safety model, failsafes
│   ├── performance.md        # TensorRT rates, SITL diagnosis, bottlenecks
│   ├── troubleshooting.md    # Bug log and fixes
│   ├── deployment.md         # Reproducibility guide for a fresh Jetson
│   ├── roadmap.md            # Remaining tasks & dependency graph
│   ├── research.md           # Road-occlusion recovery theory
│   └── evaluation.md         # Verification evidence & metrics
├── assets/
└── diagrams/
```

## One-Sentence Description

The aircraft looks straight down at a road, decides in real time where the drivable surface continues, and flies itself along that surface using only what its nadir camera sees — frame by frame.

## Architectural Centrepiece: The Two-Computer Split

Everything flight-critical lives on the Pixhawk. Perception, planning, and high-level decision-making live on the Jetson. The two communicate exclusively over MAVLink. The flight controller never depends on the perception stack; if the network fails, the pilot or PX4 failsafes remain fully capable.

## How to Read This Documentation

Start with `docs/architecture.md` for the system view, then `docs/ros2.md` for the middleware. Hardware bring-up is in `docs/hardware.md`. The perception path (including the completed TensorRT conversion and 12-class head) is detailed in `docs/perception.md`. Planning, control, safety, performance numbers, the full troubleshooting log, and the deployment recipe complete the set.

All figures referenced in the original progress report are original.

## Repository & Attribution

- Original repository: ACHYUT13807/semantic-drone (private)
- Author: Achyut Tiwari (B.Tech. Computer Engineering, NIAMT Ranchi)
- Submitted to: Dr. Anubhav Mishra, Namtech Research Park, IIT Gandhinagar
- Compiled: July 2026 (includes Part XVI — post-α improvements and first outdoor flight)

## License

See LICENSE file.
