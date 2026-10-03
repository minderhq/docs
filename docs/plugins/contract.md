# Contract reference

## `PluginMetadata`

Returned by `register()`. Fields the registry reads:

| Field | Type | Notes |
|-------|------|-------|
| `name` / `version` / `description` / `author` | `str` | required |
| `dependencies` / `capabilities` / `data_sources` / `databases` | `list[str]` | optional metadata |
| `registered_at` | `datetime` | defaults to now (UTC) |
| `api_version` | `str` | defaults to `minder.dev/v1` (stable — distinct from the [manifest](manifest.md)'s `minder.dev/v1alpha1`; the two surfaces version independently, don't reuse one for the other) |

## Lifecycle (`Plugin` protocol)

`register` · `initialize` · `health_check` · `collect_data` · `analyze` ·
`shutdown` — all `async`. Plus the optional `apply_config(config)`. Duck-typed on
`register`; inheriting `Plugin` / `PluginBase` is optional.

## Backend config (`self.config`)

The registry constructs each module plugin with a dict of backend handles,
`plugin_class(config)`. `PluginBase` stores it as `self.config`. Read connection
details from it instead of hard-wiring hosts, and don't read `POSTGRES_*` or other
platform credentials from the environment.

| Key | Present | Shape |
|-----|---------|-------|
| `redis` | always | `{host, port, password, db}` |
| `influxdb` | always | `{enabled, host, port, token, org, bucket}` |
| `database` | only when the operator has configured the plugin database role | `{host, port, user: "minder_plugins", password, database, schema: "plugin_data"}` |

There is **no** `postgres` or `qdrant` key; `config["postgres"]` raises `KeyError`.

`database` is a **least-privilege** Postgres handle, not the platform's own
credentials:

- The `minder_plugins` role can only use its own `plugin_data` schema and has
  **no access to the platform's tables**.
- Its `search_path` is pinned to `plugin_data`, so an unqualified `CREATE TABLE`
  lands there, owned by the plugin. `schema` is passed too, so you can
  schema-qualify your own DDL.
- **The key may be missing.** If `DB_PLUGIN_PASSWORD` isn't set, no `database`
  key is injected at all (the registry fails closed and never falls back to the
  owner credentials). Use `self.config.get("database")` and degrade gracefully,
  for example by disabling the feature or reporting it from `health_check()`.

```python
db = self.config.get("database")
if db is None:
    self.db_enabled = False      # no plugin database role on this instance
else:
    dsn = (f"postgresql://{db['user']}:{db['password']}"
           f"@{db['host']}:{db['port']}/{db['database']}")
```

The `redis` and `influxdb` handles still carry the platform's shared credentials;
see [Security](../security.md).

## Config: JSON Schema + UI Schema

The simple `CONFIG_SCHEMA` list is compiled to standard **JSON Schema** — so
config scales to any shape (nested objects, arrays, enums, conditionals) while
simple plugins stay one-liners:

```python
from minder_plugin_sdk import resolve_config_schema, validate_config

schema, ui = resolve_config_schema(plugin)   # (JSON Schema, UI hints)
validate_config(plugin, {"level": 3})         # raises ConfigError if invalid
```

Advanced plugins can declare a full `CONFIG_JSONSCHEMA` + `UI_SCHEMA` directly.

## Capabilities

A plugin declares (or the SDK infers) what it does — an **open vocabulary**;
unknown capabilities are ignored:

```python
from minder_plugin_sdk import capabilities
capabilities(plugin)   # {"config", "data-source", "ai-tools", ...}
```

`data-source` · `ai-tools` · `actions` · `webhook-ingest` · `scheduler` ·
`connection` · `ui-panel`. Per-capability protocols (`DataSource`, `Scheduler`,
`WebhookHandler`, `ConnectionProvider`, `UIPanelProvider`) let you type-check the
one you implement.

## Requirements

```python
REQUIRES = {"services": ["influxdb"], "optional_services": ["qdrant"], "bundles": ["rag"]}
```

Validated against `KNOWN_SERVICES` / `KNOWN_BUNDLES`. The platform can refuse to
enable a plugin whose hard services are missing, and offer to enable its bundles.

## Design

Why this shape scales to thousands of plugin types without plugins shipping their own UI code — see
[RFC 0001](https://github.com/minderhq/plugin-sdk/blob/main/docs/rfc/0001-extensible-plugin-contract.md).
