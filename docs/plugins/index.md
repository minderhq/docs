# Plugins

Minder is extended by **plugins** — and they run **no arbitrary uploaded code**.
A plugin is a declarative handler the platform drives: it can pull data, expose
AI tools for function-calling, ingest webhooks, and more. Safety is a design
property, not a scanner bolted on afterward.

## Data sources & Talents

Plugins come in two shapes — a plugin can be one, the other, or both:

- **Data sources** collect a series over time and store it (prices into
  InfluxDB, headlines, repo stars…), which the platform charts and searches.
  Most of the catalog is this shape.
- **Talents** are AI capabilities the LLM can call — a plugin's `AI_TOOLS`
  exposed for function-calling (`get_weather`, `wiki_summary`, `convert`). A
  Talent needs no storage; it's a pure request/response ability that grounds or
  extends the model (`category: "ai-tool"`, e.g. the
  [`wikipedia`](https://github.com/minderhq/plugins/tree/main/wikipedia)
  plugin). Talents are the unit the marketplace is built to discover and sell.

The technical layer stays "AI tools" (the function-calling schema); **"Talent"**
is the product name for a sellable one. See the
[authoring guide](authoring.md#extension-points-class-attributes) for `AI_TOOLS`.

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
