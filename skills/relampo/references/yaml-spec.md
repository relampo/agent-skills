# Relampo YAML reference

Targets YAML contract **1.0.5**.

## Root sections

| Section | Required | Purpose |
| --- | --- | --- |
| `test` | yes | Metadata. `name` required; `description`, `version` optional |
| `variables` | no | Global values used as `{{name}}` |
| `data_source` | no | CSV/TXT input — see [data-sources.md](data-sources.md) |
| `http_defaults` | no | Defaults for every HTTP step |
| `grpc_defaults` | no | Defaults for gRPC steps — see [protocols.md](protocols.md) |
| `webrtc_defaults` | no | Defaults for WebRTC steps |
| `cookies` | no | Cookie policy and seeded cookies |
| `cache_manager` | no | In-memory response cache |
| `metrics` | no | Reporting configuration, see below |
| `scenarios` | yes | Flows, each with `load` and `steps` |

Keep **one scenario per file**. The editor's Debug and Run views handle only one.

## Skeleton

```yaml
test:
  name: "My test"
  version: "1.0"

variables:
  base_url: "https://api.example.com"

http_defaults:
  base_url: "{{base_url}}"
  timeout: 30s
  follow_redirects: true
  headers:
    Accept: "application/json"

scenarios:
  - name: "Main"
    load:
      type: constant
      users: 10
      duration: 5m
      ramp_up: 10s
    steps:
      - get: /health
```

## http_defaults

| Field | Type | Notes |
| --- | --- | --- |
| `base_url` | string | One URL, not a list |
| `headers` | map | Applied to every request |
| `auth` | object | See [auth](#auth) |
| `timeout` | duration | For example `30s` |
| `follow_redirects` | bool | |
| `allowed_redirect_origins` | list | Bare origins only, no path or query |
| `disable_http2` | bool | |
| `max_idle_conns`, `max_idle_conns_per_host`, `max_conns_per_host` | int | Connection pool |
| `dial_timeout`, `tls_handshake_timeout`, `response_header_timeout` | duration | |
| `retrieve_embedded_resources` | bool | Fetch page assets |
| `embedded_resource_types` | list | For example `["image/png"]` |
| `embedded_resource_blacklist` | list | URL substrings to skip |
| `retry_policy` | map | **Accepted by the schema but inert.** The engine does not read it. Use step-level [`retry`](#retry) instead |

## auth

```yaml
auth:
  type: bearer          # bearer | basic | api_key
  token: "{{access_token}}"
```

| `type` | Fields used |
| --- | --- |
| `bearer` | `token` |
| `basic` | `username`, `password` |
| `api_key` | `name`, `value`, `in` (`header` default, or `query`). Aliases `api-key`, `apikey` |

**Inheritance is a stack.** `http_defaults.auth` is the base. Entering a `group`, `controller`, `transaction`, or `one_time` that declares its own `auth` pushes it, and leaving pops it. A plain `request` has no `auth` field — it always uses the top of the stack.

```yaml
http_defaults:
  auth: { type: bearer, token: "{{base_token}}" }

steps:
  - transaction:
      name: "Checkout"
      auth: { type: basic, username: "{{u}}", password: "{{p}}" }
      steps:
        - post: /cart      # uses basic, not the bearer above
  - get: /profile          # back to bearer
```

## scenarios

Each scenario takes `name`, `load`, `steps`, and optionally `cookies`, `cache_manager`, `error_policy`, `think_time`.

For `load`, see [load-and-intent.md](load-and-intent.md).

### Step types

- **HTTP**: `request`, `get`, `post`, `put`, `delete`, `patch`, `head`, `options`
- **Other protocols**: `websocket`, `grpc`, `webrtc`, `sql`, `sql_connection` — see [protocols.md](protocols.md)
- **Containers**: `group`, `controller`, `transaction`, `parallel`, `one_time`, `balanced`
- **Flow**: `if`, `loop`, `retry`
- **Timing and checks**: `think_time`, `assertions`

### HTTP requests

```yaml
# short
- get: /health
- post: /login

# long
- request:
    name: "Login"
    method: POST
    url: /login
    headers:
      Authorization: "Bearer {{token}}"
    query_params:
      page: "1"
    body:
      user: "{{username}}"
    timeout: "45s"
    enabled: true
```

Object bodies are serialized to JSON, and Relampo sets `Content-Type: application/json` when appropriate.

Shorthand steps keep sibling keys. Fields written next to a `get:` line are preserved rather than folded into the request.

### Assertions

At 1.0.5 an assertion has exactly six fields: `type`, `value`, `path`, `name`, `max_ms`, `size`.

| `type` | Fields used |
| --- | --- |
| `status` | `value` |
| `contains` / `body_contains` | `value` |
| `not_contains` / `body_not_contains` | `value` |
| `regex` / `body_matches` | `value` |
| `jsonpath` / `json` | `path`, `value` |
| `response_time_max` / `response_time` / `response_time_ms` | `value` or `max_ms` |
| `response_size` | `value` or `size` |
| `header` | `name`, `value` |

```yaml
assertions:
  - { type: status, value: 200 }
  - { type: jsonpath, path: "$.data.token", value: "abc" }
  - { type: response_time_ms, max_ms: 500 }
  - { type: header, name: "Content-Type", value: "application/json" }
```

**Removed at 1.0.5**: `pattern`, `expression`, `from`, `ignore_case`, and `enabled` are no longer assertion fields. They never worked on assertions and are now out of the contract. They remain valid on **extractors**.

Short map form:

```yaml
assert:
  status: 200
  body_contains: "ok"
```

### Extractors

| `type` | Fields used |
| --- | --- |
| `jsonpath` / `json` | `path` (alias `expression`) |
| `regex` | `pattern`, `group` |
| `header` | `header` |
| `xpath` | `path` |
| `boundary` | `left_boundary`, `right_boundary` |
| `body` | whole body |

Shared fields: `var` (alias `variable`), `from`, `default`, `transform`, `capture_mode` (`first`/`all`/`index`/`random`), `capture_index`, `template`, `match_no`.

`from` accepts `body` (default), `headers`, `status_line`, `request_url`, `request_headers`.

```yaml
extractors:
  - { type: jsonpath, var: token, path: "$.data.token", default: "" }
  - { type: regex, var: csrf, pattern: 'csrf="([^"]+)"', group: 1 }
  - { type: header, var: session_id, header: "X-Session-Id" }
```

Prefer `path` over `expression` in new scripts. Both parse; `expression` exists for editor compatibility.

Short map form:

```yaml
extract:
  token: "jsonpath($.token)"
  req_id: "header.X-Request-Id"
```

### think_time

A duration, or a map for varied pauses.

```yaml
- think_time: 2s

- think_time:
    min: 1s
    max: 5s
    distribution: uniform
    enabled: true
```

Map fields: `duration`, `min`, `max`, `mean`, `std_dev`, `distribution`, `enabled`.

### retry

Wraps steps and repeats them on matching failures.

```yaml
- retry:
    attempts: 3
    on: [502, 503, 504]
    backoff: exponential
    initial_delay: 500ms
    max_delay: 5s
    multiplier: 2
    steps:
      - get: /flaky
```

`steps` is required. `on` is a list of status codes.

### error_policy

```yaml
error_policy:
  on_error: stop      # stop | continue (default continue)
```

Valid at scenario and step level. Step overrides scenario.

**Only `on_error` is read by the engine.** The fields `on_4xx`, `on_5xx`, and `on_timeout` are accepted by the schema but have no effect. Do not rely on them.

### Containers

```yaml
- group:
    name: "Browse"
    steps: [ ... ]

- transaction:
    name: "Checkout"
    steps: [ ... ]

- one_time:
    name: "Seed data"
    steps: [ ... ]
```

`one_time` runs once instead of on every iteration. Use it for setup and teardown, especially SQL writes.

`balanced` splits traffic by share:

```yaml
- balanced:
    name: "Mix"
    type: total
    mode: iterations      # iterations | virtual_users
    steps:
      - group: { name: "Search", percentage: 70, steps: [ ... ] }
      - group: { name: "Checkout", percentage: 30, steps: [ ... ] }
```

`name` and `type: total` are required. Percentages must be above 0, at most 100, and sum to 100. `think_time` is not a valid direct child.

### Flow control

```yaml
- if: "{{is_premium}} == true"
  steps:
    - get: /premium

- loop: 5
  steps:
    - get: /page

- loop:
    count: 10
    break_on: "{{done}}"
    steps:
      - get: /poll
```

### throughput

```yaml
- throughput:
    enabled: true
    target_rps: 20
```

### Multipart uploads

```yaml
files:
  - field: file
    path: /tmp/invoice.pdf
    mime: application/pdf
```

### Overrides

`cache_override` and `cookie_override` accept `inherit`, `enabled`, or `disabled`.

## cookies and cache_manager

```yaml
cookies:
  policy: standard              # standard | ignore_cookies
  clear_each_iteration: false
  cookies:
    - { name: "session", value: "abc", domain: "example.com", path: "/" }

cache_manager:
  clear_each_iteration: true
  max_elements: 1000
```

A seeded cookie requires `name`, `value`, and `domain`. The cache is in memory and lasts only for the run.

## metrics

```yaml
metrics:
  percentiles: [50, 90, 95, 99]
```

This configures reporting only. It does not change how the runner generates load.

## Checking your work

```bash
relampo validate scenario.yaml
```

`validate` checks structure offline for HTTP, WebSocket, gRPC, and SQL. It is **lenient about meaning**: an unknown assertion type, an unknown load type, or a misspelled built-in all pass. Use this reference to confirm field names; do not treat a passing validation as proof the test is correct.

Full product docs: https://docs.relampo.com
