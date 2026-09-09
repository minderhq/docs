# AI setup

Minder runs LLM inference through **Ollama**. In the default *internal* mode the
platform runs its own Ollama container and auto-pulls the configured models
during setup — no manual step required. You can also point Minder at an
external Ollama, or run a failover pair.

!!! note
    Ollama listens on port `11434` **internal to the Docker network only** — it
    is not exposed on the host. Reach it from another container, from inside the
    Ollama container (`docker exec`), or via the services that proxy to it.

## Ollama modes: internal, external, failover

The mode is recorded in the root `.env` and selected with the `ollama-mode` verb:

- **Internal (default)** — `OLLAMA_BASE_URL` is empty. The platform-managed
  `minder-ollama` container runs (gated behind the `internal-ollama` compose
  profile, which activates only when `OLLAMA_BASE_URL` is empty).
- **External / native host** — `OLLAMA_BASE_URL` is set to a URL. The internal
  container stays inactive and services talk to your external Ollama instead
  (same host native install, or a remote GPU box).
- **Failover** — an external **primary** with automatic fallback to the internal
  container when the primary is unreachable, and automatic return once it
  recovers. See [Failover mode](#failover-mode) below.

```bash
# Platform-managed Ollama container (default)
bash setup.sh ollama-mode internal

# External Ollama — defaults to http://host.docker.internal:11434
bash setup.sh ollama-mode external

# External Ollama at a specific URL
bash setup.sh ollama-mode external http://192.168.1.50:11434

# Failover: external primary + fallback to the internal container
bash setup.sh ollama-mode failover http://192.168.1.50:11434
```

This edits `.env` (`OLLAMA_BASE_URL`, plus `OLLAMA_FAILOVER_PRIMARY` for
failover); it does **not** restart — run `bash setup.sh restart` to apply. In
external mode the local Ollama container isn't started, saving RAM/CPU; failover
keeps it running as a warm standby.

### The URL must be reachable *from inside the containers*

`OLLAMA_BASE_URL` is resolved by the service containers on the Docker network —
**not** by your host shell. An address that works from the host (or only on your
LAN's DNS) can still be unreachable from a container:

- **Ollama running natively on the same machine as Docker** — use
  `http://host.docker.internal:11434` (Docker Desktop's host alias) or the
  host's LAN IP (e.g. `http://192.168.1.50:11434`). `http://localhost:11434`
  will **not** work from a container (there, `localhost` is the container
  itself).
- **A bare hostname** (e.g. `http://gpu-node:11434`) only works if that name
  resolves *inside the containers*.

Ollama must also listen on all interfaces for containers to reach it — start the
native server with `OLLAMA_HOST=0.0.0.0` (not the default `127.0.0.1`).

!!! warning "Silent failure if the URL is unreachable"
    If `OLLAMA_BASE_URL` is unreachable, a document upload still returns `200`
    but stores a **zero-vector** (the document is never retrievable), and
    queries return an error. Always verify **from a container**, not the host:

    ```bash
    # expect 200 — this is what the RAG/model-management services actually see
    docker exec minder-rag-pipeline curl -s -o /dev/null -w '%{http_code}\n' \
      --max-time 5 "$OLLAMA_BASE_URL/api/tags"
    ```

## Single-host setup (Ollama already running natively)

If you already run Ollama on the same machine you're installing Minder on, point
Minder at it with **external mode** rather than running a second instance:

```bash
# Before installing, in .env:
OLLAMA_BASE_URL=http://host.docker.internal:11434   # see the address notes above

# Or right after installing:
bash setup.sh ollama-mode external http://host.docker.internal:11434
bash setup.sh restart
```

Internal mode would start a *second*, platform-managed Ollama container
alongside your native one — two processes competing for the same GPU/RAM for no
benefit. External mode gives every Minder service (RAG pipeline, model
management, gateway, OpenWebUI) the same access to your native instance and
keeps the internal container inactive.

## Failover mode

Failover gives you a fast external Ollama without the single point of failure of
plain external mode: if the external host goes away, the platform serves from
its own internal container and returns to the external host once it's back.

```bash
bash setup.sh ollama-mode failover http://192.168.1.50:11434
bash setup.sh restart
```

A small `minder-ollama-router` (nginx) sits in front of Ollama with the external
host as **primary** and the internal `minder-ollama` container as **backup**. All
services point at the router
(`OLLAMA_BASE_URL=http://minder-ollama-router:11434`), so failover is transparent
— OpenWebUI included. When the primary stops answering, requests fail over to the
internal container; the router re-probes the primary and returns to it
automatically. The internal container stays running as a warm standby (an idle
Ollama loads no model into RAM, so it costs almost nothing until it serves a
fallback request).

```bash
# Which backend is active:
bash setup.sh status    # → "Ollama Backend": primary REACHABLE / UNREACHABLE (fallback)
```

!!! note "Model availability on fallback"
    Fallback is seamless only for models the internal container actually has —
    by default `llama3.2` + `nomic-embed-text`. A knowledge base or query pinned
    to a model that lives **only** on the external primary (e.g. a large model
    the internal host can't run) will fail while the primary is unreachable.
    Keep your default and embedding models available on both sides.

## Automatic model downloads

On startup in internal mode, Minder pulls the models listed in `OLLAMA_MODELS`
(default `llama3.2` + `nomic-embed-text`), skipping any already present. Models
live in a Docker volume, so they survive container recreation.

Configure in the root `.env`:

```bash
# Models to auto-pull on startup (comma-separated)
OLLAMA_MODELS=llama3.2,nomic-embed-text

# Which models the services use for each purpose
OLLAMA_LLM_MODEL=llama3.2
OLLAMA_EMBEDDING_MODEL=nomic-embed-text

# Empty = internal container mode; set to a URL = external mode
OLLAMA_BASE_URL=
```

## Suggested models

| Model | Size | Notes |
|-------|------|-------|
| **llama3.2** | ~2.0 GB | General-purpose LLM; fast, good quality. Recommended default. |
| **mistral** | ~4.1 GB | Better for complex reasoning; slower, needs more RAM. |
| **qwen2.5** | ~2.5 GB | Multilingual; good for non-English text and translation. |
| **nomic-embed-text** | ~274 MB | Text embeddings; fast and efficient. Recommended default. |
| **mxbai-embed-large** | ~669 MB | Higher-quality embeddings; slower, more accurate. |

### Manual model management

```bash
docker exec minder-ollama ollama list            # installed models
docker exec minder-ollama ollama pull mistral    # download another
docker exec minder-ollama ollama rm mistral      # remove one
docker exec minder-ollama ollama show llama3.2   # model info
```

Change the auto-pull set by editing `OLLAMA_MODELS` in `.env` (leave it empty to
disable pulls), then `bash setup.sh restart`.

## Using your models

- **RAG pipeline** uses `OLLAMA_EMBEDDING_MODEL` for embeddings and
  `OLLAMA_LLM_MODEL` for generation — see [RAG methods](rag-methods.md).
- **Model management** lists, pulls, deletes, and tests Ollama models via the
  control-plane UI ([Using Minder](using-minder.md#model-management-platform)).
- **OpenWebUI** is the web chat frontend (reached via the reverse proxy). It
  auto-detects installed models — pick one from the dropdown and start chatting.

### Plugin tools in the chat UI

Minder's read-only plugin tools (e.g. `get_crypto_price`, `get_weather`,
`get_news`) are exposed as an OpenAPI tool server at
`GET /v1/ai/tools/openapi.json` on the api-gateway — a standard Open WebUI
extension point (**Settings → Admin → Tool Servers**, type `openapi`). The stack
pre-registers it on first startup. Only read-only tools appear by design;
mutating plugin actions stay authenticated and are never exposed as freely
callable tools.

!!! note "Known upstream gap"
    Some Open WebUI versions only fully activate a pre-seeded tool-server
    connection after an admin opens **Settings → Admin → Tool Servers** once and
    clicks **Save** (even though it's already listed) — see
    [open-webui#18140](https://github.com/open-webui/open-webui/issues/18140). If
    a chat with tools enabled doesn't see Minder's tools, do that once.

## Troubleshooting

**Models not downloading** — check `docker logs minder-ollama`, then manually
`docker exec minder-ollama ollama pull llama3.2`. Common causes are insufficient
disk space or network issues; for a corrupted download, `ollama rm` then
re-pull.

**Out of space** — inspect with `docker exec minder-ollama du -sh /root/.ollama`
and remove unused models with `ollama rm <model>`.

## Resource guidance

| Model | RAM |
|-------|-----|
| llama3.2 | 4 GB minimum |
| mistral | 8 GB recommended |
| qwen2.5 | 6 GB recommended |
| nomic-embed-text | 1 GB |

CPU: 4 cores is a workable minimum, 8+ is comfortable. Allow ~3 GB of disk for
the base models (llama3.2 + nomic-embed-text) and 2–8 GB per additional model;
20 GB free is a safe starting point.
