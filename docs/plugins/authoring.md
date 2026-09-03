# Authoring guide

## Install

```bash
pip install "git+https://github.com/minderhq/plugin-sdk"
```

## The smallest plugin

A plugin is a **class** the registry duck-types by the presence of `register()`.
Inherit `PluginBase` for sensible lifecycle defaults so you only write what you need:

```python
from minder_plugin_sdk import PluginBase, PluginMetadata

class MyPlugin(PluginBase):
    __all__ = ["MyPlugin"]

    async def register(self) -> PluginMetadata:
        return PluginMetadata(
            name="my-plugin", version="1.0.0",
            description="what it does", author="you",
        )

    async def collect_data(self) -> dict:
        self._last = {"value": 42}   # TODO: a real fetch (add httpx to your deps)
        return self._last
```

The registry drives: `register()` → `initialize()` → `health_check()` (60s loop)
→ `collect_data()` (hourly or on demand) → `analyze()` → `shutdown()`.

!!! warning "The one gotcha"
    `health_check()` **must** return `{"healthy": <bool>, ...}` — monitoring reads
    `health["healthy"]`. `PluginBase` does this for you unless you override it.

## Extension points (class attributes)

| Attribute | What it does |
|-----------|--------------|
| `CONFIG_SCHEMA` | API-editable config, rendered as a form in the client. |
| `ACTIONS` / `READ_ONLY_ACTIONS` | Methods invokable over HTTP (`POST /v1/plugins/<name>/actions/<method>`). |
| `AI_TOOLS` | Ollama / OpenAI function-calling tools, mapped to an action. |
| `DISPLAY` | Branding for the plugin's card: `{label, summary, logo, color, category}` (`logo` = a lucide icon name). |
| `REQUIRES` | `{services, optional_services, bundles}` the plugin needs from the platform. |

## Plugin-driven UI

Each config field declares **how it renders** — the trusted client draws it (a
plugin never ships HTML):

```python
CONFIG_SCHEMA = [
    {"key": "NOTES", "type": "string", "widget": "textarea", "rows": 4},
    {"key": "ENABLED", "type": "bool", "widget": "toggle"},
    {"key": "CITY", "type": "string", "widget": "autocomplete",
     "options_action": "search_cities"},   # a READ_ONLY action returning options
]
```

Widgets: `text` · `textarea` · `code` · `secret` · `number` · `slider` ·
`toggle` · `select` · `multiselect` · `radio` · `autocomplete` · `date` ·
`datetime` · `color` · `file` · `kv-list`. Unknown widgets fall back to a text
input.

## Test it

The SDK ships a harness:

```python
from minder_plugin_sdk import check_plugin, run_lifecycle

def test_contract():
    assert check_plugin(MyPlugin()) == []   # empty ⇒ honours the contract
```

Or from the CLI: `minder-plugin validate my_plugin.py`.

See the **[contract reference](contract.md)** for the full API, and
**[publishing](publishing.md)** to get your plugin into the catalog.
