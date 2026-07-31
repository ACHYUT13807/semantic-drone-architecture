# Original Contribution — Road-Occlusion Recovery Theory

## Overview

Part XIV of the Prototype 1-α progress report records an original theoretical contribution developed with Dr. Anubhav Mishra: a recovery strategy for situations in which the road is temporarily occluded from the downward camera (by a vehicle, a bridge, foliage, or a sharp crest). This document summarises the problem statement, the proposed approach, and the status of the idea within the project.

## The Problem

A pure vision-based road follower that relies solely on the current frame has no memory. When the road disappears from the image the planner receives an empty mask, A* returns “no path,” and the aircraft either stops or continues on its last heading — both of which are unsafe or ineffective. Occlusions of a few seconds are common in real environments; a practical system must recover from them gracefully.

## Proposed Theory

The recovery theory treats the last known centreline as a short-term prior. While the mask remains empty the planner continues to advance along a decaying extrapolation of that centreline, simultaneously monitoring for the reappearance of road pixels. When a sufficient connected component of road reappears, the system re-initialises the skeleton chain and resumes normal centreline tracking. Safety bounds (maximum extrapolation distance, maximum time without a mask, altitude floor) prevent the aircraft from flying indefinitely into unknown space.

The idea is deliberately simple: it does not require a full simultaneous-localisation-and-mapping solution or a pre-loaded map. It only requires that the centreline extractor already present in the v7 planner be given a short temporal memory and a set of conservative recovery limits.

## Status

The theory has been articulated and discussed; it has not yet been implemented in the flight code. It remains a parallel research track that can be pursued after the critical-path items of Prototype 1-α are closed. Implementation would consist of a modest extension to the planner node and additional logging so that recovery events can be evaluated on real flights.
