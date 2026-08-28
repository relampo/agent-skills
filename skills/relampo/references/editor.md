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

**One scenario per file.** The editor validates and runs a single scenario. A second scenario blocks Debug and Run. Write one scenario per file.

**`load.stages` is rejected by the editor.** It reports:

> Load stages are documented but unsupported by the editor and Pulse runtime. Remove stages before running.

The CLI runs staged load correctly. Only the editor refuses it. If a user needs stages, keep them and use `relampo run`; if they need the editor's Debug view, switch that file to `constant` or `ramp`.

**Disabled nodes stay in the file.** They are written as `enabled: false` and do not execute. Recording writes duplicate redirect targets this way.

## Round-trip safety

The editor preserves fields it does not recognize, so hand-written and future YAML survives a save. It also keeps sibling keys next to shorthand steps such as `get:`.

It checks known fields against a contract bundle synced from the CLI source. **That bundle currently sits at 1.0.4 while the CLI is at 1.0.5.** So a field added in 1.0.5 may show as unknown in the editor and still run correctly under `relampo run`.

An editor warning is not proof the CLI will reject the file. Confirm with:

```bash
relampo validate flow.yaml
```

## What an agent should do

- Write the YAML, check it with `relampo validate`, run it with `relampo run`.
- Suggest `relampo editor` or `relampo studio` when a person wants to inspect a flow visually or step through a Debug run.
- Do not drive the editor's HTTP API. Everything it offers, the CLI offers headlessly.
