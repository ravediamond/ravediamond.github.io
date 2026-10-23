# Paper vs hardware

One post per paper I have run against the SO-101 / Orin setup. Each post has the same shape: what the paper claims and **how it measures it** (protocol, n, baselines), what I ran with the gate ID, their number beside mine, what the gap means, a one-line verdict, and what would change it. The gates are in the [experiment log](experiments.md); the long-form reading of every paper is in the private repo's `journal/book/` and is being folded in here as each experiment runs.

Verdict words are the register's: **reproduces** (their claim holds here within the bar), **fail** (the pre-registered bar was not met and the confound checklist is clean), **not proven** (bar not met, a confound is open), **not tested** (no run that tests the paper's claim yet).

## Posted

| # | Paper | arXiv | Gate(s) | Verdict |
|---|---|---|---|---|
| 1 | [LeWorldModel](blog/posts/2026-10-23-paper-vs-hardware-leworldmodel.md) | 2603.19312 | G0, G10, G1b | dynamics and readout reproduce; planning fails the commanded-joint baseline |
| 2 | [DINO-WM](blog/posts/2026-10-23-paper-vs-hardware-dino-wm.md) | 2411.04983 | W2 / W2′, G1b | representation reproduces; planning fails; encoder cost rules out the loop |
| 3 | [RoboJEPA](blog/posts/2026-10-23-paper-vs-hardware-robojepa.md) | 2610.10515 | action-conditioning check, Orin latency | small-end reproduces; 526 ms encoder off the robot |
| 4 | [H-JEPA](blog/posts/2026-10-23-paper-vs-hardware-h-jepa.md) | 2610.06805 | G6b | fail, outside the paper's regime (no entity-frequency gap) |
| 5 | [D-JEPA](blog/posts/2026-10-23-paper-vs-hardware-d-jepa.md) | 2609.24749 | W1-R | not tested (proxy labels); executed-candidate version scheduled for the G7 session |
| 6 | [ContactGuard](blog/posts/2026-10-23-paper-vs-hardware-contactguard.md) | 2608.13438 | W1-C1 | does not reproduce (imagined 0.70 vs current 0.82); corruption control pending |
| 7 | [FAIL-Detect and SAFE](blog/posts/2026-10-23-paper-vs-hardware-fail-detect-and-safe.md) | 2503.08558, 2506.09937 | W1-A/C | FAIL-Detect's band fails as its authors warn; SAFE probe not yet cleanly tested |
| 8 | [ReAP](blog/posts/2026-10-23-paper-vs-hardware-reap.md) | 2610.16309 | REAP head | not proven (6/8 at 7/42; two confounds, ramp-target re-score scheduled) |
| 9 | [VLA-Corrector](blog/posts/2026-10-23-paper-vs-hardware-vla-corrector.md) | 2609.06508 | borrowed signal | fail; the paper's GRU verifier is the next floor baseline |
| 10 | [Robometer](blog/posts/2026-10-23-paper-vs-hardware-robometer.md) | 2603.02115 | G9b | fails as a zero-shot judge; episode length wins 4/4 held-out tasks |
| 11 | [SmolVLA](blog/posts/2026-10-23-paper-vs-hardware-smolvla.md) | 2506.01844 | multitask gates, vla.cpp parity | reproduces; the ablation tables are the manual |
| 12 | [TurboVLA](blog/posts/2026-10-23-paper-vs-hardware-turbovla.md) | port review, path A | fact 7 (data) is the full story; port claim withdrawn |
| 13 | [VLA-JEPA](blog/posts/2026-10-23-paper-vs-hardware-vla-jepa.md) | 2602.10098 | $10 train, Orin run, H0 | runs as advertised; robustness claim untested; reranking no-go |
| 14 | [REKI](blog/posts/2026-10-23-paper-vs-hardware-reki.md) | 2610.22178 | G14 | oscillation mechanism reproduces (24 % vs their 18.6 %); fix untested |
| 15 | [Real-Time Chunking](blog/posts/2026-10-23-paper-vs-hardware-real-time-chunking.md) | 2506.07339 | G13 prefix overwrite | hard-mask point reproduces in spirit; soft-mask comparison scheduled |

## Scheduled (post follows the run)

| Paper | arXiv | Planned gate | What decides it |
|---|---|---|---|
| SEVO | 2605.11114 | §18 wrist-camera blanking, then top+side fine-tune | SmolVLA 79 → 12 with a wrist cam on their SO-101; mine is top + wrist |
| Planning Limits of JEPA | 2609.39235 | §19 VLA + WM selection; ranking test on the 69 rollouts | their real-simulator control explains G1b; the selection protocol is the test |
| PSG-JEPA + SA/IDM | 2608.06799, 2609.03565 | §7 three-arm grounding | judged on the object-state probe, not arm pose |
| CF-JEPA | 2610.00727 | §8 participation ratio, then factorisation | collapse under perturbation first |
| PEEK | 2509.18282 | §21 mask overlay for ACT | wrong-object rate |
| Mulligan | 2610.05882 | §11 Sobol + failed-layouts-first | paired fine-tunes on operator-placed layouts |
| Spotter + ReTrace | 2609.36808, 2610.15814 | §4 repair vs retreat vs retry | the retreat arm |
| FARM | 2609.11445 | readout on W1 / VLA-JEPA states, common-horizon truncation | their SO-101 row (0.58–0.70 zero-shot, 0.83 adapted) is the prior |
| TD-JEPA, PRIME | 2607.25337 | §16 pre/post-grasp split | a cost that ranks time can plan badly post-contact |
| Augmentation in post-training | 2610.05994 | §20 noise floor | same checkpoint twice |
| Robo-COP, SEAR | 2610.09228, 2610.18792 | G24 adoption window; flywheel controls | verify-before-adopt; best-of-3 and sealed judge |
| HyMeS | 2608.09410 | harness stage verification (k-of-w vote) | their SO-101 result is the reference |

## Read, not testable here

V-JEPA 2.1 (encoder too slow for the loop; used offline), JEPA-WM design study (rules adopted in W1 training), DeepJEPA (planner not the bottleneck), Imagine-RL and the residual-RL pair (no torque sensing, no RL budget), FailSafe (no simulator injection; leader-arm version noted), STDR (RL from scratch), FOREWARN (VLM-on-imagined-latents; at-rest only), VLAs-as-Tools (unit-level fine-tuning noted for G24), RegenHarness and HexaShell (design papers; rules adopted), No Free Checker (survey; report-card template adopted), blind precise assembly (torque control; its Wilson-interval figure is the evaluation standard I point at), 2610.19375 one-step fidelity (the matched-epochs control in §7).
