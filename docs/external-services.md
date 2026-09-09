# External services

Minder is backed by a set of third-party and infrastructure services — storage
engines, an LLM runtime, observability, and a reverse proxy. Most of these run
locally in Docker as part of a self-hosted install. Three **stateful data
services — Redis, PostgreSQL, and Qdrant — can be pointed at external cloud
providers** instead of the bundled containers, entirely through environment
variables and without any code changes.

!!! note
    Inference (Ollama) can likewise be run locally or pointed at an external
    host. That is configured separately — see [AI setup](ai-setup.md). This page
    covers the **data** services. For standing up an instance in the first
    place, see [Self-hosting](self-hosting.md).

## Service overview

All services attach to an internal Docker network (`minder-network`) and are
reachable from other containers by container name. Host-published ports are
loopback-bound by default; external access goes through the reverse proxy.

| Category | Services | Purpose |
|----------|----------|---------|
| Inference | Ollama, Open WebUI | Local LLM runtime + chat UI (see [AI setup](ai-setup.md)) |
| Storage | PostgreSQL, Redis, Qdrant, Neo4j, MinIO, RabbitMQ, Schema Registry | Relational data, cache/sessions, vectors, graph, object store, message queue, schema registry |
| Observability | Prometheus, Grafana, Alertmanager, InfluxDB, Telegraf, Jaeger, OpenTelemetry Collector | Metrics, dashboards, alerting, time-series, tracing |
| Exporters | postgres-, redis-, rabbitmq-, node-exporter, cAdvisor, blackbox-exporter | Metrics sources scraped by Prometheus |
| Networking | Traefik, Authelia | Reverse proxy / TLS, and SSO / forward-auth |

Of these, only **Redis, PostgreSQL, and Qdrant** are designed to be swapped for
an external provider. Everything else is expected to run locally.

## Swappable data services

### Redis (cache, rate limiting, sessions)

Compatible with AWS ElastiCache, Redis Cloud / Redis Labs, Azure Cache for
Redis, and Google Cloud Memorystore.

```bash
# root .env
REDIS_HOST=your-redis-cluster.example.com
REDIS_PORT=6379
REDIS_PASSWORD=CHANGEME
```

Example endpoint format (AWS ElastiCache):

```bash
REDIS_HOST=minder-redis.xxxxx.use1.cache.amazonaws.com
REDIS_PORT=6379
REDIS_PASSWORD=CHANGEME
```

### PostgreSQL (primary database)

Compatible with AWS RDS, Heroku Postgres, Neon, Supabase, Google Cloud SQL, and
Railway.

```bash
# root .env
POSTGRES_HOST=your-postgres-db.example.com
POSTGRES_PORT=5432
POSTGRES_USER=minder
POSTGRES_PASSWORD=CHANGEME
POSTGRES_DB=minder
POSTGRES_SSLMODE=require
```

Some providers expose a single connection URL instead of discrete fields:

```bash
DATABASE_URL=postgresql://user:CHANGEME@host:5432/dbname
```

### Qdrant (vector database)

Compatible with Qdrant Cloud or a self-hosted Qdrant cluster. Qdrant backs the
RAG pipeline — see [RAG methods](rag-methods.md).

```bash
# root .env
QDRANT_HOST=your-cluster.qdrant.io
QDRANT_PORT=6333
QDRANT_API_KEY=CHANGEME
QDRANT_TLS=true
```

## Usage scenarios

### Local (default)

All services run in local Docker containers. Internal-only services are reached
by container name on the Docker network:

```bash
bash setup.sh start
```

| Service | Internal address |
|---------|------------------|
| PostgreSQL | `minder-postgres:5432` |
| Redis | `minder-redis:6379` |
| Qdrant | `minder-qdrant:6333` |

### Hybrid (local + external)

Run some services locally and point others at the cloud — for example, local
Redis with a managed PostgreSQL:

```bash
# root .env
POSTGRES_HOST=your-postgres-db.example.com
POSTGRES_PASSWORD=CHANGEME

REDIS_HOST=localhost
REDIS_PORT=6379
```

### Full external

Point every data service at an external provider and run only the application
services locally:

```bash
# root .env
REDIS_HOST=redis-cloud.example.com
POSTGRES_HOST=your-postgres-db.example.com
QDRANT_HOST=your-cluster.qdrant.io
```

### Per-machine configuration

Minder uses **one root `.env` per machine**. Each deployment keeps its own
endpoints, so the same commands behave correctly everywhere:

```bash
# development machine — root .env
POSTGRES_HOST=localhost
REDIS_HOST=localhost
QDRANT_HOST=localhost
```

```bash
# production machine — root .env
POSTGRES_HOST=your-postgres-db.example.com
REDIS_HOST=your-redis-cluster.example.com
QDRANT_HOST=https://your-cluster.qdrant.io
```

Then, on each machine:

```bash
bash setup.sh start   # reads the root .env and starts
```

## Migrating from local to external

### Redis

1. Provision the external Redis instance.
2. Set `REDIS_HOST` / `REDIS_PASSWORD` in the root `.env`.
3. Restart the services that use it:
   ```bash
   docker compose restart api-gateway plugin-registry
   ```
4. Verify:
   ```bash
   docker logs minder-api-gateway | grep -i redis
   ```

### PostgreSQL

1. Provision the external database.
2. Initialize its schema by running the initialization SQL that ships with the
   platform against the new database.
3. Set `POSTGRES_HOST` / `POSTGRES_PASSWORD` in the root `.env`.
4. Restart the affected services:
   ```bash
   docker compose restart plugin-registry
   ```

### Qdrant

1. Create a Qdrant Cloud cluster (or stand up your own) and obtain its URL and
   API key.
2. Set `QDRANT_HOST` / `QDRANT_API_KEY` in the root `.env`.
3. Restart the RAG pipeline:
   ```bash
   docker compose restart rag-pipeline
   ```

## Troubleshooting

**Connection refused** — confirm the endpoint is reachable
(`telnet <host> <port>`), that the provider's firewall / security group allows
your IP, and that the password is correct.

**TLS / SSL errors** (e.g. `WRONG_VERSION_NUMBER`) — enable TLS explicitly:

```bash
REDIS_TLS=true
POSTGRES_SSLMODE=require
```

**DNS resolution** (`Name or service not known`) — double-check the hostname and
DNS; try the provider's IP directly to isolate the issue.

!!! warning "Reachability is resolved from inside the containers"
    Application services connect to these endpoints from *within* the Docker
    network, not from your host shell. An address that resolves on the host — or
    only on your LAN's DNS — can still be unreachable from a container. Verify
    from inside a container, e.g.:

    ```bash
    docker exec minder-plugin-registry ping -c 3 your-postgres-db.example.com
    ```

## Best practices

**Security**

- Never commit a `.env` containing real credentials.
- Use different credentials per environment (dev / staging / production) and
  rotate them regularly.
- Prefer provider IAM authentication where available (e.g. AWS RDS).

**High availability**

- Use clustered / multi-AZ deployments for Redis and PostgreSQL in production.
- Enable connection pooling and application-side retry logic.
- Use Qdrant replication in production.

**Performance**

- Co-locate services in the same region to minimize latency.
- Enable Redis persistence and PostgreSQL connection pooling.

**Monitoring**

- Health-check external endpoints and alert on connection failures.
- Track resource usage (CPU, memory, connection counts) on the provider side.

## Quick reference

| Service | Variable | Local default | Example external |
|---------|----------|---------------|------------------|
| Redis | `REDIS_HOST` | `minder-redis` | `your-redis-cluster.example.com` |
| PostgreSQL | `POSTGRES_HOST` | `minder-postgres` | `your-postgres-db.example.com` |
| Qdrant | `QDRANT_HOST` | `minder-qdrant` | `your-cluster.qdrant.io` |
