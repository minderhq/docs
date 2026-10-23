# Authoring guide

## Install

```bash
pip install minder-plugin-sdk        # once published
# or, from source:
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
        self._last = {"value": 42}   # replace with a real fetch (add httpx to your deps)
        return self._last
```

The registry drives: `register()` → `initialize()` → `health_check()` (60s loop)
→ `collect_data()` (hourly or on demand) → `analyze()` → `shutdown()`. It does so
by calling the runner your plugin is hosted in, so your code runs in a separate
process (see [Platform context](#platform-context)).

!!! warning "The one gotcha"
    `health_check()` **must** return `{"healthy": <bool>, ...}` — monitoring reads
    `health["healthy"]`. `PluginBase` does this for you unless you override it.

## Extension points (class attributes)

| Attribute | What it does |
|-----------|--------------|
| `CONFIG_SCHEMA` | API-editable config, rendered as a form in the client. |
| `ACTIONS` / `READ_ONLY_ACTIONS` | Methods invokable over HTTP (`POST /v1/plugins/<name>/actions/<method>`). |
| `AI_TOOLS` | Ollama / OpenAI function-calling tools, mapped to an action. Declare each tool's `effect` (see [Tool effects](#tool-effects)). |
| `DISPLAY` | Branding for the plugin's card: `{label, summary, logo, color, category}` (`logo` = a lucide icon name). |
| `REQUIRES` | `{services, optional_services, bundles}` the plugin needs from the platform. |

## Tool effects

Minder runs a read-only tool straight away and pauses a tool that changes or
sends data until the user approves that exact call. So each `AI_TOOLS` entry
declares what it does:

```python
AI_TOOLS = [{
    "name": "send_invoice",
    "description": "Email an invoice to a customer.",
    "parameters": {"type": "object", "properties": {
        "to": {"type": "string", "sensitivity": "recipient"},
        "amount": {"type": "number", "sensitivity": "amount"},
    }, "required": ["to", "amount"]},
    "action": "send_invoice",
    "effect": "send",
}]
```

| `effect` | Meaning |
|----------|---------|
| `read` | No side effects. Runs without approval. |
| `write` | Changes data inside the user's own systems. |
| `send` | Data leaves to a third party or a recipient (message, email, webhook, publish). |
| `destructive` | Deletes, or changes something irreversibly (including moving money). |

Declare the most consequential thing the tool can do. A tool without `effect`
still loads, but Minder then infers a class and treats a tool it cannot
classify as `write`. Minder also does not trust a declaration that strong
evidence contradicts (a `read` tool with a recipient argument, or
`delete_note` declared `read`). Optional fields are `idempotent`, a
per-argument `sensitivity` (`recipient`, `url`, `amount`, `free_text` or
`bound:<attr>`) and `hosts`. `minder-plugin validate` fails on an invalid value
and prints a `WARN` for a missing or contradicted `effect`. See
[Tool approvals](../api-reference.md#tool-approvals) for what the user and API
callers see.

## Platform context

Your plugin runs in a process that holds **no platform credentials** and cannot
reach platform services directly. It asks the platform for what it needs through
`self.ctx`, and the platform checks every call against a grant for the current
invocation (this plugin, the invoking tenant and owner, and the capabilities the
plugin declared). A plugin that uses `self.ctx` declares `CONTRACT_VERSION = 2`.

| Call | Capability |
|------|------------|
| `await ctx.secrets.get(key)` | The plugin's own secret for the invoking tenant |
| `await ctx.plugins.invoke(plugin, action, **kw)` | Another plugin, only along a declared `dependencies` edge |
| `await ctx.documents.index(kb_id, filename, content, content_type)` | Index a document on behalf of the invoking user |
| `await ctx.timeseries.write(...)` / `.latest(...)` | Time-series data, for plugins that declare `TIMESERIES` and only their declared measurements |
| `await ctx.telegraf.*` | Only for a plugin that declares `MANAGES = ["telegraf"]`, and only in the `privileged` group |

A call that isn't granted raises `CapabilityDenied`; one that can't be served
right now raises `CapabilityUnavailable`. Rules for plugin code, checked by
`minder-plugin validate` as a lint (not a security boundary): don't read
platform secrets from the environment, don't import platform internals, return
JSON-serialisable results, and fetch URLs you don't control with the SDK's
`safe_http` (declare where you connect with `EGRESS`: `"public"`, `"lan"` or a
list of hosts). The `lan` egress, and `MANAGES`, are for the `privileged` group
only. See the [contract reference](contract.md) for the backend handle a plugin
receives.

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
