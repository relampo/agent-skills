# Maintaining this skill

## Ground truth

The skill documents the shipped Relampo product. When the skill and the product disagree, the product source wins for **what exists**, and the public docs win for **recommended usage**.

Sources, in order of authority:

1. **Relampo backend source** — the YAML specification document, the compatibility contract JSON (a code-generated field inventory, currently schema **1.0.5**), and the runtime built-in registrations.
2. **The YAML editor** — its bundled contract version, its semantic validation messages, and its product spec. The editor's bundle can lag the backend; today it is at 1.0.4.
3. **https://docs.relampo.com** — cross-check only. If the backend ships something the public docs do not list, document it here and report the docs gap.

## Re-syncing

1. Read the backend YAML specification and diff the field lists against `references/yaml-spec.md`.
2. Diff the compatibility contract's `known_fields` per node type against what the skill documents. This is the most reliable check available.
3. Re-read the built-in registration calls in the runtime and update `references/builtins.md`. Take function names and arity from the registrations, not from prose.
4. Re-run `<command> --help` for every command in `references/cli-commands.md`.
5. Check the editor's contract bundle version and update the lag note in `references/editor.md`.
6. Validate every example:

```bash
for f in skills/relampo/assets/examples/*.yaml; do relampo validate "$f"; done
```

`relampo validate` is **lenient**. It accepts unknown assertion types, unknown load types, and misspelled built-ins. It proves the file parses, not that it is correct. Verify field names against the contract, not against a passing validation.

7. Update `CHANGELOG.md` and `metadata.version` in `SKILL.md`.

## Publishing

skills.sh installs from the GitHub default branch and does not display `metadata.version`. Pushing to `main` is the release.

## Security scanning

skills.sh runs a security audit on the published skill. It flags installer-script URLs (those ending in `.sh` or `.ps1`) and any command that downloads a script and pipes it straight into a shell. Keep install instructions as a plain link to https://docs.relampo.com/cli/installation in both `SKILL.md` and `README.md`. Do not paste installer commands inline.

## Scope

This skill teaches **using** Relampo. Engine development belongs in the product source repository.
