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

<div class="grid cards" markdown>

-   :material-puzzle-outline:{ .lg .middle } &nbsp;__Build a plugin__

    ---

    Extend Minder with a data source, an AI tool, or a webhook ingestor using the
    plugin SDK — declarative, no uploaded code.

    [:octicons-arrow-right-24: Plugin guide](plugins/index.md)

-   :material-server:{ .lg .middle } &nbsp;__Self-host the platform__

    ---

    Run the core on hardware you own (licensed; the repository and images are
    private).

    [:octicons-arrow-right-24: Self-hosting](self-hosting.md)

-   :material-view-dashboard-outline:{ .lg .middle } &nbsp;__Use Minder__

    ---

    The control-plane UI: knowledge bases, RAG pipelines, the graph, plugins,
    models, and admin.

    [:octicons-arrow-right-24: Using Minder](using-minder.md)

-   :material-console:{ .lg .middle } &nbsp;__Command-line client__

    ---

    The open-source `minder` CLI for scripting against a running instance.

    [:octicons-arrow-right-24: CLI](cli.md)

</div>

!!! note "Docs in progress"
    This site covers the **plugin ecosystem**, self-hosting, the control-plane
    UI, and the CLI. Architecture and deeper operations guides are being verified
    against the code before they're published here.
