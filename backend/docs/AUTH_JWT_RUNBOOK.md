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
- [ ] Fail-closed on dependency outage (Postgres/Redis/shop-api/RPC) for writes.
- [ ] Deny-by-default for new WS/action surfaces.

## 16. End-to-end testnet proof (NEAR login → create/join → finish → claim)

This section is the runbook for issue #1809: proving the full player journey on
NEAR testnet end to end. It ties the auth flows above to the checklist in
`frontend/docs/NEAR_WALLET_TESTNET_CHECKLIST.md`.

### 16.1 Journey stages

1. **NEAR login** — challenge/nonce issued (§6), signature verified with domain
   separation and `account_id` binding, session established as httpOnly cookies
   (§1). No JS-readable access token is ever produced.
2. **Create / join** — cookie-authenticated mutation (§5 CSRF) with an
   idempotency key (§12). Duplicate create/join retries must resolve to the same
   game, never a second one.
3. **Finish** — server is the source of truth for the outcome; clients submit
   intents only (§9).
4. **Claim** — cookie-authenticated mutation (§5), authorized for the claiming
   principal, idempotent so a retried claim cannot double-pay.

### 16.2 Proof requirements

- The journey is exercised by `auth.e2e-spec.ts` and
  `auth-token-security.e2e-spec.ts`, plus the frontend RTL wallet-reject path.
- Forged account sessions must be impossible: a signature that does not bind the
  claimed `account_id`, or that fails domain separation, is rejected.
- Replayed nonces and parallel refreshes are rejected per §4 and §6.
- `returnTo` redirects are allowlisted per §7; open-redirect attempts fail.

### 16.3 Failure modes specific to the journey

| Condition                     | Behavior                                        |
| ----------------------------- | ----------------------------------------------- |
| User rejects NEAR signature   | No session; nonce discarded; no cookies set     |
| Replayed nonce                | `401`, nonce already consumed                   |
| Parallel refresh              | One succeeds, others treated as reuse (§4)      |
| Open redirect `returnTo`      | `400`, redirect refused                         |
| Dependency outage on claim    | Fail closed, `503`, no state change             |

### 16.4 Acceptance criteria

- [ ] Forged account sessions impossible.
- [ ] Cookie/CSRF story complete.
- [ ] `NEAR_WALLET_TESTNET_CHECKLIST.md` updated.
- [ ] e2e green.
