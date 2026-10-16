---
name: relampo
description: >
  Author, correlate, debug, and run Relampo YAML performance tests with the Relampo CLI.
  Use whenever the user mentions Relampo, load testing, performance testing, VUs, RPS,
  scenario.yaml, correlate tokens/sessions/CSRF, extractors, assertions, Load by Intent,
  distributed load via GitHub Actions or GitLab, recording HTTP traffic into YAML, converting
  Postman/collections into load scripts, debugging multi-step API flows before load,
  migrating JMeter/.jmx plans, or needs a k6/JMeter alternative that stays in YAML.
  Also use when installing or setting up the Relampo CLI, writing smoke performance tests,
  fixing broken dynamic values in recorded flows, scoped CSV/TXT datasets, or Spark request hooks.
license: MIT
metadata:
  author: relampo
  version: "1.1.1"
  docs: https://docs.relampo.com
---

# Relampo

Relampo turns real application flows into **versionable YAML** load tests. Prefer the CLI and YAML model below over inventing k6/JMeter scripts.

Official docs: https://docs.relampo.com

Baseline: **released CLI v2.10.3**, compatibility contract **1.0.10**. `test.version` is test metadata, not a schema selector.

## Preconditions

1. Check the CLI and its version:
```bash
relampo version
```
2. If it is missing or older than the baseline, use **https://docs.relampo.com/cli/installation**. Check `--help` before using newer flags on an older build.
3. Then prepare the environment:
```bash
relampo setup
relampo version
```
4. For HTTPS recording, trust the Relampo CA after `relampo setup` (`relampo cert status`).

If `relampo` is installed but not found, add `~/.local/bin` to `PATH`.

## Default workflow

```text
setup → record | converter → correlate [--apply] → validate → smoke → run → github/gitlab run
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
| GitHub distributed | `relampo github run flow.yaml --repo owner/perf-tests --nodes 2 --vus 100 --duration 10m` |
| GitLab distributed | `relampo gitlab run flow.yaml --repo group/perf-tests --nodes 2 --vus 100 --duration 10m` |

**Use `validate` for offline checks.** `relampo debug` is an alias of `relampo studio` and opens a browser. Validation checks structure and built-in syntax, not successful API behavior. After target approval, run a one-VU, one-iteration smoke file before increasing load.

**Decision tree**
- Have a live UI/API flow to capture → **record** (proxy client through `:8888`).
- Have Postman/collection-like assets → **converter**.
- Have a JMeter plan (`.jmx`) → read [jmeter-migration.md](references/jmeter-migration.md) before translating components or load settings.
- Know the HTTP sequence already → **hand-write YAML** from the template below.
- After any record/convert/manual edit → **correlate** if responses create tokens/ids reused later.
- Before load → **validate**, then smoke-test the flow on an approved target. `run` validates again before execution.

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

Use **exactly one scenario per file**. The CLI and workers execute only the first scenario, even though validation checks all scenarios. Multi-scenario execution is not part of this baseline.

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

## Built-in functions and datasets

Relampo has about 180 built-ins for test data and value transforms. Canonical syntax is `{{_name(args)}}`. Full inventory: [references/builtins.md](references/builtins.md).

```yaml
body:
  order_id: "{{_uuid()}}"
  email: "user-{{_uuid()}}@example.com"
  created: "{{_time(\"yyyy-MM-dd\")}}"
  signature: "{{_hmacSha256({{payload}},{{secret}})}}"
```

- Use canonical `{{_name(args)}}`. Bare words are variables; quote literal text.
- Invalid built-ins block HTTP request preparation. `validate` checks names, syntax, arity, and constant argument types. Dynamic values still need runtime checks.
- Direct UUID/random/time calls evaluate per occurrence. For one value reused within a request, declare a variable such as `request_id: "{{_uuid()}}"`, then use `{{request_id}}`.
- Ordinary missing variables and unknown legacy `$name` aliases can still become empty strings. Assert on required response values.
- `${...}` is for variables only. Dates use Java tokens; `E` is unsupported.
- CSV/TXT bindings can live at root, step, or request level. `per_vu` starts every VU at the first row; `per_worker` is not worker partitioning. Read [data-sources.md](references/data-sources.md) before choosing a mode.
- Use [Spark hooks](references/spark.md) for small request-specific variable logic. Prefer extractors and built-ins when they cover the task.

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

Use `github` with `git`/`gh`, or `gitlab` with `git`/`glab`. Free-tier limits are **4 nodes**, **500 total VUs**, and **1 hour**. Licensed limits depend on `relampo license status`.

```bash
relampo github run scenario.yaml --repo owner/perf-tests --nodes 2 --vus 100 --duration 10m --html distributed-report.html
relampo gitlab run scenario.yaml --repo group/perf-tests --nodes 2 --vus 100 --duration 10m --html distributed-report.html
```
**Publication requires explicit approval.** These commands can commit and push generated files and active datasets. `--trigger=false` prevents starting the run, not publication. Active CSV/TXT files are packaged even if ignored. Use dedicated execution repositories and synthetic data. Read [distributed.md](references/distributed.md) first.

## Progressive references

Read only what the task needs:

| File | When |
| --- | --- |
| [references/yaml-spec.md](references/yaml-spec.md) | Authoring or editing YAML structure |
| [references/jmeter-migration.md](references/jmeter-migration.md) | Migrating JMeter/.jmx plans, component mappings, and behavior differences |
| [references/builtins.md](references/builtins.md) | Test data, value transforms, date and crypto helpers |
| [references/data-sources.md](references/data-sources.md) | CSV/TXT parameterization |
| [references/spark.md](references/spark.md) | Before/after HTTP hooks and variable logic |
| [references/protocols.md](references/protocols.md) | WebSocket, gRPC, WebRTC, SQL steps |
| [references/load-and-intent.md](references/load-and-intent.md) | Load profiles and intent search |
| [references/correlation.md](references/correlation.md) | Tokens, apply-ids, confidence |
| [references/cli-commands.md](references/cli-commands.md) | Exact CLI flags / next steps |
| [references/editor.md](references/editor.md) | Editor and Studio behaviour and limits |
| [references/distributed.md](references/distributed.md) | GitHub/GitLab publication, datasets, and reports |
| [references/gotchas.md](references/gotchas.md) | Common failure modes |
| [assets/examples/](assets/examples/) | Copy-paste starter scripts |

## Agent output expectations

1. Produce **commit-ready YAML** with synthetic data and no secrets/PII. Supply credentials from environment variables or an untracked vars file; keep their values out of commands and reports.
2. Give **exact shell commands** the user can run.
3. Run `relampo validate path.yaml` before suggesting load. `relampo run` validates again as it starts.
4. Check fields and runtime behavior against the references. Passing validation does not prove that assertions, dynamic values, or an API flow are correct.
5. Export reports with `--html`/`--json` when summarizing results.
6. Get explicit approval before load or distributed publication. Keep backend Go development in the product source repo.
