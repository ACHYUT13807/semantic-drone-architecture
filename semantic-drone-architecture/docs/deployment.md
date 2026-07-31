# Deployment and Reproducibility Guide

## Overview

This document is a condensed, actionable version of Appendix J of the Prototype 1-α progress report. It describes how to provision a fresh Jetson Orin Nano so that the semantic-drone workspace builds, the RealSense driver runs, the pinned Python environment is restored, and the four-node pipeline can be launched.

## Base Operating System

Flash JetPack 6.x (L4T R36.3.0) via the NVIDIA SDK Manager from an x86 host. Use the Desktop image (not Server) so that rviz and rqt remain available for debugging. Expand the root filesystem to fill the SSD, enable SSH, and set a unique hostname.

## System Packages and ROS 2 Humble

Install the single ROS 2 apt repository (never two). Install the Humble desktop packages and the RealSense packages from the Intel repository after importing the correct GPG key. Confirm with `ros2 doctor` and `realsense-viewer`.

## Pinned Python Environment

```bash
python3 -m pip install --user 'setuptools==79.0.1' 'wheel'
# TensorFlow 2.15, MAVSDK 2.8.4, protobuf==3.20.3, OpenCV, etc.
```

The exact pin list is the artefact that lets TensorFlow and MAVSDK coexist. Reproducing it on a new board is the difference between a working system and a day of dependency archaeology.

## Repository Build

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

Verify that `ros2 run semantic_drone camera_node` (and the other three nodes) start without import errors.

## Model Weights

Copy the `.weights.h5` file into the models directory. On the Jetson the architecture is rebuilt in code and the weights are loaded; a full `.keras` archive saved under Keras 3 will not load under the Jetson’s Keras 2.15.

## First Smoke Test

Launch the four-node graph with the RealSense (or the Gazebo bridge). Confirm with `ros2 topic hz` that images, masks and pose messages are flowing. Only then connect MAVSDK to a real or simulated vehicle.

## What Is Deliberately Omitted

Firmware flashing, ESC mapping, outdoor GPS procedures and the full pre-flight checklist remain in the original progress report and in `hardware.md` / `safety.md`. This guide covers only the software image that must be reproducible on every new Jetson.
