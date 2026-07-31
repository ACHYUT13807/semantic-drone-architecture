# Planning: Occupancy Grids, A* and Centreline Guidance

## Overview

The planner turns a binary road mask into a sequence of waypoints the control node can follow. It constructs an occupancy grid, selects an automatic goal, searches with A*, and (in the current v7 implementation) extracts a centreline skeleton so that the aircraft tracks the middle of the road rather than hugging a shoulder. Continuous replanning keeps the path fresh as the view changes.

## Occupancy Grid and Costmap

The segmentation mask is projected into a 2-D grid whose free cells correspond to road pixels and whose occupied cells correspond to everything else. Several costmap protections were added after early flights and simulations revealed failure modes:

- `largest_road_component` restricts goal selection to the main connected road region, discarding small noise blobs.
- Road-protected inflation prevents obstacle dilation from pinching narrow corridors shut.
- A reachable-free mask guarantees that the selected goal is actually reachable from the current position.

These guards turned an otherwise brittle grid into a map that A* can reliably search.

## Automatic Goal Selection

Because there is no external destination, the system must invent its own goal on every replan. The original heuristic picked a cell far along the visible road in the direction of travel, ignoring the top 20 % of the image (the most distorted region of a nadir view). A subsequent bug allowed the goal to land on an occupied cell; a radial spiral search (`_nearest_free_goal`) corrected it.

## A* Pathfinding

A* searches the free cells from the drone’s current grid coordinate to the chosen goal, returning the shortest path under an 8-connected neighbourhood. The path is then converted into a short list of world-frame waypoints. When the mask is empty the planner logs the fact and refuses to emit a path, preventing the control node from receiving nonsense setpoints.

## Centreline-Guided Planner (v7)

Edge-hugging behaviour of the original goal selection was diagnosed as the dominant source of lateral oscillation. Version 7 replaces the single auto-goal with a pipeline that:

1. Skeletonises the road mask (Zhang–Suen or `cv2.ximgproc.thinning`).
2. Prunes the skeleton to the longest main branch by a double-BFS.
3. Samples an ordered chain of approximately 12 forward waypoints along that branch.
4. Feeds A* one target at a time, advancing the chain as the aircraft progresses.

The result is a smooth centreline track rather than a sequence of shoulder-hugging goals. Combined with the costmap protections listed above, the v7 planner is the configuration that flew on 16 July 2026 and that will be used for the confirmatory post-fix flight.

## Continuous Replanning and Tuning

The entire perception–planning cycle runs continuously. Tuning parameters control how far ahead the skeleton is sampled, how aggressively the chain advances, the inflation radius, and the minimum free-cell clearance. These numbers were derived from both simulation and the first outdoor flight logs; they are recorded in the parameter appendix of the original progress report.

## Status

The planner is functionally complete. The remaining work is validation of the centreline track after the camera-rotation and Pure-Pursuit patches, not further algorithmic invention.
