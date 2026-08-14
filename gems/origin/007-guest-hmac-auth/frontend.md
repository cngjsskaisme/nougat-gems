# Guest HMAC Authentication — Frontend

## Intent

Keep Guest Session and Signature details out of Application Code by handling them in a shared API Transport Layer.

## Lifecycle

```text
App Start
  ↓ restore short-lived session if valid
No valid session?
  ↓ init
Store in memory
  ↓
Schedule refresh before expiry
```

If refresh persistence is required, decide whether to use Browser Storage or Cookies as part of an explicit Threat Model. Because the Secret is client-accessible, XSS risk must be considered. Real Cookie/Storage Key names do not belong in a public Gem.

## Request Wrapper

Every protected API request passes through a shared wrapper.

```text
ensureSession
→ timestamp
→ nonce
→ canonical request
→ HMAC signature
→ send
```

Frontend and Backend must define Method, Path, and Body normalization rules identically at the byte level.

## Retry

If a 401 is classified as Session expiry and the response body has not been consumed, issue a new Session and rebuild the original request for **at most one retry**.

Because Timestamp and Nonce are part of the Signature, do not retransmit the previous signed request unchanged.

## Boundaries

UI Components never handle Tokens, Secrets, or Signatures directly. Those responsibilities belong to the Session Manager and API Transport.

## Validation

- FE/BE Canonicalization Contract Test
- single-flight concurrent Refresh
- prevent infinite 401 retry loops
- never restore an expired Session
- ensure Secrets never enter logs or analytics events
