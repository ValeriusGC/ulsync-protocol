# ulsync protocol specification v1

**Created:** 2026-08-26 10:26:24 +0500  
**Updated:** 2026-08-26 10:28:14 +0500  
**Version:** 1  
**Document type:** specification

This document is the wire contract. A server written in Go and a package written in Dart, produced independently, must converge on these files. Divergence is a failing test on a fixture, not a first run on two devices.

v1 describes one envelope, three client endpoints, a live feed, and an operations surface that is not the client protocol. Batches of several envelopes in one push, tombstones, `part` values other than `full`, payload compression, and WebSocket are outside this version.

## 1. Envelope

An envelope is one JSON object. On the wire it is not the storage row: `user_id` is never in the JSON, and `server_seq` appears only in some responses.

### 1.1. Fields on the wire

Each field: JSON type, required on every envelope that carries it, who writes it, what it means.

| Field | Type | Required | Written by | Meaning |
|---|---|---|---|---|
| `id` | string | yes | client | Identity of the record. Stable across devices. |
| `part` | string | yes | client | Which slice of the record this envelope holds. Identity is `(id, part)`. The only value used in this version is `full`. |
| `entity_type` | string | yes | client | Codec key for the receiving client. The server does not validate it and keeps no list of types. |
| `created_at_ms` | number (integer) | yes | client | Time the record was created. |
| `last_edited_at_ms` | number (integer) | yes | client | Time of the last edit. First rank of conflict resolution. At creation equals `created_at_ms`. |
| `revision` | number (integer) | yes | client | Edit counter. Second rank of conflict resolution. Greater wins. |
| `source_id` | string | yes | client | Identifier of the producing installation. Third rank of conflict resolution. The server compares it and does not interpret it. Must be non-empty. |
| `flags` | number (integer) | yes | client | Protocol flags. This version sends `0`. Bit assignments for deletion and merge are outside this version. |
| `schema_version` | number (integer) | yes | client | Version of the payload format on the producing client. This version sends `1`. |
| `payload_encoding` | string | yes | client | Hint to the receiving client about how to decode the payload bytes. This version sends `json`. The server copies the string and does not interpret it. |
| `payload` | string | yes | client | Payload bytes, encoded as base64. |
| `server_seq` | number (integer) | pull only | server | Per-user monotonic cursor. See §1.3. |

`created_at_ms` and `last_edited_at_ms` are milliseconds (thousandths of a second), not microseconds, counted from the Unix epoch 1970-01-01T00:00:00Z in UTC. The `_ms` suffix is the unit. Go `time.Time` and Dart `DateTime` count microseconds internally; converting to this field is integer milliseconds, not a second truncation and not a microsecond value left as-is.

`payload` uses RFC 4648 section 4: the standard alphabet (`A–Z`, `a–z`, `0–9`, `+`, `/`) **with** padding `=`. It does not use base64url (RFC 4648 section 5: the URL- and filename-safe alphabet `-` `_`, usually without padding). Go `encoding/base64.StdEncoding` and Dart `dart:convert` `base64` match section 4; `URLEncoding` and `base64Url` match section 5. Different defaults are a typical "works on my machine" failure.

JSON numbers for the integer fields must be numbers, not numeric strings.

### 1.2. `user_id` is absent

The envelope has no `user_id`. The owner is the `sub` (subject) claim of the access token: a JSON Web Token (JWT) sent as `Authorization: Bearer <token>`. If the client named the owner in the body, it could write into another user's store.

The server takes `sub` after verifying the token against a JSON Web Key Set (JWKS): a published set of **public** keys used to check signatures. The private key stays with whoever issues tokens.

### 1.3. `server_seq` is not on push

`server_seq` is present only in pull responses and in live `envelope` events. A push request must not rely on it. If a client sends it on push, the server ignores it.

The field is omitted from the push response as well. The client's cursor moves only from pull results, never from push results. Returning the number from push would make it possible to set the cursor forward and skip envelopes the client has not seen.

## 2. Conflict resolution

When two envelopes share `(id, part)` for the same user, one wins. Comparison uses three ranks, in order:

1. greater `last_edited_at_ms` wins;
2. if equal, greater `revision` wins;
3. if equal, greater `source_id` wins, compared as UTF-8 bytes (the same order as SQLite `TEXT` with `BINARY` collation).

The outcome does not depend on arrival order. Whichever envelope arrives first, the same one remains. Without the third rank, two receivers given the same set of envelopes can store different states.

If all three ranks are equal, the stored row is not inferior to the incoming one: the upsert (a single insert-or-update statement whose condition decides) does not replace it, and the response is `applied: false`.

`created_at_ms` is never updated after insert.

The per-user sequence is incremented before the upsert decides. A rejected envelope does not receive a new `server_seq` on its row; the consumed number may never appear. That is why gaps in `server_seq` are legal (§6).

## 3. Endpoints

Sync endpoints, except `GET /health`, require `Authorization: Bearer <token>`. The server verifies the signature (JWKS, or a development shared secret when explicitly configured) and reads `sub`. Missing, expired, or badly signed tokens, and a missing or empty `sub`, produce `401`.

Request and response bodies on the JSON endpoints are `Content-Type: application/json`.

### 3.1. `POST /v1/sync/push`

Request:

```json
{"envelopes":[<envelope>, …]}
```

Response:

```json
{"results":[{"id":"<id>","part":"<part>","applied":true}]}
```

Each result names the envelope and whether the upsert stored it. `server_seq` is physically absent from this response: the cursor moves only from pull (§1.3).

This version accepts exactly one envelope in `envelopes`. Zero envelopes is `400`. More than one is `413`.

### 3.2. `GET /v1/sync/pull`

Query:

| Parameter | Default | Meaning |
|---|---|---|
| `since` | `0` | Exclusive lower bound: return rows with `server_seq > since`. |
| `limit` | `100` | Maximum envelopes in this response. Maximum allowed value is `500`. |
| `live` | omitted | Omitted: answer immediately, possibly empty. `sse`: live stream (§4). `poll`: long polling (the server holds the HTTP request until an envelope exists or 55 seconds elapse). |

`since` must be an integer ≥ 0. `limit` must be an integer in `1…500`. Any other `live` value, a non-numeric `since` or `limit`, a negative `since`, or a `limit` outside `1…500` is `400`.

Immediate and long-poll responses:

```json
{"envelopes":[<envelope with server_seq>, …],"next_cursor":<integer>}
```

Envelopes are ordered by `server_seq` ascending. `next_cursor` is the `server_seq` of the last envelope in the page. If the page is empty, `next_cursor` equals the request `since`.

### 3.3. `GET /health`

No token. No secrets in the body.

```json
{"version":"<string>","started_at":"<RFC 3339 UTC>","storage":"<path>"}
```

`version` is the process version. `started_at` is when this process started, UTC, RFC 3339 (for example `2026-08-26T05:26:24Z`); JSON has no datetime type, so the string form is part of the contract. `storage` is the configured filesystem path of the database file, not its contents.

This is a liveness check for process supervisors, not an operations panel.

## 4. Live feed

Server-Sent Events (SSE) is a one-way HTTP stream: the server writes, the client reads. The content type is `text/event-stream`. In a browser the receiver is `EventSource`.

`GET /v1/sync/pull?since=N&limit=M&live=sse` is the live feed. After the client is caught up, the connection stays open.

Headers on the response:

- `Content-Type: text/event-stream`
- `Cache-Control: no-cache`
- `X-Accel-Buffering: no` (disables response buffering in Nginx, otherwise events sit in a proxy until the buffer fills)

Each event is flushed before the next is written.

Two named events and one comment:

```
event: envelope
data: {"id":"…","part":"full",…,"server_seq":1}

event: cursor
data: {"next_cursor":1}

: ping
```

`envelope` carries one envelope including `server_seq`, the same object as in a pull page. `cursor` follows a burst of envelopes and reports `next_cursor` as in §3.2. `: ping` is an SSE comment (a line that begins with `:`). It is heartbeat: a write every 15 seconds so a mobile carrier NAT (network address translation) does not drop a silent connection. Comments are not delivered to the client application; they only keep the connection alive.

Blank lines between events are significant. [fixtures/live/stream.txt](fixtures/live/stream.txt) is a recorded body: one `envelope`, one `cursor`, one `: ping`, with those separators.

The token is checked when the stream opens. The server does not close the stream when the token's `exp` elapses. Reopening with a fresh token is the client's job.

Long polling (`live=poll`) is the fallback where a stream cannot pass: the server holds the request until an envelope is ready or 55 seconds elapse, then returns the JSON of §3.2. 55 seconds sits under the common 60-second idle limit of reverse proxies.

## 5. Limits and status codes

| Condition | Code |
|---|---|
| `limit` omitted | 100 envelopes |
| `limit` greater than 500, less than 1, or not an integer | `400` |
| `since` omitted | `0` |
| `since` not an integer ≥ 0 | `400` |
| `live` set to anything other than `sse` or `poll` | `400` |
| Request body larger than 1 MiB (1,048,576 bytes) | `413` |
| Push with more than one envelope | `413` |
| Push with zero envelopes | `400` |
| Missing, expired, or badly signed token | `401` |
| Missing or empty `sub` | `401` |

`401` responses on `/v1/*` do not explain which check failed.

## 6. Compatibility rules

Gaps in `server_seq` are legal. A client must not treat them as an error. The per-user counter is consumed before the upsert knows whether the envelope is stored; a rejected envelope leaves a hole.

Unknown JSON fields are ignored by both sides. Adding a field later must not break a v1 reader.

An unknown `schema_version` is stored as sent and is not interpreted by the server. The receiving client chooses a codec, or waits until it has one. The server has no opinion about payload versions.

## 7. `applied: false` is not an error

`applied: false` means the server already holds a row that is not inferior to the one just sent (§2). HTTP status is success.

The client **must** treat this as success and drop the row from its send queue.

Otherwise a lost network response retries forever: the second send of the same envelope always loses the comparison (the condition requires a strictly superior incoming row), so a client that retries on `applied: false` never drains the queue. The same rule makes a retry after a lost `applied: true` idempotent.

## 8. The server does not interpret payload

`payload` is bytes. After base64 decoding, the server stores those bytes and returns those bytes. It does not parse JSON, does not require UTF-8, and does not re-encode.

[fixtures/envelope/non_utf8_payload.json](fixtures/envelope/non_utf8_payload.json) is that promise as a program: the payload decodes to `FF FE 00 41`, which is neither UTF-8 nor JSON. A push of that envelope followed by a pull must yield the same bytes.

## 9. Operations (not the client protocol)

These three routes are not part of the client sync protocol. They are served only on the operations listener (`admin.bind`, default `127.0.0.1:8081`), never on the sync listener.

If `admin.token` is set, all three require `Authorization: Bearer <admin.token>`. If it is empty, the listener must be a loopback address; a non-loopback bind with an empty token is a startup failure, not a runtime 401.

| Method and path | Body | Purpose |
|---|---|---|
| `GET /admin` | HTML page | Read-only view of process health, counters, and redacted configuration. Envelope contents are never shown. |
| `GET /admin/events` | SSE stream | The same snapshot, pushed about once a second. |
| `POST /admin/token-check` | `{"token":"…"}` | Checks a user access token with the same verifier as `/v1/*`. Response is `{"valid":true,"subject":"…"}` or `{"valid":false,"reason":"…"}`. Unlike `/v1/*`, the reason is included: the operator needs to see why a token fails. |

Editing configuration through this surface is outside this version.

## 10. Glossary

| Term | Meaning |
|---|---|
| SSE | Server-Sent Events: a one-way HTTP stream (`text/event-stream`). The server writes; the client reads. |
| long polling | The server holds an HTTP request until an event exists or a timeout elapses; the client then opens the next request. |
| JWKS | JSON Web Key Set: a published list of **public** keys used to verify JWT signatures. The private key stays with the issuer. |
| upsert | One SQL statement that inserts a row or updates the existing row; the `WHERE` clause decides whether the update runs. |
| base64url | RFC 4648 section 5 alphabet (`-` `_`), usually unpadded. **Not** used for `payload`. |
| JWT | JSON Web Token: the signed access token in `Authorization: Bearer`. |
| `sub` | Subject claim inside the JWT; the user identifier the server uses as owner. |
