# Capability bundles

Minder groups its services into **bundles** — capability groups you turn on and
off as a unit. You choose *what the platform can do* by enabling and disabling
bundles, **not** by editing Compose files. This keeps a small deployment light
(run only what you use) and a larger one complete.

## The bundles

- **`core`** — the always-on kernel (the API gateway, plugin registry,
  marketplace, the data stores, the reverse proxy and SSO). It can't be disabled.
- **`inference`** — the local Ollama LLM runtime (see [AI setup](ai-setup.md)).
- **`rag`** — the RAG pipeline: knowledge bases, ingestion, and retrieval
  (see [RAG methods](rag-methods.md)).
- **`graph-rag`** — the knowledge-graph service (spaCy NER + Neo4j).
- **`chat`** — the OpenWebUI chat frontend.
- **`voice`** — text-to-speech and speech-to-text.
- **`monitoring`** — the observability stack: Prometheus, Grafana, Alertmanager,
  Jaeger, and the exporters (see [Monitoring](monitoring.md)).

## Choosing a starting set

At install time, `--profile` seeds the initial bundle set:

```bash
bash setup.sh install --profile minimal   # core only
bash setup.sh install --profile standard   # core + inference + rag + chat (the default)
bash setup.sh install --profile full       # everything (except failover-gated sidecars)
```

`start` afterwards honours the recorded state, so you enable a profile once and
subsequent starts bring up the same set.

## Managing bundles

```bash
bash setup.sh bundle status                       # bundles, their services, and enable state
bash setup.sh bundle enable rag                    # bring a bundle's services up
bash setup.sh bundle enable monitoring             # e.g. turn on Prometheus/Grafana/Jaeger
bash setup.sh bundle disable monitoring            # report what would be orphaned (notify-only)
bash setup.sh bundle disable monitoring --stop-orphans   # + actually stop them
bash setup.sh bundle reconcile                     # converge running containers to the enabled set
```

You can also manage bundles from the control-plane UI — the **Bundles** section
has an *Available* view (bundles you haven't turned on) and an *Installed* view
(what's on), with per-bundle toggles, a page-level **Reconcile**, and
export/import of the whole set. Enabling/disabling a bundle needs an **admin**
account. See [Using Minder](using-minder.md).

## How it works

A service runs if — and only if — at least one **enabled** bundle *claims* it. A
service that no enabled bundle claims is an **orphan**, and teardown is
reference-counted: disabling one bundle only stops a service if no other enabled
bundle still needs it. That's why `disable` reports orphans before stopping
anything, and why `--stop-orphans` is a separate, explicit step.

!!! note "Enable vs. create"
    Enabling a bundle starts containers that already exist. A service that has
    never been materialised on this host comes back as `pending_create` until the
    next `bash setup.sh start` / `restart` converges it — the in-platform toggle
    starts and stops containers, but it doesn't create brand-new ones.

## Related pages

- [Self-hosting](self-hosting.md) — install and first-run.
- [Using Minder](using-minder.md) — the Bundles UI.
- [Production](production.md) — the full deployment topology.
