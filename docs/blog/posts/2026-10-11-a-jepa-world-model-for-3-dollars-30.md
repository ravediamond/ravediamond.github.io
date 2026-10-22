---
date: 2026-10-11
title: A JEPA world model from pixels for $3.30
slug: a-jepa-world-model-for-3-dollars-30
---

# A JEPA world model from pixels for $3.30

LeWM (the `stable-worldmodel` recipe: a JEPA trained end to end from pixels, no reconstruction) on the 182 bowls episodes, two camera views, 100 epochs. Gate G0 asked whether the model is real before anything is built on it.

<!-- more -->

## Getting it to train

Eleven setup problems between the paper's code and this data and hardware, on vast.ai x86 and on the Jetson GPU. $0.28 of smoke runs. A 5-epoch model reacted to gross action errors (episode swap, frozen action) but not to ±0.5 s timing shifts, and its latent had an effective rank of 5.8 out of 192: too short to judge.

## G0, the faithful run

Pre-registered bars: beat copy-last; every corruption ratio ≥ 1.3 at K = 30; no collapse.

- Open-loop error at K = 30: **0.31 vs 0.89** for copy-last.
- Timing shifts: ratios 1.99 and 1.58. The model uses the actions.
- Effective rank 24–30. No collapse.
- Linear probes on the latent: grasp onset AUROC 0.94, holding 0.995.
- Cost: $3.30, plus $2.96 of failed attempts before it.

Pass. This model, called **W1** from here on, is the one every later gate tests.

## C1, the first disappointment

Does the imagined latent predict rollout failures better than the current one? On the 81 gate rollouts: current latent 0.68, imagined 0.72, and **policy identity alone 0.76**. Most failures came from TurboVLA, so "which policy ran" explains more than any latent. Not proven, confounded. The lesson that stuck: compute the cheap baseline before the model, every time.
