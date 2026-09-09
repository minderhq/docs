# Knowledge graph

Minder offers two retrieval paradigms over the same documents. The
[vector RAG pipeline](rag-methods.md) embeds text chunks and matches them by
similarity; the **knowledge graph** takes a different approach — spaCy NER
extracts entities and their relationships from your text, stores them in
**Neo4j**, and lets you query over the graph instead of over vectors. It is
served by a separate service, `graph-rag`, and is a distinct surface from the
vector pipeline. See [Graph RAG](rag-methods.md#graph-rag) for how the two
paradigms differ.

The graph is also unrelated to the plugin-dependency graph shown on the
Marketplace pages — that lives in the same Neo4j instance but holds unrelated
data.

## Building the graph

`POST /v1/construct-graph` builds a Neo4j knowledge graph from your
documents and their extracted entities. The build is **atomic** — it runs in a
single transaction, so a mid-build failure can never leave a half-built graph.

Re-posting the same `document_id` is a **full replace** of that document's
edges, not an upsert: the old `RELATES_TO` / `MENTIONS` relationships are
dropped before the rebuild and any orphaned entities are cleaned up. Omitting a
relationship on a re-POST therefore removes it. Document metadata
(`title` / `source` / `metadata`) is the exception — it upserts via `COALESCE`,
so an omitted field keeps its previous value rather than being blanked.

Entity extraction on its own is available through `POST /v1/extract` (spaCy NER
over a block of text).

## Exploring the graph

| Endpoint | What it does |
|----------|--------------|
| `POST /v1/graph/search` | Free-text search for entities whose `text` or `label` matches `query` (case-insensitive `CONTAINS`); returns `{text, label, description}` per hit, `limit` 1–50 (default 5). |
| `GET /v1/graph/stats` | Graph overview — total node / relationship / document / entity counts plus the per-NER-label entity distribution (`entity_types`). Confirms a `construct-graph` actually populated the graph. |
| `GET /v1/graph/documents` | Lists the Document nodes (id / title / source / created_at / entity_count) in your graph, newest first. |
| `POST /v1/retrieve` | Graph-based retrieval over entity relationships. |
| `POST /v1/entity-context` | Retrieve the context and neighbours around an entity. |
| `DELETE /v1/graph/document/{document_id}` | Delete a document's graph — its relationships, Document node, and orphaned entities. Entities still referenced by another of your documents are kept. |

Deleting a document is idempotent: if the document is absent for you, it returns
`200` with zero counts. Everything above is scoped to your own graph (see
[Owner-scoping](#owner-scoping)).

## The correlation engine

`POST /v1/graph/correlate` runs the **correlation engine** over your graph. It
derives edges the raw NER graph misses:

| Edge type | Meaning |
|-----------|---------|
| `CO_OCCURS` | Corpus co-mention — two entities appearing together. |
| `SIMILAR_TO` | Embedding neighbour. |
| `SAME_AS` | Entity resolution — the same real-world entity. |
| `CORRELATES_WITH` | Temporal signal↔signal correlation over InfluxDB series, carrying lead-lag `lag` / `direction` and Granger `granger_p_value` / `granger_causal_direction`. |
| `HAS_SIGNAL` | Links an entity to a quantitative signal. |
| `IS_A` | Taxonomy classification (see [Taxonomy review queue](#taxonomy-review-queue)). |

The request body `{correlators?}` selects which correlators to run (default:
all), and the response returns per-correlator edge counts. The endpoint is
**rate-limited to 5 requests per minute** per caller.

For the `IS_A` taxonomy edge, deterministic tiers 1 and 2 always run. An
optional tier 3 proposes an LLM-classified `IS_A_CANDIDATE` that is held for
human review and never auto-published.

### Reading an entity's correlations

`GET /v1/graph/correlations?entity=<text>&limit=&max_hops=&include_centrality=`
returns
`{found, co_occurring, similar, same_as, correlated_signals, indirect, is_a, entity_centrality_score}`
for one entity:

- **`correlated_signals`** surfaces the quantitative signals the entity's own
  signal moves with, following `Entity→HAS_SIGNAL→Signal→CORRELATES_WITH→Signal`.
- **`max_hops`** (2–3) additionally returns `indirect`: entities reachable via a
  bounded multi-hop path with no direct edge, each tagged with `hops` / `via`.
- **`is_a`** gives the entity's taxonomy category.
- **`include_centrality=true`** attaches a 0–1 normalized degree-centrality
  `centrality_score` to every result, plus a top-level
  `entity_centrality_score`.

## Owner-scoping

Every graph read, the delete, and the correlation engine are scoped to the
caller's own graph, derived from the `owner_id` in the JWT:

- `GET /v1/graph/stats`, `GET /v1/graph/documents`, and
  `DELETE /v1/graph/document/{id}` see only your graph. A `document_id` owned by
  a different caller is treated as **absent**, not as a permission error.
- The correlation engine correlates over **shared ∪ self** — your own data plus
  data explicitly shared with you — and **never across tenants**.

Reads require a valid JWT; writes require JWT.

## Taxonomy review queue

Tier-3 taxonomy candidates are LLM self-consistency guesses that are never
auto-published — they wait in a review queue for a human decision:

| Endpoint | What it does |
|----------|--------------|
| `GET /v1/graph/taxonomy/review-queue` | List pending tier-3 candidates awaiting review. Owner-scoped: a regular caller sees only their own entities' candidates; admin/service sees every tenant's queue. |
| `POST /v1/graph/taxonomy/review-queue/{candidate_id}/approve` | Approve a candidate — promotes it to a live `IS_A` edge (`method='llm_tier3'`), now surfaced by `/v1/graph/correlations`' `is_a`. |
| `POST /v1/graph/taxonomy/review-queue/{candidate_id}/reject` | Reject a candidate — no `IS_A` edge is written, and it is never re-attempted (the entity keeps its tier-2 coarse classification). |

In the UI this is the [Taxonomy Review](using-minder.md#taxonomy-review-ragtaxonomy-review)
page (`/rag/taxonomy-review`) — a curation queue for the graph's entity-type
taxonomy, so the graph keeps a coherent schema instead of every extraction
inventing its own types. Admin work.

## Entity merge review

The graph can end up with one real-world entity showing up as more than one node
— often across organizations. The
[Entity Merge Review](using-minder.md#entity-merge-review-ragentity-merges) page
(`/rag/entity-merges`) scans for these and proposes merges. Cross-org merges are
**dual-control**: each side's owner must approve before the merge applies, so
one tenant can't unilaterally fold another's data together. Owner/admin work.

## Document graph-visibility

In the UI, each document's graph-visibility can be set to **private**,
**shared**, or **team**, which governs how widely a document's entities
participate in the shared graph.

## In the UI / via the API

- **In the UI** — the [Knowledge Graph](using-minder.md#knowledge-graph-raggraph)
  page (`/rag/graph`) lets you build the graph, explore it, review entity
  correlations, manage a document's graph-visibility, and delete a document from
  the graph. Entity-merge and taxonomy review have their own pages (above).
- **Via the API** — the endpoints above are served by `graph-rag` on
  `http://localhost:8008` and reachable through the gateway at
  `/v1/graph-rag/<path>`. See the [API reference](api-reference.md) for the full
  route list.
