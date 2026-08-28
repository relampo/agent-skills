# Correlation

Correlation finds values produced by earlier responses — tokens, session ids, CSRF tokens, dynamic ids, hidden form fields — and reuses them in later requests through variables and extractors.

## When to correlate

- After `relampo record` or converter output that includes a multi-step authenticated flow.
- Whenever a later request hardcodes a value that really changes per session.

Recording must include responses, or there is nothing to match against:

```bash
relampo record --port 8888 --responses --output recording.yaml
```

## Workflow

1. **Analyze**, without rewriting anything:
```bash
relampo correlate recording.yaml
```
2. **Adjust confidence.** The default is `0.25`:
```bash
relampo correlate recording.yaml --min-confidence 0.9   # stricter
relampo correlate recording.yaml --min-confidence 0.0   # broader
```
3. **Apply**:
```bash
relampo correlate recording.yaml --apply -o recording.correlated.yaml
```
4. **Apply selected candidates only**:
```bash
relampo correlate recording.yaml --apply-ids "1,2,6-8" -o recording.correlated.yaml
```
5. **Check**:
```bash
relampo validate recording.correlated.yaml
```

Keep the raw recording. Always write correlated output to a new file with `-o`.

## Manual extractors

Use these when automatic correlation misses a value or picks the wrong source.

JSON token:

```yaml
extractors:
  - type: jsonpath
    from: body
    var: access_token
    path: $.data.access_token
    default: ""
```

Reuse it:

```yaml
headers:
  Token: "{{access_token}}"
```

Regex from an HTML body:

```yaml
extractors:
  - type: regex
    from: body
    var: main_page
    pattern: (?is)<input[^>]*name=["']main_page["'][^>]*value=["']([^"']+)["']
    group: 1
    default: ""
```

Response header:

```yaml
extractors:
  - type: header
    var: session_id
    header: "X-Session-Id"
```

Boundary:

```yaml
extractors:
  - type: boundary
    from: body
    var: csrf
    left_boundary: 'name="csrf" value="'
    right_boundary: '"'
```

Short form:

```yaml
extract:
  token: "jsonpath($.token)"
  req_id: "header.X-Request-Id"
```

Use `path` in new scripts. `expression` is an accepted alias kept for editor compatibility.

## Fail closed

An extractor that matches nothing leaves the variable empty, and an empty token still sends a request. Assert on every authenticated step so the failure is visible:

```yaml
assertions:
  - { type: status, value: 200 }
```

## Tips

- Cookies may already be handled by the scenario cookie policy. Still extract tokens that travel in headers or bodies.
- `--json` writes the analysis to a file, which helps when reviewing a large recording.
- Correlate before you scale. A broken token turns a load test into a 401 benchmark.
