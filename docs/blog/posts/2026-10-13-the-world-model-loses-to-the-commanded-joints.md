---
date: 2026-10-13
title: The world model loses to the commanded joints
slug: the-world-model-loses-to-the-commanded-joints
---

# The world model loses to the commanded joints

G1b asked the question that decides whether a latent world model can plan on this arm: given the expert chunk and seven decoys, does imagining each one rank the expert first? Two world models, one bigger pretrained one, and a baseline with no model at all. The baseline won.

<!-- more -->

## G1b: plannable range

Bar: top-1 ≥ 0.8 at some target distance.

- **W1** (192-d, trained here): best 0.60 at d = 20–30. Fail.
- **W2** (frozen DINOv2-S patches + learned dynamics): arm-pose R² 0.92 vs W1's 0.57, so the representation is far better; on G1b, 0.55–0.61. Fail. A decision with W2 takes 1.8 s on the Mac anyway (G2), so a third, bigger model W3 was not run.
- **RoboJEPA** (pretrained, V-JEPA 2.1 features): the encoder takes 526 ms for two views on the Orin NX and 1 025 ms on the Mac. Off the robot. Zero-shot on our latents, the 1B model is borderline on a partial sample and action conditioning does work on our arm (9–10 % below shuffled actions). Not enough to earn a role.

## The diagnosis

Top-1 among 8 compounds pairwise errors: W2's pairwise accuracy is 0.92–0.93, and 0.92^7 ≈ 0.56, which is exactly its top-1. A 0.8 top-1 bar needs ~97 % per comparison. Regret replaced top-1 as the metric.

Then the control: score chunks with the **commanded joint trajectory alone, no images, no model**. Top-1 0.68–0.90, regret ≈ 0. On a position-controlled arm, the actions already say where the arm will be. G1b was measuring kinematics, and the joints are a better kinematics model than any latent trained on 182 episodes.

## What it means

A world model only has something to add where the actions are blind: object state, contact, failures. Prediction quality (G0 passed) is not decision quality (G1b failed). The next gates test exactly that boundary, and the planning role is gone for good.
