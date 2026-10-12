# Verification API (v1)

Status: **v0 draft**. OpenAPI: [`openapi.json`](openapi.json), checked against the code on every test run. Decided in the MindHub ADR §2b.

This is the only MindHub API a Minder installation calls after it is linked (linking itself is [`installation/PROTOCOL.md`](../installation/PROTOCOL.md)). It is a deliberately small, stable surface, never MindHub's internal structures.

| Endpoint | Auth | Purpose |
|---|---|---|
| `POST /v1/installations/{id}/entitlements:refresh` | RFC 9421, installation key | a fresh [entitlement token](../token/PROFILE.md) for this installation |
| `GET /v1/installations/{id}/revocations` | RFC 9421, installation key (also when **revoked**) | the jtis revoked before their `exp`, and whether the installation is revoked |
| `GET /.well-known/mindhub/keyset` | public | the latest root-signed key set (`text/plain`, a `v4.public` token) |
| `PUT /v1/internal/keyset` | service auth, `admin` | upload a key set from the root ceremony; checked against the pinned roots, and `seq` must grow |

Usage reporting (`POST /v1/installations/{id}/usage`) arrives with usage-based products.

## Runtime client contract

1. Cache durably:
   - the latest verified token
   - the key set and the highest key-set `seq`
   - the revocation list
2. Refresh in the background at `policy.refresh_after`, and never on a request path. Use `GET /.well-known/mindhub/keyset` for the key set. It is root-signed, so it can come from any mirror.
3. On `503`, keep the cached token. Its `exp` and `grace_seconds` (from the token) decide what happens next.
4. Poll `revocations` on the same cadence. Drop any cached token whose `jti` is listed. If `installation_revoked` is set, stop using MindHub grants.
5. Every request is signed per [`http-signatures/PROFILE.md`](../http-signatures/PROFILE.md). Fix your clock if you get `signature_expired`.

**Token contents.** These come from entitlement at refresh time:
- top-level grants become token grants, and a bundle's components become its `provider_ids`;
- only grants in `active` or `grace` that apply to this customer, this organization or this installation are included (personal *user* grants never are);
- `exp` is 7 days and `refresh_after` is 24 h (ADR F3, held as settings, overridable per plan later);
- grace and the offline flag are the strictest across the token's grants.

## Threat model

| Threat | Mitigation |
|---|---|
| A stolen token used on another installation | `sub` is bound to the installation; runtimes verify `sub`. The token alone also can't call this API, because that needs the installation key |
| Replayed refresh or revocation requests | RFC 9421 `created` window plus single-use nonces |
| A forged key set (compromised web server, CDN, mirror) | Key sets are verified against roots pinned in the runtime, and `seq` only grows. The upload endpoint checks the same before publishing |
| Rollback to a key set that still trusts a revoked key | Runtimes refuse a `seq` lower than the highest they have accepted, and the upload refuses a `seq` that isn't new |
| A revocation missed by the runtime | The feed returns the **whole** current list, not a cursor, so an out-of-order commit can't hide an entry. Tokens are short-lived anyway (`exp` 7 days) |
| A suspended or refunded entitlement still usable | Unexpired tokens for that customer and product go onto the revocation list immediately (event-driven), and the next token omits the grant |
| A revoked installation | Its key stops authenticating refresh; it can still read `revocations` to learn its status |
| A compromised signing key | Revoke it in a new key set (signer RUNBOOK) |
| The signer issuing a malformed token | The signer verifies each token with the runtime's own profile code before returning it, so it never hands out what a runtime would reject |
| Upstream outage (entitlement, signer) | `503` "keep the cached token"; decisions continue from the cache and policy |
| Enumeration of installations or customers | Unknown, unsigned and wrong-key requests all get the same `401` |
| Abuse or volume | Rate limiting at the edge, with per-installation budgets |
| Audit | Each issuance emits `verification.token_issued` (jti, installation, products, kid, exp; never the token), consumed by audit |
