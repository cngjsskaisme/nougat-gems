# Streaming Proxy FE Program Design Document

## 1. Document Purpose

This document defines the design for delivering the backend's NDJSON stream to the browser without loss in a Nuxt-based frontend application, and for the browser to progressively parse it and convert it into UI events.

The frontend scope includes the following two execution environments.

```text
Browser Runtime
Nuxt/Nitro Server Runtime
```

Nitro serves as the server runtime owned by the frontend application and acts as a Reverse Proxy.

---

## 2. Design Scope

### In Scope

- Browser Streaming API Client
- `ReadableStream`-based Chunk reading
- UTF-8 incremental decoding
- NDJSON Buffer and line-by-line parsing
- Stream Event Dispatcher
- `AbortController`-based cancellation
- Nitro `routeRules` Reverse Proxy
- Endpoint selection per Browser/SSR execution environment
- Authentication header and 401 retry coordination
- Buffering prevention in Proxy and intermediate layers

### Out of Scope

- Detailed chat screen composition
- UI component design per business event
- Backend LLM calls and event generation
- Encryption and decryption
- Conversation storage and RAG processing

---

## 3. Target Structure

```text
UI / Chat State
    ↓
Streaming API Facade
    ↓
Client Transport
    ├─ Request Builder
    ├─ Authentication Adapter
    ├─ Abort Manager
    └─ Retry Coordinator
    ↓
fetch()
    ↓
/api/**
    ↓
Nitro routeRules Reverse Proxy
    ↓
Backend Streaming Endpoint
    ↓
NDJSON Chunks
    ↓
ReadableStream Reader
    ↓
Incremental Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓
Event Dispatcher
    ↓
Chat State / Rendering
```

---

## 4. Key Components

### 4.1 Streaming API Facade

Provides a streaming call interface to the upper chat layer.

**Responsibilities**

- Receive request model
- Communicate stream start and end states
- Provide event callbacks or Async Iterator
- Return cancellation handle
- Convert Transport errors to application errors

The upper layer should not directly know about `fetch`, `ReadableStream`, or Buffer handling methods.

### 4.2 Client Transport

Performs the actual HTTP request in the browser.

**Responsibilities**

- Use relative path `/api` in Browser Runtime
- Use internal Backend Base URL or runtime configuration in SSR Runtime
- Compose request Headers
- Validate response status and Content-Type
- Expose `Response.body`
- Connect cancellation signal

High-level response transformation functions that load the entire response into memory are not used for streaming requests.

### 4.3 Authentication Adapter

Injects authentication metadata required for transport.

Example logical items are as follows.

```text
Authorization
Timestamp
Nonce
Signature
```

SSR internal requests may use a separate internal request identification Header.

Authentication generation logic is separated from the Streaming Parser.

### 4.4 Retry Coordinator

Handles retryable errors such as authentication expiration.

```text
Initial Request
    ↓ 401
Guest/User Session Initialize or Refresh
    ↓
Create New Request
    ↓
Retry Once
```

If some stream events have already been applied to the UI, the entire request is not automatically retried. Automatic retry is limited to authentication failures before consuming the response body.

### 4.5 Stream Reader

Reads Chunks sequentially from `Response.body`.

```text
Response.body
    ↓
getReader()
    ↓
read()
    ↓
Uint8Array Chunk
```

Empty body, non-streaming response, and early termination are clearly distinguished.

### 4.6 Incremental Decoder

Incremental decoding is used because Chunk boundaries and UTF-8 character boundaries do not align.

```text
Uint8Array Chunk
    ↓
TextDecoder(stream mode)
    ↓
Decoded Text Fragment
```

On stream termination, the Decoder's remaining internal buffer is flushed.

### 4.7 Line Buffer

Handles situations where a single NDJSON line is split across multiple Chunks or a single Chunk contains multiple lines.

```text
buffer += decodedFragment
lines = split by newline
completeLines = all except last
buffer = last incomplete line
```

Only completed lines are passed to the Parser, and the last incomplete line is preserved until the next Chunk.

### 4.8 NDJSON Parser

Parses each completed line as an independent JSON Event.

**Responsibilities**

- Ignore empty lines
- JSON parsing
- Event Contract validation
- Handle unknown event types
- Record parse error location and request identifier

Whether a single line error leads to overall connection termination or is converted to an error event is determined by contract policy. The default is to terminate the stream on protocol error.

### 4.9 Event Dispatcher

Dispatches parsed events to the chat state layer.

Representative logical events are as follows.

```text
meta
text
block
dealCards
comparison
questions
done
error
```

The Event Dispatcher does not directly call rendering components but converts events to state change commands.

### 4.10 Abort Manager

Cancels the stream in situations such as user stop, screen navigation away, or starting a new request.

```text
UI Stop Action
    ↓
AbortController.abort()
    ↓
fetch cancellation
    ↓
Nitro connection close
    ↓
Backend cancellation propagation
```

Cancellation is distinguished from normal errors so that unnecessary error messages are not displayed in the UI.

### 4.11 Nitro Reverse Proxy

Forwards `/api/**` requests to the backend in the Nuxt server runtime.

```text
Browser /api/deals/chat
    ↓
Nitro routeRules
    ↓
Backend /api/deals/chat
```

**Responsibilities**

- Provide same-origin API
- Simplify CORS boundaries
- Forward request Headers and Body
- Forward response Status and Headers
- Pass Streaming Body through without reassembly
- Adjust cache and compression policies

A structure where a separate Nuxt API Handler reads the entire response and returns it again is avoided.

---

## 5. Full Request Sequence

```text
User
    ↓
Chat State
    ↓
Streaming API Facade
    ↓
Client Transport
    ↓ fetch /api/**
Nitro routeRules Reverse Proxy
    ↓
Backend Streaming Endpoint
    ↓ NDJSON chunks
Nitro Reverse Proxy
    ↓
ReadableStream Reader
    ↓
Incremental Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓
Event Dispatcher
    ↓
Chat State / UI
```

---

## 6. Stream State Model

```text
idle
  ↓ start
connecting
  ↓ headers received
streaming
  ↓ done event or EOF
completed
```

Exception states are as follows.

```text
connecting → failed
streaming → failed
connecting → cancelled
streaming → cancelled
```

### State Principles

- `connecting`: Before response Headers arrive
- `streaming`: Processing one or more Chunks
- `completed`: Normal `done` event or agreed-upon normal EOF
- `failed`: Network, HTTP, protocol, or parsing error
- `cancelled`: User intentionally aborted

---

## 7. NDJSON Contract Handling

One Event must be represented as a single line of JSON.

```text
{event 1}\n
{event 2}\n
{event 3}\n
```

### FE Validation Items

```text
type
requestId or conversationId(optional)
sequence(optional)
payload
timestamp(optional)
```

If `sequence` is provided, duplicate, out-of-order, and missing events can be detected.

If there is remaining text in the Buffer on stream termination, it is checked whether it is the last completed Event. If it is incomplete JSON, it is treated as a protocol error.

---

## 8. Error and Retry Policy

| Section | Error | Handling Principle |
|---|---|---|
| Before request | Authentication info generation failure | Abort transmission |
| Before response | 401 | Retry once after session initialization/refresh |
| Before response | 4xx | Convert to business error, no automatic retry |
| Before response | 5xx | Mark as retryable, no automatic repetition |
| Streaming | Connection interrupted | Whether to preserve received content is determined by event contract |
| Streaming | Invalid JSON | Stream termination and protocol error |
| Streaming | `error` event | Convert server error message to safe UI message |
| Streaming | EOF without `done` | Default is to treat as abnormal termination |
| User Action | Abort | Terminate with `cancelled` state |

If a partial response has already been displayed, automatically retrying the same request may produce duplicate messages, so an explicit user re-request is the principle.

---

## 9. Proxy Design Requirements

### Streaming Preservation

- The Proxy must not buffer the entire response.
- The response Body must not be transformed into a JSON object.
- Avoid forced Content-Length calculation.
- Pass the NDJSON Content-Type as-is.
- Disable Proxy buffering in intermediate web servers.

### Header Policy

Example forwarding targets are as follows.

```text
Content-Type
Cache-Control
Connection-related streaming headers where applicable
Request/Trace ID
```

Hop-by-Hop Headers are cleaned up according to runtime and Proxy standards.

### Cache Policy

Streaming APIs are not cached.

```text
Cache-Control: no-store
```

### Environment Configuration

The Proxy Target may vary by deployment environment.

```text
Local Backend
Development Backend
Staging Backend
Production Backend
```

If `routeRules` uses build-time configuration, the necessary environment variables must be provided before the build.

---

## 10. Operational Deployment Structure

```text
Browser
    ↓ HTTPS
Apache / Nginx / Load Balancer
    ↓
Nuxt/Nitro
    ↓ routeRules proxy
Backend
```

Each intermediate layer verifies the following.

- Response buffering disabled
- Idle Timeout is longer than the maximum stream duration
- No Chunk Flush delay
- Compression does not excessively delay small Chunk delivery
- Connection termination is propagated to Backend cancellation

---

## 11. Observability

Metadata that can be recorded on the FE is as follows.

```text
requestId
connection start time
first-byte time
first-event time
event count
last sequence
stream duration
completion status
error category
abort reason
```

The default principle is not to record actual user messages and Event Payloads in operational logs.

---

## 12. Non-Functional Requirements

### Responsiveness

- Apply to state as soon as the first Event arrives.
- Do not wait for the entire response to complete.
- Avoid excessive full message re-rendering per Chunk.

### Reliability

- Separate Buffers and AbortControllers for multiple concurrent streams.
- Clean up open Readers on screen exit.
- Prevent duplicate termination for `done`, EOF, and Abort.

### Compatibility

- Verify that `fetch`, `ReadableStream`, `TextDecoder`, and `AbortController` are available in supported browsers.
- Do not call Browser-specific APIs in SSR.

---

## 13. Test Design

### Unit Tests

- When a single Event is split across multiple Chunks
- When a single Chunk contains multiple Events
- When a multi-byte character is split at a Chunk boundary
- Presence or absence of a newline on the last line
- Invalid JSON and unknown Event Types
- Abort state transitions

### Integration Tests

- Full streaming from Browser → Nitro → Backend
- One retry after 401
- Target switching per Proxy environment
- Backend propagation of intermediate connection termination
- Long-running stream Timeout

### Operational Verification

- Nginx/Apache/Load Balancer buffering status
- First Event latency
- Content-Type and Cache-Control per deployment environment
- Memory and connection stability as concurrent connections increase

---

## 14. Completion Criteria

- The browser directly reads `Response.body` and progressively processes Events.
- NDJSON Events are accurately restored regardless of Chunk boundaries.
- The Nitro Proxy does not buffer or reassemble the response.
- User cancellation is propagated from Browser to Backend.
- 401 retry is performed only once before stream consumption.
- Normal completion, error, EOF, and cancellation states are clearly distinguished.
- No Streaming latency occurs in intermediate deployment layers.
