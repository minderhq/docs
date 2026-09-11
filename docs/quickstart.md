# Quickstart

The fastest path from a running instance to your **first answer over your own
document**. It should take a few minutes once the platform is up.

!!! note "Before you start"
    You need a **running Minder instance** ([Self-hosting](self-hosting.md)) and a
    **reachable model** — the default local Ollama, or an external one
    ([AI setup](ai-setup.md)). If a query later returns *"Error generating
    response"*, the model backend is unreachable; fix that first.

There are two ways to do this: the **control-plane UI** (recommended the first
time) or the **HTTP API** (for scripting). Both do the same four things — create
a knowledge base, upload a document, build a pipeline over it, and ask a question.

## Path A — the control-plane UI

1. **Open the console and sign in.** Browse to the client (over `localhost` a
   local account works; SSO needs real DNS/TLS — see
   [Authentication](authentication.md)). Most pages are browsable without login,
   but creating things needs you to be signed in.
2. **Create a knowledge base.** Go to **RAG → Knowledge Bases**, create one (give
   it a name and an optional description) — this is the data your questions are
   answered from.
3. **Upload a document.** Open the knowledge base and upload a file — PDF, Word,
   a spreadsheet, an image to OCR, a recording to transcribe, and
   [many more formats](ingestion.md). Upload runs as a background job: it comes back as `processing` and
   the page polls until it's `completed` (with chunk/vector counts) or `failed`.
   You can expand a document to inspect its stored chunks — useful for telling a
   bad extraction apart from a retrieval problem.
4. **Create a pipeline.** Go to **RAG → Pipelines** and create a pipeline over
   your knowledge base. A pipeline is what you actually ask questions against.
5. **Ask a question.** Open **Ask**, pick your pipeline, and ask. Conversations
   persist, so you can keep a thread going.

See [Using Minder](using-minder.md) for the full tour of the console.

## Path B — the HTTP API

All calls go through the API Gateway (`http://localhost:8000` by default). See
the [API reference](api-reference.md) for the full surface and
[Authentication](authentication.md) for obtaining a token.

```bash
# 1. Create a knowledge base (name required, description optional)
KB=$(curl -s -X POST http://localhost:8000/v1/rag/knowledge-bases \
  -H 'Content-Type: application/json' \
  -d '{"name":"My Docs","description":"my documents"}' | jq -r '.id')

# 2. Upload a document into it
curl -X POST "http://localhost:8000/v1/rag/knowledge-bases/$KB/upload" \
  -F "file=@report.pdf"

# 3. Create a pipeline over the knowledge base
PIPE=$(curl -s -X POST http://localhost:8000/v1/rag/pipeline \
  -H 'Content-Type: application/json' \
  -d "{\"name\":\"my-pipe\",\"knowledge_base_ids\":[\"$KB\"]}" | jq -r '.pipeline_id')

# 4. Ask a question
curl -s -X POST "http://localhost:8000/v1/rag/pipeline/$PIPE/query" \
  -H 'Content-Type: application/json' \
  -d '{"question":"What is in my docs?","top_k":3}' | jq '.answer'
```

!!! tip "From the command line"
    The open-source [`minder` CLI](cli.md) wraps the same gateway:
    `minder rag create-kb "My Docs" "my documents"`, `minder rag pipelines`,
    `minder rag query <pipeline_id> "what is X?"`.

## What next

- **Tune retrieval** — the query above used the default (dense) method. Minder
  also has HyDE, Self-RAG, auto, corrective, and RAPTOR methods, plus re-ranking,
  hybrid and parent-child retrieval. See [RAG methods](rag-methods.md).
- **Explore the knowledge graph** — a second retrieval paradigm over the same
  documents (entities and relationships). See
  [Using Minder](using-minder.md#knowledge-graph-raggraph).
- **Extend the platform** — add data sources and AI tools with
  [plugins](plugins/index.md).
- **Turn capabilities on and off** — see [Bundles](bundles.md).
