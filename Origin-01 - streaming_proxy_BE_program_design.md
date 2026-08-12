# Streaming Proxy BE Program Design Document

## 1. Document Purpose

This document defines the server-side design for the chat backend to progressively generate LLM and internal processing results as NDJSON Events and reliably deliver them to the browser through a frontend Reverse Proxy.

The backend does not wait for the entire response to be generated; it flushes Events as soon as they are ready and propagates client connection termination as cancellation of internal tasks.

---

## 2. Design Scope

### In Scope

- Streaming HTTP Controller
- NDJSON Response Header
- Event Contract
- LLM Chunk reception and conversion
- Semantic/Event Parser
- NDJSON serialization and flush
- Backpressure handling
- Client Disconnect and Abort propagation
- Normal completion and error Events
- Proxy-friendly response policy
- Observability and operational Timeout

### Out of Scope

- Browser `ReadableStream` parsing
- Nitro `routeRules` configuration details
- Chat UI rendering
- Encryption and decryption
- Search and RAG detailed logic
- LLM Prompt content

---

## 3. Target Structure

```text
HTTP Streaming Controller
    ↓
Request Validator
    ↓
Chat Service
    ↓
Chat Orchestrator / LLM Gateway
    ↓ token or semantic chunk
Section/Event Parser
    ↓ structured event
Event Sequencer
    ↓
NDJSON Serializer
    ↓
Stream Writer / Flush
    ↓
Nitro Reverse Proxy
    ↓
Browser
```

The cancellation path propagates in the reverse direction.

```text
Browser Disconnect
    ↓
Nitro Upstream Connection Close
    ↓
HTTP Controller Abort Signal
    ↓
Chat Service
    ↓
LLM Gateway Cancellation
```

---

## 4. Key Components

### 4.1 Streaming HTTP Controller

Handles the HTTP lifecycle of the Streaming Endpoint.

**Responsibilities**

- Request validation
- Streaming Response Header configuration
- Abort Signal creation and connection
- Subscription to the Chat Service's Event Stream
- NDJSON Write and Flush
- Connection termination and resource cleanup

The Controller does not directly perform business logic such as search, Prompt assembly, or LLM provider-specific processing.

### 4.2 Request Validator

Validates required values of the request before starting streaming.

```text
conversationId(optional)
message
mode(optional)
client capabilities(optional)
requestId
```

Since it is difficult to switch to a normal JSON error response after sending the response Header, validation is completed before the stream starts whenever possible.

### 4.3 Chat Service Stream Interface

Provides a structured Event Stream to the Controller.

The recommended abstract form is as follows.

```text
streamChat(request, abortSignal) → Async Event Stream
```

The Chat Service does not directly handle the HTTP Response object. This separates Streaming logic from business logic and ensures testability.

### 4.4 LLM Gateway

Abstracts the provider's Streaming API into a common Chunk Stream.

```text
Provider Token/Chunk
    ↓
Normalized LLM Chunk
```

Connects the Abort Signal to the provider SDK or HTTP request cancellation functionality.

### 4.5 Section/Event Parser

Transforms the LLM's Raw Text or Chunks into semantic Events that the frontend can understand.

```text
Raw Token Stream
    ↓
Incremental Section Parser
    ↓
Semantic Block
    ↓
Event
```

Representative Events are as follows.

```text
meta
text
block/dealCards
block/comparison
block/questions
done
error
```

Rather than using the LLM Raw Text directly as the UI contract, the Parser generates a stable Event Contract.

### 4.6 Event Sequencer

Assigns a request-level sequence number to each Event.

```text
requestId
sequence
 type
payload
```

The sequence number can be used by the FE to detect duplicates, out-of-order, and missing Events.

### 4.7 NDJSON Serializer

Serializes each Event as a single line of JSON.

```text
serialize(event) + "\n"
```

Even if there are line breaks within the Event Payload, JSON string escaping ensures that one Event remains physically one line.

### 4.8 Stream Writer

Writes serialized Events to the response stream and flushes as soon as possible.

**Responsibilities**

- Check writable state
- Wait for Backpressure
- Detect Write failures
- Flush
- Prevent duplicate termination
- Trigger Abort on connection termination

### 4.9 Abort Coordinator

Consolidates the following cancellation causes into a single Abort Signal.

```text
client disconnect
user cancellation
gateway timeout
server shutdown
internal policy cancellation
```

When an Abort occurs, search, DB queries, LLM calls, Parser, and post-processing are all halted to the extent possible.

---

## 5. HTTP Response Design

### Recommended Response Headers

```text
Content-Type: application/x-ndjson; charset=utf-8
Cache-Control: no-store, no-transform
X-Content-Type-Options: nosniff
```

Additional Streaming and buffering prevention Headers may be added depending on the actual framework and infrastructure.

### Response Status

- Request validation failures are returned as 4xx before the stream starts.
- Errors that occur after the stream has started are delivered as `error` Events.
- In the case of a critical connection error, no further Events can be written, so the connection is terminated.

### Content-Length

Not set because the total size cannot be known in advance.

---

## 6. Event Contract

All Events use a common Envelope.

```json
{
  "version": "1",
  "requestId": "request-id",
  "sequence": 1,
  "type": "text",
  "payload": {},
  "timestamp": "ISO-8601"
}
```

### Event Types

| Type | Purpose |
|---|---|
| `meta` | Initial metadata such as conversation ID, model, processing mode |
| `text` | Incremental text fragments |
| `block/*` | Structured UI blocks |
| `done` | Normal completion and final metadata |
| `error` | Safe error information occurring after stream start |

### Completion Rules

A normal stream sends exactly one `done` Event at the end.

```text
0..N data events
    ↓
1 done event
    ↓
stream close
```

If an error Event has been sent, the subsequent policy is fixed to one of the following.

```text
error → stream close
```

The default principle is not to continue sending normal Events after an error.

---

## 7. Normal Processing Sequence

```text
Browser
    ↓ POST streaming request
Nitro Proxy
    ↓
Streaming Controller
    ↓ validate request
Chat Service
    ↓
LLM Gateway
    ↓ chunk
Event Parser
    ↓ event
Event Sequencer
    ↓
NDJSON Serializer
    ↓ line
Stream Writer
    ↓ flush
Nitro Proxy
    ↓
Browser
    ⋮
Stream Writer
    ↓ done event
Browser
```

The first Event can be a `meta` Event containing the conversation identifier and stream ready state.

---

## 8. Error Handling Sequence

### Error Before Stream Start

```text
Request
    ↓
Validation Failure
    ↓
HTTP 4xx JSON Error
```

### Business Error After Stream Start

```text
Streaming Started
    ↓
Service / LLM Error
    ↓
Sanitized error Event
    ↓
Flush
    ↓
Stream Close
```

### Connection Termination Error

```text
Write Failure or Client Disconnect
    ↓
Abort Signal
    ↓
Cancel LLM and Internal Tasks
    ↓
Release Resources
```

If the connection has already been terminated, no additional error Event is written.

---

## 9. Backpressure Design

The client or Proxy may consume data at a slower rate than the server generates it.

**Handling Principles**

- Check the write result of the Stream Writer.
- If the internal Buffer exceeds a threshold, pause the generation of the next Event.
- If the LLM provider does not support pausing, use a bounded Queue.
- If the Queue limit is exceeded, safely terminate the stream and cancel related tasks.
- Unbounded memory Buffers are not allowed.

Backpressure considers both per-request memory usage and the total number of concurrent connections across the server.

---

## 10. Abort and Resource Cleanup

### Cancellation Propagation Targets

```text
LLM streaming request
search request
DB query where supported
RAG retrieval
summary generation
parser loop
pending stream write
```

### Cleanup Items

- Pending Message state cleanup
- Release open Reader/Writer
- Remove Timers
- Remove temporary Buffers
- Remove request-level Context
- Prevent duplicate Finalize

Client cancellation is not saved as a normal completion. However, whether to save partial responses is determined by a separate conversation storage policy.

---

## 11. Proxy Compatibility Design

The backend complies with the following so that intermediate Proxies can pass the stream through as-is.

- Use NDJSON Content-Type
- Include a newline for each Event
- Flush immediately after Event generation
- Do not aggregate the entire response into an object and return it at once
- No caching
- Minimize latency caused by transformation and compression
- Avoid long periods with no output or apply a Heartbeat policy

### Heartbeat

If there may be long periods without Events during LLM or search processing, a Heartbeat Event can be used optionally.

```text
heartbeat
```

Heartbeats are separated from UI data and are used solely to prevent Proxy Idle Timeout. The interval is set shorter than the infrastructure Timeout.

---

## 12. Timeout Policy

Distinguishable Timeouts are as follows.

```text
request validation timeout
first-event timeout
idle timeout
overall stream timeout
upstream LLM timeout
write timeout
```

When a Timeout occurs, all related tasks are halted through the Abort Coordinator.

The `overall stream timeout` is set considering user experience and the maximum LLM processing time, and is kept shorter than the Timeout of the Proxy and Load Balancer.

---

## 13. Observability

The following are measured per request.

```text
requestId
conversationId(optional)
stream start time
first event time
first text time
event count
byte count
last sequence
LLM duration
stream duration
completion status
abort source
error category
```

Event Payload raw text and user prompts are excluded from default logs.

### Key Metrics

- Time to First Event
- Time to First Text
- Normal completion rate
- Client Abort ratio
- Proxy/Network Disconnect ratio
- LLM error rate
- Event parsing error rate
- Maximum Buffer size per request
- Number of concurrent Streaming connections

---

## 14. Non-Functional Requirements

### Scalability

- The Controller is kept stateless.
- Per-request state is maintained only for the duration of the connection.
- File Descriptor, Thread/Event Loop, and memory limits are adjusted according to the number of concurrent connections.

### Reliability

- Terminal state is managed as a single entity to prevent duplicate calls to `done`, `error`, and close.
- One request's error is isolated so it does not affect other request streams.
- On server shutdown, new connections are blocked and existing streams are cleaned up within a limited time.

### Consistency

- The Event Contract is versioned.
- Event ordering is guaranteed.
- Structured Block Events are schema-validated before being sent.

---

## 15. Test Design

### Unit Tests

- Verify that Event serialization maintains single-line NDJSON
- Event Sequence increment
- Single normal `done` transmission
- No additional Events sent after error
- Internal task cancellation on Abort
- Backpressure Queue limit

### Integration Tests

- Full flow from LLM Chunk → Parser → NDJSON → Client
- LLM cancellation propagation on Client Disconnect
- Immediate Chunk delivery through Nitro Proxy
- Long periods with no output and Heartbeat
- Upstream LLM Timeout
- Multiple concurrent streams

### Failure Tests

- Forced Proxy connection termination
- Slow client
- LLM mid-stream error
- JSON serialization failure
- Open streams during server shutdown
- Queue Overflow

### Operational Verification

- Proxy Buffering disabled
- Load Balancer Idle Timeout
- First Event and Chunk Flush latency
- Memory usage as concurrent connections increase

---

## 16. Completion Criteria

- NDJSON Streaming Headers are set after request validation.
- LLM or internal Events are flushed line by line as soon as they are generated.
- Event Contract and Sequence are consistent across all responses.
- A normal stream terminates with exactly one `done` Event.
- Errors after stream start terminate after a safe `error` Event.
- Client Disconnect is propagated to LLM and internal task cancellation.
- No unbounded Buffer is created for slow clients.
- Real-time Chunk delivery is maintained even through Nitro and external Proxies.
