# PostgreSQL

PostgreSQL is Minder's primary relational database. This page covers two related
operational tasks:

1. Running Minder's own **schema migrations** — the routine, day-to-day operation.
2. Performing a **major PostgreSQL version upgrade** — occasional and higher risk,
   because data directories are not compatible across major versions.

It assumes you are on the host, in your Minder checkout, using the `bash setup.sh`
entrypoint (see [Self-hosting](self-hosting.md)). PostgreSQL runs as the
`minder-postgres` container on the internal Docker network; its port is not exposed
to the LAN by default.

!!! note "Running against an external / managed PostgreSQL"
    You can point Minder at a managed PostgreSQL provider (AWS RDS, Neon, Supabase,
    Google Cloud SQL, and others) instead of the bundled container, entirely through
    environment variables. See [External services](external-services.md) for the
    connection variables and the local-to-external migration steps. The schema
    migration procedure below applies the same way against an external database.

## Databases

A standard install provisions several databases in the same PostgreSQL instance:

- The **main application database** (`minder`) — users, sessions, and metadata.
- The **marketplace database** — plugin/tool catalog data.
- The **schema-registry database** — isolated storage backing the schema registry.
- One **database per data-ingestion plugin** — created and initialized by the owning
  plugin, not by the migration step.

The exact set present depends on which capability bundles and plugins you have
enabled.

## Schema migrations (routine)

Schema migrations apply application-level schema changes. They do **not** touch the
PostgreSQL server version, and this is the normal way to bring a database up to date
after an update.

```bash
# Apply pending migrations
bash setup.sh migrate
```

The `migrate` command applies pending migrations inside the service containers that
own schema. Services that manage their own schema on startup are skipped cleanly, and
per-plugin databases are initialized by their owning plugin rather than by this step —
so it is safe to run repeatedly.

If a migration fails, see [Troubleshooting](troubleshooting.md#database-issues).

## Major version upgrade (occasional)

!!! danger "A major upgrade can cause data loss"
    PostgreSQL data directories are **not** compatible across major versions — you
    must dump and restore. Do not proceed without a verified backup and a rollback
    plan.

### Pre-upgrade checklist

- [ ] Full logical backup of all databases created and verified
- [ ] Application stopped
- [ ] Maintenance window scheduled
- [ ] Anyone affected notified of downtime
- [ ] Rollback procedure understood

### Step 1 — Create a comprehensive backup

```bash
#!/bin/bash
set -euo pipefail

echo "=== PostgreSQL backup ==="

# Stop all services
bash setup.sh stop

# Or use the built-in, multi-database-aware backup command instead:
# bash setup.sh backup

BACKUP_DIR="/tmp/postgres-migration-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Logical dump of ALL databases (roles + data)
docker exec minder-postgres pg_dumpall -U minder > "$BACKUP_DIR/full_backup.sql"

# Also snapshot the raw data volume as a fallback.
# The volume name is prefixed with the compose project name; confirm the exact
# name with `docker volume ls | grep postgres` before running these commands.
docker run --rm \
  -v minder_postgres_data:/data \
  -v "$BACKUP_DIR:/backup" \
  alpine tar czf /backup/postgres_data.tar.gz -C /data .

if [ -s "$BACKUP_DIR/full_backup.sql" ] && [ -s "$BACKUP_DIR/postgres_data.tar.gz" ]; then
    echo "Backup OK: $BACKUP_DIR"
    ls -lh "$BACKUP_DIR"
else
    echo "Backup FAILED"; exit 1
fi
```

### Step 2 — Upgrade

Update the pinned `postgres:` image tag in the Compose configuration that ships with
the platform (for example `postgres:18-trixie` to the new major/variant), then rebuild
the container from the logical dump.

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${1:?Usage: $0 <backup_directory>}"
[ -f "$BACKUP_DIR/full_backup.sql" ] || { echo "Backup not found"; exit 1; }

# 1. Update the postgres image tag in the Compose configuration
#    e.g. postgres:18-trixie -> postgres:<new-major>-<variant>

# 2. Remove the old data volume (IRREVERSIBLE — you are relying on the dump)
docker volume rm minder_postgres_data

# 3. Start the new PostgreSQL
bash setup.sh start postgres

# 4. Wait for readiness
sleep 30
docker exec minder-postgres psql -U minder -c "SELECT version();"

# 5. Restore from the logical dump
cat "$BACKUP_DIR/full_backup.sql" | docker exec -i minder-postgres psql -U minder

# 6. Sanity check
docker exec minder-postgres psql -U minder -l
docker exec minder-postgres psql -U minder -d minder -c "\dt"

echo "Upgrade complete."
```

### Step 3 — Post-upgrade validation

```bash
#!/bin/bash
set -euo pipefail

echo "1. Version:"
docker exec minder-postgres psql -U minder -c "SELECT version();"

echo "2. Databases present:"
docker exec minder-postgres psql -U minder -l

echo "3. Tables in the main DB:"
docker exec minder-postgres psql -U minder -d minder -c "\dt"

echo "4. Re-apply schema migrations and start the platform:"
bash setup.sh migrate
bash setup.sh start
sleep 10
bash setup.sh status
```

### Rollback

If the upgrade fails, restore the previous image tag and the raw data snapshot:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${1:?Usage: $0 <backup_directory>}"

bash setup.sh stop

# Restore the Compose configuration to the previous postgres image tag.

# Remove the failed volume and restore the raw data snapshot
docker volume rm minder_postgres_data
docker run --rm \
  -v minder_postgres_data:/data \
  -v "$BACKUP_DIR:/backup" \
  alpine tar xzf /backup/postgres_data.tar.gz -C /data

bash setup.sh start
echo "Rollback complete."
```

## Success criteria

The upgrade is complete when:

- PostgreSQL starts on the target version.
- All expected databases exist (`docker exec minder-postgres psql -U minder -l`).
- Tables are present in each database.
- `bash setup.sh migrate` completes cleanly.
- The application connects and `bash setup.sh status` is healthy.

## Troubleshooting

### Container won't start after an upgrade

```bash
docker logs minder-postgres
```

A `database files are incompatible with server` or catalog-version error means the
data directory is from a different major version — you must restore from the logical
dump rather than the raw volume snapshot.

### Connection errors

```bash
docker network ls | grep minder
docker ps -a --filter name=minder-postgres
docker exec minder-postgres env | grep POSTGRES
```

For general database connectivity problems (including `pg_isready` checks and
connection-count inspection), see [Troubleshooting](troubleshooting.md#database-issues).
