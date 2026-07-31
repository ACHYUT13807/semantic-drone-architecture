# Performance, Rates and Distributed SITL

## Overview

This document records the measured rates of the pipeline, the diagnosis of the 4.5 Hz bottleneck observed in distributed simulation, the reasons HITL is not viable on the current virtual-machine host, and the inference latency targets for the Jetson.

## Nominal Rates

On the cobot desktop (native Ubuntu) the end-to-end pipeline comfortably exceeds 15 Hz with the HSV segmenter. With the MobileNetV2 network under TensorFlow the rate is lower but still sufficient for the current camera frame rate of 10–15 fps. Control setpoints are issued at a higher rate than perception so that the Pure-Pursuit tracker remains smooth even when a segmentation frame is delayed.

## Distributed SITL Topology

For hardware-in-the-loop style testing the Gazebo world and PX4 SITL run on a separate simulation host while the real ROS 2 pipeline runs on the Jetson. The two machines are joined by:

- ROS 2 DDS discovery (ROS_DOMAIN_ID shared, ROS_LOCALHOST_ONLY=0),
- MAVLink over UDP (udpin://:14540).

Cross-machine discovery was verified: nodes started on the VM appear in `ros2 node list` on the Jetson and vice versa.

## The 4.5 Hz Bottleneck

When the full pipeline was first run in the distributed configuration the observed rate collapsed to approximately 4.5 Hz. Systematic isolation showed that the bottleneck was not the ROS bridge, not the network, not the USB camera, and not the protobuf version. The root cause was Gazebo’s software rendering on the VM, which delivered camera frames at only ~20 % of real time. Once the simulation host was given adequate GPU resources (or the camera rate was artificially limited) the pipeline recovered.

## Why HITL Is Not Viable on the Current VM

Hardware-in-the-loop that places the real Pixhawk in the loop with a simulated world requires low-latency, deterministic timing that the present virtual-machine environment cannot guarantee. The project therefore uses pure SITL for algorithm development and moves directly to the real aircraft for final validation, accepting that some sim-to-real gaps remain.

## Jetson Inference Targets

The production path is TensorRT conversion of the MobileNetV2 U-Net. Until that conversion is complete the network runs under TensorFlow. Measured latency leaves adequate margin for a 10 Hz camera but little headroom for higher rates or additional perception features. TensorRT is therefore on the critical path for any future increase in frame rate or the addition of depth-based obstacle awareness.
