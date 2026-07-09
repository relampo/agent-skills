# Relampo gotchas

## Install / PATH

- Installer puts binaries under `~/.local/bin` on many systems. Ensure `PATH` includes it before diagnosing “command not found”.
- Always verify with `relampo version` after install.

## HTTPS recording

- Encrypted traffic needs `relampo setup` and a **trusted Relampo CA**. Check `relampo cert status`.
- Without trust, HTTPS capture will be incomplete or fail TLS.

## Recording redirects

- Default recording follows JMeter-style redirect replay: `http_defaults.follow_redirects: true`.
- Immediately duplicated redirect targets may be written with `enabled: false`.
- Use `--redirect-mode legacy` only if you intentionally need the older shape.

## Variables

- Undefined `{{var}}` becomes **empty string**, not an error. Guard critical tokens with assertions (`status`, `jsonpath`) so empty auth fails loudly.

## Correlation

- Do not blindly `--apply` every candidate at `--min-confidence 0.0` on large sites — review or use `--apply-ids`.
- Keep the raw recording; write correlated output with `-o`.

## debug vs run

- `relampo debug` / `studio` runs **1 VU, 1 iteration** for diagnosis in the visual editor — not a load test.
- Use `relampo run` for load and report export.
- The legacy standalone `report` command is removed; use `run` / `distributed run` export flags.

## Check after edits

```bash
relampo debug scenario.yaml
```

Always re-check after manual YAML edits or selective correlation apply. `relampo run` validates YAML before load starts; invalid scripts fail early.

## Distributed

- Requires authenticated `gh`.
- Respect CLI limits: ≤4 nodes, ≤500 VUs, ≤1h.
- Upload CSV/assets the scenario needs; missing files fail only when workers run.

## Platform

- Windows: avoid bash-only multi-line examples; use single-line commands or native line continuation.
- Prefer absolute paths for `data_source.file` and upload lists when cwd is ambiguous.

## Scope of this skill

- This skill teaches **using** Relampo (YAML + CLI). It is not the place to change Relampo’s Go engine, licenses, or SaaS control plane.
