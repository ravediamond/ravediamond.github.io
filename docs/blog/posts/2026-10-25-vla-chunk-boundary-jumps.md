---
date: 2026-10-25
title: "Why your VLA's actions jump at every chunk boundary on real hardware, and a 0.4 ms fix"
slug: vla-chunk-boundary-jumps
---

# Why your VLA's actions jump at every chunk boundary on real hardware, and a 0.4 ms fix

Run a chunked VLA asynchronously on a real arm and the command jumps every time a new chunk takes over. On my SO-101 with SmolVLA, the step at the switch was a median 3.4 times a normal step, and on a test boundary served by the real engine the jump was 58.7°. A gradient-free prefix overwrite inside the denoising loop brings the jump down to the size of a normal step, adds 0.4 ms per chunk on a Jetson Orin NX, and leaves the default path bit-identical. It has not been tested for task success on the arm yet. This post covers what I measured, and what I did not.

<!-- more -->

## The symptom

SmolVLA predicts a chunk of 50 actions at a time. Run synchronously, the arm waits for every chunk. Run asynchronously, the next chunk is computed while the current one is still executing, and the arm switches to it when it arrives.

I measured the switch offline on recorded episodes: 20 episodes of `ravediamond/so101_bowls_all`, 254 boundaries per setting, inference delays of d = 8, 12 and 20 control steps. The metric is the jump ratio: the size of the joint step at the switch divided by the median step inside the chunk. 1.0 means the switch looks like any other step.

Naive asynchronous execution gave a median jump ratio of **3.39 / 3.38 / 3.68** (d = 8 / 12 / 20) through vla.cpp, and 3.42 / 3.69 / 3.77 in PyTorch. On a test boundary served by the real `vla-server` (episode 2, d = 8), the jump at the switch was **58.7°**. With the fix it was **1.4°**.

## Why it happens

The new chunk is sampled from the observation at request time. By the time it arrives, d steps have passed and the arm has executed d more actions of the old chunk. The new chunk also starts from fresh noise. Nothing in the sampling ties its first rows to the actions that are still executing, so its row d can pick a different trajectory from where the old chunk left the arm.

```
chunk k      |== executed ==|-- d steps of inference --|-- leftover (not executed) --|
                            ^ request (state at t)     ^ chunk k+1 arrives
chunk k+1                   |-- rows 0..d-1 (skipped) -|-- row d onward executes --->
                                                       ^ switch: row d of k+1 vs last executed row of k
```

The leftover is the part of chunk k that would have run next. Naive execution throws it away and jumps to row d of chunk k+1.

## Budget first

None of this matters if the chunk does not arrive in time. With 50-action chunks and a new chunk started after 30 actions, the next one has to arrive within H − s = 20 steps, about 667 ms at 30 Hz.

SmolVLA in PyTorch on the Orin NX took **974 ms** per chunk, so it does not fit with any correction. In the offline study, naive PyTorch was 1028 ms, the gradient-free variants about 1032–1039 ms, and RTC's exact guidance 1962 ms. Served by vla.cpp with `--flash-attn --mm-prec default`, the same checkpoint takes **238 ms** per chunk, inside the budget with room to spare. So the fix had to live in the engine.

## The fix

Inside SmolVLA's Euler denoising loop, I correct the predicted clean chunk towards the leftover at every step. With x_t the current sample, v the velocity and t the flow time:

```
x1 = x_t − t·v
v ← v − W ⊙ (Y − x1) / t
```

- **Y** is the leftover, normalised with the checkpoint's action stats.
- **W** is Real-Time Chunking's soft prefix weight (Black, Galliker, Levine; arXiv [2506.07339](https://arxiv.org/abs/2506.07339)). It is 1 on the first d rows and decays as `c·expm1(c)/(e − 1)` to 0 at the end of the leftover, over the real action dimensions.
- Without a leftover, **W = 0** and the update does nothing. With fixed inputs and noise, the default path is bit-identical to the existing build (max |Δ| = 0.0), and vla.cpp's 14 tests pass on the Orin.

There are no gradients. `prev_actions` (the not-yet-executed rest of the previous chunk) and `inference_delay` are new fields in the engine inputs, the C API, the Python bindings and the server's request. The change is on my fork, branch [`feat/smolvla-prefix-overwrite`](https://github.com/ravediamond/vla.cpp/tree/feat/smolvla-prefix-overwrite), commit [`805570d`](https://github.com/ravediamond/vla.cpp/commit/805570d), on top of [vla.cpp](https://github.com/VinRobotics/vla.cpp).

## Results

Medians over 254 boundaries per cell, 20 episodes, degrees, five arm joints. Prefix error is how far the first d rows of the new chunk are from the leftover. Drift is the distance to the recorded actions, a check that the correction does not drag the chunk away from the demonstrations.

| | d | jump ratio | accel ratio | prefix error | drift |
|---|---|---|---|---|---|
| vla.cpp naive | 8 / 12 / 20 | 3.39 / 3.38 / 3.68 | 3.63 / 3.91 / 4.63 | 1.60 / 1.69 / 1.88 | 1.51 / 1.55 / 1.51 |
| **vla.cpp overwrite** | 8 / 12 / 20 | **0.98 / 0.90 / 1.05** | 1.21 / 1.19 / 1.29 | **0 / 0 / 0** | 1.50 / 1.50 / 1.45 |
| PyTorch naive | 8 / 12 / 20 | 3.42 / 3.69 / 3.77 | 3.81 / 4.13 / 4.27 | 1.65 / 1.69 / 1.91 | 1.48 / 1.53 / 1.53 |
| PyTorch overwrite | 8 / 12 / 20 | 0.93 / 0.96 / 1.06 | 1.23 / 1.26 / 1.34 | 0 / 0 / 0 | 1.49 / 1.55 / 1.49 |

- **The jump drops by 71.0 / 73.3 / 71.5 %** through vla.cpp (72.8 / 74.0 / 71.7 % in PyTorch). The switch now looks like an ordinary step.
- **Drift is unchanged.**
- **The prefix matches the leftover exactly**: max prefix error 0.0 over 762 boundaries.
- **Cost: +0.4 ms per chunk**, measured as 371.0 → 371.4 ms wall per call from Python. That includes copying two 512² images; the engine's own inference is the 238 ms above.

Against the real server, the first 8 rows of the corrected chunk match the leftover to 1e-6°. The naive chunk was off by up to 47° on the same rows.

## How it differs from RTC

RTC solves the same problem with guidance: one vector-Jacobian product through the policy per denoising step. On π0.5 it costs 14 → 35 ms for 5 denoising steps, 76 → 97 ms per chunk (RTC, App. A.3, Table 3). In vla.cpp that path is not available today, because ggml lacks backward rules for flash attention and concat.

RTC does test an inpainting variant in its appendix (A.4, Fig. 8). That variant overwrites the *noisy* prefix with forward-noised committed actions under a hard mask on the first d rows. It ran in simulation only (Kinetix), gave "some benefit" over naive async, and was beaten by guided RTC on solve rate. RTC's real-robot runs compare guided RTC, synchronous, naive async and temporal ensembling only; no overwrite variant ran on hardware.

Mine is a different variant. It corrects the predicted *clean* chunk with RTC's soft mask over the whole overlap, which amounts to guidance with an identity Jacobian at weight 1/t. So the Kinetix ranking does not transfer as is, in either direction.

## What is not proven

This is smoothness, measured offline on recorded episodes through the real engine. **Task success on the arm with the overwrite has not been tested.** That test is registered as G13a and has not run.

Forcing the new chunk to agree with the old one could hurt success: if the old chunk was heading the wrong way, the new one now follows it for d steps. If G13a shows that, the fallback is training-time action conditioning (arXiv 2512.05964), which fine-tunes the policy to take a clean prefix instead of correcting it at inference.

## Try it

The model is the SmolVLA checkpoint `ravediamond/smolvla_2026-09-24_14-34-40` (step 20000), converted to GGUF with vla.cpp's `scripts/convert_smolvla_to_gguf.py`. The data is [`ravediamond/so101_bowls_all`](https://huggingface.co/datasets/ravediamond/so101_bowls_all). Build the branch on the Orin with `-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=87 -DCMAKE_BUILD_TYPE=Release -DVLA_SPM=OFF`. The boundary study ran as:

```bash
VLA_LIBRARY=~/workspace/git/vla.cpp/build-orin/libvla.so PYTHONPATH=~/workspace/git/vla.cpp/bindings/python \
  python scripts/check_vlacpp_prefix_overwrite.py \
  --gguf ~/vla-models/smolvla-so101-bowls-020000.gguf \
  --episodes-from shared/gate/data/2026-10-19_g13b_offline/g13b_summary.json --out outputs/vlacpp_overwrite
```

The check script and the per-boundary rows live in my research repo, which is not public yet.

Every paper I test this way, with its bar and verdict, is on the [paper vs hardware page](https://ravediamond.github.io/papers/).
