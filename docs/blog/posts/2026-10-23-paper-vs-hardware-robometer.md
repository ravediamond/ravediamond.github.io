---
date: 2026-10-23
title: "Paper vs hardware 10: Robometer"
slug: paper-vs-hardware-robometer
---

# Paper vs hardware 10: Robometer

Robometer (Liang, Korkmaz, Zhang, Hwang et al.; arXiv 2603.02115) is a 4B reward model trained on a million trajectories. I tested it zero-shot as a judge on my own rollouts and on a 2 099-rollout public SO-101 benchmark. Episode length beat it on every held-out task. The paper never computes episode length.

<!-- more -->

## What the paper claims, and how it measures it

Qwen3-VL-4B with learned tokens: a progress token after each frame (causal; progress in 10 bins, success at the last frame) and a preference token after two videos. Three losses: preference, progress, success. Pairs are built from expertise (expert vs failure), **instruction negatives** (same frames, wrong instruction → progress 0), and video rewind. Data: RBM-1M, 21 embodiments; **SO-101 is not in it**.

Evaluation on RBM-EVAL (976 OOD trajectories): **VOC** (Pearson r between predicted reward and frame index on successes) 0.92 and **Kendall τ** on the failed < suboptimal < successful ordering 0.64, vs RoboReward-8B 0.82 / 0.47. Zero-shot on RoboFAC, VOC 0.65. Ablations on LIBERO-90: adding failure data lifts τ from 0.74 to 0.92. Downstream: online RL on a DROID π0 base 20 → 85 % (**20 trials**); offline IQL on SO-101 "2.4× the best baseline" (0.85 vs 0.50 on an oven task; 0.15 vs 0.05 on a plate task — 3 vs 1 successes in 20); failure detection on 100 DROID trajectories, F1 0.81 vs GPT-5-mini 0.33. The authors' own limitation: no access to contact or grasp stability.

## What I ran

**G9a-AB (2026-10-15/16).** Zero-shot on ArmnetBench, 2 099 rollouts, 7 policies, top camera. Bar: pooled AUROC ≥ 0.85 and within-policy ≥ 0.8 on ≥ 5/7. Complete set: pooled **0.818** [0.790, 0.843]; within-policy 5/7; **episode length alone 0.938**; beyond length (quintile median 0.687, CV gain +0.016) both bars missed. Fail: not a zero-shot judge.

**G9a-own (2026-10-16).** On my 88 labelled bowls rollouts: AUROC **0.577** [0.44, 0.70]; TPR 0.25 at the calibrated threshold; policy identity 0.878; **wrong-object failures scored highest**. Fail.

**G9b (written 2026-10-22).** The LoRA fine-tune on held-out-task folds never trained (the job died at preprocessing). The zero-shot baseline on policy rollouts only: stack 0.564, tool 0.616, cable 0.761, ring 0.933; **length 0.903 / 0.855 / 0.946 / 0.965, better on 4/4**; CV gain over length +0.008 [−0.001, 0.017]. No verdict.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Ranking quality | VOC 0.92, τ 0.64 (OOD embodiments) | AUROC 0.82 on ArmnetBench, 0.58 on own bowls |
| Episode-length baseline | not run | 0.938 / 4-of-4 held-out tasks |
| Wrong-object failures | not analysed | scored highest |
| Downstream n | 20 trials | — |

## What the gap means

Robometer and my result agree once the question is phrased right. The model answers "how far along does this look" and does that well (VOC 0.92 is real). My failure mode is identity — the wrong bowl, nested — and a trajectory-comparison objective with instruction negatives at the dataset level has no reason to catch it at the frame level on an embodiment it never saw. The paper's own "pull tissue" and "fold towel" rows (F1 0.58, 0.76) are its version of the same gap. The number that should be in every reward-model paper and is in none of the ten I read: **episode length**. On my tasks it is the judge every learned judge must beat first.

**Verdict:** fails as a zero-shot judge on this arm; the gap is the task's, not the model's.

**What would change it:** an instruction-negative test on my own frames (swap the instruction, fix the frames, does the score move?) before any LoRA; and labels for suboptimal-but-successful episodes, which τ needs and I do not have.
