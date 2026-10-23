# Manifest plugins

For simple ingestion there's a separate, fully declarative mechanism: a YAML
**manifest** that supplies *parameters only* — no code at all. A manifest plugin
pairs one trigger with one action:

| Triggers | Actions |
|----------|---------|
| `webhook`: a signed inbound request (`secretRef` is required) | `store-vector`: embed text into a collection |
| `schedule`: a five-field cron expression | `http-request`: a templated outbound HTTP call |
| `poll`: a periodic HTTP GET | `plugin-action`: another installed plugin's declared action |

Outbound URLs (`poll`, `http-request`) are checked at runtime: private, loopback,
link-local and cloud-metadata addresses are refused, and redirects are
re-checked. For anything richer, write a [module plugin](authoring.md).

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
