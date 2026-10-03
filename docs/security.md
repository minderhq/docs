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

- **JWT authentication** at the API Gateway (HS256), with bcrypt-hashed
  credentials and Redis-backed rate limiting (60-second window, **fail-open** —
  if Redis is unreachable, requests are allowed rather than blocked).
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
- **No arbitrary code execution in plugins** — plugins are manifest-based, fixed,
  reviewed handlers. Safety is a design property, not a scanner bolted on after.

### What's *not* in place (don't assume)

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
INFLUXDB_TOKEN=CHANGEME_INFLUXDB_SECRET_40_CHARS
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
| `DB_PLUGIN_PASSWORD` | 64 hex | Least-privilege role `minder_plugins` that in-process plugins connect as. It is confined to the `plugin_data` schema and has no access to platform tables |
| `REDIS_PASSWORD` | 64 hex | Redis |
| `RABBITMQ_PASSWORD` | 64 hex | RabbitMQ |
| `MINIO_ROOT_PASSWORD` | 64 hex | MinIO root user |
| `JWT_SECRET` | 128 hex | Signs Minder JWTs |
| `MODEL_PROVIDER_VAULT_SECRET` | 128 hex | Encrypts stored cloud-provider API keys. **Never clear it**, see [below](#vault-and-license-secrets-never-clear) |
| `PLUGIN_SECRETS_VAULT_SECRET` | 128 hex | Encrypts per-plugin connector secrets. **Never clear it** |
| `LICENSE_KEY_VAULT_SECRET` | 128 hex | Encrypts license keys at rest. **Never clear it** |
| `LICENSE_KEY_HASH_SECRET` | 128 hex | Keys the license-key lookup hash. **Must never change** |
| `NEO4J_AUTH` | `neo4j/` + 32 hex | Neo4j |
| `INFLUXDB_TOKEN` | 80 hex | InfluxDB |
| `AUTHELIA_STORAGE_ENCRYPTION_KEY` | 64 hex | Encrypts Authelia's own storage. **Never clear it** |
| `AUTHELIA_SESSION_SECRET`, `AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET`, `AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET` | 64 hex | Authelia sessions, password-reset links, OIDC tokens |
| `MINDER_OIDC_CLIENT_SECRET` | 64 hex | API Gateway's OIDC client secret with Authelia |
| `MINDER_AUTHELIA_ADMIN_PASSWORD` | 64 hex (or your own) | Authelia `admin` login, printed once when generated |
| `GRAFANA_PASSWORD` | 64 hex | Grafana admin |
| `WEBUI_SECRET_KEY` | 64 hex | Open WebUI session signing |
| `SERVICE_SYNC_TOKEN` | 64 hex | Internal service-to-service token (plugin AI-tool catalog sync) |

Without `DB_PLUGIN_PASSWORD`, plugins get **no** database handle at all (fail
closed). They never fall back to the owner credentials, and the plugin registry
logs an error at startup.

### A new secret appeared after an upgrade

Releases sometimes add a secret to the list above (for example,
`DB_PLUGIN_PASSWORD`). `setup.sh` fills missing secrets on `install`/`start`.
To avoid desyncing live services, it **refuses** to generate secrets while the
stack is running, with one exception: a brand-new secret that its service
re-applies from `.env` on every boot. `DB_PLUGIN_PASSWORD` is one of these, and
setup adds it automatically even on a running stack, as long as the key is
absent from `.env`.

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

1. **Clear the keys to rotate** in the root `.env`, for example `JWT_SECRET=`.
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
    | `JWT_SECRET` | Picked up on restart. All existing tokens become invalid, so users sign in again |
    | `REDIS_PASSWORD`, `MINIO_ROOT_PASSWORD` | Picked up when the container is recreated on start |
    | `DB_APP_PASSWORD`, `DB_PLUGIN_PASSWORD` | Picked up on start: the services re-apply the `minder_app` and `minder_plugins` role passwords from `.env` on every boot. No manual step needed |
    | `POSTGRES_PASSWORD` | Run `bash setup.sh sync-postgres-password` once the stack is up. It runs `ALTER USER minder`, so the live database matches `.env`. It only updates the `minder` owner role; the two roles above sync themselves |
    | `RABBITMQ_PASSWORD`, `NEO4J_AUTH`, `INFLUXDB_TOKEN`, `GRAFANA_PASSWORD` | Saved in the service's data volume at first initialization. Setup has no sync command for these, so change the password inside the service with its own tooling, then put the same value in `.env`. Don't regenerate them blindly |

4. **Verify:**
   ```bash
   bash setup.sh status
   curl http://localhost:8000/health
   ```

### Vault and license secrets (never clear)

The services accept `MODEL_PROVIDER_VAULT_SECRET`, `PLUGIN_SECRETS_VAULT_SECRET`
and `LICENSE_KEY_VAULT_SECRET` as a **comma-separated list**:

- the **first** entry is the primary key, used for all new encryption;
- **every** entry is tried when decrypting, so data encrypted under an older
  entry stays readable as long as that entry is still in the list.

You rotate one of these by **prepending** a new value and keeping the old one
after it (`NEW,OLD`). Never replace the old value or clear the list. Model
providers have a re-encryption endpoint,
`POST /v1/model-providers/rotate` (admin or org admin). After the restart, call
it once per organization to move that organization's stored keys onto the new
primary. Only remove an old entry after
everything encrypted with it has been re-encrypted.

!!! warning "Current setup tooling doesn't preserve a multi-entry value"
    Setup's secret self-heal currently treats a comma-separated vault value as
    malformed. With the stack running, it refuses to start, and on a stopped
    stack, `bash setup.sh start` (or a full `restart`) **replaces the whole list
    with one new random value**, which orphans the encrypted data. The same
    happens on a running stack if setup runs with `MINDER_ALLOW_SECRET_REGEN=1`. Until setup
    supports multi-entry vault secrets, **don't rotate these secrets** unless
    there is a confirmed compromise. If a list was replaced, restore it from the
    newest `.env.backup-*` file.

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
      INFLUXDB_TOKEN: CHANGEME
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
- **401 on API requests** — confirm `JWT_SECRET` is consistent across services and
  the token hasn't expired. See [Authentication](authentication.md).

More diagnostics are in [Troubleshooting](troubleshooting.md).

## Resources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Docker secrets](https://docs.docker.com/engine/swarm/secrets/)
- [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [JWT best practices (RFC 8725)](https://datatracker.ietf.org/doc/html/rfc8725)
