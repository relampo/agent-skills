# Relampo gotchas

Baseline: CLI **v2.10.3**, compatibility contract **1.0.10**.

## Missing values and failed built-ins

Assert on required response values before scaling a flow.

- **Undefined `{{var}}` becomes an empty string**, not an error. An empty token still sends a request, and a clean 401 looks like a valid result.
- **Failed built-ins block HTTP preparation.** Bad names, syntax, arity, and constant types are also caught by validation. Dynamic types and ranges still need runtime checks.
- **Bare words inside built-ins are variables, not text.** `{{_upper(name)}}` reads a variable called `name`; `{{_upper("name")}}` uppercases the literal.
- **Direct calls evaluate per occurrence.** Reuse a declared variable for one generated value within a request. `_counter` state is per VU, not globally unique.
- **Validation does not prove successful behavior.** Missing ordinary variables can still become empty strings. Guard required response values:

```yaml
assertions:
  - { type: status, value: 200 }
  - { type: jsonpath, path: "$.data.token" }
```

## What validation proves

`relampo validate` checks structure and built-in templates. It does not prove API success, dynamic argument correctness, or that every supplied field executes. Unknown model fields can be ignored. Confirm fields against [yaml-spec.md](yaml-spec.md), then run an approved one-VU smoke test.

## debug is not a terminal command

`relampo debug` is an alias of `relampo studio`. It opens the browser editor's Debug view and runs 1 VU and 1 iteration. It needs a person.

From a terminal use `relampo validate`. Then `relampo run` for load.

## Fields that parse but do nothing

- `http_defaults.retry_policy` — accepted, never read. Use a step-level `retry` block.
- Root `cookies`/`cache_manager` and scenario `think_time` — not execution fields. Put cookie/cache blocks inside the scenario and pauses in steps or requests.
- `metrics` — reporting configuration only. It does not change generated load.

## HTTP policies and Spark

`on_4xx`, `on_5xx`, and `on_timeout` work for HTTP requests. `next_iteration` skips the current iteration; `stop_user` stops the failing VU. Legacy `stop` does not stop the whole test. See [error_policy](yaml-spec.md#error_policy).

Failed Spark scripts log an error and roll back that script's variable changes, but do not abort the request. Do not use a thrown JavaScript error as a flow-control guard. See [spark.md](spark.md).

## Assertions changed at 1.0.5

`pattern`, `expression`, `from`, `ignore_case`, and `enabled` are no longer assertion fields. An assertion has `type`, `value`, `path`, `name`, `max_ms`, `size`.

`pattern`, `expression`, and `from` are valid on extractors. `ignore_case` and `enabled` are not runtime extractor fields. Use `(?i)` for a case-insensitive regex. Do not copy assertion syntax from scripts written before 1.0.5.

## stages

- The CLI runs `load.type: stages`. Some editor builds reject it. An editor warning does not prove a CLI failure.
- Stage durations must sum exactly to `load.duration`.
- `--vus`, `--duration`, and `--rps` are rejected for a staged scenario. Edit the stages instead.

## Editor lag

Check the active editor/Studio build instead of assuming a fixed contract lag. The editor source includes 1.0.10, but deployed builds can differ. Preservation of an unknown field is not proof that it executes. See [editor.md](editor.md).

## One scenario per file

The CLI and workers execute only the first scenario, although validation checks all. Split flows into separate files. Multi-scenario planning documents are not shipped execution support.

## one_time and uploads

`one_time` executes once per VU, not once per test. Do not use it to guard a global database reset. Prepare global fixtures outside load.

Missing multipart files can be logged and skipped. Attachment paths use the working directory, unlike CSV fallback and media YAML-relative paths. Assert on the upload result.

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

- CSV/TXT paths resolve from the working directory first, then the YAML directory.
- `strategy: unique` runs out. Decide between `on_exhausted: recycle` and `stop` on purpose.
- `mode: per_worker` is rejected in a Studio debug run. Use `per_vu` while debugging.
- `per_vu` can reuse the same rows across VUs. `per_worker` is VU-to-row mapping, not worker partitioning.
- `on_exhausted: stop` stops the current VU iteration, not the whole run.
- Distributed preparation packages active datasets and fails locally for missing files. It commits them even if ignored.
- Studio uploads persist in `.relampo/datasources`; Save downloads YAML only.

## Secrets

- Keep credentials out of YAML and command literals. Use environment-derived values or an untracked vars file.
- `--json-all` exports full request and response bodies and headers. It may contain secrets. Do not share those reports.
- Use `record --mask-sensitive` when recording flows that carry credentials.

## Distributed

- Use authenticated `gh` for GitHub or `glab` for GitLab. The old `distributed` command is removed.
- Free tier: **4 nodes**, **500 total VUs**, **1 hour**. Licensed limits depend on the active license.
- Preparation can commit and push files. `--trigger=false` prevents execution, not publication. Get explicit approval.
- Report flags take filenames only. Download GitHub reports or GitLab job artifacts manually; local `run` still writes local reports.

## Platform

- Windows: replace bash `\` line continuations with a backtick in PowerShell or `^` in CMD, or use one line.

## Scope of this skill

This skill teaches **using** Relampo, meaning YAML and the CLI. It is not the place to change the Relampo engine, licensing, or control plane.
