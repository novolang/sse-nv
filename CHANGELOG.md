# Changelog

All notable changes to sse-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `ssereader` — the load-bearing interface, and it answers no `Result`
  at all. The WHATWG parser has no failure mode: an unknown field, a
  malformed `retry`, an `id` holding a NULL and a line with no colon are
  all ignored, so a decoder that refused any of them would disconnect
  from a server behaving exactly as specified. The reader is the
  specification's four buffers, and `SseSignal` is what a block produced
  — a dispatched event, a new reconnection time, a comment, or a block
  dropped for size. The size limit is the one thing the specification
  does not have, and it is reported as `SseOverflow` and resynchronised
  at the next blank line rather than raised, because an unbounded buffer
  and a disconnection are both worse.
- `sseevent` — the event as its three parts, with the last event id on
  every event rather than tracked separately by the caller. The id
  buffer is not reset by a dispatch, which is the whole of the
  resumption mechanism; `ssereader.resuming` is the other half.
- `ssewire` — the line grammar as three arms and no fourth for
  "malformed". `split_lines` handles all three terminators, treats a
  carriage return and line feed pair as one, and holds a lone trailing
  carriage return back in the remainder because the next byte may be the
  line feed that belongs to it.
- `ssewriter` — encoding is not the inverse of parsing, and the
  difference is the newline: a `data` value may not contain one, so an
  event whose data is three lines is three `data` fields. `encode`
  writes them. `id_only` and `keep_alive` are the two blocks that
  deliberately dispatch nothing.
- `ssereconnect` — the two responses that mean stop for good, as a
  value. A 204 and a media type that is not `text/event-stream` are the
  only permanent stops; a client that added 404 or 503 to that list
  stops reconnecting to a server that was merely restarting. The delay
  is a number, because waiting is `[time]` work and this package is
  `core`.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  sse-nv.<module>.<fn>`.
- **No dependencies.** The format is lines of text. An event's `data` is
  usually JSON and is left as the text it arrived as, so a caller parses
  it with the decoder it already has.
