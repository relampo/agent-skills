# Relampo CLI commands

Help:

```bash
relampo help
relampo help record
relampo record --help
```

## setup / version / cert

```bash
relampo setup
relampo version
relampo cert status
```

`setup` prepares the environment and CA material for HTTPS recording.

## record

Capture HTTP(S) traffic through a local proxy and write YAML.

```bash
relampo record --port 8888 --responses --output my-test.yaml
```

- Point the client/app at the proxy, exercise the flow, stop with Ctrl+C.
- Default recording uses JMeter-style redirect replay: `http_defaults.follow_redirects` enabled; immediate duplicate redirect targets may be written as `enabled: false`.
- Use `--redirect-mode legacy` only when you need the previous recording shape.

## converter

```bash
relampo converter
```

Opens the browser converter UI to turn collections/source assets into Relampo YAML.

## correlate

```bash
relampo correlate my-test.yaml
relampo correlate my-test.yaml --apply -o my-test.correlated.yaml
relampo correlate my-test.yaml --apply-ids "1-3,5" -o selected.yaml
relampo correlate my-test.yaml --min-confidence 0.0
relampo correlate my-test.yaml --min-confidence 0.9
```

Next step after apply: validate/debug, then run.

## editor / debug / studio

```bash
relampo editor
relampo debug my-test.correlated.yaml
```

- `debug` is an alias of `studio`: opens the visual Debug view with the script mounted and runs **1 VU / 1 iteration** for diagnosis, not load.

## run

```bash
relampo run my-test.correlated.yaml
relampo run my-test.yaml --vus 50 --duration 10m
relampo run my-test.yaml --var base_url=https://staging.example.com
relampo run my-test.yaml --vars-file env/staging.yaml
relampo run my-test.yaml --html report.html --json report.json --csv report.csv --text report.txt
```

`run` validates YAML (and local data source references) before starting load. Use `relampo debug` for a single-pass functional check.

## distributed

```bash
relampo distributed setup --repo owner/perf-tests
relampo distributed files upload --repo owner/perf-tests --files "[/path/test.yaml,/path/users.csv]"
relampo distributed run /path/test.yaml --repo owner/perf-tests --nodes 2 --vus 10 --duration 2m
```

Requires `git`, `gh`, and authenticated GitHub CLI. See `distributed.md` for limits.

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
relampo debug login-flow.correlated.yaml
relampo run login-flow.correlated.yaml --vus 50 --duration 10m --html report.html
```
