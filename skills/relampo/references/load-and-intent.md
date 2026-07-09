# Load profiles and Load by Intent

## Common load fields

- `duration` — how long the scenario runs
- `iterations` — if `> 0`, each VU runs that many iterations; if `0` and duration set, loop until duration ends
- If neither duration nor iterations is set, each VU runs once

## constant (default day-to-day)

```yaml
load:
  type: constant
  users: 10
  duration: 5m
  ramp_up: 10s
  ramp_down: 10s
  iterations: 0
```

Empty `type` is treated as `constant`. `users` = concurrent VUs.

## ramp

```yaml
load:
  type: ramp
  start_users: 1
  end_users: 50
  duration: 10m
  iterations: 0
```

## ramp_up_down

```yaml
load:
  type: ramp_up_down
  users: 50
  duration: 10m
  ramp_up: 2m
  ramp_down: 1m
  iterations: 0
```

## throughput

```yaml
load:
  type: throughput
  target_rps: 20
  duration: 10m
  ramp_up: 1m
  ramp_down: 1m
  iterations: 0
```

## stages

```yaml
load:
  type: stages
  duration: 10m
  stages:
    - type: constant
      users: 5
      duration: 2m
    - type: ramp_up
      start_users: 5
      end_users: 20
      duration: 3m
    - type: ramp_down
      start_users: 20
      end_users: 0
      duration: 5m
```

Stage types: `constant`, `ramp_up`, `ramp_down`. Stage durations must cover full `load.duration` without gaps/overlap. Do not mix top-level `users`/`target_rps`/intent fields with `stages`.

## Load by Intent

Adaptive capacity search: declare target and health limits; Relampo raises/lowers VUs during the run.

```yaml
load:
  type: intent
  duration: 30m
  target_unit: vus      # e.g. vus (or product-supported units)
  target_value: 500
  min_vus: 25
  max_vus: 500
  p95_max_ms: 3000
  error_rate_max_pct: 5
  warmup: 2m
  window: 30s
  aggressiveness: medium
```

Use intent when the goal is “find stable capacity under latency/error constraints,” not a fixed VU smoke test.

## CLI overrides

```bash
relampo run scenario.yaml --vus 50 --duration 10m
```

YAML load config is the source of truth for intent/stages; flags are best for quick constant-style overrides when supported.
