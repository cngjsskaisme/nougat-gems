
Now let's look at the frontend side.
Let's check the frontend (client) side.

Let's check the frontend part in `nougat-playground-fe`.




Now we have the overall picture. Here is a summary of the analysis results.

---

## `/api/guest/init` Logic Analysis

### 1. API Server (nougat-playground / api-server)

#### Controller — `@/srv/nougat-playground/apps/api-server/src/security/guest-auth.controller.ts:5-14`

```ts
@Controller('guest')
export class GuestAuthController {
  @Public()
  @Post('init')
  async initGuest() {
    return this.guestAuthService.initGuest();
  }
}
```

- Path: `POST /api/guest/init` (NestJS global prefix `/api` + `guest` + `init`)
- Due to the `@Public()` decorator, it **bypasses** [SecurityAuthGuard](cci:2://file:///srv/nougat-playground/apps/api-server/src/security/security-auth.guard.ts:20:0-241:1) authentication.

#### Service — `@/srv/nougat-playground/apps/api-server/src/security/guest-auth.service.ts:10-24`

```ts
async initGuest() {
  const guestToken = randomUUID();
  const secret = randomBytes(32).toString('base64');
  const key = `guest:${guestToken}`;
  await redis.set(key, JSON.stringify({ secret }), 'EX', GUEST_TOKEN_TTL_SECONDS);
  return { guestToken, secret, expiresIn: GUEST_TOKEN_TTL_SECONDS };
}
```

- **guestToken**: Generated with `randomUUID()` (for client identification)
- **secret**: `randomBytes(32)` base64 encoded (secret key for HMAC signing)
- Stored in Redis with key `guest:{token}` as `{ secret }`, **TTL 300 seconds** (5 minutes)
- Response: `{ guestToken, secret, expiresIn: 300 }`

#### Rate Limiting — `@/srv/nougat-playground/apps/api-server/src/security/rate-limit.middleware.ts:25-28`

```ts
if (normalizedPath === '/api/guest/init') {
  await this.enforceLimit([ratelimit:${this.getClientIp(request)}](cci:1://file:///srv/nougat-playground/apps/api-server/src/security/security-auth.guard.ts:86:2-98:3), GUEST_INIT_RATE_LIMIT);
  return next();
}
```

- **5 times per minute per IP** (`GUEST_INIT_RATE_LIMIT = 5`) limit
- On exceed: `429 Too Many Requests`

#### Authentication Guard (for subsequent API calls) — `@/srv/nougat-playground/apps/api-server/src/security/security-auth.guard.ts:30-73`

All `/api/` requests other than `/api/guest/init` go through [SecurityAuthGuard](cci:2://file:///srv/nougat-playground/apps/api-server/src/security/security-auth.guard.ts:20:0-241:1):

1. Extract token from `Authorization: Guest {token}` header
2. Look up secret in Redis using `guest:{token}`
3. Verify `X-Timestamp` header (±30 seconds allowed)
4. Verify `X-Nonce` + `X-Signature`
   - **Signature**: `HMAC-SHA256(secret, method + canonicalPath + normalizedBody + timestamp + nonce)`
   - **nonce**: Redis `SET NX` to prevent replay attack (30 seconds TTL)
5. Compare signature with `timingSafeEqual` (prevent timing attack)

---

### 2. Frontend (nougat-playground-fe)

#### Plugin — `@/srv/nougat-playground-fe/plugins/guest-init.client.ts:1-7`

```ts
export default defineNuxtPlugin(async () => {
  await ensureGuestSession().catch(() => {
    // First authenticated request will retry initialization.
  })
})
```

- Client-side Nuxt plugin, automatically executed on page load

#### Composable — `@/srv/nougat-playground-fe/composables/useApi.ts`

**Session initialization flow** ([ensureGuestSession](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:338:0-347:1), [fetchGuestSession](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:311:0-336:1)):

1. **Cookie restore** ([restoreGuestSessionFromCookieOnce](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:85:0-112:1)): Restore `{ token, secret, expiresAtMs }` from `nougat_guest_session_v1` cookie (valid only until 5 seconds before expiration)
2. **Session check**: Skip if token/secret exist in memory and within 30 seconds before expiration
3. **POST `/guest/init`** ([fetchGuestSession](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:311:0-336:1)): POST request with empty body, receive `{ guestToken, secret, expiresIn }` from response
4. **Apply session** ([applyGuestSession](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:285:0-309:1)):
   - Store token/secret in memory variables
   - Calculate `expiresAtMs = Date.now() + expiresIn * 1000`
   - Store session in cookie (SameSite=Lax, Secure conditional)
   - Schedule auto-renewal timer 30 seconds before expiration ([scheduleRefresh](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:276:0-283:1))

**Authentication header generation for API requests** ([getClientAuthHeaders](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:358:0-422:1)):

```ts
return {
  'Authorization': `Guest ${guestToken}`,
  'X-Timestamp': timestamp,      // Date.now()
  'X-Nonce': nonce,              // randomUUID or fallback
  'X-Signature': signature,      // HMAC-SHA256(secret, method+path+body+timestamp+nonce)
}
```

**401 retry logic** ([executeFetch](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:432:0-489:1)):

- On 401 response from client → call [refreshGuestSession()](cci:1://file:///srv/nougat-playground-fe/composables/useApi.ts:349:0-356:1) then **retry once** (`retried` flag prevents infinite loop)
- `/guest/init` path requests without auth headers via `skipAuth = true`

---

### 3. Overall Sequence Summary

```
[Browser load]
  └─ guest-init.client.ts plugin
       └─ ensureGuestSession()
            ├─ Try cookie restore → skip if valid
            └─ POST /api/guest/init (no auth, 5 req/min per IP limit)
                 └─ Response: { guestToken, secret, expiresIn: 300 }
                      └─ Store in memory + cookie, schedule renewal timer

[Subsequent API calls]
  └─ apiFetch() / executeFetch()
       └─ getClientAuthHeaders(method, path, body)
            └─ Generate HMAC-SHA256 signature
                 └─ Authorization: Guest {token}
                    X-Timestamp, X-Nonce, X-Signature
       └─ On 401 → refresh session and retry once
```

**Key features**:
- **Stateless guest authentication**: Redis-based short-term session + HMAC signing instead of JWT
- **Replay attack prevention**: nonce + timestamp window (±30 seconds)
- **Auto-renewal**: Timer 30 seconds before expiration + on-demand renewal on 401
- **Cookie persistence**: Restore session from cookie on page reload to prevent duplicate init
