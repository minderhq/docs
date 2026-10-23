# External services

Minder is backed by a set of third-party and infrastructure services: storage
engines, an LLM runtime, observability, and a reverse proxy. A self-hosted
install runs all of them as Docker containers. This page says which of them can
run somewhere else, which cannot, and how to handle credentials.

!!! note
    For standing up an instance in the first place, see
    [Self-hosting](self-hosting.md).

## Service overview

All services attach to internal Docker networks (`minder-network` and a few
narrower ones) and are reachable from other containers by container name.
Host-published ports are loopback-bound by default; external access goes through
the reverse proxy.

| Category | Services | Purpose |
|----------|----------|---------|
| Inference | Ollama, Open WebUI | Local LLM runtime + chat UI (see [AI setup](ai-setup.md)) |
| Storage | PostgreSQL, Redis, Qdrant, Neo4j, MinIO, RabbitMQ, Schema Registry | Relational data, cache/sessions, vectors, graph, object store, message queue, schema registry |
| Observability | Prometheus, Grafana, Alertmanager, InfluxDB, Telegraf, Jaeger, OpenTelemetry Collector | Metrics, dashboards, alerting, time-series, tracing |
| Exporters | postgres-, redis-, rabbitmq-, node-exporter, cAdvisor, blackbox-exporter | Metrics sources scraped by Prometheus |
| Networking | Traefik, Authelia | Reverse proxy / TLS, and SSO / forward-auth |

## What can run elsewhere

Two services have a supported switch that points the platform at an instance
you run yourself, set with `setup.sh` and recorded in the root `.env`:

| Service | Command | Notes |
|---------|---------|-------|
| Ollama (inference and embeddings) | `bash setup.sh ollama-mode internal\|external\|failover\|pool [url]` | `external` points at your own Ollama, `failover` adds an automatic fallback to the bundled container, `pool` spreads requests over several Ollama servers. See [AI setup](ai-setup.md). Cloud and self-hosted model providers are a separate, per-organization layer: see [Connect your models](model-providers.md). |
| Text-to-speech and speech-to-text | `bash setup.sh tts-stt-mode internal\|external\|failover [url]` | The same idea for the `tts-stt` service. An external host must run the same Minder release and trust this install's service keys (see the `SERVICE_PUBLIC_KEYS` note in `.env.example`), otherwise audio and video uploads fail to ingest with a `401`. |

Both commands only edit `.env`. Run `bash setup.sh restart` to apply.

## Data services are bundled

PostgreSQL, Redis, Qdrant, Neo4j, MinIO and RabbitMQ run as Minder's own
containers, and there is **no `.env` setting that points the platform at an
external provider for them**. The services reach them by fixed container names
on the internal network (for example `postgres`, `redis` and `qdrant`), set in
the Compose file. The services' settings for these hosts are not exposed through
`.env`, and nothing reads a TLS or API-key setting for them (the Qdrant clients
use plain `http://`). Setting `POSTGRES_HOST`, `REDIS_HOST` or `QDRANT_HOST` in
`.env`, or a `DATABASE_URL`, does not repoint the application services.

If you need a managed database today, that is a change to the Compose file and
to what each service supports, not a configuration switch, and it is not
something Minder tests or documents. Run the bundled containers, and protect the
data with the [backup and restore](backup-restore.md) tooling and an off-device
copy.

## Reaching the data services

Internal-only services publish no host port. Reach them from inside the Docker
network by container name:

| Service | Internal address |
|---------|------------------|
| PostgreSQL | `minder-postgres:5432` |
| Redis | `minder-redis:6379` |
| Qdrant | `minder-qdrant:6333` |

See [Service access](service-access.md) for how to get a shell, a health check
or a log from each one.

## Credentials

- Never commit a `.env` containing real credentials. Setup generates strong
  values for every secret and sets the file's mode to `600`.
- Use different credentials per deployment, and rotate them on a schedule. Some
  secrets encrypt stored data and must never be cleared. See
  [Security](security.md#credential-rotation).
- Reachability is resolved from inside the containers. An address that resolves
  on your host, or only on your LAN's DNS, can still be unreachable from a
  container. Verify from inside one, for example:

  ```bash
  docker exec minder-rag-pipeline sh -c \
    'curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 "$OLLAMA_BASE_URL/api/tags"'
  ```

## Related pages

- [AI setup](ai-setup.md): the Ollama modes in full.
- [Production](production.md): deployment topology and the go-live checklist.
- [Backup & restore](backup-restore.md): protecting the data services' volumes.
