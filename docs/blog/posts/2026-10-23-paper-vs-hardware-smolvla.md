---
date: 2026-10-23
title: "Paper vs hardware 11: SmolVLA"
slug: paper-vs-hardware-smolvla
---

# Paper vs hardware 11: SmolVLA

SmolVLA (Shukor, Aubakirova, Capuano et al., Hugging Face; arXiv 2506.01844) is the policy behind almost every number on this site. Its paper's ablation tables are the manual for running it on a Jetson; three of them predicted things I later measured.

<!-- more -->

## What the paper claims, and how it measures it

A 450M VLA: SmolVLM-2 (SigLIP + SmolLM2) with the image at 512×512, no tiling, pixel-shuffled to **64 visual tokens per frame**; the state projected to one token *into the VLM prefix*; only the **first 16 of 32 LLM layers** run; a ~100M action expert alternating cross-attention (to the VLM's keys/values) and causal self-attention over the chunk, trained by conditional flow matching with 10 Euler steps, chunk 50. Only the expert is trained. Pretraining: 481 community SO-100 datasets, 22.9k episodes, with task strings rewritten by a VLM and cameras manually mapped to a consistent order.

Evaluation: LIBERO and Meta-World at **10 trials per task**; real SO-100 and SO-101 tasks with 50 demos each (10 per 5 start positions), scored by sub-steps; baselines π0 fine-tuned on the same data and ACT from scratch. Headline: LIBERO 87.3 for 0.45B vs π0 86.0; SO-100 real 78.3 vs π0 61.7 vs ACT 48.3; SO-101 LEGO 90 / 50 in-/out-of-distribution. The ablations: community pretraining is worth **+27 points** on SO-100 (Table 5); cross+self attention 85.5 vs cross-only 79.0; flow matching 80.3 vs L1 75.3; **state in the VLM prefix 80.3 vs in the expert 73.3** (Table 11); chunk 50 fine; and **executing 1 / 10 / 30 / 50 actions before re-observing: 80.3 / 82.8 / 70.8 / 51.8** (Table 13). Asynchronous inference keeps success (73.3 vs 78.3) and doubles throughput (19 vs 9 cubes in fixed time). No seeds, no intervals.

## What I ran

**The bowls gate (10 trials, operator-judged).** SmolVLA PyTorch on the Orin: 9/10 at **974 ms** per chunk. Through vla.cpp: 9/10 at 390 ms; with flash attention 8/10 at **238 ms**. Same weights, 4× faster, same task success. The task is at ceiling (ACT 9–10/10, VLA-JEPA 9/10), so it separates engines, not policies.

**Parity.** The GGUF against PyTorch on fixed frames and shared noise: median max |Δ| **0.80°** against a 1° line (noise-to-noise spread in PyTorch itself 12.2°).

**Chunk boundaries (G13b, G13b-in-vla.cpp).** With a 238 ms chunk at 30 Hz the inference delay is d ≈ 7 steps; a fresh chunk from fresh noise jumps at the junction. Gradient-free prefix overwrite cuts the median jump **72–74 %** offline and 71–73 % through the engine at +0.4 ms per chunk.

**Oscillation (G14).** 24 % of failed episodes reverse direction at ≥ 3 consecutive chunks.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Real-arm success | SO-101 LEGO 90 % ID | bowls 8–9/10 (ceiling) |
| Latency | not reported for Jetson | 974 ms → 238 ms (vla.cpp + FA) |
| Re-observe interval | Table 13: full chunk costs 29 pts | d ≈ 7, within the "free" band; boundary handled by prefix overwrite |
| Pretraining value | +27 pts | not re-measured; every fine-tune here starts from it |

## What the gap means

Nothing here contradicts the paper; the paper predicts the work. Table 13 is my boundary problem in the authors' own numbers: executing a whole chunk before re-observing is ruinous, re-observing every ≤ 10 steps is nearly free — *if* the junction is handled, which is what prefix overwrite and RTC are for. Table 11 is a checklist item for the plugin (state in the prefix, not the expert). Table 5 is the honest footnote under every fine-tune: the community pretraining is doing more than any architecture choice I could make. The one thing I measured that the paper does not is the engine: on an Orin the first 4× is engineering, and the control-loop research cannot start until it is done.

**Verdict:** reproduces; the ablation tables are the operating manual.

**What would change it:** nothing needs changing; the open question from this paper on my arm is the async stack's refill threshold g, which I run through vla.cpp and have not pre-registered.
