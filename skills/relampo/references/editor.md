# Editor and Studio

Relampo has a visual editor for YAML test plans. It is a tool for people, not for agents. Know what it does so you can hand off to it, and so you can explain its warnings.

## Two commands

```bash
relampo editor                 # opens the editor in your browser
relampo studio flow.yaml       # opens the editor with the Debug view and the script mounted
relampo debug flow.yaml        # alias of studio
```

`relampo debug` no longer prints a terminal report. It opens the browser Debug view, which runs **1 VU and 1 iteration** for diagnosis, not load.

**From a terminal, use `relampo validate`.** It is the offline check, it needs no browser, and it covers HTTP, WebSocket, gRPC, and SQL structure.

## Capabilities

The editor runs on its own and edits YAML files. It never runs the `relampo` binary. Debug and Run work only when a Studio backend answers, and the backend advertises what it supports:

| Capability | Unlocks |
| --- | --- |
| `debug` | The Debug view |
| `loadRun` | The Run view |
| `dataSourceFiles` | The data-source file browser |

When no backend answers, the editor hides those controls and works as a plain editor. If a user reports a missing Debug tab, the backend is absent or has not advertised the capability.

## Limits that affect the YAML you write

**One scenario per file.** This is an engine limit too: the CLI/worker runtime executes only the first scenario. A successful validation of several scenarios does not mean all will run.

**Stage support depends on the editor build.** Older editors report:

> Load stages are documented but unsupported by the editor and Pulse runtime. Remove stages before running.

The baseline CLI supports staged load. If an editor rejects it, keep the staged file for `relampo run` and create a separate one-VU, one-iteration smoke file for diagnosis.

**Disabled nodes stay in the file.** They are written as `enabled: false` and do not execute. Recording writes duplicate redirect targets this way.

## Round-trip safety

The editor preserves fields it does not recognize, so hand-written and future YAML survives a save. It also keeps sibling keys next to shorthand steps such as `get:`.

It checks known fields against a contract bundle synced from the CLI source. Bundles are versioned; the editor source includes 1.0.10. The deployed editor or embedded Studio can use a different bundle. Check the active build before calling a warning a CLI incompatibility.

An editor warning is not proof the CLI will reject the file. Confirm with:

```bash
relampo validate flow.yaml
```

## Studio datasets

Uploaded CSV/TXT files persist in `<session-folder>/.relampo/datasources`. With `studio path/to/flow.yaml`, the session folder is the YAML directory. Without a file, it is the working directory at startup.

New and Upload YAML keep that session folder. Save downloads YAML only; it does not copy datasets or move the session folder. Keep `.relampo/datasources` beside the saved YAML when sharing or moving it.

Uploading the same filename replaces that dataset. A failed upload preserves the existing file. Unwritable session folders fail instead of falling back to temporary storage; managed directories cannot be symlinks. Studio does not delete datasets at shutdown.

Use synthetic data. Add `.relampo/datasources/` to `.gitignore` unless intentionally versioning it. Distributed preparation still commits active CSV/TXT datasets regardless of ignore rules.

## What an agent should do

- Write the YAML, check it with `relampo validate`, run it with `relampo run`.
- Suggest `relampo editor` or `relampo studio` when a person wants to inspect a flow visually or step through a Debug run.
- Use CLI workflows for agent automation. Offer Studio when a human needs request-level visual diagnosis; offline validation is not a substitute for inspecting a live flow.
