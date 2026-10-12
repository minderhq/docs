# Entitlement token profile (v1)

Status: **v0 draft**. Decided in the MindHub ADR §2a and §2c (F2).

- Verifier (runtime side, verify-only): `mindhub_contracts.entitlement_token` and `mindhub_contracts.paseto`
- Issuer (MindHub signer only): `mindhub_signer.tokens`
- Golden vectors: [`vectors.json`](vectors.json) (`python -m tools.token_vectors --check`)
- Official PASETO/PASERK vectors this is tested against: `official/` (ISC licence)

## Format

Every document is **PASETO `v4.public`** (Ed25519). A verifier checks the literal prefix `v4.public.` before decoding anything and rejects every other version and purpose. The footer is exactly `{"kid":"<PASERK k4.pid>"}`. The key id is looked up **only** in local trust material and never fetched from anywhere the token names.

Each document type has its own **implicit assertion**, so a signature over one type can never verify as another:

| Document | Signed by | Implicit assertion |
|---|---|---|
| key set | offline **root** key (pinned in the runtime) | `mindhub-keyset-v1` |
| entitlement token | online **signing** key listed in the key set | `mindhub-entitlement-v1` |

All times are ISO 8601 strings with a timezone (PASETO convention).

## Key set (`GET /.well-known/mindhub/keyset`)

```json
{"typ": "mindhub-keyset", "iss": "https://hub.…", "seq": 7,
 "iat": "…", "exp": "…",
 "keys": [{"pid": "k4.pid.…", "public": "k4.public.…", "status": "active",
           "not_before": "…", "not_after": "…"}]}
```

| `status` | Meaning for verifiers |
|---|---|
| `next` | published ahead of use; tokens signed with it are **rejected** until a newer key set marks it `active` |
| `active` | current signing key |
| `retired` | verify-only, until its last token expires |
| `revoked` | compromised; nothing it signed is trusted |

Verification rules:
1. The `kid` must be a pinned root, and the signature must verify.
2. `typ` must match.
3. `exp` must be in the future.
4. `seq ≥` the highest `seq` already accepted (**rollback protection**). The runtime stores it durably.
5. Every `pid` must equal `k4.pid(public)`, and no `pid` may repeat.

The runtime pins the current root **and the next root**, so even a root rotation needs no emergency release.

## Entitlement token (`POST /v1/installations/{id}/entitlements:refresh`)

```json
{"iss": "https://hub.…", "sub": "inst_…", "aud": "minder-runtime",
 "iat": "…", "nbf": "…", "exp": "…", "jti": "…",
 "customer_id": "cus_…", "organizations": ["…"],
 "grants": [{"product_id": "bundle.devsecops",
             "provider_ids": ["plugin.github"],
             "versions": ">=3.0.0 <4.0.0",
             "constraints": {"seats": 25},
             "source": {"type": "subscription", "ref": "sub_…"},
             "valid_until": "…"}],
 "policy": {"refresh_after": "…", "grace_seconds": 0, "on_expiry": "deny", "offline": true}}
```

Verification rules:
1. Signing key: listed in the verified key set, with status `active` or `retired`.
2. Token issued (`iat`) within that key's `[not_before, not_after]`.
3. `iss` equals the configured issuer and the key set's `iss`. `aud` is `minder-runtime`. `sub` is **this** installation.
4. `iat` and `nbf` are no later than now plus skew, `exp` is later than now minus skew, and `nbf < exp`. Skew defaults to 5 minutes.
5. Vocabulary is strict:
   - `product_id` and `provider_ids` must be valid product ids
   - `versions` must be a valid range
   - `source.type` is one of `subscription`, `purchase`, `trial`, `free`, `enterprise`, `admin`
   - `on_expiry` is `deny` in v1

   An unknown value is a rejection, and a new value means a new contract major.
6. Tokens are at most 64 KiB.

**What is not in the token on purpose:**
- **No durations or plan names are hard-coded anywhere.** `exp`, `refresh_after`, `grace_seconds` and `offline` are set per product or plan by MindHub (ADR F6c, no decided numbers).
- What happens after `exp` (grace, then deny) is the runtime's state machine, driven by `policy`.
- Capabilities are never gated; `provider_ids` are what the runtime gates.

## Lifecycle

| State | Where it's decided | What the runtime sees |
|---|---|---|
| issued / active | entitlement service | a fresh token on each refresh |
| suspended | entitlement (dispute, payment failure) | the grant is missing from the next token; the old token lapses at `exp`; urgent cases appear in `/v1/revocations` |
| expired | commerce or entitlement | no grant; the token's `exp` passes |
| revoked | entitlement or admin | `jti` and installation in `/v1/revocations`; the next token has no grant |

## Keys and rotation

Implemented by the signer:
- Signing keys are non-exportable Transit or KMS keys behind the `Signer` interface (`mindhub_signer.signers`). The in-memory `LocalSigner` refuses to run in production.
- The root key signs key sets only, in an offline ceremony.
- **Rotation:** publish the new key as `next`, flip it to `active` and the old one to `retired`, then drop the old one after the longest token lifetime plus grace.
- **Compromise:** publish `revoked` in a new key set (`seq + 1`), force runtimes to refresh, and reissue the affected tokens.
