# Frontend AI Agent Prompt

You are a Frontend Security Engineer.

Implement a Frontend authentication layer that uses the Backend's Guest Authentication system.

The goal is to allow users to use APIs without being aware of the Guest Session.

---

## Guest Session

Backend

```
POST /api/guest/init
```

Response

```json
{
  "guestToken": "...",
  "secret": "...",
  "expiresIn": 300
}
```

---

## App Startup

On app startup, automatically execute:

```
ensureGuestSession()
```

Order

1.

Check Cookie

Guest Session exists

If not expired, restore

2.

If not found

```
POST /guest/init
```

3.

Store response

* Memory
* Cookie

4.

Schedule auto-renewal

---

## Cookie

Example name

```
nougat_guest_session_v1
```

Stored content

```json
{
  "token": "...",
  "secret": "...",
  "expiresAt": 123456789
}
```

Options

* SameSite=Lax
* Secure (HTTPS)

---

## Auto-Renewal

Guest Session expiration

30 seconds before

automatically call

```
POST /guest/init
```

Replace with a new Session.

---

## API Wrapper

All API requests

use a common Wrapper.

Example

```
apiFetch()
```

Automatically performs

* Guest Session check
* Authorization generation
* Timestamp generation
* Nonce generation
* Signature generation

Header

```
Authorization: Guest {token}

X-Timestamp

X-Nonce

X-Signature
```

---

## Signature

Same rules as Backend

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

## 401 Handling

On 401 occurrence

Reissue Guest Session

```
POST /guest/init
```

Perform

On success

Original request

retry only once.

Infinite retries are prohibited.

---

## Session Management

Use Memory first.

On page refresh

restore from Cookie.

Memory and Cookie are always synchronized.

---

## Implementation Structure

Recommended structure

```
plugins/

guest-init.client.ts

composables/

useGuestSession.ts

useApi.ts

utils/

signature.ts

cookie.ts
```

---

## Implementation Quality

* TypeScript Strict
* Composition API
* API Wrapper reuse
* Separated Signature Utility
* Modularized Session management
* Testable structure
* 100% identical Signature rules with Backend
