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

The host constructs each code plugin with a dict of backend handles,
`plugin_class(config)`. `PluginBase` stores it as `self.config`. Read connection
details from it instead of hard-wiring hosts, and don't read `POSTGRES_*` or
other platform credentials from the environment. A plugin that runs in a
`plugin-runner` (every catalog code plugin does) has no such credentials in its
environment at all.

| Key | Present | Shape |
|-----|---------|-------|
| `database` | only when the operator has configured the plugin database role | `{host, port, user, password, database, schema}` |

There is **no** `postgres`, `qdrant`, `neo4j` or `rabbitmq` key, and a
runner-hosted plugin gets **no** `redis` or `influxdb` key either:
`config["postgres"]` raises `KeyError`. Use the platform context
(`self.ctx`, see [Platform context](authoring.md#platform-context)) for secrets,
time-series data, document indexing and calls to other plugins. A catalog plugin
that used to write to InfluxDB directly skips that sink in a runner until it
uses `ctx.timeseries`.

`database` is a **least-privilege** Postgres handle, not the platform's own
credentials:

- It connects as a role of your plugin's own (`minder_plugins_<plugin>`),
  confined to a schema of your plugin's own (`plugin_<plugin>`), with **no
  access to the platform's tables** or to another plugin's schema.
- Its `search_path` is pinned to your schema, so an unqualified `CREATE TABLE`
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

See the [plugin trust model](../security.md#plugin-trust-model).

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
`connection` · `ui-panel`. These are what a plugin *offers*. The platform
capabilities it *uses* through `self.ctx` are a separate, granted set (see
[Platform context](authoring.md#platform-context)). Per-capability protocols (`DataSource`, `Scheduler`,
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
