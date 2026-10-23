---
date: 2026-10-23
title: "Paper vs hardware 3: RoboJEPA"
slug: paper-vs-hardware-robojepa
---

# Paper vs hardware 3: RoboJEPA

RoboJEPA (Zholus, Assran et al., Meta FAIR; arXiv 2610.10515) is a scaling law for latent world models trained on 15 000 hours of robot video, including 21.5k LeRobot community episodes from arms like mine. Two gates: can the encoder run on the Orin, and does the pretrained predictor use my actions.

<!-- more -->

## What the paper claims, and how it measures it

A deterministic next-feature predictor on a **frozen V-JEPA 2.1-G** encoder (width 1664, patch 16). Per timestep the sequence is all views' patches plus an action token and a proprio token; attention is joint space-time-view, timestep-causal, **local to 8 timesteps** so it rolls out past its training length; QK-norm for stability. Loss is half teacher-forced L1, half autoregressive L1 from a prefix (the exposure-bias fix). 23 datasets, 12 embodiments, 26 action spaces; AdamW with a shared flat learning-rate stage and per-budget cool-downs.

The scaling law is of **per-token L1 error over a 9-step latent rollout on held-out scenes** (DROID holdout and RoboCasa), compute counted in FLOPs, compute-optimal frontier across 22M–2B. The functional form was chosen by **extrapolation error on 4B and 8B held out of the fit** (Table 2): a second-order law wins (0.6 / 1.4 ×10⁻³) over a plain power law (2.0 / 3.7). Downstream: closed-loop CEM planning in RoboCasa with capability emerging in order (end-effector control → holding → scene geometry → object dynamics, the last only at ~10²² FLOPs); a real Franka table (Table 3: 8B grasp 67 %, lift 50 %, pick-and-place 27 % success; π0.5 as a reference, not a baseline); "over 50 000 evaluation episodes" across both platforms, per-task n in an appendix. Planning is "several seconds per step for 8B, not real-time".

## What I ran

**Gate B, encoder on device.** Bar: 2-view encoder p50 ≤ 200 ms on the Orin NX. Result: **526 ms** (one view 398 ms); the 100M predictor step 252 ms, 300M 487 ms. No-go on device. On the Mac (MPS) the encoder is 1 025 ms and the predictor's attention does not run. Any use is off-board on CUDA.

**Gate A, does the pretrained predictor use my actions.** Two bars on 9-step rollouts of my bowls latents: (1) L1 ≥ 15 % below copy-last for the 300M model; (2) ≥ 5 % below shuffled actions. Result: (1) 300M **11.8 %**, 100M 10.5 %, 1B 15.8 % [14.6, 17.0] on only 30 clips; (2) **8.9–10.4 %** for every size. The primary bar fails; the 1B is borderline on a partial sample; **action conditioning works on my arm** at every size.

**G1b-RJ** (the 1B on the ranking gate) was cancelled as a gate on 2026-10-15: neither outcome would change the build, since the encoder cannot run in the loop.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Action use | implicit in the rollout error | 9–10 % below shuffled, all sizes |
| Rollout error vs model size | clean frontier, near irreducible E ≈ 0.22 | 100M 10.5 → 300M 11.8 → 1B 15.8 % below copy-last (30 clips) |
| Encoder latency | not reported (multi-GPU planning) | 526 ms / 2 views, Orin NX |
| Real-robot n | appendix | 30 clips, no robot |

## What the gap means

My arm is in their training mix, and the pretrained model does read its actions — the Gate A shuffled-action margin is the far-left point of their scaling curve, at zero fine-tuning cost. What it cannot do is run where the arm is. The paper is explicit that planning is not real-time even on a GPU cluster; on an Orin the encoder alone eats sixteen control ticks. So the honest use is offline: feature extraction for a judge or a dataset-level probe. The method lesson is the one to keep: a scaling claim is an **extrapolation test** — fit on some sizes, predict the held-out ones, report the error. My own scaling angle (50 / 100 / 182 / 362 episodes) is a plot until it does that.

**Verdict:** action conditioning reproduces at the small end; the encoder is off the robot.

**What would change it:** a distilled encoder at ≤ 100 ms, which is not my experiment to run.
