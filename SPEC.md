# ulsync protocol specification v1

**Created:** 2026-08-26 10:26:24 +0500  
**Updated:** 2026-09-16 19:53:33 +0300  
**Version:** 4  
**Document type:** specification

This document is the wire contract. A server written in Go and a package written in Dart, produced independently, must converge on these files. Divergence is a failing test on a fixture, not a first run on two devices.

v1 describes one envelope, three client endpoints, a live feed, and an operations surface that is not the client protocol. This version describes a push of 1…500 envelopes in one request and `part` values other than `full`. Payload compression and WebSocket remain outside this version.

## 1. Envelope

An envelope is one JSON object. On the wire it is not the storage row: `user_id` is never in the JSON, and `server_seq` appears only in some responses.

### 1.1. Fields on the wire

Each field: JSON type, required on every envelope that carries it, who writes it, what it means.

| Field | Type | Required | Written by | Meaning |
|---|---|---|---|---|
| `id` | string | yes | client | Identity of the record. Stable across devices. |
| `part` | string | yes | client | Which slice of the record this envelope holds. Identity is `(id, part)`. `full` is the complete snapshot of the record. Any other non-empty string is an application-defined slice: the server does not interpret the string and keeps no registry of names. Application examples such as `done` or `deleted` are not reserved protocol values. There is no tombstone type. Last-write-wins (§2) compares inside one `(id, part)` pair and does not cross to a neighbouring name. |
| `entity_type` | string | yes | client | Codec key for the receiving client. The server does not validate it and keeps no list of types. |
| `created_at_ms` | number (integer) | yes | client | Time the record was created. |
| `last_edited_at_ms` | number (integer) | yes | client | Time of the last edit. First rank of conflict resolution. At creation equals `created_at_ms`. |
| `revision` | number (integer) | yes | client | Edit counter. Second rank of conflict resolution. Greater wins. |
| `source_id` | string | yes | client | Identifier of the producing installation. Third rank of conflict resolution. The server compares it and does not interpret it. Must be non-empty. Must be unique per installation (§1.4). |
| `flags` | number (integer) | yes | client | Protocol flags. This version sends `0`. Hiding a record is not a bit in this field: it is an application-defined part. There is no tombstone type. |
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

### 1.4. A unique `source_id` per installation

`source_id` is the third rank of §2 and the only tiebreaker when two envelopes carry the same `last_edited_at_ms` and the same `revision`. An installation is one installed copy of a client: one device, one app install. Two installations that report the same `source_id` turn such a pair into a genuine tie: no receiver can break it, each side keeps its own copy, the push of the other side is answered `applied: false` (§7), and two different payloads under one `(id, part)` stay different forever. Every other way two copies of the same user's store can diverge is detectable and repairable; this one is not.

A client therefore **must** create `source_id` once per installation and **must not** let it reach another device. The two ways it travels are a restored device backup and a cloned virtual machine image; keeping the value out of platform backups is the client's job. A `source_id` that changes between launches is wrong for a different reason: every edit then looks like it came from a fresh installation, and the tiebreaker stops being stable.

This is a client duty. The server compares the string and cannot tell two installations that share a value apart.

### 1.5. Origin of a store

`origin` is a non-empty string that names the **application contour** this store belongs to: one deployment of one application (production, staging, or a private server). It is not a URL, not a user id, and not `source_id`. Envelope identity remains `(id, part)` for a given user (§1.1). The server never reads `origin` from `payload`, from `entity_type`, or from the JWT `iss` (issuer) claim.

The client mints `origin` **once per application contour**, typically the reverse-DNS name of the application plus a project UUID (Universally Unique Identifier), stores it in source control next to the server URL, and sends the same value from every installation of that contour. An `origin` minted per device is a protocol violation: the second installation would be refused forever.

Two stores that report the same `origin` are, for this protocol, one store. Two contours of one application **must** use two values; sharing one value across production and staging is a configuration error, not a protocol hole.

The store is in one of two modes. The words are part of the contract.

An **open** store has no origin in its configuration. The first well-formed `Ulsync-Origin` on `GET /v1/sync/hello` is recorded and becomes the store origin. That first write is **imprint**: it is store metadata, not an envelope and not a `server_seq`. Mail endpoints (`POST /v1/sync/push`, `GET /v1/sync/pull`, `POST /v1/sync/diff`, and the live feed of §4) never imprint. A well-formed header against an as-yet-unimprinted open store on those four is `400` with `{"error":"origin_required"}`: hello must run first. A missing header on those four is honoured (a **legacy client**) and leaves the store unimprinted.

An **authored** store has the origin in its configuration before any client talks to it. The store is named even while the envelope table is empty. A different value or a missing header is refused.

#### Header `Ulsync-Origin`

Every `/v1/sync/*` request from a current client carries this header. It is not the CORS (Cross-Origin Resource Sharing) header `Origin`, and it does not replace `Authorization`.

| Header | Required | Meaning |
|---|---|---|
| `Ulsync-Origin` | On an authored store: yes. On an open store: no for a legacy client, yes for a current client | The client's origin. Characters: `A–Z`, `a–z`, `0–9`, `.`, `_`, `/`, `-`. Length 1–256 |

A present header that is empty is treated as missing (`origin_required` where a missing header is an error). A present header that is longer than 256 characters, or that contains a character outside the class, is `origin_invalid`.

`GET /health` does not use this header.

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

The `Ulsync-Origin` header (§1.5) is compared on every `/v1/sync/*` route after the token is accepted. Codes for a missing, invalid, or mismatched value are in §3.5. `GET /health` does not carry the header.

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

Each result names the envelope and whether the upsert stored it. `server_seq` is physically absent from this response: the cursor moves only from pull (§1.3). The `results` array is in request order: index *i* is envelope *i*. A mix of `applied: true` and `applied: false` under HTTP `200` is legal: each element is the ordinary outcome of §2 for that `(id, part)`. Last-write-wins does not abort the request.

This version accepts 1…500 envelopes in `envelopes`. 500 matches the maximum `limit` on pull (§3.2) and the maximum `items` on diff (§3.4). Zero envelopes is `400`. More than 500 is `413`; the JSON body includes `limit` (integer), the maximum this server accepts in one push, so the client sees the ceiling. This version's ceiling is 500:

```json
{"error":"too many envelopes","limit":500}
```

Two envelopes in one request that share the same `(id, part)` are `400`. Nothing from that request is stored. The client chooses a winner before sending; the server does not pick one inside the array.

The server parses and validates every envelope **before** any write. A malformed forty-seventh element does not leave forty-six rows on the store. After that check passes, one store transaction compares the whole array: either every envelope is compared by §2, or none is. A comparison that loses is `applied: false` on that result. It is not a reason to roll the transaction back.

[fixtures/push/request_single.json](fixtures/push/request_single.json) is one envelope. [fixtures/push/request_batch.json](fixtures/push/request_batch.json) is two envelopes with different `id` values, both `part` `full`; [fixtures/push/response_batch.json](fixtures/push/response_batch.json) is the matching `200` body. [fixtures/push/request_two_parts.json](fixtures/push/request_two_parts.json) is one `id` with `part` `full` then `part` `done`; [fixtures/push/response_two_parts.json](fixtures/push/response_two_parts.json) is the matching `200` body. The name `done` is an application example, not a reserved protocol value; [fixtures/envelope/part_done.json](fixtures/envelope/part_done.json) is that envelope alone.

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

### 3.4. `POST /v1/sync/diff`

Divergence check: the client reports the version it holds for a batch of records, and the server answers which of them it does not hold at all and which it holds in a version that **loses** to the client's by the conflict rule of §2. It is not a pull: no envelopes, no `payload`, no `server_seq` are returned, and nothing is written.

Request:

```json
{"items":[{"id":"<id>","part":"full","last_edited_at_ms":1756100000000,
           "revision":3,"source_id":"<installation>"}]}
```

| Field | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | yes | Record identity, as in §1.1 |
| `part` | string | yes | Slice, as in §1.1. Identity is `(id, part)` |
| `last_edited_at_ms` | number (integer) | yes | First rank of §2, as the **client** holds it |
| `revision` | number (integer) | yes | Second rank of §2, as the **client** holds it |
| `source_id` | string | yes | Third rank of §2, as the **client** holds it. Must be non-empty |

`items` is required and must be a JSON array of those objects. The three ranks are sent together because §2 ranks by all three, in order. A request that carries only some of them cannot be answered without guessing the rest.

`entity_type` is **not** part of this request. Identity of a stored row is `(id, part)` (§1.1); the server keeps no list of types and cannot use one to match rows. A client that stores several types under one `id` cannot be answered by any endpoint, and this one is not an exception.

Response:

```json
{"missing":[{"id":"<id>","part":"full"}],
 "stale":[{"id":"<id>","part":"full","last_edited_at_ms":1756000000000,
           "revision":1,"source_id":"<other installation>"}]}
```

- `missing` — the server holds no row for this `(id, part)` and this user.
- `stale` — the server holds a row that **loses** to the one in the request by §2. The three rank fields in a `stale` entry are the **server's** values, present so a human reading the response can see how far behind it is.
- A row that wins, or ties on all three ranks, appears in neither list. A tie means the two versions are equivalent by §2 (§7), so there is nothing for the client to do.
- Both arrays are always present. Nothing to report is `{"missing":[],"stale":[]}`, not an omitted field and not `null`.
- Entries appear in the order of first appearance in the request. A key repeated in the request is answered once; comparison uses the ranks from the first occurrence.

Both lists mean the same thing to the client: mark the record for sending and push it. The push upsert still decides by §2, so this endpoint cannot be used to overwrite a newer row on the server.

The question this endpoint answers is whether the client's copy would win if sent. A weaker comparison leaves holes that live forever, and two of them are ordinary rather than exotic. First: equal `revision`, the client's `last_edited_at_ms` greater. A revision-only check reports nothing, yet the client's copy is the winner and is not queued for sending. Second: the client's `revision` lower **and** its `last_edited_at_ms` greater — a revision-only check reads this as the server being ahead, while §2 says the client wins. Revision counters of two installations are unrelated numbers, so the second case needs no unusual timing at all. In both, a pull cannot repair the divergence: an envelope that loses is skipped, and the local copy is not marked for sending.

Ranking here is **the same rule as §2, not a second one**. The server implements the comparison once and calls it from both the upsert and this endpoint (§3.1, §2). Two copies of the rule drift apart; one function called twice cannot.

| Condition | Code |
|---|---|
| `items` empty | `400` |
| `items` longer than 500 | `413` |
| `id`, `part`, or `source_id` missing or empty | `400` |
| `last_edited_at_ms` or `revision` not an integer, or negative | `400` |
| Body larger than 1 MiB | `413` |
| Missing, expired, or badly signed token, missing or empty `sub` | `401` |

500 matches the maximum `limit` on pull (§3.2), so a client has one batch size for the whole protocol. One request item is about 120 bytes, so a full batch is about 60 KiB against the 1 MiB body limit.

The user comes from the verified token, never from the body, exactly as in §3.1 and §3.2. A key belonging to another user is reported as `missing`, because for this user it does not exist.

[fixtures/diff/request.json](fixtures/diff/request.json) is a four-item request: a three-rank tie, a key the server does not hold, a stale row that loses on `last_edited_at_ms` at equal `revision`, and a stale row that loses on `last_edited_at_ms` despite a greater `revision`. [fixtures/diff/response_gaps.json](fixtures/diff/response_gaps.json) is the matching response. [fixtures/diff/request_tie.json](fixtures/diff/request_tie.json) is a one-item tie; the expected body is [fixtures/diff/response_empty.json](fixtures/diff/response_empty.json).

### 3.5. `GET /v1/sync/hello`

Handshake: the client names its origin, the server records it on an open store that has no origin yet or compares it with the origin the store already holds, and nothing about envelopes is read or written.

The request has no body. `Ulsync-Origin` is as in §1.5. A missing, expired, or badly signed token, and a missing or empty `sub`, produce `401` as in §3.1.

Response `200`:

```json
{"origin":"com.example.app/7c3e9a12-4b56-4d8e-9f01-2a3b4c5d6e7f","user_id":"<sub>"}
```

`origin` is the value the store holds **after** this request (after imprint it equals the header). `user_id` is the `sub` of the verified token, so the client can confirm it is not talking to another account on the same store.

| Condition | Code | Body |
|---|---|---|
| Authored or already imprinted store, header present and equal | `200` | as above |
| Open store, header present, store has no origin yet | `200` and the store is imprinted | as above |
| Header present, well-formed, and different from the store origin | `409` | `{"error":"origin_mismatch","store_origin":"<store>","request_origin":"<header>"}` |
| Hello, header missing or empty (open or authored) | `400` | `{"error":"origin_required"}` |
| Header present but not matching the character class or longer than 256 | `400` | `{"error":"origin_invalid"}` |
| Missing, expired, or badly signed token | `401` | as in §3.1 |

Hello **may** write one row of store metadata on an open store. It **must not** insert, update, or delete an envelope, and **must not** allocate `server_seq`.

The same comparison and the same mismatch, invalid, and auth codes apply to `POST /v1/sync/push`, `GET /v1/sync/pull`, `POST /v1/sync/diff`, and the live feed of §4. Live in this list is that feed (`GET /v1/sync/pull` with `live=sse`); this version has no separate `GET /v1/sync/live` path. Those four must not imprint. Extra rules for them:

- open store, header absent — the request proceeds (legacy client); the store stays as it was;
- open store, not yet imprinted, header present — `400` `origin_required` (hello first);
- authored store, header absent — `400` `origin_required`;
- any store, header present and different from the stored origin — `409`.

There is no second, weaker check. A present header that mismatches is `409` on every `/v1/sync/*` route.

[fixtures/origin/hello_response.json](fixtures/origin/hello_response.json) is a `200` body. [fixtures/origin/mismatch.json](fixtures/origin/mismatch.json) is a `409` body. [fixtures/origin/origin_required.json](fixtures/origin/origin_required.json) and [fixtures/origin/origin_invalid.json](fixtures/origin/origin_invalid.json) are the two `400` bodies.

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

The token is checked when the stream opens. `Ulsync-Origin` is compared then too (§3.5). The server does not close the stream when the token's `exp` elapses. Reopening with a fresh token is the client's job.

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
| Push with more than 500 envelopes | `413` |
| Push with two envelopes that share `(id, part)` | `400` |
| Push with zero envelopes | `400` |
| Diff with an empty `items` array | `400` |
| Diff with more than 500 items | `413` |
| Diff with an item missing `source_id`, or with a non-integer or negative rank | `400` |
| Missing, expired, or badly signed token | `401` |
| Missing or empty `sub` | `401` |
| `Ulsync-Origin` present but not in the character class, or longer than 256, on hello or any `/v1/sync/*` | `400` |
| `Ulsync-Origin` well-formed and different from the store origin | `409` |
| Authored store, `Ulsync-Origin` missing or empty | `400` |
| Open store, not yet imprinted, `Ulsync-Origin` present on push, pull, diff, or live | `400` |
| Hello, `Ulsync-Origin` missing or empty | `400` |

`401` responses on `/v1/*` do not explain which check failed.

## 6. Compatibility rules

Gaps in `server_seq` are legal. A client must not treat them as an error. The per-user counter is consumed before the upsert knows whether the envelope is stored; a rejected envelope leaves a hole.

Unknown JSON fields are ignored by both sides. Adding a field later must not break a v1 reader.

An unknown `schema_version` is stored as sent and is not interpreted by the server. The receiving client chooses a codec, or waits until it has one. The server has no opinion about payload versions.

A server that does not implement §3.4 answers `404` (or `405`). A client **must** treat that as "divergence check unavailable" and continue working; the check is an additional safety net, not a precondition for sync.

A server that does not implement §3.5 answers `404` (or `405`) to `GET /v1/sync/hello`. A client **must** treat that as "origin handshake unavailable" and continue; an old server cannot refuse a foreign application. A current client talking to a current server **must** call hello before the first push, pull, diff, or live of that client instance.

A current server in **open** mode that receives a `/v1/sync/*` request other than hello **without** `Ulsync-Origin` **must** honour it: that is a legacy client, and old clients with a new server are required to work. A current server in **authored** mode **must not**: missing origin is `400`. This is the only compatibility exception the operator opts into by setting `origin` in configuration.

A client that sends one envelope remains valid. Old clients with a new server of this version are required to work. A server that still accepts only one envelope answers `413` to a longer array; a current client **may** treat that as failure of the push and **must not** be specified here as required to split the array. New clients with an old server are not required to work.

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
| origin | Non-empty string naming the application contour this store belongs to. Sent as `Ulsync-Origin`. Not a URL, not a user id, not `source_id`. |
| open store | A store with no origin in configuration. The first well-formed hello header imprints it. |
| authored store | A store whose origin was set in configuration before any client. A missing or different header is refused even when no envelopes exist. |
| imprint | The first write of origin into store metadata on an open store. Only hello does this; mail endpoints never imprint. |
