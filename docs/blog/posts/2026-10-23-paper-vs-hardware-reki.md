---
date: 2026-10-23
title: "Paper vs hardware 14: REKI"
slug: paper-vs-hardware-reki
---

# Paper vs hardware 14: REKI

REKI (Marrie, Pacaud, Ramanana Rahary, Chen, Böhle; Kyutai / Inria; arXiv 2610.22178) gives a VLA a memory through register tokens and measures the oscillation it removes. I voided my own latency measurement of it and then found the same oscillation in my policies with their metric.

<!-- more -->

## What the paper claims, and how it measures it

A 2.4B VLA (Qwen3.5-2B + a flow-matching expert) with **32 register tokens per frame** in the expert: they cross-attend the frame's context, self-attend the registers of the K − 1 previous frames (K = 3), never read the action tokens, and are cached as keys/values across chunks — so memory costs +0.1 ms. Pretrained on public data only; 21.8 ms per chunk on an H100. The augmentation detail that matters for a webcam rig: random camera dropout and shuffled camera order, worth > 20 points under viewpoint change.

Evaluation: LIBERO-Plus 90.4; RoboTwin Clean2Rand 65.8; RoboCasa365 38.4; baselines quoted from their papers. The memory ablation (Fig. 5a): a K = 3 model evaluated at K = 1 scores like a K = 1 model — the gain comes from *reading* past frames at inference, mostly the first one. The oscillation result (Fig. 5b): an episode oscillates if an arm **reverses direction at three or more consecutive chunks**; **4.1 % with memory, 14.8 % when the same model is evaluated without it, 18.6 % trained without**; two-arm tasks 24.5 → 3.1; RoboTwin, 50 tasks, **20 matched seeds each**. Mechanism offered: perceptual aliasing — two poses each look like demonstration states heading toward the other.

## What I ran

**G12 (2026-10-18), voided the same day.** The bar was memory ≤ 8 GB and chunk ≤ 250 ms on the Orin; the script reported 3.46 GB and 318 ms. The re-check found it timed a random-init Qwen2.5-1.5B with no vision tower and a hand-written random expert; the parts did not sum (135 + 230 ≠ 318). REKI's Orin latency is **unmeasured**.

**G14 (2026-10-18).** REKI's oscillation criterion on my 69 published rollouts: reversal at ≥ 3 consecutive chunks, cosine < −0.5, > 3° per chunk. **Failed episodes 6/25 = 24.0 %** (CI 11.5–43.4 %); succeeded 8/44; **all episodes 14/69 = 20.3 %**, which is the number comparable with REKI's (theirs is over all episodes). Trajectory check: the flagged failures are real re-grasp loops and pan dithering; two unflagged loops were missed. All my policies are memoryless.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Oscillation, no memory | 18.6 % (trained without), 14.8 % (evaluated without) | 20.3 % of all episodes (24.0 % of failed) |
| With memory | 4.1 % | not tested |
| Criterion | reversal ≥ 3 consecutive chunks, 20 seeds/task | same criterion, 69 episodes |
| Latency on Orin | — | void; unmeasured |

## What the gap means

G14 is the cleanest external replication of a mechanism in my register: a memoryless policy on a SO-101 dithers at about the rate a memoryless 2.4B VLA dithers on RoboTwin, under the same definition. What I cannot yet say is whether memory fixes it here, because adding registers to SmolVLA means retraining. The paper's Fig. 5a points at the cheap half: for any policy that has a past-frame cache, the same checkpoint evaluated with and without it is a free experiment. SmolVLA has none, so the port is the only route, and it costs $5–10 plus a robot session; the amendments put the free comparison and the real latency measurement before it.

**Verdict:** the oscillation mechanism reproduces; the fix is untested; the latency gate was void and is to be redone.

**What would change it:** a real REKI chunk timed on the Orin with the released checkpoint, then 32 registers in SmolVLA's expert against the 20.3 % line.
