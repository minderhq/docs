# Connect your models: Ollama, vLLM and cloud providers

This page explains how to make Minder use **your** language models. That covers
the bundled Ollama, an Ollama you already run, a vLLM server, any other
OpenAI-compatible server, or a cloud provider such as OpenAI, Anthropic,
OpenRouter or z.ai. Read it right after [installing](self-hosting.md), before
you upload your first documents.

!!! tip "Short version"
    1. Decide where **embeddings** run. They always come from the install's
       Ollama (bundled or yours), so that choice is made with `setup.sh`.
    2. Optionally add **providers** (vLLM, cloud and so on) in the client under
       **Providers & defaults**.
    3. Pick a **default model per task** on the same page.
    4. Run `bash setup.sh verify --deep` to prove the whole path works.

## How Minder chooses a model

There are two layers. Mixing them up is the most common setup mistake.

| Layer | Where it is configured | What it serves |
|---|---|---|
| **Install backend**: one Ollama endpoint for the whole install | `.env` via `setup.sh` (`ollama-mode`, `model-tier`) | **All embeddings**, the reranker's fallback, and the *install default* models |
| **Organization providers**: any number per organization | Client → *Providers & defaults* (or the `/v1/model-providers` API) | Answer, chat, extraction and utility models for that organization |

Generation models are resolved in this order. The first one that is set wins:

1. a model picked for a single question (API only);
2. the pipeline's or knowledge base's own **LLM model**;
3. the organization's **default for the task** (*Providers & defaults → Default models*);
4. the install default from `.env` (`OLLAMA_LLM_MODEL`).

The four tasks:

| Task | Used for | Notes |
|---|---|---|
| **Answer** | Writing knowledge-base answers | The model most users "feel" |
| **Chat** | The assistant chat and its tools | Must support **tool calling**, otherwise chat still works but tools are silently skipped |
| **Extraction** | Building the knowledge graph (entities and relations) | A local model is preferred |
| **Utility** | Query rewrite, HyDE, multi-query, reranking with an LLM, summaries | Defaults to the answer model when that is local |

If a saved default stops existing (a model was deleted, or a provider was
disabled), Minder retries once with the install default and marks the answer
as degraded. An organization admin should then fix the default.

## Which setup should I use?

| Your situation | Use | Section |
|---|---|---|
| Single machine, no GPU tuning wanted | Bundled Ollama | [Bundled Ollama](#option-1-bundled-ollama-default) |
| Ollama already installed on the same machine (common on Windows/macOS with a GPU) | Your own Ollama as the install backend | [Your Ollama on the same machine](#option-2-your-ollama-on-the-same-machine) |
| A separate GPU box on your LAN or private VPN | Remote Ollama as install backend, or as an org provider | [Ollama on another machine](#option-3-ollama-on-another-machine) |
| High-throughput serving, many users | vLLM as an org provider; embeddings stay on Ollama | [vLLM](#option-4-vllm) |
| LM Studio, llama.cpp server, TGI, LocalAI | OpenAI-compatible provider | [Other OpenAI-compatible servers](#option-5-other-openai-compatible-servers) |
| OpenAI, Anthropic, OpenRouter, z.ai GLM | Cloud provider | [Cloud providers](#option-6-cloud-providers) |

You can combine them. A typical setup: Ollama for embeddings and a small
utility model, plus vLLM or a cloud model for answers.

## Option 1: Bundled Ollama (default)

With nothing configured, `OLLAMA_BASE_URL` is empty and Minder runs its own
`minder-ollama` container. It pulls the models listed in `OLLAMA_MODELS` during
install and again on every start.

Pick models that match your hardware with a **model tier**:

```bash
bash setup.sh model-tier             # detect the hardware and suggest a tier
bash setup.sh model-tier apply cpu   # pi | cpu | gpu-8gb | gpu-16gb | gpu-24gb | multi-gpu | datacenter
bash setup.sh restart
```

| Tier | Embedding model | Answer/chat model |
|---|---|---|
| `pi` | `nomic-embed-text` | `llama3.2:1b` |
| `cpu` | `bge-m3` | `llama3.2` |
| `gpu-8gb` | `bge-m3` | `llama3.1:8b` |
| `gpu-16gb` | `qwen3-embedding:8b` | `deepseek-r1:8b-llama-distill-q4_K_M` |
| `gpu-24gb` and up | `qwen3-embedding:8b` | vLLM models (see [vLLM](#option-4-vllm)), with Ollama fallbacks |

A tier writes `OLLAMA_MODELS`, `OLLAMA_LLM_MODEL` and `OLLAMA_EMBEDDING_MODEL`
to `.env`. On an install that already has knowledge bases, the embedding model
is **kept** unless you pass `--confirm-reembed`, because changing it makes old
vectors unusable (see [Embeddings](#embeddings-and-the-reranker)).

The bundled container has no GPU passthrough by default. For an NVIDIA GPU,
uncomment the `runtime: nvidia` block of the `ollama` service in
`docker/docker-compose.yml` and set `CUDA_VISIBLE_DEVICES`. Running your own
Ollama (option 2) is often simpler.

## Option 2: Your Ollama on the same machine

Use this when Ollama is already installed natively, for example the Ollama app
on Windows or macOS with your GPU.

**1. Make Ollama reachable from containers.** Minder's services run inside
Docker. To them, `localhost` is the container itself, never your machine.

- Ollama must listen on all interfaces. Start it with `OLLAMA_HOST=0.0.0.0`.
  - Windows: set it as a user environment variable, then quit and restart the Ollama app.
  - Linux: add `Environment="OLLAMA_HOST=0.0.0.0"` to the systemd unit.
- Allow port `11434` from the Docker network in your firewall. Do not expose it
  to the internet.

**2. Bound Ollama's default context length.** This is critical on desktops.
The Ollama app has a *Context length* setting. It becomes the default for
every client that doesn't send its own value. Minder always sends one
(`OLLAMA_MAX_NUM_CTX`, 16384 by default), but other tools on the same Ollama
may not, including the optional *Minder AI Chat* (Open WebUI). At a 256K
default, a 14B model needs about 50 GB for its context alone. That spills into
system RAM and can **freeze the whole machine**. Set the slider to **16K**, or
start the server with `OLLAMA_CONTEXT_LENGTH=16384`.

**3. Point Minder at it:**

=== "Windows / macOS (Docker Desktop)"

    ```bash
    bash setup.sh ollama-mode external http://host.docker.internal:11434
    bash setup.sh restart
    ```

=== "Linux (Docker Engine)"

    `host.docker.internal` does **not** resolve on Linux unless you add it.
    Use the host's LAN IP instead:

    ```bash
    ip -4 addr show | grep inet          # find the LAN address, e.g. 192.168.1.20
    bash setup.sh ollama-mode external http://192.168.1.20:11434
    bash setup.sh restart
    ```

**4. Pull the models on the host yourself.** In external mode Minder does
**not** pull models:

```bash
ollama pull nomic-embed-text      # or the embedding model in OLLAMA_EMBEDDING_MODEL
ollama pull llama3.2              # or the model in OLLAMA_LLM_MODEL
```

**5. Check it from inside a container:**

```bash
# expect 200; the single quotes make the container expand its own OLLAMA_BASE_URL
docker exec minder-rag-pipeline sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 "$OLLAMA_BASE_URL/api/tags"'
```

!!! warning "Unreachable Ollama fails quietly on upload"
    If the install backend is unreachable, document uploads can still return
    success while storing unusable vectors. Always run step 5, then
    `bash setup.sh verify --deep`.

## Option 3: Ollama on another machine

There are two ways to use a separate GPU box, and they can be combined:

- **As the install backend.** It then serves embeddings for everyone. Do
  option 2 with that machine's address:
  `bash setup.sh ollama-mode external http://192.168.1.50:11434`. For
  resilience, `bash setup.sh ollama-mode failover http://192.168.1.50:11434`
  falls back to the bundled container while the box is down (see
  [AI setup → Failover](ai-setup.md#failover-mode)).
- **As an organization provider.** It then serves answer and chat models for
  one organization only. In the client: *Providers & defaults → Add provider →
  Ollama (self-hosted)*. Enter `http://192.168.1.50:11434`. Leave the API key
  empty unless your Ollama sits behind an authenticating proxy.

Private-network and VPN addresses (for example `100.x.y.z` carrier-grade NAT
addresses) are accepted for self-hosted providers, as long as the Docker host
itself can reach them. Loopback and cloud-metadata addresses are always refused.
Self-hosted providers are refused entirely on a multi-tenant deployment
(`SAAS_MODE=true`), where every provider URL must be a public endpoint.

## Option 4: vLLM

vLLM is the right choice for serving larger models to many users. Minder talks
to it through its **OpenAI-compatible** API.

**1. Start vLLM** so that it listens on the network, with a bounded context
and tool calling enabled. Tool calling is needed for the *Chat* task:

```bash
docker run --gpus all -p 8000:8000 --ipc=host vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-7B-Instruct \
  --max-model-len 16384 \
  --enable-auto-tool-choice --tool-call-parser hermes
```

- `--tool-call-parser` must match the model family: `hermes` for Qwen2.5,
  `llama3_json` for Llama 3.x, `mistral` for Mistral. Without these two flags
  vLLM can't serve tool calls. Minder's chat then falls back to answering
  without tools.
- Minder doesn't send a context size to OpenAI-compatible servers, so
  `--max-model-len` is the only limit. Keep it at 8192 or more. RAG prompts
  carry up to about 6000 tokens of retrieved context plus the question.
- If you start vLLM with `--api-key`, you will enter that key in Minder.

**2. Add it as a provider.** *Providers & defaults → Add provider → vLLM
(self-hosted)*:

| Field | Value |
|---|---|
| Base URL | `http://<host>:8000/v1`. Include `/v1`. Use the LAN IP or `host.docker.internal`, never `localhost` or `127.0.0.1` (always refused). |
| API key | Your `--api-key`. Leave it empty if you didn't set one: the provider is saved without a key and Minder sends no `Authorization` header |
| Models | Click **Fetch models**, tick the ones you want, then **save**. A model only appears in pickers once it is saved on the provider. |

Then click **Test**. It sends one short prompt; it doesn't check tool support.

**3. Use it.** Under *Default models*, choose the vLLM model (listed with the
provider's name next to it) for *Answer* and/or *Chat*. Embeddings still come
from the install's Ollama.

The vLLM preset is marked **local**. It may use private addresses, its
traffic is never PII-masked, and it may answer from `local_only` knowledge
bases.

!!! note "Status"
    The vLLM path uses the same OpenAI-compatible adapter as the cloud
    providers. It is not yet covered by Minder's automated end-to-end tests.
    After configuring, confirm with `bash setup.sh verify --deep`.

## Option 5: Other OpenAI-compatible servers

LM Studio, llama.cpp's `llama-server`, Hugging Face TGI and LocalAI all expose
an OpenAI-compatible `/v1` API, so they work like vLLM:

- Enable network serving in the tool (for example LM Studio's *Serve on local
  network*). Note the base URL, which ends in `/v1`.
- Add the server with its self-hosted preset: **LM Studio (self-hosted)**,
  **llama.cpp server (self-hosted)** or **TGI (self-hosted)**. For any other
  server on your own hardware (LocalAI, for example), use **vLLM
  (self-hosted)**. These presets are marked *local*, with the same rules as
  vLLM. The generic *OpenAI-compatible (custom)* preset is treated as a cloud
  provider, which refuses private addresses and applies PII masking.
- Click **Fetch models**. If the server has no model list (HTTP 404), type the
  model id exactly as the server expects it.
- Don't use port `11434` for a non-Ollama server. Minder treats that port as
  Ollama.

## Option 6: Cloud providers

*Providers & defaults → Add provider*, then choose a preset:

| Preset | Base URL (pre-filled) | Notes |
|---|---|---|
| OpenAI | `https://api.openai.com/v1` | Model list is built in |
| Anthropic | `https://api.anthropic.com` | Model list is built in; tools are translated automatically |
| OpenRouter | `https://openrouter.ai/api/v1` | Use **Fetch models** |
| z.ai (GLM) | `https://api.z.ai/api/paas/v4` | Pay-as-you-go keys |
| z.ai Coding Plan | `https://api.z.ai/api/coding/paas/v4` | Coding-plan keys. Each key type works only on its own endpoint. An auth error tells you to switch. |
| OpenAI-compatible (custom) | — | Any other hosted OpenAI-compatible API |

Before a cloud provider can be enabled, you must **acknowledge** that prompts
leave your installation. Here is what is sent:

- **Answers:** the question plus the retrieved document passages.
- **Chat:** the conversation history, tool definitions and tool results.
- **Utility and extraction:** only if you set a cloud model for those tasks.

**PII masking** is on by default for cloud providers. Before a request leaves
the install, emails, phone numbers, IBAN and card numbers, Turkish ID numbers
(TCKN), IP addresses and credentials in URLs are replaced with placeholders
such as `[EMAIL_1]`. The real values are restored in the answer. The client
has no switch for masking yet; it can only be turned off per provider through
the API (`pii_masking_enabled: false`), and doing so is logged.

API keys are stored encrypted (`MODEL_PROVIDER_VAULT_SECRET` in `.env`) and are
only ever shown masked. Keep a backup of that secret: if it is lost, stored
keys can't be decrypted and must be re-entered.

## Keep sensitive knowledge bases on local models

Set a knowledge base's **Data policy** to **Local only**. Its content can then
only be answered by local models: the install's Ollama, or a provider marked
local (Ollama or vLLM self-hosted presets). Asking it through a cloud model
returns:

```
this knowledge base is local_only: its content may not be sent to a cloud
model. Pick a local model (or a self-hosted provider marked local)
```

To answer with the local install default instead of refusing, set
`LOCAL_ONLY_FALLBACK=local` in `.env`, then `bash setup.sh restart`. Such
answers are marked degraded.

## Embeddings and the reranker

- **Embeddings always come from the install's Ollama** (`OLLAMA_BASE_URL`).
  Providers (vLLM, cloud, a per-organization Ollama) don't serve embeddings.
- Each knowledge base's embedding model is fixed when it is created. The
  default comes from `OLLAMA_EMBEDDING_MODEL`. **It can't be changed later.**
  To switch, create a new knowledge base and upload the documents again. Pick
  the embedding model before you import a large corpus.
- The **reranker** (`bge-reranker-base`, about 280 MB) runs on the CPU inside
  `rag-pipeline`. It is downloaded from Hugging Face on first start. Until it
  is ready, reranking falls back to the LLM. For air-gapped installs, set
  `RERANKER_ALLOW_DOWNLOAD=false` and pre-seed the `rag_reranker_models`
  volume.

## Sizing: will the model fit?

A local model is fast only while it fits **entirely in GPU memory**. Its
footprint is roughly the model file size plus the context cache, which grows
with `OLLAMA_MAX_NUM_CTX` (default 16384). An example measured on a 16 GB GPU
at a 16K context:

| Model | Footprint | Result |
|---|---|---|
| `llama3.2` (3B) | about 3.8 GB | Fits easily |
| `deepseek-r1:14b` | about 11.1 GB | Fits |
| 26–32B models (`qwen3:30b`, `gemma4:26b`, `*:32b`) | 18–20 GB+ | **Does not fit.** Layers spill into system RAM, which is slow and can freeze a busy machine. |

Check what a loaded model really uses with
`curl http://<ollama>:11434/api/ps`: `size_vram` equal to `size` means fully on
the GPU. The model pickers don't warn about this yet, so choose defaults with
this table in mind.

On Windows with Docker Desktop, leave memory for the models. The WSL VM that
runs Docker is capped in `%UserProfile%\.wslconfig` (`memory=`). Setting
`autoMemoryReclaim=dropCache` makes it return idle cache to Windows.

## Who can change these settings

- **Default models:** an organization's owners and admins.
- **Providers:** the client's *Providers & defaults* page is shown to members who
  hold the `org.ai.manage` permission. The API accepts the instance `admin` role
  or an organization `owner`/`admin`.
- **Install backend** (`ollama-mode`, `model-tier`): whoever runs `setup.sh`
  on the host.

`.env` changes take effect after `bash setup.sh restart`. Provider and default
changes in the client take effect immediately; defaults are cached for up to
30 seconds.

## Verify and troubleshoot

Run the end-to-end check after every change:

```bash
bash setup.sh verify --deep
```

It runs as a dedicated synthetic test organization, uploads sample documents,
and asks questions with your **install defaults**. It checks citations and the
local-only guard, then deletes the knowledge bases it made. Failing checks
print a fix hint. It doesn't use your organization's defaults or providers. To
check those, ask a question on the *Ask* page.

| You see | Likely cause | Fix |
|---|---|---|
| `Error generating response …` / "LLM backend unavailable" | The install Ollama is unreachable | Do steps 1 and 5 of [option 2](#option-2-your-ollama-on-the-same-machine). Check `OLLAMA_BASE_URL`. |
| "Chat completion failed: a required backend is unreachable" | Chat model backend down | Same as above, or the provider is down |
| "That model can't be used: … is not pulled" when saving defaults | Model missing on the Ollama | `ollama pull <model>` on that Ollama host |
| "The local model server is unreachable" when saving defaults | Ollama down or wrong URL | As above |
| "provider base_url is not permitted …" | `localhost`/`127.0.0.1`, or a private address on a non-local preset | Use the LAN IP or `host.docker.internal`, and the *vLLM (self-hosted)* or *Ollama (self-hosted)* preset |
| "authentication failed — check the API key" | Wrong key, or a z.ai key on the other z.ai endpoint | Fix the key or switch the preset |
| "this endpoint does not expose a model list (HTTP 404)" | Server has no `/models` | Type the model ids manually |
| "(unreadable -- vault secret mismatch)" | `MODEL_PROVIDER_VAULT_SECRET` changed | Restore the old secret, or re-enter the keys |
| Chat answers but never uses tools | Chat model or server lacks tool calling | Pick a tool-capable model; for vLLM add `--enable-auto-tool-choice --tool-call-parser …` |
| The machine freezes on the first question | Model spills into RAM, often because of a huge default context in the Ollama app | Set Ollama's context length to 16K and use a model that fits (see [Sizing](#sizing-will-the-model-fit)) |
| Uploads succeed but answers find nothing | Embeddings failed (Ollama unreachable or embedding model missing) | Fix Ollama, then delete and re-upload the documents |
