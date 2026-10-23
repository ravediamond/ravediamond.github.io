---
date: 2026-10-23
title: "Paper vs hardware 9: VLA-Corrector"
slug: paper-vs-hardware-vla-corrector
---

# Paper vs hardware 9: VLA-Corrector

VLA-Corrector (Song, Qian, Feng, Song; arXiv 2609.06508) wraps a frozen π0.5 in a tiny learned verifier that routes a recovery prompt. I borrowed one of its signals — cosine drift of the latent — as a monitor. It caught nothing. The part of the paper I should have borrowed is the verifier's design.

<!-- more -->

## What the paper claims, and how it measures it

The policy is frozen (OpenPI `pi05_libero`). The verifier reads only observable history: two 112×112 views → **parameter-free spatial statistics** (3×4×4 pooled RGB plus mean and std, 54 features per view) → Linear 64; state and action → MLP 32; a GRU over an 8-step window; two heads, binary risk and a five-way stage. Teacher labels come from a rule-based state machine on simulator state (f = 1 within 10 steps of a failure event, 0 beyond 20). Gate: τ = 0.40, **four consecutive positive windows**, 10-step cooldown; the stage selects a recovery prompt fragment appended to the instruction; up to 100 recovery actions, then the original instruction resumes. Verifier training: 120 episodes, 30 epochs, CPU; 0.46 ms inference.

Evaluation: LIBERO and LIBERO-Plus difficulty 4–5 under action noise, target shift, goal shift and all three, **1 000 episodes per condition**; three configurations — no assistance, a **privileged rule verifier** (the efficacy upper bound), the learned verifier. Combined perturbation 70.7 → 82.6 (rule) / 80.2 (learned): the learned verifier is within 2.4 points of the oracle. Verifier quality: accuracy 93.1 %, **precision 61.1 %, recall 94.9 %**, AUROC 0.90; recovery success 75 %. Ablations: single frame −15 F1, state-action only −15.8. Real arm (unnamed), three tasks, **40 trials each**, target moved after execution starts: 82.5 → 88.3 average, zero-shot from sim-trained features.

## What I ran

**W1-A (2026-10-19).** Cosine drift of W1's latent as a mid-rollout screener on 50 episodes, beside L2 surprise and the live reflex. Bar: ≥ 5/8 failures at ≤ 1/20 false interventions. Result: cosine drift **0/8, AUROC 0.30** — below chance; L2 surprise 2/8 at 2/42; the live reflex flagged 42/42 successes. Fail.

I never built the paper's verifier itself.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Signal tested | 54-feature GRU verifier, AUROC 0.90 | cosine drift of W1 latent, AUROC 0.30 |
| Gate design | recall-tuned (95 %), 4-window confirm, cooldown | single threshold |
| Cost of a false trigger | a recovery prompt, ~100 steps | a wasted trial (my bar: ≤ 1/20) |
| Upper bound | privileged verifier, within 2.4 pts | none |

## What the gap means

Cosine drift was a wrong borrowing — the paper's verifier is not a drift threshold. Two things from the paper matter more than the result. First, the **precision/recall split**: a 61 % / 95 % verifier is a usable gate when a false trigger costs a re-prompt rather than a trial. My monitor bar (≤ 1/20 false interventions) assumes the expensive kind; if the intervention is "re-observe and re-prompt", the bar can relax and the 6/8-at-7/42 risk head from the ReAP post may already be a useful gate. Second, the **parameter-free front end plus a GRU** reaches 0.90 on LIBERO at zero cost on the Orin; it is the right floor baseline to put under any W1-based monitor, and it is now in the amendments as one.

**Verdict:** the borrowed signal fails; the paper's verifier is untested here and is the next floor baseline.

**What would change it:** the 54-feature GRU on my 69 rollouts, beside the risk head, with the recovery cost stated in the bar.
