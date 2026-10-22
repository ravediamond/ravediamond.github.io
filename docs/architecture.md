# Architecture

Fifteen versions in three weeks, each one forced by a gate result. The current design is **v1.5**. The short version: the robot station is a *harness*; a coding agent is its operator; scripted motions and a small VLA are interchangeable executors; the verifier is a per-task plugin with a stated credibility tier; every trial, whoever dispatched it, is a labelled record that feeds the next fine-tune and the next teleop session.

!!! warning "Status"
    v1.5 is specified and partly built (harness code with 284 tests, first LEGO teleop session: 60 episodes). It has **not** run end-to-end on the arm yet. Everything on the [results page](index.md) comes from v0.8–v1.1 gates.

## v1.5 in one picture

```
Claude Code (operator)  ── reads skills ──  use-case brief + station-session / failure-review
        │  MCP
        ▼
Run layer (task-agnostic)     run journal (goal, progress, events), budgets per failure class,
        │                     resume after a session ends, agent trajectory + nested trace
Harness tools                 observe · do_action · run_batch · run_policy · undo · tickets · estop
        │
Executors                     scripted_pick_place  |  run_policy (SmolVLA 450M via vla.cpp, prefix overwrite)
        │
Task plugin                   catalogue · planner · perception config · VERIFIER (tiered) · task gates
        │
Robot                         SO-101 · top + wrist cameras · scale · Jetson Orin NX
                              W1 world model in shadow mode: logged, scored, never aborts
```

### The contract

```
do_action(action, executor="auto", key=(run, action_id, attempt))
  → { trial_id, executor_status, verdict, verifier_evidence, budgets_left }

executor_status ∈ { done, timeout, clamped, error }        # what the motion did
verdict         ∈ { satisfied, violated, ambiguous, stale } # what the verifier says
ok              := executor_status == done and verdict == satisfied
```

- **Two fields, never one.** The executor never writes the verdict; the verifier never reads the agent's text or the plan. `ambiguous` (scale unstable, mask count disagrees) and `stale` (observation older than the last motion) are first-class and can only lead to a budgeted re-observe, never to success.
- **Idempotent dispatch.** The same `key` twice returns the same trial record. This closes "the agent retried after a lost tool response and the arm picked twice".
- **Budgets the agent cannot raise.** Per-class counters (re-observe, undo, refill pause, executor switch); exhausting one opens a ticket to the agent, then to the owner.

### The verifier is a plugin slot with credibility tiers

```
Verifier(context, trial) → { verdict, evidence, tier }
tier 1  physical measurement        scale, count, limit switch
tier 2  deterministic perception    mask count, colour, pose at rest   (bowls: the G9c-3 judge, 0 false positives)
tier 3  learned / VLM judge         Robometer, VLM on before/after clips
tier 4  owner label                 ground truth, slow
```

Tiers 1–2 can pass a reliability gate and admit trials to `demos/` on their own. Tier 3 only pairs with a tier-2 verdict, and only after a **verifier report card** (false positives and negatives against owner labels on *that* task) exists. This rule comes directly from the gates: the sliding-window VLM judge caught 0/8 failures, the checkpoint judge flagged 41/42 successes, Robometer scored 0.58 zero-shot. A learned verifier earns its place per task, never by default. Fine-tuning a policy on tier-3 demos without a spot-check trains it on the judge's mistakes.

### The data flywheel

Every trial is written in LeRobot v3 with `task_string`, `layout_id`, executor and version, verdict and evidence. `demos/` is a view: satisfied trials from tier-1/2 verifiers. The next teleop session is planned from the log: failed layouts first (at most a third of the budget), then farthest-point coverage of the workspace. A policy is adopted for a skill only through **G24**: enough verified demos covering the failing skill, a verification window with scripted fallback, per-call finish rate not below the incumbent, kit success not dropping; otherwise revert. Every "prefer executor X for Y" note is versioned and marked unverified after a policy swap.

### Where the world model sits

Shadow mode. W1 scores every trial (outcome readout, surprise trace) and the scores are logged next to the verdict, because G10 showed it reads outcomes perfectly and the screening gates showed it does not yet anticipate them reliably. It becomes an active veto only if a fresh-rollout gate passes against the joint-only baseline. Candidate gate: object-state and contact events (G7), where the commanded joints are blind.

---

## How it got here

??? note "Version history (click to expand)"

    | Date | Version | What it was | What killed or confirmed it |
    |---|---|---|---|
    | 09-29 → 10-08 | pre-architecture | policy benchmarks: ACT, SmolVLA, vla.cpp, TurboVLA, VLA-JEPA | all working policies at 9–10/10; TurboVLA closed |
    | 10-09 | v0.1–v0.3 | VLA-JEPA executes, LeWM reranks 8 samples (best-of-N); then end-to-end multi-view LeWM with a failure head | **H0**: samples bundled, reranking 1.84× slower, the mean does half the job for free |
    | 10-10 | v0.4–v0.7 | frozen VLA-JEPA + internal-feature monitor + LeWM grasp veto | **C1**: policy identity (0.76) beat every latent probe; data confounded |
    | 10-11 | **v0.8** | Uma-style hierarchy: System 2 VLM plans and judges, System 1 world model picks chunks within its plannable range, weekly learning loop; gates G0–G9 | G0 passed |
    | 10-11 → 10-14 | v0.8.1–v0.8.3 | plannable-range gate, sub-goal spacing rule, Robometer as judge candidate; regret replaces top-1; world model off-board | **G1b** fail (joint baseline wins), **G6a** fail (distance ≠ progress), RoboJEPA no-go on device |
    | 10-16 → 10-17 | v0.8.4–v0.8.6 | deterministic top-camera judge becomes primary; world model formalised as pre-execution veto + surprise monitor; CEM planning dropped | **G9c-3** 0 false positives; **G10** pass |
    | 10-18 | v0.9 | autonomous agentic loop (Uma System 0–3 × NVIDIA ASPIRE harness lessons), ten bug fixes, `so101_harness` package | G6b fail (H-JEPA hierarchy dropped); G11-A 12.8 ms; G12 voided |
    | 10-18 | v1.0 | real-time stack: RTC inpainting, timing prior, REKI register cache, code-space verifier, parallel screener | reviewed the same evening: core / gated / deferred tiers; paper numbers corrected; G14 keeps registers |
    | 10-20 | **v1.1** | verification and repair *at rest* only; prefix overwrite in vla.cpp; W1 shadow mode; every trial recorded | 10-19 offline gates: every mid-episode verifier failed (G15, W1-A/B/C, judge τ, G16-B); G13b prefix overwrite pass |
    | 10-20 → 10-21 | v1.2–v1.3 | new task (LEGO kitting): one segmentation per action at rest, flat tray, scripted geometric pick first, scale as judge, ACT back in the learned track | G-lat fail, G-lat-2 pass; HexaShell |
    | 10-21 | v1.4 | station as a harness for Claude Code: executors behind MCP tools, use-case briefs, three skills, plugin contract, long-running runs with a journal | first LEGO teleop (60 episodes) and snapshot sessions |
    | 10-22 | **v1.5** | harness hardened from nine harness / self-improvement / evaluation papers: two-field results, idempotent dispatch, per-class budgets, verifier isolation and report card, verified scripted picks become demos, G24 adoption protocol, edit-triggered checks, verifier as a tiered plugin slot | proposed; not yet on the arm |

### Three decisions that shaped it

1. **Verify at rest, not mid-motion.** Six mid-episode verifiers (stage trackers from proprio, masks and latents; sliding-window and checkpoint VLM judges; world-model reflexes) failed their gates on the same day. One deterministic judge at rest with 0 false positives passed. Everything downstream is built around that asymmetry.
2. **The agent stays out of the motion loop.** Claude Code dispatches unit actions and batches, handles tickets, plans the next session, and edits tools under a check. It never sits inside a 33 ms tick (HexaShell's 739 s of agent time for 26 s of robot time is the cautionary number).
3. **A task is a plugin.** Catalogue, planner, perception config, verifier and task gates are the only things that change between bowls, bricks and whatever comes next. The harness, the logging schema and the adoption protocol do not.

## Papers that changed a decision

LeWorldModel (JEPA from pixels, stable end-to-end) · LeWM/ContactGuard (multi-view recipe) · D-JEPA (prediction accuracy ≠ decision alignment) · The Planning Limits of Latent World Models (horizon must reach the target) · H-JEPA (tested as G6b, dropped) · RoboJEPA (scaling; no-go on device) · VLA-JEPA, SmolVLA, REKI (register tokens, oscillation metric) · Real-Time Chunking (G13b) · FAIL-Detect, SAFE (conformal bands, W1-C2) · Robometer, Flora-Rewarder (judging) · RegenHarness, SEAR, Robo-COP, Mulligan, ReTrace, No Free Checker, HexaShell (v1.4–v1.5).
