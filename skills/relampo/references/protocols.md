# Non-HTTP protocols

Relampo runs WebSocket, gRPC, WebRTC, and SQL steps beside HTTP steps. Variable support and checks depend on the protocol; SQL does not expose the HTTP extractor/assertion lists.

Baseline: CLI **v2.10.3**, compatibility contract **1.0.10**.

Check any script that mixes protocols before you run it:

```bash
relampo validate mixed-protocol.yaml
```

## WebSocket

A `websocket` step opens one connection and runs a list of actions on it.

| Field | Notes |
| --- | --- |
| `url` | **Required.** Absolute `ws://` or `wss://` |
| `actions` | **Required.** List of `send`, `receive`, or `close` |
| `name` | Step label in reports |
| `headers` | Handshake headers |
| `subprotocols`, `expected_subprotocol` | Negotiation |
| `timeout` | Whole-step timeout |
| `max_message_size` | Bytes |
| `expected_close_codes` | List of accepted close codes |
| `cookie_override` | `inherit`, `enabled`, `disabled` |
| `enabled`, `on_error`, `error_policy` | Same meaning as HTTP steps |

Each action carries exactly one of `send`, `receive`, or `close`.

- **`send`**: `type` (**required**, `text` or `binary`), then `data` or `data_base64`, plus `timeout`, `delay_before`.
- **`receive`**: `type` (optional), `timeout`, `extractors`, `assertions`.
- **`close`**: `code`, `reason`, `timeout`, `delay_before`.

```yaml
- websocket:
    name: "Notifications"
    url: wss://api.example.com/events
    headers:
      Authorization: "Bearer {{access_token}}"
    timeout: 30s
    actions:
      - send:
          type: text
          data: '{"action":"subscribe","channel":"orders"}'
      - receive:
          type: text
          assertions:
            - type: body_contains
              value: subscribed
          extractors:
            - type: jsonpath
              var: subscription_id
              path: "$.id"
      - close:
          code: 1000
```

Extract and assert on `receive`, never on `send`. A `send` has no response to inspect.

## gRPC

Unary calls only. Set shared connection settings once in `grpc_defaults`.

```yaml
grpc_defaults:
  target: api.example.com:443
  protoset: ./api.protoset
  connection_scope: per_vu     # per_vu (default) | run
```

| Field | Notes |
| --- | --- |
| `method` | **Required.** Full path, `/package.Service/Method` |
| `target` | Falls back to `grpc_defaults.target` |
| `protoset` | Compiled descriptor set |
| `reflection` | Use server reflection instead of a protoset |
| `metadata` | Map, the gRPC equivalent of headers |
| `timeout` | Default `30s` |
| `message` | Protobuf JSON. Supports `{{var}}` |
| `payload_base64` | Raw bytes. **No interpolation.** Cannot combine with `message` |
| `expected_status` | For example `OK`, `NOT_FOUND` |
| `extractors`, `assertions`, `enabled`, `on_error`, `error_policy` | As with HTTP |

```yaml
- grpc:
    name: "Create order"
    method: /orders.v1.Orders/Create
    metadata:
      authorization: "Bearer {{access_token}}"
    message:
      customerId: "{{customer_id}}"
      total: 4200
    expected_status: OK
    extractors:
      - type: jsonpath
        var: order_id
        path: "$.id"
```

Use `message` whenever you need variables in the payload. `payload_base64` is fixed bytes and cannot be correlated.

Record gRPC traffic with:

```bash
relampo record grpc --upstream api.example.com:443 --protoset ./api.protoset --output grpc-flow.yaml
```

## WebRTC

The most complex step type. A `webrtc` container holds `data_channels` and `media_tracks`, then a `steps` list of actions.

Exactly one direct `webrtc_offer` must precede one direct `webrtc_connect`. HTTP signaling steps can run between them. Nested WebRTC containers and `parallel`, `one_time`, or `balanced` inside WebRTC are unsupported.

| Container field | Notes |
| --- | --- |
| `timeout` | Default `30s` |
| `ice_gathering_timeout` | Default `7s`. Must be above zero and at most `timeout` |
| `max_message_size` | Default 16 KiB, hard limit 1 MiB |
| `ice_transport_policy` | `all` or `relay` |
| `ice_servers` | At most 16 |
| `data_channels` | At most 32. `label` required, plus `ordered`, `protocol`, and one of `max_retransmits` or `max_packet_lifetime_ms` |
| `media_tracks` | At most 8. `id`, `kind` (`audio`/`video`), `direction` (`sendonly`/`recvonly`/`sendrecv`), `codec`, `source` (required unless `recvonly`), `loop`, `sample_duration` |

| Action | Fields |
| --- | --- |
| `webrtc_offer` | `export` **required**, and the name must start with `__relampo_` |
| `webrtc_connect` | `answer` **required**, plus `timeout` |
| `webrtc_send` | `channel` **required**, plus `data` or `data_base64` |
| `webrtc_receive` | `channel` **required**, plus `type` (`text`/`binary`/`any`), `timeout`, `extractors`, `assertions` |
| `webrtc_wait_state` | `state` (`connected` or `data_channel_open`); `channel` required for `data_channel_open` |

```yaml
- webrtc:
    name: "Join room"
    timeout: 30s
    data_channels:
      - label: chat
        ordered: true
    steps:
      - webrtc_offer:
          export: __relampo_webrtc_offer
      - request:
          name: "Exchange SDP"
          method: POST
          url: https://api.example.com/signaling
          body:
            sdp: "{{__relampo_webrtc_offer}}"
          extractors:
            - type: jsonpath
              var: __relampo_webrtc_answer
              path: $.sdp
          assertions:
            - type: status
              value: 200
      - webrtc_connect:
          answer: "{{__relampo_webrtc_answer}}"
      - webrtc_wait_state:
          state: data_channel_open
          channel: chat
      - webrtc_send:
          channel: chat
          data: '{"type":"ping"}'
      - webrtc_receive:
          channel: chat
          type: text
```

The signalling exchange runs through your own HTTP steps. Export the offer, POST it to your signalling endpoint with a normal request, extract the answer into a `__relampo_`-prefixed variable, then pass it to `webrtc_connect`.

The `__relampo_` prefix matters: names with two leading underscores are always variables and never parsed as built-in functions.

### Encoded audio/video

Media sources must already be encoded:

- Audio: Opus in Ogg.
- Video: VP8/VP9 in IVF, or H.264 Annex-B.
- Sending tracks require `source`; `recvonly` forbids it.
- Source paths resolve from the YAML directory. Sending starts after connection.
- `loop` defaults to false. `sample_duration` is an optional positive timing override.
- Limits: 8 tracks, 256 MiB per regular source file, and 4 MiB per encoded sample.

Receive tracks drain and count RTP packets for quality metrics. They do not decode/play media or support media-content assertions. There is no transcoding, browser media capture, or WebRTC recording. Offerer-side signaling remains your YAML's responsibility.

## SQL

Use SQL steps for setup, teardown, and verification. They do not measure your application's HTTP performance.

| Field | Notes |
| --- | --- |
| `dialect` | **Required.** `postgres` (also `postgresql`) or `mysql` |
| `kind` | `query` (also `select`, `read`) or `exec` (also `write`). Defaults to `query` |
| `connection` | **Required.** See below |
| `query` | **Required** |
| `params` | Positional values. Use these, never string concatenation |
| `allow_writes` | **Must be `true`** when `kind: exec` |
| `validate_connectivity` | Check the connection during validation |
| `timeout`, `on_error`, `error_policy` | As with HTTP |

`connection` requires `host`, `database`, `user`, and `password`. Optional: `port`, `ssl_mode`, `max_open_conns`, `max_idle_conns`, `conn_max_lifetime`.

```yaml
- sql:
    name: "Look up user"
    dialect: postgres
    kind: query
    connection:
      host: db.internal
      database: relampo
      user: "{{db_user}}"
      password: "{{db_password}}"
      ssl_mode: require
    query: "SELECT id, email FROM users WHERE email = $1"
    params:
      - "{{email}}"
```

Postgres uses `$1`, `$2`. MySQL uses `?`. Keep credentials in variables supplied from the environment or an untracked vars file. Keep values out of command literals and committed files.

Writes need both settings:

```yaml
- sql:
    name: "Reset fixture"
    dialect: postgres
    kind: exec
    allow_writes: true
    connection: { host: db.internal, database: relampo, user: "{{db_user}}", password: "{{db_password}}" }
    query: "DELETE FROM orders WHERE customer_id = $1"
    params: ["{{customer_id}}"]
```

`one_time` runs once **per VU**. It does not make a database write run-wide. Use idempotent per-user setup there; prepare global fixtures separately before load. SQL parameters prevent value concatenation, but they do not make writes safe to repeat.
