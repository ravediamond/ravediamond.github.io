---
date: 2026-10-23
title: "Paper vs hardware 7: FAIL-Detect and SAFE"
slug: paper-vs-hardware-fail-detect-and-safe
---

# Paper vs hardware 7: FAIL-Detect and SAFE

Two papers, one experiment: both calibrate a **time-varying conformal band** on successful rollouts and flag a run that leaves it. FAIL-Detect (Xu et al., TRI; arXiv 2503.08558) does it on learned OOD scores with no failure data; SAFE (Gu et al., UofT/TRI; arXiv 2506.09937) on a probe of the VLA's own features trained with failures. I ran the band on W1's surprise (W1-C2). It failed, and the way it failed is the way FAIL-Detect's own hardware section describes.

<!-- more -->

## What the papers claim, and how they measure it

**FAIL-Detect**: a scalar score per step from the policy's observation embedding — logpZO (a flow from the embedding distribution to N(0, I); the score is the log-density), NatPN, RND — plus post-hoc baselines (STAC, which needs 256 policy samples per step; PCA-kmeans; smoothness). A band over time is fitted to the success distribution; a flag is raised when the score exceeds it. Sixteen simulated settings (four robomimic tasks × two policies × ID/OOD); summary is **top-1 count across settings**: logpZO 10/16. Hardware: Franka bimanual, two tasks, fewer rollouts; logpZO top-1 on 8/12. The caveat is in §VI: with ID-only bands, OOD-but-successful rollouts are flagged, "at times yielding low or close to zero TNR"; the fix needs successes from every new setting. Detection time for STAC exceeds the average success duration.

**SAFE**: the VLA's last-layer features already separate success from failure across tasks (a shared "failure zone" in t-SNE). A tiny MLP or LSTM on them, trained on seen tasks with trajectory-level labels, generalises to unseen tasks. Threshold by **functional conformal prediction** on successful calibration rollouts, FPR ≤ α at any time step. Three random seen/unseen splits; ROC-AUC on the **max-so-far** score (a flag anywhere on a success is a false positive); 14 adapted baselines. Sim unseen AUROC ~0.78 vs best baseline ~0.74; real Franka 30 + 30 per task and WidowX 532 rollouts; < 1 ms overhead.

## What I ran

**W1-C2 (2026-10-19).** A time-varying functional CP band on W1's long surprise, fitted on 22 successes, tested on the other 22 successes and the 8 failures, 20 seeds. Bar (median): successes flagged ≤ 1/20, failures ≥ 50 %. Result: successes **22.7 %** (IQR 13.6–27.3), failures 46 %, stalls 34 %. Fail. The band as registered fits μ, σ and the radius on the same successes: on synthetic exchangeable data its false-alarm rate is 0.14 at 20 anchors — about α only with separate μ/σ data. So the band was not conformal across time as written, and even a correct band sits on a surprise distribution that widens on fresh data (W1-A: the live reflex fitted on one session flagged 42/42 successes on the next).

**C1 probe (2026-10-11)**, the SAFE-shaped question — does the imagined latent predict failure better than simpler signals? — found **policy identity alone at 0.76** beating every probe (current 0.68, imagined 0.72), because the failure set was dominated by one policy.

## Their number, my number

| | Papers | This arm |
|---|---|---|
| False alarms at the registered α | ≈ α by construction | 22.7 % (α = 5 %) |
| Failures caught | logpZO top-1 in 10/16 settings; SAFE unseen AUROC 0.78 | 46 % |
| Transfer across sessions | "close to zero TNR" without new-setting successes (FAIL-Detect §VI) | 42/42 successes flagged on the next session |
| Policy-identity baseline | not run | 0.76, beats every probe |

## What the gap means

The unsupervised band fails on hardware the same way in Palo Alto and in Paris: the success distribution shifts with the setting and the band does not know. The supervised probe (SAFE's design) is the one that works on hardware in every paper I read, and the C1 result says my version of it is blocked on a confound the papers do not check — the failure labels came mostly from one policy. The register now requires a policy-identity and episode-length baseline under every monitor result, which neither paper runs.

**Verdict:** FAIL-Detect's band fails here as its authors warn it can; SAFE's probe is not yet tested cleanly.

**What would change it:** SAFE-MLP on SmolVLA's action-expert input features exported from vla.cpp, three random task splits, with episode length beside it.
