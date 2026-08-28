# Relampo gotchas

## Silent failures — read this first

These produce a test that passes and measures nothing.

- **Undefined `{{var}}` becomes an empty string**, not an error. An empty token still sends a request, and a clean 401 looks like a valid result.
- **Failed built-ins also become empty strings.** Bad arity, bad types, and bad ranges fail quietly.
- **Bare words inside built-ins are variables, not text.** `{{_upper(name)}}` reads a variable called `name`; `{{_upper("name")}}` uppercases the literal.
- **Built-in results are cached per request.** Two `{{_uuid()}}` calls in one request give the same value. Only `_counter` and `_sequence` advance every time.
- **`validate` will not catch any of this.** Guard every value that matters with an assertion:

```yaml
assertions:
  - { type: status, value: 200 }
  - { type: jsonpath, path: "$.data.token" }
```

## validate is lenient

`relampo validate` checks structure, not meaning. It accepts an unknown assertion type, an unknown load type, and a misspelled built-in. A passing validation means the file parses, not that the test is right. Check field names against [yaml-spec.md](yaml-spec.md).

## debug is not a terminal command

`relampo debug` is an alias of `relampo studio`. It opens the browser editor's Debug view and runs 1 VU and 1 iteration. It needs a person.

From a terminal use `relampo validate`. Then `relampo run` for load.

## Fields that parse but do nothing

- `http_defaults.retry_policy` — accepted, never read. Use a step-level `retry` block.
- `error_policy.on_4xx`, `on_5xx`, `on_timeout` — accepted, never read. Only `on_error` works.
- `metrics` — reporting configuration only. It does not change generated load.

## Assertions changed at 1.0.5

`pattern`, `expression`, `from`, `ignore_case`, and `enabled` are no longer assertion fields. An assertion has `type`, `value`, `path`, `name`, `max_ms`, `size`.

They are still valid on **extractors**. Do not copy assertion syntax from scripts written before 1.0.5.

## stages

- The CLI runs `load.type: stages`. **The editor rejects it.** An editor error here does not mean the file is broken.
- Stage durations must sum exactly to `load.duration`.
- `--vus`, `--duration`, and `--rps` are rejected for a staged scenario. Edit the stages instead.

## Editor lag

The editor's contract bundle is at 1.0.4 while the CLI is at 1.0.5. A field added in 1.0.5 can show as unknown in the editor and still run. The editor preserves unknown fields, so nothing is lost on save. Confirm with `relampo validate`.

## One scenario per file

The editor's Debug and Run views handle a single scenario. A second one blocks both. Split flows into separate files.

## HTTPS recording

- Encrypted traffic needs `relampo setup` and a **trusted Relampo CA**. Check with `relampo cert status`.
- Without trust, capture is incomplete or TLS fails.

## Recording

- Default redirect handling is JMeter-style replay, with `http_defaults.follow_redirects: true`. Duplicated redirect targets may be written as `enabled: false`.
- Use `--redirect-mode legacy` only when you need the older shape.
- `--responses` is required if you plan to correlate. Without it there are no response bodies to match against.
- `--exclude '*.(css|js)'` is not a valid glob. Use repeated flags, a comma list, or a regex.

## Correlation

- Do not apply every candidate at `--min-confidence 0.0` on a large site. Review, or use `--apply-ids`.
- Keep the raw recording. Write correlated output to a new file with `-o`.
- After applying, assert status codes on authenticated steps so an empty token fails loudly.

## Data sources

- Paths resolve on the machine running the test. Prefer absolute paths.
- `strategy: unique` runs out. Decide between `on_exhausted: recycle` and `stop` on purpose.
- `mode: per_worker` is rejected in a Studio debug run. Use `per_vu` while debugging.
- For distributed runs, upload data files first. A missing file fails on the workers, not locally.

## Secrets

- Never commit credentials in YAML. Pass them with `--var` or `--vars-file`.
- `--json-all` exports full request and response bodies and headers. It may contain secrets. Do not share those reports.
- Use `record --mask-sensitive` when recording flows that carry credentials.

## Distributed

- Needs an authenticated `gh`.
- Limits: **≤4 nodes**, **≤500 total VUs**, **≤1 hour**.

## Platform

- Windows: replace bash `\` line continuations with a backtick in PowerShell or `^` in CMD, or use one line.

## Scope of this skill

This skill teaches **using** Relampo, meaning YAML and the CLI. It is not the place to change the Relampo engine, licensing, or control plane.
