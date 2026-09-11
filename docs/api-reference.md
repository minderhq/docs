# API reference

The Minder platform exposes RESTful APIs from **8 core FastAPI microservices**,
fronted by a reverse proxy (TLS termination and routing). This page enumerates
every route as wired in each service, grouped by service.

All examples use the API Gateway base URL `http://localhost:8000` for a local,
self-hosted deployment. Adjust the host/port for your own deployment.

!!! note "Self-hosted deployment"
    This reference describes a self-hosted deployment. The API Gateway implements
    JWT authentication with bcrypt-hashed credentials and Redis-backed rate
    limiting; single sign-on is handled by Authelia (OIDC). See
    [Authentication](authentication.md) for the full auth model.

## Overview

### Core API services

| Service | Container | Port | Summary |
|---------|-----------|------|---------|
| API Gateway | `minder-api-gateway` | 8000 | JWT+bcrypt auth, Redis rate-limit, httpx proxy to every other core service |
| Plugin Registry | `minder-plugin-registry` | 8001 | Manifest install, health loop, service discovery, AI-tool aggregation, tool discovery/execution, per-plugin licensing |
| Marketplace | `minder-marketplace` | 8002 | Discovery/search/featured, license tiers, dependency graph (Neo4j) |
| Orchestrator | `minder-orchestrator` | 8003 | AI-agent orchestration: OpenWebUI tool-server discovery/execution, chat completions with a multi-turn tool loop |
| RAG Pipeline | `minder-rag-pipeline` | 8004 | Knowledge bases, doc ingest, Qdrant vectors; Standard/Conversational/HyDE/Self-RAG/auto/corrective/RAPTOR RAG via the query `method` field, adaptive `rerank`/`compress` flags, and `hybrid`/`parent_context` retrieval strategies (see `GET /capabilities`) |
| Model Management | `minder-model-management` | 8005 | Ollama list/pull/delete/test (some endpoints are placeholders) |
| TTS / STT | `minder-tts-stt` | 8006 | Text-to-speech (Piper offline default, WAV; gTTS fallback, MP3), speech-to-text |
| Graph-RAG | `minder-graph-rag` | 8008 | spaCy NER, Neo4j knowledge-graph construction and retrieval |

Minder also ships a separate web client (a React/Vite single-page app) as its
management UI. It is a static SPA, not a FastAPI backend — it has no
`/docs`/OpenAPI schema and is not part of the API surface documented here. It is
the browser UI for the plugin-config, RAG knowledge-base/pipeline/query,
marketplace catalog, and model-management/AI-tools endpoints below.

**Conventions used below**

- `ANY` = the route accepts `GET, POST, PUT, DELETE, PATCH`.
- `{path:path}` = a catch-all path segment (everything after the prefix is forwarded verbatim).
- Ports are the host-published ports; internally each service also sits behind the reverse proxy.
- **`GET /health`** returns `200` (`healthy`/`degraded`) when the service is serviceable and **`503`** (`unhealthy`) when a *critical* dependency (its Postgres/Redis/Qdrant/Neo4j/Ollama) is unreachable. Each body carries a `status` field and a per-dependency `checks` map plus service-specific fields.

## Interactive documentation

Every FastAPI service serves Swagger UI, ReDoc, and the raw OpenAPI spec on its own port:

```
http://localhost:<port>/docs          # Swagger UI (interactive)
http://localhost:<port>/redoc         # ReDoc
http://localhost:<port>/openapi.json  # OpenAPI schema
```

Ports: `8000` (gateway), `8001` (plugin-registry), `8002` (marketplace),
`8003` (orchestrator), `8004` (rag-pipeline), `8005` (model-management),
`8008` (graph-rag). The `tts-stt` service is internal-only by default (no host
port), so its `/docs` is reachable from inside the service network / via the
reverse proxy rather than at `localhost:8006`.

The interactive `/docs` page for each service is the **authoritative,
always-current** source for request/response schemas. The tables below enumerate
every route as wired in code; for exact field-level payloads use `/docs`.

## API Gateway — `http://localhost:8000`

Central entry point: authentication, rate limiting, and request proxying (the
OpenWebUI function-calling bridge lives in the orchestrator service, reached
through here at `/v1/ai/*` — see its own section below).

### Authentication

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/auth/register` | Create a user — body `{username, email, password}`. `username` must be 1-50 chars matching `^[A-Za-z0-9_.-]+$`, `email` a valid address, `password` ≥8 chars. No `role` field — every self-registered account is created as `"user"`; admin is only ever granted via Authelia OIDC group membership. **201** on success, **409** if the username/email exists, **422** on a bad body |
| POST | `/v1/auth/login` | Obtain a JWT — body `{username, password}` → `{access_token, token_type, expires_in, user}` (**401** on bad creds) |
| POST | `/v1/auth/refresh` | Refresh an access token (bearer token in the `Authorization` header) → `{access_token, token_type, expires_in}` |
| GET | `/v1/auth/oidc/login` | Start the Authelia SSO redirect — the platform's single-login entry point. Sets short-lived `oidc_state`/`oidc_nonce` httponly cookies (CSRF/replay defense), then 302s to Authelia's `/api/oidc/authorization` |
| GET | `/v1/auth/oidc/callback` | Authelia's redirect target — exchanges the auth code, verifies the ID token, provisions/loads the user, mints a Minder JWT, and 302s to the client's `/auth/callback#token=...` (fragment, never logged server-side) |

See [Authentication](authentication.md) for how browser login, OIDC, and the
JWT model fit together.

### User & team management (`/v1/auth/users`, `/v1/teams`, `/v1/invites`)

Organizations/teams/RBAC — gateway-native routes (not proxied to a backing
service). A user's JWT carries a `teams` claim (a flat list of team ids) minted
at login/OIDC-callback time and carried forward unchanged on refresh (matching
`role`'s existing staleness-until-next-login behavior).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/auth/users` | List users, paginated (`limit`/`offset`). **Instance admin-only** |
| PATCH | `/v1/auth/users/{user_id}/role` | Change a user's instance `role` (`user`/`admin`, **422** otherwise). **Instance admin-only.** **409** for an OIDC-linked account — Authelia's `groups` claim overwrites `role` on every login, so this would silently be undone. No last-admin guard exists here (an admin can demote the last other admin, or themselves) |
| POST | `/v1/teams` | Create a team — body `{name, description?}`. Any authenticated user; creator becomes its `team_admin` |
| GET | `/v1/teams` | List all teams on the instance, paginated (`limit`/`offset`) — any authenticated user (a team roster isn't sensitive); there is no "my teams only" filter |
| GET | `/v1/teams/{team_id}` | Team details + full member list |
| PATCH | `/v1/teams/{team_id}` | Update `name`/`description` — that team's `team_admin` or an instance admin |
| DELETE | `/v1/teams/{team_id}` | Delete a team — same permission. Team name uniqueness is case-insensitive |
| POST | `/v1/teams/{team_id}/members` | Add a member — body `{user_id, team_role: "member"\|"team_admin"}` — that team's `team_admin` or an instance admin |
| PATCH | `/v1/teams/{team_id}/members/{user_id}` | Change a member's `team_role` — same permission. **409** if this would demote the team's sole remaining `team_admin` (unless the caller is an instance admin) |
| DELETE | `/v1/teams/{team_id}/members/{user_id}` | Remove a member — that team's `team_admin`/instance admin, **or the member removing themselves**. Same last-`team_admin` 409 guard applies to a self-removal |
| GET | `/v1/teams/{team_id}/export` | Export the team's own subset as a downloadable JSON bundle — team metadata, the roster, and the **team-shared** (`visibility='team'`) KBs/pipelines only (private/shared KBs and org-global plugin config are out of scope; document/vector payloads are **excluded**). That team's `team_admin` or an instance/platform admin. Audited |
| POST | `/v1/teams/{team_id}/import` | Import a team bundle into this team — body `{bundle, policy?}` (`skip`/`rename`/`fail`, same semantics and **422** conditions as the org import). Imported rows are stamped to this team; members join as plain `member`. Same permission; audited. A team bundle can't be imported through the org endpoint or vice-versa (distinct bundle formats) |
| POST | `/v1/invites` | Issue a team invite — body `{email, team_id, team_role}`. That team's `team_admin` or an instance admin. Invite defaults to a 7-day expiry |
| GET | `/v1/invites?team_id=` | List a team's invites, paginated — that team's `team_admin` or an instance admin |
| POST | `/v1/invites/{invite_id}/revoke` | Revoke a pending invite — same permission (restricted to `team_admin`/instance admin, not the invited email) |
| GET | `/v1/invites/by-token/{token}` | Look up invite info by token (email, team, role, effective status — a `pending` row past `expires_at` reports as `expired`) — **public**, no auth. Possession of the unguessable token is the credential |
| POST | `/v1/invites/by-token/{token}/redeem` | Redeem an invite — adds the **authenticated** caller to the invite's team at its `team_role` and marks it `accepted`. There is no separate signup branch: a new person registers/logs in via the normal `/v1/auth/*` or OIDC flow first, then redeems while authenticated. Atomic — membership insert and status update commit in one transaction |

Team membership and OIDC/Authelia group membership are independent axes — being
in Authelia's `admins` group grants instance `role: admin`, it does not imply
membership in any team, and vice versa.

### Organizations (`/v1/organizations`)

Multi-tenant org provisioning — a separate axis from teams: an org is the tenant
boundary, a team is a grouping *within* the caller's tenant context.
Gateway-native routes, not proxied.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/organizations` | Provision a new org — atomically creates the org, its primary owner (`owner_user_id`, defaults to the caller), and a default team. **Instance admin-only.** **409** on a taken `slug` |
| POST | `/v1/organizations/switch` | Switch the caller's active tenant context — body `{organization_id}`. Re-mints the JWT with `active_tenant_id` set to the target; the caller's home `tenant_id` is unchanged. Any authenticated user, but **member-only**: **403** if they don't belong to the target org |
| GET | `/v1/organizations/mine` | The orgs the **caller** belongs to (for the org switcher) — any authenticated user. Empty list for a user with no org membership yet, not an error |
| GET | `/v1/organizations` | List all organizations with member counts, paginated (`limit`/`offset`). **Instance admin-only** (platform-operator view) |
| GET | `/v1/organizations/{org_id}/members` | List an org's members — a member of that org, or a platform admin. **403** for a non-member (org membership is itself tenant-scoped) |
| POST | `/v1/organizations/{org_id}/members` | Add a member or change their `org_role` (upsert) — body `{user_id, org_role}`. That org's own `owner`/`admin`, or an instance/platform admin |
| DELETE | `/v1/organizations/{org_id}/members/{user_id}` | Remove a member — same permission. **409** if this would remove the org's **last owner** |
| POST | `/v1/organizations/{org_id}/invites` | Invite a user to the org by email — body `{email, org_role, max_uses}`. Same permission as member management |
| GET | `/v1/organizations/{org_id}/invites` | List an org's pending/spent invites, paginated. Same permission |
| POST | `/v1/organizations/{org_id}/invites/{invite_id}/revoke` | Revoke a pending org invite. Same permission. **404** if the invite doesn't belong to this org |
| GET | `/v1/organizations/{org_id}/export` | Export the org's structured data as a downloadable JSON bundle (org metadata, member list by email, KB **metadata**, pipelines, org-global plugin config — large document/vector payloads are **excluded**). Same permission as member management (org `owner`/`admin` or instance/platform admin). Served as a file attachment; audited |
| POST | `/v1/organizations/{org_id}/import` | Import a bundle into this org — body `{bundle, policy?}`, `policy` one of `skip` (default) / `rename` / `fail` (**422** otherwise, or on a bad format/version or a `fail`-policy collision). All policies are non-destructive; imported members always join as plain `member`. Returns a report of what was written and every conflict resolved. Same permission; audited |

Org-invite redemption reuses the already-documented
`POST /v1/invites/by-token/{token}/redeem` — there is no separate
`/v1/organizations/...` redemption endpoint.

### Proxy routes

Forwarded over the internal service network via httpx to the backing service.
Most routes use the shared proxy client's 30s default timeout;
model-management, rag-pipeline, tts-stt, and graph-rag (marked **long-timeout**
below) get a 300s ceiling instead — their backend work (Ollama model pulls,
document ingestion/embedding, audio synthesis/transcription, NER + Neo4j writes)
can legitimately run well past 30s.

| Method | Path | Target |
|--------|------|--------|
| GET | `/v1/plugins` | plugin-registry (list) |
| ANY | `/v1/plugins/{path:path}` | plugin-registry |
| GET | `/v1/bundles` | plugin-registry (list, mirrors the `/v1/plugins` GET/wildcard split) |
| GET/POST | `/v1/bundles/{path:path}` | plugin-registry's bundle control-plane (enable/disable/reconcile) — writes require JWT |
| ANY | `/v1/rag/{path:path}` | rag-pipeline (prefix maps to the service root) — **long-timeout** |
| ANY | `/v1/conversations/{path:path}` | rag-pipeline conversation-history bridge — a top-level prefix (not nested under `/v1/rag/*` or a pipeline id) since a conversation isn't pipeline-scoped. `GET /v1/conversations/mine` lists the caller's own conversations |
| GET/POST | `/v1/models` | model-management `/models` (list / pull) — **long-timeout** |
| ANY | `/v1/models/{path:path}` | model-management `/models/{path}` — the gateway adds the `models/` resource segment, so use `/v1/models/{id}` — **long-timeout** |
| ANY | `/v1/marketplace/{path:path}` | marketplace (prefix forwarded as-is, matching plugin-registry) |
| ANY | `/v1/graph/{path:path}` | marketplace's plugin dependency/conflict/recommendation graph — a second, disjoint route namespace the same service exposes |
| GET/POST | `/v1/tts`, `/v1/tts/{path:path}` | tts-stt (writes require JWT). `POST /v1/tts` returns binary WAV/MP3 audio, not JSON — the proxy passes non-JSON bodies through raw instead of force-decoding them. **long-timeout** |
| GET/POST | `/v1/stt`, `/v1/stt/{path:path}` | tts-stt (writes require JWT) — `POST /v1/stt` is a multipart audio upload. **long-timeout** |
| ANY | `/v1/graph-rag/{path:path}` | graph-rag's own `/v1/*` routes (`extract`, `construct-graph`, `retrieve`, `entity-context`, `graph/search`, `graph/stats`, `graph/documents`, `graph/document/{id}`, `graph/correlate`, `graph/correlations`), which are unprefixed at the service — the gateway adds the `graph-rag/` segment so this doesn't collide with marketplace's own `/v1/graph/*` proxy above (writes require JWT). **long-timeout** |
| GET | `/v1/tools` | plugin-registry (tool discovery, list) |
| GET/POST | `/v1/tools/{path:path}` | plugin-registry (tool detail, `.../execute`, license `validate`) — a deliberately separate prefix from `/v1/plugins/{path:path}` above so the registry's plugin-adjacent APIs don't collide at the gateway (writes require JWT; `execute` is also independently JWT-gated inside plugin-registry itself) |
| GET/POST/PATCH | `/v1/licensing/{path:path}` | plugin-registry per-plugin licensing (required-tier lookup/validate/update) — writes require JWT |
| GET/POST | `/v1/ai/{path:path}` | orchestrator (see its own section below) — **not** gated by the blanket writes-require-JWT rule the other rows carry: each route enforces its own auth, or deliberately none. **long-timeout** |

### Ops

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Gateway health + downstream dependency status (no auth) |
| GET | `/v1/status` | Fans out to all 8 core services' own `/health` over the internal service network (no single service's `/health` is reachable from a browser directly). Never fails even if every downstream is unreachable — each entry reports `reachable: false` instead |
| GET | `/v1/containers/{name}/logs` | Proxies to Plugin Registry's `GET /v1/containers/{name}/logs?tail=` (JWT-gated there, not here — see Plugin Registry's Containers section below) |
| GET | `/metrics` | Prometheus metrics |

**Authentication:** JWT (HS256) with bcrypt-hashed credentials. Send
`Authorization: Bearer <token>` on protected routes. See
[Authentication](authentication.md).
**Rate limiting:** Redis-backed, 60-second window, **fail-open** (requests are
allowed if Redis is unreachable).

```bash
# Health
curl -s http://localhost:8000/health | jq '.status'

# Register + login
curl -X POST http://localhost:8000/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "email": "admin@example.com", "password": "..."}'

TOKEN=$(curl -s -X POST http://localhost:8000/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "..."}' | jq -r '.access_token')

# Proxied call
curl -s http://localhost:8000/v1/plugins -H "Authorization: Bearer $TOKEN" | jq '.'
```

## Plugin Registry — `http://localhost:8001`

Plugin registration, discovery, and lifecycle management. Plugins are
**manifest-based** — there is **no arbitrary code execution** (security by
design). The registry runs a 60-second health loop, stores service-discovery
data in Redis, and auto-syncs with the marketplace.

### Plugins

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/plugins` | List registered plugins (`GET /plugins` is a legacy alias) |
| GET | `/v1/plugins/{plugin_name}` | Plugin details |
| POST | `/v1/plugins/install` | Install a plugin from its manifest (fixed handlers only, admin-only) |
| DELETE | `/v1/plugins/{plugin_name}` | Uninstall a plugin (admin-only) |
| POST | `/v1/plugins/{plugin_name}/enable` | Enable a plugin (admin-only) |
| POST | `/v1/plugins/{plugin_name}/disable` | Disable a plugin (admin-only) |
| POST | `/v1/plugins/{plugin_name}/collect` | Trigger a data-collection run |
| GET | `/v1/plugins/{plugin_name}/health` | Plugin health status |
| GET | `/v1/plugins/{plugin_name}/analysis` | The plugin's `analyze()` output, returned verbatim (schema is plugin-defined; the registry does not reshape it). 404 unknown / 403 disabled / 503 not-running |
| GET | `/v1/plugins/{plugin_name}/actions/{action}` | Invoke a **read-only** action, unauthenticated — only names in the plugin's `READ_ONLY_ACTIONS` (a declared subset of `ACTIONS`); query params are passed as keyword args |
| POST | `/v1/plugins/{plugin_name}/actions/{action}` | Invoke a plugin write/execute action (JWT-gated; only names in the plugin's `ACTIONS`) |
| GET | `/v1/plugins/{plugin_name}/config` | Config schema + effective values, secrets masked (JWT-gated) |
| PUT | `/v1/plugins/{plugin_name}/config` | Update config: validate → persist → apply live, no restart (JWT-gated) |
| GET | `/v1/plugins/ai/tools` | Aggregated AI-tool definitions across all plugins |

### Webhooks

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/plugins/reload-webhook` | Re-register a plugin's webhook routes (admin-only) |
| POST | `/v1/force-webhooks` | Force re-registration of all webhook routes (JWT-gated; unversioned `/force-webhooks` kept as a deprecated alias) |
| POST | `/webhook/{path:path}` | Generic inbound webhook / event trigger |

### Service discovery

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/services/register` | Register a microservice for discovery (admin JWT or service token) |
| GET | `/v1/services` | List registered services |
| GET | `/v1/services/{service_name}` | Service details |
| GET | `/v1/services/{service_name}/health` | Check a registered service's health (admin JWT or service token) |
| DELETE | `/v1/services/{service_name}` | Unregister a service (admin JWT or service token) |
| GET | `/v1/proxy` | List services that can be proxied |
| ANY | `/v1/proxy/{service_name}/{path:path}` | Dynamic proxy to a registered service (admin JWT or service token) |

### Bundles

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/bundles` | The bundle model: each capability bundle, whether it's enabled, its claimed services (each with active/orphaned status and its pinned Docker `image` — `null` for a locally-built service), and the platform-wide orphaned-services list. `503` if the compose file isn't mounted |
| POST | `/v1/bundles/{name}/enable` | Enable a bundle (admin-gated). Persists intent and starts already-materialised claimed containers via a least-privilege docker-socket proxy — it cannot *create* new containers, so a never-materialised service comes back as `pending_create` until the next host converge |
| POST | `/v1/bundles/{name}/disable` | Disable a bundle (admin-gated); stops its claimed containers, same persistence model as enable |
| POST | `/v1/bundles/reconcile` | Re-apply the persisted enable-state to running containers (admin-gated) — start/stop drift correction without changing intent |

### Containers

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/containers/{name}/logs` | Recent stdout/stderr for one of the 8 core services (`?tail=`, default 200, max 2000), JWT-gated (log output can carry stack traces or an accidentally-logged secret). `name` is checked against a fixed allowlist before building a container name. `404` unknown service or container not running, `503` if the socket proxy itself is unreachable |

Fetched over the same least-privilege docker-socket proxy `/v1/bundles` uses,
extended with a GET-only, no-exec/attach/create logs rule. Docker's logs API
returns a multiplexed stream whenever the container isn't a tty (which every
Minder container is) — this endpoint demuxes it server-side so the response is
plain `{stream, text}` lines, not raw bytes.

### Backups (`/v1/backups`)

A job-queue front end for the host-level backup/restore tooling — **not** a
Docker orchestrator (this router never talks to Docker directly). It reads/writes
small JSON job files in a bind-mounted directory; a host-side cron job does the
real work. All endpoints are admin-only. Reachable through the gateway
(`GET/POST /v1/backups`, `GET/POST /v1/backups/{path:path}`).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/backups` | List existing backup archives |
| POST | `/v1/backups` | Enqueue a backup job (**202**, returns the job record) |
| GET | `/v1/backups/jobs` | List recent backup/restore jobs (most recent 50) |
| GET | `/v1/backups/jobs/{job_id}` | Get one job's status/output (404 if unknown) |
| POST | `/v1/backups/{name}/restore` | Enqueue a restore job (**202**). Destructive — beyond admin-only gating, the body must echo `{"confirm_filename": "<name>"}` exactly matching the archive being restored |

### Tools (`/v1/tools`)

Tool discovery and execution. Execution dispatches the plugin action
**in-process** (the registry already runs the plugins, so there is no HTTP hop
back to itself). Reachable through the gateway (`GET /v1/tools`,
`GET/POST /v1/tools/{path:path}`).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/tools` | Discover all executable tools (paginated) |
| GET | `/v1/tools/{tool_name}` | Tool details |
| POST | `/v1/tools/{tool_name}/execute` | Execute a tool in-process (license-validated against the JWT `sub`, JWT-gated) |
| GET | `/v1/tools/plugins/{plugin_id}/tools` | Tools for one plugin (by marketplace UUID) |
| POST | `/v1/tools/validate` | Validate a user's tier against a tool |

### Licensing (`/v1/licensing`)

Per-plugin **required** license tier. The tier lives on the registry's own
`plugins` table row (`license_tier`/`license_key`); the real per-user license
store stays in the marketplace. Reachable through the gateway
(`GET/POST/PATCH /v1/licensing/{path:path}`).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/licensing/plugins/{plugin_name}/license/tier` | Get a plugin's required license tier |
| POST | `/v1/licensing/plugins/{plugin_name}/license/validate` | Validate a license key against the plugin's required tier |
| PATCH | `/v1/licensing/plugins/{plugin_name}/license` | Update a plugin's required tier/key (**admin**/service-gated) |

### Ops

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Service health + plugin/service counts |
| GET | `/metrics` | Prometheus metrics |

First-party module plugins ship with the platform and are loaded by the registry
on startup (see `GET /v1/plugins`). For writing and publishing your own, see the
[Plugins](plugins/index.md) documentation.

## Marketplace — `http://localhost:8002`

Plugin/tool discovery, licensing, and dependency management. Catalog data lives
in PostgreSQL; the dependency/conflict graph is backed by **Neo4j**.

### Catalog (`/v1/marketplace`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/marketplace/plugins` | List catalog plugins — filterable; paginated via `limit`/`offset` (canonical; `page`/`page_size` accepted but deprecated). Response carries both `total`/`limit`/`offset` and `page`/`page_size`/`total_pages` |
| GET | `/v1/marketplace/plugins/search` | Full-text search (same `limit`/`offset` pagination) |
| GET | `/v1/marketplace/plugins/featured` | Featured plugins |
| GET | `/v1/marketplace/plugins/{plugin_id}` | Plugin details |
| POST | `/v1/marketplace/plugins` | Create a catalog entry (called by plugin-registry) |
| PUT | `/v1/marketplace/plugins/{plugin_id}` | Update catalog metadata (partial; name/display_name/description/author/pricing_model/base_tier/featured/requires_services). 404 if unknown, 422 if empty. `status` is **not** updatable here (**422**, pointing at `/submissions/*`); a developer editing their own listing is limited to `draft`/`rejected` and cannot set `featured` |

### Installation management (`/v1/marketplace/plugins`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/marketplace/plugins/{plugin_id}/install` | Install from the catalog. 409 once the caller hits `MAX_PLUGINS_PER_USER` (default 100) currently-installed plugins — re-enabling an already-installed plugin doesn't count toward the cap |
| DELETE | `/v1/marketplace/plugins/{plugin_id}/uninstall` | Uninstall |
| POST | `/v1/marketplace/plugins/{plugin_id}/enable` | Enable |
| POST | `/v1/marketplace/plugins/{plugin_id}/disable` | Disable |
| GET | `/v1/marketplace/plugins/{plugin_id}/installations` | List installations for a plugin, all users (admin-gated) |
| GET | `/v1/marketplace/installations/me` | List the *authenticated user's* installed plugins, across the whole catalog, with plugin metadata inlined — a disjoint prefix, not nested under `/plugins/`, since `GET /plugins/{plugin_id}` is registered first and would swallow a literal segment like `installed` as `{plugin_id}` |

### Ratings & reviews (`/v1/marketplace/plugins/{plugin_id}/ratings`)

End-user star ratings on an already-approved plugin — distinct from the
admin submission-approval workflow below. The aggregate (`rating_average`,
`rating_count`) is also mirrored onto each plugin's catalog entry.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/marketplace/plugins/{plugin_id}/ratings` | The plugin's aggregate plus its individual reviews, newest first — paginated via `limit` (1–100, default 20) / `offset`. Public read. **404** if the plugin is unknown |
| POST | `/v1/marketplace/plugins/{plugin_id}/ratings` | Submit or edit the caller's own rating — body `{rating: 1–5, review_text?}` (`review_text` ≤ 4000 chars). **Install-gated:** **403** unless the caller has an installation row for this plugin (a since-uninstalled one still counts). One review per user per plugin — a second submission **upserts** (edits) the existing one rather than erroring. **201**; **404** if the plugin is unknown |

### Submission / review workflow (`/v1/marketplace/submissions`)

The human side of the catalog: a developer's listing is created as a `draft` via
`POST /v1/marketplace/plugins`, then moves `draft → submitted → in_review →
approved` (or `rejected`, and back to `submitted` on resubmit) before it becomes
publicly visible. First-party auto-synced listings (`origin='first_party'`) skip
this — they're created already `approved`. Every accepted transition is validated
against the state machine (**409** on an illegal move) and appends an audit row.
Developer actions are scoped to the caller's own submissions by the JWT `sub`;
reviewer actions are admin-only.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/marketplace/submissions/mine` | List the caller's own submissions (any status), newest first — scoped to the JWT `sub`; paginated via `limit`/`offset` same as the catalog above |
| POST | `/v1/marketplace/submissions/{plugin_id}/submit` | Developer (or admin/service): move an own `draft`/`rejected` submission to `submitted`. A non-owner gets **404** (same as unknown id) |
| GET | `/v1/marketplace/submissions?status=submitted` | **Admin** review queue for a given status (default `submitted`), oldest-first; `origin='submitted'` only; paginated via `limit`/`offset` |
| POST | `/v1/marketplace/submissions/{plugin_id}/claim` | **Admin**: `submitted → in_review` (records the reviewer) |
| POST | `/v1/marketplace/submissions/{plugin_id}/approve` | **Admin**: `in_review → approved` (publicly visible via the catalog read paths) |
| POST | `/v1/marketplace/submissions/{plugin_id}/reject` | **Admin**: `submitted`/`in_review` → `rejected`; `notes` **required** (**422** if missing) |
| POST | `/v1/marketplace/submissions/{plugin_id}/archive` | **Admin/service**: `approved → archived` (delist) |

### AI-tool catalog (`/v1/marketplace/ai`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/marketplace/ai/tools` | List AI tools (filter by tier / active) |
| GET | `/v1/marketplace/ai/tools/{tool_name}` | Tool details |
| GET | `/v1/marketplace/ai/plugins/{plugin_id}/tools` | Tools for one plugin |
| POST | `/v1/marketplace/ai/sync` | Sync AI tools from a plugin manifest |
| DELETE | `/v1/marketplace/ai/plugins/{plugin_id}/tools` | Remove a plugin's tools |

### Licensing (`/v1/marketplace/licenses`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/marketplace/licenses` | List / inspect licenses |
| GET | `/v1/marketplace/licenses/lookup` | Admin/service-only: the tier of a specific user's active license for a specific plugin — `?user_id=&plugin_id=` → `{tier, active}`. Backs plugin-registry's tool-tier enforcement |
| POST | `/v1/marketplace/licenses/validate` | Validate a license against a tier |
| POST | `/v1/marketplace/licenses/activate` | Activate a license (admin/service-gated) |

### Dependency graph (`/v1/graph`, Neo4j-backed)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/graph/dependencies/{plugin_id}` | Resolve a plugin's dependencies |
| POST | `/v1/graph/dependencies` | Register/update dependency edges (**admin**/service-gated; the graph is a shared platform-wide control) |
| GET | `/v1/graph/conflicts/{plugin_id}` | Detect conflicts |
| POST | `/v1/graph/recommendations` | Recommend related plugins |
| GET | `/v1/graph/health` | Dependency-graph subsystem health |

### Ops

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Service health |
| GET | `/metrics` | Prometheus metrics |

## Orchestrator — `http://localhost:8003`

AI-agent orchestration: OpenWebUI tool-server discovery, tool execution dispatch,
and chat completions with a multi-turn tool loop. Reached through the gateway at
`/v1/ai/*` (see the Proxy routes table above); the external paths and behavior
are stable regardless of where the code lives.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/ai/functions/definitions` | Aggregated AI-tool (function) definitions from all plugins, in OpenAI function schema. Deliberately open (no auth) — OpenWebUI's Tool Server integration has no way to attach a Minder JWT |
| GET | `/v1/ai/tools/openapi.json` | OpenAPI 3.x spec of Minder's read-only (GET-only) plugin tools, consumable directly as an OpenWebUI "Tool Server." Deliberately open, same reason as above — only ever includes GET-method tools, never mutating/admin ones |
| POST | `/v1/ai/functions/{function_name}` | Execute a named AI tool; proxied to the plugin's endpoint (forwards the caller's JWT, if any), returned in OpenAI function-result format. Not independently auth-gated at this route — the downstream plugin endpoint enforces its own auth, so a mutating tool call without a valid JWT is rejected there |
| POST | `/v1/ai/chat/completions` | Chat via Ollama. **Requires a valid Minder JWT** (this endpoint calls Ollama directly with nothing else in the request path to gate it, unlike the tool-discovery/execution routes above). Plugin function-calling is **opt-in** via `"minder_tools": true` (orchestrator offers plugin tools **plus one synthesized `ask_<pipeline-name>` tool per RAG pipeline the caller owns** (owner-scoped) — the chat↔RAG bridge — **plus a `find_correlations` tool over graph-rag's correlation engine** — executes the model's `tool_calls` forwarding the caller's JWT, and feeds results back). Without the flag it's a plain Ollama `/api/chat` passthrough |

### Ops

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Service health |

## RAG Pipeline — `http://localhost:8004`

Chunking, embedding, retrieval, and generation. Documents are embedded into
**Qdrant**; embeddings and generation run through **Ollama**. The live query
endpoint supports Standard and Conversational RAG (set `conversation_id` for
multi-turn history), plus **HyDE**, **Self-RAG**, **auto** (decision engine),
**corrective**, and **RAPTOR** RAG — all selectable via the `method` field on
`POST /pipeline/{id}/query`
(`standard`/`hyde`/`self_rag`/`auto`/`corrective`/`raptor`). Orthogonal `rerank`
and `compress` flags, and the `hybrid` (dense+BM25) and `parent_context`
(small-to-big) retrieval strategies, are also wired. `GET /capabilities` reports
what's active on the host. For the method-by-method detail, see
[RAG methods](rag-methods.md).

| Method | Path | Description |
|--------|------|-------------|
| POST | `/initialize` | Initialize the Ollama client / warm the pipeline |
| GET | `/capabilities` | What's actually live on this host (rerank backend, hybrid/parent-context availability, etc.) — see [RAG methods](rag-methods.md) |
| GET | `/decision-stats` | Cumulative analytics for the `method="auto"` routing engine — total decisions, strategy/complexity distributions, mean confidence. `available:false` when the auto engine isn't initialized (e.g. Ollama unreachable). In-memory (resets on restart) |
| POST | `/knowledge-bases` | Create a knowledge base (`name` required, `description` optional; pick embedding + LLM model) |
| GET | `/knowledge-bases` | List knowledge bases — paginated via `limit`/`offset`; returns the shared `{items, total, limit, offset}` envelope |
| GET | `/knowledge-bases/{kb_id}` | Get a single knowledge base (404 if unknown) |
| PATCH | `/knowledge-bases/{kb_id}` | Update a KB's **mutable metadata** — `name` / `description` / `llm_model` / `visibility` / `team_id` — in place, WITHOUT touching its documents or vectors (JWT-gated; 404 if unknown). `embedding_model` and the chunk params are immutable (changing them would invalidate the stored vectors). Setting `visibility: "team"` requires a `team_id` (from the request or the KB's existing one) **and** caller membership in that team (checked against the JWT `teams` claim, 403 otherwise); switching to any other visibility clears `team_id`. Team visibility grants **read only** — updating/deleting the KB or its documents stays owner-only regardless of team membership |
| DELETE | `/knowledge-bases/{kb_id}` | Delete a KB — drops its Qdrant collection + PostgreSQL row (404 if unknown) |
| POST | `/knowledge-bases/{kb_id}/upload` | Upload a document into a KB — PDF, Word (`.docx`/`.doc`), spreadsheets, images (OCR), audio/video (transcribed), and [many more formats](ingestion.md); the type is detected from content, not the extension, and an unrecognized file is rejected **415**. **413** over `MAX_UPLOAD_SIZE_MB` (default 50MB) — also capped a level up by the gateway's own `MAX_PROXY_BODY_SIZE_MB` (150MB). Returns **503** if the embedding backend is unreachable — the doc is NOT indexed (no silent zero-vector). Response includes a `document_id`, one per upload call. Optional form field `build_tree: bool` opts into RAPTOR tree construction for this document (see [RAG methods](rag-methods.md)) |
| GET | `/knowledge-bases/{kb_id}/documents` | List documents in a KB, one entry per upload — not per chunk (404 if the KB is unknown). Returns the shared `{items, total, limit, offset}` envelope |
| GET | `/knowledge-bases/{kb_id}/documents/{document_id}/chunks` | List a single document's stored chunks (`chunk_index` + `text`) — lets a caller tell a bad extraction/OCR apart from a retrieval/generation issue. RAPTOR tree-summary nodes excluded. Paginated via `limit`/`offset`; returns the shared `{items, total, limit, offset}` envelope. 404 if the KB or document is unknown |
| DELETE | `/knowledge-bases/{kb_id}/documents/{document_id}` | Delete a single document's chunks/vectors, leaving the rest of the KB intact (404 if the KB or document is unknown) |
| POST | `/pipeline` | Create a RAG pipeline over one or more knowledge bases. The creator (JWT `sub`) is recorded as `owner_id` for owner-scoping |
| GET | `/pipeline` | List RAG pipelines — paginated via `limit`/`offset`; returns the shared `{items, total, limit, offset}` envelope. Optional `?owner_id=` filters to that owner's pipelines (+ legacy null-owner ones), used by the orchestrator to owner-scope the `ask_<pipeline>` chat tools |
| GET | `/pipeline/{pipeline_id}` | Get a single RAG pipeline (404 if unknown) |
| PATCH | `/pipeline/{pipeline_id}` | Update a pipeline's `name` and/or `knowledge_base_ids` in place (JWT-gated; 404 if the pipeline or a supplied KB is unknown) |
| DELETE | `/pipeline/{pipeline_id}` | Delete a pipeline (referenced KBs are left intact; 404 if unknown) |
| POST | `/pipeline/{pipeline_id}/query` | Query a pipeline (retrieval + generation). **Owner-scoped**: only the creator (or the service/admin principals) may query it — a non-owner → **403**. Legacy pipelines with no recorded owner stay open. This protects both a direct call and the OpenWebUI `ask_<pipeline>` chat tool (the gateway forwards the end user's JWT) |
| POST | `/v1/pipeline/{pipeline_id}/conversations/{conversation_id}/share` | Mark a conversation shared so any authenticated user holding the `conversation_id` can continue it. Owner-only (the true first-writer) — a non-owner → **403** |
| GET | `/v1/conversations/mine` | List the caller's own conversation history — `{items, total, limit, offset}` envelope, newest first, owner-scoped. Reached through the gateway at `/v1/conversations/mine` |
| GET | `/health` | Service health |
| GET | `/metrics` | Prometheus metrics |

The singular `/knowledge-base[...]` forms still work as deprecated, hidden
aliases.

**Document identity:** each upload call gets its own `document_id`, stamped on
every chunk's Qdrant payload — `source` (filename) alone can't tell two separate
uploads of the same filename apart. Chunks uploaded before this existed have no
`document_id`; the list/delete endpoints fall back to grouping those by `source`,
with a synthetic `legacy:<filename>` id.

```bash
# Create a knowledge base, then upload a document into it
KB=$(curl -s -X POST http://localhost:8004/knowledge-bases \
  -H 'Content-Type: application/json' \
  -d '{"name":"My Docs","description":"my documents"}' | jq -r '.id')

curl -X POST "http://localhost:8004/knowledge-bases/$KB/upload" -F "file=@doc.pdf"

# Query: create a pipeline over the KB, then query it
PIPE=$(curl -s -X POST http://localhost:8004/pipeline \
  -H 'Content-Type: application/json' \
  -d "{\"name\":\"my-pipe\",\"knowledge_base_ids\":[\"$KB\"]}" | jq -r '.pipeline_id')
curl -X POST "http://localhost:8004/pipeline/$PIPE/query" \
  -H 'Content-Type: application/json' -d '{"question":"What is in my docs?","top_k":3}'
```

## Model Management — `http://localhost:8005`

Model lifecycle over the Ollama runtime.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/models` | List local models (live from Ollama) — paginated via `limit`/`offset`; returns the shared `{items, total, limit, offset}` envelope |
| POST | `/models` | Pull a model (admin-gated) — body `{"model_id": "..."}`. **201** on a fresh pull, **200** if it already exists. `model_id` rejects a custom registry host (**422**) — only the default Ollama library, optionally namespaced (e.g. `namespace/model:tag`), is supported. Concurrent pulls are capped via `MAX_CONCURRENT_MODEL_PULLS` (default 1) — excess concurrent requests queue rather than racing the shared Ollama volume |
| GET | `/models/{model_id}` | Model details, including a `capabilities` list (e.g. `tools`) sourced from Ollama's own model metadata — **not** a guarantee the model reliably uses tools when offered them (**404** if unknown) |
| DELETE | `/models/{model_id}` | Delete a local model (admin-gated; **404** if unknown) |
| POST | `/models/{model_id}/test` | Quick test-prompt inference — body `{"prompt": "..."}`. JWT-gated (any logged-in user, not admin-only — deliberate self-service intent) and rate-limited (5/min) |
| GET | `/health` | Service health |
| GET | `/metrics` | Prometheus metrics |

### Cloud model providers (`/v1/model-providers`)

The credential vault behind the [Cloud Providers](using-minder.md#cloud-providers-platformproviders)
page: per-org, admin-managed connections to an OpenAI-compatible or Anthropic
vendor — or a self-hosted OpenAI-compatible server (e.g. **vLLM**) flagged
`is_local`. Management routes require the instance `admin` role **or** an
`owner`/`admin` of the acting org. Stored keys are encrypted at rest; reads
only ever return a masked form, never the plaintext.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/model-providers/presets` | Suggested "add provider" configurations (OpenAI, Anthropic, **vLLM (self-hosted)**, generic OpenAI-compatible) to pre-fill the create form — each carries `adapter`, `is_local`, a `base_url_placeholder`, `requires_api_key`, and help text. Static, credential-free metadata |
| POST | `/v1/model-providers` | Create a provider — body `{adapter: "openai_compatible"\|"anthropic", name, base_url?, api_key, is_local?}` (`adapter` **422** otherwise; `api_key` required and non-empty even for a keyless vLLM server). `is_local: true` marks self-hosted/local compute. **201** |
| GET | `/v1/model-providers` | List this org's providers (keys masked) |
| GET | `/v1/model-providers/{id}` | One provider (masked). **404** if not in the caller's tenant |
| PATCH | `/v1/model-providers/{id}` | Update `name`/`base_url`/`api_key`/`enabled`/`is_local` (partial) |
| DELETE | `/v1/model-providers/{id}` | Delete a provider. **204**; **404** if unknown |
| POST | `/v1/model-providers/{id}/test` | Verify stored credentials with one minimal real completion against the adapter's first catalog model → `{ok, detail}`. Works even on a disabled provider. Rate-limited 5/min **unless** the row is `is_local` |
| POST | `/v1/model-providers/generate` | Run one completion through an enabled provider — body `{model_id: "remote:<provider_id>:<catalog_id>", prompt, context?, temperature?}`. Any authenticated tenant member (not admin-only). Rate-limited 5/min **unless** the provider is `is_local` (local compute has no per-call cost) |

## TTS / STT — via the gateway (`/v1/tts`, `/v1/stt`)

!!! note "Reached through the API Gateway, not a host port"
    Unlike the other core APIs, `minder-tts-stt` is internal-only by default:
    nothing binds `:8006` on the host in the default deployment. The gateway
    proxies `/v1/tts`/`/v1/stt` to it over the service network; a host `:8006`
    only appears when the optional `tts-stt-router` failover profile is active.

Speech synthesis and recognition. ~12 languages supported; **Turkish is the
default**. Every route below also has a legacy unversioned alias (e.g. `/tts`
alongside `/v1/tts`) kept for compatibility — new callers should use the `/v1/`
path.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/tts` | Text-to-speech — Piper offline (WAV) by default, gTTS fallback (MP3) for non-bundled languages. Binary body (no `response_model`); language/duration reported via `X-Language`/`X-Duration` headers. `text` capped at `TTS_MAX_TEXT_LENGTH` (default 5000 chars, 422 past that). Optional `voice` field (e.g. `"male"`/`"female"`) picks among that language's bundled Piper voices — silently falls back to the language's default on an unknown/unsupported value rather than erroring; ignored entirely for gTTS-only languages |
| GET | `/v1/tts/languages` | Supported TTS languages |
| GET | `/v1/tts/voices?language=` | Voice choices for `language` — only ones whose `.onnx` is actually bundled on this deployment. Empty for a gTTS-only language (no per-voice selection) or a language with no bundled Piper voice. English currently offers `default`/`female`/`male`; every other bundled language has just `default` |
| POST | `/v1/stt` | Speech-to-text (Google backend) — multipart `file` upload + `language` form field. **Locale-qualified codes** (`tr-TR`, `en-US`, …) are required; a bare `tr` is rejected. Audio capped at `STT_MAX_AUDIO_SIZE_MB` (default 25MB, 413 past that) |
| GET | `/v1/stt/languages` | Supported STT languages — a **different code set than TTS above** (`tr-TR` not `tr`); don't reuse a TTS language code here or vice versa |
| GET | `/health` | Service health |
| GET | `/metrics` | Prometheus metrics |

Reachable through the gateway at `/v1/tts`/`/v1/tts/{path:path}` and
`/v1/stt`/`/v1/stt/{path:path}` (writes require JWT) — see the API Gateway's
Proxy routes table above. STT mic recordings are normalized to WAV before
transcription, so non-WAV uploads (mp3, ogg, …) work the same way, not just
recordings.

## Graph-RAG — `http://localhost:8008`

Entity extraction and knowledge-graph construction/retrieval, backed by
**Neo4j**. Every route below also has a legacy unversioned alias kept for
compatibility — new callers should use the `/v1/` path.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/extract` | spaCy NER entity extraction from text |
| POST | `/v1/construct-graph` | Build a Neo4j knowledge graph from documents/entities. Atomic (one transaction) — a mid-failure can't leave a half-built graph. Re-posting the same `document_id` is a FULL REPLACE of that document's edges (old `RELATES_TO`/`MENTIONS` dropped before rebuild, orphaned entities cleaned up) — not a simple upsert, so omitting a relationship on re-POST removes it. Document metadata (title/source/metadata) upserts via `COALESCE` — an omitted field keeps its previous value rather than blanking |
| POST | `/v1/retrieve` | Graph-based retrieval over entity relationships |
| POST | `/v1/entity-context` | Retrieve context / neighbors around an entity |
| POST | `/v1/graph/search` | Free-text search the graph for entities whose `text` or `label` matches `query` (case-insensitive `CONTAINS`); returns `{text, label, description}` per hit, `limit` 1–50 (default 5) |
| GET | `/v1/graph/stats` | Graph overview — total node / relationship / document / entity counts + the per-NER-label entity distribution (`entity_types`); confirms a `construct-graph` populated the graph. Scoped to the caller's own graph (owner_id from the JWT); requires a valid JWT |
| GET | `/v1/graph/documents` | List the Document nodes (id / title / source / created_at / entity_count) in the caller's own graph, newest first. Requires a valid JWT |
| DELETE | `/v1/graph/document/{document_id}` | Delete a document's graph — its relationships, Document node, and orphaned entities (entities still referenced by another of the caller's own documents are kept). Scoped to the caller's own graph — a document_id owned by a different caller is treated as absent, not a permission error. Idempotent: returns 200 with zero counts if the document is absent for this caller |
| POST | `/v1/graph/correlate` | Run the **correlation engine** over the caller's graph — derives edges the raw NER graph misses: `CO_OCCURS` (corpus co-mention), `SIMILAR_TO` (embedding neighbour), `SAME_AS` (entity resolution), `CORRELATES_WITH` (temporal signal↔signal over InfluxDB series, carrying lead-lag `lag`/`direction` and Granger `granger_p_value`/`granger_causal_direction`), `HAS_SIGNAL` (entity↔signal), and `IS_A` (taxonomy — deterministic tiers 1+2 always run; an optional tier 3 proposes an LLM-classified `IS_A_CANDIDATE` pending human review, never auto-published). Body `{correlators?}` selects which (default all); returns per-correlator edge counts (rate-limited 5/min per caller). Owner-scoped (correlates over `shared ∪ self`, never across tenants) |
| GET | `/v1/graph/correlations` | Read an entity's correlations — `?entity=<text>&limit=&max_hops=&include_centrality=` → `{found, co_occurring, similar, same_as, correlated_signals, indirect, is_a, entity_centrality_score}`. `correlated_signals` surfaces the quantitative signals the entity's own signal moves with (`Entity→HAS_SIGNAL→Signal→CORRELATES_WITH→Signal`). `max_hops` (2-3) additionally returns `indirect`: entities reachable via a bounded multi-hop path with no direct edge, each item tagged `hops`/`via`. `is_a` gives the entity's taxonomy category. `include_centrality=true` attaches a 0-1 normalized degree-centrality `centrality_score` to every result plus a top-level `entity_centrality_score`. Owner-scoped |
| GET | `/v1/graph/taxonomy/review-queue` | List pending tier-3 taxonomy candidates (LLM self-consistency guesses, never auto-published) awaiting human review. Owner-scoped: a regular caller sees only their own entities' candidates; admin/service sees every tenant's queue |
| POST | `/v1/graph/taxonomy/review-queue/{candidate_id}/approve` | Approve one pending candidate — promotes it to a live `IS_A` edge (`method='llm_tier3'`), now surfaced by `/v1/graph/correlations`' `is_a` |
| POST | `/v1/graph/taxonomy/review-queue/{candidate_id}/reject` | Reject one pending candidate — no `IS_A` edge is written, and it's never re-attempted (the entity keeps its tier-2 coarse classification) |
| GET | `/health` | Service health |
| GET | `/metrics` | Prometheus metrics |

Reachable through the gateway at `/v1/graph-rag/{path:path}` — the gateway
forwards to this service's own `/v1/*` paths (e.g. a call to
`POST /v1/graph-rag/extract` reaches `POST /v1/extract` on graph-rag). The
distinct `graph-rag/` segment avoids a path collision with marketplace's own
`/v1/graph/*` dependency-graph proxy (a different subsystem that shares the
prefix by coincidence). Writes require JWT. For how graph RAG differs from text
RAG, see [RAG methods](rag-methods.md).

## Error handling

Errors follow the standard FastAPI shape:

```json
{ "detail": "Human-readable error message" }
```

| Status | Meaning |
|--------|---------|
| 200 | Success |
| 400 | Bad request — semantically invalid input the route rejected (e.g. undecodable audio, an unknown config key) |
| 401 | Unauthorized (missing/invalid JWT) |
| 403 | Forbidden — authenticated but not permitted (insufficient license tier, a disabled account, or a disabled plugin) |
| 404 | Not found — unknown resource, **or a malformed id** (e.g. a non-UUID `plugin_id` is treated as "no such plugin", not a server error) |
| 422 | Validation error — FastAPI request-body/param validation (wrong type, out-of-range, empty required field) |
| 429 | Rate limited (Redis-backed, 60s window; fail-open) |
| 500 | Internal server error — a genuine bug |
| 503 | Service unavailable — a required downstream/backend (DB, Ollama, Qdrant, …) is unreachable; **retryable** |

**Conventions (enforced platform-wide):**

- **Malformed or not-found input is always a 4xx, never a 5xx.** Validation
  happens at the request boundary (typed bodies, min-length/bounded fields,
  existence checks) so bad input can't fail deep in a driver and surface as a
  500. A malformed id 404s like an absent one; an empty/undecodable payload
  422/400s rather than 500.
- **5xx messages are sanitized.** `503`/`500` bodies carry a generic message —
  the raw driver/exception string is **never** returned to the caller in
  production. `503` means "backend down, retry"; `500` means "a real bug, check
  logs".

## Plugin system

Plugins are **manifest-based** and support no arbitrary code execution by design.
New actions must be implemented as fixed handlers in the codebase.

Lifecycle (as implemented):

```
register() → initialize() (READY) → health_check() (60s loop)
           → collect_data() (hourly or manual) → shutdown()
           + analyze()
```

Plugins advertise **AI tools** for Ollama function-calling. A module plugin
declares an `AI_TOOLS` class attribute (a manifest plugin uses its `ai_tools`
key), each tool being:

```json
{ "name": "...", "description": "...", "parameters": { "type": "object", "properties": {} }, "action": "...", "required_tier": "community" }
```

`action` maps to `POST /v1/plugins/<plugin>/actions/<action>`. Optional
`required_tier` (`free` | `community` | `pro` | `enterprise`, default
`community`) sets the license tier the marketplace records for the tool — the
plugin-registry enforces it at execution time (a user below that tier gets a 403
"License tier too low"). An absent or unrecognized value falls back to
`community`. `GET /v1/plugins/ai/tools` aggregates these into OpenAI/Ollama tool
defs; drive the end-to-end loop via `POST /v1/ai/chat/completions` with
`"minder_tools": true` (see the API Gateway section).

Plugins can write to any storage backend (postgres, qdrant, neo4j, minio,
influxdb) and publish async events through rabbitmq. See the
[Plugins](plugins/index.md) documentation for how to write and publish one.

## Monitoring

FastAPI services expose Prometheus metrics on `/metrics`; Prometheus scrapes them
and Grafana visualizes the results. See [Service access](service-access.md) for
the full observability port map, and [Monitoring](monitoring.md) for dashboards
and alerting.
