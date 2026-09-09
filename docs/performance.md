# Performance and tuning

Practical guidance for understanding and tuning the performance of a self-hosted
Minder deployment.

!!! note "No canned benchmark numbers"
    Real-world performance depends heavily on the hardware you run on, the LLM
    model you choose, and how many services are active at once. This page gives
    you the levers and the tools to measure your **own** deployment rather than
    generic figures — always benchmark on your target hardware.

## What dominates performance

On constrained ARM hardware (for example a Raspberry Pi 4: a few cores, limited
RAM, no discrete GPU), the biggest factors are, in rough order:

1. **LLM inference (Ollama).** Model size and quantization dominate everything. A
   7B model on a small box is slow and memory-hungry; small quantized models are
   far more usable. This is almost always the bottleneck for RAG and chat.
2. **Memory pressure.** A full container stack plus an LLM can exceed available
   RAM, causing swap and severe slowdowns. Trim what you don't need.
3. **Vector search (Qdrant)** and **graph queries (Neo4j)** for RAG / graph-RAG.
4. **Startup time.** Cold starts — especially services that download models or
   language data — take a while; this is one-time per container start.

## Tuning levers

### 1. Choose a smaller / quantized Ollama model

Model choice is the single highest-impact lever. Prefer small, quantized models
on constrained hardware. See [AI setup](ai-setup.md) for the model catalogue and
per-model RAM guidance.

- Models to auto-pull are configured via `OLLAMA_MODELS` in the root `.env`.
- List what is loaded (Ollama is internal-only, so query it from the container):

    ```bash
    docker exec minder-ollama ollama list
    ```

### 2. Local vs. external Ollama

Ollama runs in one of two modes, controlled by `OLLAMA_BASE_URL`:

- **empty** — the profile-gated internal Ollama container runs locally.
- **set** — inference is delegated to an external / native host, and the internal
  container stays inactive.

If the local box is underpowered for your model, pointing `OLLAMA_BASE_URL` at a
more capable machine is often the most effective optimization available. There is
also a failover mode (external primary with an internal warm standby) — see
[AI setup](ai-setup.md) for both.

### 3. Container resource limits

Set CPU/memory limits and reservations on individual services to keep a runaway
service from starving the LLM. On a memory-constrained host, **stopping services
you are not using** is more effective than micro-tuning limits — capabilities are
grouped into bundles you can disable as a unit (see
[Self-hosting](self-hosting.md)).

### 4. Qdrant (vector DB)

- Keep embedding dimensionality reasonable for the model in use.
- Reasonable chunk sizes in the RAG pipeline reduce vector count and speed up
  retrieval.

### 5. Neo4j (graph-RAG)

- Ensure appropriate indexes/constraints exist for the entities you query.
- Constrain graph-retrieval depth for graph-RAG queries.

### 6. Redis caching and rate limiting

The API gateway uses Redis for rate limiting (a short rolling window, fail-open).
Redis is also available for caching expensive results where it helps. Keep Redis
healthy — if it degrades, the rate limiter fails open (allows traffic) rather than
blocking.

## Application-level patterns

If you build services or plugins around Minder, these general FastAPI / async
patterns are worth following.

### Use async I/O and reuse clients

```python
# Reuse a single httpx.AsyncClient with connection pooling
http_client = httpx.AsyncClient(
    limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
    timeout=httpx.Timeout(30.0, connect=10.0),
)
```

Avoid creating a new client per request, and avoid blocking calls (`time.sleep`,
sync DB drivers) inside async handlers.

### Paginate list endpoints

Always bound result sets (`limit` / `offset`) rather than returning everything.

### Avoid N+1 queries

Prefer a single query with a join / `= ANY($1)` over per-item lookups in a loop.

## Monitoring

The platform ships an observability stack — use it to get **real** numbers for
your deployment instead of guessing. See [Monitoring](monitoring.md) for the full
stack, and [Troubleshooting](troubleshooting.md) for diagnosing memory / resource
pressure and slow responses.

Quick resource snapshot:

```bash
docker stats --no-stream
```

## Profiling (when you need detail)

```bash
pip install pyinstrument
pyinstrument python -m uvicorn main:app
```

For a quick in-process profile use the stdlib:

```python
import cProfile, pstats
profiler = cProfile.Profile()
profiler.enable()
# ... exercise the code path ...
profiler.disable()
pstats.Stats(profiler).sort_stats("time").print_stats(10)
```

## Load testing

If you want throughput numbers, generate them against your own deployment with a
tool like [Locust](https://locust.io/) — and remember the LLM is usually the
limiting factor, so test the RAG / query paths realistically with the model you
actually run. Write a small `locustfile.py` that exercises your real endpoints
(for example `POST /pipeline/{id}/query`), then:

```bash
pip install locust
locust -f locustfile.py --host http://localhost:8000
```

See [RAG methods](rag-methods.md) for the query endpoint and retrieval options
worth exercising.

## Resources

- [FastAPI performance tips](https://fastapi.tiangolo.com/deployment/concepts/)
- [Ollama documentation](https://github.com/ollama/ollama)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Neo4j performance guide](https://neo4j.com/docs/)
