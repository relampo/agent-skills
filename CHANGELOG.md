# Changelog

## 1.1.0

Refresh against released CLI **v2.10.3** and compatibility contract **1.0.10**.

### Added

- Spark before/after HTTP hook reference and a complete example.
- Root, step, and request CSV bindings with bundled synthetic datasets.
- GitLab.com and Self-Managed execution examples, runner tags, and provider prerequisites.
- Persistent Studio dataset storage, encoded WebRTC media limits, HTTP connection scope, and DNS cache fields.
- Explicit approval boundaries for distributed publication and dataset exposure.

### Changed

- Replace removed `distributed` commands with `github` and document `gitlab`.
- Separate free-tier limits from active license limits.
- Explain automatic CSV/TXT packaging, including files ignored by the execution repository.
- Distributed report flags take filenames only; download GitHub publications or GitLab job artifacts manually.
- Document built-in static validation, HTTP preparation failures, and per-occurrence generation versus request-cached variables.
- Keep editor compatibility notes version-aware instead of assuming a fixed contract lag.

### Fixed

- Correct CSV cursor modes, uniqueness, exhaustion, and relative path resolution.
- Require one scenario per file for CLI and workers, not only the editor.
- Explain `one_time` per-VU execution and working HTTP status/timeout policies.
- Correct WebRTC signaling order and protocol-specific assertion support.
- Remove unsupported `sql_connection` and standalone throughput guidance.
- Correct root cookie/cache fields, scenario pauses, multipart failure behavior, and per-VU counter uniqueness assumptions.

## 1.0.1

Full refresh against Relampo YAML contract **1.0.5**, tested with CLI builds from 2026-08-26.

### Added

- `references/builtins.md` — about 180 built-in functions across nine families, with canonical `{{_name(args)}}` syntax, argument rules, error codes, and Java date-format tokens.
- `references/protocols.md` — WebSocket, gRPC, WebRTC, and SQL steps.
- `references/data-sources.md` — `data_source` CSV and TXT parameterization, with `mode`, `strategy`, and `on_exhausted`.
- `references/editor.md` — editor and Studio behaviour, capability gating, and the one-scenario-per-file limit.
- `SKILL.md` now documents six built-in rules that prevent silent empty values.
- YAML coverage for `think_time`, `retry`, `error_policy`, `throughput`, `one_time`, `balanced`, `if`, `loop`, `auth` inheritance, `cookies`, `cache_manager`, `metrics`, and the expanded `http_defaults`.
- CLI coverage for `relampo validate`, `relampo studio`, `relampo record grpc`, and `relampo run --rps`.
- Full `load` block reference including SLO thresholds and `run_until_stopped`.
- New examples: `builtins-data-gen.yaml`, `websocket-flow.yaml`, `intent-capacity.yaml`.

### Changed

- **Pre-load check is now `relampo validate`, not `relampo debug`.** `debug` is an alias of `studio` and opens the browser editor, so it cannot run headlessly.
- Assertions documented at 1.0.5: `type`, `value`, `path`, `name`, `max_ms`, `size`. The fields `pattern`, `expression`, `from`, `ignore_case`, and `enabled` are no longer assertion fields. They remain valid on extractors.
- Examples and docs prefer `path` over the `expression` alias in extractors.
- Install instructions now link to https://docs.relampo.com/cli/installation instead of embedding installer script URLs.
- Guidance to keep one scenario per file.

### Fixed

- Documented that `load.type: stages` runs in the CLI but is rejected by the editor, and that `--vus`, `--duration`, and `--rps` are rejected for staged scenarios.
- Documented that the editor's contract bundle lags the CLI (1.0.4 against 1.0.5), so unknown-field warnings there are not CLI errors.
- Documented that `relampo validate` is lenient — it accepts unknown assertion types, load types, and built-in names.
- Marked `http_defaults.retry_policy` and `error_policy.on_4xx`/`on_5xx`/`on_timeout` as accepted by the schema but not read by the engine.

## 1.0

Initial Relampo agent skill pack.
