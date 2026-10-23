---
date: 2026-10-23
title: "Paper vs hardware 5: D-JEPA"
slug: paper-vs-hardware-d-jepa
---

# Paper vs hardware 5: D-JEPA

D-JEPA (Liu, Wu, Wen, You, Zhang, Hao, Lin, Su; arXiv 2609.24749) names my strongest negative result — latent distance misorders the few candidates that compete for execution — and proposes a fix. I ran the fix with the wrong labels. This post says what the paper's test needs and why mine was not it.

<!-- more -->

## What the paper claims, and how it measures it

The "decision-local prediction gap": a world model's latent geometry can be globally informative (Spearman 0.90 between predicted and realised cost over all 63 candidates on a Push-T start) and still misorder the top four (**0.11**). The fix is **not a training loss**. It is a permutation-equivariant set operator over the whole candidate set: per candidate, a descriptor from the terminal latent minus the goal and its native rank; a shared encoder, two Set-Transformer layers, and a bounded correction δ = ε·tanh(·) with **ε = 0.2**, so the operator can only reorder within 2ε of the native minimum and can never overturn a confident choice (Prop. 2). Loss: success mass under a softmax over scores, a local ordering term near the boundary, and a trust penalty on unnecessary correction.

The supervision is the expensive part: **every candidate in the set is executed from the restored start and labelled** — counterfactual execution. Evaluation: Push-T confirmation cohort of 256 starts (Reacher 128, Granular 64) with fitting, calibration and evaluation starts **disjoint**, Wilson intervals in the figures, paired counts; Push-T 83.6 → 87.9, Reacher 86.7 → 93.8. Real PiPER with a V-JEPA 2-AC planner, **50 paired starts per task**: Push-T 56 → 74, two-cube stack 72 → 88 (gains/losses 12/3 and 10/2).

## What I ran

**W1-R (2026-10-22).** The operator exactly as App. A.2 describes it, on frozen W1's G1b candidates, five seeds, τ calibrated on 20 % of training episodes. Labels: **proxy** — the executed expert chunk marked success, the seven decoys marked failure. Bar: mean regret ≤ 0.10 and ≤ action-only + 0.05. Result: W1-R regret **0.2631 vs W1 0.2630**; the gate chose "never replace" in 24 of 25 operators because W1 ranks its own training episodes at 0.67–0.91 and validation at 0.45–0.61, so calibration found nothing to fix. An exploratory perfect rank-2 swap would reach 0.115–0.154: the bar is out of reach for a rank-local operator with K = 8 even with perfect labels.

An earlier draft of the angle (NEXT_ANGLES §1) described D-JEPA as a loss term on the world model. Claude Code caught the error when the protocol had to cite the section; the implementation is the operator, the write-up says plainly that it does not test the paper's claim.

## Their number, my number

| | Paper | This arm |
|---|---|---|
| Labels | executed outcomes of every candidate | proxy: expert = success, decoys = failure |
| Candidate set | 63 (sim), planner's pool (real) | 8 |
| Fit / calibration / eval | disjoint starts, Wilson CIs | train / 20 % cal / held-out episodes |
| Gain | +4 to +17 points | 0.000 regret |

## What the gap means

Nothing about D-JEPA was tested. Proxy labels are a tautology for the operator (the "success" is whichever candidate the expert took), so the operator has nothing to learn that the native rank does not already express. The paper's claim needs the labels the paper pays for: the outcome of actually executing each candidate from the same start. On my arm that is affordable only in one place — the G7 cube session, where the undo primitive restores the layout and G9c-3 labels at rest; four to eight executed candidates per start for a subset of starts is two hours of a robot day. The bounded-correction design, independently of the result, is the right safety property for any re-ranker on a physical arm.

**Verdict:** not tested; the proxy version fails and is renamed. The paper's protocol is written into the amendments for G7.

**What would change it:** executed outcomes on ≥ 25 held-out starts.
