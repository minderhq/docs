# Publishing a plugin

Community & first-party plugins live in the
[**minderhq/plugins**](https://github.com/minderhq/plugins) catalog. Each plugin is
a top-level `<name>/__init__.py` package that imports from `minder_plugin_sdk`.

## Steps

1. **Scaffold** from [plugin-template](https://github.com/minderhq/plugin-template)
   (“Use this template”) or `minder-plugin scaffold <name>`.
2. **Add it** to the catalog as `<name>/__init__.py`, exporting your class via
   `__all__`.
3. **Validate & test** locally:
   ```bash
   pip install -e ".[dev]"
   minder-plugin validate <name>/__init__.py
   pytest -q
   ```
4. **Open a PR.** CI runs `minder-plugin validate` on every plugin and a suite
   that auto-discovers and contract-checks each one — adding a plugin dir brings
   it under test automatically.

## How it reaches a running Minder

The plugin-registry **discovers the catalog's plugins on startup** from a
read-only copy it only parses, never imports, and lists them at `/v1/plugins`.
It then asks a `plugin-runner` container to load each one: a code plugin in the
`trusted` or `privileged` group, a declarative [manifest](manifest.md) plugin in
the `declarative` group. Plugin code is arbitrary Python inside its runner, which
has no platform secrets and a restricted network. That's why catalog plugins
are reviewed and validated before they merge, and why a plugin gets only a
least-privilege database role of its own (its own `plugin_<name>` schema, no
access to platform tables). The catalog's `runner_group` for a plugin says which
group hosts it, and a plugin that needs LAN egress or manages a sidecar belongs
in `privileged`. See the [plugin trust model](../security.md#plugin-trust-model).

The platform pins this public catalog as a git submodule, the way it pulls in the
web client, so an instance runs the catalog revision that its platform release
pins. Third parties can also install a **manifest** plugin straight from a git
repository, without a rebuild (`POST /v1/plugins/install-from-git`, Platform
Admin only; only the manifest is fetched and validated, and repository code is
never imported or run).

## Governance

Follow the repo conventions (Conventional-Commits PR titles, `component:*` labels):
see the [org contributing guide](https://github.com/minderhq/.github/blob/main/CONTRIBUTING.md).
