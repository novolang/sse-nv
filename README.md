# sse-nv

Server-sent events are a way for a server to push a stream of messages
to a client over one long-lived HTTP response. The wire format is
`text/event-stream`, specified in the
[WHATWG HTML standard](https://html.spec.whatwg.org/multipage/server-sent-events.html),
and it is what a browser's `EventSource` reads. This package decodes and
encodes that format in novo-lang.

## What the format is

A stream is UTF-8 text, made of lines. A line ends with a carriage
return and a line feed, a line feed alone, or a carriage return alone.
All three are terminators, and a carriage return followed by a line
feed is one terminator rather than two.

Lines are grouped into **blocks** by blank lines. Each line is one of
four things.

| The line | What the parser does |
| --- | --- |
| Empty | Dispatches the block |
| Begins with a colon | A comment. Ignored |
| Contains a colon | A field. The name is before the first colon, the value after it |
| Contains no colon | A field of that name, with an empty value |

Exactly one leading space is removed from a field's value, so
`data: hello` and `data:hello` both carry `hello` and `data:  hello`
carries ` hello`.

Four field names have a meaning. Any other is ignored.

| Field | Effect |
| --- | --- |
| `event` | Sets the event type of the block being accumulated |
| `data` | Its value is appended to the data buffer, with a newline after it |
| `id` | Sets the last event id |
| `retry` | Sets the reconnection time in milliseconds, when the value is entirely ASCII digits |

**Dispatching** is what a blank line does. If the data buffer is empty,
the parser resets the buffers and fires nothing. Otherwise it removes
one trailing newline from the data, takes the event type, which is
`"message"` when no `event` field set one, and fires one event. A
`data` line with an empty value still puts a newline in the buffer, so
it dispatches an event whose data is empty.

The **last event id** is not reset by a dispatch. A stream that sends an
id once and then fifty events without one produces fifty events all
carrying that id. When the connection ends, a client sends that id back
in a `Last-Event-ID` request header, and the server resumes from there.

A client whose connection ends waits the reconnection time and opens the
stream again. A response whose status is not 200, or whose
`Content-Type` is not `text/event-stream`, means stop permanently
instead (section 9.2.2 says to "fail the connection"). A server uses 204
No Content to tell clients to go away. A network error, where no
response arrived, is retried.

The parser has no failure mode. An unknown field, a `retry` that is not
a number, an `id` holding a NULL character and a line with no colon are
all ignored, and the stream carries on. This package therefore declares
no error type.

Every function in this package performs no input or output. It opens no
socket and waits out no reconnection delay: the reconnection time is a
number it computes and the program using it waits.

## Install

```
novo pkg add sse-nv
```

## Example

```novo
use std.bytes
use ssereader
use ssereconnect

fn main() [io]
    // A reader holds the bytes that have arrived but are not yet a
    // whole event, plus the last event id and the reconnection time.
    let r = ssereader.reader(ssereader.default_limits())

    // Bytes the transport read. `feed` answers at most one signal.
    let wire = bytes.to_byte_list(bytes.from_str("id: 42\ndata: hello\n\n"))
    let step = ssereader.feed(r, wire)
    match step.signal
        None    => println("a partial block; read more")
        Some(s) =>
            match s
                // A block that had data. This is the event.
                SseDispatch(e) => println("${e.kind}: ${e.data} (id ${e.id})")
                // The server asked for a different reconnection delay.
                SseRetry(ms)   => println("retry in ${ms}ms")
                // A keep-alive, which the parser ignores.
                SseComment(t)  => println("comment: ${t}")
                // A block too large for the limits, dropped.
                SseOverflow(n) => println("dropped ${n} bytes")

    // The connection ended. This is what to do about it.
    match ssereconnect.after_close(200, "text/event-stream",
                                   ssereader.retry_millis(step.reader))
        SseStopPermanently(why) => println("stopping: ${why}")
        SseReconnectAfter(ms)   =>
            println("wait ${ms}ms, then resume from ${ssereader.last_event_id(step.reader)}")
```

The program prints `message: hello (id 42)`, then `wait 3000ms, then
resume from 42`.

## What the package contains

| Module | Contents |
| --- | --- |
| `sseevent` | The dispatched event: its type, its data and the last event id in force when it fired. |
| `ssewire` | One line of a stream: the field names, the line shapes, and splitting text into complete lines. |
| `ssereader` | The stream parser, fed bytes and drained of events, retry directives, comments and overflows. |
| `ssewriter` | A server's side: an event turned back into the lines of a block. |
| `ssereconnect` | What a client does when a connection ends, including the responses that mean stop for good. |

## How to choose an entry point

**`ssereader.feed` takes bytes and answers at most one signal.**
`ssereader.take` answers the next signal already buffered, adding
nothing. A read that delivered forty events gives one from `feed`, so a
caller loops on `take` until it answers `None`.

**`ssereader.feed_text` is the same parser over text.** Take it when the
transport already decoded UTF-8, or when a test is feeding a literal.

**`ssereader.resuming` starts a reader that already knows the last event
id.** Take it on every reconnection. A fresh reader reports an empty id
for every event until the server sends another one.

**`ssewriter.encode` writes one event.** `ssewriter.data` is the
shortest block a server can send, `ssewriter.keep_alive` the smallest
thing that holds a connection open, and `ssewriter.id_only` moves a
client's resumption point without delivering an event.

## The rules a user needs

1. **Exactly one leading space is removed from a field's value.** Not
   all of them. A stream sending indented text keeps its indentation.
2. **A carriage return alone ends a line.** So does a line feed, and so
   does the pair, which counts as one terminator. A parser that handles
   only the line feed reads a whole stream from an older server as one
   line.
3. **A block whose data buffer is empty dispatches nothing.** A block of
   nothing but an `id` line updates the id and fires no event, and so
   does a comment. A decoder that fired an empty event delivers one
   every few seconds to any server that uses a bare `id` as its
   heartbeat.
4. **Several `data` fields become one event, joined by newlines**, with
   one trailing newline removed. `sseevent.data_lines` splits them back.
5. **The last event id survives a dispatch; the event type does not.**
   An event that sends no `event` field is of type `"message"`. An event
   that sends no `id` carries whatever id was last set, however long ago.
6. **An `id` whose value holds a NULL character is ignored**, and the
   previous id stands. An `id` line with no value sets the id to the
   empty string, and a reconnecting client then sends no header.
7. **A `retry` value that is not entirely ASCII digits is ignored**, and
   the previous reconnection time stands. `retry: 5s` and `retry: -1`
   change nothing.
8. **An unknown field name is ignored.** Field names are
   case-sensitive, so `Event:` is not `event:`.
9. **A `data` value may not contain a newline on the wire.** An event
   whose data has three lines is encoded as three `data` fields.
   `ssewriter.encode` does that; a server building the line itself sends
   a block whose second line is whatever came after the newline.
10. **A partial block is discarded when the connection ends.** The
    stream stopped in the middle of an event, so the event was never
    dispatched.
11. **A response that is not a 200 event stream means stop reconnecting,
    permanently.** That covers 204 No Content, which a server sends on
    purpose, every other status, and a `Content-Type` that is not
    `text/event-stream`. A network error with no response is retried,
    and so is a stream that ended. Pass status 0 to
    `ssereconnect.after_close` for a network error.
12. **Send the last event id back when reconnecting.** The header is
    `Last-Event-ID`. An absent header and an empty one are different: a
    client with no id sends none.
13. **A block larger than the limit is dropped, not an error.** The
    specification sets no size limit and a caller cannot buffer without
    one. A block whose data passes `max_event_bytes`, or which holds a
    line longer than `max_line_bytes`, is dropped, and the reader
    reports `SseOverflow` at the blank line that ends it. The stream
    continues after that line.
14. **A leading byte order mark is removed** from the start of the
    stream, even when its three bytes arrive in separate reads. A byte
    order mark anywhere else is kept.
15. **A carriage return is acted on as soon as it arrives.** A line
    feed in the next read is taken as the second half of the same
    terminator. `ssewire.split_lines`, for text that is not a live
    stream, holds a trailing carriage return back in the remainder
    instead.
16. **An id holding a NULL, a carriage return or a line feed is not
    written.** `ssewriter.encode` leaves it out, because the reader
    would ignore it or read it as two lines. An event type holding a
    line terminator is written up to the first one.

## What is not included

- **A socket, and the wait between reconnections.** This package opens
  nothing and sleeps for nothing. A program using it reads bytes itself,
  and waits out the number `ssereconnect.after_close` gives it.
- **The HTTP request and response.** Asking for a stream is an HTTP
  request with an `Accept` header, which is
  [http-codec-nv](https://novo-lang.org/packages/http-codec-nv)'s work.
  This package handles the body.
- **JSON.** An event's `data` is very often a JSON document, and this
  package leaves it as the text it arrived as, so a caller parses it
  with the JSON decoder it already has.
- **A retry schedule.** Backoff, jitter and a cap on attempts are policy
  a client owns. This package answers what the specification says: the
  delay the server asked for, or the decision not to reconnect at all.

## Related packages

- [websocket-codec-nv](https://novo-lang.org/packages/websocket-codec-nv)
  is a two-way protocol over one connection. Take it when the client
  sends messages too. Server-sent events are one direction only, and
  they reconnect and resume by themselves.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) reads
  and writes the HTTP/1.1 messages this format travels inside. Its
  reader has the same feed-and-drain shape as this package's.
- [jsonrpc-nv](https://novo-lang.org/packages/jsonrpc-nv) is a
  request-and-response protocol with its own framing, for a peer that
  answers rather than streams.

## Tests

```bash
novo test tests/ssereader_tests.nv   # the line grammar and the parser
novo test tests/ssewriter_tests.nv   # the encoder and the reconnection rules
bash tests/coverage.sh               # line coverage over src/
```

The normative source is the WHATWG HTML standard's server-sent events
section. Its example streams are in the suite, each fed whole and in
pieces of one, two, three and seven bytes: the stock ticker whose three
`data` lines make one event, the four blocks whose second resets the id
with a bare `id` line, the empty `data` fields, the two identical
events, and the `add` and `remove` events.

The suite also asserts that exactly one leading space is removed from a
value, that all three line terminators end a line and that a carriage
return and line feed pair is one of them, that a block with no data
dispatches nothing, that the last event id survives a dispatch while the
event type does not, that a malformed `retry` leaves the time alone,
that an overlong line drops its block without ending it early, and that
every event, including empty and multi-line data, decodes back to
itself after `ssewriter.encode`.

## Licence

Apache-2.0. See [LICENSE](LICENSE).
