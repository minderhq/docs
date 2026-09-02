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

**Open-core + hosted.** The core is open source (Apache-2.0) and yours to
self-host forever; a hosted option is offered separately for those who'd rather
not run it themselves — the core never requires it.

## Start here

- **[Build a plugin](plugins/index.md)** — extend Minder with a data source, an AI
  tool, or a webhook ingestor, using the plugin SDK.
- **[The platform](https://github.com/minderhq/minder)** — services, the
  control-plane UI, and the one-command install.
- **[Plugin SDK](https://github.com/minderhq/plugin-sdk)** — the authoring
  contract, a worked reference plugin, and the `minder-plugin` CLI.

!!! note "Docs in progress"
    This site currently covers the **plugin ecosystem** in full. Self-hosting,
    architecture, and operations guides are being verified against the code
    before they're published here — until then, see the
    [`docs/` in the main repo](https://github.com/minderhq/minder/tree/main/docs).
