# Guest HMAC Authentication — Backend

## Intent

Issue short-lived Guest Sessions and validate a Timestamp, Nonce, and HMAC Signature on every protected request.

## Session Init

Protect the public Init Endpoint with an IP/client-based Rate Limit so sessions cannot be created without bound.

```text
Generate guestToken
Generate random session secret
Store {secret} with short TTL
Return token + secret + expiresIn
```

The actual Endpoint name, Cache Key Prefix, TTL, and Rate Limit are environment-specific configuration rather than part of the Gem contract.

## Signed Request

```text
Authorization: Guest <token>
X-Timestamp: <timestamp>
X-Nonce: <nonce>
X-Signature: <signature>
```

Frontend and Backend must share the same Canonicalization Contract for the signing input.

```text
METHOD
+ CanonicalPath
+ NormalizedBody
+ Timestamp
+ Nonce
```

## Verification Order

1. Validate the Guest Token format.
2. Load the Session Secret.
3. Validate the Timestamp Window.
4. Consume the Nonce atomically using a `SET-if-absent` equivalent.
5. Recompute the Signature.
6. Compare in constant time.

A Nonce must be usable only once during its short TTL.

## Responsibilities

- Controller: HTTP contract
- Auth Guard/Middleware: validation order and rejection
- Signature Service: canonicalization + HMAC
- Session Repository: TTL / atomic nonce storage
- Rate Limiter: protect the public Init Endpoint

Concrete class names and project file structures are not part of the Gem contract.

## Validation

- block replay of the same request/Nonce
- reject requests outside the Timestamp Window
- Signature fails when Body or Path changes
- use timing-safe comparison
- reject requests after Session expiry
- enforce Rate Limit on the Init Endpoint
