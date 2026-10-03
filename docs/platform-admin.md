# Platform Admin

**Platform Admin** is the cross-tenant operator role on a self-hosted Minder
instance. This page explains how an operator gets it, how to grant and revoke
it, and what happens to sessions when it changes. For how it fits with the
instance and organization roles, see
[Authentication → Roles & permissions](authentication.md#roles-and-permissions).

## What it gates

Platform Admin is a flag on the account, separate from the instance `role`. The
gateway re-checks it against the database on every request, so a stale token
can't keep it. It is required for:

- the **Users** page and the `/v1/auth/users` routes: list users across every
  tenant, change an instance role, reset passwords, deactivate and reactivate
  accounts;
- cross-tenant organization and team administration, for organizations you
  aren't a member of;
- core service container logs (`GET /v1/containers/{name}/logs`), because they
  can hold every tenant's data.

| Role | Scope | How you get it |
|------|-------|----------------|
| Instance `admin` | Some instance-level actions. On its own it gives **no** cross-tenant user administration. | Authelia's `admins` group on an SSO login. |
| Organization `owner` / `admin` | Their own organization and its sub-organizations. | Organization membership. |
| **Platform Admin** | Every tenant. | Only the ways described on this page. |

Platform Admin is never granted through the API, the web UI or an SSO group.
After the one-time bootstrap paths below, the server CLI is the only way to
grant or revoke it, and every change is written to the audit log.

## Fresh install: the first admin is bootstrapped

On a fresh self-hosted install, the **first account with the instance `admin`
role to sign in** becomes Platform Admin automatically. Normally that is the
`admin` account the setup CLI creates in Authelia, which is in the `admins`
group. This happens **once**:

- The grant is audited.
- After any account has ever been Platform Admin on the instance, the bootstrap
  is closed for good. Revoking every Platform Admin later doesn't re-open it.
- A self-registered account is created with the `user` role, so it is never a
  candidate on its own.
- An account that was linked to SSO from a pre-existing local account (see
  [Moving a local account to SSO](authentication.md#moving-a-local-account-to-sso))
  is never bootstrapped. If it really is yours, grant it with the CLI.

The first sign-in already carries the claim, so no refresh is needed. Grant
Platform Admin to anyone else with the [CLI](#grant-revoke-and-list).

## Upgraded install: the one-time setup code

An install that already had instance admins but **no** Platform Admin when it
was upgraded doesn't use the automatic bootstrap. An existing admin account
can't be proven to belong to the operator, so the claim needs proof of server
access instead: a one-time **setup code** that only the server CLI issues.

Until someone claims Platform Admin, the Users page and the other routes listed
above are unavailable. The instance tells you a code is needed:

- the api-gateway logs a warning at startup ("Platform Admin setup code
  required"), with the command to run;
- `bash setup.sh status` prints the same reminder.

Neither one issues or shows a code.

### Issue the code

On the server:

```bash
bash setup.sh platform-admin setup-code
```

The command prints a code in the form `XXXXXX-XXXXXX-XXXXXX`:

- It is printed to your terminal only. It is never written to the setup log or
  the gateway logs, and only a hash is stored. Don't paste it into shared logs
  or chats.
- It is valid for **24 hours** and works **once**.
- Running the command again issues a new code, and any earlier code stops
  working.
- If the instance doesn't need a code (it has a Platform Admin, or it is a
  fresh install that uses the automatic bootstrap), the command says so and
  issues nothing.

### Claim it

Sign in as an **active instance admin**, then submit the code. The setup CLI
points you to the **Users** page of the web client for this. Support for
entering the code there is being added to the client, so if your client
version doesn't show a setup-code prompt, call the API directly:

```http
GET /v1/auth/platform-admin/claim
Authorization: Bearer <access_token>
```

**Response:** `{ "setup_code_required": true }`. This is `true` only for an
active instance admin while the instance is waiting for a claim. It never
returns the code.

```http
POST /v1/auth/platform-admin/claim
Authorization: Bearer <access_token>
Content-Type: application/json

{ "code": "XXXXXX-XXXXXX-XXXXXX" }
```

**Response:** `{ "granted": true, "refresh_required": true }`. Your current
token doesn't carry the new claim yet, so call `POST /v1/auth/refresh` or sign
in again.

The code isn't case-sensitive, and dashes and spaces are ignored. A successful
claim consumes the code and permanently closes setup-code mode. Any later grant
or revoke goes through the [CLI](#grant-revoke-and-list).

| Status | Meaning |
|--------|---------|
| `200` | Platform Admin granted. Refresh your token. |
| `403` | The caller isn't an active instance admin, the code is wrong, no code is active, the code expired, or this account submitted too many wrong codes for this code. The `detail` says which. For the last three, issue a fresh code on the server. |
| `404` | No setup code is in use on this instance. |
| `409` | You are already a Platform Admin. |
| `429` | Too many attempts. Attempts are rate-limited per account and per client IP. |

Every refused attempt is audited (without the code). Each account can submit a
limited number of wrong codes per issued code, so one account can't use up the
code for everyone. A fresh code resets that limit.

A CLI grant also closes setup-code mode, so
`bash setup.sh platform-admin grant <username>` works on an upgraded install
too.

## Grant, revoke and list {#grant-revoke-and-list}

The server CLI is always available, on any install. Run it on the host, with
the stack running:

```bash
bash setup.sh platform-admin grant <username>
bash setup.sh platform-admin revoke <username>
bash setup.sh platform-admin list
```

- **`grant`** takes effect on the user's next sign-in or token refresh
  (`POST /v1/auth/refresh`). It also permanently closes the first-admin
  bootstrap and setup-code mode.
- **`revoke`** removes the flag and ends **all** of the user's sessions. Their
  existing tokens are refused within a few seconds, and they can't be
  refreshed. The user has to sign in again, and gets a token without the claim.
- **`list`** shows the current Platform Admins, with their email, and marks
  deactivated accounts.

Granting to someone who already has it, or revoking from someone who doesn't,
changes nothing and isn't audited. Each real change writes an audit entry and
prints the current Platform Admins, so you can tell them about the change.

!!! warning "Don't remove the last Platform Admin"
    `revoke` warns when no active Platform Admin is left, but doesn't stop
    you. After that, nobody can administer users across tenants, and the
    automatic bootstrap and the setup code don't come back. Recover with
    `bash setup.sh platform-admin grant <username>`.

    The API is stricter. `PATCH /v1/auth/users/{user_id}/status` refuses to
    deactivate the last active Platform Admin (`409`).

## SSO account linking

Signing in through SSO never takes over a local account just because the
usernames match. To move an existing local account to SSO, approve it on the
server **before** the user's first SSO login:

```bash
bash setup.sh sso-link approve <username>
bash setup.sh sso-link revoke <username>   # withdraw a pending approval
bash setup.sh sso-link list                # pending approvals
```

Linking a local account that was a Platform Admin **clears** Platform Admin.
Re-grant it with the CLI if that's what you want. See
[Authentication → Moving a local account to SSO](authentication.md#moving-a-local-account-to-sso)
for the full rules.

## Related pages

- [Authentication](authentication.md): sign-in, tokens, session revocation and
  roles.
- [Organizations & teams](organizations-teams.md): what Platform Admin manages.
- [Self-hosting](self-hosting.md) and [Upgrading](upgrades.md).
