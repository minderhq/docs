# Minder

**Your private AI platform — local inference, your own data, and an extensible
tool ecosystem — that installs on modest hardware with a single command.**

Minder is **local-first and self-hostable**. It runs on hardware you own (down to
a Raspberry Pi) and gives you:

- **RAG + a knowledge graph** over your own documents and data,
- **local LLMs** via Ollama — no cloud keys, nothing leaves the box by default,
- an **extensible plugin & tool ecosystem** with *no arbitrary code execution* —
  plugins are declarative, fixed, reviewed handlers,
- **one modern control-plane UI** for everything that isn't chat,
- and **honest, verifiable** capabilities.

**Proprietary core, open ecosystem.** The Minder platform (core) is a commercial
product — self-hostable with a license, with a freemium trial planned. Its
**ecosystem is open source** and the focus of these docs: the plugin SDK, the
plugin catalog, and the web client, all freely available.

## Start here

- **[Build a plugin](plugins/index.md)** — extend Minder with a data source, an AI
  tool, or a webhook ingestor, using the plugin SDK.
- **[Self-hosting the platform](self-hosting.md)** — running the core (licensed;
  the repository and images are private).
- **[Using Minder](using-minder.md)** — the control-plane UI: knowledge bases,
  RAG pipelines, the graph, plugins, models, and admin.
- **[Command-line client](cli.md)** — the open-source `minder` CLI for scripting
  against a running instance.
- **[Plugin SDK](https://github.com/minderhq/plugin-sdk)** — the authoring
  contract, a worked reference plugin, and the `minder-plugin` CLI.

!!! note "Docs in progress"
    This site covers the **plugin ecosystem**, self-hosting, the control-plane
    UI, and the CLI. Architecture and deeper operations guides are being
    verified against the code before they're published here — until then, see
    the [`docs/` in the main repo](https://github.com/minderhq/minder/tree/main/docs).
