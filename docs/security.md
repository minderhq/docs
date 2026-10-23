# Security

This page covers Minder's security posture — what protections are in place, what
is deliberately *not* yet in place — and how to manage the credentials and secrets
a deployment needs. For the login/SSO/JWT model in detail, see
[Authentication](authentication.md).

!!! warning "A default install is development-grade"
    Out of the box Minder is functional end-to-end but **not** hardened for
    public-internet exposure. The measures below describe what exists today plus
    the steps to take before a production deployment. Work through the
    [hardening checklist](#production-hardening-checklist) before exposing an
    instance.

## Security posture

### What's in place

- **JWT authentication** at the API Gateway. The gateway signs access tokens
  with **EdDSA** (an Ed25519 key pair that setup generates). Every other
  service gets only the public key, so it can verify a token but never mint
  one. Credentials are stored as bcrypt hashes. See
  [Authentication](authentication.md#jwt-signing-and-keys).
- **Redis-backed rate limiting** at the gateway, in 60-second windows:
  anonymous callers per client IP, signed-in callers per IP and user (a higher
  limit), and the login and register routes per IP only. It is **fail-open**:
  if Redis is unreachable, requests are allowed rather than blocked. The limits
  are `RATE_LIMIT_PER_MINUTE` (60), `RATE_LIMIT_AUTHENTICATED_PER_MINUTE` (300)
  and `RATE_LIMIT_STATUS_POLL_PER_MINUTE` (600, document upload status polls).
- **Service-to-service identity.** A service that calls another service's
  internal routes signs a short-lived token with **its own** Ed25519 key, and
  each internal route accepts only the callers on its allowlist. There is no
  shared service token. A service whose keys are missing logs an error at
  startup and its service calls are refused (`401`), so run setup after an
  upgrade.
- **Security response headers, on by default.** The gateway adds
  `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`,
  `X-Frame-Options: DENY` with `Content-Security-Policy: frame-ancestors 'none'`
  (the anonymous public-chat paths are the exception: their `frame-ancestors`
  is the endpoint's own allowed origins), and `Cache-Control: no-store` on
  auth and invite-token responses. It adds `Strict-Transport-Security` only
  when the request arrived over TLS, directly or through a trusted proxy. A
  header a route already set is never replaced, and error and `429`
  responses carry the headers too. The web client's own server adds its
  headers (including a Content-Security-Policy) in the same way. Settings, all
  in the root `.env`: `SECURITY_HEADERS_ENABLED` (default `true`),
  `SECURITY_HEADERS_REFERRER_POLICY`, `SECURITY_HEADERS_HSTS`,
  `CLIENT_SECURITY_HEADERS` (`on` by default; set `off` only when your own
  reverse proxy sets them) and `CLIENT_CSP`.
- **Authelia SSO / OIDC** — enabled and enforcing forward-auth on the gated
  routers, and a real OIDC identity provider that mints the Minder JWT from a
  verified Authelia identity. See [Authentication](authentication.md).
- **Traefik v3 reverse proxy** — the single ingress; TLS termination and routing
  by Docker labels (`exposedByDefault: false`, so nothing is exposed unless
  explicitly labelled). Not Nginx.
- **Network isolation** — storage backends (PostgreSQL, Redis, Qdrant, Neo4j,
  MinIO, RabbitMQ, schema-registry) run internal-only on the Docker network and
  publish no host port. Host ports that do exist are **loopback-bound**
  (`127.0.0.1`) — see [Service access](service-access.md).
- **IP-whitelist middleware** on the admin surfaces routed through Traefik (the
  dashboard, the message-queue management UI, the graph-database browser).
- **Least-privilege Docker access** — services that need the Docker Engine API
  (bundle enable/disable, container logs) go through a proxy with an explicit
  allowlist of paths and methods, never a raw `docker.sock` mount.
- **Out-of-process plugins.** Plugin code runs in separate `plugin-runner`
  containers, not in the plugin registry, with no platform secrets in their
  environment. Each plugin that needs a database gets its own least-privilege
  Postgres role. See [Plugin trust model](#plugin-trust-model).

### Plugin trust model

Plugins run **out of process**. The plugin registry keeps every policy decision
(who may call what, admin-only actions, tier checks, scheduling, config,
secrets) and calls a **plugin-runner** container only after those checks. The
registry image contains no plugin code: it parses a read-only copy of the
catalog and never imports it.

Runners are grouped by how much they are trusted:

| Group | Hosts | How it reaches the network |
|-------|-------|----------------------------|
| `declarative` | Declarative (T0) manifest plugins only. No plugin code runs. | Direct, but only to the public hosts each manifest declares. The engine checks every hop, refuses any non-public address and pins the connection to the address it checked. |
| `trusted` | Catalog code plugins that only need public HTTP(S). | Through an egress proxy only. The runner sits on internal Docker networks with no route out. |
| `privileged` | Plugins that need LAN access or manage a sidecar (today `network` and `telegraf`). | Direct LAN and internet access under firewall rules in its own network namespace. These block cloud metadata and link-local addresses, the Docker host's networks and gateways, and anything listed in `EGRESS_PROXY_DENY_CIDRS`. |

Every runner runs with a read-only root filesystem, no Linux capabilities,
`no-new-privileges`, and **no** `env_file` or platform secret in its
environment. Its API listens only on an internal control network, and it
accepts only requests carrying a short-lived, single-use token that
plugin-registry signs for that specific group. A runner that has no
registry public key refuses every request.

A plugin gets only what the platform grants it, through the platform context
(`self.ctx`): its own secrets, calls to plugins it declares a dependency on,
document indexing on behalf of the invoking user, and the time-series and
sidecar capabilities it declared. The registry checks the grant on every call.
What a plugin receives at load is limited to:

- **A scoped database role.** `database` connects as a role of the form
  `minder_plugins_<plugin>`, confined to that plugin's own `plugin_<plugin>`
  schema, with no privilege on the platform's tables or on another plugin's
  schema. Without `DB_PLUGIN_PASSWORD` a plugin gets no database handle at all.
- **No Redis, InfluxDB, Neo4j or RabbitMQ handle.** Time-series data goes
  through the registry's time-series capability and `timeseries-gateway`, the
  only process that holds the InfluxDB admin token. A plugin that used to write
  to Neo4j or RabbitMQ directly (the `network` plugin) has those sinks off in a
  runner.

!!! warning "Isolation is not complete"
    Plugin code in the `trusted` and `privileged` groups is still arbitrary
    Python inside its container. The in-process egress guard in the runner is a
    backstop for honest plugins, not a boundary: the boundary is the network
    layer around it. Per-plugin egress allowlists for community code are
    not shipped. List your host's own public and
    LAN addresses in `EGRESS_PROXY_DENY_CIDRS`, because the runner cannot
    discover addresses the Docker host has on its other interfaces.

Catalog plugins arrive as pull requests to the public
[plugins](https://github.com/minderhq/plugins) repo. They are reviewed, and CI
runs `minder-plugin validate` and a contract suite on each one. Submitted
marketplace listings go through an admin review workflow before they're
publicly visible (first-party listings are pre-approved). See
[Publishing](plugins/publishing.md).

**Tool effects and approvals.** Plugins declare an `effect` on each AI tool:
`read`, `write`, `send` or `destructive`. In tool-calling chat, `read` tools
run straight away. Any other tool, or a tool with no declared or unknown
effect, pauses until the user approves that exact call. See
[Tool approvals](api-reference.md#tool-approvals).

### What's *not* in place (don't assume)

- **Complete plugin sandboxing.** Plugins run in separate runner containers,
  but code plugins are not run in a per-plugin sandbox. See
  [Plugin trust model](#plugin-trust-model).
- **RBAC is partial.** A user's `role` is checked on a specific set of admin-only
  actions (model pull/delete, bundle enable/disable/reconcile, plugin lifecycle
  and service-registry routes, the marketplace admin endpoints, and admin-only
  plugin actions). **Most other write endpoints still only require a valid JWT,
  not a particular role** — authorization is not yet uniform. Don't build
  workflows that assume broader per-role enforcement than that.
- **Full browser SSO needs real DNS + TLS.** Over plain `localhost` or a bare LAN
  IP the forward-auth 302 works but end-to-end browser SSO does not.

## Secrets

Configuration and secrets live in a **single root `.env`** — the one source of
truth. `bash setup.sh` self-heals it: any value left as a `CHANGEME` placeholder
(or missing) is replaced with a strong random value on `install`/`start`, and the
file is chmod-ed to `600`. A fresh install therefore already has real secrets,
not placeholders.

The placeholders shipped in `.env.example` are, for example:

```bash
POSTGRES_PASSWORD=CHANGEME_POSTGRES_SECRET_32_CHARS
REDIS_PASSWORD=CHANGEME_REDIS_SECRET_32_CHARS
JWT_SECRET=CHANGEME_JWT_SECRET_MINIMUM_64_CHARS_RECOMMENDED
NEO4J_AUTH=neo4j/CHANGEME_NEO4J_PASSWORD_16_CHARS
```

Only set your own value if you want a specific one. A value you supply must have
the same shape as a generated one (lowercase hex, exact length; see the table
below). Otherwise `setup.sh` treats it as a placeholder and replaces it. The one
exception is the Authelia admin password, which may be any value you choose. Authelia's `admin` password follows the
same self-heal: it's generated per deployment, argon2id-hashed, and the plaintext
is printed to the terminal **once** when first generated — record it then (see
[Authentication](authentication.md)).

!!! danger "Never commit real secrets"
    Keep `.env` out of version control. It is already covered by `.gitignore`.
    Use distinct credentials per deployment — each machine has its own root
    `.env`.

### Generated secrets

`setup.sh` generates and manages every secret below. "Length" is the length of
the generated value: lowercase hex, two characters per random byte.

| Secret | Length | Used for |
|--------|--------|----------|
| `POSTGRES_PASSWORD` | 64 hex | PostgreSQL owner role `minder` (migrations, schema) |
| `DB_APP_PASSWORD` | 64 hex | Non-superuser runtime role `minder_app`. Each service's query pool connects as it, which is what makes row-level security apply |
| `DB_PLUGIN_PASSWORD` | 64 hex | Base secret for the per-plugin least-privilege roles (`minder_plugins_<plugin>`). The registry derives each role's password from it and creates the role when the plugin loads. Each role is confined to its own `plugin_<plugin>` schema |
| `OPENWEBUI_DB_PASSWORD` | 64 hex | Open WebUI's own database role, which owns the `openwebui` database only |
| `REDIS_PASSWORD` | 64 hex | Redis |
| `RABBITMQ_PASSWORD` | 64 hex | RabbitMQ |
| `MINIO_ROOT_PASSWORD` | 64 hex | MinIO root user |
| `JWT_SECRET` | 128 hex | Legacy HS256 signing key. Only the API Gateway receives it, and only to accept sessions that predate the EdDSA upgrade during a bounded window. No other service holds it. See [JWT signing and keys](authentication.md#jwt-signing-and-keys) |
| `JWT_SIGNING_PRIVATE_KEY`, `JWT_PUBLIC_KEYS` | Ed25519 key (base64 PEM) | The gateway's EdDSA signing key and the list of public keys every service verifies with. Setup generates both. Only the gateway receives the private key |
| `SERVICE_SIGNING_KEY_<SERVICE>` | Ed25519 key | One signing key per service that calls another service's internal routes. Each is given to its own service only. `SERVICE_PUBLIC_KEYS` carries every public half |
| `PLUGIN_T4_SIGNING_KEY` | base64 Ed25519 seed | The key plugin-registry signs its calls to remote plugins with. The public half is published at `/.well-known/minder/plugin-jwks.json` |
| `MODEL_PROVIDER_VAULT_SECRET` | 128 hex | Encrypts stored cloud-provider API keys. **Never clear it**, see [below](#vault-and-license-secrets-never-clear) |
| `PLUGIN_SECRETS_VAULT_SECRET` | 128 hex | Encrypts per-plugin connector secrets. **Never clear it** |
| `LICENSE_KEY_VAULT_SECRET` | 128 hex | Encrypts license keys at rest. **Never clear it** |
| `LICENSE_KEY_HASH_SECRET` | 128 hex | Keys the license-key lookup hash. **Must never change** |
| `NEO4J_AUTH` | `neo4j/` + 32 hex | Neo4j |
| `INFLUXDB_ADMIN_TOKEN` | `apiv3_` + 64 hex | InfluxDB's admin token. Only `timeseries-gateway` and InfluxDB receive it. **Never regenerated**: InfluxDB reads it once |
| `TIMESERIES_TOKEN_TELEGRAF`, `TIMESERIES_TOKEN_GRAPH_RAG`, `TIMESERIES_TOKEN_PLUGIN_REGISTRY` | 64 hex | One `timeseries-gateway` credential per consumer. Each is given to that consumer only |
| `AUTHELIA_STORAGE_ENCRYPTION_KEY` | 64 hex | Encrypts Authelia's own storage. **Never clear it** |
| `AUTHELIA_SESSION_SECRET`, `AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET`, `AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET` | 64 hex | Authelia sessions, password-reset links, OIDC tokens |
| `MINDER_OIDC_CLIENT_SECRET` | 64 hex | API Gateway's OIDC client secret with Authelia |
| `MINDER_AUTHELIA_ADMIN_PASSWORD` | 64 hex (or your own) | Authelia `admin` login, printed once when generated |
| `GRAFANA_PASSWORD` | 64 hex | Grafana admin |
| `WEBUI_SECRET_KEY` | 64 hex | Open WebUI session signing |

Older installs may still have `INFLUXDB_TOKEN`, `SERVICE_SYNC_TOKEN` or
`SERVICE_SYNC_TOKEN_ON_MIGRATED_ROUTES` in `.env`. Nothing reads them any more,
and setup removes those lines (after saving the usual backup).

Without `DB_PLUGIN_PASSWORD`, plugins get **no** database handle at all (fail
closed). They never fall back to the owner credentials, and the plugin registry
logs an error at startup.

### A new secret appeared after an upgrade

Releases sometimes add a secret to the list above (for example,
`DB_PLUGIN_PASSWORD` or the `TIMESERIES_TOKEN_*` credentials). `setup.sh` fills
missing secrets on `install`/`start`. To avoid desyncing live services, it
**refuses** to generate secrets while the stack is running, with one exception:
a brand-new secret whose service re-applies it from `.env` on every boot.
`DB_PLUGIN_PASSWORD`, `OPENWEBUI_DB_PASSWORD`, the `TIMESERIES_TOKEN_*`
credentials and `INFLUXDB_ADMIN_TOKEN` are of this kind, and setup adds them
automatically even on a running stack, as long as the key is absent from
`.env`. The JWT and service signing keys are also filled on a live stack.

For any other new secret, or if the key is present but empty or a placeholder,
setup stops with `Refusing to regenerate .env secrets`. Either:

- stop the stack, then start it again (`bash setup.sh stop`, then
  `bash setup.sh start`), so setup fills the key while nothing is running; or
- run setup with `MINDER_ALLOW_SECRET_REGEN=1` set. This is an explicit opt-in
  that lets setup generate secrets on a live stack. Like a normal run, it only
  fills keys that are missing, empty, placeholders, or malformed. It never
  touches a well-formed value you already have.

### File permissions

`setup.sh` sets `600` on `.env` automatically. To verify:

```bash
ls -la .env   # should show -rw------- (owner read/write only)
```

## Credential rotation

Rotate on a regular cadence (quarterly is a reasonable default), and immediately
on suspected compromise. Before you start, take a backup (`bash setup.sh backup`).
Before setup regenerates any secret, it saves the current file as
`.env.backup-<timestamp>` next to `.env`.

!!! danger "Not every secret can be regenerated"
    Clearing a secret makes `setup.sh` replace it with a **new random value**.
    That is fine for a password that a service re-reads on start. For a secret
    that **encrypts or hashes stored data**, it orphans that data permanently:

    | Secret | What breaks if it's regenerated |
    |--------|---------------------------------|
    | `MODEL_PROVIDER_VAULT_SECRET` | Stored cloud-provider API keys can no longer be decrypted |
    | `PLUGIN_SECRETS_VAULT_SECRET` | Stored per-plugin connector secrets can no longer be decrypted |
    | `LICENSE_KEY_VAULT_SECRET` | Encrypted license keys can no longer be revealed or rotated |
    | `LICENSE_KEY_HASH_SECRET` | Existing license keys stop validating |
    | `AUTHELIA_STORAGE_ENCRYPTION_KEY` | Authelia can no longer read its own encrypted storage |

    **Never clear these.** If one was regenerated by mistake, restore the old
    value from the newest `.env.backup-*` file before any further restart.

### Regenerable secrets

These can be rotated by clearing them and letting setup generate new values:

1. **Clear the keys to rotate** in the root `.env`, for example `REDIS_PASSWORD=`.
2. **Apply with the stack stopped.** Setup refuses to regenerate secrets while
   the stack is running.
   ```bash
   bash setup.sh stop
   bash setup.sh start
   ```
3. **Sync the ones a data store keeps.** Editing `.env` alone does **not** change
   a password that a data store saved at first initialization:

    | Secret | How the new value takes effect |
    |--------|--------------------------------|
    | `JWT_SECRET` | Only the legacy HS256 key, which the gateway uses just to honour sessions from before the EdDSA upgrade. Rotating it ends those sessions. To rotate the key that signs current tokens, see [Signing key rotation](#signing-key-rotation) |
    | `REDIS_PASSWORD`, `MINIO_ROOT_PASSWORD` | Picked up when the container is recreated on start |
    | `DB_APP_PASSWORD`, `DB_PLUGIN_PASSWORD` | Picked up on start: the services re-apply the `minder_app` role password, and each per-plugin role's derived password, from `.env` on every boot. No manual step needed |
    | `POSTGRES_PASSWORD` | Run `bash setup.sh sync-postgres-password` once the stack is up. It runs `ALTER USER minder`, so the live database matches `.env`. It only updates the `minder` owner role; the roles above sync themselves |
    | `RABBITMQ_PASSWORD`, `NEO4J_AUTH`, `GRAFANA_PASSWORD` | Saved in the service's data volume at first initialization. Setup has no sync command for these, so change the password inside the service with its own tooling, then put the same value in `.env`. Don't regenerate them blindly |
    | `TIMESERIES_TOKEN_*` | Picked up on start. `INFLUXDB_ADMIN_TOKEN` is the exception: InfluxDB reads it once, so setup never regenerates it |

4. **Verify:**
   ```bash
   bash setup.sh status
   curl http://localhost:8000/health
   ```

### Signing key rotation

The gateway's EdDSA key and the per-service keys are not hex secrets, so
clearing them is not how you rotate them:

- **JWT signing key.** Add the new public key to `JWT_PUBLIC_KEYS`
  (comma-separated, and keep the old one), restart, then replace
  `JWT_SIGNING_PRIVATE_KEY` with the matching new private key and restart
  again. Tokens signed with the old key stay valid until they expire.
- **Service signing keys.** Delete the `SERVICE_SIGNING_KEY_<SERVICE>` line,
  run setup again, and restart. Upgrade every service together.

### Vault and license secrets (never clear)

The services accept `MODEL_PROVIDER_VAULT_SECRET`, `PLUGIN_SECRETS_VAULT_SECRET`
and `LICENSE_KEY_VAULT_SECRET` as a **comma-separated list**:

- the **first** entry is the primary key, used for all new encryption;
- **every** entry is tried when decrypting, so data encrypted under an older
  entry stays readable as long as that entry is still in the list.

Each entry has the same shape as a generated value: 128 lowercase hex
characters (`openssl rand -hex 64`). Setup accepts a valid list as it is and
never regenerates a vault secret that already has a value, whether the stack is
running or stopped, and even with `MINDER_ALLOW_SECRET_REGEN=1`. It only fills
one that is missing, empty or still a placeholder.

To rotate one of these secrets:

1. **Prepend the new key.** Generate a new value and put it in front of the
   current one, separated by a comma, in the root `.env`:
   ```bash
   MODEL_PROVIDER_VAULT_SECRET=<NEW>,<OLD>
   ```
   Never replace the old value or clear the list.
2. **Restart** so the services load the new list:
   ```bash
   bash setup.sh restart
   ```
   New data is now encrypted with the new key, and existing data stays readable
   through the old one.
3. **Re-encrypt, where an endpoint exists.** Model providers have one:
   `POST /v1/model-providers/rotate` (admin or org admin). Call it once per
   organization to move that organization's stored keys onto the new primary.
   `PLUGIN_SECRETS_VAULT_SECRET` and `LICENSE_KEY_VAULT_SECRET` have no
   re-encryption endpoint, so keep their old entry in the list.
4. **Drop the old key only after re-encryption.** Once everything encrypted with
   the old entry has been re-encrypted, remove it (`<NEW>` alone) and restart
   again. If you're not sure, leave it in: an extra entry costs nothing, but a
   missing one makes the data it encrypted unreadable.

!!! warning "Setup stops on a malformed vault value"
    If a vault secret has a value that isn't a valid list (for example a
    truncated entry, an uppercase or non-hex character, or a wrong length),
    setup **refuses to start**. It logs which key is malformed and what it
    expects, and **writes nothing** to `.env`. It doesn't replace the value,
    because that would orphan the encrypted data. Fix the value by hand, or
    restore it from the newest `.env.backup-*` file, then run setup again.

`LICENSE_KEY_HASH_SECRET` is a single value with **no** rotation path. Every
stored license-key lookup hash is keyed with it, so it must stay the same for
the life of the installation. It is kept separate from `LICENSE_KEY_VAULT_SECRET`
so that the vault key can rotate without touching the hashes.

### On suspected compromise

Stop the stack first (`bash setup.sh stop`), rotate as above, restart, and
review the gateway access logs (`docker logs minder-api-gateway --tail 1000`).

## Secrets management in production (optional)

Minder's built-in mechanism is the single root `.env` (one per machine — there is
no `.env.development`/`.staging`/`.production` layering). For a hardened
deployment you may *optionally* layer an external secrets system on top; these are
forward-looking, not built in:

=== "Docker Swarm secrets"

    ```yaml
    services:
      postgres:
        secrets:
          - postgres_password
        environment:
          POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
    secrets:
      postgres_password:
        external: true
    ```

=== "Kubernetes Secret"

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: minder-credentials
    type: Opaque
    stringData:
      POSTGRES_PASSWORD: CHANGEME
      REDIS_PASSWORD: CHANGEME
      JWT_SECRET: CHANGEME
      INFLUXDB_ADMIN_TOKEN: apiv3_CHANGEME
    ```

## Production hardening checklist

- [ ] Configure real DNS for the public hostnames.
- [ ] Replace the self-signed `.local` certificates with real TLS (ACME /
      Let's Encrypt via Traefik, or your own certs) so Authelia SSO/2FA works as
      full browser SSO.
- [ ] Confirm no `CHANGEME_` placeholders remain: `grep CHANGEME_ .env` should
      print nothing.
- [ ] Verify `.env` permissions are `600` and it is not committed to git.
- [ ] Add firewall rules; remember host ports are loopback-bound, so protect
      access to the box itself.
- [ ] Establish a credential-rotation cadence.
- [ ] If you enable [transactional email](email.md), keep `SMTP_PASSWORD` in
      `.env` only, use `starttls` or `tls` (not `none`), and publish
      SPF/DKIM/DMARC for the sender domain.
- [ ] Add alerting on failed authentication attempts and anomalous access (see
      [Monitoring](monitoring.md)).
- [ ] Keep Traefik and the service images updated (see [Upgrading](upgrades.md)).

## Verifying the basics

```bash
# No default credentials remain
grep CHANGEME_ .env            # should print nothing

# File permissions
ls -la .env                    # -rw------- (600)

# Services healthy
bash setup.sh status
```

## Troubleshooting

- **A service exits immediately or is unhealthy** — check its logs
  (`docker logs minder-<service>`); confirm `.env` exists and is populated.
- **"password authentication failed" (PostgreSQL)** — the live DB password and
  `.env` have diverged; run `bash setup.sh sync-postgres-password`.
- **"WRONGPASS" (Redis)** — `REDIS_PASSWORD` in `.env` doesn't match what Redis is
  running with; restart Redis after fixing it.
- **401 on API requests** — confirm the token hasn't expired, and that every
  service has the same `JWT_PUBLIC_KEYS` (setup writes it). A service that has
  none refuses EdDSA tokens. See [Authentication](authentication.md).

More diagnostics are in [Troubleshooting](troubleshooting.md).

## Resources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Docker secrets](https://docs.docker.com/engine/swarm/secrets/)
- [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [JWT best practices (RFC 8725)](https://datatracker.ietf.org/doc/html/rfc8725)
