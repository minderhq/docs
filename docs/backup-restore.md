# Backup & restore

Minder ships built-in backup and restore commands that snapshot the platform's
persistent state. You can drive them three ways — the `setup.sh` CLI on the host,
the control-plane UI, or the admin-only Backups API — but they all queue the same
host-side work. This page covers what is captured, how to run and confirm a
restore, and the retention policy you must layer on top.

!!! note "Admin-only"
    Backup and restore touch every service's data, so they are **admin-only** on
    every surface (CLI on the host, the `/platform/backups` UI, and the
    `/v1/backups` API).

## What is captured

Persistent state lives in two places, both covered by a backup:

- **Docker named volumes** — `postgres_data`, `neo4j_data`, `qdrant_data`,
  `redis_data`, `rabbitmq_data`, `minio_data`, `ollama_data`, `influxdb_data`,
  and the rest.
- **MinIO object-store buckets**, including the `backup-archives` bucket where
  archives are kept.

See [Production](production.md#backups) for the same list in the operational
context.

## Running a backup

=== "CLI (on the host)"

    ```bash
    bash setup.sh backup
    bash setup.sh restore
    ```

=== "UI"

    The **Backups** page (`/platform/backups`) lets you enqueue a full backup of
    the platform's data and, when you need it, restore from one. Both run as
    background jobs listed under **Recent Jobs** with their status. See
    [Using Minder](using-minder.md).

=== "API"

    Enqueue a backup with `POST /v1/backups` (see the table below). Reachable
    through the API Gateway; admin-only. See the
    [API reference](api-reference.md).

## The job-queue model

The `/v1/backups` API is a **job-queue front end** for the host-level
backup/restore tooling — it is **not** a Docker orchestrator, and the router
never talks to Docker directly. It reads and writes small JSON job files in a
bind-mounted directory; a **host-side cron job** does the real work. Enqueuing a
backup or restore returns **`202`** with a job record, and you poll the job
endpoints for status.

| Method | Path | Description |
|--------|------|-------------|
| GET  | `/v1/backups` | List existing backup archives |
| POST | `/v1/backups` | Enqueue a backup job (**202**, returns the job record) |
| GET  | `/v1/backups/jobs` | List recent backup/restore jobs (most recent 50) |
| GET  | `/v1/backups/jobs/{job_id}` | Get one job's status/output (`404` if unknown) |
| POST | `/v1/backups/{name}/restore` | Enqueue a restore job (**202**) — see below |

All endpoints are admin-only and reachable through the gateway.

## Restore

!!! warning "Restore is destructive"
    A restore overwrites live platform data. Beyond admin-only gating, the API
    guards it: the request body must echo
    `{"confirm_filename": "<name>"}` **exactly matching the archive** being
    restored. In the UI the same guard appears as an explicit confirm step.

`POST /v1/backups/{name}/restore` enqueues a restore job (**202**). The
`confirm_filename` in the body must match the archive `name` in the path, or the
restore will not proceed.

### Full restore after a failed upgrade

If an upgrade fails and you need to roll the whole stack back to a pre-upgrade
backup, take the platform down, restore, and bring it back up:

```bash
bash setup.sh stop
bash setup.sh restore
bash setup.sh start
```

The full procedure and its place in the upgrade runbook is in
[Upgrading](upgrades.md#full-restore).

## PostgreSQL caveat

A stateful database restore needs care. For a routine full restore the built-in
commands are enough, but a PostgreSQL **major-version** change is a logical
dump-and-restore (`pg_dumpall`), not a raw volume swap — the new server refuses an
old-format data directory. Follow the dedicated procedure in
[PostgreSQL](postgresql.md#major-version-upgrade-occasional).

!!! tip "If the Postgres password and volume diverge"
    After restoring or rebuilding the database volume, a mismatch between the
    configured password and the volume can block connections. Reconcile it with:

    ```bash
    bash setup.sh sync-postgres-password
    ```

## Off-device retention

!!! warning "Snapshots are not a retention policy"
    `bash setup.sh backup`/`restore` snapshot state into `backup-archives`, but
    they do **not**, by themselves, ship it off-box. For production, **establish
    and test a real retention policy with an off-device copy** — an archive that
    only lives on the same host does not survive that host being lost.

## In the UI / via the API / via the CLI

The same backup and restore operations are available three ways — pick whichever
fits what you're doing:

- **UI** — `/platform/backups`, with **Recent Jobs** ([Using Minder](using-minder.md)).
- **API** — the admin-only `/v1/backups` endpoints ([API reference](api-reference.md)).
- **CLI** — `bash setup.sh backup` / `bash setup.sh restore` on the host
  ([Production](production.md#backups)).

## Related pages

- [Production](production.md) — operational topology and the backups overview.
- [Upgrading](upgrades.md) — rollback and full restore in the upgrade runbook.
- [PostgreSQL](postgresql.md) — dump/restore and the major-version upgrade.
- [API reference](api-reference.md) — the Plugin Registry backups endpoints.
- [Using Minder](using-minder.md) — the Backups page in the control-plane UI.
