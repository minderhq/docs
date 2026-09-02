# Plugins

Minder is extended by **plugins** — and they run **no arbitrary uploaded code**.
A plugin is a declarative handler the platform drives: it can pull data, expose
AI tools for function-calling, ingest webhooks, and more. Safety is a design
property, not a scanner bolted on afterward.

## The three repos

| Repo | What it's for |
|------|---------------|
| [**plugin-sdk**](https://github.com/minderhq/plugin-sdk) | The authoring contract + SDK: `PluginBase`, config/JSON-Schema helpers, the `minder-plugin` CLI, a worked reference plugin. |
| [**plugin-template**](https://github.com/minderhq/plugin-template) | A “Use this template” starter repo — scaffolds a working plugin wired to the SDK. |
| [**plugins**](https://github.com/minderhq/plugins) | The catalog — first-party & community plugins, validated in CI. |

## 60-second start

```bash
# scaffold a plugin (or click "Use this template" on plugin-template)
pip install "git+https://github.com/minderhq/plugin-sdk"
minder-plugin scaffold my-plugin

# implement collect_data / actions / AI tools, then:
minder-plugin validate my_plugin.py   # contract + requirements check
minder-plugin inspect  my_plugin.py   # capabilities, config schema, requires
```

Then open a PR to [`minderhq/plugins`](https://github.com/minderhq/plugins) — CI
validates every plugin automatically.

Next: the **[authoring guide](authoring.md)**.
