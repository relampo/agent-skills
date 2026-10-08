# Relampo CLI commands

Baseline: CLI **v2.10.3**. Check installed `--help` when using another version.

Help for any command:

```bash
relampo help
relampo record --help
```

## setup / version / cert / license

```bash
relampo setup            # environment and CA material for HTTPS recording
relampo version
relampo cert status      # also: cert install, cert export
relampo license          # CLI license and enterprise entitlements
```

If `relampo` is not found after install, add `~/.local/bin` to `PATH`.

## validate

Check a scenario offline. **This is the check to use from a terminal.**

```bash
relampo validate flow.yaml
```

Covers HTTP, WebSocket, gRPC, and SQL structure. No browser and no server needed.

It checks built-in names, syntax, arity, and constant argument types as well as structure. Dynamic argument values need runtime checks. Unknown fields can be ignored by the model. Passing validation is not proof of correct assertions or successful API behavior.

## record

Capture HTTP(S) and WebSocket traffic through a local proxy and write YAML.

```bash
relampo record --port 8888 --responses --output my-test.yaml
```

| Flag | Purpose |
| --- | --- |
| `--port` | Proxy port, default 8888 |
| `--responses` | Capture response headers and bodies. **Needed for correlation** |
| `--output` | Output YAML file |
| `--force` | Overwrite an existing output file |
| `--base-url` | Base URL(s) of the app, comma-separated. Auto-detected if empty |
| `--include` | Record only URLs matching this pattern |
| `--exclude` | Skip URLs. Repeatable. Glob (`*.css`), extensions (`png,jpg`), or regex |
| `--grouping` | `none` (default), `separator`, `controller`, `transaction`, `first-only` |
| `--prefix` | Ask for a step-name prefix, and let you change it as the flow moves |
| `--name` | Value for `test.name` |
| `--mask-sensitive` | Mask sensitive values in the generated YAML |
| `--redirect-mode` | `jmeter` (default) or `legacy` |
| `--redirect-disabling` | Disable duplicate redirect-target requests. Default true |
| `--preserve-compressed-responses` | Keep bodies compressed (legacy) |

Point the client at the proxy, exercise the flow, then stop with Ctrl+C.

`--exclude '*.(css|js)'` is not a valid glob. Use repeated flags, a comma list, or a regex.

### record grpc

```bash
relampo record grpc --upstream api.example.com:443 --protoset ./api.protoset --output grpc-flow.yaml
```

| Flag | Purpose |
| --- | --- |
| `--upstream` | **Required.** TLS target `host:port` |
| `--listen` | Local address, default `127.0.0.1:50051` |
| `--protoset` | Compiled descriptor set |
| `--reflection` | Use server reflection. Default true |
| `--responses` | Retain payloads |
| `--output`, `--force`, `--mask-sensitive`, `--name` | As with `record` |

## converter

```bash
relampo converter
```

Opens the browser converter to turn collections into Relampo YAML.

## correlate

```bash
relampo correlate my-test.yaml
relampo correlate my-test.yaml --apply -o my-test.correlated.yaml
relampo correlate my-test.yaml --apply-ids "1-3,5" -o selected.yaml
relampo correlate my-test.yaml --min-confidence 0.9
relampo correlate my-test.yaml --json
```

| Flag | Purpose |
| --- | --- |
| `-a, --apply` | Write correlated YAML |
| `--apply-ids` | Apply only these candidate IDs, e.g. `1,2,5` or `1-4,7` |
| `-o, --output` | Output file |
| `--min-confidence` | 0.0 to 1.0, default **0.25** |
| `--json` | Save results as JSON |
| `--json-output` | JSON output path |

See [correlation.md](correlation.md).

## editor / studio / debug

```bash
relampo editor                 # visual editor
relampo studio flow.yaml       # editor with the Debug view, script mounted
relampo debug flow.yaml        # alias of studio
```

All three open a browser. `debug` runs 1 VU and 1 iteration for diagnosis, not load. For a headless check use `relampo validate`. See [editor.md](editor.md).

## run

```bash
relampo run my-test.yaml
relampo run my-test.yaml --vus 50 --duration 10m
relampo run my-test.yaml --rps 200 --duration 10m
relampo run my-test.yaml --var base_url=https://staging.example.com
relampo run my-test.yaml --vars-file env/staging.yaml
relampo run my-test.yaml --html report.html --json report.json --csv report.csv --text report.txt
```

| Flag | Purpose |
| --- | --- |
| `--vus` | Override virtual users |
| `--duration` | Override duration, e.g. `10s`, `1m` |
| `--rps` | Override target requests per second. Must be above 0 |
| `--var` | `key=value`, repeatable |
| `--vars-file` | Variables from a YAML or JSON file |
| `--html`, `--csv`, `--text` | Export reports |
| `--json` | Summary JSON. Takes precedence over `-o` |
| `--json-all` | Full request and response bodies and headers. **May contain secrets** |
| `-o, --output` | Alias of `--json` |
| `--retrieve-embedded-execution` | Fetch embedded resources at run time |
| `--embedded-resource-types-execution` | Types to retrieve, repeatable |
| `--embedded-resource-blacklist-execution` | URL substrings to skip, repeatable |

`--vus`, `--duration`, and `--rps` are **rejected** when the scenario uses `load.type: stages`.

`run` validates the YAML and local data-source references before load starts.

Keep secrets out of committed YAML and command literals. Use an untracked vars file or environment-derived values. Avoid `--json-all` for shared reports.

## github / gitlab

```bash
relampo github setup --repo owner/perf-tests
relampo github run /path/test.yaml --repo owner/perf-tests --nodes 2 --vus 100 --duration 10m
relampo gitlab setup --repo group/perf-tests
relampo gitlab run /path/test.yaml --repo group/perf-tests --nodes 2 --vus 100 --duration 10m --html report.html
relampo gitlab run /path/test.yaml --repo group/perf-tests --host gitlab.example.com --runner-tags load-test --nodes 2 --vus 100 --duration 10m
```

The removed `distributed` command is not an alias. Use `git` and authenticated `gh` for GitHub, or `git` and authenticated `glab` for GitLab.

Provider commands include `setup`, `run`, `push`, and `files upload|update|remove`. Obtain explicit approval: these commands can commit and push files.

| Flag | Meaning on provider `run` |
| --- | --- |
| `--nodes` | Worker count, default 2 |
| `--vus` | Total VUs across workers |
| `--duration` | YAML duration override, subject to license limits |
| `--repo` | GitHub repository or GitLab project |
| `--trigger=false` | Publish/prepare without starting execution |
| `--watch=false` | Start without waiting; report export flags require `--watch=true` |
| `--json`, `--html`, `--csv`, `--text` | Generate remote reports; **filename only**, not local paths |
| `--host` | GitLab host; falls back to `GITLAB_HOST` or `gitlab.com` |
| `--runner-tags` | GitLab worker runner tags |

Reports require manual download: GitHub publication links or GitLab job artifacts. Active CSV/TXT bindings are packaged automatically. See [distributed.md](distributed.md).

## Other

```bash
relampo language es
relampo update
relampo uninstall
```

## Typical end-to-end

```bash
relampo record --port 8888 --responses --output login-flow.yaml
relampo correlate login-flow.yaml --apply -o login-flow.correlated.yaml
relampo validate login-flow.correlated.yaml
relampo run login-flow.correlated.yaml --vus 50 --duration 10m --html report.html
```
