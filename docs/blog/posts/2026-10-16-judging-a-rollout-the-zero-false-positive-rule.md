---
date: 2026-10-16
title: "Judging a rollout: the zero-false-positive rule"
slug: judging-a-rollout-the-zero-false-positive-rule
---

# Judging a rollout: the zero-false-positive rule

Ten operator-judged trials are fine. An autonomous loop that records hundreds of rollouts and fine-tunes on the good ones is not, because every success it mislabels trains the policy on a failure. This is the day the judge became the most important component in the stack, and the day a published 4B reward model lost to 40 lines of OpenCV.

<!-- more -->

## Robometer-4B, zero-shot

On ArmnetBench (2 099 SO-101 rollouts, 7 policies), a first run on the partial set gave a qualified pass at AUROC 0.861. Re-scoring the 462 episodes lost to one CUDA OOM (two whole tasks, the longest episodes) brought the complete set to **0.818**: fail. Episode length alone scores 0.938. Missing data is not random, and a partial result is provisional until it covers every task.

On our own 88 rollouts: AUROC 0.58–0.62. The main failure mode (grasping the wrong bowl) is an object-identity question that a trajectory-comparison reward model has no reason to answer.

## G9c: a deterministic judge

Top camera, HSV masks for the two bowls, centroid geometry: is the blue blob inside the pink one at the end? No learning.

| Version | End rule | Accuracy | False positives |
|---|---|---|---|
| G9c | last frame | 86.4 % | **0** |
| G9c-2 | arm back at rest | 29/88 invalid (failing policies rarely return to rest) | 0 |
| G9c-3 | stillness or timeout | **94.3 %** | **0** |

All twelve G9c errors are successes whose recording continues past the operator's reset, so the final frame shows the bowls separated again. Judged at the marked reset instead: 95.5 %. G9c-3 misses its pre-registered 95 % bar by 0.7 points and is adopted anyway, because the one number that matters for a training loop is the false-positive count, and it is zero.

## The rule

A judge that never calls a failure a success can be slow, narrow and dumb. One that does, however clever, poisons the data. Everything later (the verifier tiers in v1.5) comes from this table.
