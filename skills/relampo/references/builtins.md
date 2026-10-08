# Built-in functions

Relampo has about 180 built-in functions. Use them to generate test data, transform values, and build headers without external scripts.

Baseline: CLI **v2.10.3**, compatibility contract **1.0.10**.

## Contents

- [Syntax](#syntax), [interpolation rules](#six-rules-for-reliable-interpolation), [arguments](#arguments), [errors](#errors)
- [General](#general), [text](#text), [math](#math), [encoding](#encoding), [crypto](#crypto)
- [Data and lists](#data-json-and-lists), [random and sequences](#random-and-sequences)
- [Calendar](#calendar), [date formats](#date-formats), [HTTP helpers](#http-helpers)

## Syntax

Canonical form. Use this in new scripts:

```
{{_functionName(arg1,arg2)}}
```

Zero-argument functions can drop the parentheses: `{{_uuid}}` works like `{{_uuid()}}`.

| Form | Meaning |
| --- | --- |
| `{{_name(args)}}` | Built-in call. **Use this.** |
| `{{name}}` | Variable |
| `${name}` | Variable only. A built-in name here is **rejected**. |
| `{{$name}}` | Legacy alias. Read it in old scripts; do not write it. |

An explicitly defined variable always wins. If you define a variable named `$uuid`, then `{{$uuid}}` returns your value, not a generated UUID.

Names starting with two underscores (`__name`) are always variables, never built-ins. The runtime uses this prefix for its own values.

## Six rules for reliable interpolation

Use these rules when generating data or reusing values.

**1. Invalid built-ins block HTTP preparation.** Failed built-ins prevent that HTTP request from being sent. Validation checks names, syntax, argument count, and constant argument types. Runtime checks cover dynamic values and ranges. Debug can still show successful sibling expressions beside a failed expression.

**2. Bare words are variables, not text.** `{{_upper(name)}}` reads a variable called `name`. `{{_upper("name")}}` uppercases the literal text. If you meant text and forgot the quotes, you get an empty string.

**3. Direct calls evaluate per occurrence.** Two direct `{{_uuid()}}` calls generate different UUIDs. For a shared request value, define `variables: {request_id: "{{_uuid()}}"}` and reuse `{{request_id}}`. Declared variable results are cached within that request, not across the full flow.

**4. An unknown `_name` is a hard error. An unknown `$name` is not.** `{{_notReal()}}` fails loudly in strict mode. `{{$notReal}}` is treated as a missing variable and becomes empty.

**5. Built-ins need `{{ }}`.** `${_uuid()}` is rejected. The `${...}` form is reserved for variables.

**6. Date formats use Java tokens**, not Go layouts and not strftime. See [Date formats](#date-formats).

## Arguments

**Quoting.** Single and double quotes both work. Escapes follow Go rules, so `\"`, `\\`, and `\n` are valid. A literal comma inside a string must be escaped as `\,` or it splits the arguments.

```yaml
url: "/search?q={{_urlEncode(\"tea, black\")}}"     # comma inside quotes still splits
url: "/search?q={{_urlEncode(\"tea\\, black\")}}"   # escaped comma is safe
```

**Bare literals.** Numbers (`7`, `2.6`), `true`, `false`, and `null` need no quotes.

**Nesting works.** A variable or another built-in can be an argument:

```yaml
body:
  city: "{{_lower({{city_name}})}}"
  total: "{{_add({{qty}},5)}}"
  key: "{{_sha256({{_concat(\"user-\",{{user_id}})}})}}"
```

When the whole argument is a single `{{...}}`, its native type is preserved. A number stays a number instead of becoming text.

## Errors

| Code | Cause |
| --- | --- |
| `unknown_function` | `_name` is not a registered built-in |
| `invalid_arguments` | Wrong number of arguments |
| `invalid_argument_type` | Wrong type, such as text where a number is required |
| `invalid_range` | For example `_randomInt(8,2)` or `_substring("abc",4,1)` |
| `invalid_syntax` | Unclosed parentheses, quotes, or braces; a built-in inside `${...}` |
| `division_by_zero`, `missing_path`, `no_match`, `invalid_timezone`, `missing_variable` | Function-specific |

Error messages report the function name and argument position. They never include argument values, so secrets stay out of logs.

---

## General

| Function | Arguments | Returns | Legacy |
| --- | --- | --- | --- |
| `_uuid()` | — | UUID v4 | `{{$uuid}}` |
| `_guid()` | — | Same as `_uuid` | |
| `_random(min?,max?)` | Both optional, both together | Float in `[0,1)`, or `[min,max)` | `{{$random}}` |
| `_randomInt(min,max)` | Integers, inclusive | Integer | `{{$random_int(a,b)}}` |
| `_randomBoolean()` | — | `true` or `false` | |
| `_lower(text)` | | Lowercase text | |
| `_add(a,b)` | Numbers | Integer if both are integers | |
| `_time(format?,varName?)` | | Unix milliseconds if no format; formatted text otherwise | |
| `_now(format?,varName?)` | | ISO 8601 with offset by default | |

`varName` stores the result in a hidden variable so later steps reuse the same instant.

Legacy-only aliases: `{{$timestamp}}` (seconds), `{{$timestamp_ms}}` and `{{$date.now}}` (milliseconds), `{{$random_string(n)}}`, `{{$count(x)}}`.

## Text

| Function | Arguments |
| --- | --- |
| `_upper(text)` | |
| `_capitalize(text)` | First letter only |
| `_titleCase(text)` | Every word |
| `_trim(text)`, `_ltrim(text)`, `_rtrim(text)` | |
| `_replace(text,search,replacement)` | First match only. Fails if `search` is empty |
| `_replaceAll(text,search,replacement)` | Every match |
| `_substring(text,start,length?)` | Counts characters, not bytes |
| `_left(text,n)`, `_right(text,n)` | |
| `_length(value)` | Text or list |
| `_concat(...)` | Any number of values |
| `_padLeft(text,length,pad?)`, `_padRight(text,length,pad?)` | Pad defaults to a space |
| `_slug(text)` | Lowercase, accents removed, hyphenated |
| `_split(text,separator,index?)` | List, or one element when `index` is given |
| `_regexExtract(text,pattern,group?)` | Fails with `no_match` if nothing matches |
| `_regexReplace(text,pattern,replacement)` | Use `$1` for capture groups |
| `_contains(text,sub)`, `_startsWith(text,prefix)`, `_endsWith(text,suffix)` | Boolean |

```yaml
{{_capitalize("hello WORLD")}}            # Hello WORLD
{{_titleCase("hello WORLD")}}             # Hello World
{{_substring("héllo",1,3)}}               # éll
{{_slug("Héllo, World!")}}                # hello-world
{{_split("a,b,c",",",1)}}                 # b
{{_regexExtract("id=42","id=(\\d+)")}}    # 42
{{_regexReplace("a1 b2","(\\d)","[$1]")}} # a[1] b[2]
```

## Math

| Function | Notes |
| --- | --- |
| `_subtract(a,b)`, `_multiply(a,b)` | Integer if both are integers |
| `_divide(a,b)` | Always a float. Fails on zero |
| `_mod(a,b)` | Fails on zero |
| `_round(value,decimals?)` | 0 to 18 decimals, default 0 |
| `_floor(value)`, `_ceil(value)` | Integer |
| `_abs(value)` | |
| `_min(...)`, `_max(...)`, `_avg(...)`, `_sum(...)` | Several values, or one list |
| `_clamp(value,min,max)` | |

```yaml
{{_divide(5,2)}}      # 2.5
{{_round(2.56,1)}}    # 2.6
{{_clamp(12,0,10)}}   # 10
{{_avg(2,4)}}         # 3
```

## Encoding

| Function | Aliases |
| --- | --- |
| `_urlEncode(text)`, `_urlDecode(text)` | |
| `_base64Encode(text)`, `_base64Decode(text)` | |
| `_htmlEncode(text)`, `_htmlDecode(text)` | |
| `_xmlEncode(text)`, `_xmlDecode(text)` | `_xmlEscape`, `_xmlUnescape` |
| `_jsonEncode(value)` | `_jsonEscape`, `_jsonStringify` |
| `_jsonDecode(text)` | `_jsonUnescape`, `_jsonParse` |

```yaml
{{_urlEncode("a b+c")}}      # a+b%2Bc
{{_base64Encode("hello")}}   # aGVsbG8=
```

## Crypto

All return lowercase hex.

| Function | Arguments |
| --- | --- |
| `_md5(value)`, `_sha1(value)`, `_sha256(value)`, `_sha512(value)` | |
| `_hmacMd5(value,key)`, `_hmacSha1(value,key)`, `_hmacSha256(value,key)`, `_hmacSha512(value,key)` | |

Uppercase spellings also work: `_hmacMD5`, `_hmacSHA1`, `_hmacSHA256`, `_hmacSHA512`. Error messages report the lower-camel name.

These reject maps and lists. Convert structured values with `_jsonEncode` first:

```yaml
signature: "{{_hmacSha256({{_jsonEncode({{payload}})}},{{secret}})}}"
```

## Data, JSON, and lists

| Function | Arguments |
| --- | --- |
| `_jsonGet(json,path)` | Dotted path. Alias `_jsonPath`. Fails if the path is missing |
| `_jsonSet(json,path,value)` | Returns updated JSON |
| `_count(collection)` | List, map, or text |
| `_join(list,separator)` | |
| `_listGet(list,index)` | Fails if out of range |
| `_listRandom(list)` | Fails on an empty list |
| `_listContains(list,value)` | Boolean |
| `_get(varName)` | Variable value, or nothing if unset |
| `_default(value,fallback)` | Fallback when value is empty or missing |
| `_coalesce(...)` | First non-empty value |

```yaml
{{_jsonGet(\"{\\\"user\\\":{\\\"name\\\":\\\"Ada\\\"}}\",\"user.name\")}}   # Ada
{{_default({{maybe_empty}},"fallback")}}
{{_coalesce({{a}},{{b}},"last resort")}}
```

Note the path style is dotted (`user.name`, `items.1`), not JSONPath `$.user.name`. JSONPath belongs in extractors and assertions, not here.

## Random and sequences

| Function | Arguments |
| --- | --- |
| `_randomFloat(min,max,decimals?)` | 0 to 18 decimals |
| `_randomString(length,charset?)` | Alphanumeric by default |
| `_randomAlpha(length)`, `_randomNumeric(length)`, `_randomAlphanumeric(length)`, `_randomHex(length)` | |
| `_randomFrom(...)` | Picks one value. Fails if any value is missing |
| `_randomFromMultipleVars(...)` | Variable **names**. Picks one variable's value |
| `_counter(start?,increment?,name?)` | Defaults 1 and 1. Scoped per VU |
| `_sequence(start?,increment?,name?)` | Same shape, separate namespace |
| `_char(codePoint)` | One character |

`_counter` and `_sequence` advance on each call, including calls through declared variables. Their state is per VU; identical counters across VUs do not guarantee globally unique values. Prefer UUIDs for independent account names across VUs.

```yaml
body:
  email: "user-{{_uuid()}}@example.com"
  sequence: "{{_counter(1,1,\"signup\")}}"
  ref: "{{_randomHex(16)}}"
  plan: "{{_randomFrom(\"free\",\"pro\",\"team\")}}"
```

## Calendar

Default format when omitted: `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`.

| Function | Arguments |
| --- | --- |
| `_timestamp(unit?)`, `_epoch(unit?)` | `ms` (default) or `s` |
| `_dateTimeConvert(value,inputFormat,outputFormat)` | |
| `_timeShift(value,inputFormat,shift,outputFormat?)` | Shift like `+1d`, `-3h`. Units `ms,s,m,h,d,w,mo,y` |
| `_addTime(...)`, `_subtractTime(...)` | Two forms, see below |
| `_randomDate(from,to,format)` | One format parses both bounds |
| `_startOf(...)`, `_endOf(...)` | Two forms, see below. Units `second,minute,hour,day,week,month,quarter,year` |
| `_startOfQuarter(value,inputFormat,outputFormat?)`, `_endOfQuarter(...)` | |
| `_dayOfWeek`, `_dayOfMonth`, `_dayOfYear`, `_weekOfYear`, `_month`, `_year`, `_hour`, `_minute`, `_second`, `_millisecond`, `_quarter` | No arguments for now, or `(value,format)` |
| `_timezone()` | Execution timezone name |
| `_toTimezone(...)` | Several forms, see below |
| `_daysBetween(start,end,format?)`, `_hoursBetween`, `_minutesBetween`, `_secondsBetween` | Whole units, can be negative |
| `_durationBetween(start,end,unit?,format?)` | Unit `ms,s,m,h,d`, default `ms` |
| `_isWeekend`, `_isWeekday` | Boolean |
| `_isLeapYear(...)` | No arguments, a year number, or `(value,format)` |

`_dayOfWeek` is ISO: Monday is 1, Sunday is 7.

**Functions with two argument forms.** These pick a form by inspecting the arguments, so order matters:

```yaml
# _addTime / _subtractTime
{{_addTime("2024-02-28 23:30:15","yyyy-MM-dd HH:mm:ss","+1mo","yyyy-MM-dd HH:mm:ss")}}
{{_addTime("2024-02-28 23:30:15",1,"day","yyyy-MM-dd HH:mm:ss")}}

# _startOf / _endOf — unit first, or value first
{{_startOf("day","2024-02-28 23:30:15","yyyy-MM-dd HH:mm:ss")}}
{{_startOf("2024-02-28 23:30:15","yyyy-MM-dd HH:mm:ss","week","yyyy-MM-dd HH:mm:ss")}}

# _toTimezone — format first, or source timezone first
{{_toTimezone("2024-02-28 23:30:15","yyyy-MM-dd HH:mm:ss","UTC","yyyy-MM-dd HH:mm:ssXXX")}}
{{_toTimezone("2024-02-28 23:30:15","UTC","America/New_York","yyyy-MM-dd HH:mm:ss")}}
```

### Date formats

Tokens follow Java `SimpleDateFormat`.

| Token | Meaning | Token | Meaning |
| --- | --- | --- | --- |
| `yyyy` / `yy` | Year, 4 or 2 digits | `mm` / `m` | Minute |
| `MMMM` / `MMM` | Month name, full or short | `ss` / `s` | Second |
| `MM` / `M` | Month number | `SSS` | Milliseconds |
| `dd` / `d` | Day of month | `a` | AM or PM |
| `HH` / `H` | Hour, 24-hour | `XXX` / `XX` / `X` | ISO offset |
| `hh` / `h` | Hour, 12-hour | `Z` | RFC 822 offset |

Text inside single quotes passes through unchanged: `'date='yyyy-MM-dd` produces `date=2026-08-26`.

**`E` (day name) is not supported.** It fails with an unsupported-token error.

```yaml
{{_time("yyyyMMdd")}}             # 20260826
{{_time("yyyy-MM-dd HH:mm:ss")}}  # 2026-08-26 14:30:15
```

## HTTP helpers

| Function | Returns |
| --- | --- |
| `_queryParam(url,name)` | First value of that parameter, or empty |
| `_bearer(token)` | `Bearer <token>`. Fails on an empty token |
| `_basicAuth(user,password)` | `Basic <base64>`. Fails if either is empty |

```yaml
headers:
  Authorization: "{{_bearer({{access_token}})}}"
```

Prefer the [`auth` block](yaml-spec.md#auth) over building headers by hand. Use these helpers when a header must be assembled in an unusual place.
