---
date: 2026-10-23
title: "Paper vs hardware 8: ReAP"
slug: paper-vs-hardware-reap
---

# Paper vs hardware 8: ReAP

ReAP (Liu, Qing, Wei, Hou; arXiv 2610.16309) trains a failure-supervised visual space and an action-conditioned risk predictor on top of it. My "ReAP-style risk head" is the closest any monitor got to my bar — and it is a stripped version with two confounds the paper's own design removes.

<!-- more -->

## What the paper claims, and how it measures it

Stage 1, a detector: frozen DINOv2 patches from each camera → Perceiver-style cross-attention from 32 learned queries → a compact state z_t → a risk head, trained with class-weighted BCE against a **ramp target**: 0 on successes, t/(T−1) on failures. No onset labels. Stage 2, a predictor: the policy's action chunk cross-attends the frozen latent, a residual MLP predicts the latent H steps ahead, and the frozen head reads risk from it; loss is latent-transition error plus risk BCE. Alarm = detector OR predictor, each with its own conformal threshold on held-out successes.

Metric **bTWA**: half earliness-weighted recall (a failure detected at t_i of T_i scores 1 − t_i/T_i) plus half specificity; averaged over five α; mean ± std over three seeds. Offline: LIBERO-10 and RoboTwin-8 with four policies; baselines from three families (action uncertainty, OOD, classifiers). ReAP 0.70–0.78 vs best baseline 0.58–0.76. Real: three π0.5 tasks, monitor live, alarms logged without acting. The ablations are the valuable part: a **random, untrained** detector scores 0.650 vs the learned 0.683 and mean-pooled DINOv2 beats every baseline alone (Table 3a); **shuffled chunks** from other episodes of the same task drop 0.683 → 0.618, zero actions → 0.52 (Table 3b). Detector latency 9.7 ms, no policy inference needed.

## What I ran

**REAP monitors (2026-10-23).** On W1-A's 50 episodes (42 ok / 8 fail), 5-fold × 5 seeds: W1 L2 surprise, a logistic or MLP risk head on [W1 latent, 30 commanded actions, step], elapsed time. Bar: ≥ 5/8 failures at ≤ 1/20 false interventions. Result: surprise 2/8 at 2/42 (AUROC 0.86); **logistic head 6/8 at 7/42 (AUROC 0.93)**; MLP 7/8 at 13/42; elapsed 0/8. On all 69: logistic 23/25 at 4/44 (0.97), but 17 of the extra failures are TurboVLA's. Verdict: **not proven**. Two confounds: the secondary set's failures are mostly one policy, and the onsets were labelled by Gemini with two misses of 110 and 289 frames. Latency on the Orin: W1 tick 17.6 ms, 25-step imagination 153 ms off the control thread, risk head 18 ms; the policy's chunk slows 238 → 304 ms with the monitor sharing the GPU.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Representation | failure-supervised Perceiver over DINOv2 | W1's prediction-trained latent |
| Target | ramp, no onsets | Gemini onsets + step feature |
| Action control | shuffled chunk: 0.683 → 0.618 | not run |
| Best result | bTWA 0.70–0.78 | 6/8 at 7/42 (bar 5/8 at 2/42) |
| Latency | 9.7 ms detector | 17.6 ms W1 + 18 ms head |

## What the gap means

Table 3a says the encoder is not where my gap is: a random projection of DINOv2 is within 0.03 of their learned one, so W1's latent is probably fine. The two things their design has and mine lacks are exactly my two confounds. The **ramp target** removes the onset labels from the verdict — the 25 hand labels I owe become a validation set instead of a prerequisite. The **shuffled-chunk control** decides whether "action-conditioned" belongs in the name: train once, evaluate with other episodes' chunks; if the number does not drop, the head is a state monitor. Both are offline and free and are in the amendments; bTWA goes beside my bar so the comparison is on their metric too.

**Verdict:** closest to the bar of anything tried; not proven; two confounds with published fixes.

**What would change it:** the ramp retrain and the shuffled-chunk control, then the 25 onsets as a check.
