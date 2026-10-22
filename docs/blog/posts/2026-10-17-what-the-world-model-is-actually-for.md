---
date: 2026-10-17
title: What the world model is actually for
slug: what-the-world-model-is-actually-for
---

# What the world model is actually for

With a judge that is right at rest, the world model finally had a fair question: can it read the judge's verdict from the latent, and can it see it coming? G10 ran on the Mac for free. It is the one world-model gate that passed cleanly, and the next post is about how much of it survived contact with fresh data.

<!-- more -->

## G10

Bars: outcome readout at the judge's timestamp AUROC ≥ 0.90; anticipation 30 steps earlier must beat policy identity by +0.10.

- **Step 1, readout at t_j:** W1 AUROC **1.000**, W2 0.976. The latent at rest contains the outcome.
- **Step 2, anticipation at t_j − 30:** the imagined latent beats policy identity by **+0.179** (0.993 against ground truth).
- **Mid-chunk surprise** (prediction error of the next latent): **1.82×** higher on missed grasps, **2.58×** on stalls, from the first step.

Pass. The architecture of record changed the same day: no planning, no progress tracking. The world model is a **pre-execution veto** (imagine the chunk, refuse it if the imagined outcome reads as failure) and a **mid-chunk surprise monitor**.

## G11-A: does it fit in the tick?

Dual-camera surprise check on the Orin NX, FP16 CUDA: **12.78 ms** mean, 12.95 ms max, inside a 33 ms control tick with 20 ms to spare. Pass.

A correction row was added the next day: the benchmark had loaded the checkpoint with `strict=False` and a key remap that did not fit the image's transformers version, so its encoder was random. The timing stands (same architecture); the latent values from that run are void. A strict-load check now runs before every Jetson gate.

## Honest reading, written before the next gates

G10 is retrospective: W1 was scored on the same 69 rollouts it had seen as unlabeled training video. The real test is a fresh rollout and a fixed threshold. That test came two days later.
