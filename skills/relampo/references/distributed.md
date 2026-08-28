# Distributed execution

Relampo runs a scenario across several **GitHub Actions** workers, so you do not manage load generators yourself.

## Prerequisites

- `git` and the GitHub CLI (`gh`)
- An authenticated CLI: `gh auth login`
- The scenario and any data files ready to upload

## Limits (CLI contract)

- Up to **4** nodes
- Up to **500** total VUs
- Up to **1 hour** per run

## Commands

Prepare the repository:

```bash
relampo distributed setup --repo owner/perf-tests
```

Upload scripts and data files. `--files` is a comma-separated list **inside brackets and quotes**:

```bash
relampo distributed files upload \
  --repo owner/perf-tests \
  --files "[/path/test.yaml,/path/users.csv]"
```

Run:

```bash
relampo distributed run /path/test.yaml \
  --repo owner/perf-tests \
  --nodes 2 \
  --vus 100 \
  --duration 10m \
  --html distributed-report.html
```

Relampo prepares the repository, places the scenario under `scenarios/`, generates the Actions workflow, triggers the run, watches it, and exports aggregated reports locally.

## Reports

CSV and text reports can include a **request events** section, one row per request, when the run produced per-request events. Columns:

```
request_event, timestamp, started_at, vu, request_id, step_path, name, method,
path, status, latency_ms, error, transaction, transaction_id, chain_id,
chain_role, redirect_index, redirect_source, protocol, operation, protocol_status
```

Bodies, headers, redirect payloads, and assertion details are stripped before export, so these files are safer to share than `--json-all`.

## Data sources

Files named by `data_source` must exist for the process running the test. Upload them to the distributed repository first. A missing file fails on the workers, not on your machine.

Consider `mode: per_worker` so each worker reads its own slice. See [data-sources.md](data-sources.md).

## Windows tip

Replace bash `\` line continuations with a backtick in PowerShell or `^` in CMD, or write the command on one line.
