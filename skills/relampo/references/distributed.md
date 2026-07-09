# Distributed execution

Relampo can run a scenario across multiple **GitHub Actions** workers without self-managed load generators.

## Prerequisites

- `git` and GitHub CLI (`gh`)
- Authenticated CLI: `gh auth login`
- Scenario (and any CSV/assets) available to upload into the distributed repo

## Limits (CLI contract)

- Up to **4** distributed nodes
- Up to **500** total VUs
- Up to **1 hour** per run

## Commands

Prepare environment (repo):

```bash
relampo distributed setup --repo owner/perf-tests
# short form often works as --repo perf-tests under the authenticated user/org
```

Upload scripts and data files (`--files` is a comma-separated list **inside brackets and quotes**):

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

What happens: Relampo prepares/uses a GitHub repository, places the scenario under `scenarios/`, generates the Actions workflow, triggers the run by default, watches execution, and can export aggregated reports locally.

## Data sources

Files referenced by `data_source` must exist for the process running the test. On GitHub Actions, upload them into the distributed repository before or as part of the run workflow.

## Windows tip

On Windows, replace bash `\` line continuations with PowerShell backtick (`` ` ``) or CMD `^`, or run as a single line.
