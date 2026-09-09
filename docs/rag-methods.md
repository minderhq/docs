# RAG methods

Minder splits retrieval-augmented generation across two services:

- **RAG pipeline** (`rag-pipeline`) — text RAG over your documents. One query
  endpoint, `POST /pipeline/{id}/query`, supports several retrieval and
  generation methods, all selected with a single `method` field.
- **Graph RAG** (`graph-rag`) — a different paradigm: entities and
  relationships extracted from your text (spaCy NER) and stored in Neo4j, then
  queried over the graph. See [Using Minder](using-minder.md#knowledge-graph-raggraph).

Every method and enhancer below is reachable through the live query endpoint.
Which ones actually run depends on the host's installed libraries and models —
`GET /capabilities` reports exactly what is active on a given deployment, so
capabilities **self-degrade by hardware** rather than failing.

## Requesting a method

All methods and enhancers are requested on `POST /pipeline/{id}/query`:

```json
{
  "question": "...",
  "top_k": 5,
  "method": "standard | hyde | self_rag | auto | corrective | raptor | multi_query",
  "conversation_id": "...",
  "rerank": false,
  "compress": false,
  "hybrid": false,
  "parent_context": false,
  "metadata_filter": { "source": "...", "document_id": "..." },
  "llm_model": "..."
}
```

RAPTOR additionally needs `build_tree: true` set on the earlier
`POST .../upload` call(s) that populated the pipeline's knowledge base(s) — see
[RAPTOR](#raptor) below.

### Choosing the generation model

`llm_model` picks the LLM used to generate the answer — not the embedding model,
which is fixed at ingest time and can never be overridden per query (it must
match whatever was used to embed the documents). It resolves most-specific
first: an explicit per-query `llm_model` → the pipeline's configured `llm_model`
→ the first referenced knowledge base's `llm_model` → the platform default.
Omit the field to use whatever the pipeline or knowledge base already has.

### Honest degradation

Minder never silently substitutes a method you didn't ask for:

- **An unknown `method` is rejected with `422`**, listing the valid values —
  `standard`, `hyde`, `self_rag`, `auto`, `corrective`, `raptor`, `multi_query`
  (case-insensitive). Conversational RAG is enabled with `conversation_id`, not
  a `method` value.
- **The response reports the *effective* method.** When a requested capability
  couldn't run on this host, `method_details.degraded` explains why (e.g.
  *"hybrid requested but BM25 unavailable — used dense retrieval"*,
  *"self_rag: quality evaluator unavailable — single pass, no refinement"*), and
  a top-level `degraded` boolean mirrors whether that list is non-empty.
  `method_details.retrieval` reports the retrieval strategy actually used
  (`dense`, `hybrid`, `parent_context`, or `raptor`).
- **Quality metrics are honest.** Self-RAG's `evaluated` is `false` when no
  evaluator ran, and `threshold_met` is `null` rather than a bogus `true` for a
  threshold that was never measured.

## Methods

### Standard RAG (dense retrieval)

The default. Chunks are embedded and stored as dense vectors in Qdrant; a query
is embedded and matched by cosine similarity (top-k), and the retrieved context
is passed to the LLM for generation.

### Conversational RAG (multi-turn)

Set `conversation_id` on the query. Minder fetches the recent turns of that
conversation and **condenses the follow-up into a standalone query** using the
history (one LLM call) before retrieval — so a follow-up that relies on
anaphora ("that company", "it") retrieves on the resolved entity rather than the
bare pronoun. Generation still sees your original question. If the rewrite can't
run (LLM error, no history yet), retrieval falls back to the raw question and
`method_details.degraded` says so.

Conversation history is per-user by default, scoped to your own
`conversation_id`s, with an explicit opt-in to share a thread. Long
conversations can pressure the model's context window.

### HyDE

Retrieve using an LLM-generated *hypothetical* answer to the question instead of
the raw question — useful when the answer's vocabulary differs from the way the
question is phrased. `method: "hyde"`.

### Self-RAG

A self-critique loop that grades its own generation and can refine it.
`method: "self_rag"`.

### auto (decision engine)

`method: "auto"` lets a decision engine pick per query — choosing `top_k`,
whether to re-rank, and HyDE/Self-RAG toggles. `method_details.decision` reports
the full decision and an `applied` summary. The engine's suggested retrieval
strategy is advisory: the actual retriever is still selected by the `hybrid` /
`parent_context` flags, and any mismatch is reported under
`method_details.degraded` rather than silently applied.

### Corrective RAG (CRAG)

`method: "corrective"` grades the retrieval and re-retrieves with a refined
query when the initial results are weak. An optional web-search fallback exists
but only runs when a search API key is configured.

### Multi-Query

`method: "multi_query"` has the LLM generate up to three alternate phrasings of
your query, retrieves each separately, and merges the results (deduped by
source and text, highest score wins) with the original retrieval. It targets a
query/answer vocabulary mismatch that `hybrid` and `rerank` alone don't fix —
neither of those changes *what* is searched for, only how already-found
candidates are scored.

### RAPTOR

`method: "raptor"` — hierarchical retrieval over a summary tree built at ingest
time. Opt in per upload with `build_tree: true` on `POST .../upload` (off by
default, since building the tree adds real LLM-summarization cost). Minder
clusters the document's chunk embeddings, summarizes each cluster with the
knowledge base's own model, embeds the summaries, and repeats for up to three
levels. Summary nodes live in the same collection as the leaf chunks and carry
the same `document_id`, so deleting the document cleans up the whole tree.

Retrieval is a "collapsed tree": a plain top-k search across every level, with
no traversal — a broad question naturally lands nearer an abstract summary, a
specific one nearer a leaf. Every other method excludes summary nodes by
default, so a knowledge base with a tree never leaks summary text into unrelated
queries. On a knowledge base with no tree, `method: "raptor"` is a harmless
no-op identical to standard dense retrieval.

### Graph RAG

Entity extraction and knowledge-graph retrieval, served by the `graph-rag`
service (spaCy NER + Neo4j) — a separate surface from the vector pipeline above.
See [Using Minder](using-minder.md#knowledge-graph-raggraph) for the control-plane UI.

## Enhancers

These are orthogonal and apply to any method:

| Enhancer | Flag | What it does |
|----------|------|--------------|
| **Re-ranking** | `rerank: true` | Re-orders retrieved candidates with a cross-encoder when `sentence-transformers` is installed, otherwise an LLM re-rank. `GET /capabilities` reports which backend is active. |
| **Contextual compression** | `compress: true` | Extracts only the query-relevant sentences from retrieved chunks before generation. |

## Retrieval strategies

| Strategy | Flag | What it does |
|----------|------|--------------|
| **Hybrid (dense + BM25)** | `hybrid: true` | Combines dense vector search with a lexical BM25 index (requires `rank-bm25`). The BM25 index is built lazily from the stored chunks and invalidated on upload. |
| **Parent-child (small-to-big)** | `parent_context: true` | Matches precise child chunks, then returns each with its neighbouring window (adjacent chunks) for fuller context. |

When more than one retrieval flag is set, precedence is
`parent_context` > `hybrid` > `raptor` > dense; the ignored flags are recorded
in `method_details.degraded`.

## Metadata filtering

`metadata_filter: {"source": "...", "document_id": "..."}` restricts retrieval
to matching chunks, orthogonal to the method and retrieval strategy:

- **`source`** — exact filename match.
- **`document_id`** — scope to a single upload.

Both are stamped on every chunk at ingest; when both are set they are ANDed. The
response's `method_details.metadata_filter` echoes back exactly what was applied.
No user-defined tags exist yet — filtering by anything other than these two
fields would need an ingest-time change first.

!!! note "Hybrid's lexical side"
    Under `hybrid`, the BM25 corpus is unfiltered by design and the metadata
    filter is applied as a post-filter on the merged results. A very narrow
    filter can therefore return fewer than `top_k` hits under hybrid
    specifically.

## Not supported

Some techniques are out of scope for the current single-node, local-first
architecture: **agentic RAG** (needs a full tool-calling agent framework),
**streaming RAG** (the flow is batch, not SSE/WebSocket), **federated RAG**
(multi-node federation), and **long-context RAG** (bounded by the local model's
context window).

---

`GET /capabilities` on the RAG pipeline is the source of truth for what is
active on any given host — always check it rather than assuming a method or
enhancer is available.
