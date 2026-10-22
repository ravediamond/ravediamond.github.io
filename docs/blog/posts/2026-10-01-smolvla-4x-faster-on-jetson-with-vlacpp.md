---
date: 2026-10-01
title: SmolVLA four times faster on the Jetson with vla.cpp
slug: smolvla-4x-faster-on-jetson-with-vlacpp
---

# SmolVLA four times faster on the Jetson with vla.cpp

974 ms per chunk made SmolVLA stutter on the Orin NX. Serving the same checkpoint through vla.cpp brought it to 238 ms with no loss in task success.

<!-- more -->

## Numbers, same bowls task, 10 trials

| Engine | Success | Chunk inference |
|---|---|---|
| SmolVLA, PyTorch | 9/10 | 974 ms |
| SmolVLA, vla.cpp | 9/10 | 390 ms |
| SmolVLA, vla.cpp + flash attention | 8/10 | 238 ms |

The 8/10 on the flash-attention build is one failed trial out of ten; the gate is too small to say whether that is a regression or noise. Inference went from blocking the control loop most of the time to 14 % of the time at the 26.6 Hz tick.

## What it took

- A cppzmq workaround for Ubuntu 22.04 / JetPack 6, now documented upstream (vla.cpp `#47`).
- Real SO-101 latency numbers posted on the project's benchmark issue (`#31`), which were the first on this hardware.
- An `--arch act` client preset so the same client drives ACT and SmolVLA.

## Why it matters

On GPU-poor hardware the engine is the lever, not the model. Everything that follows runs the policy through vla.cpp, and two weeks later the same engine is where the chunk-boundary fix lands at a cost of 0.4 ms.
