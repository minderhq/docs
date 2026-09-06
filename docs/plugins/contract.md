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

Why this shape scales to thousands of plugin types without arbitrary code — see
[RFC 0001](https://github.com/minderhq/plugin-sdk/blob/main/docs/rfc/0001-extensible-plugin-contract.md).
