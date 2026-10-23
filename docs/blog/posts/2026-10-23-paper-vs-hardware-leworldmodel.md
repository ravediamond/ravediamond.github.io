---
date: 2026-10-23
title: "Paper vs hardware 1: LeWorldModel"
slug: paper-vs-hardware-leworldmodel
---

# Paper vs hardware 1: LeWorldModel

*One paper, one arm. This series takes each paper I actually tested on a SO-101 + Jetson Orin NX, states what the paper claims and how its authors measured it, puts my number next to theirs, and says what the gap means. Numbers are from `journal/EXPERIMENTS.md`; gate IDs are the register's.*

LeWorldModel (Maes, Le Lidec, Scieur, LeCun, Balestriero; arXiv 2603.19312) is the recipe behind W1, the world model that runs on my arm. Four gates tested it: dynamics (G0), planning (G1b), progress (G6a), outcome readout (G10).

<!-- more -->

## What the paper claims, and how it measures it

A 15M-parameter JEPA trained end-to-end from pixels: a ViT-tiny encoder (patch 14, 12 layers, width 192, ~5M) whose CLS token becomes one 192-d latent per frame; a 6-layer predictor with AdaLN zero-init action conditioning (~10M); loss ‖ẑ_{t+1} − z_{t+1}‖² + 0.1 · SIGReg, where SIGReg pushes the batch of latents toward an isotropic Gaussian by testing 1024 random 1-d projections for normality. No EMA target, no stop-gradient; SIGReg alone prevents collapse. Planning is CEM with a terminal cost ‖ẑ_H − z_g‖² inside MPC.

Evaluation is simulation only: Push-T, Two-Room, Reacher, OGBench Cube. The headline is **success at matched training FLOPs** (Fig. 6): Push-T 90 % vs DINO-WM's 13 % at the same compute, planning 48× faster (0.98 s vs 47 s). Table 1 is a linear-probe table: a linear readout of the latent predicts Push-T block location with r = 0.986 and block angle with r = 0.902. There is no real robot, no non-learned planner baseline, and the authors list "short planning horizons" as the limitation.

## What I ran

W1 is LeWM's recipe on 182 teleop episodes from two webcams (multi-view variant, ContactGuard-style mean-pool), 100 epochs, $3.30 on vast.ai.

**G0, dynamics.** Bar: beat copy-last; every action-corruption ratio ≥ 1.3 at K = 30; no collapse. Result: open-loop error at K = 30 **0.31 vs copy-last 0.89**; timing-shift ratios 1.99 / 1.58; effective rank 24–30 of 192 (the 5-epoch run was 5.8). Pass. The model learns real action-conditioned dynamics.

**G1b, planning as ranking.** Bar: rank the expert chunk first among 8 candidates at top-1 ≥ 0.8 at some horizon. Result: **best 0.60** at d = 20–30. Fail. The diagnosis (G1 diagnosis row) found why: an action-only baseline with no model at all, ranking by commanded-joint distance, scores **0.68–0.90**. On a position-controlled arm the commanded joints already determine the outcome of a 7-step chunk; the latent adds noise.

**G6a, progress from latent distance.** Bar: Spearman ≤ −0.8 between latent distance to goal and time. Result: −0.32 to −0.50. Fail. Latent distance is not progress here (the TD-JEPA paper, 2607.25337, finds the same gap in simulation: Spearman 0.65 for LeWM's Euclidean distance vs step count).

**G10, outcome readout at rest.** Bar: AUROC ≥ 0.90 on success/failure from the final latent. Result: **1.000**. Pass. Table 1's probe result transfers: the latent contains the task outcome.

## Their number, my number

| | Paper (sim) | This arm |
|---|---|---|
| Dynamics learned | yes (VoE test, qualitative) | yes: 0.31 vs 0.89 copy-last |
| Linear probe of state | r 0.986 (block location) | arm-pose R² 0.57 (W1), outcome AUROC 1.00 |
| Planning success | Push-T 90 % at matched FLOPs | top-1 0.60 vs no-model 0.68–0.90 |
| Baseline without a world model | not run | run; it wins |

## What the gap means

Not that the paper is wrong. The paper's planner is compared to other learned planners at equal compute; it is never compared to "do nothing clever". On a cheap arm the clever-nothing baseline is strong because the commanded trajectory *is* the plan, and the latent planner has to beat it, not DINO-WM. The honest role for the model here is the one G10 passed: a verifier at rest. The planning-limits paper (2609.39235) later showed with a real simulator as a perfect predictor that terminal-latent-distance scoring against a distant goal falls from 92 % to 41 %; G1b scored against a goal many chunks away, so it sat in exactly that regime. A subgoal inside the rollout is the fix the paper proves; it has not been tried here yet.

**Verdict:** dynamics and readout reproduce; planning does not survive the baseline the paper did not run.

**What would change it:** G7 (object-state ranking where the joints are blind), with a near subgoal. That gate is frozen and waiting for a cube-rollout session.
