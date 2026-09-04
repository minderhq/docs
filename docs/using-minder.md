# Using Minder

Once an instance is running ([Self-hosting](self-hosting.md)), almost everything
you do day-to-day happens in the **control-plane UI** — a browser app served
alongside the platform (`minder-client`). Chat itself is a separate app,
[OpenWebUI](https://github.com/open-webui/open-webui); this UI is knowledge
bases, pipelines, the graph, plugins, models, voice, and admin.

!!! note "Browsing vs. logging in"
    Most pages are readable without an account — you can look around before
    deciding to self-host. Making changes (creating a knowledge base, pulling a
    model, installing a plugin) needs you to be logged in; some admin actions
    need an admin account specifically. Each page says which applies.

## Signing in

Two ways in, and only one works until you have a real domain:

- **Local account** — a username/password form, with a link to switch to
  "Create an account" (username + email + password) if you don't have one yet.
  This is the path that works over `localhost` or a bare LAN IP.
- **SSO (Authelia)** — a "Sign in with SSO" button, shown *only* once the
  deployment sets `VITE_OIDC_LOGIN_URL` (a real Traefik hostname + TLS). Over
  plain `localhost`/LAN access it's hidden rather than shown as a dead link,
  since the OIDC redirect can't resolve `*.minder.local` without real DNS.

If an SSO sign-in fails partway (denied consent, expired code), you're sent
back to the login form with the reason shown — it doesn't fail silently.

## The home dashboard

Logged in or not, the home page (`/`) is a live snapshot, not a static
sitemap: your knowledge base / pipeline / enabled-bundle / model counts (each
a link to that section), a health strip, and a single **"Suggested next
step"** card that points at whatever's next in the RAG setup flow (create a
knowledge base → upload a document → build a pipeline → ask it) — the same
logic that drives the step-by-step banner on the RAG pages themselves, so the
two never disagree about what to do next.

## Knowledge & RAG

This is the core loop: get documents in, then ask questions over them.

### Knowledge Bases (`/rag`)

Create a knowledge base (name + optional description) and upload documents to
it — this is the data your pipelines search over. Upload runs as a background
job: the response comes back immediately as `processing`, and the page polls
until it flips to `completed` (with chunk/vector counts) or `failed`. Each
document can be expanded to inspect its actual stored chunks — useful for
telling a bad extraction/OCR apart from a retrieval problem. Browsing is open
to everyone; creating, uploading, or deleting needs login.

### Pipelines (`/rag/pipelines`)

A **pipeline** is what you actually ask questions against: it points at one
or more knowledge bases and answers using Minder's own retrieval (RAG), not
OpenWebUI's separate, disconnected "Knowledge" feature. Create one over at
least one knowledge base, then open **Ask**. The page also surfaces a
reference of the available retrieval methods and stats on the auto-router's
recent decisions.

### Ask (`/ask`)

Pick a pipeline and chat with it. Conversations persist — reopening the page
can continue your most recent thread instead of starting blank, and there's a
"New chat" action once a thread has messages.

### Knowledge Graph (`/rag/graph`)

A different retrieval paradigm from vector-search pipelines: spaCy extracts
entities and relationships from your text, Neo4j stores them, and this page
lets you build the graph, explore it, review entity correlations, manage a
document's graph-visibility (private/shared/team), and delete a document from
the graph. It's a separate graph from the plugin-dependency graph shown on
the Marketplace pages — same Neo4j instance, unrelated data.

### Conversations (`/rag/conversations`)

Every thread you've started in **Ask**, across any pipeline, most-recently-active
first — reopen one to continue it instead of hunting for it inside a single
pipeline. It's your own history, so it needs login.

### Entity Merge Review (`/rag/entity-merges`)

Scans the knowledge graph for one real-world entity showing up as more than one
node (often across orgs) and proposes merges. Cross-org merges are
**dual-control** — each side's owner must approve before the merge applies, so
one tenant can't unilaterally fold another's data together. Owner/admin work.

### Taxonomy Review (`/rag/taxonomy-review`)

A curation queue for the graph's **entity-type taxonomy**: pending suggestions
(e.g. a newly-seen entity type) to approve or reject, so the graph keeps a
coherent schema instead of every extraction inventing its own types. Admin work.

## Extending: plugins, tools, bundles

Three related-but-distinct surfaces, each with a browsable "available" view
and a "what's actually active" view:

| Available (browse/install) | Installed (manage) |
|---|---|
| `/plugins/available` | `/plugins/installed` |
| `/ai-tools/available` | `/ai-tools/installed` |
| `/bundles/available` | `/bundles/installed` |

- **Plugins** — browsing is open to everyone; installing, enabling, disabling,
  uninstalling, or editing a plugin's config needs login (installs are
  per-user).
- **AI Tools** — the function-calling tools an LLM can actually call.
  "Available" is Marketplace's durable catalog (includes tools from plugins
  that aren't currently running, and can lag slightly behind "Installed");
  "Installed" is what's callable *right now*, from plugins Plugin Registry
  currently has loaded. Both are read-only pages — there's nothing to log in
  for.
- **Bundles** — feature bundles (groups of services). "Available" shows ones
  you haven't turned on; "Installed" shows what's on, the Docker image each
  claimed service actually runs, and export/import for the whole set.
  Browsing is open to everyone; enabling/disabling/reconciling needs an
  **admin** account (not just any login).

## Models & voice

### Model Management (`/platform`)

Pull, delete, and test Ollama models. Browsing and testing a prompt are open
to any logged-in user; pulling and deleting a model need an admin account.

!!! tip
    If you're already in OpenWebUI for chat, its own Admin Panel → Connections
    → Ollama → Manage offers the same pull/delete against the same Ollama
    instance, with more per-model settings (system prompts, parameters).

### Voice (`/platform/voice`)

Try Minder's text-to-speech and speech-to-text engines directly in the
browser — around 12 languages, Turkish by default. Browsing is open to
everyone; synthesizing or transcribing needs login.

### Status (`/platform/status`)

Health, reported version, and recent logs for every core service. The health
grid itself is open to everyone; viewing logs needs login, since they can
contain stack traces treated as sensitive.

### Backups (`/platform/backups`)

Enqueue a full backup of the platform's data and, when you need it, **restore**
from one — a guarded action behind an explicit confirm. Both run as background
jobs listed under "Recent Jobs" with their status. Admin-only: it touches every
service's data.

## Teams & organizations

- **Teams** (`/teams`) — group users into teams. Any logged-in user can
  create one (and becomes its team admin); managing an existing team's
  membership or settings needs that team's own admin, or an instance admin.
- **Users** (`/users`, admin-only) — change a user's role. Accounts linked to
  Authelia SSO show their role read-only, since it's re-derived from
  Authelia's group membership on every login — change it there instead.
- **Organization** (`/organization`) — your organization, its members, and
  switching between orgs you belong to (invite links can be copied or
  revoked here too).
- **Billing** (`/billing`) — your organization's current plan and subscription
  (tier, renewal date). Upgrades and payment-method changes hand off to the
  billing provider's secure hosted pages / customer portal rather than taking
  card details in-app. Org owner/admin.
- **All Organizations** (admin-only) — every organization on the instance,
  and a form to provision a new one (it gets a primary owner and a default
  team automatically).
- **Invitation** — the page an invite link opens to, for accepting it.
- **Audit Log** (admin-only) — the append-only record of privileged actions
  (role changes, org/member management, plugin review, and more), with
  actor, time, source, and before/after state.

### Settings (`/settings`)

Your own account as Minder currently sees it (username, email, role) and a
log-out action. A note points SSO users at Authelia's own portal for changing
a password or display name — this page doesn't do that, since Authelia is the
actual identity source for SSO logins.

## Marketplace: submitting & reviewing plugins

A lighter-touch corner of the UI, mostly relevant once you're publishing or
moderating plugins rather than just using the platform:

- **Submit a Plugin** — publish your own plugin to the marketplace. Every
  submission starts as a private draft and goes through admin review before
  anyone else can see or install it.
- **Review Queue** (admin-only) — developer-submitted plugins waiting on
  review, oldest first.
- **My Licenses** — the plugin tiers licensed to your account. Licenses are
  currently granted by an administrator; there's no self-service upgrade yet.

---

Everything above is a *browser* view of the same api-gateway the
[CLI](cli.md) talks to — pick whichever fits what you're doing.
