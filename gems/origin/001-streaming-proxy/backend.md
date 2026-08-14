# Streaming Proxy — Backend

## Problem

Returning a long-running generation task only after it fully completes increases time-to-first-response and may leave server work running even after the client disconnects.

## Architecture

```text
HTTP Controller
    ↓ validate
Application Service
    ↓ async events
Event Sequencer
    ↓
NDJSON Serializer
    ↓
Bounded Stream Writer
    ↓ flush
Reverse Proxy
    ↓
Client
```

Cancellation propagates in the opposite direction.

```text
Client Disconnect → HTTP Abort → Service → Upstream Cancellation
```

## Event Contract

Each Event is serialized as exactly one line of JSON.

```json
{
  "version": "1",
  "requestId": "request-id",
  "sequence": 1,
  "type": "content.text",
  "payload": {},
  "timestamp": "ISO-8601"
}
```

A successful Stream emits exactly one Terminal Event. Validation failures before streaming begins are returned as HTTP errors; failures after the Stream starts are represented as sanitized Error Events.

## Responsibilities

The Controller owns only the HTTP lifecycle. It should not perform business work such as retrieval, prompt assembly, or provider orchestration. The Service should not know about the HTTP Response object.

The Stream Writer owns writability, backpressure, flushing, write failures, and duplicate termination protection.

## Backpressure

Do not allow unbounded buffering. Give each request a bounded queue, slow upstream consumption when the consumer is slow, and safely cancel the request if an uncontrollable upstream exceeds the allowed buffer.

## Failure / Recovery

- validation failure before streaming → HTTP 4xx
- business failure after streaming starts → sanitized error event → close
- client disconnect → cancel upstream without attempting additional writes
- no Events after a Terminal Event

## Observability

Record metadata such as requestId, first-event time, event count, byte count, last sequence, duration, abort source, and error category rather than raw payloads.

## Validation

- each Event is exactly one NDJSON line
- sequence numbers increase monotonically
- exactly one Terminal Event is emitted
- disconnect propagates to upstream cancellation
- slow clients cannot grow the buffer beyond its configured bound
