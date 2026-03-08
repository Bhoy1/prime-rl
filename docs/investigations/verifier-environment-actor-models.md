# Investigation: Verifier Environment + Multi-Actor LoRA

## Problem

When using the same base model with separate LoRA adapters per actor (multi-agent LoRA mode),
the `actor_models` parameter is threaded through prime-rl's orchestration pipeline but the
downstream `verifiers` library does not accept or use it.

## Root Cause

The verifiers library (pinned at `45fae8d`, also confirmed on `main`) has no concept of
`actor_models` or per-actor model routing:

1. **`Environment.run_rollout()`** does NOT accept `actor_models` — no `**kwargs` either.
   Passing it raises `TypeError: got an unexpected keyword argument 'actor_models'`.

2. **`Environment.run_group()`** has `**kwargs` so `actor_models` passes silently, but the
   kwarg is never consumed — it's dropped.

3. **`EnvClient.run_rollout()`** (ZMQ server path) also does not accept or serialize
   `actor_models` in `RunRolloutRequest`.

4. **`Environment` base class** has no `actors` property or `get_actor()` method.
   prime-rl guards this with `hasattr(env, "actors")` so it doesn't crash, but multi-agent
   mode can never activate unless an environment package manually defines `.actors`.

## Affected Code Paths in prime-rl

| File | Line(s) | Issue |
|------|---------|-------|
| `orchestrator/vf_utils.py` | 97-105 | `env.run_rollout(..., actor_models=actor_models)` — crashes when not None |
| `orchestrator/vf_utils.py` | 126-134 | `env.run_group(..., actor_models=actor_models)` — silently dropped |
| `orchestrator/scheduler.py` | 211 | Passes `actor_model_names` to `run_rollout` |
| `orchestrator/orchestrator.py` | 513, 933 | Passes `actor_model_names` to `evaluate_env` |
| `entrypoints/rl.py` | 119-126 | Checks `env.actors` / `env.get_actor()` which don't exist in base |

## What Works Today

- **Single model, no LoRA**: `actor_models` is `None`, so the kwarg is not passed (guarded by `or None`).
- **Single LoRA**: Scheduler sets `self.model_name = self.lora_name` (scheduler.py:296), so the
  LoRA adapter name is passed as the regular `model` parameter. Works correctly.

## What Doesn't Work

- **Multi-agent LoRA** (same model, N separate LoRA adapters): The scheduler populates
  `actor_model_names = {"actor_0": "run_actor_0", "actor_1": "run_actor_1", ...}` and loads
  each adapter on the inference server. But the environment receives a single `model` string
  and has no way to route different actors to different adapters within the same rollout.

- **Multi-model**: `actor_endpoints` are injected into `env.args` as a workaround, but the
  `actor_models` dict itself is still not consumed by the environment.

## Required Changes (in verifiers repo)

1. Add `actor_models: dict[str, str] | None = None` to `Environment.run_rollout()` and
   `Environment.run_group()` signatures.
2. Pass `actor_models` through `RunRolloutRequest` / `RunGroupRequest` ZMQ serialization.
3. Thread `actor_models` into the `rollout()` method so environment implementations can
   route inference calls per-actor.
4. Optionally define `actors` property and `get_actor()` on the `Environment` base class.
5. Bump the verifiers pin in prime-rl after the changes land.
