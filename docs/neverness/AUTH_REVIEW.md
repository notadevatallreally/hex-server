# Neverness authentication and session review

Review pass started: 2026-09-16

Baseline: `neverness` branch at upstream `ee0f06b` plus the two known CT103-only local fixes. Those fixes do not touch the authentication paths reviewed here.

## Confirmed critical finding — HConnect trusts client-supplied identity

**ID:** NCR-007  
**Severity:** critical  
**Area:** `hconnect_server.py`, `profile_db.py`, `proxy.py`

The initial concern was that `proxy.py` trusts the `steamId` URL parameter when `STEAM_WEB_API_KEY` is not configured. End-to-end tracing shows a broader problem: HConnect itself does not authenticate the token or username it receives.

### HConnect auth flow

For an HConnect packet with `instance == "auth:req"`, the server:

1. Decodes the JSON body.
2. Reads `user` directly from the request.
3. Reads `token` directly from the request.
4. If the token is a string beginning with `steam:`, strips the prefix and treats the remainder as a Steam ID.
5. Calls `db_get_or_create_user(username, steam_id=steam_id)`.
6. Assigns the returned profile to `self.user_profile`.
7. Derives the connection's client identity from that profile.

No password, MAC/signature, nonce, server-side session lookup, shared secret, Steam validation, or callback to the HTTP auth proxy occurs in this HConnect path.

Relevant logic is effectively:

```python
auth_data = json.loads(body.decode("utf-8"))
username = auth_data.get("user", "TestPlayer")
token = auth_data.get("token") or ""
steam_id = None
if isinstance(token, str) and token.startswith("steam:"):
    steam_id = token[len("steam:"):]
profile = db_get_or_create_user(username, steam_id=steam_id)
self.user_profile = profile
self._set_client_identity_from_profile(profile)
```

### Database lookup behavior

`profile_db.db_get_or_create_user()` derives an ID from the supplied Steam ID when present, otherwise from the supplied username. It first queries `users.id`, then falls back to an exact `users.name` lookup:

```python
uid = (player_id_from_steam(steam_id)
       if steam_id else player_id_from_name(name))
...
SELECT ... FROM users WHERE id=?
...
SELECT ... FROM users WHERE name=?
```

Therefore, a client that can reach HConnect can select an existing account by supplying either:

- the account's Steam ID in a plaintext `steam:<id>` token, or
- the account's exact stored username even without a valid Steam-backed token.

The HTTP proxy's password or Steam-ticket verification does not protect this direct HConnect path because HConnect never verifies that its `auth:req` values originated from the proxy.

### Scope

This is exploitable whenever the HConnect service is reachable by an untrusted client. It is not dependent on `STEAM_WEB_API_KEY` being absent; even a correctly validated HTTP `/steam/login` response does not make the later HConnect token trustworthy because the token is just replayable/forgeable plaintext and has no server-side validation.

### Consequences

Once HConnect selects the profile, the connection is given that profile's persistent identity and subsequent profile/game operations run in that account context. This can expose or modify persistent account state through the normal authenticated service surface.

The repair should not be designed piecemeal. The complete session/auth protocol, reconnect behavior, expected client token format, and compatibility requirements should be mapped before choosing whether to use signed short-lived tokens, opaque server-side sessions, another validation mechanism, or a restricted network boundary.

## Proxy Steam validation

`proxy.py` has two modes:

- With `STEAM_WEB_API_KEY` and a Steam ticket, it validates the ticket through Steam and treats the returned Steam ID as authoritative.
- Without the API key, it explicitly trusts the URL's `steamId` parameter.

The latter remains a security problem, but NCR-007 supersedes it in severity because bypassing the proxy entirely is sufficient.

## Non-Steam HTTP login

`/auth/hexlogin` does verify the stored password hash when a password exists and rejects unknown accounts. That check is meaningful only on the HTTP endpoint; the resulting HConnect identity is not cryptographically bound to the successful login.

`/auth/hexregister` performs account creation and stores a password hash when provided.

`/auth/hexchangepass` verifies the old password when one is stored before changing it.

## Transition/TOTP routes

`/auth/hextransition` and `/auth/hextotp` derive a UID from the supplied username, create/update the auth record, and return `steam:<uid>` without performing the password checks used by `/auth/hexlogin`.

These routes require further protocol-context review to determine whether they were intended to be callable only after some other trusted transition. As implemented at the HTTP layer, no such prior state is visible in the handler.

Because NCR-007 already permits direct HConnect identity selection, these routes do not currently represent the only authentication bypass, but they should still be corrected as part of the eventual auth redesign if they are retained.

## Generic auth fallback

A broad proxy branch matching other paths containing `steam` or `auth` returns success with the constant token `test_token_abc123`. This needs protocol inventory before removal because some legacy client probes may rely on a generic success response. It should not be treated as proof of authentication.

## Admin / moderator / founder flags

The Steam login URL accepts `Admin`, `Mod`, and `Founder` query parameters and `db_set_user_flags()` persists them into `users.flags`.

Current repository-wide searches at the reviewed revision find the meaningful `admin`, `mod`, and `founder` flag names being written in `proxy.py` but do not identify a corresponding authorization consumer. No privilege-escalation claim is therefore made at this point.

**Current classification:** suspicious/insecure input surface, apparently inert in this revision. Re-check whenever command/admin features are added or another consumer is found.

## Remaining authentication/session review

Still to trace before closing this pass:

- exact auth success response and profile-stream transition after `self.user_profile` binding
- connection/session ID creation and lifetime
- reconnect/resume behavior and whether any identity is revalidated
- active-client registries and duplicate simultaneous logins
- tournament `player_handlers` ownership and stale-handler cleanup
- whether service requests enforce that referenced player/session IDs belong to `self.user_profile`
- logout/disconnect cleanup
- token/session replay behavior across server restart
- any network topology assumptions that were intended to make HConnect private behind the proxy

No authentication fix has been applied during this review.
