# Semantic Drone Architecture

**Autonomous Semantic Drone Road-Following System**  
Prototype 1-α Progress Documentation Repository

This repository contains the complete technical documentation for the Autonomous Semantic Drone project — a quadrotor that performs real-time semantic road following using only onboard vision, without pre-loaded maps or GPS waypoint lists. The system classifies every pixel of a downward-facing camera image into road versus non-road, builds an occupancy grid, plans a path with A*, and flies the resulting centreline via offboard control on PX4.

The documentation is organised to mirror the physical and software architecture of the aircraft. It is intended both as a durable record of the working configuration at Prototype 1-α and as the companion reference for the project website.

## Project Snapshot

| Component | Technology |
|-----------|------------|
| Flight controller | Pixhawk 6C Mini running PX4 1.17.0 |
| Companion computer | NVIDIA Jetson Orin Nano 8GB (JetPack 6.x / Ubuntu 22.04) |
| Camera | Intel RealSense D455 (RGB + depth) |
| Middleware | ROS 2 Humble |
| Flight interface | MAVSDK-Python 2.8.4 over MAVLink |
| Segmentation | MobileNetV2 U-Net + Gabor texture fusion (val_road_iou ≈ 0.80) |
| Classical fallback | HSV colour thresholding |
| Planning | A* over semantic costmap with centreline skeletonisation (v7) |
| Simulation | Gazebo + PX4 SITL (distributed topology supported) |
| Airframe | Quad-X carbon fibre, 6S 12000 mAh |

On 16 July 2026 the aircraft completed its first outdoor autonomous Offboard flight under real GPS lock. That flight exercised the full perception–planning–control pipeline and revealed a clear oscillation whose root cause (180° camera mounting rotation) and corrective patches are documented herein.

## Repository Layout

```
.
├── README.md                 # This file
├── LICENSE
├── docs/
│   ├── architecture.md       # System architecture & data flow
│   ├── ros2.md               # ROS 2 Humble usage, nodes, asyncio integration
│   ├── hardware.md           # Pixhawk, Jetson, RealSense, power, mounts
│   ├── perception.md         # Camera pipeline, HSV, U-Net rewrite, TensorRT
│   ├── planning.md           # Occupancy grids, A*, auto-goal, centreline v7
│   ├── control.md            # Control node, Pure Pursuit, offboard sequence
│   ├── safety.md             # Pilot gate, layered safety model, failsafes
│   ├── performance.md        # SITL rates, Jetson inference, bottlenecks
│   ├── troubleshooting.md    # Bug log and fixes
│   ├── deployment.md         # Reproducibility guide for a fresh Jetson
│   ├── roadmap.md            # Remaining tasks & dependency graph
│   ├── research.md           # Road-occlusion recovery theory
│   └── evaluation.md         # Verification evidence & metrics
├── assets/                   # Images, diagrams, logs
└── diagrams/                 # Architecture drawings
```

## One-Sentence Description

The aircraft looks straight down at a road, decides in real time where the drivable surface continues, and flies itself along that surface using only what its nadir camera sees — frame by frame.

## Why “Semantic”

Most existing road-following drones either replay a pre-recorded GPS track, follow a coloured line laid for them, or rely on a remote pilot. Semantic means the drone builds a per-pixel understanding of the scene: every pixel is classified as road or not-road. This generalises to unmarked rural roads, changing light, and curves the vehicle has never seen, because the system recognises *road-ness* rather than a specific track or colour blob.

## Architectural Centrepiece: The Two-Computer Split

Everything flight-critical lives on the Pixhawk. Perception, planning, and high-level decision-making live on the Jetson. The two communicate exclusively over MAVLink. The flight controller never depends on the perception stack; if the network fails, the pilot or PX4 failsafes remain fully capable.

This separation is the single most important design principle of the project. It is also the source of most of the integration work: reconciling ROS 2’s executor with MAVSDK’s asyncio event loop, keeping coordinate frames consistent, and ensuring that a stalled vision pipeline cannot prevent the aircraft from being recovered.

## Status at Prototype 1-α

- Flight controller brought up, calibrated, and bench-verified.
- Companion ↔ flight-controller serial link verified with MAVSDK.
- Perception network rebuilt (MobileNetV2 encoder + Gabor fusion + Dice + CE loss).
- Centreline-guided planner (v7) implemented and costmap protections added.
- First outdoor autonomous flight completed (16 July 2026).
- Remaining items are physical (ESC mapping, USB 3 port, mount-plate CAD tweak) and a short list of software validation steps — not open architectural questions.

## How to Read This Documentation

Start with `docs/architecture.md` for the system view, then `docs/ros2.md` for the middleware that glues the pipeline together. Hardware bring-up is covered in `docs/hardware.md`. The perception rewrite that rescued near-empty masks is detailed in `docs/perception.md`. Planning and control follow in their respective files. Safety architecture, performance diagnosis, the full troubleshooting log, and the deployment recipe for a fresh Jetson complete the set.

All figures referenced in the original progress report are original; no third-party dataset samples or paper figures have been reproduced.

## Repository & Attribution

- Original repository: ACHYUT13807/semantic-drone (private)
- Author: Achyut Tiwari (B.Tech. Computer Engineering, NIAMT Ranchi)
- Submitted to: Dr. Anubhav Mishra, Namtech Research Park, IIT Gandhinagar
- Compiled: July 2026

This documentation repository is maintained as the public-facing, Markdown-only companion to the working code and the project website.

## License

See LICENSE file.

---

*Progress review of the semantic drone project at Prototype 1-α: subsystems built and verified, the segmentation rewrite, dependency and SITL engineering on the Jetson, the first outdoor autonomous flight, and the remaining path to a corrected centreline-tracking flight.*
