# MindHub HTTP Message Signatures profile (RFC 9421), v1

Status: **v0 draft**. The verifier is `mindhub_chassis.http_signatures`, and the golden vectors are in [`vectors.json`](vectors.json) (`python -m tools.http_sign --check`).

Installations sign every request to MindHub with their Ed25519 key. The profile is deliberately narrow. Anything outside it is rejected.

## Request headers

```
Content-Digest: sha-256=:<base64 SHA-256 of the exact body bytes, also for an empty body>:
Signature-Input: sig1=("@method" "@path" "content-digest");created=<unix seconds>;nonce="<16-64 base64url chars>";keyid="<inst_… or fp_…>";alg="ed25519"
Signature: sig1=:<base64 Ed25519 signature>:
```

- **Exactly one** signature, labelled `sig1`.
- Covered components are **exactly** `"@method" "@path" "content-digest"`. If the request has a query string, they are `"@method" "@path" "@query" "content-digest"`.
- Parameters are `created`, `nonce`, `keyid` and `alg="ed25519"`, each exactly once.
- `keyid` is the installation id. Before registration it is the key fingerprint (`fp_` plus the base64url SHA-256 of the raw public key).

## Signature base (RFC 9421 §2.5)

```
"@method": POST
"@path": /v1/installations/device-authorizations
"content-digest": sha-256=:…:
"@signature-params": ("@method" "@path" "content-digest");created=…;nonce="…";keyid="…";alg="ed25519"
```

Lines are joined with `\n` and there is no trailing newline. `@signature-params` is the `Signature-Input` value after `sig1=`, byte for byte. `@path` is the path as sent, so proxies in front of MindHub must not rewrite it. `@query` is `?` followed by the raw query string.

## Verification

1. Parse with the fixed patterns above. Reject a wrong label, components or parameters.
2. `|server time − created| ≤ 300 s` (configurable; at most 900).
3. `Content-Digest` must equal the digest of the received body.
4. Resolve `keyid` to a key. An unknown or revoked key is rejected.
5. Verify the Ed25519 signature over the signature base.
6. Accept `(keyid, nonce)` only once until the window has passed. Only a valid signature consumes a nonce.

Failures return `401` with a stable reason code: `malformed_signature`, `wrong_covered_components`, `wrong_parameters`, `duplicate_parameter`, `signature_expired`, `content_digest_mismatch`, `unknown_key`, `bad_signature` or `replayed`.

**Clock skew.** Installations should sync time (NTP). A request rejected with `signature_expired` tells the client its clock is off.
