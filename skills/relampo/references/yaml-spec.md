# Relampo YAML quick reference

Every valid script uses these **root** sections:

| Section | Required | Purpose |
| --- | --- | --- |
| `test` | yes | Metadata (`name` required; `description`, `version` optional) |
| `variables` | no | Global reusable values via `{{name}}` |
| `data_source` | no | External CSV/TXT input |
| `http_defaults` | no | Default base_url, headers, timeout, redirects, embedded resources |
| `error_policy` | no | `on_error` / `on_4xx` / `on_5xx` / `on_timeout` → `stop` \| `continue` |
| `scenarios` | yes | Executable flows with `load` + `steps` |

## Skeleton

```yaml
test:
  name: "My test"
  description: "Optional"
  version: "1.0"
variables:
  base_url: "https://api.example.com"
data_source:
  type: csv
  file: "users.csv"
  mode: per_vu
http_defaults:
  base_url: "{{base_url}}"
  headers:
    Accept: "application/json"
  timeout: "30s"
  follow_redirects: true
error_policy:
  on_error: stop
  on_4xx: continue
  on_5xx: stop
scenarios:
  - name: "Main"
    load:
      type: constant
      users: 10
      duration: 5m
      ramp_up: 10s
      iterations: 0
    steps:
      - get: /health
```

## Variables

- Syntax: `{{var}}` in URL, headers, query params, body strings/object string fields.
- Missing vars → empty string.

## data_source

```yaml
data_source:
  type: csv          # csv | txt
  file: "/path/users.csv"
  mode: per_vu       # per_vu | shared | per_worker
  strategy: sequential  # sequential | random | unique
  variable_names: "email,password"   # no header; positional
  # or header bind:
  bind:
    email_col: user_email
    pass_col: user_password
  on_exhausted: recycle  # recycle | stop
```

## http_defaults

- `timeout`: e.g. `30s`
- `follow_redirects`: follow 301/302/307/308 when true
- `retrieve_embedded_resources` + `embedded_resource_types` + `embedded_resource_blacklist`

## scenarios

Per scenario: `load`, `cookies`, `cache_manager`, `error_policy`, `steps`.

### Step types

- HTTP: `request`, `get`, `post`, `put`, `delete`, `patch`, `head`, `options`
- Data/DB: `data_source`, `sql`
- Containers: `group`, `controller`, `transaction`, `parallel`, `one_time`, `balanced`
- Flow: `if`, `loop`, `retry`
- Other: `assertions`, `think_time`

### Short vs long HTTP form

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

Object bodies are JSON-serialized; Relampo sets `Content-Type: application/json` when appropriate.

### Assertions

Short map:

```yaml
assert:
  status: 200
  body_contains: "ok"
```

List form types: `status`, `contains`, `not_contains`, `regex`, `jsonpath`/`json`, `header`, `response_time`/`response_time_max`, `response_size`.

### Extractors

Short map:

```yaml
extract:
  token: "jsonpath($.token)"
  req_id: "header.X-Request-Id"
```

List form types: `regex`, `jsonpath`/`json`, `header`, `xpath`, `boundary`, `body`.

Common list fields: `type`, `var`, `from` (`body`|`headers`|`status_line`), `expression`/`pattern`, `default`, `capture_mode` (`first`|`all`|`index`|`random`).

### Multipart

```yaml
files:
  - field: file
    path: /tmp/invoice.pdf
    mime: application/pdf
```

### Overrides

- Request `error_policy` overrides scenario, which overrides root.
- `cache_override` / `cookie_override`: `inherit` | `enabled` | `disabled`.

## Cookies / cache (scenario)

```yaml
cookies:
  policy: standard   # standard | ignore_cookies
  clear_each_iteration: false
cache_manager:
  clear_each_iteration: true
  max_elements: 1000
```

Full product docs: https://docs.relampo.com
