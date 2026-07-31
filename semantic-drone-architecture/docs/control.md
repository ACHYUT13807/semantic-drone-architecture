# Control Node and Offboard Flight Sequence

## Overview

The control node is the final software stage on the Jetson. It consumes waypoints from the planner, issues MAVSDK offboard setpoints to the Pixhawk, streams the vehicle’s pose back into ROS so the planner can close the loop, and enforces the pilot-gate safety interlock. This document describes the node’s responsibilities, the Pure-Pursuit tracker, the takeoff and flight sequence, and the lessons learned from the first outdoor autonomous flight.

## Responsibilities

1. Establish and maintain the MAVSDK connection (UDP for SITL, serial for hardware).
2. Republish PX4 telemetry as a ROS `geometry_msgs/PointStamped` on `/mavsdk/pose`.
3. Consume the planner’s waypoint stream and convert each waypoint into an offboard position setpoint.
4. Run a Pure-Pursuit tracker that smooths the path and limits lateral acceleration.
5. Gate all autonomous motion behind an explicit pilot acknowledgement (`_wait_for_pilot`).
6. Respect altitude floors, geofence, and the hardware kill switch.

## Pose Feedback

The pose stream is the wire that closes the perception–planning–control loop. It is implemented as an asyncio coroutine that reads `drone.telemetry.position_velocity_ned()` and publishes the North-East-Down coordinates. Because the same topic is used in both simulation and real flight, the planner never needs a sim-only odometry source.

## Pure Pursuit

After the first outdoor flight revealed a clear oscillation, the simple “fly to next waypoint” logic was replaced by a Pure-Pursuit controller. The look-ahead distance and the deceleration profile near each waypoint were tuned from the flight logs. Together with the 180° camera-rotation correction they eliminate the dominant oscillatory mode.

## Flight Sequence

1. Connect to the vehicle (no health-check gate so that bench work without GPS can proceed).
2. Wait for the pilot to acknowledge autonomy via the RC switch or a ROS service.
3. Arm and take off to a safe altitude (default 20 m, timeout 60 s).
4. Enter Offboard mode and begin following the planner’s waypoints.
5. Continuously replan and update the Pure-Pursuit target.
6. On loss of planner messages or on pilot override, fall back to Position or Altitude mode and await further commands.

## Safety Integration

The control node never bypasses PX4’s own pre-arm checks or failsafes. The pilot gate is an additional software interlock that prevents the aircraft from taking off under autonomy until a human has confirmed that the area is clear and that the kill switch is functional. Details of the layered safety model appear in `safety.md`.

## Status

The control node flew the first outdoor autonomous mission on 16 July 2026. The residual work is confirmation that the Pure-Pursuit and camera-rotation patches produce a stable centreline track on the subsequent flight.
