# Encryption/Decryption BE Program Design Document

## 1. Document Purpose

This document defines the server-side design for the chat backend to decrypt encrypted requests delivered from the browser and to encrypt the processing results before returning them.

The protected segment is as follows.

```text
Browser ↔ Chat Backend
```

The backend manages RSA key pairs and decrypts the per-request AES key, which the browser wrapped with the RSA public key, using the private key. The actual prompt and response data are encrypted and decrypted using AES.

---

## 2. Design Scope

### Included Scope

- RSA key pair generation or secure loading
- Public key provision API
- Key identifier and key version management
- Encrypted request Envelope validation
- RSA-based AES key decryption
- AES-based request data decryption
- Internal service delivery of plaintext requests
- AES encryption of processing results
- Error response and audit log policies

### Excluded Scope

- Frontend UI and state management
- LLM Prompt content
- Chat business rules
- External delivery features such as Telegram
- Streaming responses
- User account authentication policies

---

## 3. Target Structure

```text
HTTP Router
    ↓
Encrypted Request Validator
    ↓
Key Resolver
    ↓
RSA Key Unwrapper
    ↓
AES Request Decryptor
    ↓
Plain Chat Request
    ↓
Chat / LLM Service
    ↓
Plain Chat Response
    ↓
AES Response Encryptor
    ↓
Encrypted Response Builder
    ↓
HTTP Response
```

The encryption/decryption layer is placed as an independent security boundary between HTTP routing and chat business logic. The chat service is unaware of the encryption Envelope and handles only plaintext domain models.

---

## 4. Key Components

### 4.1 Key Manager

Manages RSA key pairs and key metadata.

**Responsibilities**

- Key loading or generation at server startup
- Active key and previous key management
- Key lookup based on `keyId`
- Public key serialization
- Key rotation and disposal
- Private key access control

**Recommended Operational Principles**

- Separate temporary key generation in development environments from persistent key management in production environments.
- Do not include production private keys directly in source code or general environment variables.
- In multi-instance environments, all instances should use the same active key set.
- During key rotation, keep the previous private key together for a short grace period.

Generating new keys on every server startup may cause in-flight requests that used the existing public key to fail upon restart, so a key storage and rotation strategy is required in production environments.

### 4.2 Public Key Controller

Provides the public key and metadata that clients will use to encrypt requests.

```text
GET /public-key
```

**Response Logical Model**

```text
version
keyId
algorithm
publicKey
expiresAt(optional)
```

The public key response can be cached, but Cache-Control appropriate to the key rotation cycle and expiration policy should be applied.

### 4.3 Encrypted Request Validator

Validates the structure and size of the encryption Envelope.

**Validation Items**

- Whether the contract version is supported
- `keyId` presence and format
- Whether the RSA/AES algorithm is supported
- Presence of `encryptedKey`, `iv/nonce`, `ciphertext`, `tag`
- Base64 or agreed encoding validity
- Allowed maximum request size
- `requestId` format and duplication

Requests that fail validation are rejected immediately without attempting decryption.

### 4.4 RSA Key Unwrapper

Decrypts the encrypted AES key in the request Envelope using the private key.

```text
Encrypted AES Key
    ↓
Resolve Private Key by keyId
    ↓
RSA Private-Key Decryption
    ↓
AES Session Key
```

Detailed causes of decryption failure are not exposed in external responses.

### 4.5 AES Request Decryptor

Decrypts the request body using the AES key and IV/Nonce, and verifies integrity.

```text
Ciphertext + IV/Nonce + Tag
    ↓
AES Decryption and Integrity Verification
    ↓
UTF-8 Plaintext
    ↓
JSON Parse
    ↓
Plain Request Model
```

If integrity verification fails, plaintext data is not generated or passed to the business layer.

### 4.6 Secure Request Context

A security context used only for a single request.

```text
requestId
keyId
AES session key
request cryptographic metadata
createdAt
```

This context is immediately discarded upon request processing completion, cancellation, or error. AES keys are not stored in DB, cache, or message queues.

### 4.7 AES Response Encryptor

Encrypts the business processing result with the AES key recovered from the request.

```text
Plain Response
    ↓
Canonical Serialization
    ↓
New Response IV/Nonce
    ↓
AES Encryption
    ↓
Ciphertext + Tag
```

A different IV/Nonce from the request is used for the response. Even when using the same key, IV/Nonce must never be reused.

### 4.8 Encrypted Response Builder

Composes the encrypted response in the contract format.

```json
{
  "version": "1",
  "requestId": "request-id",
  "iv": "base64...",
  "ciphertext": "base64...",
  "tag": "base64..."
}
```

The response is linked to the `requestId` of the original request.

---

## 5. Public Key Provision Sequence

```text
Browser
    ↓ GET /public-key
Public Key Controller
    ↓
Key Manager
    ↓ active public key + metadata
Public Key Controller
    ↓
Browser
```

The private key is not included in any response.

---

## 6. Chat Request Processing Sequence

```text
Browser
    ↓ encrypted request
HTTP Router
    ↓
Envelope Validator
    ↓
Key Manager ── resolve by keyId
    ↓
RSA Key Unwrapper
    ↓ AES session key
AES Request Decryptor
    ↓ plain request
Chat / LLM Service
    ↓ plain response
AES Response Encryptor
    ↓ encrypted response
HTTP Router
    ↓
Browser
```

Plaintext domain DTOs are used between the encryption layer and the chat service, but plaintext must not leak through logs or exception messages.

---

## 7. Key Management Policy

### Active Key

The current key exposed to new client requests.

### Previous Key

The key that was exposed just before key rotation, used only for decryption during a limited grace period to process in-flight requests.

### Retired Key

A key whose grace period has ended, securely removed from server memory and key storage.

### Key Rotation Flow

```text
Create New Key Pair
    ↓
Register New keyId
    ↓
Publish New Public Key
    ↓
Keep Previous Private Key Temporarily
    ↓
Expire Previous keyId
    ↓
Securely Remove Previous Private Key
```

Key rotation must consider zero-downtime deployment and multi-instance synchronization.

---

## 8. Error Handling Policy

| Error Type | Server Handling | External Response Principle |
|---|---|---|
| Invalid Envelope | Reject before decryption | Generalized request error |
| Unknown `keyId` | Return status requiring re-fetch of latest public key | Do not expose internal key information |
| RSA decryption failure | Abort request and record security event | Generalized decryption error |
| AES integrity failure | Discard plaintext | Generalized decryption error |
| Plaintext JSON validation failure | Do not pass to business layer | Invalid request |
| LLM processing failure | Generate error response model and encrypt if necessary | Do not expose internal provider information |
| Response encryption failure | Do not return plaintext response | Server error |
| Request cancellation | Remove key context, halt subsequent processing | Connection termination or cancellation response |

Detailed causes of encryption/decryption failure, private key existence, and padding errors are not exposed to attackers.

---

## 9. API Contract

### 9.1 Public Key API

```text
GET /public-key
```

**Success Response**

```text
version
keyId
algorithm
publicKey
expiresAt(optional)
```

### 9.2 Chat API

```text
POST /chat
Content-Type: application/json
```

**Encrypted Request**

```text
version
requestId
keyId
keyAlgorithm
contentAlgorithm
encryptedKey
iv/nonce
ciphertext
tag
```

**Encrypted Success Response**

```text
version
requestId
iv/nonce
ciphertext
tag
```

Error responses also use a separate standard error contract to ensure no sensitive internal information is included.

---

## 10. Security Design Principles

- HTTPS must be used mandatorily.
- Authenticated encryption modes are used.
- RSA private key access is restricted to the encryption/decryption module.
- RSA is used only for AES key decryption.
- Different IV/Nonce values are used for requests and responses.
- Keys and plaintext are not included in operational logs, APM, or traces.
- Request size limits and processing time limits are applied.
- Rate limiting is applied to requests that repeatedly submit invalid ciphertext.
- Decryption details must not be inferable through error messages or processing times.
- If data is delivered to an LLM provider after the encryption layer, that segment is managed under separate security and privacy policies.

This structure protects the communication between the browser and the chat backend. Since the backend processes plaintext, it does not guarantee end-to-end confidentiality for the backend itself and the subsequent LLM call segment.

---

## 11. Observability and Audit Logs

The following items can be recorded.

```text
requestId
keyId
contract version
ciphertext size
decryption duration
encryption duration
result status
error category
client metadata(minimized)
```

The following items must not be recorded.

```text
private key
AES session key
plaintext request
plaintext response
full encrypted key
full ciphertext
```

Security events are separated from general application logs to restrict access.

---

## 12. Non-Functional Requirements

### Availability

- In the event of key storage failure, new requests are safely rejected.
- Multiple instances must be able to use the same key version.
- During key rotation, requests based on the previous public key must be handled in a limited manner.

### Performance

- RSA operations are limited to one AES key decryption per request.
- Actual message processing uses AES.
- Encryption/decryption time and LLM processing time are measured separately.

### Scalability

- Future algorithm replacement is supported through contract versions and algorithm identifiers.
- The encryption/decryption module maintains an interface that allows independent deployment or replacement from the Chat Service.

---

## 13. Test Design

### Unit Tests

- Active key lookup and previous key lookup
- Normal RSA key decryption
- Invalid `keyId` handling
- Normal AES decryption and encryption
- Ciphertext, tag, and IV tamper detection
- Response IV/Nonce reuse prevention

### Integration Tests

- Full flow from public key lookup to encrypted response return
- Previous key request processing during key rotation
- Server restart and multi-instance environments
- Encrypted error response policy when LLM errors occur
- Key context removal on request cancellation

### Security Tests

- Padding Oracle and error message exposure checks
- Limits on oversized ciphertext requests
- Rate limiting for repeated decryption failure requests
- Whether plaintext and keys are exposed in logs and traces
- Key storage permissions and key rotation procedure verification

---

## 14. Completion Criteria

- The roles and lifecycles of public and private keys are separated.
- Decryption is performed only after encryption Envelope validation.
- The Chat Service handles only plaintext domain models and is unaware of the encryption implementation.
- All normal responses are encrypted with AES before being returned.
- Plaintext responses are not returned when response encryption fails.
- Key rotation and multi-instance synchronization are possible in production environments.
- Plaintext and keys do not remain in logs or persistent storage.
