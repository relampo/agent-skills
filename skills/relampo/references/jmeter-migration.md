# JMeter migration

Read this reference before translating a JMeter plan (`.jmx`) into Relampo YAML.
Use the same mappings in Claude, Cursor, Copilot, and other agents.

Baseline: CLI **v2.10.3**, compatibility contract **1.0.10**. This guide covers HTTP plans and common controllers.

## Migration workflow

1. List enabled Thread Groups, requests, scoped configuration, controllers, timers, scripts, and plugins in the source plan.
2. Produce one YAML file per Thread Group. Preserve request names and execution order.
3. Use the table below. Apply inherited configuration only to the requests it affects.
4. Report each approximation and unsupported component. Include its source name and the behavior still missing.
5. Validate each file. Prepare one VU and one iteration for an approved smoke run before restoring the original load.

For multiple Thread Groups, report whether the original plan runs them together or in sequence.
Separate YAML files require external coordination to preserve that schedule.
The CLI and workers execute only the first scenario in a file.

## Component mappings

These are behavior mappings. The notes identify cases that need manual work.
See [yaml-spec.md](yaml-spec.md) for complete field shapes.

| JMeter component | Relampo YAML | Migration rule |
| --- | --- | --- |
| Test Plan | `test`, `variables`, `http_defaults`, `scenarios` | Keep test metadata and use one scenario per file. |
| Thread Group (`ThreadGroup`) | `scenarios[0].load` | Threads → `users`; ramp-up seconds → `ramp_up` with a unit; loop count → `iterations`. Read the load rules below. |
| HTTP Request (`HTTPSamplerProxy`) | `request` | Copy `name`, `method`, `url`, headers, query, and body. Preserve form encoding, raw bodies, redirects, and uploads. |
| HTTP Request Defaults | `http_defaults` | Build `base_url` from protocol, host, and port. Keep scoped differences on the affected requests. |
| HTTP Header Manager | `http_defaults.headers` or `request.headers` | Put shared headers in defaults. Put scoped headers on each affected request. |
| User Defined Variables | `variables` | Keep initial values. Use data sources or hooks for values that change during execution. |
| Variable reference `${name}` | `{{name}}` | Use canonical interpolation. Treat JMeter properties and function calls separately. |
| JMeter functions | Built-ins such as `{{_uuid()}}` | Translate each function using [builtins.md](builtins.md). Match arguments and evaluation timing. |
| CSV Data Set Config (`CSVDataSet`) | `data_source` | Map columns, sharing, scope, and exhaustion explicitly. Read the CSV rules below. |
| Regular Expression Extractor (`RegexExtractor`) | `request.extractors`, `type: regex` | Reference name → `var`; expression → `pattern`; capture group → `group`. Translate match selection explicitly. |
| JSON Extractor (`JSONPostProcessor`) | `request.extractors`, `type: jsonpath` | Reference name → `var`; JSONPath → `path`; default value → `default`. Check the path against representative responses. |
| Boundary Extractor | `request.extractors`, `type: boundary` | Set `var`, `left_boundary`, `right_boundary`, and explicit capture selection. |
| XPath Extractor | `request.extractors`, `type: xpath` | Set `var` and `path`. Verify namespaces and selected text; XPath2 and CSS selectors need a separate review. |
| Response Assertion | `request.assertions` | Use `status`, `contains`, `not_contains`, `regex`, or `header` for the field and comparison being tested. |
| JSON Assertion | `request.assertions`, `type: jsonpath` | Use `path` and `value` for equality. Review other operators separately. |
| Duration Assertion | `request.assertions`, `type: response_time_ms` | Copy the limit in milliseconds to `max_ms`. |
| Size Assertion | `request.assertions`, `type: response_size` | Check the measured response and comparison. Do not assume every JMeter size operator has an equivalent. |
| Transaction Controller | `transaction: {name, steps}` | Keep the transaction name and child order. Compare reported timings; JMeter parent-sample and timer options are not equivalent fields. |
| Simple Controller | `group: {name, steps}` | Keep the logical group. Translate its scoped configuration onto supported fields. |
| Once Only Controller | `one_time: {name, steps}` | Runs once per VU. Review nested-loop placement and initialization order. |
| Loop Controller | `loop: N` with sibling `steps` | Use for a fixed child-loop count. Keep Thread Group loops in `load.iterations`. |
| If Controller | `if: "condition"` with sibling `steps` | Rewrite the condition in Relampo syntax. Review when the condition is evaluated. |
| Constant Timer | Standalone `think_time` before each affected request | Convert milliseconds to a duration, for example `500ms`. Expand inherited timer scope. |
| Uniform Random Timer | `think_time: {min, max, distribution: uniform}` | Set `min` to the constant offset and `max` to offset plus random range. |
| Constant Throughput Timer | `load.type: throughput` or `request.throughput` | Approximation: convert samples/minute to requests/second. Review timer scope and shared versus per-thread behavior. |
| Throughput Controller | Review `balanced` for alternative traffic branches | A controller chooses executions; a throughput load sets a request rate. Preserve skipped executions and percentages explicitly. |
| HTTP Cookie Manager | Scenario `cookies` | Preserve `clear_each_iteration`, cookie policy, and seeded cookie domains. |
| HTTP Cache Manager | Scenario `cache_manager` | Preserve supported cache options. Compare cache-hit behavior before comparing results. |
| HTTP Authorization Manager | `http_defaults.auth` or container `auth` | Supports basic, bearer, and API key auth. Scope through a container; a plain request has no `auth` field. Review other schemes separately. |
| JSR223/BeanShell pre/post processor | Request-local `spark` hooks | Manually rewrite supported logic as JavaScript. Prefer extractors and built-ins. Read [spark.md](spark.md). |
| Listeners / JTL output | CLI report flags `--html`, `--json`, `--csv` | Relampo produces its own report formats. Review metric definitions before comparing results. |

## Behavior differences

### Load and control flow

- Use `load.type: constant` for a standard closed Thread Group with a fixed user count.
- Convert numeric JMeter seconds to durations such as `30s`. `load.iterations` is a count per VU.
- For duration-based loops, use `duration` and omit a positive `iterations` limit. Review plans that combine both limits.
- Preserve run-forever behavior with `run_until_stopped: true`; it cannot combine with `duration` or `iterations`.
- Treat scheduler delays and plugin Thread Groups as manual mappings. Use [load-and-intent.md](load-and-intent.md) for supported profiles.
- Keep `one_time` for per-VU work. A write that must run once for the full test needs external fixture preparation.
- Review While, ForEach, Module, Interleave, setUp, and tearDown controllers separately. A bounded `loop` or `group` does not preserve them automatically.
- Map HTTP error actions to `error_policy`: `continue`, `next_iteration`, or `stop_user`. Whole-test stop actions need a separate solution.

### Scope and timers

Expand inherited timers, assertions, and post-processors onto each affected request.
Place standalone `think_time` steps **before** requests to preserve JMeter timer placement.
An inline `request.think_time` runs **after** the request in this baseline.
When several timers affect a request, preserve each delay before that request.
JMeter documents timer scope in its [component reference](https://jmeter.apache.org/usermanual/component_reference.html#timers).

### Extractors and assertions

- Attach an extractor to the request that produces its value. Reuse the result as `{{var}}` in later requests.
- Set `default` explicitly. Assert on required response values and select an error policy before sending dependent requests.
- Relampo uses Go regular expressions. Rewrite lookaround and pattern backreferences using the [Go regexp syntax](https://pkg.go.dev/regexp/syntax).
- Use `group: 1` for a single capture group. Legacy templates such as `$1$-$2$` are supported.
- Translate JMeter match selection using `capture_mode`, as shown below. Raw `match_no` values have different meanings.
- `capture_mode: all` returns comma-joined values. It does not create JMeter's numbered variables or match-count variables.
- JSONPath extraction does not use the regex capture-selection path. Review filters, multiple matches, and JMESPath expressions separately.
- Keep literal containment and regex matching distinct. Relampo regex assertions use `value`; regex extractors use `pattern`.

| JMeter regex match selection | Relampo fields |
| --- | --- |
| First match (`1`) | `capture_mode: first` |
| Match N (`N > 0`) | `capture_mode: index`, `capture_index: N` |
| Random match (`0`) | `capture_mode: random` |
| All matches (negative) | `capture_mode: all`; rewrite consumers of numbered variables |

The source semantics are documented under [Regular Expression Extractor](https://jmeter.apache.org/usermanual/component_reference.html#Regular_Expression_Extractor).
In this Relampo baseline, legacy `match_no: 0` means all matches and `match_no: -1` means random.

### CSV data and variables

Read [data-sources.md](data-sources.md) before choosing a mode.

| JMeter data behavior | Relampo mapping and limit |
| --- | --- |
| One cursor shared by threads | `mode: shared`; shared only within one process and one data-source node |
| Independent cursor per thread | `mode: per_vu`; every VU starts at the first row |
| Recycle on EOF | `on_exhausted: recycle` |
| Stop thread on EOF | Manual gap: `on_exhausted: stop` stops the current iteration, not the VU |
| Header row names | `bind: {column_name: variable_name}`; consumes the header |
| Explicit names without a header | `variable_names: "email,password"`; every row is data |

Root data binds once per VU iteration. Step and request bindings advance when their execution reaches that node.
Match the source plan's data-advance timing before moving a CSV configuration into a nested scope.
Separate nodes have separate cursors even if they read the same file.
`per_worker` is a fixed row mapping by VU index; it does not partition data among workers.
Review custom delimiters, encodings, and EOF sentinel handling. Normalize the file when the requested option has no documented field.

Convert JMeter properties such as `${__P(host)}` to explicit variables supplied through `--var` or `--vars-file`.
Relampo replaces ordinary missing variables with empty strings. JMeter can retain an unresolved variable reference.
See [JMeter functions and variables](https://jmeter.apache.org/usermanual/functions.html) for that source behavior.

## Complete migration example

Source plan: one Thread Group with a shared CSV, a Transaction Controller, two HTTP requests, and a scoped 500 ms timer.
The login response supplies a JSON token and a regex CSRF value for the profile request.
The YAML starts with one VU and one iteration for a smoke run.

Save this as `jmeter-login.yaml`:

```yaml
test:
  name: "Migrated JMeter login"
  version: "1.0"

variables:
  base_url: "https://api.example.com"

data_source:
  type: csv
  file: users.csv
  mode: shared
  strategy: sequential
  on_exhausted: recycle
  bind:
    email: email
    password: password

http_defaults:
  base_url: "{{base_url}}"
  timeout: 10s
  headers:
    Accept: application/json
    Content-Type: application/json

scenarios:
  - name: "Login users"
    load:
      type: constant
      users: 1
      iterations: 1
    cookies:
      policy: standard
      clear_each_iteration: true
    error_policy:
      on_error: stop_user
    steps:
      - transaction:
          name: "Login and profile"
          steps:
            - think_time: 500ms
            - request:
                name: "Login"
                method: POST
                url: /api/login
                body:
                  email: "{{email}}"
                  password: "{{password}}"
                extractors:
                  - type: jsonpath
                    var: access_token
                    path: $.token
                    default: ""
                  - type: regex
                    var: csrf
                    pattern: '"csrf"\s*:\s*"([^"]+)"'
                    group: 1
                    capture_mode: first
                    default: ""
                assertions:
                  - {type: status, value: 200}
                  - {type: regex, value: '"token"\s*:\s*"[^"]+"'}
                  - {type: regex, value: '"csrf"\s*:\s*"[^"]+"'}
            - think_time: 500ms
            - request:
                name: "Profile"
                method: GET
                url: /api/me
                headers:
                  Authorization: "Bearer {{access_token}}"
                  X-CSRF-Token: "{{csrf}}"
                assertions:
                  - {type: status, value: 200}
```

Save synthetic `users.csv` beside the YAML:

```csv
email,password
ada@example.com,synthetic-password-1
grace@example.com,synthetic-password-2
```

Validate without sending traffic:

```bash
relampo validate jmeter-login.yaml
```

After target approval, run the smoke flow against the chosen test environment:

```bash
relampo run jmeter-login.yaml --var base_url=https://staging.example.com --html smoke.html --json smoke.json
```

Compare the request sequence, wire data, extracted values, timer placement, CSV rows, cookies, assertions, and transaction timing.
Then restore the source Thread Group's load settings and list any remaining differences.
Validation alone does not prove runtime equivalence.
