---
date: 2026-10-23
title: "Paper vs hardware 15: Real-Time Chunking"
slug: paper-vs-hardware-real-time-chunking
---

# Paper vs hardware 15: Real-Time Chunking

Real-Time Chunking (Black, Galliker, Levine, Physical Intelligence; arXiv 2506.07339) treats the next action chunk as an inpainting problem. My gradient-free prefix overwrite is the paper's "hard mask" line, and the paper says where it should lose.

<!-- more -->

## What the paper claims, and how it measures it

With chunk length H, control period Δt and inference time δ, the delay is d = ⌊δ/Δt⌋ steps. Asynchronous inference must start the next chunk d steps early, and the junction is arbitrary: a new chunk from fresh noise may pick a different strategy (their bifurcation figure). Temporal ensembling averages across chunks and is not guaranteed valid when the distribution is multimodal. RTC: freeze the first d actions (they will be executed before the new chunk arrives), attend to the remaining overlap with an exponentially decaying weight, leave the rest free, and enforce it through ΠGDM guidance on the flow's velocity field with a clipped guidance weight — one vector-Jacobian product through the policy per Euler step. The soft mask (eq. 5): weight 1 for i < d, decaying over the overlap, 0 after.

Evaluation: twelve dynamic Kinetix tasks chosen because quasi-static benchmarks are saturated by open-loop chunking; **2 048 rollouts per point with Wilson 95 % intervals**; delays 0–4; baselines naive async, BID, temporal ensembling. RTC is above every baseline at every delay; **hard masking (freeze only d steps) is below soft masking at small d**; temporal ensembling is worst even at d = 0. Real: π0.5, bimanual, H = 50, Δt = 20 ms, 5 denoising steps, d ≈ 6, plus +100 / +200 ms injected latency; six tasks, **10 trials per task per method per delay**, metric **throughput** (task progress per minute). RTC is flat across added latency; synchronous falls linearly; both temporal-ensembling variants trip the protective stop at +100 ms and score zero. Cost: 97 vs 76 ms per chunk on their stack.

## What I ran

**G13b (2026-10-19, offline, Orin).** SmolVLA on 20 bowls episodes, 254 chunk boundaries, d = 8 / 12 / 20. Methods: naive, exact RTC guidance, and gradient-free prefix overwrite. Bars: propose the gradient-free method if it cuts the median jump ≥ 50 % vs naive and stays within 20 % of exact. Result: prefix overwrite cuts the median jump **72–74 %** at every d, drift unchanged; **exact RTC cut only 21–41 %** in this implementation (weaker than LeRobot's identity mode — flagged to check). Timing: naive 1 028 ms, overwrite 1 035 ms, exact 1 962 ms per chunk in PyTorch on the Orin; every method misses the 667 ms line because the policy itself is too slow in PyTorch.

**In vla.cpp (same day).** Ported as `prev_actions` + `inference_delay`; jump cut 71.0 / 73.3 / 71.5 % through the engine, prefix error 0.0, **+0.4 ms** per chunk, default path bit-identical. Exact RTC is not possible in vla.cpp today: ggml lacks backward rules for flash attention and concat.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Hard vs soft mask | soft > hard at small d (Fig. 5) | hard only; soft not run |
| Cost | +21 ms (VJP per Euler step) | +0.4 ms (overwrite); exact +930 ms in PyTorch |
| Metric | throughput, task progress/min | median boundary jump (instrument) |
| n | 2 048 rollouts/point; real 10/task/delay | 254 boundaries, 20 episodes |

## What the gap means

The overwrite is the paper's hard mask with the guidance weight sent to infinity on the frozen steps and zero elsewhere. The paper's Fig. 5 says that works and that soft masking beats it when d is small relative to H — which is my regime (d ≈ 7 of 50). That is a precise published prediction I have not tested, and testing it needs no engine work: implement eq. 5 in PyTorch, run the same 254 boundaries with shared noise, pair against the overwrite. Only if soft wins by more than the pairing noise are ggml backward kernels worth writing. And the result metric should be theirs: a smoother junction is for throughput; the jump is the instrument check, not the claim.

**Verdict:** the gradient-free point reproduces in spirit; the paper predicts a better point I have not measured.

**What would change it:** the soft-mask comparison offline, then throughput on the robot day with sync / async / async + prefix.
