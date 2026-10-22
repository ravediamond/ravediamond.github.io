---
date: 2026-10-22
title: "v1.5: the station as a harness for a coding agent"
slug: the-station-as-a-harness
---

# v1.5: the station as a harness for a coding agent

Three weeks of gates reduced the world-model stack to a verifier and a shadow signal. What remained was the actual bottleneck of a one-person robotics lab: deciding what to record next, running trials unattended, and keeping the training data clean. v1.5 is that loop, specified from nine recent papers on agent harnesses and self-improvement, and partly built. It has not run end to end on the arm yet; this post is a design note, not a result.

<!-- more -->

## The shape

```
Claude Code (operator) ── skills: use-case brief, station-session, failure-review
        │ MCP
Run layer       journal, budgets per failure class, resume, full trace
Harness tools   observe · do_action · run_batch · run_policy · undo · tickets · estop
Executors       scripted_pick_place | run_policy (SmolVLA 450M via vla.cpp)
Task plugin     catalogue · planner · perception · VERIFIER (tiered) · gates
Robot           SO-101 · cameras · scale · Jetson Orin NX · W1 in shadow mode
```

The agent dispatches unit actions and batches; it never sits inside the 33 ms tick (HexaShell's 739 s of agent time for 26 s of robot time is the number to avoid). A new task is a plugin: catalogue, planner, perception config, verifier, gates. The harness and the logging schema do not change.

## Three contracts

1. **Two-field results.** `executor_status` (done / timeout / clamped / error) and `verdict` (satisfied / violated / ambiguous / stale) are separate; `ok` is their conjunction. The executor never writes the verdict, the verifier never reads the agent's text. `ambiguous` can only lead to a budgeted re-observe.
2. **Idempotent dispatch.** Every call carries `(run, action_id, attempt)`; a repeat returns the existing trial. This closes "the agent retried after a lost tool response and the arm picked twice".
3. **Verifier tiers.** 1 physical measurement (a scale, a count), 2 deterministic perception at rest (the G9c-3 judge), 3 learned or VLM judge, 4 owner label. Tiers 1–2 can pass a gate and admit trials to `demos/` alone. Tier 3 only pairs with tier 2, and only after a report card on that task. The gate on tier 3 is the 10-19 and 10-20 results written as policy.

## The flywheel

Every trial is a LeRobot v3 record with task string, layout id, executor and version, verdict and evidence. `demos/` is a view over satisfied trials from tier-1/2 verifiers. The next teleop session is planned from the log: failed layouts first, at most a third of the budget, then farthest-point coverage of the workspace. A policy replaces a scripted executor only through an adoption protocol (G24): enough verified demos on the failing skill, a verification window with scripted fallback, per-call finish rate not below the incumbent, revert otherwise.

## First task

LEGO kitting: pick bricks from bulk into a carousel, verify by weight. Chosen because it has a tier-1 verifier for free, and because a slow robot only caps volume on a product, where it would lose outright on a service. First teleop session: 60 episodes; harness code: 284 tests. The gates for it (H1, G22, K1, G24) are frozen and waiting for the arm.
