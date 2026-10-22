---
date: 2026-10-08
title: "TurboVLA: a policy that ignored its instruction"
slug: turbovla-a-policy-that-ignored-its-instruction
---

# TurboVLA: a policy that ignored its instruction

Ten days of work, several variants, a published PyPI package, and a closed project. TurboVLA fine-tuned on the two-task bowls data learned to perform whichever task it had seen last, whatever the instruction said. The failure taught me two things about data that the later datasets were built around.

<!-- more -->

## What happened

- Checkpoint conversion from the LIBERO warm start worked (verified 2026-09-08).
- Trained on bowls, it rolled out at 0/3 on the last variant. On inspection it did the same motion for "blue in pink" and "pink in blue": a copycat.
- An upstream-port spec was written for the maintainers, and the honest estimate was about one chance in three of passing a 50-episode benchmark. I stopped.

## Why

1. **Instruction-dependent behaviour needs data where the instruction matters.** In the bowls set, the frames where the two tasks differ (which bowl to grasp first) are about 6 % of all frames. The rest is identical reach, lift, place. A policy can fit 94 % of the data while ignoring the instruction entirely.
2. **Idle frames teach freezing.** The earlier recordings started with the arm at rest for a second or two. TurboVLA learned "stay" as a valid action. Every dataset since has its idle starts trimmed with the same script.

## What survived

`lerobot-policy-turbovla-so101` stays on PyPI as a working plugin policy; the bowls dataset was re-cut; the two lessons went into the recording guide. Not every project needs to end in a merge to be worth the time.
