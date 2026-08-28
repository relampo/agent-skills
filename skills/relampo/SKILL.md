---
name: relampo
description: >
  Author, correlate, debug, and run Relampo YAML performance tests with the Relampo CLI.
  Use whenever the user mentions Relampo, load testing, performance testing, VUs, RPS,
  scenario.yaml, correlate tokens/sessions/CSRF, extractors, assertions, Load by Intent,
  distributed load via GitHub Actions, recording HTTP traffic into YAML, converting
  Postman/collections into load scripts, debugging multi-step API flows before load,
  JMeter-like declarative scripts, or needs a k6/JMeter alternative that stays in YAML.
  Also use when installing or setting up the Relampo CLI, writing smoke performance tests,
  or fixing broken dynamic values in recorded flows.
license: MIT
metadata:
  author: relampo
  version: "1.0.1"
  docs: https://docs.relampo.com
---

# Relampo

Relampo turns real application flows into **versionable YAML** load tests. Prefer the CLI and YAML model below over inventing k6/JMeter scripts.

Official docs: https://docs.relampo.com

This skill targets **YAML contract 1.0.5**, tested against CLI builds from 2026-08-26 or later.

## Preconditions

1. Check the CLI and its version:
```bash
relampo version
```
2. If it is missing, or older than the 2026-08-26 build, install it: **https://docs.relampo.com/cli/installation**
3. Then prepare the environment:
```bash
relampo setup
relampo version
```
4. For HTTPS recording, trust the Relampo CA after `relampo setup` (`relampo cert status`).

If `relampo` is installed but not found, add `~/.local/bin` to `PATH`.

## Default workflow

```text
setup → record | converter → correlate [--apply] → validate → editor/studio (human) → run → distributed run
```

| Goal | Default command |
| --- | --- |
| Capture real traffic | `relampo record --port 8888 --responses --output flow.yaml` |
| Capture gRPC traffic | `relampo record grpc --upstream host:443 --output grpc.yaml` |
| Convert a collection | `relampo converter` |
| Detect dynamic values | `relampo correlate flow.yaml` |
| Apply correlations | `relampo correlate flow.yaml --apply -o flow.correlated.yaml` |
| Selective apply | `relampo correlate flow.yaml --apply-ids "1,2,6-8" -o flow.correlated.yaml` |
| **Check without running** | `relampo validate flow.correlated.yaml` |
| Visual edit | `relampo editor` |
| Visual debug (1 VU, 1 iteration) | `relampo studio flow.correlated.yaml` |
| Local load | `relampo run flow.correlated.yaml --vus 50 --duration 10m` |
| Rate-based load | `relampo run flow.yaml --rps 200 --duration 10m` |
| Reports | `relampo run flow.yaml --html report.html --json report.json --csv report.csv` |
| Distributed | `relampo distributed run flow.yaml --repo perf-tests --nodes 2 --vus 100 --duration 10m` |

**Use `validate`, not `debug`, from a terminal.** `relampo debug` is an alias of `relampo studio` and opens the browser editor's visual Debug view. It needs a human. `relampo validate` is the offline check that works headlessly and covers HTTP, WebSocket, gRPC, and SQL structure.

**Decision tree**
- Have a live UI/API flow to capture → **record** (proxy client through `:8888`).
- Have Postman/collection-like assets → **converter**.
- Know the HTTP sequence already → **hand-write YAML** from the template below.
- After any record/convert/manual edit → **correlate** if responses create tokens/ids reused later.
- Before load → **validate**. `relampo run` also validates as it starts. Never scale a flow you have not checked.

## Minimal YAML template

```yaml
test:
  name: "Guest application performance test"
  description: "Example flow with token extraction and reuse"
  version: "1.0"

variables:
  base_url: "https://api.example.com"
  fcm: ""

http_defaults:
  base_url: "{{base_url}}"
  timeout: 10s
  headers:
    Accept: application/json

scenarios:
  - name: "Guest user flow"
    load:
      type: constant
      users: 10
      duration: 1m
      ramp_up: 10s
    steps:
      - request:
          name: "Create guest user"
          method: POST
          url: /api/1.0/auth/guest
          body:
            fcm: "{{fcm}}"
            device_name: "android"
            device_os: "Android"
          extractors:
            - type: jsonpath
              from: body
              var: access_token
              path: $.data.access_token
              default: ""
          assertions:
            - type: status
              value: 201

      - request:
          name: "Home"
          method: GET
          url: /api/2.1/home
          headers:
            Token: "{{access_token}}"
            Isguest: "true"
          assertions:
            - type: status
              value: 200
```

Short form is fine for simple calls: `- get: /health`. Use long `request` when you need headers, body, extractors, or assertions.

Keep **one scenario per file**. The editor's Debug and Run views only handle one, and single-scenario files stay easier to correlate and version.

## Variable and correlation rules

- Interpolate with `{{var}}` in URLs, headers, query params, and body string fields.
- Missing variables become empty strings — fail closed with assertions when a token is required.
- Prefer automatic correlation on recordings, then review:
```bash
relampo correlate recording.yaml
relampo correlate recording.yaml --apply -o recording.correlated.yaml
relampo correlate recording.yaml --min-confidence 0.9
```
- Extractors: `jsonpath` / `json`, `regex`, `header`, `xpath`, `boundary`, `body` (also short `extract:` map).
- Assertions: `status`, `contains`, `not_contains`, `regex`, `jsonpath`, `header`, `response_time_ms`, `response_size` (also short `assert:` map).

## Built-in functions

Relampo has about 180 built-ins for test data and value transforms. Canonical syntax is `{{_name(args)}}`. Full inventory: [references/builtins.md](references/builtins.md).

```yaml
body:
  order_id: "{{_uuid()}}"
  email: "user{{_counter(1,1,\"signup\")}}@example.com"
  created: "{{_time(\"yyyy-MM-dd\")}}"
  signature: "{{_hmacSha256({{payload}},{{secret}})}}"
```

**Six rules that prevent silent failures.** Each one produces a test that passes and measures nothing.

1. **Errors become empty strings.** A failed built-in returns `""` instead of stopping the run. Assert on every value that matters.
2. **Bare words are variables, not text.** `{{_upper(name)}}` reads a variable. `{{_upper("name")}}` uppercases the literal.
3. **Results are cached per request.** Two `{{_uuid()}}` calls in one request return the same value. Only `_counter` and `_sequence` advance every time.
4. **An unknown `_name` fails; an unknown `$name` does not.** `{{$notReal}}` silently becomes empty.
5. **Built-ins need `{{ }}`.** `${_uuid()}` is rejected; `${...}` is for variables only.
6. **Dates use Java tokens** (`yyyy-MM-dd HH:mm:ss`), and `E` is unsupported.

Write canonical `{{_uuid()}}`. Legacy `{{$uuid}}` still works and appears in older scripts, but do not write new ones.

## Load modes (defaults)

- Day-to-day local: `constant` with `users`, `duration`, optional `ramp_up`.
- Capacity search: `load.type: intent` with `target_unit`, `target_value`, `min_vus`/`max_vus`, latency and error thresholds.
- Other types: `ramp`, `ramp_up_down`, `throughput`, `stages` — see [references/load-and-intent.md](references/load-and-intent.md).

CLI overrides for a run:
```bash
relampo run scenario.yaml --vus 50 --duration 10m
relampo run scenario.yaml --rps 200 --duration 10m
relampo run scenario.yaml --var base_url=https://staging.example.com
relampo run scenario.yaml --vars-file env/staging.yaml
```

`--vus`, `--duration`, and `--rps` are rejected when the scenario uses `load.type: stages`. Edit the stages instead.

## Distributed execution

Requires `git`, `gh`, and `gh auth login`. CLI contract limits: **≤4 nodes**, **≤500 total VUs**, **≤1 hour**.

```bash
relampo distributed files upload --repo perf-tests --files "[scenario.yaml,users.csv]"
relampo distributed run scenario.yaml --repo perf-tests --nodes 2 --vus 100 --duration 10m --html distributed-report.html
```

## Progressive references

Read only what the task needs:

| File | When |
| --- | --- |
| [references/yaml-spec.md](references/yaml-spec.md) | Authoring or editing YAML structure |
| [references/builtins.md](references/builtins.md) | Test data, value transforms, date and crypto helpers |
| [references/data-sources.md](references/data-sources.md) | CSV/TXT parameterization |
| [references/protocols.md](references/protocols.md) | WebSocket, gRPC, WebRTC, SQL steps |
| [references/load-and-intent.md](references/load-and-intent.md) | Load profiles and intent search |
| [references/correlation.md](references/correlation.md) | Tokens, apply-ids, confidence |
| [references/cli-commands.md](references/cli-commands.md) | Exact CLI flags / next steps |
| [references/editor.md](references/editor.md) | Editor and Studio behaviour and limits |
| [references/distributed.md](references/distributed.md) | GitHub Actions distributed runs |
| [references/gotchas.md](references/gotchas.md) | Common failure modes |
| [assets/examples/](assets/examples/) | Copy-paste starter scripts |

## Agent output expectations

1. Produce **commit-ready YAML** (no secrets/PII). Pass credentials with `--var` or `--vars-file`.
2. Give **exact shell commands** the user can run.
3. Run `relampo validate path.yaml` before suggesting load. `relampo run` validates again as it starts.
4. Remember that `validate` is lenient — it checks structure, not meaning. It accepts an unknown assertion type or built-in name without complaint. Read the reference files; do not rely on validation to catch an invented field.
5. Export reports with `--html`/`--json` when summarizing results.
6. Do not teach Relampo backend Go development from this skill — that belongs in the product source repo `AGENTS.md`.
