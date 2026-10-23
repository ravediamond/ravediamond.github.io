---
date: 2026-10-23
title: "Paper vs hardware 2: DINO-WM"
slug: paper-vs-hardware-dino-wm
---

# Paper vs hardware 2: DINO-WM

DINO-WM (Zhou, Pan, LeCun, Pinto; arXiv 2411.04983) is the recipe behind W2: frozen DINOv2 patches plus a learned dynamics model. It had the better representation and lost the same planning gate as W1, more slowly.

<!-- more -->

## What the paper claims, and how it measures it

Three parts, one trained: a **frozen** DINOv2 encoder gives N patch embeddings per 224×224 frame; a ViT transition model takes H frames of patches with the action MLP-embedded and concatenated to every patch, attends causally at the frame level, and predicts the whole next frame of patches at once; a transposed-conv decoder trained separately is for visualisation only (backpropagating its loss into the predictor hurt, their A.4.3). Loss: ‖p(enc(o), φ(a)) − enc(o_{t+1})‖², teacher-forced. Because the encoder is frozen, collapse is impossible by construction; the price is a representation that cannot adapt to the task. Planning: CEM in MPC to an image goal, cost ‖ẑ_T − z_g‖²; gradient descent through the model was tried and was worse (A.5.3).

Six simulated environments. Metric: success on **50 sampled (start, goal) pairs** (Chamfer distance on 10 instances for the deformable ones). Table 1: Push-T 0.90 vs DreamerV3 0.30, IRIS 0.32, TD-MPC2 0.00. Table 2, the number that matters most for anyone picking a representation: Push-T with DINO **CLS** 0.44, ResNet 0.20, R3M 0.42, DINO **patches** 0.90. No standard errors, no real robot, no non-learned baseline.

## What I ran

W2: frozen DINOv2-S patches, learned dynamics, trained on the same 182 episodes. Lost to a disk failure; **W2′** retrained on 2026-10-23 (seed 3073, $0.65) reproduces its probes exactly.

**G1 probes.** Arm-pose R² **0.92** (W1: 0.57), gripper-open AUROC 0.999, grasp onset 0.995. The frozen patch representation encodes the arm far better than W1's end-to-end CLS latent, which is Table 2's CLS-vs-patch result on real frames.

**G1b, ranking.** Bar: top-1 ≥ 0.8. Result: **0.55–0.61** at every horizon. Fail. Open-loop error 0.62 of copy-last (W1: 0.36) — the better representation came with *worse* dynamics; action-corruption ratio only 1.13 (reacts less to wrong actions than W1).

**G2, latency.** A single decision (10 rollouts × 30 steps) on the Mac: W1 108 ms, **W2 1 814 ms**. The DINOv2 encoder alone is hundreds of ms per frame on an Orin. W3 (bigger) was not run because the decision rule said it would not change the build.

**G10 outcome readout.** AUROC 0.976 (W1 1.000).

## Their number, my number

| | Paper (sim) | This arm |
|---|---|---|
| Patch > CLS for state | 0.90 vs 0.44 (Push-T success) | R² 0.92 vs 0.57 (arm pose) |
| Planning success | 0.90 Push-T | top-1 0.55–0.61; no-model baseline 0.68–0.90 |
| Planning time | 47 s/episode (LeWM's measurement) | 1.8 s per decision on a Mac; off the Orin |
| Pairwise accuracy | not reported | 0.92–0.93; top-1 ≈ pairwise⁷ |

## What the gap means

The representation result transfers and the planning result does not, for the same reason as W1: nothing in the paper is compared to a planner with no model. The one new thing W2 taught: a better frozen encoder does not give better dynamics for free; the learned predictor on top of DINOv2 patches reacted *less* to wrong actions than the small end-to-end model. And DINO-WM-style online planning is off the table on an Orin on latency alone; the role is offline ranking or verification, where W2′'s patch tokens are the ones to use for anything object-level (G7).

**Verdict:** representation reproduces; planning fails the baseline; the encoder cost rules out the loop.

**What would change it:** an object-state test (G7) where the patch representation's spatial detail is the thing being measured, scored offline.
