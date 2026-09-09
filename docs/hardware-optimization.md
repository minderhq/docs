# Hardware optimization

How to choose and size the hardware you run Minder on, and how to fit the full
stack — LLM inference, vector search, and a knowledge graph — onto constrained
machines like an ARM single-board computer.

!!! note "Measure on your own hardware"
    There are no canned benchmark numbers here. Real behaviour depends on your
    CPU, RAM, storage, and the model you run. Use the observability stack to get
    real figures for *your* box — see [Monitoring](monitoring.md). For the
    software-side tuning levers (model choice, container limits, service
    databases), see [Performance and tuning](performance.md); this page is about
    the hardware underneath them.

## What the platform demands from hardware

Minder runs the whole stack in Docker. On a single machine the two scarce
resources are almost always **RAM** and **CPU**, and **LLM inference (Ollama)**
dominates both. In rough order of impact:

1. **RAM.** The full container stack plus a loaded LLM can easily exceed a small
   machine's memory, forcing swap and severe slowdowns.
2. **CPU.** CPU-only inference is compute-bound; more (and faster) cores directly
   improve generation speed.
3. **Storage I/O.** Vector search and graph queries are I/O sensitive; slow disks
   show up as query latency.
4. **GPU (optional).** A discrete GPU dramatically speeds inference but is not
   required and is absent on most ARM boards.

## Sizing guidance

The dominant factor is the LLM you intend to run — see the per-model RAM table in
[AI setup](ai-setup.md). Size the rest of the box *around* that.

| Resource | Minimum | Comfortable | Notes |
|----------|---------|-------------|-------|
| **RAM** | 8 GB | 16 GB+ | Shared across the whole container stack *and* the model. Small quantized models fit 8 GB; larger models need headroom or should be offloaded. |
| **CPU** | 4 cores | 8+ cores | CPU-only inference scales with core count and clock. ARM and x86 both work. |
| **Disk (free)** | ~20 GB | 50 GB+ | Base models (a small chat model + an embedding model) need only a few GB, but Docker images, database volumes, and additional models add up. Budget 2–8 GB per extra model. |
| **Swap** | some | — | Useful as a safety margin, but heavy swapping tanks performance far more than it does on a server. Treat swap activity as a signal to trim, not a solution. |

### ARM (Raspberry Pi and similar) vs x86

Minder runs on both ARM and x86. A Raspberry Pi 4 (quad-core ARM, 8 GB) is a
workable **development / evaluation** target, with caveats:

- Prefer **small, quantized** models. A ~3B chat model plus a small embedding
  model are sensible defaults on ARM CPU-only inference.
- 7B-and-larger models will run but are slow and memory-hungry on ARM; they may
  swap or hit out-of-memory conditions alongside the rest of the stack.
- If you need larger models, offload inference to a more capable host rather than
  running them locally — see [Offloading inference](#offloading-inference) below.

An x86 box with more RAM, faster cores, and (optionally) a GPU removes most of
these constraints.

### Storage: type matters

Vector (Qdrant) and graph (Neo4j) workloads are I/O sensitive:

- On a single-board computer, boot and store data from a **USB-SSD or NVMe** where
  possible rather than an SD card. Slow flash storage surfaces directly as query
  latency and slow container startup.
- If you must use an SD card, use a high-endurance, high-speed card and expect
  lower throughput.
- On x86, prefer SSD/NVMe over spinning disks for the database volumes.

### CPU vs GPU inference

- **CPU-only** inference works everywhere and is the default on machines without a
  discrete GPU (including all Raspberry Pi 4 boards, which have no CUDA GPU).
  Model size and quantization are what determine whether it's usable.
- **GPU** inference is far faster for larger models. If you have an accelerator,
  run inference on that host. Ollama honours the usual GPU environment variables
  (for example `CUDA_VISIBLE_DEVICES`) on hosts that have one; on a GPU-less
  board they simply have no effect.
- The practical pattern for a small local box is: run the platform on the small
  machine and point inference at a GPU host (see below).

### Thermal and power

Sustained inference keeps CPUs busy for long stretches, which matters on passively
cooled or power-limited hardware:

- Small-board computers can **thermally throttle** under sustained load. Use
  adequate cooling (heatsink and/or fan) if you plan to run inference locally for
  any length of time.
- Use a power supply that meets the board's rated current. Under-powered supplies
  cause instability that looks like software faults.

## Fitting the stack into limited RAM

### Offloading inference

The single most effective move on an underpowered box is to stop running the LLM
on it. Point Minder at a more capable machine and the local host only has to run
the lightweight services:

- Set the external-Ollama mode so inference runs on a native or GPU host, or use
  **failover** mode (external primary with the local container as a warm standby).
- Full instructions, address caveats, and model-availability notes are in
  [AI setup](ai-setup.md).

### Right-size the memory-hungry services

Beyond inference, a few services are memory-sensitive on a small box. The software
knobs live in [Performance and tuning](performance.md); the hardware-fit rule of
thumb is:

- **Keep the graph database's JVM heap small** so it coexists with everything
  else, rather than letting it size a heap for a dedicated server.
- **Cap the cache** (for example an eviction policy with a memory ceiling) well
  under the machine's free RAM if memory pressure appears.
- **Leave the relational database's defaults modest** — server-sized
  `shared_buffers` / `work_mem` values will contend with the LLM and the JVM.
- **Stop capabilities you don't use.** On a memory-constrained host, disabling
  whole capability bundles frees far more than micro-tuning limits — see
  [Self-hosting](self-hosting.md).

### Container resource limits

Docker Compose CPU/memory limits keep any one service from starving the machine.
Most services ship with conservative limits already; the heavy inference container
deliberately leaves its limits for you to set, because the right number depends on
your hardware. Observe real peak usage first, then set a limit:

```bash
docker stats --no-stream
```

To constrain a service, add a limits block for it in the Compose file:

```yaml
services:
  <service>:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
```

!!! warning "Don't over-provision reservations"
    `reservations` that exceed available RAM will prevent a container from
    starting at all. Reserve conservatively on a small box. Note that
    `deploy.replicas` is a Swarm-only field and is not how Minder scales.

## Monitoring hardware usage

Use the built-in observability stack rather than guessing — it gives you host and
per-container CPU, memory, disk, and network:

- **node-exporter** — host CPU / memory / disk / network.
- **cAdvisor** — per-container CPU / memory.
- **Grafana** — visualises both.

See [Monitoring](monitoring.md) for the full stack and alerting, and
[Troubleshooting](troubleshooting.md) for diagnosing memory and resource pressure.

Quick live snapshots from the host:

```bash
# Per-container CPU / memory
docker stats --no-stream

# Host memory
free -h

# Disk
df -h
```

## Practical checklist

1. **Watch total RAM first.** It's shared across the whole stack plus the model;
   the inference container and the graph database are usually the largest
   consumers.
2. **Offload big models.** Point inference at an external or GPU host for anything
   larger than a small quantized model.
3. **Use fast storage.** Prefer USB-SSD / NVMe over SD cards; the databases are
   I/O sensitive.
4. **Cool the box.** Provide adequate cooling and a properly rated power supply if
   you run inference locally.
5. **Don't over-provision limits.** Reservations above available RAM stop
   containers from scheduling.
6. **Treat swap as a warning.** Heavy swapping means it's time to trim services or
   offload inference, not to add more swap.
