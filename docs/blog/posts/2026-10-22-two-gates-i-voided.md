---
date: 2026-10-22
title: "Two gates I voided, and the two-line check that would have caught them"
slug: two-gates-i-voided
---

# Two gates I voided, and the two-line check that would have caught them

On 2026-10-18 two Jetson gates went into the register with bars frozen beforehand: G11-A passed, G12 failed. Both were voided the same day. G11-A timed the right model with the wrong weights; G12 timed the wrong model. Neither was a training or hardware problem. Both were caught by checking what the script had actually loaded.

<!-- more -->

## G11-A: 12.78 ms of a random encoder

G11 Phase A asked whether W1, the multi-view JEPA world model, can run its surprise check inside the 33.3 ms control tick on the Orin NX, motors off. Bar: under 20 ms per check. Result: mean 12.78 ms, p95 12.87, max 12.95. Pass, 20.5 ms of headroom.

The next Jetson run was meant to measure W1's error on a still scene. No camera was attached, so only the checkpoint load could run, through the live loader, which refuses `strict=False`. It failed: **192 missing keys, 192 unexpected**, all in the ViT encoder.

The checkpoint was saved under transformers 5.19.0, which names ViT layers `encoder.layers.N.attention.q_proj.*`. The Docker image ships 5.5.4, which names the same tensor `encoder.encoder.layer.N.attention.attention.query.*`. A key remap renamed `layers` → `layer` and `q_proj` → `attention.query`, but did not add the inner `encoder.` level, so the remapped keys still missed. Under 5.19.0 all 279 checkpoint keys load verbatim, no remap.

G11-A's benchmark used the same remap with `strict=False`. All 192 encoder tensors stayed at random initialisation. The timing stands, since the architecture and compute are the same, but no latent and no surprise value from that script came from the trained encoder. The unit tests passed throughout: they only exercised the `layers` → `layer` rename on a stand-in model.

Register: **timing valid, latents void.**

## G12: parts that don't sum

G12 asked whether REKI, a 2.4B VLA with register-token memory, fits the Orin: memory ≤ 8 GB, chunk ≤ 250 ms. Result: 3.46 GB, 318.2 ms. Fail, and a decision to port the registers into SmolVLA instead.

Re-reading the report: 135.4 + 230.3 is not 318.2. The expert time came from a separate loop. Then the benchmark itself: a randomly initialised Qwen2.5-1.5B, text only, 256 random tokens, no vision tower, none of Qwen3.5's Gated DeltaNet layers, followed by a hand-written random expert, in eager PyTorch. And 3.46 GB matches that 1.64B-parameter proxy, not REKI's 2.4B. So 318 ms is neither REKI's latency on the Orin nor a bound on it.

Register: **G12 void; REKI on the Orin is unmeasured.**

## The two-line check

```python
m = W1LiveMonitor.from_checkpoint(ckpt)  # load_state_dict(strict=True), never strict=False
assert set(m.model.state_dict()) == set(torch.load(ckpt, map_location="cpu", weights_only=False)["state_dict"])
```

The first line raises on any missing or unexpected key. The second confirms the checkpoint's keys were used verbatim, not through a remap that happened to fit. On G11-A's setup the first line stops at 192 keys before a single timing is taken.

## Parts must sum

G12's check is arithmetic. If a report splits a latency into parts, the parts add up to the total, measured in the same loop. And the thing measured must be the thing named: parameter count, vision tower, layer types. A 2.4B model that peaks at the memory of a 1.64B one is a second sum that does not hold.

## What changed

Both gates had frozen bars, and both results were real measurements of something. Pre-registration stops the bar from moving after the number; it says nothing about what produced the number. So the checks now come before the number: a strict load with a key-set comparison for any checkpoint, and a sum over the parts for any latency breakdown.

G11-A's 12.78 ms survives. G12 is open again: the REKI decision now rests on the G14 oscillation audit and on a real measurement, with the vision tower and the real expert.
