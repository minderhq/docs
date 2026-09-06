# Manifest plugins

For simple **`webhook → store-vector`** ingestion there's a separate, fully
declarative mechanism: a YAML **manifest** that supplies *parameters only* — no
code at all. Reach for it when you just need to pipe a webhook into the vector
store; for anything richer, write a [module plugin](authoring.md).

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
