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
    This page describes a self-hosted deployment. Authorization is checked per
    route, not uniformly across the whole write surface — see
    [Roles & permissions](#roles-and-permissions).

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
   First login creates the account. It links to a pre-existing local account
   with the same username only if an operator approved that link beforehand
   (see [Moving a local account to SSO](#moving-a-local-account-to-sso)).
   Every login syncs username, email, and role from Authelia's `groups` claim,
   including the first login that links an approved local account, so a user in
   Authelia's `admins` group gets `role: admin` starting with that first SSO
   login.
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
For an account that signs in through Authelia, profile and password changes
happen there, not in Minder; Settings links straight to Authelia's own portal.
A locally-registered account changes its password through Minder instead — see
[Password management](#password-management). See
[Using Minder](using-minder.md) for the full UI tour.

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
  "expires_in": 900,
  "user": {
    "id": 1,
    "username": "alice",
    "email": "alice@example.com",
    "role": "user",
    "created_at": "2026-01-01T00:00:00",
    "must_change_password": false
  }
}
```

`expires_in` is in seconds: `JWT_EXPIRATION_MINUTES × 60` (`900` by default).
`must_change_password` is `true` after an administrator has reset the password
(see [Password management](#password-management)).

### Moving a local account to SSO {#moving-a-local-account-to-sso}

An SSO login never takes over a local account just because the usernames
match. Anyone can register a local account, so a matching username alone
doesn't prove it belongs to the SSO user. Without an approval, the first SSO
login gets a **separate** account: its username gets a suffix
(`<username>-<8 characters>`), and the local account is left as it was. The
refused link is logged and audited.

To move a genuine local account to SSO, so the user keeps their data under the
same account, approve it on the server **before the user's first SSO login**:

```bash
bash setup.sh sso-link approve <username>   # allow the link
bash setup.sh sso-link revoke <username>    # withdraw a pending approval
bash setup.sh sso-link list                 # list pending approvals
```

- Check that the local account really belongs to that person first. `approve`
  shows its registered email, which was self-declared at registration and
  never verified.
- Approve **before** the first SSO login. Once the SSO identity has its own
  account, it is never linked to another one. In that case, the user keeps
  using the SSO account.
- The next SSO login with that username takes over the account and uses up the
  approval. The link replaces the local password (the account becomes
  SSO-only), ends the account's existing sessions, and takes the role and email
  from Authelia.
- A linked account loses Platform Admin if it had it, and is never made
  Platform Admin automatically. Re-grant it with the CLI if needed; see
  [Platform Admin](platform-admin.md).
- `approve` and `revoke` are audited. They are refused for an account that is
  already linked to SSO.

### Token lifetime & sessions {#token-lifetime-and-sessions}

Access tokens are short-lived. To keep a session going, the client renews the
token with `POST /v1/auth/refresh` until the session reaches its absolute cap.
Three settings in the root `.env` control this. Compose passes all three to the
gateway. A setting that is missing from `.env` uses its default, and a change
takes effect when you restart the stack:

| Setting | Default | Meaning |
|---------|---------|---------|
| `JWT_EXPIRATION_MINUTES` | `15` | Lifetime of every access token (`expires_in`). Every route except `/v1/auth/refresh` rejects a token once it expires. |
| `JWT_REFRESH_GRACE_MINUTES` | `720` (12 h) | How long after expiry a token can still be exchanged at `/v1/auth/refresh`, so a laptop that slept or a throttled background tab can renew instead of being logged out. `0` disables the window. |
| `JWT_SESSION_MAX_HOURS` | `24` | Absolute session cap, counted from the original sign-in (password or SSO login). Refreshing, switching organization and other token re-issues don't extend it. `0` disables the cap. |

```http
POST /v1/auth/refresh
Authorization: Bearer <access_token>
```

**Response:** `{ "access_token": "<jwt>", "token_type": "bearer", "expires_in": 900 }`

What refresh does:

- It accepts a valid token, or one that expired at most
  `JWT_REFRESH_GRACE_MINUTES` ago. The new token always gets a fresh issue
  time and a full `JWT_EXPIRATION_MINUTES` lifetime.
- It re-checks the account against the database and re-derives the claims from
  current state: instance `role`, `teams`, organization claims and Platform
  Admin status. A promotion, demotion or team change therefore takes effect on
  the next refresh, without a new login. An active organization you switched
  into is kept while you are still an unsuspended member of it.
- It returns **`401`** and does not issue a token when:
    - the account is disabled or no longer exists;
    - the account's sessions were revoked after the token was issued (see
      below);
    - the session is older than `JWT_SESSION_MAX_HOURS`
      (`"Session has expired -- sign in again"`);
    - the token is a service token or has no subject.

  In each of these cases the user has to sign in again.
- It is rate-limited, with **`429`** over the limit: 60 requests per minute per
  client IP and 10 per minute per account.

Clients should refresh shortly before `exp` and retry once on a `401`. The web
client does both automatically.

#### Session revocation

The gateway checks every access token against the account on every request:
its own routes, and every route it proxies to a backend service. A token is
refused with **`401`** within a few seconds when the account is deactivated or
deleted, or when its sessions are revoked. Its refresh is refused too. The
check is exact: a token issued even a fraction of a second before the
revocation is refused.

These actions revoke **all** of a user's existing sessions:

| Action | Notes |
|--------|-------|
| The user changes their own password | The response carries a replacement token, so only the device that made the change stays signed in. |
| An administrator resets the user's password | Platform Admin or org-scoped reset; see [Password management](#password-management). |
| The account is deactivated | Reactivating the account doesn't bring the old tokens back. |
| Instance role demotion (`admin` → `user`) | Through `PATCH /v1/auth/users/{user_id}/role`, or the role sync on SSO login. A promotion revokes nothing. |
| Organization role demotion (`owner` → `admin`/`member`, `admin` → `member`) | Also applies when the user is removed from an organization or their membership is suspended. |
| Platform Admin is revoked | Done with the server CLI (`platform-admin revoke`). |

After a revocation, the next sign-in issues a token with claims taken from the
current database state.

#### 401 on expired or invalid tokens

If a request carries an `Authorization: Bearer` token that fails verification,
the gateway answers **`401`** and doesn't forward the request. Failing
verification means the token is expired, malformed, wrongly signed, or belongs
to a disabled or revoked session. This also applies to read routes that work
without a token. To make an anonymous request, leave the `Authorization`
header out instead of sending a stale token. A request with two
`Authorization` headers gets **`400`**.

### JWT secret

Tokens are signed with `JWT_SECRET`, which lives in the root `.env` file. The
setup CLI auto-generates a strong value if you leave the placeholder in place.
The same secret must be consistent across services that validate tokens.

## Password management

These endpoints manage the passwords of **locally-registered** accounts. An
SSO-linked account gets **`409`** from all of them: its password lives in
Authelia, so change or reset it in the Authelia portal. New passwords must be
at least 8 characters.

| Endpoint | Who may call it | What it does |
|----------|-----------------|--------------|
| `POST /v1/auth/change-password` | Any signed-in user, for their **own** account | Body: `{current_password, new_password}`. Returns `200` with a replacement token, in the same shape as the `/refresh` response; adopt it to stay signed in. Revokes every other session. Returns `400` if `current_password` is wrong or the new password equals the current one. |
| `POST /v1/auth/users/{user_id}/reset-password` | **Platform Admin** | Reset any user's password across tenants (see below). |
| `POST /v1/organizations/{org_id}/members/{user_id}/reset-password` | A caller with `org.members.manage` on the organization, or a Platform Admin | Reset the password of an account managed by that organization (see below). |

**Both reset endpoints:**

- Take `{"mode": "set", "new_password": "..."}`, or `{"mode": "generate"}` to
  have the server create a strong temporary password.
- Return the temporary password **once**, in that response only
  (`Cache-Control: no-store`). It is never stored in plaintext or logged.
- Set `must_change_password` on the account and revoke its sessions.
- Can't be used on your own account (`409`); use
  `POST /v1/auth/change-password` instead.

The org-scoped reset has extra rules:

- It only works on an account the organization **manages**: one that belongs
  to that organization's tree and no other.
- Only an owner-level caller (`org.roles.manage`) can act on an admin-level
  account.
- Nobody can reset an owner's password through it.
- `GET /v1/organizations/{org_id}/members` reports for each member whether the
  caller can act on that account (`managed_by_caller`) and whether this reset
  would be accepted (`resettable_by_caller`). These flags are hints: the action
  routes check everything again.

### Deactivating and reactivating accounts

Deactivation is a soft, reversible removal. The account can no longer sign in
(password or SSO) or refresh, its existing tokens are refused at once, and its
data stays intact.

| Endpoint | Who may call it |
|----------|-----------------|
| `PATCH /v1/auth/users/{user_id}/status`, body `{"is_active": false\|true}` | **Platform Admin** |
| `PATCH /v1/organizations/{org_id}/members/{user_id}/account-status`, same body | A caller with `org.members.manage` on the organization, or a Platform Admin. Same managed-account and owner-level rules as the org-scoped reset. |

Deactivation applies to the account everywhere, not only in one organization.
To take a user out of a single organization, suspend or remove their
membership there instead. Both routes refuse your own account (`409`). The
Platform Admin route also refuses the last active Platform Admin and the last
active instance admin, and both routes refuse the last active owner of an
organization (`409`).

## Roles & permissions {#roles-and-permissions}

Authorization in Minder has three layers:

- **Instance role** (`user` / `admin`). On a self-hosted instance this comes
  from Authelia's `groups` claim: members of the `admins` group get
  `role: admin`. It still gates some instance-level actions, but on its own
  it does **not** give cross-tenant user administration.
- **Platform Admin**: the only cross-tenant identity. It is a separate flag on
  the account, re-checked against the database on every request, and it gates
  `/v1/auth/users/*` (list users, change instance role, reset passwords,
  deactivate accounts). On a fresh install, the first instance admin to sign in
  becomes Platform Admin automatically, once. After that, it is granted and
  revoked only with the server CLI (`platform-admin grant|revoke <username>`).
  An upgraded install that has no Platform Admin issues a one-time setup code
  with `platform-admin setup-code`, which an instance admin enters on the
  **Users** page. After a grant, refresh the token
  (`POST /v1/auth/refresh`) or sign in again to pick up the new claim. See
  [Platform Admin](platform-admin.md) for the operator steps.
- **Organization and team roles**: the built-in `owner` / `admin` / `member`
  organization roles and `team_admin` / `member` team roles. Organizations
  can also define **custom roles** made of permission keys (for example
  `org.members.manage`, `org.roles.manage`, `org.billing.manage`) and nestable
  **permission groups**. A role defined in an organization can be used
  throughout its sub-organizations. A role gives a member permissions only
  once it is assigned to them, optionally with an expiry. Use
  `GET /v1/organizations/{org_id}/members/{user_id}/effective-permissions`
  to see what a member can actually do.

See [Organizations & teams](organizations-teams.md) for the membership model
and [API reference](api-reference.md) for the routes.

!!! warning
    Authorization is checked **per route**. The [API reference](api-reference.md)
    lists who may call each route. A route listed as "any authenticated user"
    only checks for a valid token, so don't assume broader per-role
    restrictions than the ones documented.

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

- Confirm you sent `Authorization: Bearer <token>` and the token has not
  expired. Access tokens last `JWT_EXPIRATION_MINUTES` (15 by default); renew
  them with `POST /v1/auth/refresh` (see
  [Token lifetime & sessions](#token-lifetime-and-sessions)).
- If refresh also returns `401`, the session can't be renewed and you have to
  sign in again. This happens when the session is past `JWT_SESSION_MAX_HOURS`
  (24 h by default), when the account was deactivated, or when its sessions
  were revoked (password change or reset, role demotion, organization
  removal or suspension). The response `detail` says which.
- A stale token gets a `401` even on routes that work without a token. Leave
  the `Authorization` header out to call them anonymously.
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
