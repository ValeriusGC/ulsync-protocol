# ulsync-protocol

**Created:** 2026-08-26 10:26:24 +0500  
**Updated:** 2026-08-26 10:26:24 +0500  
**Version:** 1  
**Document type:** instruction

Wire contract for ulsync: the envelope JSON, the three sync HTTP endpoints, and the golden examples both implementations must accept.

This repository holds text and examples. It is not a library. A Go server and a Dart package read the same files and must agree without talking to each other. Putting the specification inside the server repository would make the server correct by definition, and a mismatch would look like an opinion instead of a defect.

## Who this is for

Authors of an ulsync server or client. Anyone else implementing the same wire format.

## How to read

1. [SPEC.md](SPEC.md) is the specification. Behaviour not written there is not part of v1.
2. [fixtures/](fixtures/) are the examples that tests load. They are not illustrations for a human reader.

The Go server and the Dart package add this repository as a git submodule and run their tests against the fixtures. Changing a fixture without changing `SPEC.md` and `CHANGELOG.md` is a defect: one side will pass and the other will fail, and the failure will show up only when both run.

## License

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

A contract that cannot be implemented without a legal review will not be adopted. Apache 2.0 is the already-chosen license of the ulsync family; it lets anyone implement the format, including in closed products, provided they keep the copyright notices.

## Layout

```
SPEC.md                 wire format and endpoints
fixtures/envelope/      one envelope, including a non-UTF-8 payload
fixtures/push/          one request and two responses
fixtures/pull/          a page and an empty page
fixtures/live/stream.txt recorded Server-Sent Events body
docs/OPEN_QUESTIONS.md  findings that do not belong in the current change
```
