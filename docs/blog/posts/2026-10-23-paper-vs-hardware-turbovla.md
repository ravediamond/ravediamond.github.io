---
date: 2026-10-23
title: "Paper vs hardware 12: TurboVLA"
slug: paper-vs-hardware-turbovla
---

# Paper vs hardware 12: TurboVLA

TurboVLA (Xie, Yao, Wu et al.; arXiv 2607.27205) drops the language model from the VLA and binds vision to text with Grounding-DINO-style cross-attention. My port failed on a two-task dataset, I concluded it ignored the instruction, then I found my port differed from the paper in four places, then I ran the paper's own trainer on my data. The faithful model ignores the instruction too. The data was the whole story.

<!-- more -->

## What the paper claims, and how it measures it

V + L → A instead of V → L → A: DINOv3 patch features and BERT token features, each projected to 256-d, fused by **six bidirectional cross-attention layers initialised from Grounding DINO** (`--pretrained_init_ckpt groundingdino_swint_ogc.pth`), then an ACT-style decoder with 12 learnable action queries; the robot **state enters the decoder memory only** (eq. 6/9), because "it is required for translating task-conditioned scene features into executable actions but not necessary for visual-language correspondence". Plain ℓ1 on the chunk. 0.2B parameters, 31 ms per chunk on a 4090.

Evaluation: LIBERO with one mixed-suite model, **50 rollouts per task**, 97.7 average (π0.5 96.9 at 3× the latency, SmolVLA 88.8); RoboTwin 2.0 multi-task 60.2; real Piper, four tasks, **40 trials each**, against π0.5 on the same data. The ablation that matters here, Table 3: **remove language, LIBERO average 97.7 → 70.8, LIBERO-Goal 97.4 → 11.6**. On data where the instruction is the only thing that distinguishes targets, this architecture uses it. One-way cross-attention costs 1.2–1.6 points (Table 5). The Grounding DINO initialisation is stated once and **never ablated**.

## What I ran

**Several variants, Sept 29 – Oct 8.** 0/3 on the last; a two-week diagnosis with six probes concluded the model ignored the instruction on the two-task bowls data. The data finding survived: on 94 % of frames the instruction is redundant with the arm state ("fact 7": the side decision is visible in ~6 % of frames and the pose leaks it), so a copycat policy fits the data.

**The port review (2026-10-22).** Against upstream `turbovla/models/`: (1) the six fusion layers were **random-init** in every variant — the checkpoint converter transferred DINOv3 only; (2) the fusion block is a different module — two independent cross-attentions instead of `BiAttentionBlock` (shared affinity, layer-scale 1e-4, drop-path) plus a text self-attention layer; (3) the state token was **prepended to vision before fusion** instead of entering the decoder memory; (4) decoder depth, head and normalisation differ. So "TurboVLA ignores the instruction" was never established; a port that ignores it was.

**Faithful reproduction, step 1 (2026-10-22).** The free control: upstream code, released LIBERO checkpoint, task swap over the ten LIBERO-Goal instructions on 40 frames. Threshold 0.095 normalised (= 5° of SO-101 pan). Result **0.105** — the real model moves its actions when the instruction changes; my port moved 0.1–0.25°, 20–55× less. Path A was frozen on that number: task swap ≥ 5° with per-layout ≥ 75 % → the port was the bug; task swap < 1° with the control ≥ 0.095 → fact 7.

**Faithful reproduction, path A (2026-10-23).** Upstream's own trainer on the bowls data, full LIBERO init (666 of 672 tensors transferred; the 6 width-dependent ones re-initialised), 35k steps, batch 64 (128 OOM on 24 GB), ≈ $2.0 plus ~$0.8 of failed starts. Result: **task swap 0.020°** (frames 0.006–0.053°), per-layout 38.9 %, start reach 26.7°; the control still 0.105. The paper's recipe, with the paper's initialisation and the paper's fusion block, moves its actions by two hundredths of a degree when the instruction changes on this data. Fact 7 is the full story.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Instruction sensitivity | LIBERO-Goal 97.4 → 11.6 without language | LIBERO control 0.105 (reads it); port 0.1–0.25° and **faithful 0.020°** on bowls (neither does) |
| Fusion init | Grounding DINO, never ablated | port random vs faithful Grounding DINO: **no effect on instruction use** on this data |
| Real-arm success | 80–92.5 %, 40 trials | 0/3 on the last port |
| Data | multi-task, instruction decisive | two tasks, instruction redundant on 94 % of frames |

## What the gap means

Two claims, now both answered. The **data** claim stands and is now the whole result: on frames where the instruction is redundant with the arm state 94 % of the time, the paper's architecture with the paper's weights learns the copycat too (0.020° vs 0.105 for the same model on LIBERO-Goal, where the instruction is decisive). The **model** claim — that *my port* was the reason — is false: the four divergences are real, documented, and made no difference to instruction use. Table 3 of the paper and path A together say the same thing from two sides: this architecture reads language exactly when the data makes language necessary, and not otherwise. A by-product: the Grounding DINO initialisation, never ablated in the paper, is ablated here on one dataset (random vs faithful) with no effect on the quantity that mattered.

The lesson I am keeping is about order, not architecture: parity against upstream should have come before the two-week diagnosis, and the diagnosis should have ended with a data-side control (train the paper's model on data where the instruction is decisive) before any sentence about the model. Both are now rules in the register.

**Verdict:** the data claim (fact 7) is the full story; the port claim is withdrawn; correction row written.

**What would change it:** nothing about this dataset. The next test is the paper's model on data where the instruction decides early and often (the tidy-loop sessions), which is the only way to learn whether it helps on this arm.
