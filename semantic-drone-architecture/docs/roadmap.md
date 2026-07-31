# Roadmap and Remaining Tasks

## Overview

Prototype 1-α is the first configuration in which every major subsystem exists in a working form on the target hardware. The residual work is a short, concrete list of physical and software tasks required for a corrected centreline-tracking outdoor flight. This document presents those tasks as a dependency graph and indicates which items have already been closed by the first outdoor flight of 16 July 2026.

## Immediate Hardware Blockers (Critical Path)

1. ESC wiring, actuator mapping and spin-direction verification (props off).
2. Outdoor GPS lock to clear the “Fence requires position” pre-arm.
3. RealSense on a true USB 3.x port (lift the 640×480 @ 15 fps cap).
4. Mount-plate geometry fix (3×3 cm aperture versus Jetson pocket).
5. End-to-end verification of the flight-configuration serial link (TELEM2 → UART1).

## Immediate Software Tasks (Near Critical Path)

1. Domain fine-tuning / validation of the MobileNetV2 U-Net on real D455 frames after the 180° camera-rotation correction.
2. TensorRT conversion of the fine-tuned network.
3. Confirmation that the Pure-Pursuit and deceleration patches eliminate the oscillation observed on 16 July.
4. Post-fix centreline-tracking outdoor flight.

## Parallel Tracks

- Perception upgrades (depth-based obstacle awareness, higher frame rate once TensorRT is in place).
- Original research on road-occlusion recovery (see `research.md`).
- Further costmap and planner tuning based on additional outdoor logs.

## Dependency Graph (Simplified)

```
GPS lock ──────────────────────┐
ESC mapping & spin direction ──┼──▶ first corrected centreline flight
USB 3 port for RealSense ──────┤
Camera-rotation + Pure-Pursuit patches (already applied)
Domain fine-tune + TensorRT ───┘
```

Once the critical-path items above are closed, Prototype 1-α is complete. Subsequent work moves the system from “first corrected flight” to “reliable repeated autonomous road following under varying conditions.”
