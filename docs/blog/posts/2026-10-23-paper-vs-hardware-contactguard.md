---
date: 2026-10-23
title: "Paper vs hardware 6: ContactGuard"
slug: paper-vs-hardware-contactguard
---

# Paper vs hardware 6: ContactGuard

ContactGuard (Zheng, Johnson-Roberson, Zhi; arXiv 2608.13438) is the closest paper to what I run: a LeWM-style world model used as a pre-contact failure monitor on a real arm. Their monitor works. Mine, built to the same protocol, does not, with the opposite sign.

<!-- more -->

## What the paper claims, and how it measures it

Shared ViT-tiny per camera, **mean-pool three views** then a linear projection → one latent; predictor of four AdaLN-zero blocks; LeWM loss trained teacher-forced on unlabelled rollouts *and* teleop; then frozen. The probe: anchor 15 frames (0.5 s) before the first gripper-closure command, roll out **K = 30 steps** under the policy's actual chunk, standardise the imagined latent, fit a class-balanced ℓ2 logistic regression on ~250 labelled grasp attempts per task, freeze one threshold per task on validation. Online: trigger on the first open→close in the upcoming chunk, abort before contact if P(fail) > τ.

Evaluation on an AgileX Piper with an ACT policy, four grasp settings, test episodes excluded from everything. **Closed loop**: 50 monitored live rollouts per setting with a 5-s pause-and-resume so the would-be outcome is recorded; AUC 0.99 / 0.95 / 0.90 / 0.92 vs a single-view LeWM and a no-rollout probe. **Offline pools** (n = 101–144, five seeded 5-fold CV): ContactGuard 0.98 vs **current latent 0.84** — same encoder, labels, probe, pools, only the imagined future replaced by the present one — with paired bootstrap CIs excluding zero on every task. **Action corruption** (Table 3): shuffled chunks from other episodes collapse the full model from 0.965 to 0.535; zero and mean actions to ~0.84. So the signal is the specific action's imagined consequence. A counterfactual action swap raises P(fail) by +0.25 to +0.65. Proprioception added to the probe *hurts* on all four (small-data shortcut). Runtime 19 ms on a 5090.

## What I ran

**W1-C1 (2026-10-19).** The protocol as written: anchor 15 frames before the first close, 30-step imagination under the executed chunk, logistic probe, **current latent as the matched baseline**, 5-fold × 5 seeds on the 69 published rollouts (65 scored; two stalls and two others never close). Bar: AUROC ≥ 0.85 and ≥ 0.05 above the current latent. Result: imagined **0.70** (folds 0.72 ± 0.13), current **0.82** (0.83 ± 0.11). Imagination is 0.12 worse. The fold threshold gives 27 % held-out false aborts.

W1 itself follows the multi-view ContactGuard recipe (mean-pooled views), which is why the comparison is as direct as it gets.

## Their number, my number

| | Paper (Piper, ACT) | This arm (SO-101, mixed policies) |
|---|---|---|
| Imagined-latent probe | 0.98 | 0.70 |
| Current-latent probe | 0.84 | 0.82 |
| Difference | +0.14, CI excludes 0 | **−0.12** |
| Labelled attempts | ~250 per task | 65 |
| Views | 3 | 2 |
| World-model training data | rollouts + teleop | teleop only |
| Arm | torque-controlled | position-controlled servos |

## What the gap means

The current-latent probes agree (0.84 vs 0.82): the encoder reads "this state looks risky" about as well on both arms. The imagined-latent probes do not: theirs adds information, mine subtracts it. The candidates, none proven: their world model saw the deployed policy's own rollouts in training and mine saw only teleop, so my imagination is off-distribution exactly at the states the monitor is for; 65 labels vs 250; two views vs three; and a torque-controlled approach that carries contact information a position-controlled one does not. The paper gives the control that would say which: Table 3's action corruption. If shuffled chunks do not push my imagined-latent AUROC below 0.70, the rollout carries no action information on my data and the encoder, not the probe, is the gap. That test is offline and free and is now in the amendments.

**Verdict:** does not reproduce; the matched baseline wins on my arm; mechanism unknown.

**What would change it:** the corruption test, then W1 retrained with policy rollouts in the mix.
