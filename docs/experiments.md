# Experiment log

Every gate, in date order. **Bar** is the pass rule written down before the run; **Result** is what was measured; **Verdict** applies the bar and nothing else. A row is never rewritten to fit a later result; corrections are added as new rows (see G9a-AB, G11-A, G12). Costs are vast.ai invoices where they exist.

!!! note "Method"
    Pre-register the rule, compute the cheap baseline first (episode length, policy identity, commanded joints), run the experiment only if one of its outcomes changes the next decision, and label anything decided after the number as exploratory. This is what kept the negative results below trustworthy.

## Policies on the arm

Task: "put the blue bowl in the pink bowl" (both directions), 10 trials, operator-judged, 85 trials recorded with video.

| Date | Policy / engine | Machine | Success | Chunk inference | Verdict |
|---|---|---|---|---|---|
| 09-29 | ACT | M1 Mac | 10/10 | – | works |
| 09-29 | SmolVLA (PyTorch) | M1 Mac | 9/10 | – | works |
| 09-29 | ACT | Orin NX | 9/10 | 59 ms | works |
| 09-29 | SmolVLA (PyTorch) | Orin NX | 9/10 | 974 ms | works, slow |
| 10-01 | SmolVLA via vla.cpp | Orin NX | 9/10 | 390 ms | works |
| 10-01 | SmolVLA via vla.cpp, flash attention | Orin NX | 8/10 | 238 ms | works |
| 09-29 → 10-08 | TurboVLA, several variants | M1 / Orin NX | 0/3 last variant | – | **closed**: ignores the instruction ("copycat"); task-distinguishing action was ~6 % of frames |
| 10-08 | VLA-JEPA (Qwen3-VL-2B + DiT-B, world model off at inference), fine-tuned ~$9.7 | Orin NX | 9/10 | 355 ms | works |

Reading: ceiling. Bowls cannot separate the policies; the next comparison runs on the hardest primitive (flip, inside grasp, instruction switch).

## World model

LeWM = `stable-worldmodel` JEPA, trained from pixels. **W1**: 192-d pooled latent, multi-view (ContactGuard recipe). **W2**: frozen DINOv2-S patches + learned dynamics.

| Date | ID | Question | Bar | Result | Verdict | Cost |
|---|---|---|---|---|---|---|
| 10-08 | H0 | Do VLA-JEPA's 8 samples vary enough that picking one helps? | oracle/mean error ≤ 0.7; latency N=8 ≤ 1.5×; baseline < 85 % | 0.76 (null 0.92); 1.84×; baseline 90 %; averaging the 8 chunks gets 45 % of the oracle gain with no model | **no-go** | free |
| 10-08 | LeWM env | Does LeWM train on our data? | losses fall, no collapse | pass on vast.ai and on the Jetson GPU; 11 setup problems fixed | pass | $0.28 |
| 10-09/10 | 5-epoch diag. | Does a short model react to wrong actions? | diagnostic | reacts to gross errors (episode swap, frozen action), not to ±0.5 s timing shifts; effective rank 5.8/192 | weak | small |
| 10-11 | **G0** | Faithful 100-epoch multi-view LeWM | beats copy-last; every corruption ratio ≥ 1.3 at K=30; no collapse | K=30 error 0.31 vs copy-last 0.89; timing shifts 1.99 / 1.58; effective rank 24–30; grasp-onset AUROC 0.94 | **pass** | $3.30 (+$2.96 failed runs) |
| 10-11 | C1 | Does the imagined latent predict failures better than simpler signals? (81 rollouts) | imagined beats current latent | policy identity alone 0.76 beats every probe (current 0.68, imagined 0.72) | **not proven** (confounded: TurboVLA made most failures) | free |
| 10-12 | **G1b** W1 | Plannable range: rank the expert chunk first among 8 | top-1 ≥ 0.8 at some distance | best 0.60 (d = 20–30) | **fail** | free |
| 10-12 | G1 probes | Does the latent encode the arm? | pose R² > 0.57 | W1 0.57; **W2 0.92**; gripper-open AUROC 0.999 | W2 pass | $0.06 |
| 10-12 | G2 | One decision (K=10 rollouts, H=30) within a chunk? | ready before the next chunk | W1 108 ms; **W2 1 814 ms** (Mac MPS) | W2 too slow | free |
| 10-12 | G1 W2 | W2 on G1b; decides whether W3 (bigger) runs | ≥ 0.8 at any d | 0.55–0.61; open-loop worse than W1; reacts less to wrong actions | **fail**, W3 not run | < $1 |
| 10-12/13 | G1 diagnosis | Why does selection fail? | regret ≤ 0.2 | top-1 ≈ pairwise^7; **action-only baseline (no model) top-1 0.68–0.90, regret ≈ 0**; W2 regret 0.18, W1 0.26; no conditioning bug | G1b measured kinematics, not world-model value; regret becomes the metric | free |
| 10-13 | RoboJEPA Gate B / B' | Can V-JEPA 2.1-G run on the Orin NX / Mac? | 2-view encoder p50 ≤ 200 ms | **526 ms** Orin; 1 025 ms Mac; predictor 252–487 ms | **no-go on device**; off-board must be CUDA | free |
| 10-13 | RoboJEPA Gate A | Does pretrained RoboJEPA predict our latents and use the actions? | 9-step L1 ≥ 15 % below copy-last; ≥ 5 % below shuffled actions | 300M 11.8 %, 1B 15.8 % on 30 clips; shuffled-action gap 8.9–10.4 % | no-go for the primary; action conditioning works on our arm | $1.2 |
| 10-14 | G1b-RJ | RoboJEPA 1B on G1b | – | **cancelled**: neither outcome changes the build | – | – |
| 10-14 | **G6a** | Done / stall from latent distance, zero labels | monotonicity ρ ≤ −0.8; done AUROC ≥ 0.95 | ρ −0.32 to −0.50; done 0.97–1.00 (easy contrast); 32 % false stalls | **stall by distance dropped** | free |
| 10-17 | **G10** | Outcome readout at the judge's time, and anticipation 30 steps earlier | step 1 AUROC ≥ 0.90; step 2 beats policy identity by +0.10 | step 1: W1 **1.000**, W2 0.976; step 2: +0.179; surprise spikes 1.82× on misses, 2.58× on stalls | **pass**: pre-execution veto and surprise monitor | free |
| 10-18 | **G6b** | H-JEPA level-2 slow-feature progress (stride 15, SIGReg) | ρ ≤ −0.80; false stalls ≤ 10 % | ρ −0.73; false stalls 52 % | **fail**; hierarchy dropped | free |
| 10-18 | **G11-A** | Live dual-camera surprise check latency on the Orin NX | < 20 ms inside a 33 ms tick | **12.78 ms** mean, 12.95 max | **pass** (timing) | free |
| 10-18 | G11-A re-check | Did it run the trained W1? | – | no: `strict=False` + key remap under a different transformers version left the encoder random; timing stands, latents void | correction row | free |
| 10-19 | **W1-A** | W1 as mid-rollout screener on 50 fresh episodes (L2 surprise, cosine drift, live reflex) | one signal ≥ 5/8 failures at ≤ 1/20 false interventions | L2 2/8 at 2/42 (AUROC 0.86); cosine drift 0/8; **live reflex flags 42/42 successes** | **fail** → shadow mode | free |
| 10-19 | G11-B-cal | Refit the reflex thresholds from episode maxima (69 rollouts) | recall ≥ 50 % on 25 failures | 3/25 at 1/44 false alarms; success and failure maxima overlap | **fail** → shadow mode | free |
| 10-19 | **W1-C1** | Event-anchored grasp check from the imagined latent (5-fold × 5 seeds) | AUROC ≥ 0.85 and ≥ 0.05 above the current latent | imagined 0.70, current 0.82 | **fail** | free |
| 10-19 | W1-C2 | Time-varying conformal band (FAIL-Detect / SAFE) on W1 surprise | successes flagged ≤ 1/20; failures ≥ 50 % | successes 22.7 %, failures 46 %; band not conformal across time when μ, σ are fit on the same successes | **fail** | free |

## Judges and rewards

| Date | ID | Question | Bar | Result | Verdict | Cost |
|---|---|---|---|---|---|---|
| 10-15/16 | G9a-AB | Robometer-4B zero-shot on ArmnetBench (2 099 rollouts, 7 policies) | pooled AUROC ≥ 0.85; within-policy ≥ 0.8 on ≥ 5/7 | partial set 0.861 (462 episodes lost to OOM) | qualified pass, superseded | $5 |
| 10-16 | G9a-AB complete | same, all episodes re-scored | same | pooled **0.818**; episode length alone 0.938 | **fail**: not a zero-shot judge | $0.78 |
| 10-16 | G9a-own | Robometer-4B zero-shot on own rollouts (88) | AUROC ≥ 0.9 | 0.58–0.62; policy identity 0.88 | **fail** | $0.25 |
| 10-16 | G9c | Deterministic top-camera judge (HSV + geometry) | accuracy ≥ 95 %, AUROC ≥ 0.95 | 86.4 % / 0.919; **0 false positives**; all 12 errors are recordings ending after the operator reset | pre-registered fail | free |
| 10-16 | G9c-2 | + automatic end rule (arm at rest) | same | 29/88 invalid: failing policies rarely return to rest | fail | free |
| 10-16 | **G9c-3** | + stillness-or-timeout end rule | same | **94.3 %**, 0 false positives | narrow fail / **adopted** | free |
| 10-19 | Judge τ | Fit Gemini Flash sliding-window FAIL threshold (50 episodes, 319 windows) | ≤ 1/20 successes interrupted | τ 0.98, 1/42 false; **caught 0/8**; ~10 s per call | not usable | ~$1–2 |
| 10-19 | G16-B | Two checkpoint VLM calls instead (after grasp, at rest) | ≤ 2/42 successes flagged, ≥ 6/8 failures | 41/42 successes flagged (nested bowls, pre-lift checkpoint) | **fail** → VLM judge dropped | $0.26 |
| 10-20 | **G-flora** | Flora-Rewarder-27B, 64-frame side-by-side clip, terminal prompt | at the largest k with ≤ 2/42 successes flagged, ≥ 6/8 failures | **8/8, 2/42, AUROC 1.00**, deterministic 5/5, 6 s/episode | **pass** (non-commercial licence; used as evidence only) | $0.7 |
| 10-20 | **G-flora-abl** | Same input on general models: is it the fine-tune or the input? | same, per model | Qwen3.5-27B 7/8, 2/42 (0.91) **pass**; Qwen3.5-9B 16/42 **fail**; Gemini Flash 8/8, 3/42 (0.96) narrow fail | input format did the work; size matters | $4.3 |

## Execution and engines

| Date | ID | Question | Bar | Result | Verdict |
|---|---|---|---|---|---|
| 10-18 | G12 | REKI 2.4B on the Orin NX | ≤ 8 GB, ≤ 250 ms | 3.46 GB; 318 ms | fail, then **voided**: the bench timed a stand-in model, not REKI |
| 10-18 | **G14** | Do our policies oscillate between poses? (REKI's metric, 69 rollouts) | ≥ 10 % of failed episodes | **24 %** of failed, 18 % of succeeded; real re-grasp loops on inspection | pass: register tokens stay in the plan |
| 10-19 | **G13b** | RTC guidance vs gradient-free prefix conditioning at chunk boundaries (20 episodes, 254 boundaries) | best gradient-free cuts median jump ≥ 50 %, drift ≤ +10 % | **prefix overwrite 72–74 %** at d = 8/12/20; exact RTC 21–41 % at 2× latency; every method misses the 667 ms budget, the policy itself is too slow | propose prefix overwrite |
| 10-19 | G13b in vla.cpp | Port of prefix overwrite into vla.cpp's SmolVLA | same | 71–73 %; prefix error 0.0; **+0.4 ms** per chunk; default path bit-identical; 14/14 tests | **pass** |
| 10-19 | G15, G15-v2, W1-B | Stage tracker from proprio / masks / latent prototypes | premature ≤ 2 %, missed ≤ 10 % | 48 / 47 %; design unusable (gripper reads 1.1° when holding); latent 22 / 31 % | **fail** → stage changes only on the judge at rest |

## Data

| Date | What | Result |
|---|---|---|
| 10-01 | `ravediamond/so101_bowls_all_trimmed` | 182 episodes, idle starts trimmed, 61 462 frames at 30 fps, public |
| 10-14 | Community SO-101 data catalogue (metadata only) | 203 usable datasets / 50 h; strict fit 71 / 14.4 h; ArmnetBench 2 099 labelled rollouts |
| 10-14 | `ravediamond/so101-bowls-policy-rollouts` | 69 policy rollouts, 46 success / 23 failure, labelled, public |
| 10-13 → 15 | Tidy-loop teleop (pink bowl + cubes: towers, transfers, flips, corrections) | ~180 episodes; merged into a 21-task set of 653 episodes |
| 10-16 | SmolVLA fine-tuned on the 21-task set, 30k steps | $2.94; first Orin gate: single objects 6/6, named cube among two 2/6 |

## Lessons the numbers paid for

1. A world model that predicts well can still be useless for choosing actions. Prediction quality is not decision quality.
2. On a position-controlled arm, the commanded joints already say where the arm will be. Test the world model on object state and contact, not on arm motion.
3. Raw latent distance is not progress. A progress signal has to be trained on progress labels.
4. Spatial features beat a single pooled vector (W1 pose R² 0.57 vs W2 0.92). Never PCA-compress the latent.
5. Judging is easier than imagining, but zero-shot judges are not free either: compute episode length and policy identity before any model.
6. Top-1 of N compounds errors (top-1 ≈ pairwise^7). Use regret.
7. Missing data isn't random: 22 % of episodes lost to one OOM were two whole tasks, and re-scoring them flipped the verdict.
8. Idle frames teach freezing; instruction-dependent behaviour needs data where the instruction matters.
9. Freeze the rule before the number. It is what makes the failures above worth publishing.
