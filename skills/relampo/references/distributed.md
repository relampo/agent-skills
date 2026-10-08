# Distributed execution

Relampo runs one scenario across GitHub Actions or GitLab CI workers. Baseline: CLI **v2.10.3**.

## Prerequisites

- GitHub: `git`, `gh`, and `gh auth login`.
- GitLab: `git`, `glab`, and `glab auth login --hostname gitlab.com` (use your host for Self-Managed).
- A dedicated execution repository/project, separate from application CI.
- Exactly one scenario, valid local files, and approved test targets.

## Publication boundary

`setup`, `push`, `run`, and file actions can create or modify execution repositories, commit files, and push them. Obtain explicit approval before running them.

`--trigger=false` prevents starting a workflow/pipeline, not commits or pushes. `--watch=false` starts without waiting. Neither flag makes publication read-only.

Active CSV/TXT datasets are copied to `scenarios/data`, committed even if ignored, and referenced with portable YAML paths. Inspect them for secrets/PII first. Preparation fails for missing datasets before replacing an existing scenario.

## Limits

Free tier: **4 nodes**, **500 total VUs**, **1 hour**. Licensed runs use active license limits. Check `relampo license status`. VUs are the total across nodes, not per node.

Load by Intent requires a single node in this baseline. Do not suggest a multi-node intent search.

## Commands

The old `distributed` command is removed. Replace it with `github`.

```bash
relampo github setup --repo owner/perf-tests
relampo github run scenario.yaml --repo owner/perf-tests --nodes 2 --vus 100 --duration 10m --html report.html
```

GitLab example:

```bash
relampo gitlab setup --repo group/perf-tests
relampo gitlab run scenario.yaml --repo group/perf-tests --nodes 2 --vus 100 --duration 10m --html report.html
```

For Self-Managed:

```bash
relampo gitlab run scenario.yaml --repo group/subgroup/perf-tests --host gitlab.example.com --runner-tags load-test --nodes 2 --vus 100 --duration 10m
```

GitLab accepts a project name, `group/subgroup/project`, or full HTTPS project URL. `--host` defaults to `GITLAB_HOST`, then `gitlab.com`. `--runner-tags` selects worker runners. Relampo generates `.gitlab-ci.yml` and rejects unrelated CI configuration. Preparation and Git pushes do not start a GitLab pipeline; the run command triggers it explicitly by default.

## Files and preparation

Active CSV/TXT files are packaged automatically. Upload other approved files when needed. File actions accept a bracketed comma-separated list:

```bash
relampo github files upload --repo owner/perf-tests --files "[scenario.yaml,users.csv]"
relampo gitlab files upload --repo group/perf-tests --files "[scenario.yaml,users.csv]"
relampo gitlab push scenario.yaml --repo group/perf-tests
relampo gitlab run scenario.yaml --repo group/perf-tests --trigger=false
```

`push` publishes without starting execution. `run` defaults to `--trigger=true --watch=true`.

## Reports

In v2.10.3, GitHub Actions and GitLab pipelines generate reports remotely. GitLab pipeline report generation was added in v2.10.3.

`--json`, `--html`, `--csv`, and `--text` accept **filenames only**, such as `report.html`, not `out/report.html`. Download reports manually through GitHub publication links or GitLab job artifacts. These flags require `--watch=true`. `--watch=false` starts without waiting for report generation and publication.

Local `relampo run` report flags still write local file paths. Keep local and distributed report behavior separate.

CSV and text reports can include a **request events** section, one row per request, when the run produced per-request events. Columns:

```
request_event, timestamp, started_at, vu, request_id, step_path, name, method,
path, status, latency_ms, error, transaction, transaction_id, chain_id,
chain_role, redirect_index, redirect_source, protocol, operation, protocol_status
```

Bodies, headers, redirect payloads, and assertion details are stripped from public-safe request events. URLs, names, query values, and errors can still expose sensitive data. Review reports before sharing.

## Data sources

`per_worker` is fixed VU-to-row mapping, not worker partitioning. `shared` shares a cursor within each worker process only. Use explicit worker-specific datasets when cross-worker uniqueness matters. See [data-sources.md](data-sources.md).

## Windows tip

Replace bash `\` line continuations with a backtick in PowerShell or `^` in CMD, or write the command on one line.
