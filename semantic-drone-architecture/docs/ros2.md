# ROS 2 Humble as Used in This Project

## Overview

ROS 2 Humble is the middleware that glues the entire semantic-drone pipeline together. It is not an operating system; it is a set of libraries and conventions that let independent processes (nodes) discover one another and exchange typed messages on a decentralised publish/subscribe bus built on DDS. The project uses Humble because it is the Long-Term Support release that targets Ubuntu 22.04 — the exact userland provided by JetPack 6 on the Jetson Orin Nano.

This document explains how ROS 2 is actually used: the node graph, Quality-of-Service settings that caused silent failures, the asyncio integration problem that arises when MAVSDK is embedded inside a ROS node, the concrete package layout, and the launch and introspection practices that make the system debuggable.

## What ROS 2 Is (and Is Not)

A ROS system is a graph of nodes connected by topics. A node that produces data publishes it; a node that needs that data subscribes. Neither end knows the identity of the other. Discovery is decentralised (no central master that can crash). Quality-of-Service knobs control reliability, durability and history depth. Humble adds real-time-friendly transport options and a mature Python client library (rclpy).

Everything in the perception–planning–control pipeline is a ROS 2 node written in Python. The flight-critical real-time loops remain inside PX4; ROS never touches the motors directly.

## The Node Graph

Four primary nodes form the closed loop:

- `camera_node` (or the RealSense driver) publishes colour frames on `/image`.
- `segmentation_node` subscribes to `/image` and publishes a binary road mask on `/mask`.
- `planner_node` subscribes to `/mask` and `/mavsdk/pose`, runs A* (and the centreline chain), and publishes waypoints or a `nav_msgs/Path`.
- `control_node` subscribes to the waypoints, issues MAVSDK offboard setpoints, and republishes the vehicle pose on `/mavsdk/pose` so the planner can close the loop.

In simulation an additional `gz_camera_bridge` node converts Gazebo’s camera topic into a ROS 2 `sensor_msgs/Image` so the rest of the pipeline sees identical message types on real and simulated hardware.

## Quality of Service — The Silent Failure Mode

DDS requires that a publisher and a subscriber have compatible QoS profiles before they connect. Sensor data is conventionally `BEST_EFFORT`; control data is usually `RELIABLE`. A mismatch produces no error message — the subscription simply never receives anything. Early in bring-up the camera node’s default best-effort publisher failed to connect to a reliable subscriber on the segmentation side. Explicitly setting both ends to `ReliabilityPolicy.RELIABLE` and `HistoryPolicy.KEEP_LAST` with depth 1 resolved the “no images reaching the segmenter” symptom.

Always inspect QoS with `ros2 topic info <topic> -v` when a callback appears never to fire.

## The asyncio ↔ ROS 2 Integration Problem

This is the single most conceptually important software problem in the project.

ROS 2’s `rclpy.spin(node)` blocks the calling thread and runs callbacks. MAVSDK-Python is built on asyncio; its API consists of coroutines that must run inside an event loop. Two blocking “run forever” constructs cannot share a single thread.

The control node solves the conflict with an async-first pattern:

```python
async def spin_ros(node):
    while rclpy.ok():
        rclpy.spin_once(node, timeout_sec=0.0)
        await asyncio.sleep(0.005)   # yield so MAVSDK coroutines can run

async def main():
    rclpy.init()
    node = ControlNode()
    await node.connect()
    await asyncio.gather(
        spin_ros(node),
        node._pose_stream(),
        node._control_loop(),
    )
asyncio.run(main())
```

The short sleep is the cooperative yield point. Without it one loop starves the other. Blocking work inside any coroutine freezes the entire event loop; therefore every MAVSDK call that waits for the vehicle is wrapped in `asyncio.wait_for(..., timeout=...)`.

A related crash in the pose stream was traced not to logic but to VM resource starvation that collapsed the underlying gRPC channel. A reconnect-with-exponential-backoff wrapper was designed so that a transient drop self-heals rather than ending the flight.

## Package Layout and Build

The workspace follows the standard colcon layout. The Python package `semantic_drone` contains:

- `core/` — shared library code (`px4.py`, segmentation helpers, costmap utilities, skeleton goal logic).
- `nodes/` — the four executable nodes plus the Gazebo bridge.
- `launch/` — Python launch files that start the whole graph with the correct parameters and remappings.

A recurring source of “imports fail under `ros2 run`” was the omission of the `core` sub-package from `setup.py`. Once `find_packages()` (or an explicit list) included it, the modules became importable after `colcon build --symlink-install` and `source install/setup.bash`.

## Introspection Tools Used Daily

```bash
ros2 node list
ros2 node info <node>
ros2 topic list
ros2 topic echo <topic>
ros2 topic hz <topic>
ros2 topic info <topic> -v   # QoS
```

These commands are the primary means of verifying that frames are flowing, that the mask contains road pixels, that the planner is emitting waypoints, and that the pose feedback is live.

## Launch Files

A single launch file starts the four nodes (plus RealSense or the Gazebo bridge) with consistent parameters. Launch arguments select simulation versus hardware, the connection URL for MAVSDK, and safety-related constants such as minimum altitude. The same launch file is used on the Jetson for both distributed SITL and real flight; only the camera source and the MAVLink URL change.

## Lessons Embodied in the ROS Layer

1. Explicit QoS is mandatory; defaults silently disconnect.
2. The same planner code must run on real pose telemetry, not simulation odometry.
3. Asyncio and the ROS executor must be deliberately interleaved; naïve blocking calls freeze the aircraft.
4. Every “wait for the vehicle” call needs a timeout.
5. Package installation (`setup.py`) is part of the functional correctness of the system.

The ROS 2 layer is deliberately thin: it supplies the typed bus, the isolation, and the tooling. All algorithmic intelligence lives in the perception, planning and control modules that sit on top of it. That thinness is what allowed individual nodes to be replaced (HSV → MobileNetV2 U-Net, simple auto-goal → centreline skeleton chain) without rewriting the rest of the graph.
