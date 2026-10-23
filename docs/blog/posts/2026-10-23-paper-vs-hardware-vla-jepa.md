---
date: 2026-10-23
title: "Paper vs hardware 13: VLA-JEPA"
slug: paper-vs-hardware-vla-jepa
---

# Paper vs hardware 13: VLA-JEPA

VLA-JEPA (Sun, Zhang, Qi et al.; arXiv 2602.10098) pretrains a VLA on human video through a latent world model. I trained it for $10, ran it on the arm at 9/10, and tested whether its eight samples were worth choosing between. The paper's honest ablation row says what to expect: robustness, not accuracy.

<!-- more -->

## What the paper claims, and how it measures it

A Qwen3-VL backbone with a flow-matching head. Latent-action tokens are read out of the VLM given the **initial frame only** plus the instruction — no future frame ever enters the VLM, which is the leakage-free principle — and an autoregressive latent world model predicts frozen V-JEPA 2 state chunks from them; L_WM is the teacher-forced prediction error. Robot data adds action tokens after the latent ones and a flow-matching loss; the joint objective is L_FM + β L_WM. Pretrained on Something-Something-v2 (220k human videos) and DROID; fine-tuned per benchmark.

Evaluation: LIBERO at **500 episodes per suite**, 97.2 vs π0.5 96.9 — and **96.1 without human video** (+1.1). SimplerEnv: Google Robot 65.2 with human video, **78.4 without** (the authors attribute the reversal to the real-to-sim gap). LIBERO-Plus, seven perturbation axes: 79.5 vs OpenVLA-OFT 69.6, and **62.9 without human video** (+16.6, the number the paper is about). Real Franka, 100 demos, three tasks: 0.70 vs π0 0.57 in distribution. No intervals; ±1 on LIBERO is inside binomial noise at this n.

## What I ran

**Training.** Qwen3-VL-2B + DiT-B, world model off at inference, ~$9.7 on vast.ai over two days.

**The bowls gate.** 9/10 on the Orin NX at 355 ms per chunk (PyTorch); in the twelve-VLA latency benchmark through vla.cpp, 177 ms minimum.

**H0 (2026-10-08).** The question the policy's design invites: do its 8 flow samples vary enough that picking one with a world model would help? Bars: oracle/mean error ≤ 0.7, latency at N = 8 ≤ 1.5×, baseline success < 85 %. Result: 0.76 (null 0.92); **1.84×** (643 vs 349 ms); baseline **90 %**. No-go. Averaging the eight chunks got 45 % of the oracle's gain with no model at all. Reranking was dropped the same day, and with it the first architecture version.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| In-distribution success | +1.1 over no-human-video (LIBERO) | 9/10, same as ACT and SmolVLA (ceiling) |
| Perturbation robustness | +16.6 (LIBERO-Plus) | not tested |
| Sample diversity useful? | not asked | oracle/mean 0.76; mean-of-8 gets 45 % of it free |
| Latency | 108.7 ms (4090) | 355 ms PyTorch / 177 ms vla.cpp on Orin NX |

## What the gap means

The paper's own ablation rows say the human-video pretraining buys robustness to perturbation, not nominal accuracy, and sometimes costs accuracy (SimplerEnv). A nominal gate at ceiling, which is all I ran, cannot see that and did not. If I ever run a perturbation gate — lighting, background, camera — this paper predicts which policy survives and why; a nominal gate showing no difference would confirm it, not refute it. The H0 result is mine, not theirs: the samples do not differ enough to be worth a chooser, and the cheapest chooser (the mean) is half as good as an oracle for free. That one afternoon ended a design.

**Verdict:** trains and runs as advertised; the robustness claim is untested here; sample reranking is a no-go.

**What would change it:** a perturbation gate against SmolVLA on the same layouts, after the camera question (SEVO) is settled.
