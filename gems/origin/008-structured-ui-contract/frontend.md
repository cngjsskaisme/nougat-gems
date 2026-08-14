# Structured UI Contract — Frontend

## Architecture

```text
API Response
  ↓ decrypt if applicable
Runtime Contract Validation
  ↓
Normalize
  ↓
Message / View State
  ↓
Component Registry
  ↓
Renderer

User Event
  ↓
Action Dispatcher
  ↓
Action Registry
  ↓
Payload Validation
  ↓
State Guard
  ↓
Handler
```

## Component Registry

Never execute a Backend string directly as a Component name or Import Path. Render only Components explicitly registered in the Registry.

The Registry selects Components; it does not perform API calls, mutate state, or execute Actions.

## Action Dispatcher

UI Components emit Events only. The Dispatcher verifies that the Action exists, validates its Payload Schema, checks current State and Transition Guards, and only then invokes the Handler.

## State Machine

Workflows with external Side Effects benefit from explicit state transitions.

```text
idle → editing → ready → executing → completed
                          ↘ error
```

Use Guards to block duplicate execution, missing required values, or transitions into a completed state before real success has occurred.

## Fallback

Isolate these failures to a safe Fallback within the affected UI boundary:

- contract version mismatch
- unknown component/action
- props/payload validation failure
- renderer error
- invalid state transition

Fallback UI must not expose Stack Traces, internal Endpoints, Secrets, or raw Prompts.

## Security

- disable Raw HTML by default or sanitize it
- block dangerous URL schemes
- prohibit arbitrary iframe/script execution
- enforce an external-link allowlist
- do not automatically persist sensitive Drafts in permanent Browser Storage
