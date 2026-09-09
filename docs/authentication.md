# Authentication

Minder has a **single login entry point**: the "Log in" button in the top-right
of the control-plane UI. Clicking it takes you to Authelia's own login page (the
platform's identity provider); coming back mints a Minder session automatically.
There is no separate Minder-specific username/password form and no scattered
per-page login boxes.

Under the hood, Authelia issues a verified identity via **OIDC**, the **API
gateway** (`minder-api-gateway`, port `8000`) exchanges that for its own **JWT**,
and every other service trusts that JWT. Authelia also gates several other web
UIs (Grafana, OpenWebUI, MinIO, Jaeger) directly at the reverse-proxy layer,
independent of Minder's own login — see [Authelia](#authelia-sso-oidc) below.

!!! note "Self-hosted deployment"
    This page describes a self-hosted deployment. Some hardening (RBAC across the
    full write surface, TLS everywhere) is not yet fully applied — see
    [Roles](#roles-partially-enforced).

## How browser login works

```
┌─────────────┐   1. click "Log in"    ┌──────────────────────┐
│   Client    ├───────────────────────▶│  GET /v1/auth/oidc/  │
│  (browser)  │                        │  login (api-gateway) │
└──────▲──────┘                        └──────────┬───────────┘
       │                                           │ redirect
       │ 4. #token=<minder-jwt>                    ▼
       │                                ┌──────────────────────┐
       │                                │  Authelia login page │
       │                                │  (SSO session reused │
       │                                │  if you're already   │
       │                                │  logged in elsewhere)│
       │                                └──────────┬───────────┘
       │                                           │ authorization code
       │           3. verify + mint Minder JWT     ▼
       └────────────────────────────────GET /v1/auth/oidc/callback
                                        (api-gateway ↔ Authelia,
                                         server-to-server)
```

1. The client's "Log in" button links straight to `GET /v1/auth/oidc/login`.
2. That redirects your browser to Authelia. If you already have an Authelia
   session (e.g. you're logged into Grafana/OpenWebUI, or the client's own
   forward-auth gate already established one), this step can complete silently
   with no extra prompt.
3. Authelia redirects back with an authorization code. The api-gateway exchanges
   it for a verified identity, then looks up or provisions a matching Minder user.
   First login creates the account, or links it to a pre-existing local account
   with the same username. Every login syncs username, email, and role from
   Authelia's `groups` claim — including the first login that links a pre-existing
   local account, so a user in Authelia's `admins` group gets `role: admin`
   starting with that first SSO login.
4. The api-gateway mints a normal Minder JWT and redirects your browser to
   `/auth/callback#token=...`. The client reads the token from the URL fragment
   (never sent to any server, so it never lands in an access log) and you're
   logged in.

From here, every API call the client makes carries that JWT the usual way:

```http
GET /v1/plugins
Authorization: Bearer <access_token>
```

### Your account

Click your username (top-right, once logged in) to open **Settings** — it shows
your username, email, and role as Minder sees them, plus a "Log out" button.
Because your real account lives in Authelia, actual profile and password changes
happen there, not in Minder; Settings links straight to Authelia's own portal.
See [Using Minder](using-minder.md) for the full UI tour.

### Local register/login (for scripting and dev)

The original username/password endpoints still exist and mint the exact same JWT
shape — useful for scripts, CI, or local dev without a browser — but the UI no
longer exposes a form for them. They are also what the [CLI](cli.md) uses.

```http
POST /v1/auth/register
Content-Type: application/json

{ "username": "alice", "email": "alice@example.com", "password": "CHANGEME" }
```

```http
POST /v1/auth/login
Content-Type: application/json

{ "username": "alice", "password": "CHANGEME" }
```

**Response (same shape from either login path):**

```json
{
  "access_token": "<jwt>",
  "token_type": "bearer",
  "expires_in": 1800,
  "user": { "id": 1, "username": "alice", "email": "alice@example.com", "role": "user" }
}
```

```http
POST /v1/auth/refresh
Authorization: Bearer <access_token>
```

!!! note
    There is no account-level password-change or reset endpoint for
    locally-registered accounts — only register, login, refresh, and the OIDC
    login/callback are implemented. If you need password management, log in via
    Authelia instead; its own portal handles it.

### Roles (partially enforced)

Logging in via Authelia sets your Minder `role` from Authelia's `groups` claim:
membership in the `admins` group becomes `role: admin`, everyone else gets
`role: user`. You can see your own role on the Settings page.

Role checks currently cover a specific set of admin-only actions — a model
pull/delete, a bundle enable/disable/reconcile, and listing who installed a
marketplace plugin — plus the organizations/teams/RBAC surface:

- Any authenticated user can create a team (becoming its `team_admin`).
- Updating or deleting a team, managing its membership, and issuing, listing, or
  revoking its invites requires that team's own `team_admin` or an instance
  `admin`.
- A team can never be left with zero `team_admin`s — demoting or removing the
  last one is rejected with a `409` unless the caller is an instance admin
  (deleting the team outright is a separate, allowed path).
- Instance-level user management (`GET /v1/auth/users`,
  `PATCH /v1/auth/users/{id}/role`) is instance-admin-only, with **no equivalent
  last-admin guard**.

!!! warning
    Everywhere else that requires auth still only checks "is there a valid JWT,"
    not "does this JWT's role allow it." Don't build workflows that assume broader
    per-role restrictions are enforced than that.

### JWT secret

Tokens are signed with `JWT_SECRET`, which lives in the root `.env` file. The
setup CLI auto-generates a strong value if you leave the placeholder in place.
The same secret must be consistent across services that validate tokens.

## Traefik (reverse proxy)

Traefik is the single entry point: TLS termination and request routing. Routing
is driven by Docker labels (not exposed by default), and the dashboard is
restricted by an IP-whitelist middleware.

Traefik wires an `authelia-forwardauth` middleware onto the routers for MinIO,
the api-gateway, Grafana, OpenWebUI, Jaeger, and the client. Other routers use an
IP-whitelist middleware instead. Authelia is enabled and running, so that
forward-auth check **is enforced** — an unauthenticated request to those routes
gets a `302` redirect to the Authelia portal.

This forward-auth gate is separate from, and composes with, the OIDC login flow
above: because the client router is forward-auth-gated, visiting the client at
all already required an Authelia session — which is *why* step 2 of the login
flow can usually complete silently.

## Authelia (SSO / OIDC) {#authelia-sso-oidc}

Authelia is Minder's identity provider and is enabled and running. Two
independent things both come from it:

1. **Forward-auth gate** — the Traefik `authelia-forwardauth` middleware enforces
   a login on the gated routers; an unauthenticated request is `302`-redirected
   to the Authelia portal.
2. **OIDC identity provider** — the api-gateway is a registered OIDC client; this
   is what mints your actual Minder JWT when you click "Log in" (see the flow
   diagram above).

!!! important "Real DNS + TLS required for browser SSO"
    Full browser SSO needs real DNS and valid TLS on the deployment. Over plain
    `localhost` or a bare LAN IP, the SSO button is hidden and you use a
    [local account](#local-registerlogin-for-scripting-and-dev) instead — see
    [Using Minder](using-minder.md).

What Authelia provides:

- Single Sign-On across services.
- OIDC-based Minder login (this page's main flow).
- Two-Factor Authentication (TOTP / WebAuthn), per Authelia's configuration.
- Brute-force protection and session regulation (Authelia defaults).
- Access-control rules per domain, in Authelia's `access_control` configuration.

### The admin password

Authelia's users database ships as a **template** holding a placeholder, not a
real hash. On first run the setup CLI generates a random admin password, stores
it in `.env` (the same self-healing mechanism as every other secret, e.g.
`JWT_SECRET`), argon2id-hashes it via Authelia's own CLI, and writes the real
value into the gitignored file that is actually mounted. Every deployment gets
its own password instead of a shared hardcoded one.

!!! warning "The plaintext is printed once"
    The generated password is printed to the terminal exactly once, the moment
    it's created — record it then. It is never shown again and is never written to
    a log file.

    ```
    ┌──────────────────────────────────────────────────┐
    │  Authelia Admin Password (generated)              │
    └──────────────────────────────────────────────────┘
    Record this now -- it will not be shown again.
      Username: admin
      Password: <random>
    ```

If you lose it, rotate: clear the admin-password line in `.env` and re-run the
setup CLI's start command to generate and print a new one.

## Troubleshooting

### API requests return 401

- Confirm you sent `Authorization: Bearer <token>` and the token has not expired.
- Confirm `JWT_SECRET` is set consistently across services.
- Check the gateway logs:

  ```bash
  docker logs minder-api-gateway --tail 100
  ```

### Rate limited (429)

The gateway applies Redis-backed rate limiting on a 60-second window (fail-open).
Confirm Redis is healthy:

```bash
docker exec -it minder-redis redis-cli -a "$REDIS_PASSWORD" ping
```

### Cannot reach a service through Traefik

```bash
docker logs minder-traefik --tail 100
```

## Additional resources

- [Self-hosting Minder](self-hosting.md) — stand up an instance first.
- [Traefik documentation](https://doc.traefik.io/traefik/)
- [Authelia documentation](https://www.authelia.com/)
- [JWT best practices (RFC 8725)](https://datatracker.ietf.org/doc/html/rfc8725)
