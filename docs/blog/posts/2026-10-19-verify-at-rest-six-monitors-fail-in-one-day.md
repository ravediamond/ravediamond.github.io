---
date: 2026-10-19
title: "Verify at rest: six mid-episode monitors fail in one day"
slug: verify-at-rest-six-monitors-fail-in-one-day
---

# Verify at rest: six mid-episode monitors fail in one day

One Jetson day, no arm motion, every mid-episode verifier in the v1.0 design run against 50 fresh rollouts with the bars frozen the night before. Stage trackers, VLM judges and the world-model reflex all failed. One offline engine experiment passed. The architecture that came out of it (v1.1) verifies only at rest, and that rule has not moved since.

<!-- more -->

## The world model, on fresh data

| Test | Bar | Result |
|---|---|---|
| W1-A: L2 surprise, cosine drift, live reflex (2.2 / 10.0) as screeners | one signal catches ≥ 5/8 failures at ≤ 1/20 false interventions | L2 2/8 (AUROC 0.86); drift 0/8; **the live reflex fires on 42/42 successes** |
| G11-B-cal: refit the reflex from episode maxima | recall ≥ 50 % on 25 failures | 3/25 at 1/44 false alarms; success and failure maxima overlap |
| W1-C1: event-anchored grasp check from the imagined latent | AUROC ≥ 0.85, ≥ 0.05 above the current latent | imagined **0.70**, current 0.82: imagination is worse |
| W1-C2: conformal band on the surprise trace (FAIL-Detect / SAFE) | successes flagged ≤ 1/20, failures ≥ 50 % | 22.7 % / 46 %; the band is not conformal across time when μ and σ are fit on the same successes |

G10's thresholds did not transfer. The world model went to **shadow mode**: scored and logged on every trial, never allowed to stop the arm.

## Stage trackers and VLM judges

- G15 (proprio + masks): 48 % premature, 47 % missed. The gripper reads 1.1° when holding a bowl rim, below every "closed on nothing" threshold, so a grasp detector built on aperture cannot fire.
- G15-v2 and the latent-prototype tracker W1-B: unusable on this gripper; 22 % / 31 %.
- Gemini Flash with sliding-window FAIL probability: threshold fitted at 0.98, **0/8 failures caught**, 10 s per call.
- Gemini Flash with two checkpoint calls (after grasp, at rest): **41/42 successes flagged**. Nested bowls read as "blue alone on the mat"; the post-grasp checkpoint comes before the lift.

## The one that passed: G13b

Chunk boundaries are where SmolVLA jerks. Real-time chunking (RTC) does inference-time guidance; the gradient-free alternative overwrites the new chunk's prefix with the tail of the old one. On 20 episodes, 254 boundaries: prefix overwrite cuts the median boundary jump **72–74 %**; exact RTC 21–41 % at twice the latency. Ported into vla.cpp the same day: 71–73 %, prefix error 0.0, **+0.4 ms** per chunk, default path bit-identical, 14/14 tests.

## v1.1

Verification and repair at rest with the deterministic judge; prefix overwrite in the engine; world model in shadow mode; every trial recorded. The simplest design of the series, and the first one forced entirely by its own negative results.
