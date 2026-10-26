---
date: 2026-10-26
title: "A correctness fix upstream made my checkpoint 47 % worse"
slug: upstream-fix-checkpoint-47-percent-worse
---

# A correctness fix upstream made my checkpoint 47 % worse

LeRobot fixed a real bug in VLA-JEPA's world predictor: its rotary position embedding was not a rotation. I had a checkpoint trained before the fix. Run under the fixed code, its world-model loss is 47 % higher on its own training distribution, worse on all 25 episodes. The fix is right. The checkpoint was trained around the bug, so the fix silently changes what the checkpoint computes. This post is about checking for that before you upgrade.

<!-- more -->

## The symptom

VLA-JEPA (LeRobot's port) has a world predictor next to the action head. It predicts future V-JEPA 2 latents and is trained with an L1 loss. LeRobot [#4835](https://github.com/huggingface/lerobot/pull/4835) changed how the predictor expands its rotary frequencies:

```python
# before #4835
emb_sin = freqs.sin().squeeze(-1).repeat(1, 1, 1, 2)
emb_cos = freqs.cos().squeeze(-1).repeat(1, 1, 1, 2)
# after #4835: each adjacent coordinate pair rotated by one frequency
emb_sin = freqs.sin().repeat_interleave(2, dim=-1)
emb_cos = freqs.cos().repeat_interleave(2, dim=-1)
```

On random tensors the difference is easy to confirm. The fixed version rotates each pair by one angle: pair norms are preserved to 8.9e-16, and the angle equals position × frequency to 1.3e-15. The old version does not preserve pair norms (error 2.19): the two coordinates of a pair get the cos and sin of *different* frequencies. So it was never a rotation.

My checkpoint, `ravediamond/vla_jepa_so101_bowls_trimmed` (an SO-101 fine-tune), was trained with the world model on, on the old code. LeRobot's docs say pretrained world-model quality "needs separate evaluation when continuing training". So I evaluated it.

## How it was caught: one change, everything else held fixed

The protocol was frozen and committed before any fixed-rotary number on real data existed:

- **Data:** the checkpoint's own 25 validation episodes, 8 windows each, start frames from a seeded generator per episode: **200 windows**.
- **Sharing:** for each window the V-JEPA 2 encoder and Qwen run once. Only the predictor runs more than once, on the *same* tensors.
- **Three arms:**
  - `legacy`: the old expansion patched in, checked bit-identical to the pre-fix commit `2e88044` (max abs diff 0.0);
  - `fixed`: LeRobot main `d3793b7`;
  - `copy_last`: predict each next position with the current one.
- **Loss:** exactly `_world_model_loss`'s L1.
- **Statistics:** paired bootstrap over episodes, 2,000 resamples.

Before the real comparison, three checks:
1. The legacy patch reproduces the pre-fix code exactly.
2. Two runs give identical losses.
3. The instrument separates a trained predictor from a random-init one. On 6 windows the trained predictor scored 1.14–1.26, random init 1.97–2.05, copy-last 2.20–2.28.

The bar had five labels: instrument fails, COLLAPSES, DEGRADES, SAFE, NOT PROVEN. DEGRADES needed R = fixed / legacy ≥ 1.10 with the CI of `fixed − legacy` above 0, and the fixed predictor still beating copy-last.

## The measurement

| | world-model L1 (mean over episodes) |
|---|---|
| `legacy` | **1.175** |
| `fixed` | **1.731** |
| `copy_last` | 2.235 |

| statistic | value | 95 % CI |
|---|---|---|
| R = fixed / legacy | **1.473** | [1.462, 1.484] |
| fixed − legacy | +0.556 | [0.547, 0.564] |
| fixed − copy_last | −0.505 | [−0.508, −0.501] |
| legacy − copy_last | −1.060 | [−1.069, −1.051] |

- **Verdict: DEGRADES.**
- **Episodes:** fixed is worse on 25 of 25.
- **Position:** the gap does not grow with distance from the context. It costs 0.53–0.61 at each of the three predicted positions, and about the same at the start, middle and end of episodes.
- **Not a collapse:** the fixed predictor still beats copy-last, so it has not forgotten everything.
- **What it means for fine-tuning:** continuing from this checkpoint on current main starts the world-model loss about half way back to the copy-last baseline.

## The decision

The protocol fixed the consequence in advance:
- My own fine-tunes from legacy checkpoints pin the old expansion, and say so in the model card.
- I drafted an opt-in flag for upstream (`world_model_rope_legacy: bool = False`). Old checkpoints set it in their config, new training keeps the fix, and nothing changes by default.

Upstream V-JEPA 2 made the same call for its released checkpoints ([facebookresearch/vjepa2#75](https://github.com/facebookresearch/vjepa2/pull/75) kept the old expansion). The draft is not posted yet.

## What is not proven

- **One checkpoint, one small fine-tune.** The LIBERO checkpoint (`lerobot/VLA-JEPA-LIBERO`) was not run: its files and dataset did not fit on the Orin's disk. The effect on the large pretrained model is unmeasured.
- **In distribution only.** The windows come from the checkpoint's own training data. This is a compatibility test (does the predictor still do what it learned), not a generalisation test.
- **No closed-loop effect.** My arm runs VLA-JEPA with the world model off at inference, so the robot's success rate is not affected. Fine-tuning is.
- **Recovery speed unmeasured.** I did not measure how many fine-tuning steps it takes to recover under the fixed rotary.

## The reusable check

When an upstream fix touches model code your checkpoint was trained with, run this before upgrading. It is an afternoon.

1. **Freeze the bar first.** What ratio counts as harmless, and what counts as degraded.
2. **Re-create the old behaviour** as a patch, and prove it bit-identical to the old commit on a few inputs.
3. **Share everything upstream of the change.** Run the unchanged parts once per sample and feed the same tensors to old and new. Then the only difference is the change.
4. **Add a trivial baseline** (copy-last, mean prediction) and require the old code to beat it. Otherwise your instrument cannot see anything.
5. **Pair by episode** and bootstrap over episodes, not windows.

Steps 2 and 3 in the evaluation script (`eval_rope_compat.py`): the old expansion is swapped in at module level, and each window is encoded once and predicted under both (excerpt; the per-position fields are trimmed).

```python
def set_mode(mode):
    import lerobot.policies.vla_jepa.world_model as W

    if not hasattr(W, "_rope_fixed"):
        W._rope_fixed = W.rotate_queries_or_keys
    W.rotate_queries_or_keys = rope_legacy if mode == "legacy" else W._rope_fixed

for e, s in windows(spans, val, a.windows_per_episode, d[-1]):
    i, g, ac, tpf = encode(pol, pre, ds[spans[e][0] + s])      # encoders + Qwen once
    pl, pf = predict(pred, i, ac, "legacy"), predict(pred, i, ac, "fixed")
    rows.append({"episode": e, "legacy": l1(pl, g), "fixed": l1(pf, g), "copy_last": l1(i, g)})
```

## Links

- [LeRobot #4835](https://github.com/huggingface/lerobot/pull/4835).
- [vjepa2#75](https://github.com/facebookresearch/vjepa2/pull/75).
- Data: [`ravediamond/so101_bowls_all_trimmed`](https://huggingface.co/datasets/ravediamond/so101_bowls_all_trimmed).
- Code and checkpoint: in my research repo, which goes public with the replication repo.
- Every test and verdict: the [paper vs hardware page](https://ravediamond.github.io/papers/).
