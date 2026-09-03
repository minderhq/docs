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
them at `/v1/plugins`. By design nothing runs arbitrary code — a plugin is fixed
handlers (or a declarative [manifest](manifest.md)), never uploaded code — so
loading a catalog plugin is safe. First-party plugins ship inside the Minder
core; wiring a running instance to also load this public catalog is a
per-deployment integration step (git-submodule vendoring — the way the web
client is already pulled in — is planned but not yet wired).

## Governance

Follow the repo conventions (Conventional-Commits PR titles, `component:*` labels):
see [issue-and-pr-conventions](https://github.com/minderhq/minder/blob/main/docs/development/issue-and-pr-conventions.md).
