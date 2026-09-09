# Service access

Once a Minder instance is running ([Self-hosting](self-hosting.md)), day-to-day
you need to *reach* its services: hit a health endpoint, open a management UI,
tail a log, or drop into a database shell. This page is the practical reference
for **how to get to each running service** and which access path applies.

For the full deployment topology and hardening steps, see
[Production](production.md); for a tour of the browser control-plane, see
[Using Minder](using-minder.md). This page is about access, not topology.

## The access model

A full install runs on the order of three dozen containers (a couple are
failover-mode sidecars that stay inactive by default), all on the internal
`minder-network` Docker network. There are three ways to reach a service:

1. **Loopback host port** — a port published on `127.0.0.1`. Reachable **on the
   box** (`curl http://localhost:<port>`) for ops and health, but **not** from
   other machines. Use this for local health and debugging.
2. **Reverse-proxy route** — a `*.minder.local` virtual host served by Traefik on
   ports `80`/`443`. This is the intended path for external access, and it is
   gated by Authelia forward-auth. See [Authentication](authentication.md).
3. **Internal-only** — no host port at all. Reachable from inside the Docker
   network by container name, or through a reverse-proxy route where one exists.

!!! note "Host ports are loopback-bound"
    Every non-proxy host port is bound to `127.0.0.1`, so it is reachable on the
    host only. External access is meant to go through the reverse proxy, which
    sits in front of the auth gate. Because the loopback ports sit *behind* that
    gate, treat access to the box itself as privileged and firewall it
    accordingly.

## Host ports (loopback)

These publish a port on `127.0.0.1` — reach them on the host at
`http://localhost:<port>`. Each core API also serves interactive docs at `/docs`.

### Core API services (FastAPI)

| Service | Container | Host port |
|---|---|---|
| API Gateway | `minder-api-gateway` | 8000 |
| Plugin Registry | `minder-plugin-registry` | 8001 |
| Marketplace | `minder-marketplace` | 8002 |
| Orchestrator | `minder-orchestrator` | 8003 |
| RAG Pipeline | `minder-rag-pipeline` | 8004 |
| Model Management | `minder-model-management` | 8005 |
| TTS / STT | `minder-tts-stt` | internal by default |
| Graph-RAG | `minder-graph-rag` | 8008 |

The TTS/STT service does not publish a host port in the default configuration —
reach it through the API Gateway proxy. It only binds a host port when its
optional failover profile is active.

### Observability and proxy

| Service | Container | Host port(s) |
|---|---|---|
| Grafana | `minder-grafana` | 3000 |
| Prometheus | `minder-prometheus` | 9090 |
| Alertmanager | `minder-alertmanager` | 9093 |
| InfluxDB | `minder-influxdb` | 8086 |
| OTel Collector | `minder-otel-collector` | OTLP ingestion (gRPC/HTTP) — trace ingestion, not a browsable UI |
| Traefik | `minder-traefik` | 80, 443, 8081 (dashboard, IP-whitelisted) |

The Jaeger tracing UI has **no loopback port**: its base image ships with no
authentication of its own, so publishing it on the host would expose the full
trace UI to anyone with host access, with no credential check. Reach it only
through its reverse-proxy route (below), where Authelia gates it. Its
trace-ingestion ports are unaffected — they are not a browsable admin surface.
See [Monitoring](monitoring.md).

```bash
# Reach loopback services on the host
curl http://localhost:8000/health     # API Gateway
curl http://localhost:8001/v1/plugins # Plugin Registry (list plugins)
curl http://localhost:8004/health     # RAG Pipeline
# open http://localhost:3000 for Grafana
```

## Reaching services through the reverse proxy

Traefik routes selected services on `*.minder.local` virtual hosts over ports
`80`/`443`. It is not exposed by default — only labeled services are routed:

| Host | Backend |
|---|---|
| `api.minder.local` | API Gateway |
| `grafana.minder.local` | Grafana |
| `jaeger.minder.local` | Jaeger UI |
| `chat.minder.local` | OpenWebUI (chat UI) |
| `minio.minder.local` | MinIO console |
| `rabbitmq.minder.local` | RabbitMQ management UI (IP-whitelisted) |
| `neo4j.minder.local` | Neo4j browser (IP-whitelisted) |

An Authelia forward-auth middleware gates the api-gateway, Grafana, OpenWebUI,
MinIO, Jaeger, and client routers: an unauthenticated request to those routes is
`302`-redirected to the Authelia portal. Because the underlying host ports are
loopback-only, there is no direct-port bypass. Full browser SSO also requires
real DNS and valid TLS on the deployment — see
[Authentication](authentication.md).

To use the `.minder.local` hostnames from another machine on your network, add
them to that machine's hosts file pointing at the host running Traefik
(for example `192.168.1.50`). For reaching an instance beyond the local network,
see [Remote access](remote-access.md).

## Internal-only services

These publish no host port. Reach them from inside the Docker network by
container name, or through a reverse-proxy route where one exists (see above).

| Service | Container | Notes |
|---|---|---|
| PostgreSQL | `minder-postgres` | Relational data (primary + auxiliary databases) |
| Redis | `minder-redis` | Cache, sessions, rate-limit counters |
| Qdrant | `minder-qdrant` | Vector store for RAG |
| Neo4j | `minder-neo4j` | Graph store; browser routed via `neo4j.minder.local` |
| MinIO | `minder-minio` | S3-compatible object store; console via `minio.minder.local` |
| RabbitMQ | `minder-rabbitmq` | Message queue; management UI via `rabbitmq.minder.local` |
| Schema Registry | `minder-schema-registry` | Apicurio schema registry |
| Ollama | `minder-ollama` | Local LLM runtime (local inference mode only) |
| Prometheus exporters | postgres / redis / rabbitmq / node / cadvisor / blackbox | Scraped by Prometheus |

```bash
# From inside the Docker network, a container name resolves via Docker DNS
docker exec minder-api-gateway curl http://minder-qdrant:6333/
docker exec minder-api-gateway curl http://minder-graph-rag:8008/health
```

## Health and logs

```bash
# Overview
bash setup.sh status
docker ps -a --filter health=unhealthy   # should be empty

# Per-service health (on the host)
curl http://localhost:8000/health   # api-gateway
curl http://localhost:8004/health   # rag-pipeline
curl http://localhost:9090/-/healthy # prometheus
curl http://localhost:3000/api/health # grafana

# Logs
docker logs minder-<service> --tail 100 -f
```

!!! note "\"no healthcheck\" is not \"unhealthy\""
    A few containers (the OTel collector and the Redis and RabbitMQ exporters)
    ship **without a healthcheck by design**, because their base images lack the
    tooling a probe would need. They show blank health in `docker ps` — confirm
    them from their logs, not their health status.

## Debugging access

Use `docker exec` to reach a service or a data backend directly:

```bash
# Health of a service from inside its own container
docker exec minder-api-gateway curl http://localhost:8000/health

# Database shells
docker exec -it minder-postgres psql -U "$POSTGRES_USER"
docker exec -it minder-redis redis-cli -a "$REDIS_PASSWORD" ping

# Ollama is internal-only — list models from inside the container
docker exec minder-ollama ollama list
```

## Troubleshooting access

**404 from the reverse proxy** — confirm the target container is up and its route
is registered:

```bash
docker logs minder-traefik --tail 50
docker ps | grep <service>
```

**Cannot reach an internal service from the host** — internal-only services
(Postgres, Redis, Qdrant, Ollama, and others) publish no host port by design.
Reach them via `docker exec` into a service container, or via their reverse-proxy
route where one exists.

**A service is unreachable** — inspect and restart it:

```bash
docker ps -a --filter name=minder-<service>
docker logs minder-<service> --tail 50
bash setup.sh restart <service>
```

The full catalog of fixes is in [Troubleshooting](troubleshooting.md).

## Related pages

- [Production](production.md) — deployment topology, ports, and hardening.
- [Using Minder](using-minder.md) — the browser control-plane tour.
- [Authentication](authentication.md) — SSO, JWTs, and the Authelia auth gate.
- [Monitoring](monitoring.md) — the observability stack.
- [Remote access](remote-access.md) — reaching an instance beyond the LAN.
- [Troubleshooting](troubleshooting.md) — diagnostics and common fixes.
