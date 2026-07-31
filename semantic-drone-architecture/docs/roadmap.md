# Roadmap and Remaining Tasks

## Overview

Prototype 1-α closed with every major subsystem verified and the first outdoor autonomous Offboard flight completed (16 July 2026). TensorRT FP16 conversion, texture fusion, actuator mapping, outdoor GPS lock, and several other items that appeared as open in earlier drafts were resolved before or by that flight. This document lists the residual work that remains after Part XVI.

## Resolved Before / By the First Outdoor Flight

- TensorRT FP16 conversion of the MobileNetV2 + Gabor network (10× speedup, 31.2 ms / ~32 FPS).
- Gabor texture fusion as a permanent part of the deployed architecture.
- Actuator mapping and ESC spin-direction verification (required for the aircraft to arm and take off under its own power).
- Outdoor GPS lock (eph ≈ 0.22 m, clearing the geofence pre-arm).
- 20 Hz setpoint streamer refactor that eliminated Offboard priming failures.
- Boot auto-start via systemd with a simple DEV_MODE flag file.

## Still Open

1. **Domain fine-tuning / validation** of the network on real D455 frames (correctness; the model is fixed, the domain is not).
2. **Confirmatory centreline-tracking outdoor flight** after the 180° camera-rotation fix, Pure-Pursuit lookahead reduction, and deceleration law.
3. RealSense on a true USB 3.x port (quality of training imagery and higher frame-rate headroom).
4. Mount-plate geometry conflict (3×3 cm aperture vs Jetson pocket) — a small CAD revision.
5. End-to-end verification of the final flight-configuration serial link (TELEM2 → UART1) under the closed airframe.
6. Implementation of the road-occlusion recovery theory (research track).

## Dependency View

The critical path to a successful centreline-tracking flight is now short:

```
Camera-rotation + Pure-Pursuit + deceleration patches (already in codebase)
        │
        ▼
Domain fine-tune on real frames (optional for a first corrected flight, required for robustness)
        │
        ▼
Post-fix outdoor flight confirming centreline track
```

Physical items (USB 3, mount plate, serial-link harness check) improve quality and reproducibility but are not blockers to a corrected flight once the software patches are loaded.

## Parallel Research Track

Road-occlusion recovery (climb to widen FOV + bridge the gap with heading-guided A* and unknown-not-obstacle cells) remains an original contribution that can be pursued independently of the centreline-tracking confirmation flight.
