---
date: 2026-10-08
title: "VLA-JEPA on the Orin for $9.7, and why best-of-N reranking is a no-go"
slug: vla-jepa-for-10-dollars-and-why-reranking-is-a-no-go
---

# VLA-JEPA on the Orin for $9.7, and why best-of-N reranking is a no-go

The first world-model idea was the obvious one: let the policy propose several action chunks, let a latent world model imagine each one, keep the best. Before building any of it, I checked whether the premise held. It did not.

<!-- more -->

## VLA-JEPA on the arm

Fine-tuned on vast.ai for about $9.7 (over the $3.5–4.7 estimate: interruptible bids cost more than they save on long runs). On the Orin NX, world model off at inference: **9/10 on bowls, 355 ms per chunk**. Same score as SmolVLA at three times the training cost. The task is at its ceiling and cannot tell them apart; that comparison waits for a harder primitive.

## H0: do the samples vary enough to pick from?

Pre-registered bars: oracle-to-mean error ratio ≤ 0.7, latency for 8 samples ≤ 1.5× one sample, baseline success below 85 %.

- The 8 samples are bundled: 1.08° spread, oracle/mean ratio 0.76 (null 0.92).
- Latency 1.84× (643 vs 349 ms).
- Baseline success 90 %, already above the bar where reranking could pay.
- **Averaging the 8 chunks, with no model at all, captures 45 % of the oracle gain.**

Verdict: no-go. A reranker cannot fix a bias the samples share, and the cheapest competitor (the mean) takes half the available gain for free.

## What changed

Reranking left the architecture the same day. The question became whether a world model can *see* a failure, not whether it can *choose* an action. That turned out to be a much better question, with a much longer answer.
