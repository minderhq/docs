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

`minder login` signs in with a **local** username and password
(`POST /v1/auth/login`; `-u`/`-p` skip the prompts). An account that exists only
through SSO has no local password, so `minder login` can't sign it in.

Config resolves as **flag → env (`MINDER_API_URL` / `MINDER_TOKEN`) →
`~/.config/minder/config.json` → default** (`http://localhost:8000`). Once
you've logged in, later commands need no flags.

### Token refresh

Access tokens are short-lived (15 minutes by default, the gateway's
`JWT_EXPIRATION_MINUTES`). The CLI renews them transparently through
`POST /v1/auth/refresh`: shortly before the token expires, or once after a
`401`. A refreshed token is written back to the same config file; a `--token` or
`MINDER_TOKEN` override is refreshed for that run only.

The gateway accepts a recently expired token for refresh within
`JWT_REFRESH_GRACE_MINUTES` (12 hours by default), up to an absolute session cap
counted from sign-in, `JWT_SESSION_MAX_HOURS` (24 hours by default). After the
cap, or when the session was revoked (password change or reset, role demotion,
deactivation), the CLI prints ``please run `minder login` again``. See
[Authentication](authentication.md#token-lifetime-and-sessions).

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
    stand up an instance first. The same operations are also available from the
    browser-based control-plane UI — see [Using Minder](using-minder.md).
