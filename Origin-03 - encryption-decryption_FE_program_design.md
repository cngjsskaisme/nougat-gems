# Encryption/Decryption FE Program Design Document

## 1. Document Purpose

This document defines an **application-layer hybrid encryption structure** to protect user prompts and server responses in the chat frontend.

The encryption protection scope is as follows.

```text
Browser ↔ Chat Backend
```

The browser generates a symmetric key to encrypt the actual data, and the symmetric key is encrypted with the server's public key before transmission. This structure does not replace transport-layer HTTPS and is premised on being applied together with HTTPS.

---

## 2. Design Scope

### Included Scope

- Server public key retrieval
- Per-request AES key generation
- RSA public key encryption of the AES key
- AES encryption of prompt and inquiry data
- Encrypted request Envelope construction
- Encrypted response decryption
- Lifecycle management of keys and plaintext data
- Encryption/decryption failure and retry handling

### Excluded Scope

- Chat UI composition
- Conversation state machine
- LLM Prompt design
- User authentication and authorization policies
- Server-side LLM invocation methods
- Streaming response handling

---

## 3. Target Structure

```text
User Input
    ↓
Plain Request Model
    ↓
Encryption Coordinator
    ├─ Public Key Provider
    ├─ AES Key Generator
    ├─ Payload Encryptor
    └─ Request Envelope Builder
    ↓
API Client
    ↓
Encrypted Response Envelope
    ↓
Response Decryptor
    ↓
Plain Response Model
    ↓
Application State
```

The frontend separates encryption implementation details from the UI and chat state management layers. The upper layers handle only plaintext request models and plaintext response models, converting to encrypted Envelopes only at the network boundary.

---

## 4. Key Components

### 4.1 Public Key Provider

Retrieves the server's public key, converts it to a usable format, and provides it to the encryption layer.

**Responsibilities**

- Public key API invocation
- Public key format and algorithm identifier validation
- Expiration information or key identifier management
- In-memory cache application
- Cache invalidation on key rotation detection

**Input**

```text
Public Key retrieval request
```

**Output**

```text
keyId
algorithm
publicKey
expiresAt(optional)
```

The public key is not secret information and can be provided to the browser. However, HTTPS, same-origin policy, and response integrity verification must be used together to prevent incorrect public keys from being injected.

### 4.2 AES Key Generator

Generates a one-time symmetric key and IV or Nonce for each encryption request.

**Design Principles**

- Use the browser's cryptographically secure random number generator.
- Generate a new AES key for each request.
- Do not reuse IV or Nonce.
- Do not write keys to persistent storage.
- Remove references after the encryption request is completed.

### 4.3 Payload Encryptor

Serializes the plaintext request object and encrypts it with AES.

**Processing Order**

```text
Plain Request
    ↓
Canonical Serialization
    ↓
UTF-8 Encoding
    ↓
AES Encryption
    ↓
Ciphertext + IV/Nonce + Authentication Tag
```

The principle is to use authenticated encryption modes to provide both data confidentiality and tamper detection simultaneously.

### 4.4 Key Wrapper

Encrypts the AES key for the request with the server's RSA public key.

```text
AES Key
    ↓
RSA Public-Key Encryption
    ↓
Encrypted AES Key
```

RSA is used only for symmetric key delivery, not for encrypting large messages.

### 4.5 Request Envelope Builder

Assembles the encryption results to conform to the network transmission contract.

The recommended logical structure is as follows.

```json
{
  "version": "1",
  "keyId": "server-key-id",
  "keyAlgorithm": "RSA",
  "contentAlgorithm": "AES",
  "encryptedKey": "base64...",
  "iv": "base64...",
  "ciphertext": "base64...",
  "tag": "base64..."
}
```

Field names and encoding methods are managed through the same version contract between FE and BE.

### 4.6 Response Decryptor

Decrypts the response that the server encrypted and returned using the same session key.

**Processing Order**

```text
Encrypted Response Envelope
    ↓
Schema Validation
    ↓
Base64 Decode
    ↓
AES Decryption and Integrity Check
    ↓
UTF-8 Decode
    ↓
JSON Parse
    ↓
Plain Response Model
```

Before decryption, the version, algorithm, required fields, and data size must always be validated.

### 4.7 Encryption Coordinator

A single entry point that coordinates the entire encryption flow.

The upper API Client uses only the following abstract interface.

```text
encryptRequest(plainRequest)
decryptResponse(encryptedResponse, sessionContext)
```

The UI and chat state layers must not be aware of encryption details such as RSA, AES, IV, and Base64.

---

## 5. Request Processing Sequence

```text
Chat State
    ↓ plain request
Encryption Coordinator
    ↓
Public Key Provider ── GET /public-key ──→ Backend
    ↓ public key
AES Key Generator
    ↓ session key + IV
Payload Encryptor
    ↓ ciphertext
Key Wrapper
    ↓ encrypted AES key
Envelope Builder
    ↓ encrypted request
API Client ── POST /chat ──→ Backend
```

If a valid public key exists in the cache, the public key retrieval step can be skipped.

---

## 6. Response Processing Sequence

```text
Backend
    ↓ encrypted response
API Client
    ↓
Envelope Validator
    ↓
Response Decryptor
    ↓ plain response
Response Normalizer
    ↓
Chat State / UI
```

Only data that has been successfully decrypted is reflected in the application state. Responses that fail decryption or integrity verification are not partially used.

---

## 7. State and Key Lifecycle Management

The frontend temporarily maintains the following session context per request.

```text
requestId
keyId
AES key
request IV/Nonce
response decryption metadata
createdAt
```

### Lifecycle Principles

- AES keys are kept in memory only until the corresponding request and response decryption are complete.
- AES keys are not stored in LocalStorage, SessionStorage, or IndexedDB.
- Plaintext prompts and decrypted responses are not recorded in debug logs.
- The session context is discarded on request cancellation, timeout, or error.
- Keys are not recovered after a browser refresh.

---

## 8. Error Handling Policy

| Error Type | FE Handling Principle |
|---|---|
| Public key retrieval failure | Abort request transmission and classify as a retryable error |
| Unsupported key version | Clear public key cache and re-fetch once |
| AES key generation failure | Do not transmit request and return a security error |
| Encryption failure | Do not send plaintext as a fallback |
| Network failure | Determine whether to resend the same ciphertext based on request idempotency policy |
| Server key mismatch response | Re-fetch the latest public key and re-encrypt the entire request with a new AES key |
| Response decryption failure | Discard response, display generalized error in UI |
| Integrity verification failure | Record as security error and prohibit data usage |

Fallback that automatically switches to plaintext requests on encryption failure is not permitted.

---

## 9. API Contract

### 9.1 Public Key Retrieval

```text
GET /api/public-key
```

**Response Logical Model**

```text
version
keyId
algorithm
publicKey
expiresAt(optional)
```

### 9.2 Encrypted Chat Request

```text
POST /api/chat
Content-Type: application/json
```

**Request Logical Model**

```text
version
requestId
keyId
encryptedKey
iv/nonce
ciphertext
tag
```

**Response Logical Model**

```text
version
requestId
iv/nonce
ciphertext
tag
```

The `requestId` of the request and response are linked to prevent encrypted responses from other requests from being incorrectly applied.

---

## 10. Security Design Principles

- HTTPS must be used mandatorily.
- Browser standard encryption APIs are used.
- Authenticated encryption modes are used.
- RSA uses verified padding schemes and is used only for wrapping AES keys.
- Per-request AES keys and IV/Nonce are not reused.
- Algorithm and key version are explicitly specified in the Envelope.
- Plaintext and keys are not included in logs, error tracking tools, or analytics events.
- When encryption functionality fails, it is handled in a Fail Closed manner.
- Encryption cannot protect plaintext from XSS, so CSP, input validation, and dependency security are required separately.

---

## 11. Non-Functional Requirements

### Performance

- The public key is cached in memory during its validity period.
- Encryption/decryption is performed asynchronously to minimize UI thread blocking.
- Large requests have their size limited in advance.

### Compatibility

- The availability of encryption APIs in supported browsers is checked during the initialization phase.
- In browsers that do not support encryption, the feature is disabled and plaintext transmission is not used as a substitute.

### Observability

Only the following metadata can be recorded.

```text
requestId
keyId
algorithm version
encryption duration
decryption duration
error category
```

Plaintext, symmetric keys, and full ciphertext are not recorded in logs as a principle.

---

## 12. Test Design

### Unit Tests

- Public key response validation
- Whether a new AES key and IV are generated for each request
- Serialization and deserialization consistency
- Normal encryption/decryption
- Failure on ciphertext, tag, or IV tampering
- Invalid key version handling

### Integration Tests

- Full flow from public key retrieval to response decryption
- Retry immediately after server key rotation
- Network interruption and request cancellation
- Key context separation when processing multiple requests simultaneously

### Security Tests

- Plaintext log exposure checks
- Confirmation that keys do not remain in storage
- Ciphertext resend and request identifier collision verification
- Verification that encryption alone is not incorrectly assumed to provide protection under XSS conditions

---

## 13. Completion Criteria

- The UI and state layers do not directly depend on the encryption implementation.
- All chat requests are transmitted as encrypted Envelopes.
- All chat responses are decrypted after integrity verification.
- On key mismatch, a secure retry is performed using the latest public key.
- Plaintext fallback is not performed on errors.
- Plaintext and symmetric keys do not remain in browser persistent storage or operational logs.
