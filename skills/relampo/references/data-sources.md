# Data sources

A `data_source` feeds each virtual user different values from a CSV or TXT file. Use it for logins, search terms, product ids, and any test that must not reuse one record.

Targets YAML contract **1.0.5**.

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
| `mode` | `per_vu` | How records are divided |
| `strategy` | `sequential` | Reading order |
| `on_exhausted` | `recycle` | What happens at end of file |
| `variable_names` | | Positional names for a file with no header |
| `bind` | | Column-to-variable map for a file with a header |

## mode

- **`per_vu`** — each VU gets its own slice of the file. Two VUs never read the same record at the same time. This is the usual choice.
- **`shared`** — all VUs draw from one pool.
- **`per_worker`** — each distributed worker gets a slice. Use it with `relampo distributed run`.

`per_worker` is rejected during a Studio debug run. Switch to `per_vu` while you debug in the editor, then switch back.

## strategy

- **`sequential`** — in file order. Repeatable, and the easiest to reason about.
- **`random`** — random record each time. Records can repeat.
- **`unique`** — no record is used twice.

`unique` runs out. Pair it with `on_exhausted: stop` when reuse would corrupt your results, such as one-time signup codes.

## on_exhausted

- **`recycle`** — start again at the first record. Fine for logins that can repeat.
- **`stop`** — end the test. Use it when every record must be used only once.

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
ada@example.com,secret1
grace@example.com,secret2
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

The path is resolved by the process running the test, not by the machine that wrote the YAML.

- Prefer absolute paths when the working directory is uncertain.
- `relampo run` checks that referenced files exist before load starts.
- For distributed runs, upload the file first. A missing file fails on the workers, not on your machine:

```bash
relampo distributed files upload --repo owner/perf-tests --files "[/path/test.yaml,/path/users.csv]"
```

## Checks

Missing data is silent. An unset variable becomes an empty string, so a login step can post an empty password and still return a valid 401 that your test happily records.

Always assert:

```yaml
assertions:
  - type: status
    value: 200
```

See also [gotchas.md](gotchas.md).
