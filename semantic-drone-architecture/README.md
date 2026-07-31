# Autonomous Semantic Drone Road-Following System

This repository documents the architecture and pipeline of Prototype 1-a of the semantic drone project.[span_0](start_span)[span_0](end_span)

## Project Overview
* The system is an aircraft that can look straight down at a road from the air and decide in real time where the drivable surface goes.[span_1](start_span)[span_1](end_span)
* It flies itself along that surface without any pre-loaded map or GPS waypoint list.[span_2](start_span)[span_2](end_span)
* The aircraft operates using only what its downward-facing camera sees, frame by frame.[span_3](start_span)[span_3](end_span)
* The architecture stitches a perception problem, a planning problem, and a control problem into a single closed loop.[span_4](start_span)[span_4](end_span)

## Core Stack Highlights
* **Companion OS**: Ubuntu 22.04.5 LTS on Jetson.[span_5](start_span)[span_5](end_span)
* **Middleware**: ROS 2 Humble.[span_6](start_span)[span_6](end_span)
* **Flight Stack**: PX4 1.17.0.[span_7](start_span)[span_7](end_span)
* **Compute Hardware**: NVIDIA Jetson Orin Nano 8GB.[span_8](start_span)[span_8](end_span)
* **Flight Controller**: Pixhawk 6C Mini.[span_9](start_span)[span_9](end_span)