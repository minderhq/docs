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

The sign-in page (`/login`, opened by the **Log in** button) offers two ways in:

- **Local account** — a username/password form. If the instance's
  [registration mode](authentication.md#registration-modes) allows it, a link
  switches to "Create an account" (username + email + password); in `invite`
  mode sign-up needs an invite link, and in `sso_only` mode there is no sign-up.
  A **Forgot password?** link appears when [transactional email](email.md) is on.
  This is the path that works over `localhost` or a bare LAN IP.
- **SSO (Authelia)** — a "Sign in with SSO" button, shown only when the client
  was built with an SSO target (`CLIENT_OIDC_LOGIN_URL`, which setup fills in
  unless you set it empty). The SSO redirect needs the `*.minder.local` hosts or
  real DNS and TLS to complete, so on a plain `localhost` or LAN address use a
  local account. A local account is always a plain `user`; the first
  administrator signs in with SSO.

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
it — this is the data your pipelines search over. A wide range of file types is
accepted (PDF, Word, spreadsheets, images via OCR, audio/video via transcription,
and more), with the type detected from content rather than the extension — see
[Ingestion & file formats](ingestion.md). Upload runs as a background
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
the Marketplace pages — same Neo4j instance, unrelated data. See
[Knowledge graph](knowledge-graph.md) for the graph API, the correlation engine,
and the review queues.

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
| `/marketplace/discover/plugins` | `/marketplace/installed/plugins` |
| `/marketplace/discover/ai-tools` | `/marketplace/installed/ai-tools` |
| `/marketplace/discover/service-bundles` | `/marketplace/installed/service-bundles` |

Older URLs such as `/plugins/available` and `/bundles/installed` redirect to
these pages.

- **Plugins** — browsing is open to everyone; installing, enabling, disabling,
  uninstalling, or editing a plugin's config needs login (installs are
  per-user).
- **AI Tools** — the function-calling tools an LLM can actually call. A tool
  that changes or sends data pauses for your approval before it runs (see
  [Tool approvals](api-reference.md#tool-approvals)).
  "Available" is Marketplace's durable catalog (includes tools from plugins
  that aren't currently running, and can lag slightly behind "Installed");
  "Installed" is what's callable *right now*, from plugins Plugin Registry
  currently has loaded. Both are read-only pages — there's nothing to log in
  for.
- **Bundles** — feature bundles (groups of services). "Available" shows ones
  you haven't turned on; "Installed" shows what's on, the Docker image each
  claimed service actually runs, and export/import for the whole set.
  Browsing is open to everyone; enabling/disabling/reconciling needs an
  **admin** account (not just any login). See [Bundles](bundles.md) for the
  model and the CLI.

### Plugin ratings & reviews

Every plugin card carries a **Ratings & reviews** panel: its live average
(e.g. `4.2★ (17)`) and the individual reviews, newest first. Opening the panel
loads the reviews on demand.

Rating a plugin is **install-gated** — you can rate one you've installed (a
plugin you tried and later uninstalled still counts). Pick 1–5 stars, add an
optional written review, and submit. You get **one review per plugin**:
submitting again edits your existing one in place (the panel shows "Editing
your review" and an **Update review** button), and your own review is
highlighted in the list. Browsing the aggregate and reviews is open to anyone;
submitting needs you to be logged in and to have installed the plugin — if you
haven't, the form stays disabled with an "Install to review" hint.

## Models & voice

### Model Management (`/platform`)

Pull, delete, and test Ollama models. Browsing and testing a prompt are open
to any logged-in user; pulling and deleting a model need an admin account.

!!! tip
    If you're already in OpenWebUI for chat, its own Admin Panel → Connections
    → Ollama → Manage offers the same pull/delete against the same Ollama
    instance, with more per-model settings (system prompts, parameters).

### Cloud Providers (`/platform/providers`)

The **Providers & defaults** page. Connect an external OpenAI-compatible
provider (OpenAI, z.ai, OpenRouter, Together, …), Anthropic, or a
**self-hosted** server (vLLM, LM Studio, llama.cpp, TGI or your own Ollama), so
its models appear alongside your local Ollama models, and pick a default model
per task. It is shown to members with the `org.ai.manage` permission, is
per-organization, and is opt-in — nothing here is on by default, and your local
setup keeps working exactly as before if you never add one. See
[Connect your models](model-providers.md).

To connect a provider, pick a **provider type** from the dropdown — it
pre-fills the form for that setup:

- **OpenAI-compatible** — OpenAI itself, or any other endpoint speaking the
  same chat-completions API (z.ai, OpenRouter, Together, Azure OpenAI, a
  self-hosted gateway). Set a base URL for a non-default vendor.
- **vLLM, LM Studio, llama.cpp server, TGI and Ollama (self-hosted)** — a server
  you run yourself, at your own base URL (e.g. `http://<your-vllm-host>:8000/v1`).
  These are flagged as **local compute**, which exempts them from the cloud rate
  limit described below.
- **Anthropic** — Anthropic's hosted Claude models.

Then give it a name, optionally set the base URL, and paste the API key. The
key is encrypted at rest and never shown again in full — only a masked form
(e.g. `sk-...ab12`).

!!! note "The API key is optional for self-hosted servers"
    A self-hosted (local) provider can be saved without a key, for a server that
    has no authentication; Minder then sends no `Authorization` header. A cloud
    provider always needs a key.

Each provider row has a **Test** button that makes one minimal real completion
call against the provider's first model, so a bad key or an unreachable
endpoint surfaces here rather than mid-chat, and a **Fetch models** action that
lists what the server serves so you can save the list. You can also disable a
provider without deleting it, rename it, replace its key or edit its model list,
or delete it outright.

!!! warning "Hosted vendors leave your machine and are metered — local ones aren't"
    Once connected, prompts routed to a **hosted** provider's models (OpenAI,
    Anthropic, …) leave your machine and are billed by the vendor — the
    opposite of the local-only default. To bound that cost, calls to a hosted
    provider's models (both inference and the **Test** button) are rate-limited
    to 5/min. A provider marked **local** (a self-hosted row) is your own compute
    with no per-call cost, so inference is exempt, and **Test** has a looser
    30/min limit.

### Voice (`/platform/voice`)

Try Minder's text-to-speech and speech-to-text engines directly in the
browser — around 12 languages, Turkish by default. Browsing is open to
everyone; synthesizing or transcribing needs login.

### Status (`/platform/status`)

Health, reported version, and recent logs for every core service. The health
grid itself is open to everyone; viewing logs needs a
[Platform Admin](platform-admin.md) account, since log output carries every
tenant's data and can contain stack traces treated as sensitive.

### Backups (`/platform/backups`)

Enqueue a full backup of the platform's data and, when you need it, **restore**
from one — a guarded action behind an explicit confirm. Both run as background
jobs listed under "Recent Jobs" with their status. Admin-only: it touches every
service's data. See [Backup & restore](backup-restore.md) for what's captured,
the job-queue model, and the guarded restore.

## Teams & organizations

See [Organizations & teams](organizations-teams.md) for the roles, guards, and
invite flow behind these pages.

- **Teams** (`/platform/teams`) — group users into teams. Any logged-in user can
  create one (and becomes its team admin); managing an existing team's
  membership or settings needs that team's own admin, or an instance admin.
- **Users** (`/platform/users`, Platform Admin only) — list users across every
  organization, change a role, reset a password, deactivate or reactivate an
  account. Accounts linked to Authelia SSO show their role read-only, since it's
  re-derived from Authelia's group membership on every login — change it there
  instead.
- **Organization** (`/organization`) — your organization's settings, and
  switching between orgs you belong to.
- **People** (`/organization/people`) — the organization's members (roles,
  teams, suspend, remove), **Invites** (invite one person or a list; pending
  invites; copy or revoke a link) and **Domains** (which email domains may join).
  Needs `org.members.manage`; the domain list needs `org.settings.manage`.
- **Requests** (`/requests`) — what you asked for (join an organization, an
  invite, a team, a plugin, a model or a plan), and requests waiting for your
  decision. See
  [Access requests](organizations-teams.md#access-requests-and-the-waiting-room).
- **Public chat** (`/rag/public-chat`) — expose a pipeline as an anonymous
  chatbot endpoint, review its conversations and feedback. Needs
  `org.chatbots.manage`.
- **Billing** (`/billing`) — your organization's current plan and subscription
  (tier, renewal date). Upgrades and payment-method changes hand off to the
  billing provider's secure hosted pages / customer portal rather than taking
  card details in-app. Needs `org.billing.view`.
- **All Organizations** (`/organizations`, Platform Admin only) — every
  organization on the instance, and a form to provision a new one (it gets a
  primary owner and a default team automatically).
- **Invitation** (`/invite/<token>`) — the page an invite link opens to, for
  accepting it.
- **Audit Log** (`/audit`) — the append-only record of privileged actions (role
  changes, org/member management, plugin review, and more), with actor, time,
  source, and before/after state. A Platform Admin sees every entry; anyone else
  with `org.audit.view` sees their active organization's entries (and its
  sub-organizations').

### Settings (`/settings`)

Your own account as Minder currently sees it (username, email, role), plus a
Minder-side display name, theme and language (English or Turkish) that follow
your account across devices, and a log-out action. A local account changes its
password here. For an SSO account the page links to Authelia's own portal
instead, since Authelia is the actual identity source for SSO logins.

## Marketplace: submitting & reviewing plugins

A lighter-touch corner of the UI, mostly relevant once you're publishing or
moderating plugins rather than just using the platform:

- **Publish → Submissions** (`/marketplace/publish/submissions`) — publish your
  own plugin to the marketplace. Every submission starts as a private draft and
  goes through admin review before anyone else can see or install it.
- **Publish → Submission review** (Platform Admin only) — developer-submitted
  plugins waiting on review, oldest first.
- **My Licenses** (`/billing/licenses`) — the plugin tiers licensed to your
  account. Licenses are currently granted by an administrator; there's no
  self-service upgrade yet.
- **Plugin sources** (`/settings/sources`, Platform Admin only) — the repositories
  plugins come from, with their linked plugins.

---

Everything above is a *browser* view of the same api-gateway the
[CLI](cli.md) talks to — pick whichever fits what you're doing.
