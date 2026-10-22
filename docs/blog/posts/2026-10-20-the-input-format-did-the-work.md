---
date: 2026-10-20
title: "Judging, part 2: the input format did the work"
slug: the-input-format-did-the-work
---

# Judging, part 2: the input format did the work

The deterministic judge is right but narrow: it knows about two coloured bowls. Any new task needs a judge that reads video. NVIDIA's Flora-Rewarder-27B, a robot-trained outcome scorer, passed where Gemini had failed the day before. The ablation says why, and it is not the robot training.

<!-- more -->

## G-flora

Input exactly as NVIDIA's terminal prompt expects: 64 frames, top and wrist camera side by side, no telemetry, greedy decoding. 50 episodes, 42 successes and 8 failures, bar frozen before any call: at the largest score threshold flagging ≤ 2/42 successes, ≥ 6/8 failures must be flagged.

- **8/8 failures, 2/42 successes, AUROC 1.00.** Successes score 4–5, failures 1–3, including both wrong-object cases and both nested-bowl successes that fooled Gemini.
- Deterministic (5/5 identical), 0 unparseable, 6 s per episode on an A100, $0.70 total.

Flora's licence is non-commercial, so it is evidence, not a component.

## G-flora-abl: fine-tune or format?

Same clip, same prompt, three general models.

| Model | Successes flagged | Failures flagged | AUROC | Verdict |
|---|---|---|---|---|
| Qwen3.5-27B (Flora's base, no robot fine-tune) | 2/42 | 7/8 | 0.91 | pass |
| Qwen3.5-9B | 16/42 | – | 0.75 | fail |
| Gemini 3.8 Flash, 64 JPEGs | 3/42 | 8/8 | 0.96 | narrow fail |

Flora's base model, untouched, passes the same bar. The 9B does not. Gemini, which caught 0/8 with sliding windows the day before, catches 8/8 with the whole episode and a terminal question. The fine-tune added a little; the input format (whole episode, both views, one outcome question at the end) and model size did the work.

## What goes into the stack

A commercial rewarder fine-tuned from Qwen3.5 on own labels is the planned tier-3 verifier. It pairs with the deterministic judge, never replaces it, and it gets a report card (false positives and negatives against owner labels) on every new task before any of its verdicts enter a training set. That rule is written into v1.5.
