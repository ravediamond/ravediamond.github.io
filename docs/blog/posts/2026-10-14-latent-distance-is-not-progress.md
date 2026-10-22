---
date: 2026-10-14
title: Latent distance is not progress
slug: latent-distance-is-not-progress
---

# Latent distance is not progress

If a world model cannot rank actions, maybe it can tell how far along a task is: distance in latent space to the goal frame should fall as the arm gets closer, and a stall is when it stops falling. Zero labels needed. G6a tested that; G6b tested a hierarchical version four days later. Both failed the same way.

<!-- more -->

## G6a

Bars: Spearman correlation between step and distance-to-goal ≤ −0.8 over successful demos; done-detector AUROC ≥ 0.95.

- Monotonicity: **−0.32 to −0.50** for every variant. The arm's trajectory loops in physical space (approach, lift, swing, place), so distance to the final frame rises and falls on the way there.
- Stall rule derived from it: 32 % false stalls on held-out successes.
- Done detector: AUROC 0.97–1.00, but its negatives were at-rest frames before any grasp. An easy contrast proves little.

## G6b: the hierarchy

H-JEPA's level-2 slow features (coarse stride 15, SIGReg loss) were supposed to smooth the loops out. Monotonicity −0.73 against a −0.80 bar; **52 % false stalls**. The hierarchy was dropped, and with it the last planning-shaped role in the stack.

## What the papers already said

TD-JEPA reports ~0.65 for raw latent distance against 0.95 for a learned cost; the Planning-Limits paper shows rollouts shorter than the target distance rank below chance even with a perfect simulator. I had read both. Running the gate anyway cost nothing on the Mac and turned "the papers say" into "on this arm, this number". A progress signal has to be trained on progress labels; nothing here produced one for free.
