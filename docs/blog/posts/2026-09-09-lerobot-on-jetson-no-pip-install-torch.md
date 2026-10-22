---
date: 2026-09-09
title: "LeRobot on a Jetson: why there is no pip install torch, and what it took"
slug: lerobot-on-jetson-no-pip-install-torch
---

# LeRobot on a Jetson: why there is no `pip install torch`, and what it took

`pip install lerobot` fails on a Jetson Orin with an error that says almost nothing. The cause: LeRobot needs Python ≥ 3.12 and `torchcodec` ≥ 0.11 on aarch64 with CUDA, and no channel (NVIDIA's redist, the community index) ships a cp312 CUDA torch wheel for Jetson. The fix is a Docker image that builds the stack from source; it is merged upstream as LeRobot `#4599` and published on GHCR. This post is about the traps underneath it.

<!-- more -->

## Why not jetson-containers

The usual answer is `dusty-nv/jetson-containers`. Its last human-merged PR was months old and its own issue thread says it is no longer maintained; its `lerobot` package lacks `torchcodec`, references a renamed extra and a CLI that no longer exists. Depending on it would mean waiting on a lane with zero throughput.

## Five things built from source

1. PyTorch 2.11 for cp312, `TORCH_CUDA_ARCH_LIST=8.7`.
2. torchcodec 0.11 with CUDA, against a local NVDEC ffmpeg.
3. torchvision 0.26.
4. FFmpeg with `--enable-cuda-nvcc --enable-nvdec --enable-cuvid`.
5. dav1d, because ffmpeg 8.1 wants a newer one than Ubuntu 22.04 ships.

## The traps

- **PyTorch auto-enables NVSHMEM** once `USE_DISTRIBUTED=1` finds the `nvidia-nvshmem` pip package, and the link fails after five hours of compiling. `USE_NVSHMEM=0`, explicitly.
- **`USE_FLASH_ATTENTION=0` is ignored** if `build/CMakeCache.txt` exists from an earlier attempt; the parallel template-heavy kernels then OOM-kill a 16 GB board. Force it with `cmake -DUSE_FLASH_ATTENTION=OFF .`.
- **`USE_DISTRIBUTED=0`, the sensible choice for a single board, crashes `lerobot-train`**: its `is_main_process()` calls `torch.distributed.is_initialized()` unconditionally, and the stub module has no such method. Build with distributed on and Gloo as the backend.
- **The PyPI torchvision installs and imports fine** against a from-source torch and only fails on the first compiled op: `operator torchvision::nms does not exist`. Build it from source, matched.
- **ffmpeg's vendored NVENC code disagrees with the Video Codec SDK headers** on a struct field. Decode only is needed, so drop NVENC; note that `./configure` auto-enables it when headers are present and only `--disable-nvenc` is the off switch.
- **A dependency's install script swallows its own failure** (`|| echo FAILED` without `exit 1`), so the Docker layer builds and the missing binary surfaces two stages later.

## The number to plan around

Full build: **5 h 33 min**, of which PyTorch alone is 92 %. Everything else is under 15 minutes.

## Result

Verified on an Orin NX 16 GB: `torch.cuda.is_available()`, a real `.cuda()` tensor out of `torchcodec`, `torch.distributed.is_gloo_available()`, `import lerobot`, a SmolVLA import. `docker pull ghcr.io/ravediamond/lerobot-jetson:latest`. Source and full build log: `ravediamond/lerobot-jetson`. The write-up reached about 49k people on LinkedIn, with a public thumbs-up from LeRobot's tech lead; it is also the image every Jetson gate on this site runs in.
