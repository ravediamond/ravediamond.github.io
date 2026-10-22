---
date: 2026-09-29
title: Three policies on a $300 arm, and a gate harness to compare them
slug: three-policies-on-a-300-dollar-arm
---

# Three policies on a $300 arm, and a gate harness to compare them

Before any research question, the arm had to run a learned policy reliably and I had to be able to measure it the same way every time. This post is the baseline everything else is compared against.

<!-- more -->

## Setup

- SO-101 follower arm (6 DoF, STS3215 servos), SO-101 leader for teleop, Logitech C920 top camera, a wrist camera.
- Jetson Orin NX 16 GB as the robot computer, headless, in a Docker image I published (the Dockerfile is merged upstream in LeRobot, `#4599`).
- A MacBook M1 for a second machine, and vast.ai for anything that needs a real GPU.
- Task: "put the blue bowl in the pink bowl", and the reverse. 182 teleop episodes, idle starts trimmed, public on the Hub as `ravediamond/so101_bowls_all_trimmed`.

## The harness

`monitor_rollout.py` runs a fixed number of trials, logs a tick file per trial (joint commands, chunk latency, timestamps), records video from both cameras, and takes one key press from the operator at the end of each trial: success or failure. Every trial since this day sits in one catalogue, 85 of them with video by mid-October. The rule that mattered later: freeze the pass bar before the run.

## Results, 10 trials each

| Policy | Machine | Success | Chunk inference |
|---|---|---|---|
| ACT | M1 Mac | 10/10 | – |
| SmolVLA 450M (PyTorch) | M1 Mac | 9/10 | – |
| ACT | Orin NX | 9/10 | 59 ms |
| SmolVLA 450M (PyTorch) | Orin NX | 9/10 | 974 ms |

Both policies work. SmolVLA at 974 ms per 50-action chunk is a problem on the Orin: the arm visibly pauses at every chunk boundary. That number is the reason the next post exists.

## What I learned

The first thing a hobby setup needs is not a better model, it is a harness that produces the same number twice. Operator judgment is fine for 10 trials; it will not be fine for 100, which is a judging problem I only understood two weeks later.
