# System Architecture and Data Flow

## 1. Purpose and Scope

This document describes the architecture of the Autonomous Semantic Drone Road-Following System as it stands at Prototype 1-α. It covers the high-level purpose of the system, the end-to-end perception–planning–control loop, the physical and logical placement of every major component, the coordinate-frame conventions that must be respected, and the design rationale behind the two-computer split that is the architectural centrepiece of the project.

The goal is not merely to list boxes and arrows. It is to give a reader who has never seen the aircraft a precise mental model of how photons arriving at the RealSense D455 become motor commands leaving the Pixhawk, what can go wrong at each interface, and why the particular decomposition into nodes, topics, and computers was chosen.

## 2. What the System Does

The aircraft looks straight down at a road from the air, decides in real time where the drivable surface continues, and flies itself along that surface without any pre-loaded map or GPS waypoint list. The only source of “where should I go next” is the continuous video stream from a nadir-facing camera.

Operationally the system must:

- Capture a continuous video stream from a downward-facing camera.
- For each frame produce a binary mask that labels every pixel as road or not-road.
- Convert that mask into an occupancy grid the planner can reason over (free = road, occupied = everything else).
- Automatically select a goal somewhere along the visible road in the direction of travel.
- Run a path-planning search (A*) over the grid to obtain a sequence of free-space cells.
- Convert those cells into waypoints expressed in a world frame the flight controller understands.
- Feed the waypoints to the flight controller so that the motors are driven to follow them while respecting altitude bounds, geofence, and kill-switch constraints.
- Repeat the entire loop many times per second so that as the drone moves and the view changes the plan is continuously refreshed.

This is a closed perception-to-actuation loop. Nothing about the road geometry is known in advance.

## 3. Why “Semantic”

A large fraction of existing road-following drones cheat in one of three ways:

1. They replay a pre-recorded GPS track and therefore never need to see the road at all.
2. They follow a coloured line or artificial marker that was laid specifically for them.
3. They are flown by a remote pilot who performs the perception with human eyes.

Semantic means the drone constructs a per-pixel understanding of the scene. Every pixel is assigned a class label (“road” versus “not-road”). The resulting free-space map is general: it can in principle follow an unmarked rural road the vehicle has never seen, under changing illumination, around curves, because the system recognises *road-ness* rather than a specific track, colour blob, or GPS breadcrumb.

The practical consequence is that the perception problem becomes a semantic-segmentation problem, the planning problem becomes a search over a dynamic occupancy grid derived from that segmentation, and the control problem becomes the translation of the resulting waypoints into safe, smooth motor commands.

## 4. The Two-Computer Split

A single architectural fact shapes the entire system: the work is divided across two computers that talk to each other over a serial/USB MAVLink link.

- The **companion computer** is an NVIDIA Jetson Orin Nano 8 GB. It runs the camera driver, the segmentation network (or HSV fallback), the occupancy-grid construction, the A* planner, the automatic goal selection, the waypoint management, and the high-level control logic that issues MAVSDK setpoints. All of the ROS 2 code and all of the machine-learning inference live here. The presence of a CUDA-capable GPU is the reason the vision workload resides on this board.

- The **flight controller** is a Pixhawk 6C Mini running PX4 1.17.0. It performs the fast, safety-critical, low-level work: reading the IMU hundreds of times per second, running the attitude and position control loops, driving the ESCs, handling arming and failsafes, and enforcing geofences. It is a hard real-time embedded system; it is emphatically not the place where a neural network runs.

The companion says “go to this position” or “take off to this altitude.” The flight controller decides the exact motor commands that realise the request and refuses anything it judges unsafe. Keeping the split clean — smart-but-slow on the Jetson, dumb-but-fast-and-safe on the Pixhawk — is the single most important design principle of the project. Most of the integration debugging that occurred during bring-up can be traced to the hand-off across this boundary.

Nothing flight-critical depends on the perception stack. If the network fails, the camera disconnects, or the planner stalls, the pilot and the PX4 failsafe logic remain fully capable of recovering the aircraft.

## 5. The Three Planes

A clean way to hold the whole system in one’s head is to separate three logical planes:

**Perception plane.** Turns pixels into a notion of free space. Nodes: camera (or RealSense driver), segmentation, and (in simulation) the Gazebo camera bridge. Output: a binary mask or occupancy grid.

**Decision plane.** Turns free space into intent. Node: planner. It selects an automatic goal and computes a path. Output: a sequence of waypoints.

**Actuation plane.** Turns intent into physical motion. Node: control (on the Jetson) together with the entire PX4 stack (on the Pixhawk). Output: motor commands and a continuous telemetry stream that flows back upward.

Data flows perception → decision → actuation. A thin feedback wire (the vehicle’s pose, republished by the control node on `/mavsdk/pose`) runs from actuation back to decision so that the planner always knows where the aircraft currently is. This is a textbook sense–plan–act architecture, the dominant paradigm for this class of autonomous robot.

## 6. End-to-End Pipeline as a Conveyor Belt

Think of the system as a continuously running conveyor belt. One trip along the belt looks like this:

1. **Capture.** The downward camera produces a frame — a grid of pixels, each carrying a colour value.
2. **Segment.** The frame is passed to a segmentation stage that emits a mask of identical spatial dimensions in which every pixel is marked road (1) or not-road (0).
3. **Project to a grid.** The mask is turned into an occupancy grid: a two-dimensional array the planner treats as a local map, with road cells free and all other cells occupied.
4. **Choose a goal.** Because there is no external destination, the system automatically selects a point far along the visible road in the direction of travel (`find_auto_goal` logic, later refined into a centreline skeleton chain).
5. **Plan.** A* searches the grid from the drone’s current position to that goal and returns the shortest free-space path.
6. **Emit waypoints.** The path is converted into a short sequence of waypoints expressed in a world frame.
7. **Follow.** The control stage feeds the waypoints to PX4 via MAVSDK; PX4 flies them.
8. **Repeat.** As the aircraft moves, the camera sees a new stretch of road and the whole belt runs again — continuous replanning.

Every stage is realised as an independent ROS 2 node. The “conveyor belt” is the ROS 2 message bus: nodes publish their outputs on topics and subscribe to the inputs they need. The stages therefore run concurrently rather than in strict sequence.

### Data Types That Flow Between Stages

| Stage          | Input                  | Output                     | Typical ROS message type                  |
|----------------|------------------------|----------------------------|-------------------------------------------|
| Camera         | raw sensor             | colour image               | `sensor_msgs/Image`                       |
| Segmentation   | colour image           | binary mask                | `sensor_msgs/Image` (mono8)               |
| Planner        | mask + current pose    | path / next waypoint       | `nav_msgs/Path` or point                  |
| Control        | waypoint + telemetry   | MAVLink setpoints          | (MAVSDK offboard call)                    |
| Pose feedback  | PX4 telemetry          | current position           | `geometry_msgs/PointStamped` on `/mavsdk/pose` |

The pose-feedback arrow is what closes the loop. Earlier versions of the planner consumed simulation odometry (`/odom`). Switching it to consume `/mavsdk/pose` was a deliberate design change that erased a sim-only dependency and allowed the identical planner code to run on real hardware.

## 7. Physical Placement of Components

```
┌───────────────────────────── Jetson Orin Nano 8GB ─────────────────────────────┐
│                                                                                 │
│   camera / RealSense ──/image──▶ segmentation_node ──/mask──▶ planner_node      │
│      ▲                                                     │                     │
│      │                                                     │ (/path, waypoints)  │
│      │                                                     ▼                     │
│                                              control_node ── MAVSDK ──┐          │
│                                                     ▲                  │          │
│                                                     └──/mavsdk/pose────┘          │
│                                              (telemetry republished)              │
└───────────────────────────────────────────────────────────────┬─────────────────┘
                                                                 │ MAVLink (serial/USB)
                                                                 ▼
                                           ┌────────── Pixhawk 6C Mini (PX4) ──────────┐
                                           │  IMU • EKF2 • position/attitude control    │
                                           │  arming • failsafes • ESC/motor outputs    │
                                           └────────────────────────────────────────────┘
                                                                 │ PWM / DShot
                                                                 ▼
                                                        ESCs ▶ Motors ▶ Props
```

Everything above the MAVLink line is ROS 2 + Python + machine learning running on Linux. Everything below it is hard real-time embedded firmware.

## 8. Coordinate Frames

A subtle but constant source of bugs is the set of coordinate frames that must be kept consistent:

- **Image frame.** Pixel coordinates (u, v), origin at the top-left, u increasing right, v increasing down. The segmentation mask lives here.
- **Grid frame.** The occupancy-grid indices (row, col) that the planner searches. Usually a direct re-indexing of the image, but the mapping must be known exactly.
- **Body frame.** Attached to the aircraft: x forward, y right, z down (aerospace FRD convention used by PX4).
- **World / local frame.** A fixed frame in which the aircraft moves. PX4 and MAVSDK commonly expose position in NED (North-East-Down) or a local ENU/NED variant depending on the interface.

Whenever the planner produces a path in image/grid coordinates and the control node must turn it into a position setpoint, a transform chain grid → body → world is required. A single sign error sends the aircraft in the opposite direction. Many of the tuning parameters that appear later exist partly to provide robustness against small frame or latency errors. The standing rule is: always know which frame a coordinate lives in before performing arithmetic on it.

## 9. Why the Chosen Stack

**ROS 2 Humble** supplies node isolation, a typed message bus, Quality-of-Service controls, and a mature ecosystem of drivers (including the RealSense driver). Humble is the LTS that matches the Ubuntu 22.04 userland shipped by JetPack 6.

**PX4** is a mature, open, real-time flight stack with first-class offboard control and a clean MAVSDK API. It owns the parts that must never be wrong — attitude stabilisation, arming logic, failsafes — so the project can concentrate on perception and planning.

**MAVSDK-Python** wraps the MAVLink protocol in a clean asynchronous Python API. The trade-off is that its asyncio event loop must be reconciled with ROS 2’s executor; that reconciliation is one of the central software problems of the project and is treated in detail in the ROS 2 document.

**HSV first, learned segmentation later.** Colour thresholding is cheap, deterministic, needs no GPU, and is trivially debuggable. It allowed the rest of the pipeline (planning, control, hardware) to be proven before the network was ready. The MobileNetV2 U-Net is the principled long-term segmenter, but it carries a domain gap and requires TensorRT for acceptable latency. The pragmatic ordering “make it work simply, then make it smart” is a recurring theme.

**Jetson Orin Nano.** It supplies a CUDA GPU in a form factor and power budget that can fly, while still running full Ubuntu and ROS 2. The 8 GB memory ceiling directly constrains model size and therefore the quantisation and architecture choices made for the segmentation network.

**RealSense D455.** It delivers both RGB and depth from a single USB device with a well-supported ROS 2 driver. Even though the current pipeline segments on colour only, the depth stream remains available for future obstacle awareness and altitude sanity checks.

## 10. Dependency Stack (Bottom-Up)

Reading from the bottom:

```
Autonomous road-following behaviour          ← the goal
─────────────────────────────────────────────
A* planner + auto-goal + continuous replanning ← decision logic
─────────────────────────────────────────────
HSV / MobileNetV2-U-Net segmentation         ← perception
─────────────────────────────────────────────
ROS 2 Humble nodes, topics, QoS              ← middleware glue
─────────────────────────────────────────────
MAVSDK-Python ↔ MAVLink ↔ PX4 1.17.0         ← flight interface + stack
─────────────────────────────────────────────
Jetson Orin Nano (Ubuntu/JetPack) | Pixhawk 6C ← hardware
```

A failure at any layer breaks everything above it. Bring-up therefore proceeds bottom-up: solidify the hardware and PX4, then the MAVSDK link, then ROS, then perception, then planning, then behaviour. The project has climbed most of this stack in simulation and has re-climbed the lower layers on real hardware; the remaining items are physical and validation steps rather than open architectural questions.

## 11. Status at the Close of Prototype 1-α

Every major subsystem exists in a working form on the target hardware, with a verified path from camera photons to motor outputs. TensorRT FP16 conversion of the MobileNetV2 + Gabor network is complete (31.2 ms end-to-end, ~32 FPS, 12-class head). The first outdoor autonomous Offboard flight under real GPS lock was flown on 16 July 2026 (26 s of accepted setpoints). An oscillation was diagnosed as a 180° camera-roll mounting error and corrected in software. Residual items are domain fine-tuning on real D455 frames, a confirmatory centreline-tracking flight after the patches, USB 3.x for the RealSense, a minor mount-plate CAD revision, and the occlusion-recovery research track. None of these items reopen the architecture.

## 12. Related Documents

- `ros2.md` — detailed treatment of the middleware, nodes, QoS, and the asyncio integration problem.
- `hardware.md` — physical components, power architecture, serial link, and mechanical design.
- `perception.md` — camera pipeline, HSV thresholds, the segmentation rewrite, and TensorRT.
- `planning.md` — occupancy grids, A*, automatic goal selection, and the centreline-guided planner v7.
- `control.md` — control node, Pure Pursuit, takeoff sequence, and pose feedback.
- `safety.md` — pilot gate and layered safety model.
- `performance.md` — rate measurements, distributed SITL diagnosis, and Jetson inference latency.
- `troubleshooting.md` — complete log of bugs found and fixed.
- `deployment.md` — reproducible provisioning of a fresh Jetson.
- `roadmap.md` — remaining tasks expressed as a dependency graph.
