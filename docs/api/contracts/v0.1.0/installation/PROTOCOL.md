# Installation identity and linking protocol (v1)

Status: **v0 draft**, implemented in `identity`. Decided in the MindHub ADR §2b and §4a.

Each Minder installation (self-hosted, or a Minder Cloud tenant runtime) has a **stable installation identity**: an `inst_…` id bound to keys that the installation generated itself. The private keys never leave it. MindHub stores only public keys. SaaS gets no privileged path. Minder Cloud is just a large installation.

## Keys

| Key | Algorithm | Used for |
|---|---|---|
| signing key | Ed25519 | every request to MindHub ([RFC 9421 profile](../http-signatures/PROFILE.md)) |
| encryption key | X25519 | HPKE (RFC 9180) wrapping of paid-KB content keys to this installation |

Keys travel as base64url (no padding) of the 32 raw bytes. A key's public name before registration is its **fingerprint**: `fp_` plus the base64url SHA-256 of the raw signing key. The Minder console shows it, and MindHub's approval page shows it too.

## Linking: device authorization (RFC 8628)

```
Installation                              MindHub                      Customer admin
 | generate Ed25519 + X25519 keys           |                                  |
 | POST /v1/installations/device-authorizations   (signed, keyid = fp_…)       |
 |----------------------------------------->|                                  |
 |<-- device_code, user_code, fingerprint --|                                  |
 | shows user_code + fingerprint            |                                  |
 |                                          |<-- GET /v1/device-authorizations/{user_code}
 |                                          |--- label, version, fingerprint ->|
 |                                          |<-- POST …/{user_code}/approve {customer, organization_ref}
 | POST /v1/installations/device-token {device_code}   (signed, same key)      |
 |----------------------------------------->|  creates inst_… now             |
 |<-- installation_id, customer, org -------|                                  |
```

- `device_code`: 256-bit secret; MindHub stores only its SHA-256. `user_code`: 8 characters from RFC 8628's 20-consonant alphabet, shown as `XXXX-XXXX`. Both expire after `device_code_minutes` (default 10).
- Poll errors follow RFC 8628 §3.5: `authorization_pending`, `slow_down` (polled faster than `interval`), `access_denied`, `expired_token`, `invalid_grant` (unknown or already used).
- Approving needs the `manage_installations` permission on the chosen customer. Denying needs only a logged-in account that knows the code.
- `organization_ref` is the Minder organization on that installation. One installation can be linked once per flow. Linking a second organization means running a second flow with a new key.

## Using the identity

Every later request is signed with `keyid = inst_…`:

| Endpoint | Purpose |
|---|---|
| `GET /v1/installations/{id}` | current status and key version; also answers for **revoked** installations, so they can find out |
| `POST /v1/installations/{id}/keys:rotate` | rotate both keys (below) |
| verification API | entitlement refresh, revocations, usage |

**Rotation.** The request is signed by the **current** key. The body carries the new keys and `proof`, the **new** key's signature over:

```
mindhub-key-rotation-v1\n{installation_id}\n{new signing key}\n{new encryption key}
```

So both keys are proven. The old key stops working at once, `key_version` increases, and the old key is retired for ever.

**Revocation (decommissioning).** A customer admin calls `DELETE /v1/customers/{customer}/installations/{id}`. The key stops authenticating, and tokens already issued lapse at `exp` (ADR §2: short-lived tokens, plus the revocation list).

**Moving to another customer.** Revoke, then link again with a new key. Identity never moves silently.

Every change emits `identity.installation_changed` (public keys only), and verification keeps its key projection from it.

## Threat model

| Threat | Mitigation | Residual risk |
|---|---|---|
| **Stolen device code** (logs, screen) | Polling must be signed by the key the code was issued to; MindHub stores only the code's hash; single use; 10-minute life | none known |
| **Stolen or guessed user code** | Codes are short-lived; each account gets `max_user_code_failures` wrong guesses per window, then 429, even for a correct code; approving needs `manage_installations` on the target customer; edge rate limiting | an admin can be **phished** into approving an attacker's code. The approval page shows label, Minder version and the key **fingerprint**, which the admin compares with their own console. Notifications will email every new link |
| **Replay** of a captured request | RFC 9421 `created` within ±`signature_window_seconds` (default 300 s), plus single-use `(keyid, nonce)` stored until the window passes; the signature covers method, path, query and body digest | none within the profile |
| **Registering someone else's public key** | `start` must be signed by the key being registered (proof of possession) | none |
| **Cloned installation** (disk image, backup restore, copied VM) | A key can't be registered twice, even after retirement. Rotating on one copy locks the other out, which is visible to the customer. `last_seen_at` is recorded | a perfect clone that never rotates is indistinguishable by design, because there is no hardware fingerprinting (ADR §4b: VM restores must keep working). Licence limits count installations, not copies. Detection signals (concurrent use from disjoint networks, nonce-pattern anomalies) are a follow-up |
| **Compromised installation key** | The customer revokes it; rotation is self-service; tokens are short-lived | requests until revocation |
| **Compromised customer admin account** | Sessions are revocable, a password change ends all sessions, lockout applies; two-factor authentication is planned | the attacker can link installations until detected; audit events |
| **Downgrade or algorithm confusion** | One fixed algorithm (`ed25519`), a fixed component set, fixed-pattern parsing; anything else is rejected | none |
| **Privacy** | MindHub learns label, Minder version, organization ref and last-seen time. No hardware identifiers, no telemetry beyond that | `label` is free text the admin chooses |

**Out of scope here:** device or instance attestation (TPM, cloud instance identity). It is optional, per deployment, and worth its cost only for high-assurance enterprise plans. It would add an attestation document to `start` without changing anything above.
