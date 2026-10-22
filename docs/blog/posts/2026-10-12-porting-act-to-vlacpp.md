---
date: 2026-10-12
title: Porting ACT to vla.cpp, so a Jetson can run it without PyTorch
slug: porting-act-to-vlacpp
---

# Porting ACT to vla.cpp, so a Jetson can run it without PyTorch

ACT is the most common SO-101 policy and vla.cpp did not have it. Merged as vla.cpp `#38` after a scoping issue (`#37`) the maintainer approved in a day. The pitch was never speed: PyTorch ACT already runs at 59 ms per 100-action chunk on the Orin. The pitch is **no PyTorch at inference**: no 5.5-hour source build, no 12–24 GB image, and CPU-only boards can run it.

<!-- more -->

## What a port involves

`CONTRIBUTING.md` lists the seven mechanical sites for a new architecture. Most of ACT's pieces already existed in-tree:

- An ACT-style transformer decoder with learned queries, from the TurboVLA implementation.
- Conv + group norm via `im2col`, from Octo. ACT's ResNet-18 uses frozen batch norm, which folds into the conv weights at conversion time.
- No flow-matching steps: ACT goes on the `num_steps` refusal list. The VAE encoder is training-only, so the latent is zeros at inference.

The converter and `act.cpp` read state and action dims, camera count and resolution from the checkpoint config rather than hard-coding the SO-101, because the maintainer asked for a parity check on `lerobot/act_aloha_sim_transfer_cube_human` as well: bimanual, 14-dim, one 480×640 camera.

## How it was tested

- `vla_predict_check` against PyTorch on fixed inputs for both checkpoints (ours and the ALOHA one).
- The same gate harness as every other policy here, with an `--engine vlacpp` path for ACT, on the real arm. The PyTorch checkpoint scores 9/10 on the Orin NX.
- An `--arch act` client preset in the LeRobot-side client fork, so the same client drives ACT and SmolVLA (no tokenizer, ImageNet normalization).

## Why a hobby lab should do ports

A port is the one contribution nobody can casually redo without your hardware, and it forces you to read the model as code rather than as a paper. The same reading is what later turned up a silently wrong objective in another policy's training code (LeRobot `#4381`) and the chunk-boundary fix that went into this engine a week after (`G13b`).
