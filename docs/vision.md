# Vision

**Minder is your private AI platform — local inference, your own data, and an
extensible tool ecosystem — that installs on modest hardware with a single
command.**

This page is the *shape and direction* of the platform: what it is for and the
principles behind it. It is a living document — where it and the product
disagree, the product is what ships.

!!! note "Proprietary core, open ecosystem"
    The Minder platform (core) is a commercial product, self-hostable with a
    licence. Its **ecosystem is open source** — the plugin SDK, the plugin
    catalog, the CLI, and the web client. Run standalone, the self-hosted product
    is local-first and never phones home.

## North Star

A person or a small team should be able to run, on hardware they own (down to a
Raspberry Pi), a complete AI system that:

- answers questions over **their own documents and data** (RAG + a knowledge graph),
- runs **local LLMs** with no cloud API keys and no data leaving the box by
  default — any remote model provider a user connects is opt-in, per-organization,
  and explicit,
- is **extended by plugins and tools** without writing or trusting arbitrary code,
- is **operated from one coherent, modern control-plane UI** — not a pile of raw forms,
- and is **honest about what it does**: every capability is real, verifiable, and
  observable, or it is clearly marked as not-yet-implemented.

If a capability can't be run end-to-end and shown working, it isn't done.

## Principles

1. **Local-first & private by default.** Inference, storage, and retrieval run on
   the user's own hardware. The default posture is "nothing leaves the box"; any
   egress (e.g. an online TTS fallback) is opt-in and obvious.
2. **One command to a working system.** `bash setup.sh` provisions the whole
   stack, fills secrets, and self-heals. Capability is toggled by **bundles**, not
   by editing compose files.
3. **Extensible without arbitrary code execution.** Plugins are **manifest-based**;
   new actions are fixed, reviewed handlers — never uploaded code. Safety is a
   design property, not a scanner bolted on afterward.
4. **Runs on modest hardware.** ARM / Raspberry Pi is a first-class target, not an
   afterthought. Features are chosen and tuned to fit (e.g. Piper for offline TTS,
   optional cross-encoder reranking that degrades gracefully when the ML extras
   are absent).
5. **Honest & verifiable.** Docs match reality; "done" means proven by running
   with real output. Not-yet-implemented paths fail loudly (clear errors), never
   silently pretend.
6. **A real product, not a demo.** A coherent, task-oriented UI; consistent APIs;
   observability from day one; secure-by-default networking.

## Pillars

### Local inference

Ollama-backed LLM runtime, with clean local ⇄ external ⇄ failover switching. Model
lifecycle (pull / list / delete / test) is first-class. See [AI setup](ai-setup.md).

### Knowledge — RAG *and* graph

Retrieval that goes beyond naive vector search: Standard / HyDE / Self-RAG / auto /
corrective methods, plus hybrid and parent-child retrieval, RAPTOR hierarchical
retrieval, and optional rerank / compress — all capability-adaptive. A parallel
**knowledge-graph** path (spaCy NER → Neo4j) for entity / relationship exploration
over the same documents. See [RAG methods](rag-methods.md).

### Extensibility — plugins, tools, marketplace

Manifest-based plugins that can write to any backend and register as **AI tools**
for LLM function-calling, plus a marketplace with a dependency / conflict graph.
See [Plugins](plugins/index.md).

### The control-plane UI

A bespoke web app that is the single, modern surface for everything that isn't
chat: knowledge bases, pipelines, the graph explorer, plugins & AI tools,
model / bundle / health ops, and voice. Chat itself stays in OpenWebUI. See
[Using Minder](using-minder.md).

### Observability & operations

Prometheus / Grafana / Jaeger / InfluxDB / OpenTelemetry wired in from the start;
self-healing provisioning; loud, honest backups. See [Monitoring](monitoring.md).

### Security & multi-user

JWT auth, licence fail-closed, Authelia SSO / 2FA enforced at the edge,
loopback-bound host ports, no-arbitrary-code plugins, and a per-deployment-generated
admin credential rather than a shipped default. Role checks currently cover a
specific set of admin-only actions; extending them across the rest of the write
surface, and enabling full browser SSO once a real domain and TLS exist, is
in-flight. See [Authentication](authentication.md).

## Who it's for

- **The self-hoster / homelabber** who wants private AI on hardware they own.
- **The small team** that needs a shared, on-prem knowledge + automation base
  without sending documents to a third party.
- **The builder** who wants to extend an AI platform with tools / plugins safely.

## Non-goals (on purpose)

- **Not a cloud SaaS when self-hosted.** Run standalone, it is local-first and
  never phones home — no account and no data leaving the box.
- **Not a model-training platform.** Minder *runs and orchestrates* models; it
  doesn't train them.
- **Not a chat UI reinvention.** OpenWebUI owns chat; Minder owns everything
  around it.
- **Not maximal complexity.** Features earn their place by a real, demonstrated
  need (e.g. multi-backend inference routing waits for a proven bottleneck).

## The UX north star

The control-plane should feel like **one modern product**, not a set of service
forms:

- **Task-first, not endpoint-first.** Screens organised around what a user is
  trying to do (ingest a document, ask a question, turn on a capability), with the
  underlying services invisible.
- **A cohesive design system.** One typographic scale, spacing rhythm, and colour
  system; consistent cards, buttons, empty / loading / error states; light **and**
  dark parity; accessible by default (labels, focus, `aria-live` status, keyboard
  paths).
- **Works where the user actually is.** Every access path is real — including
  direct `localhost` / LAN over plain HTTP — so login, copy, IDs, and links never
  assume a hostname or secure context the user doesn't have.
- **Trustworthy feedback.** Clear progress, honest errors (backend-down vs. your
  input), confirmation before destructive actions, and no silent failures.

## How we get there

The high-level shape of the sequence:

1. **Harden what exists.** Correctness, standardization, and consistency across the
   services and the client; make every documented flow provably work.
2. **Make it genuinely usable.** The modern control-plane UX; access-path parity;
   local login; task-oriented screens.
3. **Deepen the intelligence.** Richer retrieval beyond what's already wired; a
   stronger RAG ↔ graph story; better tool-calling ergonomics.
4. **Open the ecosystem.** A real marketplace submission / trust flow; more plugins.
5. **Real multi-user & production.** Role checks across the rest of the write
   surface, full browser SSO, and production hardening for a public deploy.

---

*Keep this honest: if Minder can't do something described here, either the
capability or this text is wrong — and whichever it is gets fixed.*
