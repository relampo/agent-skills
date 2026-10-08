# Spark HTTP request hooks

Baseline: CLI v2.10.3, contract 1.0.10. Spark runs small JavaScript programs inside an HTTP request.

## Shape and order

Put `spark` inside `request`. Each entry has `name`, `when`, and `script`. `when` accepts `before` or `after`; omitted means `after`.

Execution order:

1. Bind request-local CSV/TXT data.
2. Run `before` hooks.
3. Interpolate URL, headers, query, auth, body, and file fields.
4. Execute the HTTP request.
5. Run extractors.
6. Run `after` hooks.
7. Evaluate assertions.

After hooks also run for cache hits. Hooks execute in list order within their phase.

```yaml
spark:
  - name: normalize email
    when: before
    script: |
      vars.set("normalized_email", String(vars.get("email") || "").toLowerCase());
  - name: inspect response
    when: after
    script: |
      vars.set("last_status", response.status);
      vars.set("last_latency_ms", response.latency_ms);
```

See [spark-hooks.yaml](../assets/examples/spark-hooks.yaml) for a complete scenario.

## Supported API

- `vars.get("key")`: read a variable.
- `vars.set("key", value)`: write a variable used by later interpolation, hooks, or assertions.
- `log(value)` and `console.log(value)`: log a value. Keep secrets out of logs.
- After hooks only: `response.status`, `response.body` (string), and `response.latency_ms`.

Use `JSON.parse(response.body)` for JSON response logic. Prefer a JSONPath extractor when it can do the same task.

The runtime uses Goja, not a browser or Bun/Node runtime. It creates a new VM for each phase invocation. Store persistent values through `vars`, not JavaScript globals. There is no documented `request` mutation object, `response.headers`, `fetch`, `require`, or module API.

## Failure behavior

If a script fails, Relampo logs the failure and rolls back that script's variable changes. It does not abort the HTTP request. Earlier successful hooks keep their changes.

Throwing an error is not a reliable way to stop a flow. Assert on required output and use the HTTP error policy for request failures. Give transformed variables safe defaults so a failed hook cannot reuse an old result unnoticed.

Spark is request-local. Standalone `spark` steps and global hooks are not supported.
