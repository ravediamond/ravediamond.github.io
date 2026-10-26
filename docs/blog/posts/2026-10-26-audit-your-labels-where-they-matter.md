---
date: 2026-10-26
title: "Audit your labels where they matter: a VLM labeller that is right on average"
slug: audit-your-labels-where-they-matter
---

# Audit your labels where they matter: a VLM labeller that is right on average

Two evaluation problems on my SO-101 rollouts had the same shape: the labels were fine on average and wrong exactly where the metric looked.

- **Failure onsets marked by a VLM.** The median error was 6 frames, but the error was 110 and 289 frames on the episodes a failure monitor most needed.
- **A deterministic success judge.** It missed its accuracy bar, and every one of its errors was a recording that ran past the end of the task.

Neither showed in an aggregate number. Both showed when I looked per failure type and per error.

<!-- more -->

## Labels 1: failure onsets from Gemini

A failure monitor is scored on *when* it warns. For that it needs an onset frame per failed episode: the moment the attempt went wrong.

- **Setup.** I had 25 failed rollouts. By owner decision, logged as a protocol deviation before labelling, Gemini 3.8 Flash marked the onsets instead of a person.
- **How it labelled.** A coarse-to-fine contact-sheet search, told that the attempt failed and given the operator's failure note. 52 calls, temperature 0, 25 of 25 labelled.
- **The check.** A spot check against the video, to the frame, on 5 episodes:

| episode | failure | Gemini | by eye | offset |
|---|---|---|---|---|
| 2 | missed grasp, retried | 180 | 174 | +6 |
| 18 | missed grasp | 349 | 239 | **+110** |
| 30 | wrong object (released beside) | 201 | 200 | +1 |
| 36 | missed grasp | 418 | 129 | **+289** |
| 66 | wrong object (went to pink) | 7 | 10 | −3 |

The median offset is +6 frames (0.2 s), which reads as "good enough". It is not one error, though. Where the failure is a single event (release beside, wrong bowl from the start, one miss), Gemini is within 6 frames. Where the arm missed and *tried again*, Gemini marked a later retry, not the first missed closure: 3.7 s and 9.6 s late.

That matters because of what the labels were for. A monitor that warns between the first miss and the retry looks "early" against a late onset. The bias moves every positive later, in the direction that flatters the monitor.

- The REAP monitor scored on these onsets (AUROC 0.93) went into the register as **not proven, pending the onset check**, with episodes 18 and 36 queued for hand relabelling.
- The second version of that monitor removed onsets entirely: a ramp target, and every episode cut to a common horizon of 374 frames. It found no monitor at the bar either.

## Labels 2: where does the recording end?

The second case is a deterministic judge for the bowls task. It uses colour masks on the top camera plus geometry, with no learning. On 88 labelled rollouts the pre-registered bar was accuracy ≥ 95 % and AUROC ≥ 0.95. It scored **86.4 % [78.4, 93.2]**, AUROC 0.919.

The breakdown was strange in a useful way. **All 12 errors were successes called failure; there were no false positives.** The error frames showed the scene being undone: bowls apart again, the operator's hands, the arm still moving.

The judge was looking at the last frame of the *recording*. The protocol meant the end of the *task*. For many successes, the operator had started resetting the scene before the recording stopped.

- On 2026-10-14, before this judge existed, 11 Orin episodes had been marked with a `reset_start` by eye, all successes. **9 of the 12 errors were among them.**
- Judged at the marked reset (exploratory, same frozen thresholds), the same judge scores **95.5 % [90.9, 98.9]**, AUROC 0.967, still with 0 false positives.

The fix that went into the next pre-registered version (G9c-3) was a rule, not a label: judge at the first still moment after the last gripper release, or at the timeout. Coverage was 98.9 %, accuracy **94.3 % [88.5, 98.9]**, 0 false positives, and every failure type caught. That is a narrow fail, 0.7 points under the bar, reported as such. The remaining errors are unmarked resets or labels to recheck.

## What is not proven

- **The onset spot check is n = 5.** It shows the failure mode exists (retries) but not how often it happens across the 25. Episodes 18 and 36 are still waiting for hand labels.
- **The effect on monitor scores is not measured with corrected onsets.** The monitor that replaced the onsets also failed its bar, so no verdict depends on them today.
- **The reset marks are by eye** and exist for 11 Orin episodes only. M1 sessions have none.
- **The judge's last 0.7 points** are attributed to unmarked resets or label errors, not verified.

## The reusable check

1. **Spot-check machine labels per failure type, not by the median.** Pick at least one episode of each type, especially multi-attempt ones. A single median hides a bimodal error.
2. **Ask which direction the label error moves your metric.** Late onsets flatter early-warning monitors. If the bias favours your method, treat the result as not proven until the labels are fixed.
3. **When a judge's errors are all one-sided, look at the frames it judged** before tuning a threshold. Here they all came from recording past the end of the task.
4. **Judge at a defined task event,** not at the end of the file. The rule from the judge (`g9c3_test.py`) is a few lines:

```python
opened = np.nonzero(st[:, 5] > rule["g_open"])[0]          # gripper open frames
t_rel = int(opened[-1]) if len(opened) else 0              # last release
w = max(1, int(round(rule["still_s"] * fps)))              # 0.5 s window
jf, why = T - 1, "timeout"
for t in range(t_rel + w, T):
    if np.abs(arm[t] - arm[t - w]).max() <= rule["still_tol_deg"]:   # 1.5 deg
        jf, why = t, "still"; break
```

## Links

- Rollouts: [`ravediamond/so101-bowls-policy-rollouts`](https://huggingface.co/datasets/ravediamond/so101-bowls-policy-rollouts) (69 episodes, labelled).
- Earlier posts:
  - [the zero-false-positive judge](https://ravediamond.github.io/blog/2026/10/16/judging-a-rollout-the-zero-false-positive-rule/);
  - [Paper vs hardware 8: ReAP](https://ravediamond.github.io/blog/2026/10/23/paper-vs-hardware-reap/).
- Every test and verdict: the [paper vs hardware page](https://ravediamond.github.io/papers/).
