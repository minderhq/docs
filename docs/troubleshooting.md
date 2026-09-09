# Troubleshooting

Common problems and fixes when running a self-hosted Minder instance. Every
service runs as a Docker container named `minder-<service>`, provisioned and
managed through the `bash setup.sh` entrypoint (see
[Self-hosting](self-hosting.md)). Configuration lives in the root `./.env`, which
is the single source of truth — edit it there, then apply with a restart.

!!! note
    Host ports are loopback-bound by default, so `curl http://localhost:<port>`
    works on the box itself. The commands below assume you are on the host, in
    your Minder checkout.

## A service won't start

Check the logs first:

```bash
bash setup.sh logs <service>
# or
docker logs minder-<service> --tail 100
```

Common causes:

1. A configuration error or missing environment variable — check the root `./.env`.
2. A dependency (Postgres, Redis, Qdrant, Neo4j) that isn't ready yet.
3. A port conflict on the host.
4. Insufficient resources — on modest hardware such as a Raspberry Pi 4, RAM is tight.

Recreate a service after fixing its config:

```bash
bash setup.sh restart <service>
```

Find what is holding a conflicting port:

```bash
lsof -i :8000
# then stop the offending process, or adjust the port mapping
```

## A service shows "no-healthcheck" — is it broken?

Usually **no**. A few containers have **no healthcheck by design**, because their
base images lack the tooling a health probe would need:

- `minder-otel-collector`
- `minder-redis-exporter`
- `minder-rabbitmq-exporter`

`docker ps` shows these with no health status. That is expected — it is not an
"unhealthy" state. Confirm they are working from their logs, not their health:

```bash
docker logs minder-otel-collector --tail 30
docker logs minder-redis-exporter --tail 30
docker logs minder-rabbitmq-exporter --tail 30
```

## Database issues

### Can't connect to PostgreSQL

```bash
docker exec minder-postgres pg_isready -U minder
docker exec minder-postgres psql -U minder -c "SELECT 1;"
docker logs minder-postgres --tail 50
```

### Migration failed

```bash
# Re-run migrations
bash setup.sh migrate

# Or apply a SQL file directly
docker exec -i minder-postgres psql -U minder -d minder < migration.sql
```

!!! warning "Fresh reset destroys data"
    A full reset wipes every volume and cannot be undone. Only use it on a
    development instance you are willing to lose.

    ```bash
    docker compose down -v      # deletes all data
    bash setup.sh start
    ```

## Redis authentication

**Symptom:**

```
AUTH failed: WRONGPASS invalid username-password pair or Redis is loading from disk
```

**Cause:** the password being used doesn't match the one Redis is running with.

**Fix / verify:**

```bash
# Confirm the password in the root .env (source of truth)
grep REDIS_PASSWORD .env

# Test with it
docker exec minder-redis redis-cli -a "$REDIS_PASSWORD" ping   # -> PONG

# If needed, restart Redis
bash setup.sh restart redis
docker logs minder-redis --tail 50
```

!!! note
    Always edit the **root `./.env`**. Any generated per-service env file is
    rewritten from the root file by `setup.sh` on every start/restart, so edits
    there are lost.

## Neo4j not ready

Neo4j takes time to start. It backs the marketplace (plugin dependency graph) and
graph-RAG (knowledge graph).

```bash
# Wait for it to finish starting
docker logs minder-neo4j --tail 50 | grep -i "started\|remote interface"

# Test a query
docker exec minder-neo4j cypher-shell -u neo4j -p "$NEO4J_PASSWORD" "RETURN 1;"

# Check the password
grep NEO4J .env
```

## Plugin not loading

The first-party module plugins ship with the platform and are loaded from disk on
startup, so a clean install already lists them — the plugin list should not be
empty. See [Plugins](plugins/index.md) for the plugin model (manifest-based, with
no arbitrary code execution).

```bash
# Registry health and plugin list
curl http://localhost:8001/health
curl http://localhost:8001/plugins

# Registry logs
docker logs minder-plugin-registry --tail 50
```

## Memory / resource pressure

```bash
# Per-container usage
docker stats --no-stream

# Reclaim space from unused Docker resources
docker system prune -a

# Look for OOM kills
docker logs minder-<service> | grep -i "memory\|oom\|out of"
```

On low-RAM hosts, the AI services (RAG pipeline, model management) and Ollama
inference are the heaviest. Stop non-critical services temporarily if the host is
starved. See [AI setup](ai-setup.md) for pointing inference at an external Ollama
to offload the box.

## Services can't communicate

```bash
# Network exists?
docker network ls | grep minder

# Service-to-service check (by container name)
docker exec minder-api-gateway curl -s http://minder-rag-pipeline:8004/health

# Bring the stack back consistently
bash setup.sh restart
```

Services address each other by container name on the `minder-network` Docker
network.

## Routing returns 404

Minder fronts its services with **Traefik v3**. Only services with the right
Docker labels are routed, so a missing label (not a down service) is a common
cause of a 404.

```bash
docker logs minder-traefik --tail 50
bash setup.sh restart traefik
```

!!! note "Auth redirects are expected on protected routes"
    Several routes sit behind Authelia forward-auth. Unauthenticated requests to
    them are redirected (`302`) to the Authelia login portal — that is working as
    intended, not an error. Full browser SSO still requires real DNS and valid
    TLS on the deployment (see [Using Minder](using-minder.md)).

## Slow API responses

```bash
# Resource usage
docker stats --no-stream

# Turn up logging: set LOG_LEVEL=DEBUG in the root ./.env, then
bash setup.sh restart <service>

# Check DB connection count
docker exec minder-postgres psql -U minder -c \
  "SELECT count(*) FROM pg_stat_activity;"
```

## Getting help

Collect diagnostics before asking for help:

```bash
bash setup.sh status > status.txt
bash setup.sh doctor > doctor.txt
docker ps -a > containers.txt
docker stats --no-stream > stats.txt
```

### Useful commands

```bash
docker ps                               # service status
docker stats --no-stream                # resource usage
docker logs minder-<service> --tail 50  # service logs
bash setup.sh restart <service>         # restart a service
bash setup.sh doctor                    # environment diagnostics
```
