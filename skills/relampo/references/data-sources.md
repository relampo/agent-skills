# Data sources

A `data_source` binds CSV/TXT fields to variables. Use it for logins, search terms, and product IDs. A dataset mode does not guarantee different records across VUs or workers.

Baseline: CLI **v2.10.3**, compatibility contract **1.0.10**.

## Shape

```yaml
data_source:
  type: csv                          # csv | txt
  file: "/srv/relampo/users.csv"     # `path` is accepted as an alias
  mode: per_vu                       # per_vu | shared | per_worker
  strategy: sequential               # sequential | random | unique
  on_exhausted: recycle              # recycle | stop
  variable_names: "email,password"
```

| Field | Default | Notes |
| --- | --- | --- |
| `type` | | `csv` or `txt` |
| `file` | | Path to the file. `path` is an alias |
| `mode` | `per_vu` | Cursor ownership or fixed mapping |
| `strategy` | `sequential` | Reading order |
| `on_exhausted` | `recycle` | What happens at end of file |
| `variable_names` | | Positional names for a file with no header |
| `bind` | | Column-to-variable map for a file with a header |

## mode

- **`per_vu`** — each VU has an independent cursor starting at the first row. Several VUs can read the same record.
- **`shared`** — VUs share a cursor for that data-source node within one process. Distributed workers have separate processes and can reuse records.
- **`per_worker`** — fixed row mapping by VU index: `(__vu_idx - 1) % row_count`. It is not a worker-owned slice.

`per_worker` is rejected during a Studio debug run. Switch to `per_vu` while you debug in the editor, then switch back.

## strategy

- **`sequential`** — in file order. Repeatable, and the easiest to reason about.
- **`random`** — random record each time. Records can repeat; exhaustion does not advance as with sequential reading.
- **`unique`** — shuffled rows without repetition within that cursor's cycle. It does not guarantee uniqueness across VUs, nodes, or recycled cycles.

`unique` runs out. Pair it with `on_exhausted: stop` when reuse would corrupt your results, such as one-time signup codes.

## on_exhausted

- **`recycle`** — start again at the first record. Fine for logins that can repeat.
- **`stop`** — stop the current VU iteration. It does not stop the full test.

A long run with `recycle` will reuse records. If your application caches per user, that quietly inflates your results.

## Naming columns

**No header row.** Name the columns in order:

```yaml
data_source:
  type: csv
  file: "users.csv"
  variable_names: "email,password"
```

```
ada@example.com,synthetic-password-1
grace@example.com,synthetic-password-2
```

**With a header row.** Map column names to variable names:

```yaml
data_source:
  type: csv
  file: "users.csv"
  bind:
    user_email: email
    user_password: password
```

Either way, use the values as ordinary variables:

```yaml
- request:
    name: "Login"
    method: POST
    url: /auth/login
    body:
      email: "{{email}}"
      password: "{{password}}"
    assertions:
      - type: status
        value: 200
```

## Paths

Relative paths resolve from the working directory first, then the scenario directory. A same-named file in the working directory takes priority.

- Keep portable datasets beside the YAML. Use absolute paths only for local-only files when needed.
- `relampo run` checks that referenced files exist before load starts.
- `github` and `gitlab` preparation package active root, step, and request datasets into `scenarios/data`, then rewrite their YAML paths. Missing files fail before replacing the previous scenario.
- Packaging commits datasets even if the execution repository ignores CSV/TXT files. Get publication approval and use synthetic data. Manual upload is not required for active CSV/TXT bindings.

## Binding scope and order

Each data-source node has its own cursor, even when two nodes read the same file.

- Root `data_source`: bind once per VU iteration before initialization.
- A standalone `data_source` step: bind when traversal reaches that step.
- `request.data_source`: bind before that HTTP request's Spark hooks and interpolation.

```yaml
steps:
  - data_source:
      type: csv
      file: products.csv
      bind: {product: product_id}
  - request:
      method: POST
      url: /quotes
      data_source:
        type: csv
        file: regions.csv
        bind: {region: region_id}
      body:
        product: "{{product_id}}"
        region: "{{region_id}}"
      assertions:
        - {type: status, value: 200}
```

See [scoped-data-sources.yaml](../assets/examples/scoped-data-sources.yaml) and its bundled synthetic files for all three scopes.

`variable_names` treats all rows as data. `bind` consumes the first row as a header. TXT uses the CSV reader too; it is not an arbitrary line-streaming format.

## Checks

Missing files, empty datasets, and missing mapping configuration fail binding. Request-local binding failures stop before network execution. Missing header columns can still yield empty values. Ordinary unset variables can also become empty strings.

Always assert:

```yaml
assertions:
  - type: status
    value: 200
```

See also [gotchas.md](gotchas.md).
