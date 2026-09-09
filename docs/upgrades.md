# Upgrading

Minder's services run as pinned Docker images. Upgrading means moving those images
to newer tags, recreating the containers, and verifying the stack came back healthy.
This page is the **detailed image-upgrade runbook**: pre-checks, backup, the update
flow, verification, rollback, and per-image caveats.

!!! note "Where this fits"
    [Production](production.md#upgrades) has the one-paragraph summary of the same
    `setup.sh update` flow. This page is the long form. For a **PostgreSQL
    major-version** change — which needs a dump/restore, not an image swap — follow
    [PostgreSQL](postgresql.md#major-version-upgrade-occasional) instead; it is not
    repeated here.

## How versions are managed

- **The Compose file that ships with the platform is the single source of truth for
  image versions.** Tags are pinned directly on each service's `image:` line. You
  change a version by editing that tag — there is no template or regeneration step.
- **`bash setup.sh update --check`** derives the pinned image list and produces a
  **drift report**: installed vs. pinned vs. latest available upstream. It only
  reports; it does not edit anything.
- **`bash setup.sh update`** performs the update flow (pull + recreate) for the
  pinned images.
- The project may also run a periodic upstream check that opens or updates a
  tracking note when newer images are available. Applying a bump is always a
  deliberate human step: edit the pinned tag, then run `bash setup.sh update`.

!!! tip "Bottom line"
    To change an image version: edit the `image:` tag in the Compose file, then
    apply with `bash setup.sh update`.

## Pre-upgrade checklist

- [ ] Enough free disk for a backup (databases can be several GB).
- [ ] All services healthy: `bash setup.sh status` and `bash setup.sh doctor`.
- [ ] A fresh backup taken: `bash setup.sh backup` (see [Production](production.md#backups)).
- [ ] The proposed version changes reviewed via the drift report.
- [ ] Current versions noted so you can roll back.

```bash
# Report installed vs. pinned vs. latest available (drift report)
bash setup.sh update --check
```

## Standard image upgrade (no data migration)

Most image bumps — Grafana, Prometheus, Traefik, Redis minor versions, the metric
exporters, and so on — are safe in-place upgrades: the on-disk volume format stays
compatible, so pulling the new tag and recreating the container is enough.

### 1. Back up

```bash
bash setup.sh backup
```

### 2. Update the pinned version

Edit the service's `image:` tag in the Compose file. For example:

```yaml
services:
  grafana:
    image: grafana/grafana:CHANGEME   # set the new pinned tag
```

### 3. Apply

```bash
# Pull + recreate per the pinned Compose file
bash setup.sh update

# Or target a single service manually:
docker compose pull grafana
docker compose up -d grafana
```

### 4. Verify

```bash
bash setup.sh status
docker ps --format "table {{.Names}}\t{{.Status}}"
```

!!! note "\"no healthcheck\" is not \"unhealthy\""
    `minder-otel-collector`, `minder-redis-exporter`, and `minder-rabbitmq-exporter`
    have **no healthcheck by design** — they show blank health in `docker ps`, which
    is expected. Confirm them from their logs, not their health status. See
    [Troubleshooting](troubleshooting.md).

## Verification

After any upgrade, confirm the core APIs, monitoring, and data stores are back:

```bash
# Core API health (loopback-bound; run on the host)
curl -f http://localhost:8000/health    # api-gateway
curl -f http://localhost:8001/health    # plugin-registry

# Monitoring
curl -f http://localhost:9090/-/healthy  # prometheus
curl -f http://localhost:3000/api/health # grafana

# Data stores (internal — exec into the container)
docker exec minder-postgres pg_isready -U "$POSTGRES_USER"
docker exec minder-redis redis-cli -a "$REDIS_PASSWORD" ping

# Overall diagnostics
bash setup.sh doctor

# Resource usage
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

For deeper diagnostics and common post-upgrade symptoms, see
[Troubleshooting](troubleshooting.md) and [Monitoring](monitoring.md).

## Rollback

### Revert an image bump

```bash
# Restore the previous pinned tag(s) in the Compose file
# (hand-edit the image: tag back to the previous version, or revert your change)
bash setup.sh update
```

### Full restore

If a broader failure occurs, restore from the pre-upgrade backup:

```bash
bash setup.sh stop
bash setup.sh restore
bash setup.sh start
```

Backups are produced by `bash setup.sh backup` and restored by
`bash setup.sh restore`. Persistent state lives in Docker named volumes and in
object-store buckets (including `backup-archives`); see
[Production](production.md#backups) for what is captured and the retention guidance
you should layer on top.

## Per-image caveats

### Databases and stateful stores

A **PostgreSQL major-version** upgrade cannot be done by swapping the image — the new
server refuses an old-format data directory, so it needs a logical dump and restore.
That full procedure, including rollback, lives in
[PostgreSQL](postgresql.md#major-version-upgrade-occasional). Follow it there; do not
treat a major Postgres bump as a standard image upgrade.

Other stateful stores (Redis, Qdrant, Neo4j, MinIO, RabbitMQ) are generally safe
in-place across minor versions. Before any **major** bump of a stateful store, check
that upstream keeps the on-disk format compatible; if not, plan a backup-and-restore
the same way. When pointing Minder at a managed provider instead of the bundled
container, follow that provider's own upgrade path — see
[External services](external-services.md).

### CPU architecture

Minder targets modest hardware, and an 8 GB Raspberry Pi 4 (arm64) is a validated
first-class target. Every pinned image must therefore publish an `arm64` /
`linux/arm64` variant — only propose tags that ship an ARM build. Memory is the
tightest constraint on small hosts: Ollama and Neo4j are the largest RAM consumers,
so check headroom (`free -h`) before and after an upgrade. To free RAM by offloading
inference to a larger external host, see [AI setup](ai-setup.md).

## Success criteria

- [ ] All containers `healthy` (or "no healthcheck" for the three by-design
      exceptions); none stuck `restarting`.
- [ ] Core API and monitoring health checks pass.
- [ ] Database data intact (row counts match pre-upgrade for key tables).
- [ ] No error spikes in the logs for ~30 minutes:
      `docker logs minder-<service> --tail 100 -f`.
- [ ] `bash setup.sh doctor` is clean.

## Related pages

- [Production](production.md) — operational topology, backups, hardening.
- [PostgreSQL](postgresql.md) — schema migrations and the major-version upgrade.
- [Self-hosting](self-hosting.md) — first-time install and capability bundles.
- [Troubleshooting](troubleshooting.md) — diagnostics and common fixes.
- [Monitoring](monitoring.md) — health and metrics after an upgrade.
