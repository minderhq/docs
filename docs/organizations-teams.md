# Organizations & teams

Minder models multi-user access along **two independent axes**:

- An **organization** is the *tenant boundary* — the top-level container a user
  belongs to and can switch between.
- A **team** is a *grouping within an organization* — a way to gather that
  organization's users together. A team belongs to the active organization of
  the person who creates it.

On top of these sit **three separate role systems** that do not imply one
another:

| Scope | Roles | Where it comes from |
|-------|-------|---------------------|
| **Instance** | `user`, `admin` | Set from Authelia's `groups` claim on every login (membership in the `admins` group → `role: admin`); see [Authentication](authentication.md#roles-and-permissions) |
| **Organization** | `owner`, `admin`, `member`, plus [custom roles](#custom-roles-and-permission-groups) | Managed per-org via the organization member endpoints |
| **Team** | `team_admin`, `member` | Assigned when a team is created or a member is added |

!!! note "The axes are independent"
    Being in Authelia's `admins` group grants instance `role: admin` — it does
    **not** imply membership in any team or organization, and vice versa. A
    user's JWT carries a `teams` claim (a flat list of team ids) minted at
    login/OIDC-callback time; `teams` and `role` are re-derived from the
    database on every token refresh.

## Organizations

An organization is the tenant boundary. A team is a grouping *within* the
caller's tenant context.

### Provisioning

Provisioning is atomic: it creates the org, its primary owner (`owner_user_id`,
which defaults to the caller), and a **default team** in one step. A taken `slug`
is rejected with a `409`. Who may create a top-level organization:

- A **Platform Admin** can create one for any owner.
- Anyone else must already own an active top-level organization whose plan allows
  another, and becomes the new organization's owner. The owner's
  `top_level_orgs` and `owned_free_orgs` limits apply: over a hard limit the
  request is refused (`403`, `reason: entitlement`), and over a soft limit it is
  allowed and audited. An account that owns no organization can't create one this
  way.
- An account that belongs to no organization can create its own **personal
  workspace** instead (`POST /v1/organizations/personal`), when
  `SELF_SERVE_PERSONAL_ORGS` allows it (by default, in `open` registration mode).
  A personal organization becomes a standard one only when a Platform Admin
  converts it (`POST /v1/organizations/{org_id}/convert`, one-way); until then
  its owners can't issue invites that create new accounts in `invite` or
  `sso_only` mode.

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
| List an org's members | A holder of `org.members.view` on it, an ancestor org's manager, or a platform admin (`403` otherwise — org membership is itself tenant-scoped) |
| Add a member / change their `org_role` (upsert) | That org's own `owner`/`admin`, an ancestor org's `org.members.manage` holder, or an instance/platform admin. Apart from a platform admin, a caller can only change an existing member or add a user the org already manages; users from elsewhere join by accepting an invite |
| Remove a member | Same permission — **but see the last-owner guard below** |
| Invite a user by email (`{email, org_role, max_uses}`), or a list (`/invites/bulk`, always `207` with a per-email outcome) | Same permission as member management, subject to the org's [policy](#organization-policy-seats-and-limits) |
| List / revoke org invites | Same permission |
| Suspend / unsuspend one member | A holder of `org.members.suspend`. The member row stays and is still listed, but their effective permissions drop to zero. You can't suspend yourself or the last owner |

!!! warning "Last-owner guard"
    Removing, demoting or suspending an org member is rejected with a **`409`** if
    it would leave the org with no active **owner**. An organization can never be
    left ownerless. Changing the role of an existing owner or admin needs
    owner-level authority (`org.roles.manage`).

Org invites are redeemed through the same token-based flow as team invites —
there is no separate organization-specific redemption endpoint (see
[Invites](#invites) below).

### Sub-organizations and suspension

Organizations nest. `POST /v1/organizations/{org_id}/suborganizations` creates a
child (it needs `org.suborgs.create` on the parent, and a plan that allows
sub-organizations: otherwise `403` with `reason: entitlement`, even for a
Platform Admin). A sub-organization has its parent's kind. `POST
/v1/organizations/{org_id}/move` re-parents an organization or detaches it to top
level, and needs `org.suborgs.manage` on both the organization and the new
parent. `GET /v1/organizations/{org_id}/subtree` returns the whole descendant tree
with a usage and entitlement summary. A permission held on an ancestor can be
delegated down to its descendants, and a role defined in an organization can be
used throughout its sub-organizations.

An organization can be **suspended** and unsuspended as a whole, and a single
member can be suspended inside it (see the table above). Teams nest and suspend
the same way, with `team.subteams.manage`, `team.members.suspend` and the matching
routes.

### Custom roles and permission groups

Besides the built-in `owner`, `admin` and `member`, an organization can define
**custom roles** made of permission keys (for example `org.members.manage`,
`org.settings.manage`, `org.billing.manage`, `org.audit.view`), and nestable
**permission groups** that bundle permissions. Create them with
`POST /v1/organizations/{org_id}/roles` and `.../permission-groups`. A role gives
a member permissions only once it is assigned to them
(`POST .../members/{user_id}/roles`, optionally with `expires_at`), and granting
one needs `org.members.manage` plus holding every permission in the role yourself.
`POST .../members/bulk-roles` assigns one role to a list of members. See what a
member can actually do with
`GET /v1/organizations/{org_id}/members/{user_id}/effective-permissions`, and
what you can do with `GET /v1/me/permissions`.

### Organization policy, seats and limits {#organization-policy-seats-and-limits}

`GET`/`PATCH /v1/organizations/{org_id}/policy` holds the organization's
onboarding policy. Changing it needs `org.settings.manage`, and a knob's
effective value is the stricter of the org's and its ancestors':

- `invite_policy`: `admins` (default) or `members` (members may also invite
  existing accounts, single-use, at the member role);
- `allow_account_creating_invites` (default on);
- `allowed_email_domains`: exact-match domains that people must use to join;
- `team_admins_may_create_accounts`: needs a non-empty domain list;
- `require_2fa`: stored only, not yet enforced.

A plan can lock a knob (`403`, `reason: entitlement`). `GET .../seats` reports
seats used and reserved against the plan, and `GET .../limits` lists every
onboarding limit with the source that applied (tier default, operator override,
licence). A Platform Admin can override one limit for one organization with
`PUT`/`DELETE .../limits/{key}`.

### Access requests and the waiting room {#access-requests-and-the-waiting-room}

An account that belongs to no organization lands in a waiting room, where it can
ask for access (`POST /v1/access-requests`): `join` an organization that accepts
requests (an organization opts in with `PUT .../access-requests/settings`), or
through an admin's email address. A member can also ask their admins for an
`invite` for someone else, to `team_join` a team, to install a plugin
(`plugin_install`), to enable a model (`model_request`) or for a plan
(`plan_request`). Requests wait for a decision and are never approved
automatically; the admins are emailed when [email](email.md) is on. They decide
from their **Requests** inbox (`POST .../access-requests/{id}/approve` or `/deny`),
the first decision closes the request for every recipient, and approving a
`join` or `invite` goes through the same admission checks as any other join
(allowed domains, seats, a suspended organization). Approving a `plugin_install`,
`model_request` or `plan_request` only records the decision: an admin then
installs or enables it. `PUT .../access-requests/routing/{kind}` chooses who
receives each kind.

## Teams

### Creating a team

**Any active member of an organization** can create a team in it; the creator
automatically becomes its `team_admin`. The team belongs to the creator's
**active organization**. Team-name uniqueness is case-insensitive. A team can have
a parent (`parent_team_id`), which needs `team.subteams.manage` on the parent.
Listing teams shows the teams of your active organization (switching
organization changes the list); a Platform Admin sees every organization's teams.
A team you can't list is a `404`, the same as one that doesn't exist.

### Membership & team invites

| Action | Who can do it |
|--------|---------------|
| View team details | Anyone who can list the team. The roster comes only with membership of the team, `org.members.view` in its organization, or Platform Admin, and member emails only with team admin, `org.members.manage` or Platform Admin |
| Update a team's `name`/`description`, or delete it | That team's `team_admin`, or an instance admin |
| Add a member (`{user_id or email, team_role}`) | That team's `team_admin`, or an instance admin. Apart from a platform admin, only a user who is already in the team's organization, or whom it manages |
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

## Users (Platform Admin)

Cross-tenant user management is **Platform Admin only** (see
[Authentication → Roles & permissions](authentication.md#roles-and-permissions);
[Platform Admin](platform-admin.md) explains how an operator gets the role):

- List users across every tenant (paginated, optionally for one `org_id`, each
  with its org memberships).
- Change a user's instance `role` — valid values are `user` / `admin`
  (anything else is a `422`).
- Deactivate or reactivate an account, and reset its password. Org admins
  holding `org.members.manage` can do both for accounts their organization
  manages — see
  [Authentication → Password management](authentication.md#password-management).

!!! note "OIDC-linked accounts are read-only"
    Changing the role of an **OIDC-linked account** returns a **`409`**:
    Authelia's `groups` claim overwrites `role` on every login, so the change
    would silently be undone. For those users, change group membership in
    Authelia instead. In the UI, such accounts show their role read-only.

!!! note "Last-admin guard"
    Demoting the last remaining active instance admin is rejected with a
    **`409`**. A demotion revokes the demoted user's sessions.

## Invites

All invites — team and organization — use a single **token-based redemption
flow**:

1. **Look up** an invite by its token. This lookup is **public** (no auth);
   possession of the unguessable token is the credential. A `pending` invite
   past its expiry reports as `expired`.
2. **Register or log in** first, through the normal auth or OIDC flow. In
   `invite` registration mode, a new local account registers *with* the invite
   (`POST /v1/auth/register` with `invite_token`): the account is created and the
   invite consumed in one transaction, and the account gets the invite's
   organization as its home organization. An invite can create an account only if
   its organization isn't suspended and its issuer is a Platform Admin, or an
   owner or admin of that organization while it is `standard` (or the mode is
   `open`); otherwise it can still be redeemed from an existing account.
3. **Redeem** while authenticated. Redemption adds the *authenticated caller* to
   the invite's team (or org) at the invite's role and marks it `accepted`, and
   returns a fresh token with an up-to-date `teams` claim. It is atomic — the
   membership insert and the status update commit in one transaction.

## Deleting an organization

!!! danger "Deletion is irreversible"
    Deleting an organization has **no grace period and no undo**. The
    organization, its members, teams, invites and public chat endpoints are
    removed right away. A background job then deletes the org's data from every
    other service: knowledge bases, documents and their vectors, pipelines,
    plugin configuration, and the org's knowledge-graph data. If you might need
    any of it later, [export the org](#data-export-import-self-service) first,
    and remember that the export leaves out document and vector payloads.

Deletion takes two safeguards that apply to everyone, including instance
admins:

- The request must pass `confirm=<organization slug>`, matching the org's slug
  exactly. Without it the call returns **`400`** and nothing is deleted.
- The **`default`** organization can never be deleted (**`409`**).

A successful delete returns **`202`** with the id and status of its purge job.
The response also includes a `not_yet_purged` list naming any data store that
deletion doesn't clean up yet, so you know what (if anything) is left behind.
By default, child organizations are detached to top level;
`reparent_children_to=<org id>` moves them under another org instead.

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

| Policy | Behavior |
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
    The organizations, teams and roles routes on this page check **permission
    keys** (for example `org.members.manage`) or a role, route by route. A handful
    of admin-only actions elsewhere do too. **Everywhere else that requires auth still
    only checks "is there a valid JWT," not "does this JWT's role allow it."**

    Do **not** build workflows that assume broader per-role restrictions are
    enforced than what is documented. See
    [Authentication → Roles & permissions](authentication.md#roles-and-permissions)
    and [Security](security.md) for the full caveat.

## In the UI / via the API

=== "In the UI"

    The control-plane web client exposes these as dedicated pages — **Teams**,
    **Organization**, **People** (members, invites and allowed email domains),
    **Requests**, **Users** and **All Organizations** (both Platform Admin only),
    **Invitation**, and an **Audit Log** of privileged actions. See
    [Using Minder](using-minder.md#teams-organizations).

=== "Via the API"

    Every action here maps to a gateway-native route under `/v1/auth/users`,
    `/v1/teams`, `/v1/invites`, `/v1/organizations` and `/v1/access-requests`. For the full endpoint
    tables, request/response details, roles, and guards, see the
    [API reference](api-reference.md).
