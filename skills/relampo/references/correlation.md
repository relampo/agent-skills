# Correlation

Correlation captures values produced by earlier responses (tokens, session ids, CSRF, dynamic ids, hidden form fields) and reuses them in later requests via variables / extractors.

## When to correlate

- After `relampo record` or converter output that includes multi-step authenticated flows.
- Whenever a later request hardcodes a value that actually changes per session.

## Workflow

1. **Analyze** (no rewrite):
```bash
relampo correlate recording.yaml
```
2. **Compare confidence**:
```bash
relampo correlate recording.yaml --min-confidence 0.0   # broader
relampo correlate recording.yaml --min-confidence 0.9   # stricter
```
3. **Apply all selected candidates**:
```bash
relampo correlate recording.yaml --apply -o recording.correlated.yaml
```
4. **Apply only some groups**:
```bash
relampo correlate recording.yaml --apply-ids "1,2,6-8" -o recording.correlated.yaml
```
5. **Diagnose before load**:
```bash
relampo debug recording.correlated.yaml
```

## Manual extractors (when auto-apply is wrong)

JSON token:

```yaml
extractors:
  - type: jsonpath
    from: body
    var: access_token
    expression: $.data.access_token
    default: ""
```

Reuse:

```yaml
headers:
  Token: "{{access_token}}"
```

Regex from HTML/body:

```yaml
extractors:
  - type: regex
    from: body
    var: main_page
    pattern: (?is)<input[^>]*name=["']main_page["'][^>]*value=["']([^"']+)["']
    group: "1"
    default: ""
```

Header:

```yaml
extractors:
  - type: header
    var: request_id
    # header name per product extractor fields
```

Short form:

```yaml
extract:
  token: "jsonpath($.token)"
  req_id: "header.X-Request-Id"
```

## Tips

- Prefer writing a new correlated file (`-o`) instead of overwriting the raw recording until review is done.
- After apply, assert status codes on authenticated steps so empty tokens fail fast.
- Cookies may be handled by scenario cookie policy; still extract explicit tokens used in headers/bodies.
