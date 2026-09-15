# Changelog

**Created:** 2026-08-26 10:26:24 +0500  
**Updated:** 2026-09-15 12:34:56 +0300  
**Version:** 3  
**Document type:** changelog

All notable changes to this project are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Envelope format, last-write-wins conflict resolution, push / pull / health endpoints, live feed, limits, compatibility rules, and golden fixtures for protocol v1.
- Divergence-check endpoint `POST /v1/sync/diff`: the client sends version metadata only; the server names keys it does not hold and keys it holds in a losing version. Backwards compatible: an older server answers `404`, and the client keeps working.
- Requirement that `source_id` is unique per installation (§1.4). This clarifies a client duty; the envelope format is unchanged.
- Store origin: the store remembers which application contour it belongs to (`Ulsync-Origin`). An **open** store is imprinted only from `GET /v1/sync/hello`; mail endpoints never write origin.
- An **authored** store has origin in configuration before any client. A missing or different header is refused even when no envelopes exist. This deliberately breaks the legacy-client exception for that volume.
- A legacy client that omits `Ulsync-Origin` continues to work against an **open** store. A current client talking to a current server must call hello before the first push, pull, diff, or live of that instance.
