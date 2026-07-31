# Safety Architecture

## Overview

Autonomy is never allowed to start without an explicit human gate, and the flight controller always retains the final authority to refuse an unsafe command. This document describes the pilot gate, the layered safety model, the PX4 parameters that were changed for the project, and the standing rules that govern every test flight.

## The Pilot Gate

Before any autonomous takeoff the control node calls `_wait_for_pilot()`. The function blocks until an RC switch (or a ROS service) acknowledges that:

- the area is clear of people and obstacles,
- the hardware kill switch has been verified,
- the operator is ready to take manual control at any moment.

Only after that acknowledgement does the node proceed to arm and enter Offboard mode. The gate is deliberately simple and hard to bypass; it is the software equivalent of a physical safety interlock.

## Layered Safety Model

Safety is enforced at four independent layers:

1. **Hardware kill switch** — a physical RC channel that immediately cuts motor power.
2. **PX4 failsafes** — RC loss, geofence breach, battery low, attitude limits, etc. These run inside the real-time firmware and cannot be disabled by the companion computer.
3. **Software pilot gate** — the `_wait_for_pilot` interlock described above.
4. **Operator monitoring** — continuous visual and telemetry observation during every flight.

Nothing in the perception or planning stack can override layers 1 or 2.

## PX4 Parameters Changed

A small set of behavioural parameters were adjusted for the project’s operating envelope (COM_RCL_EXCEPT, NAV_DLL_ACT, geofence settings, etc.). The exact values and the rationale for each change are recorded in the parameter appendix of the original progress report. All changes are reversible and were verified on the bench before any outdoor flight.

## Standing Rules

- Props remain off until the aircraft is outdoors and the pre-flight checklist of Appendix L has been completed.
- The first motion of every session is a low-altitude, tethered or Stabilized-mode test.
- Autonomy is never enabled until the pilot gate has been exercised and the kill switch has been confirmed.
- After every flight the PX4 `.ulg` log and any ROS bag are archived with a short written note of what was tested and what was observed.

These rules are non-negotiable. They are what allow the project to move from simulation to real flight without accepting unacceptable risk.
