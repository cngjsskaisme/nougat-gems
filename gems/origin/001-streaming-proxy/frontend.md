# Streaming Proxy — Frontend

## Architecture

```text
Application State
    ↓
Streaming API Facade
    ↓
Transport
    ↓ fetch
ReadableStream
    ↓
Incremental UTF-8 Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓
Event Dispatcher
    ↓
Application State / Rendering
```

## Chunk ≠ Event

Network Chunk boundaries do not align with UTF-8 character boundaries or NDJSON line boundaries. Use `TextDecoder` incrementally and preserve the final incomplete line until the next Chunk arrives.

```text
buffer += decodedFragment
completeLines = all lines except last
buffer = last incomplete line
```

Parse only complete lines as JSON.

## Proxy Boundary

The browser may call a generalized same-origin API path while a server-runtime Reverse Proxy forwards the request to the Backend. The Proxy must not reconstruct the Response Body as an object or fully buffer it.

Real internal hosts, backend URLs, and deployment paths are not part of this Gem's contract. They belong in project Runtime Configuration.

## Retry

Retry only when it is safe—for example, an authentication failure before the response body has been consumed. Once some Events have been applied to the UI, a full automatic retry can duplicate state and should be disallowed by default.

## Abort

When the user stops a request, navigates away, or starts a replacement request, use `AbortController` to cancel the Reader and fetch so the connection closure can propagate to the Backend.

## Validation

- one Event split across multiple Chunks
- multiple Events in one Chunk
- a multibyte character split across Chunk boundaries
- final line without a trailing newline
- invalid JSON / unknown event
- distinguish Abort from abnormal EOF
