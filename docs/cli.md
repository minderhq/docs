# Command-line client

`minder` is a small, **open-source** CLI over a running Minder instance's
api-gateway — auth, health, plugins, RAG, models, AI chat, organizations,
billing, and knowledge-graph correlations. It's a companion to the platform, not
part of the licensed core: source lives at
[minderhq/cli](https://github.com/minderhq/cli).

## Install

```bash
pip install "git+https://github.com/minderhq/cli"
```

## Log in

```bash
minder login --api-url http://localhost:8000   # prompts, caches a JWT
```

Config resolves as **flag → env (`MINDER_API_URL` / `MINDER_TOKEN`) →
`~/.config/minder/config.json` → default** (`http://localhost:8000`). Once
you've logged in, later commands need no flags.

## Use

```bash
minder health                                    # api-gateway /health
minder status                                    # every service's health

minder plugins list                              # list registered plugins
minder plugins config crypto                     # show a plugin's config
minder plugins config crypto --set K=V           # update config (JWT)

minder rag kbs                                   # list knowledge bases
minder rag create-kb "My Docs" "my documents"    # create one
minder rag pipelines                             # list pipelines (--limit N)
minder rag query <pipeline_id> "what is X?"      # ask a pipeline (--top-k N)

minder models list                               # list Ollama models
minder models pull llama3.2:latest               # pull one (admin)

minder ai tools                                  # the LLM's callable tools
minder ai chat "summarise RAG" --tools           # one-shot chat (JWT)

minder org list                                  # orgs you belong to (multi-tenant)
minder org switch <organization_id>              # switch active org (re-mints + caches your JWT)

minder billing subscription                      # the org's current plan (SaaS)
minder billing checkout pro                      # start a hosted checkout for a tier
minder billing portal                            # customer-portal URL

minder graph correlations <entity>               # an entity's correlated entities/signals (--limit N)
```

Global flags (`--api-url`, `--token`, `--json`) go **after** the subcommand
(git/docker style), e.g. `minder health --json`.

## Output

Output is a compact **human view** by default — a bulleted list for
collections, `key: value` for objects, plain text for a chat reply. Pass
**`--json`** on any command for raw JSON to pipe into `jq`.

!!! note
    The CLI talks to whatever instance you point it at with `--api-url` — it
    doesn't run the platform itself. See [Self-hosting](self-hosting.md) to
    stand up an instance first. The same operations are also available from
    the browser-based control-plane UI, which isn't documented on this site
    yet ([minderhq/docs#6](https://github.com/minderhq/docs/issues/6)).
