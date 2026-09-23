# NEAR wallet — testnet manual checklist

Prerequisites: `NEXT_PUBLIC_NEAR_NETWORK=testnet` (default). Optional: `NEXT_PUBLIC_NEAR_CONTRACT_ID` if your app uses a contract other than the default testnet sign-in contract.

## End-to-end testnet proof: NEAR login → create/join → finish → claim

Run this flow on testnet before every release that touches auth, matchmaking, or the
Soroban/NEAR boundary. It is the acceptance path for issue #1809.

1. **Login (challenge/nonce)** — Open the app, click **Connect NEAR**, pick MyNearWallet
   (or another enabled wallet), complete sign-in. The backend must issue a single-use
   challenge/nonce bound to the requested `account_id`; the wallet signs the
   domain-separated message. Expect the header to show a truncated account id.
   - Reject the signature once: expect a toast with *transaction was not signed; connect
     and approve to continue* and **no** session cookie set.
   - Replay the same signed nonce: expect rejection (nonce already consumed) and no session.
2. **Session cookie** — After a successful login, confirm the session is carried by an
   `httpOnly`, `Secure`, `SameSite` cookie (ADR-004). No access token may be readable from
   JS (`document.cookie` must not expose it).
3. **Account** — Hover or long-press the account pill; full account id should appear in the
   tooltip (`title`).
4. **Create / join** — Create a game, then join it from a second wallet/session. Expect the
   server to be the source of truth for the game state; duplicate create/join requests
   (double-click, reconnect retry) must be idempotent and not create extra games.
5. **Finish** — Play the game to completion. Expect the finish transition to be authorized
   for the participating accounts only; a non-participant or expired session must be
   rejected (fail-closed on writes).
6. **Claim** — Claim the reward. Expect the claim to be authorized, idempotent, and to
   reject replayed claims. Confirm the balance/inventory reflects the claim exactly once.
7. **Refresh / rotation** — Let the session expire mid-flow and refresh. Expect a silent
   rotation via the refresh cookie; a replayed refresh token must revoke the refresh family
   and force re-login (TOKEN_REFRESH_SECURITY_GUIDE).
8. **CSRF & redirects** — Cookie-authenticated mutations must require the CSRF token; a
   mutation without it must be rejected. A `returnTo`/redirect pointing off the allowlist
   must be refused (no open redirect).
9. **Disconnect** — Click **Disconnect NEAR**; account pill disappears, session cookie is
   cleared, and **Connect NEAR** returns.
10. **Pending → confirmed** — Submit a valid contract call. Expect **Transaction pending…**,
    then **Confirmed**, and a **View on explorer** link.
11. **Explorer link** — Open the link; NEAR Explorer should show the transaction hash on testnet.
12. **Mobile** — Open the bottom menu; NEAR block should appear at the bottom of the sheet
    with **Connect NEAR** / account + disconnect.

### Failure modes to verify

- Dependency outage (Postgres/Redis/shop-api/RPC): writes fail closed, no partial state.
- Forbidden role access: admin/WS/action surfaces deny by default.
- Oversized or adversarial payloads: rejected without leaking tokens or PII in logs.
