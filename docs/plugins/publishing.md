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

The plugin-registry **discovers and loads module plugins on startup** and lists
them at `/v1/plugins`. A module plugin is Python that runs **in-process** in the
registry, so it is arbitrary code and isn't sandboxed. That's why catalog plugins
are reviewed and validated before they merge, and why plugins get a
least-privilege database role (their own `plugin_data` schema, no access to
platform tables). A declarative [manifest](manifest.md) plugin runs no plugin
code. See the [plugin trust model](../security.md#plugin-trust-model).
First-party plugins ship inside the Minder core; wiring a running instance to
also load this public catalog is a per-deployment integration step (git-submodule vendoring — the way the web
client is already pulled in — is planned but not yet wired).

## Governance

Follow the repo conventions (Conventional-Commits PR titles, `component:*` labels):
see the [org contributing guide](https://github.com/minderhq/.github/blob/main/CONTRIBUTING.md).
