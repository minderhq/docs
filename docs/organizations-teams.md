# Organizations & teams

Minder models multi-user access along **two independent axes**:

- An **organization** is the *tenant boundary* — the top-level container a user
  belongs to and can switch between.
- A **team** is a *grouping within a tenant context* — a way to gather users
  together, independent of which org owns them.

On top of these sit **three separate role systems** that do not imply one
another:

| Scope | Roles | Where it comes from |
|-------|-------|---------------------|
| **Instance** | `user`, `admin` | Set from Authelia's `groups` claim on every login (membership in the `admins` group → `role: admin`); see [Authentication](authentication.md#roles-partially-enforced) |
| **Organization** | `owner`, `admin`, `member` | Managed per-org via the organization member endpoints |
| **Team** | `team_admin`, `member` | Assigned when a team is created or a member is added |

!!! note "The axes are independent"
    Being in Authelia's `admins` group grants instance `role: admin` — it does
    **not** imply membership in any team or organization, and vice versa. A
    user's JWT carries a `teams` claim (a flat list of team ids) minted at
    login/OIDC-callback time and carried forward unchanged on refresh, matching
    `role`'s existing staleness-until-next-login behaviour.

## Organizations

An organization is the tenant boundary. A team is a grouping *within* the
caller's tenant context.

### Provisioning (instance-admin only)

Creating an organization is an **instance-admin-only** action. Provisioning is
atomic: it creates the org, its primary owner (`owner_user_id`, which defaults
to the caller), and a **default team** in one step. A taken `slug` is rejected
with a `409`.

### Switching the active organization

Any authenticated user can switch their **active tenant context** to an org they
belong to. Switching re-mints the caller's JWT with `active_tenant_id` set to
the target; the caller's *home* `tenant_id` is unchanged. It is **member-only** —
a caller who doesn't belong to the target org gets a `403`.

To populate an org switcher, a caller lists the orgs they belong to. A user with
no org membership yet gets an empty list, not an error.

### Membership & org invites

| Action | Who can do it |
|--------|---------------|
| List an org's members | A member of that org, or a platform admin (`403` for a non-member — org membership is itself tenant-scoped) |
| Add a member / change their `org_role` (upsert) | That org's own `owner`/`admin`, or an instance/platform admin |
| Remove a member | Same permission — **but see the last-owner guard below** |
| Invite a user by email (`{email, org_role, max_uses}`) | Same permission as member management |
| List / revoke org invites | Same permission |

!!! warning "Last-owner guard"
    Removing an org member is rejected with a **`409`** if it would remove the
    org's **last owner**. An organization can never be left ownerless.

Org invites are redeemed through the same token-based flow as team invites —
there is no separate organization-specific redemption endpoint (see
[Invites](#invites) below).

## Teams

### Creating a team

**Any authenticated user** can create a team; the creator automatically becomes
its `team_admin`. Team-name uniqueness is case-insensitive. Listing teams is
open to any authenticated user — a team roster isn't treated as sensitive, and
there is no "my teams only" filter.

### Membership & team invites

| Action | Who can do it |
|--------|---------------|
| View team details + full member list | Any authenticated user |
| Update a team's `name`/`description`, or delete it | That team's `team_admin`, or an instance admin |
| Add a member (`{user_id, team_role}`) | That team's `team_admin`, or an instance admin |
| Change a member's `team_role` | Same permission — **see the guard below** |
| Remove a member | That team's `team_admin`/instance admin, **or the member removing themselves** |
| Issue a team invite (`{email, team_id, team_role}`) | That team's `team_admin`, or an instance admin |
| List / revoke a team's invites | Same permission (revoke is restricted to `team_admin`/instance admin, not the invited email) |

Team invites default to a **7-day expiry**.

!!! warning "Last-`team_admin` guard"
    A team can never be left with zero `team_admin`s. Demoting or removing the
    **sole remaining** `team_admin` is rejected with a **`409`** — unless the
    caller is an instance admin. The same guard applies to a self-removal.
    Deleting the team outright is a separate, allowed path.

## Users (instance-admin)

Instance-level user management is **instance-admin-only**:

- List users (paginated).
- Change a user's instance `role` — valid values are `user` / `admin`
  (anything else is a `422`).

!!! note "OIDC-linked accounts are read-only"
    Changing the role of an **OIDC-linked account** returns a **`409`**:
    Authelia's `groups` claim overwrites `role` on every login, so the change
    would silently be undone. For those users, change group membership in
    Authelia instead. In the UI, such accounts show their role read-only.

!!! warning "No last-admin guard here"
    Unlike teams and organizations, instance user-role management has **no**
    last-admin guard — an admin can demote the last other admin, or themselves.

## Invites

All invites — team and organization — use a single **token-based redemption
flow**:

1. **Look up** an invite by its token. This lookup is **public** (no auth);
   possession of the unguessable token is the credential. A `pending` invite
   past its expiry reports as `expired`.
2. **Register or log in** first, through the normal auth or OIDC flow — there is
   no separate signup branch built into redemption.
3. **Redeem** while authenticated. Redemption adds the *authenticated caller* to
   the invite's team (or org) at the invite's role and marks it `accepted`. It
   is atomic — the membership insert and the status update commit in one
   transaction.

## Data export & import (self-service)

An org's own **`owner`/`admin`** (or an instance/platform admin) can export and
re-import that org's structured data as a single portable JSON bundle, and a
team's **`team_admin`** can do the same for the team's own subset. The bundle
downloads as an attachment (`minder-export-<slug>.json` /
`minder-team-export-<team>.json`); import it back into the **same** org/team, or
into a fresh one on **another instance** — the migration case.

Every export is filtered to the acting org/team and every imported row is
stamped with the *target* tenant, so a caller can only ever move **their own**
data — an admin of one org can't aim an export or import at another.

**What an org bundle contains**

- Organization metadata (name, slug, description).
- The **member list**, referenced by email (a stable cross-instance identity,
  not the source instance's numeric user ids).
- **Knowledge-base metadata** and **pipelines**.
- The org-**global** plugin config (per-user config overrides are excluded).

A **team** bundle is narrower: team metadata, the team roster, and only the
**team-shared** (`visibility: team`) knowledge bases and pipelines. Private and
personally-shared KBs (their owners' data, not the team's) and org-global plugin
config are deliberately out of scope for a team export.

!!! warning "Metadata only — document and vector payloads are not included yet"
    This first slice moves **structured metadata only**. A knowledge base's
    actual **document blobs and vector embeddings are not exported** — imported
    KBs arrive **empty**, and you re-ingest their documents on the target. The
    bundle records this (`notes.documents_excluded`). Streaming large KB
    payloads is a background-job feature planned as follow-up work, not part of
    this synchronous export.

**Conflict handling on import.** Importing into an org/team that already holds
colliding ids or names is resolved by a `policy` you choose — all three are
**non-destructive** (nothing existing is ever overwritten):

| Policy | Behaviour |
|--------|-----------|
| `skip` (default) | Keep the existing rows; drop the colliding incoming ones |
| `rename` | Import each collision as a **fresh copy** — a new id and a de-duplicated name (`… (imported)`), with pipeline→KB references re-pointed at the re-keyed KBs |
| `fail` | Abort the whole import if anything collides (all-or-nothing) |

A destructive `overwrite`/`merge` is intentionally **not** offered. The import
returns a report of what was written plus every conflict it resolved
(skipped / renamed / unresolved).

!!! note "Imported members never keep an elevated role"
    Members in a bundle are always added as plain `member` (org) / `member`
    (team), **never** at a claimed `owner`/`admin`/`team_admin` — otherwise a
    crafted bundle could mint privileged members, bypassing the normal
    add-member guards. Restore any elevated roles afterwards through the usual
    member-management flow. An email with no matching user on the target
    instance can't be provisioned and is reported as unresolved rather than
    invented.

These run through the gateway API (`/v1/organizations/{id}/export|import`,
`/v1/teams/{id}/export|import`) — see the [API reference](api-reference.md) for
the exact routes, bodies, and status codes. Every export/import is recorded in
the [Audit Log](using-minder.md#teams-organizations).

## Partial RBAC — read this first

!!! danger "Role enforcement is partial"
    Role checks currently cover a **specific** set of actions — including the
    organizations/teams/RBAC surface described on this page (team management,
    the last-`team_admin` guard, instance user management) plus a handful of
    admin-only actions elsewhere. **Everywhere else that requires auth still
    only checks "is there a valid JWT," not "does this JWT's role allow it."**

    Do **not** build workflows that assume broader per-role restrictions are
    enforced than what is documented. See
    [Authentication → Roles (partially enforced)](authentication.md#roles-partially-enforced)
    and [Security](security.md) for the full caveat.

## In the UI / via the API

=== "In the UI"

    The control-plane web client exposes these as dedicated pages — **Teams**,
    **Users** (admin-only), **Organization**, **All Organizations**
    (admin-only), **Invitation**, and an **Audit Log** (admin-only) of
    privileged actions. See
    [Using Minder](using-minder.md#teams-organizations).

=== "Via the API"

    Every action here maps to a gateway-native route under `/v1/auth/users`,
    `/v1/teams`, `/v1/invites`, and `/v1/organizations`. For the full endpoint
    tables, request/response details, roles, and guards, see the
    [API reference](api-reference.md).
