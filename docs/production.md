# Production

Minder is designed to run entirely on hardware you own: every service is a Docker
container, provisioned and managed by the single `bash setup.sh` entrypoint. This
page covers taking a Minder instance from a working local install to a
**production-hardened** deployment — the operational topology, health and backup
routines, upgrades, scaling, and the hardening steps a public-facing deployment
needs.

!!! note "Start with a working install first"
    This page assumes you already have Minder running. For first-time
    provisioning, see [Self-hosting](self-hosting.md). A default install is a
    **development-grade** deployment: it is functional end-to-end, but the
    hardening below (real TLS, enforced SSO, secret rotation, firewalling) is
    **not** applied out of the box. Treat this page as the checklist that closes
    that gap.

## Single source of truth

Two files define a deployment:

- **Configuration and secrets** live in the root `./.env` — the single source of
  truth. `setup.sh` self-heals it: any value left as `CHANGEME` is replaced with a
  secure random value on install/start. Edit `./.env`, then restart to apply.
- **The stack itself** is one hand-maintained Compose file that `setup.sh` always
  invokes for you. You do not regenerate or template it; enable and disable
  capabilities through **bundles** (see [Self-hosting](self-hosting.md)), not by
  editing Compose by hand.

```bash
# ./.env is the ONLY file you edit; setup.sh mirrors it where services need it.
ENVIRONMENT=production
LOG_LEVEL=INFO

# Secrets — leave as CHANGEME to auto-generate, or set explicitly
POSTGRES_PASSWORD=CHANGEME
REDIS_PASSWORD=CHANGEME
JWT_SECRET=CHANGEME
INFLUXDB_TOKEN=CHANGEME

# Inference: empty = run the local Ollama container; set a URL = offload to a host
OLLAMA_BASE_URL=
OLLAMA_MODELS=llama3.2,nomic-embed-text
```

!!! warning "Never commit real secrets"
    Keep `./.env` out of version control. Use distinct credentials per
    environment and rotate them. See
    [External services](external-services.md#best-practices) for the full
    guidance on credentials and managed data backends.

## Deployment topology

A full install defines on the order of three dozen containers (a couple are
failover-mode sidecars that stay inactive by default). Only the reverse proxy and
the application/monitoring services below publish host ports; **all storage
backends are internal-only**, reached over the `minder-network` Docker network or
fronted by the reverse proxy.

!!! note "Host ports are loopback-bound"
    Every non-proxy host port is bound to `127.0.0.1` — reachable **on the box**
    (`curl http://localhost:<port>` for ops and health still works) but **not**
    from other machines. External access is intended to go through the reverse
    proxy only. Because the loopback ports sit *behind* the proxy's auth gate,
    treat host access to the box as privileged and firewall it accordingly.

### Core API services (FastAPI)

| Service | Container | Host port |
|---|---|---|
| API Gateway | `minder-api-gateway` | 8000 |
| Plugin Registry | `minder-plugin-registry` | 8001 |
| Marketplace | `minder-marketplace` | 8002 |
| Orchestrator | `minder-orchestrator` | 8003 |
| RAG Pipeline | `minder-rag-pipeline` | 8004 |
| Model Management | `minder-model-management` | 8005 |
| TTS / STT | `minder-tts-stt` | 8006 (internal by default) |
| Graph-RAG | `minder-graph-rag` | 8008 |

Each core API serves interactive docs at `/docs`.

### Inference

- `minder-ollama` — the local LLM runtime (internal `11434`, not host-exposed).
  It runs only in **internal** mode (when `OLLAMA_BASE_URL` is empty); set a URL
  to offload inference to an external or native host. Switch modes with
  `bash setup.sh ollama-mode`. See [AI setup](ai-setup.md).
- `minder-openwebui` — the web **chat** UI, reached through the reverse proxy.
  This is distinct from Minder's own **management** SPA (`minder-client`), which
  is reverse-proxy-routed and forward-auth gated (and also published on loopback
  like the core APIs). See [Using Minder](using-minder.md) for the control-plane
  tour.

### Storage (internal-only)

| Service | Container | Role |
|---|---|---|
| PostgreSQL | `minder-postgres` | Relational data: users, sessions, metadata |
| Redis | `minder-redis` | Cache, sessions, rate-limit counters, pub/sub |
| Qdrant | `minder-qdrant` | Vector store for RAG embeddings |
| Neo4j | `minder-neo4j` | Graph store (marketplace deps + graph-RAG) |
| MinIO | `minder-minio` | S3-compatible object store |
| RabbitMQ | `minder-rabbitmq` | Async message queue / hook bus |
| Schema Registry | `minder-schema-registry` | Apicurio schema registry |

Redis, PostgreSQL, and Qdrant can be pointed at managed cloud providers instead
of the bundled containers — see [External services](external-services.md).

### Observability

Prometheus, Grafana, Alertmanager, Jaeger, the OpenTelemetry Collector, InfluxDB,
Telegraf, and a set of Prometheus exporters make up the monitoring stack. InfluxDB
also publishes `127.0.0.1:8086` directly; the rest are internal or proxy-fronted.
See [Monitoring](monitoring.md).

### Reverse proxy and auth

- **Traefik v3** is the reverse proxy, TLS terminator, and router. Routing is
  driven by Docker labels (not exposed by default); host ports `80`/`443` serve
  traffic and the dashboard on `8081` is IP-whitelisted. Minder does not use
  Nginx or HAProxy.
- **Authelia** is enabled and enforcing by default. A Traefik `forward-auth`
  middleware gates the MinIO, api-gateway, Grafana, OpenWebUI, Jaeger, and client
  routers — an unauthenticated request to those routes is `302`-redirected to the
  Authelia portal. Full browser SSO still requires real DNS and valid TLS on the
  deployment. See [Authentication](authentication.md).

## Health checks

```bash
# Overview
bash setup.sh status
docker ps -a --filter health=unhealthy   # should be empty

# Core API
curl http://localhost:8000/health   # api-gateway
curl http://localhost:8001/health   # plugin-registry
curl http://localhost:8004/health   # rag-pipeline
curl http://localhost:8008/health   # graph-rag

# Monitoring
curl http://localhost:9090/-/healthy    # prometheus
curl http://localhost:3000/api/health   # grafana

# Ollama is internal-only — list models from inside the container
docker exec minder-ollama ollama list
```

!!! note "\"no healthcheck\" is not \"unhealthy\""
    A few containers (`minder-otel-collector`, `minder-redis-exporter`,
    `minder-rabbitmq-exporter`) have **no healthcheck by design**, because their
    base images lack the tooling a probe would need. They show blank health in
    `docker ps`; confirm them from their logs, not their health status. See
    [Troubleshooting](troubleshooting.md).

## Reverse proxy and TLS

Traefik discovers services automatically via Docker labels — no manual upstream
configuration, with load balancing across replicas, TLS termination, and a
middleware pipeline (forward-auth, IP whitelist, headers). The dashboard is at
`http://localhost:8081` (IP-whitelisted).

By default TLS uses self-signed/local certificates. For a real deployment, wire
Traefik to an ACME provider (e.g. Let's Encrypt) or supply your own certificates,
and configure real DNS for the public hostnames. Valid DNS and TLS are also what
[Authentication](authentication.md) needs for full browser SSO. For exposing an
instance beyond the local network, see [Remote access](remote-access.md).

## Resource limits

Set per-service limits in the Compose file. On a single small host (an 8 GB
Raspberry Pi 4 is a validated target), size limits conservatively — Ollama and
Neo4j are the largest RAM consumers.

```yaml
services:
  api-gateway:
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2'
        reservations:
          memory: 512M
          cpus: '0.5'
```

Services restart on failure. For pointing inference at a larger external host to
free RAM on the box, see [AI setup](ai-setup.md).

## Backups

Use the built-in commands:

```bash
bash setup.sh backup
bash setup.sh restore
```

Persistent state lives in Docker named volumes (`postgres_data`, `neo4j_data`,
`qdrant_data`, `redis_data`, `rabbitmq_data`, `minio_data`, `ollama_data`,
`influxdb_data`, …) and in MinIO buckets (including `backup-archives`). For
production, establish and **test** a real retention policy with an off-device copy
— the built-in commands snapshot state but do not, by themselves, ship it
off-box.

## Upgrades

Image versions are pinned in the Compose file. Check for drift and apply updates
with the setup CLI:

```bash
bash setup.sh update --check   # report available image updates
bash setup.sh update           # pull + recreate pinned images
```

!!! warning "Database major-version upgrades need care"
    A PostgreSQL major-version bump requires a data migration, not just an image
    swap. Plan and test it separately — see [PostgreSQL](postgresql.md).

## Scaling

Traefik load-balances across replicas of a scaled **stateless** service:

```bash
docker compose up -d --scale api-gateway=2
```

Scaling only makes sense for stateless services; stateful backends and the
profile-gated Ollama container are not scaled this way. On a single small host,
headroom for horizontal scaling is limited.

## Troubleshooting

```bash
# Status / health
bash setup.sh status
docker compose ps

# Logs
docker logs minder-<service> --tail 100 -f

# Restart / recreate a service
docker compose restart <service>
docker compose up -d --force-recreate <service>

# Resource usage
docker stats --no-stream

# Diagnostics
bash setup.sh doctor
```

Common production issues:

- **Service won't start** — check `docker logs <service>`; confirm `./.env` is
  populated and dependencies are healthy.
- **High memory / OOM** — reduce Ollama model size or offload inference
  (`bash setup.sh ollama-mode`); review resource limits.
- **DB connection issues** — `docker exec minder-postgres pg_isready -U "$POSTGRES_USER"`;
  if the Postgres password and volume diverge, run
  `bash setup.sh sync-postgres-password`.
- **Auth not enforced on a route** — confirm the route is one of the
  forward-auth-gated routers; Authelia is enabled and enforcing by default.

The full catalog of fixes is in [Troubleshooting](troubleshooting.md).

## Production hardening checklist

A default install is development-grade. The items below are the gap between it and
a hardened, public-facing deployment:

- [ ] Configure real DNS for the public hostnames.
- [ ] Replace self-signed certificates with real TLS (ACME via Traefik, or your
      own certs) so Authelia SSO/MFA works as full browser SSO.
- [ ] Add firewall rules; restrict which ports are reachable externally
      (remember host ports are loopback-bound, so protect access to the box).
- [ ] Rotate secrets. Optionally layer an external secrets manager on top of the
      `./.env` mechanism — this is not built in.
- [ ] Configure Alertmanager receivers (email / Slack / PagerDuty) — these are
      placeholders by default.
- [ ] Establish and test a backup retention policy with an off-device copy.
- [ ] Review resource limits for expected load.

### Known limitations (do not assume these exist)

- **Uniform RBAC** — role-based checks cover a specific set of admin-only actions
  and the organizations/teams surface, but they are **not yet uniform**: most
  other write endpoints still only check that a JWT is valid, not its role. See
  [Authentication](authentication.md#roles-partially-enforced). Don't build
  workflows that assume broader per-role enforcement.
- **High availability / multi-server / multi-region** — Minder targets a
  single-host deployment. Any HA or cluster-orchestrated topology is aspirational,
  not current.
- **Centralized logging** — logs are per-container Docker logs; no ELK/Loki
  aggregation ships by default.
- **Published performance benchmarks** — none are quoted here because none have
  been measured for your hardware. Measure with the monitoring stack before making
  any SLO claims — see [Performance](performance.md).

## Related pages

- [Self-hosting](self-hosting.md) — first-time install and capability bundles.
- [AI setup](ai-setup.md) — inference modes and models.
- [Authentication](authentication.md) — SSO, JWTs, roles, Traefik/Authelia.
- [External services](external-services.md) — managed data backends.
- [Remote access](remote-access.md) — reaching an instance beyond the LAN.
- [Troubleshooting](troubleshooting.md) — diagnostics and common fixes.
