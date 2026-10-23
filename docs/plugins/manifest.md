# Manifest plugins

A **manifest** is a YAML file that supplies *parameters only* — no code at all.
There are two kinds, and both use `apiVersion: minder.dev/v1alpha1` and
`kind: Plugin`:

- **Trigger + action** (`spec.trigger` + `spec.action`): pair one trigger with
  one action, for simple ingestion and automation.
- **Declarative connector (T0)** (`spec.runtime: {kind: declarative}`): call
  HTTP APIs on hosts the manifest declares, and expose the results as AI tools,
  actions and a scheduled collection, with no plugin code. See
  [Declarative connectors](#declarative-connectors-t0).

`minder-plugin validate` checks both kinds against their published schemas.

| Triggers | Actions |
|----------|---------|
| `webhook`: a signed inbound request (`secretRef` is required) | `store-vector`: embed text into a collection |
| `schedule`: a five-field cron expression | `http-request`: a templated outbound HTTP call |
| `poll`: a periodic HTTP GET | `plugin-action`: another installed plugin's declared action |

Outbound URLs (`poll`, `http-request`) are checked at runtime: private, loopback,
link-local and cloud-metadata addresses are refused, and redirects are
re-checked. For anything richer than either kind, write a [code plugin](authoring.md).

## Example

```yaml
apiVersion: minder.dev/v1alpha1
kind: Plugin
metadata:
  name: discord-ingestor
  version: 0.1.0
  description: Ingest Discord messages to Qdrant
spec:
  trigger:
    type: webhook
    webhook:
      path: /discord/webhook
      secretRef: discord.webhook.secret
  action:
    type: store-vector
    store:
      collection: discord-messages
      input:
        text: "{{ .content }}"
        metadata:
          author: "{{ .author.username }}"
          timestamp: "{{ .timestamp }}"
```

## Validate

```python
from minder_plugin_sdk import validate_manifest
validate_manifest(open("manifest.yaml").read())   # raises ManifestError if invalid
```

Or: `minder-plugin validate manifest.yaml`. The JSON Schema (draft-07) ships with
the SDK. Install the optional `jsonschema` extra for full validation and `pyyaml`
to parse YAML text.

## Schemas

The SDK's bundled schemas are the manifest contract; Minder validates against the
same files. Each is published at its `$id`:

| Manifest | Schema |
|----------|--------|
| Manifest plugin (one trigger, one action) | [`manifest.v1alpha1.json`](https://minderhq.github.io/docs/schemas/manifest.v1alpha1.json) |
| Declarative connector (`spec.runtime.kind: declarative`) | [`declarative-manifest.v1alpha1.json`](https://minderhq.github.io/docs/schemas/declarative-manifest.v1alpha1.json) |

A declarative connector describes HTTP tools, actions, a schedule and a signed
webhook as data, and Minder's connector engine runs it. Its schema checks the
structure. The engine also checks what JSON Schema cannot express when it loads
the manifest, for example that every base URL's host is in the declared `egress`
and that every JMESPath expression compiles. `validate_manifest` and
`minder-plugin validate` pick the schema from `spec.runtime.kind`.

Point an editor at a schema with a modeline in the YAML file:

```yaml
# yaml-language-server: $schema=https://minderhq.github.io/docs/schemas/manifest.v1alpha1.json
```

### Fetch the schema live

A running plugin-registry also serves the same JSON Schema, unauthenticated, at
**`GET /v1/plugins/schema`** — fetch it to validate a manifest from any language,
tool, or CI pipeline without the Python SDK:

```bash
curl -s https://<your-instance>/v1/plugins/schema | jq .
```

The response is a self-contained draft-07 document (carries its own `$schema` and
`title`), so any JSON-Schema validator can consume it directly.

> **Why `v1alpha1`, not `v1`?** This declarative format is intentionally versioned
> separately from the Python plugin contract's `minder.dev/v1` (see
> [contract reference](contract.md)) — the manifest schema is still experimental
> and may gain breaking changes as more trigger/action types are added, while the
> Python `PluginMetadata.api_version` is stable. Don't copy one string into the
> other's field; `validate_manifest` only accepts `minder.dev/v1alpha1` here.

## Declarative connectors (T0)

A declarative connector is hosted by the platform's `declarative` runner group,
which only ever reads the plugin's `manifest.yml` (or `.yaml`/`.json`) and never
imports any code. The manifest's `metadata.name` must equal its directory name.
The platform validates and compiles a manifest when it loads it, and refuses an
invalid one (it is listed as failed, with the reason in the log).

A connector manifest declares, under `spec`:

- `runtime: {kind: declarative, engine: 1}`;
- `display`, `config_schema` and `limits` (timeout, response size, concurrency,
  rate);
- `egress`: the hosts it may call (an exact host, or `*.example.com` for any
  subdomain). Calls use https unless a host declares `http: true`;
- `bases`: named base URLs, with optional `none`, `api_key`, `bearer` or `basic`
  auth that reads the plugin's per-tenant secrets;
- `tools` (AI tools) and `actions`, each with JSON Schema `parameters`, optional
  `vars` and up to three chained `steps` (a request plus a JMESPath `select`),
  and an `output` mapping. A tool declares its `effect` like any other tool;
- `timeseries`, to name the measurements a scheduled collection writes.

Placeholders are dotted references only (`{params.q}`, `{config.LANG}`,
`{item}`, `{steps.search.id}`), encoded for the part of the URL they land in,
and secrets are never in their context. A failure raises, unless the operation
declares an `error_output` mapping or a fixed `on_error` value.

The engine enforces each manifest's own egress on every hop: the host must be
declared, every resolved address must be public, the connection is pinned to the
address that was checked, and redirects are re-checked. It also caps steps,
fan-out, pages, redirects, requests per invocation, and the time and size of each
request. A connector can therefore reach only the public hosts it declares.
