# Backend AI Agent Prompt

You are a Backend Security Architect.

Implement a Guest Authentication system for use in services without login functionality.

This system is for Guest Client Authentication, not User Authentication.

JWT is not used.

It uses Redis-based short-term Guest Sessions and HMAC Signatures to protect API requests.

---

## Goals

Defend against the following attacks.

* API Abuse
* Replay Attack
* Request Tampering
* Authorization Header Forgery
* Long-term Secret Theft

---

## Guest Session Issuance API

Public API

POST /api/guest/init

Accessible without authentication.

On call, it generates:

* guestToken (UUID)
* secret (32byte random)

Response

```json
{
  "guestToken": "...",
  "secret": "...",
  "expiresIn": 300
}
```

Redis Storage

Key

```
guest:{guestToken}
```

Value

```json
{
  "secret": "..."
}
```

TTL

```
300 seconds
```

---

## Rate Limit

/api/guest/init

Based on IP

5 times per minute

On exceed

```
429 Too Many Requests
```

---

## Authentication Target

/api/guest/init

Excluded

All APIs require Guest authentication.

Authorization Header

```
Authorization: Guest {guestToken}
```

---

## Request Header

All requests must include:

* Authorization
* X-Timestamp
* X-Nonce
* X-Signature

---

## Signature

Signature generation algorithm

```
HMAC-SHA256(secret)
```

Signing String

```
METHOD
+
CanonicalPath
+
NormalizedBody
+
Timestamp
+
Nonce
```

---

## Authentication Procedure

In Guard or Middleware,

perform the following in order.

1.

Check Authorization

2.

Redis

```
guest:{token}
```

Lookup

If not found

401

3.

Timestamp

Allowed range

±30 seconds

4.

Nonce

Redis

```
nonce:{token}:{nonce}
```

SET NX

TTL 30 seconds

If already exists

401

5.

Recalculate Signature

6.

timingSafeEqual()

Compare

If different

401

---

## Replay Attack

Nonce

can only be used once.

Timestamp

only allows within ±30 seconds.

---

## Security

Must use

* randomUUID()
* randomBytes(32)
* crypto.createHmac()
* crypto.timingSafeEqual()
* Redis TTL
* Redis SET NX

---

## Architecture

The following structure is recommended.

* GuestAuthController
* GuestAuthService
* SecurityAuthGuard
* RateLimitMiddleware
* SignatureService
* RedisRepository

Controller only handles requests.

Authentication logic in Guard

Signature calculation in Service

Redis access in Repository

separated accordingly.

---

## Implementation Quality

* SOLID
* TypeScript Strict
* Unit Testable
* DI-based structure
* Reusable Signature logic
* Consistent error responses
* Separated Canonical Path/Body generation functions
