# Open questions

**Created:** 2026-08-26 10:26:24 +0500  
**Updated:** 2026-09-11 11:47:10 +0300  
**Version:** 2  
**Document type:** working notes

Findings that fall outside the current step are recorded here instead of being fixed opportunistically. Each entry has three parts: what was found, where, and why it is not resolved in this change.

## `/health` storage field shape

**Found:** SPEC.md defines `"storage":"<path>"` as a string; the reference server returns an object with `path` and `size_bytes`.

**Where:** SPEC.md §3.3 `GET /health`; ulsync-server `internal/httpapi/server.go` `GET /health`.

**Why not here:** Circle 1 step 17 does not change the wire contract; a dedicated SPEC PR follows.

## `/v1/whoami` is not in the protocol SPEC

**Found:** The reference server exposes `GET /v1/whoami` so a bearer token can be checked without push or pull. SPEC.md places operations only on `admin.bind` and does not list this route on the sync listener.

**Where:** SPEC.md §9 Operations; ulsync-server `GET /v1/whoami`.

**Why not here:** Circle 1 step 17 does not change the wire contract; a dedicated SPEC PR follows.

## Pull `limit` above the configured maximum

**Found:** SPEC.md §3.2 and §5 reject `limit` outside `1…500` with `400`. The reference server clamps to `sync.pull_limit_max` and returns `200`.

**Where:** SPEC.md §3.2 and §5 pull limits; ulsync-server `GET /v1/sync/pull` query parsing.

**Why not here:** Circle 1 step 17 does not change the wire contract; a dedicated SPEC PR follows.
