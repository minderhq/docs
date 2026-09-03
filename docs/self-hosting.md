# Self-hosting Minder

Minder's core is a **proprietary product** — local-first, running entirely on
hardware you own and provisioned by a single command. Self-hosting the core needs
**licensed access** (the repository and container images are private); a freemium
trial is planned.

> The **open-source** side of Minder — the [plugin SDK](plugins/index.md), the
> plugin catalog, and the web client — is freely available. This page covers
> running the (licensed) core; the steps below assume you have access.

## Requirements

- **Docker** (with Compose) — every service runs in a container.
- **Python 3** — drives the `setup.sh` provisioning CLI.
- Modest hardware — a **Raspberry Pi 4 (arm64)** is a validated, first-class
  target; anything more is comfortable.

## Install

```bash
# Requires licensed access to the private minderhq/minder repo + images.
git clone https://github.com/minderhq/minder.git
cd minder
bash setup.sh install --profile standard   # minimal | standard | full
```

`setup.sh` is the single entrypoint (a thin shim over `python -m scripts.setup`).
It provisions the whole stack, fills secrets, and self-heals. `--profile` seeds
the initial set of **bundles** (capability groups) to enable.

## Capability bundles

You turn capabilities on and off as units — not by editing compose files:

```bash
./setup.sh bundle status                            # bundles + their services
./setup.sh bundle enable rag                         # bring a bundle up
./setup.sh bundle disable monitoring --stop-orphans  # take one down + its services
```

`core` is the always-on kernel; `rag`, `graph-rag`, `inference`, `chat`,
`voice`, and `monitoring` layer on top. A service runs only while a bundle that
claims it is enabled.

## Local vs. external inference

Inference is Ollama-backed. By default Minder runs a **local** Ollama container.
Set `OLLAMA_BASE_URL` to point at an external/native Ollama instead — resolved
from inside the containers, so use `host.docker.internal` or a LAN IP, not
`localhost`.

## Verify it's up

Host ports are **loopback-bound** by default (reachable on the box, not the LAN);
external access is via the reverse proxy, SSO-gated.

```bash
docker ps -a --filter health=unhealthy   # should be empty
curl http://localhost:8000/health         # api-gateway
curl http://localhost:8004/health         # rag-pipeline (if the rag bundle is on)
```

## Update

```bash
bash setup.sh update      # git pull already done → rebuild + rolling restart
```

## Manage it

Everything that isn't chat has a modern control-plane UI (the `minder-client`
SPA) — see [Using Minder](using-minder.md) for a tour; chat itself is
OpenWebUI. Extend the platform with [plugins](plugins/index.md).

!!! note
    Deeper operations, architecture, and hardening guides ship with the licensed
    core; the plugin-ecosystem docs on this site are the openly available subset.
