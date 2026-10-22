# What a JEPA world model actually buys you on a $300 arm

**Thesis.** On a low-cost, position-controlled arm, a latent world model earns its place as a *verifier*, not a *planner*: it reads outcomes and flags surprise, but ranking actions by imagined latents loses to a baseline that just looks at the commanded joints. The gains worth chasing on GPU-poor hardware are in evaluation, data selection and inference engineering, not in bigger policies.

Everything below was run on one SO-101 (6-DoF, STS3215 servos), a wrist and a top camera, a Jetson Orin NX 16 GB, a Mac, and rented GPUs. Total recorded cloud spend across the whole programme: **about $27**. Every result has a pre-registered pass rule, a report and, where it ran on the arm, the videos.

| Where | What |
|---|---|
| **Code and reports** | `road-to-robotics` (the working repo, reports linked per row in the [experiment log](experiments.md)) |
| **Datasets (public, Hugging Face Hub)** | [`ravediamond/so101_bowls_all_trimmed`](https://huggingface.co/datasets/ravediamond/so101_bowls_all_trimmed): 182 teleop episodes, 61k frames · [`ravediamond/so101-bowls-policy-rollouts`](https://huggingface.co/datasets/ravediamond/so101-bowls-policy-rollouts): 69 policy rollouts, 46 success / 23 failure, labelled |
| **Upstream** | LeRobot reviewer; merged: Jetson Orin Docker image (`#4599`), VLA-JEPA future-frame leak fix (`#4381`) and review of its refactor (`#4425`), draccus co-author (`#4260`, `#4265`); vla.cpp Orin NX benchmarks and prefix-overwrite port; leLab dataset-editing PRs; `lerobot-policy-turbovla-so101` on PyPI |
| **Author** | Ravindu Somawansa, Paris. Staff AI engineer (agentic systems, evaluation); 2nd place, Hugging Face × NVIDIA GTC hackathon. [GitHub](https://github.com/ravediamond) · [Hub](https://huggingface.co/ravediamond) |

---

## Three results

### 1. The world model is a verifier, not a planner

*Question.* Can a small JEPA world model (LeWM, 192-d latent, multi-view, trained on 182 episodes for $3.30) pick better action chunks or detect failures before they happen?

| Role tested | Gate | Result | Verdict |
|---|---|---|---|
| Rerank 8 VLA-JEPA samples (best-of-N) | H0 | Samples are bundled (1.08° spread); reranking adds 1.84× latency and a plain *mean* of the 8 chunks gets 45 % of the oracle gain with no model at all | no-go |
| Rank the expert chunk first among 8 (plannable range) | G1b | best top-1 0.60 (bar 0.80). An **action-only baseline, no images, no model**, reaches 0.68–0.90 with regret ≈ 0 | fail |
| Latent distance as progress / stall signal | G6a, G6b | Spearman −0.32 to −0.73 (bar −0.8); 32–52 % false stalls | fail |
| Read the outcome at the judge's timestamp | G10 step 1 | AUROC **1.000** (W1), 0.976 (W2) | pass |
| Anticipate the outcome 30 steps early (imagined latent) | G10 step 2 | beats policy identity by +0.18; surprise spikes 1.82× on missed grasps, 2.58× on stalls | pass |
| Live surprise check on the Orin NX | G11-A | **12.8 ms** per dual-camera check inside a 33 ms tick | pass |
| Live reflex thresholds on fresh rollouts | W1-A, G11-B-cal | the reflex fires on 42/42 successes; refit catches 3/25 failures at 1/44 false alarms | fail → shadow mode |
| Event-anchored grasp check from the imagined latent | W1-C1 | imagined 0.70 vs current latent 0.82: imagination is 0.12 *worse* | fail |

*Reading.* The model predicts well (G0: K=30 error 0.31 vs copy-last 0.89, reacts to action timing) and reads outcomes perfectly once they have happened. It does not anticipate them reliably enough to act on, and it never beats the commanded joints at ranking. On a position-controlled arm the actions already say where the arm will be; a world model only has something to add on object state and contact, which 182 episodes without counterfactual failures do not teach. This matches D-JEPA's point that prediction accuracy does not make latent distance decision-aligned, and *The Planning Limits of Latent World Models* on horizon. The world model stays in the stack as a logged signal while failure data accumulates.

### 2. Judging is the cheap win, and the input format matters more than the model

*Question.* What can score a rollout as success or failure on this arm, at what cost?

| Judge | Data | Result | Verdict |
|---|---|---|---|
| Deterministic top-camera judge (HSV masks + geometry, no learning), at rest | 88 rollouts | 94.3 % accuracy, **0 false positives**, every failure type caught; the remaining errors are recordings that end after the operator reset | adopted (G9c-3) |
| Robometer-4B zero-shot | ArmnetBench, 2 099 rollouts, 7 policies | pooled AUROC 0.818; **episode length alone 0.938** | fail |
| Robometer-4B zero-shot | own rollouts | AUROC 0.58–0.62 | fail |
| Gemini Flash, sliding-window FAIL probability | 50 episodes | threshold fitted at 0.98: caught **0/8** failures, ~10 s per call | fail |
| Gemini Flash, two checkpoint calls (after grasp, at rest) | 50 episodes | flags 41/42 successes | fail |
| NVIDIA Flora-Rewarder-27B, 64-frame side-by-side clip, terminal prompt | 50 episodes | **8/8 failures, 2/42 successes, AUROC 1.00**, deterministic, 6 s per episode | pass |
| Same input, Flora's base model Qwen3.5-27B, no robot fine-tune | 50 episodes | 7/8, 2/42, AUROC 0.91 | pass |
| Same input, Qwen3.5-9B | 50 episodes | 16/42 successes flagged | fail |
| Same input, Gemini Flash (64 JPEGs) | 50 episodes | 8/8, 3/42, AUROC 0.96 | narrow fail |

*Reading.* The fine-tune was not what made Flora work on this task; a whole-episode clip with both cameras and a terminal outcome question did, and only models of ~27B and up answer it well. Sliding windows and single checkpoints fail on the same episodes because nested bowls and partial lifts are ambiguous mid-episode. Two cheap baselines (episode length, policy identity) beat two published judges out of the box; they are computed first in every protocol here.

### 3. Small VLAs on a $300 arm: the bottleneck is the engine, not the model

*Question.* Which policy, which inference path, and what does it cost?

| Policy | Machine | Success (10 trials) | Chunk inference |
|---|---|---|---|
| ACT | Orin NX | 9/10 | 59 ms |
| SmolVLA 450M, PyTorch | Orin NX | 9/10 | 974 ms |
| SmolVLA 450M, vla.cpp | Orin NX | 9/10 | 390 ms |
| SmolVLA 450M, vla.cpp + flash attention | Orin NX | 8/10 | **238 ms** |
| VLA-JEPA (Qwen3-VL-2B + DiT-B), fine-tuned for $9.7 | Orin NX | 9/10 | 355 ms |
| TurboVLA | Orin NX | 0/3 | closed: ignores the instruction (task-distinguishing frames were 6 % of the data) |

*Reading.* On a single-task benchmark all three working policies hit the ceiling; the task cannot separate them, and the 2B model costs 3× the training budget of the 450M one for the same score. The real gains came from the loop: vla.cpp cut SmolVLA's latency 4×, and a gradient-free **prefix overwrite at chunk boundaries** (ported into vla.cpp, +0.4 ms) cuts the median boundary jump by 72–74 %, where exact RTC guidance manages 21–41 % at twice the latency. 24 % of failed episodes show pose oscillation (REKI's metric), which is what the register-token port targets next.

---

## What is running now

Architecture **v1.5**: an execution harness where a coding agent (Claude Code over MCP tools) dispatches unit actions to a small VLA, verifies at rest through a per-task verifier slot, logs every trial, and turns verified rollouts into the next fine-tuning set and the next list of teleops to record. The world model runs in shadow mode. Current task: a LEGO-kitting cell (pick bricks from bulk into a carousel, verify by scale), chosen because it is a product and not a labour cost. Details in [Architecture](architecture.md).

## Hardware

SO-101 follower + leader (teleop), Logitech C920 top camera (oblique), wrist camera, Jetson Orin NX 16 GB Super (headless, Docker image published), MacBook for offline evaluation, vast.ai for training (ranked by whole-job cost: GPU + disk + bandwidth). Nothing in the control loop larger than a 450M policy and a 192-d world model; everything bigger runs off-board.

## Next

- Failure data: 180+ cube/tidy teleop episodes and a 21-task SmolVLA (653 episodes) are in; the next gates test the world model on object state and contact, where the joint baseline is blind (G7).
- A commercial rewarder fine-tuned from Qwen3.5 on own labels, replacing the non-commercial Flora.
- Write-ups, in order: *Verifier, not planner* (results 1 above) · *Judging on a hobby robot* (results 2) · *Chunk boundaries and engines* (results 3).
