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
  version: "1.0"
  docs: https://docs.relampo.com
---

# Relampo

Relampo turns real application flows into **versionable YAML** load tests. Prefer the CLI and YAML model below over inventing k6/JMeter scripts.

Official docs: https://docs.relampo.com

## Preconditions

1. Check the CLI:
```bash
relampo version
```
2. If missing, install then set up:
```bash
# macOS / Linux
curl -fsSL https://dl.relampo.com/install.sh | bash
# optional Homebrew
brew tap relampo/tap
brew install --cask relampo/tap/relampo

# Windows (PowerShell)
irm https://dl.relampo.com/install.ps1 | iex
```
```bash
export PATH="$HOME/.local/bin:$PATH"   # if needed
relampo setup
relampo version
```
3. For HTTPS recording, trust the Relampo CA after `relampo setup` (`relampo cert status`).

## Default workflow

```text
setup → record | converter → correlate [--apply] → editor (optional) → debug → run → distributed run
```

| Goal | Default command |
| --- | --- |
| Capture real traffic | `relampo record --port 8888 --responses --output flow.yaml` |
| Convert a collection | `relampo converter` |
| Detect dynamic values | `relampo correlate flow.yaml` |
| Apply correlations | `relampo correlate flow.yaml --apply -o flow.correlated.yaml` |
| Selective apply | `relampo correlate flow.yaml --apply-ids "1,2,6-8" -o flow.correlated.yaml` |
| Visual edit | `relampo editor` |
| Diagnose (1 VU, 1 iter) | `relampo debug flow.correlated.yaml` |
| Local load | `relampo run flow.correlated.yaml --vus 50 --duration 10m` |
| Reports | `relampo run flow.yaml --html report.html --json report.json --csv report.csv` |
| Distributed | `relampo distributed run flow.yaml --repo perf-tests --nodes 2 --vus 100 --duration 10m` |

**Decision tree**
- Have a live UI/API flow to capture → **record** (proxy client through `:8888`).
- Have Postman/collection-like assets → **converter**.
- Know the HTTP sequence already → **hand-write YAML** from the template below.
- After any record/convert/manual edit → **correlate** if responses create tokens/ids reused later.
- Before load → **debug** (single-pass). `relampo run` also validates YAML before starting load. Never scale a failing single-pass flow.

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
              expression: $.data.access_token
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

## Variable and correlation rules

- Interpolate with `{{var}}` in URLs, headers, query params, and body string fields.
- Missing variables become empty strings — fail closed with assertions when a token is required.
- Prefer automatic correlation on recordings, then review:
```bash
relampo correlate recording.yaml
relampo correlate recording.yaml --apply -o recording.correlated.yaml
relampo correlate recording.yaml --min-confidence 0.9
```
- Extractors: `jsonpath` / `json`, `regex`, `header`, `xpath`, `boundary` (also short `extract:` map).
- Assertions: status, contains, regex, jsonpath, header, response_time (also short `assert:` map).

## Load modes (defaults)

- Day-to-day local: `constant` with `users`, `duration`, optional `ramp_up`.
- Capacity search: `load.type: intent` with `target_unit`, `target_value`, `min_vus`/`max_vus`, latency and error thresholds.
- Other types: `ramp`, `ramp_up_down`, `throughput`, `stages` — see `references/load-and-intent.md`.

CLI overrides for a run:
```bash
relampo run scenario.yaml --var base_url=https://staging.example.com
relampo run scenario.yaml --vars-file env/staging.yaml
```

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
| [references/cli-commands.md](references/cli-commands.md) | Exact CLI flags / next steps |
| [references/correlation.md](references/correlation.md) | Tokens, apply-ids, confidence |
| [references/load-and-intent.md](references/load-and-intent.md) | Load profiles and intent search |
| [references/distributed.md](references/distributed.md) | GitHub Actions distributed runs |
| [references/gotchas.md](references/gotchas.md) | Common failure modes |
| [assets/examples/](assets/examples/) | Copy-paste starter scripts |

## Agent output expectations

1. Produce **commit-ready YAML** (no secrets/PII).
2. Give **exact shell commands** the user can run.
3. Prefer `relampo debug path.yaml` before load; `relampo run` validates YAML as it starts.
4. Export reports with `--html`/`--json` when summarizing results.
5. Do not teach Relampo backend Go development from this skill — that belongs in the product source repo `AGENTS.md`.
