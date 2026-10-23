---
date: 2026-10-23
title: "Paper vs hardware 4: H-JEPA"
slug: paper-vs-hardware-h-jepa
---

# Paper vs hardware 4: H-JEPA

H-JEPA (Zhang, Terver, Rabbat, LeCun, Balestriero; arXiv 2610.06805) stacks LeWorldModels across timescales. I tested its level-2 slow feature as a progress signal (G6b) and it failed. Reading the paper afterwards, the paper's own Fig. 5 predicts that failure.

<!-- more -->

## What the paper claims, and how it measures it

Level 1 is LeWM. Each level ℓ > 1 has its own state encoder (a pointwise MLP over the level below), its own action encoder pooling s_ℓ lower actions, and its own causal predictor, stride 2 per level, all trained end-to-end with a LeWM loss per level. On real data (DROID) an **inverse-dynamics term** is required or the model collapses to "encode the predictable background, drop the arm" (their App. L). Planning is top-down by gradient descent: the top level plans macro-actions; its predicted states become subgoals for the level below.

Evaluation: four simulated environments; success rate; **three paired model + planner seeds with SE**; planner compute swept and reported as **TFLOPs per episode**; depth 1–4 at a fixed 100-TFLOP budget on 50 tasks per environment. Baseline HWM (same hierarchy, shared latent) separates "temporal decomposition" from "distinct abstract spaces". AntMaze 18 → 73 % at three levels; two levels match LeWM on Push-T, three levels fail there because episodes are short. The mechanism figure: probe NMSE per entity vs level shows body pose becoming unrecoverable while position stays; the **entity-frequency gap** (ratio of spectral centroids of fastest and slowest state components) predicts when this selective abstraction appears — above ~2× it does; **Push-T, Cube and DROID have gaps below 2× and show none**. On DROID the metric is Fréchet fidelity of the planned end-effector path against the expert's (15 clips, three seeds). No closed-loop real robot.

## What I ran

**G6b.** A level-2 slow feature (stride 15, SIGReg loss) on W1, tested as a progress signal on the bowls task. Bars: T1 Spearman between latent distance and time ≤ −0.80; false-stall rate ≤ 10 %; sanity AUROC ≥ 0.95. Result: T1 **−0.73** (fail), false stalls **52 %** (fail), sanity 0.998 (pass). The hierarchy was dropped.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Where hierarchy helps | AntMaze (frequency gap > 2×): 18 → 73 % | bowls task: one arm, one object, ~1 Hz everything |
| Where it does not | Push-T, Cube, DROID (gap < 2×) | same regime |
| Progress monotonicity | subgoal-tracking Spearman 0.89–1.00 (Fig. 8) | level-2 distance vs time −0.73 |

## What the gap means

I tested the hierarchy on data the paper says it should not change. There is no frequency gap on a single-arm pick-and-nest task; the paper's own prediction for that regime is "no selective abstraction", and that is what G6b found. So the negative result and the paper's claim do not conflict; the gate was outside the claim. Two things do transfer: the inverse-dynamics term, which W1 does not need on single-scene data but W2′ on mixed lighting might; and **Fréchet fidelity** as an offline, open-loop plan-quality metric against expert paths — it costs nothing on my 69 rollouts and would have told me before G1b that the plans do not follow the demos.

**Verdict:** fail, outside the paper's regime; the paper's mechanism test (Fig. 5) is the better reading than my gate.

**What would change it:** a task with a real timescale gap (a tidy loop with navigation-scale structure), which my bench does not have.
