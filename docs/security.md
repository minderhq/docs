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

Only set your own value if you want a specific one — anything you put in `.env`
wins over the auto-generated default. Authelia's `admin` password follows the
same self-heal: it's generated per deployment, argon2id-hashed, and the plaintext
is printed to the terminal **once** when first generated — record it then (see
[Authentication](authentication.md)).

!!! danger "Never commit real secrets"
    Keep `.env` out of version control. It is already covered by `.gitignore`.
    Use distinct credentials per deployment — each machine has its own root
    `.env`.

### Recommended secret strength

| Secret | Minimum | Notes |
|--------|---------|-------|
| `POSTGRES_PASSWORD` | 32 chars | Mixed case, digits, symbols |
| `REDIS_PASSWORD` | 32 chars | Mixed case, digits, symbols |
| `JWT_SECRET` | 64 chars (128+ recommended) | Cryptographically random |
| `INFLUXDB_TOKEN` | 32–40 chars | Alphanumeric + symbols |

The auto-generated values already meet these; the table is for when you supply
your own.

### File permissions

`setup.sh` sets `600` on `.env` automatically. To verify:

```bash
ls -la .env   # should show -rw------- (owner read/write only)
```

## Credential rotation

Rotate on a regular cadence (quarterly is a reasonable default), and immediately
on suspected compromise.

1. **Clear the keys to rotate** in the root `.env` (e.g. `JWT_SECRET=`);
   `setup.sh` regenerates emptied secret keys on the next start, backing up the
   file first.
2. **Apply:**
   ```bash
   bash setup.sh stop
   bash setup.sh start
   ```
3. **Rotate stateful secrets at the source.** Editing `.env` alone does **not**
   change a live database's stored password. After changing `POSTGRES_PASSWORD`:
   ```bash
   bash setup.sh sync-postgres-password   # ALTER USER so the live DB matches .env
   ```
4. **Verify:**
   ```bash
   bash setup.sh status
   curl http://localhost:8000/health
   ```

On suspected compromise, stop the stack first (`bash setup.sh stop`), rotate as
above, restart, and review the gateway access logs
(`docker logs minder-api-gateway --tail 1000`).

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
