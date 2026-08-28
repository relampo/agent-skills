# Load profiles and Load by Intent

Every scenario has one `load` block. Targets YAML contract **1.0.5**.

## Common fields

- `duration` — how long the scenario runs
- `iterations` — if above 0, each VU runs that many iterations; if 0 with a duration set, VUs loop until the duration ends
- If neither is set, each VU runs once

## constant

The default for day-to-day work.

```yaml
load:
  type: constant
  users: 10
  duration: 5m
  ramp_up: 10s
  ramp_down: 10s
```

An empty `type` behaves as `constant`. `users` is concurrent VUs.

## ramp

```yaml
load:
  type: ramp
  start_users: 1
  end_users: 50
  duration: 10m
```

`start_users` and `end_users` must both be above 0.

## ramp_up_down

```yaml
load:
  type: ramp_up_down
  users: 50
  duration: 10m
  ramp_up: 2m
  ramp_down: 1m
```

## throughput

```yaml
load:
  type: throughput
  target_rps: 20
  duration: 10m
  ramp_up: 1m
  ramp_down: 1m
```

## stages

```yaml
load:
  type: stages
  duration: 10m
  stages:
    - { type: ramp_up, duration: 2m, end_users: 20 }
    - { type: constant, duration: 5m, users: 20 }
    - { type: ramp_down, duration: 3m, end_users: 0 }
```

Stage types: `constant`, `ramp_up`, `ramp_down`. Stage fields: `name`, `type`, `duration`, `open_ended`, `users`, `start_users`, `end_users`.

Rules:

- Stage durations must sum exactly to `load.duration`.
- `open_ended: true` is allowed on the last stage only, and it replaces that stage's `duration`.
- These top-level fields are rejected with `stages`: `users`, `start_users`, `end_users`, `ramp_up`, `ramp_down`, `iterations`, `target_rps`, `target_unit`, `target_value`, `min_vus`, `max_vus`, `aggressiveness`.
- **`--vus`, `--duration`, and `--rps` are also rejected** for a staged scenario. Edit the stages instead.
- The **editor rejects `stages` entirely**. The CLI runs them. See [editor.md](editor.md).

## Load by Intent

Adaptive capacity search. Declare a target and health limits, and Relampo raises or lowers VUs during the run to find stable capacity.

```yaml
load:
  type: intent
  duration: 30m
  target_unit: rps        # rps (default) | vus
  target_value: 500
  min_vus: 25
  max_vus: 500
  warmup: 2m
  window: 30s
  aggressiveness: medium  # low | medium | high
  p95_max_ms: 3000
  error_rate_max_pct: 5
```

Required for `intent`: `target_value` above 0, and `min_vus`/`max_vus` above 0 with `min_vus` no greater than `max_vus`. `warmup` must be shorter than `duration`. `iterations` is not allowed.

**SLO thresholds**, all optional:

| Latency | Errors |
| --- | --- |
| `p50_max_ms` | `error_rate_max_pct` |
| `p75_max_ms` | `error_4xx_max_pct` |
| `p95_max_ms` | `error_5xx_max_pct` |
| `p99_max_ms` | |
| `p999_max_ms` | |

Use `intent` to answer "what load can this survive within these limits". Use `constant` to answer "how does it behave at exactly this load".

## run_until_stopped

```yaml
load:
  type: constant
  users: 10
  run_until_stopped: true
```

Runs until interrupted. Cannot combine with `duration` or `iterations`.

## CLI overrides

```bash
relampo run scenario.yaml --vus 50 --duration 10m
relampo run scenario.yaml --rps 200 --duration 10m
```

`--rps` must be above 0. Overrides suit constant-style runs. For `intent` and `stages`, the YAML is the source of truth.
