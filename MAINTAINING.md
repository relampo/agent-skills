# Maintaining this skill

## Ground truth

The skill documents the shipped Relampo product. When the skill and the product disagree, the product source wins for **what exists**, and the public docs win for **recommended usage**.

Sources, in order of authority:

1. **Released Relampo backend source and binary** — runtime code, tests, compatibility contract, and command help. Baseline: CLI **v2.10.3**, contract **1.0.10**, schema version **1**. `test.version` is metadata.
2. **The YAML editor** — the active build's contract bundle and semantic messages. Source includes a 1.0.10 bundle, but deployment versions can differ.
3. **https://docs.relampo.com** — cross-check only. If the backend ships something the public docs do not list, document it here and report the docs gap.

## Re-syncing

1. Find the latest stable release. Compare local files with that release before documenting them. Planning docs and unmerged changes are not shipped features.
2. Diff the compatibility contract's `known_fields` per node type against what the skill documents. This is the most reliable check available.
3. Re-read the built-in registration calls in the runtime and update `references/builtins.md`. Take function names and arity from the registrations, not from prose.
4. Re-run `<command> --help` for every command in `references/cli-commands.md`.
5. Check the active editor/Studio version. Keep compatibility notes version-aware rather than claiming a permanent lag.
6. Validate every example:

```bash
relampo validate skills/relampo/assets/examples/health-smoke.yaml
```
Repeat for every YAML under `assets/examples/`, including the Spark and scoped-data examples. Use a temporary checksum-verified release binary if the installed CLI differs. Keep the installed CLI unchanged.

`validate` checks structure and built-in names, syntax, arity, and constant types. Unknown model fields and dynamic values still need source checks. A passing validation does not prove a successful API flow. Verify field execution, not only field presence in a contract.

7. Update `CHANGELOG.md` and `metadata.version` in `SKILL.md`.
8. Check local reference links, example data paths, command flags, and the one-scenario limit. Sync installed copies only after validation. Preserve unrelated local files.

## Publishing

skills.sh installs from the GitHub default branch and does not display `metadata.version`. Pushing to `main` is the release.

## Security scanning

skills.sh runs a security audit on the published skill. It flags installer-script URLs (those ending in `.sh` or `.ps1`) and any command that downloads a script and pipes it straight into a shell. Keep install instructions as a plain link to https://docs.relampo.com/cli/installation in both `SKILL.md` and `README.md`. Do not paste installer commands inline.

## Scope

This skill teaches **using** Relampo. Engine development belongs in the product source repository.
