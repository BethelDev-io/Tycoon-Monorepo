# AUTH JWT Runbook

Operational runbook for Tycoon authentication: JWT access/refresh tokens, refresh
rotation with reuse detection, httpOnly cookie transport (ADR-004), CSRF, and
NEAR wallet challenge/nonce verification. This runbook also documents the
**cookie/header parsing parity** required by issue #1801 so that the
GamesGateway handshake accepts exactly the same credentials as the REST JWT
strategy.

Sources of truth:

- `backend/docs/TOKEN_REFRESH_SECURITY_GUIDE.md`
- `backend/docs/ADR-002-games-realtime-transport.md`
- `frontend/docs/ADR-004-session-tokens-httpOnly-cookies.md`
- `frontend/docs/NEAR_WALLET_TESTNET_CHECKLIST.md`
- `backend/test/auth-token-security.e2e-spec.ts`
- `backend/test/auth.e2e-spec.ts`

## 1. Token model

| Token   | Lifetime | Storage                          | Transport            |
| ------- | -------- | -------------------------------- | -------------------- |
| Access  | 15m      | httpOnly Secure SameSite cookie  | `Set-Cookie`         |
| Refresh | 30d      | httpOnly Secure SameSite cookie  | `Set-Cookie`         |

- Access tokens are **never** exposed to JavaScript. No `localStorage`,
  `sessionStorage`, or JS-readable cookies (ADR-004).
- Cookies are `httpOnly`, `Secure`, `SameSite=Lax` (or `Strict` for admin
  surfaces), and scoped to the API origin with `Path=/`.
- The server is the source of truth for money, dice, inventory, and admin
  mutations. A valid cookie is required for every authenticated write.

## 2. Token sources (parity contract)

The REST JWT strategy and the WebSocket handshake MUST resolve the bearer token
from the same ordered list of sources. Any divergence is a bug.

Resolution order (first match wins):

1. `Authorization: Bearer <jwt>` header.
2. `access_token` cookie (httpOnly, per ADR-004).
3. `token` cookie (legacy alias, kept for parity with older clients).

Rules:

- The `Authorization` header takes precedence over cookies when both are present.
- Cookie values are URL-decoded before verification.
- Empty, whitespace-only, or malformed values are treated as *absent* and fall
  through to the next source.
- The resolved token is verified with the **same** `JwtService` / strategy
  instance used by REST. Do not duplicate verification logic in the gateway.

## 3. Refresh rotation

Every refresh call rotates the refresh token:

1. Validate the presented refresh token signature, expiry, and `jti`.
2. Look up the refresh family (`familyId`) and the token record.
3. If the token is **unused and unrevoked**, mark it used, issue a new access +
   refresh pair in the same family, and return both as httpOnly cookies.
4. If the token was **already used or revoked**, treat it as reuse (see §4).

Rotation is atomic: the used-mark and the new-token insert happen in a single
transaction so concurrent duplicate requests cannot both succeed.

## 4. Reuse detection

Reuse of a rotated refresh token means the token was stolen or replayed.

- On detection, **revoke the entire refresh family** (`familyId`), not just the
  presented token. All descendants become invalid immediately.
- Return `401 Unauthorized` and clear both auth cookies.
- Emit a security event with the `familyId` and `userId` only. Never log the
  token value, `jti`, or any PII.
- Fail closed: if the token store (Postgres/Redis) is unavailable, reject the
  refresh rather than issuing new tokens.

## 5. CSRF strategy

Cookie-authenticated mutations require CSRF protection:

- Double-submit token: a non-httpOnly `csrf` cookie paired with an
  `X-CSRF-Token` header that must match.
- `SameSite=Lax`/`Strict` cookies as defense in depth.
- Reject state-changing requests (POST/PUT/PATCH/DELETE) with a missing or
  mismatched CSRF token using `403 Forbidden`.
- Safe methods (GET/HEAD/OPTIONS) are exempt.

## 6. NEAR wallet challenge / nonce

- Issue a single-use, time-boxed challenge nonce per login attempt.
- Verify the NEAR signature with domain separation and bind the `account_id`
  into the signed payload.
- Throttle challenge issuance per IP and per account to prevent enumeration.
- Reject replayed nonces; a nonce is consumed on first successful verify.
- If the user rejects the signature, no session is created and the nonce is
  discarded.

## 7. Redirects

- `returnTo` values are validated against an allowlist of known origins/paths.
- Reject open-redirect attempts (absolute URLs, protocol-relative `//`, and
  encoded variants) with `400 Bad Request`.

## 8. WebSocket handshake

Browsers cannot set arbitrary headers on `WebSocket`, so the cookie path is the
primary transport for browser clients. Native/CLI clients may use the
`Authorization` header.

- The gateway extracts the token during the handshake (before `connection`),
  parsing the same httpOnly auth cookies as REST.
- On success, the socket is bound to the authenticated principal and its role
  (`seat` or `spectator`).
- On failure, the handshake is rejected before upgrade. **Deny-by-default**: an
  unauthenticated socket is never admitted and never receives broadcasts.

### Stable error codes

| Code | Meaning |
| --- | --- |
| `AUTH_MISSING_TOKEN` | No token found in any source. |
| `AUTH_INVALID_TOKEN` | Token present but failed verification. |
| `AUTH_EXPIRED_TOKEN` | Token verified but is past `exp`. |
| `AUTH_FORBIDDEN_ROLE` | Authenticated but role not permitted for the action. |

These codes are part of the client contract and must remain stable.

## 9. Authorization: seat vs spectator

- `seat` principals may submit game intents (e.g. `roll`).
- `spectator` principals may observe only. Any mutating intent is rejected with
  `AUTH_FORBIDDEN_ROLE` and dropped server-side.
- The server is the source of truth for outcomes; clients submit intents, never
  results.
- Hidden information (e.g. unrevealed cards) is never broadcast to spectators.

## 10. Token expiry mid-session

- Expiry is evaluated on every inbound action, not only at handshake.
- On expiry the socket receives `AUTH_EXPIRED_TOKEN` and is disconnected.
- Clients must re-authenticate and reconnect; reconnect resumes from a snapshot
  or replays events (see ADR-002).

## 11. Multi-instance delivery

- Use the Redis adapter so broadcasts reach sockets on all instances.
- If sticky sessions are required for a given deployment, document it in the
  deployment notes; otherwise the adapter handles fan-out.
- Redis pub/sub lag is tolerated by ordering events with monotonic sequence
  numbers; clients discard out-of-order or duplicate events.

## 12. Idempotency & reconnect

- Every mutating intent carries an idempotency key. Duplicate keys (reconnect
  retries, duplicate tabs) are de-duplicated server-side.
- Reconnect restores playability via snapshot resume or event replay.

## 13. Failure modes

| Condition                         | Behavior                                  |
| --------------------------------- | ----------------------------------------- |
| Refresh reuse detected            | Revoke family, `401`, clear cookies       |
| Parallel refresh (same token)     | One succeeds, others treated as reuse     |
| Token store outage                | Fail closed, `503`, no new tokens         |
| Auth expiry mid-flow              | `401`, client re-authenticates            |
| Forbidden role                    | `403`, no data leak                       |
| Oversized / adversarial payload   | `413`/`400`, request rejected             |

## 14. Rollback

- Rotation and reuse detection can be disabled behind a feature flag if a
  regression is found; disabling reverts to single-token refresh without
  family revocation.
- Rollback must not re-enable JS-readable tokens; ADR-004 cookie transport
  stays in place.

## 15. Security checklist

- [ ] No secrets or tokens in logs; redact `Authorization` and cookie values.
- [ ] No PII in telemetry labels.
- [ ] Rate-limit `join` and `roll`.
- [ ] Metrics for connected sockets and rejected actions.
- [ ] Fail-closed on dependency outage (Postgres/Redis/shop-api/RPC) for all
      writes.
- [ ] Reuse detection revokes the refresh family.
- [ ] `returnTo` redirects are allowlisted only.

## 16. Mobile NEAR wallet bottom-sheet checklist automation notes

Automation notes for the Mobile NEAR wallet bottom-sheet flow (issue #1814).
These notes are the operational companion to
`frontend/docs/NEAR_WALLET_TESTNET_CHECKLIST.md` and describe how the
bottom-sheet checklist is exercised in CI and what the automation must assert.

### 16.1 Bottom-sheet flow

1. User taps **Connect NEAR wallet** on mobile; the bottom-sheet opens.
2. The sheet requests a challenge nonce from the API (§6) and renders the
   checklist steps (wallet detected, account selected, signature requested,
   signature verified, session established).
3. On success the sheet closes and the session cookie is set (§1).
4. On user rejection the sheet shows a non-blocking error and no session is
   created; the nonce is discarded (§6).

### 16.2 Automation assertions

- Challenge issuance is throttled per IP and per account; a burst of requests
  returns `429` and never leaks whether an account exists.
- The signed payload is domain-separated and binds `account_id`; a signature
  over a different domain or account fails verification.
- A replayed nonce is rejected and the nonce is consumed on first successful
  verify.
- The user-reject path creates no session and leaves no auth cookies set.
- The bottom-sheet checklist steps map 1:1 to the assertions above so a failing
  step names the exact check that failed.

### 16.3 Test coverage

- `auth-token-security.e2e` — cookie transport, rotation, reuse detection.
- `auth.e2e` — challenge/nonce issuance, verify, replay rejection.
- Unit signature-verify negatives — wrong domain, wrong `account_id`, tampered
  payload.
- Frontend RTL wallet-reject path — bottom-sheet shows error, no session.

### 16.4 Failure modes specific to the bottom-sheet

| Condition                | Behavior                                          |
| ------------------------ | ------------------------------------------------- |
| User rejects sign        | No session, nonce discarded, sheet shows error    |
| Replayed nonce           | `401`, nonce already consumed                     |
| Parallel refresh         | One succeeds, others treated as reuse (§4)        |
| Open redirect attempt    | `400`, `returnTo` not allowlisted (§7)            |
| Challenge burst          | `429`, throttled per IP and account (§6)          |
