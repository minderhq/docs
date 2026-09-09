# Monitoring

Minder ships a full observability stack — metrics, dashboards, tracing, and
alerting — as the **`monitoring`** capability bundle. Like every other part of
the platform it runs as Docker containers named `minder-<service>`, provisioned
through `bash setup.sh` (see [Self-hosting](self-hosting.md)).

!!! note
    Host ports are **loopback-bound** by default: reachable on the box itself
    (`curl http://localhost:<port>`) but not from other machines. External
    access is through the Traefik reverse proxy, SSO-gated. The commands below
    assume you are on the host, in your Minder checkout.

## Observability stack

| Component | Container | Host port | Role |
|---|---|---|---|
| Prometheus | `minder-prometheus` | 9090 | Metrics collection + storage |
| Grafana | `minder-grafana` | 3000 | Dashboards / visualization |
| Alertmanager | `minder-alertmanager` | 9093 | Alert routing |
| Jaeger | `minder-jaeger` | Traefik-only (no host port) | Distributed tracing |
| OTel Collector | `minder-otel-collector` | 14317 / 14318 / 18888 | OpenTelemetry pipeline |
| InfluxDB | `minder-influxdb` | 8086 | Time-series data |
| Telegraf | `minder-telegraf` | — (internal) | Metrics collection agent |

### Exporters (all internal)

| Exporter | Internal port | Scraped by Prometheus? |
|---|---|---|
| postgres-exporter | 9187 | Yes |
| redis-exporter | 9121 | Yes |
| rabbitmq-exporter | 9090 | Yes |
| node-exporter | 9100 | Yes |
| cadvisor | 8080 | No — reachable directly, no scrape job configured |
| blackbox-exporter | 9115 | No — reachable directly, no scrape job configured |

!!! note "OTel Collector ports are remapped"
    The collector uses `14317` / `14318` / `18888` instead of the OpenTelemetry
    defaults (`4317` / `4318`) to avoid a conflict with Jaeger's own OTLP ports
    on the all-in-one image.

## Quick start

Monitoring is a capability bundle, so bring the whole stack up (or down) as a
unit rather than editing container definitions:

```bash
bash setup.sh bundle enable monitoring     # start the stack
bash setup.sh bundle status                # bundles + their services
bash setup.sh bundle disable monitoring --stop-orphans   # take it down
```

### Verify

```bash
# Containers running
docker ps | grep -E "prometheus|grafana|alertmanager|jaeger|otel-collector|influxdb|telegraf"

# Prometheus targets
curl http://localhost:9090/api/v1/targets

# Prometheus / Grafana / Alertmanager health
curl http://localhost:9090/-/healthy
curl http://localhost:3000/api/health
curl http://localhost:9093/-/healthy
```

The Jaeger UI has no loopback port — it is reachable only through Traefik (e.g.
`https://jaeger.minder.local` once the reverse proxy has real DNS and TLS).

!!! note "\"no healthcheck\" is not \"unhealthy\""
    A few observability containers (for example the OTel Collector and some
    exporters) run with **no Docker healthcheck by design**, because their base
    images lack the tooling a probe would need. They show blank health in
    `docker ps`; that is expected, not a failure. See
    [Troubleshooting](troubleshooting.md) for how to confirm them from their
    logs.

### Grafana access

- URL: `http://localhost:3000`
- Default credentials: `admin` / `admin` — **change on first login**.

Grafana is also routed through Traefik behind Authelia forward-auth:
unauthenticated requests are redirected (`302`) to the Authelia portal. Full
browser SSO requires real DNS and valid TLS on the deployment; direct access on
port `3000` bypasses Traefik and works as normal. See
[Authentication](authentication.md) for the SSO model.

The Prometheus datasource is pre-provisioned to point at
`http://minder-prometheus:9090` (the internal Docker DNS name).

## Instrumentation (OpenTelemetry)

Application services are instrumented with OpenTelemetry and export to the
**OTel Collector** at `minder-otel-collector:14317` (OTLP gRPC). The collector
fans out to Jaeger for traces and to the metrics pipeline. Prometheus
additionally scrapes the app services and exporters directly.

```
services ──OTLP──▶ otel-collector (:14317 gRPC / :14318 HTTP)
                       │
                       ├──▶ Jaeger (traces, UI via Traefik only)
                       └──▶ metrics pipeline (:18888)

Prometheus (:9090) ──scrape──▶ app services + exporters
```

## Prometheus configuration

Prometheus resolves each target by its service name on the internal
`minder-network`. The default `scrape_interval` is 60s. Services with no metrics
endpoint are simply omitted. The block below is an illustrative excerpt:

```yaml
global:
  scrape_interval: 60s
  evaluation_interval: 60s

scrape_configs:
  - job_name: 'api-gateway'
    static_configs:
      - targets: ['api-gateway:8000']
    metrics_path: '/metrics'

  - job_name: 'rag-pipeline'
    static_configs:
      - targets: ['rag-pipeline:8004']
    metrics_path: '/metrics'

  - job_name: 'postgres-exporter'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis-exporter'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

cAdvisor and blackbox-exporter run and are reachable directly, but are not
scraped by Prometheus (no job configured).

## Available dashboards

Grafana is pre-provisioned with dashboards covering:

- **Minder Overview** — service health, request metrics, error rates, resource usage
- **API Gateway** — HTTP request count / latency, health, JWT auth activity
- **PostgreSQL** — database size, connection counts, query performance
- **Redis** — memory usage, connections, cache hit rate

## Alerting

### Prometheus alert rules

Alert rules are organized into groups by concern (service availability,
performance, infrastructure, plugins, RAG pipeline, model management). For
example:

```yaml
groups:
  - name: service_availability
    interval: 30s
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          category: availability
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Service {{ $labels.job }} has been down for more than 1 minute"

  - name: performance
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
          > 0.05
        for: 5m
        labels:
          severity: warning
          category: performance
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: "Error rate for {{ $labels.job }} is {{ $value | humanizePercentage }} (threshold: 5%)"
```

The shipped rule set also covers latency, Postgres/Redis resource pressure,
plugin health, RAG document-failure rate, and model-registration coverage.

### Alertmanager

Alertmanager routes by severity into three receivers: a default receiver (logs
to stdout only), a critical receiver, and a warning receiver. The actual
email/Slack notification blocks are shipped commented out — fill in real
destinations before relying on them for notifications:

```yaml
route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'service', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  routes:
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 5s
      repeat_interval: 5m
    - match:
        severity: warning
      receiver: 'warning-receiver'
      repeat_interval: 1h

receivers:
  - name: 'default-receiver'
  - name: 'critical-receiver'
    # email_configs / slack_configs: commented out — uncomment and fill in
    # real destinations (e.g. 'oncall@example.com', a Slack webhook URL) before
    # relying on this for real notifications.
  - name: 'warning-receiver'
    # email_configs: commented out, same as above.
```

Test alert delivery after changing any Alertmanager receiver.

## Best practices

- Review Grafana dashboards regularly; watch trends and anomalies.
- Set alert thresholds appropriate for your hardware — on modest hosts such as a
  Raspberry Pi 4, memory and CPU are the limiting factors.
- Re-test alert delivery whenever you change Alertmanager receivers.

!!! note "Centralized logging is not built in"
    There is currently no centralized log aggregation (no ELK/Loki). Logs are
    the per-container Docker logs. A log-aggregation layer is a possible future
    addition, not something shipped today.

## Handy commands

```bash
# Query the "up" metric across all targets
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.'

# Live container resource usage
docker stats --no-stream | grep minder

# Tail a service's logs
docker logs minder-<service> --tail 100 -f
```

## Related documentation

- [Self-hosting](self-hosting.md) — install and the capability-bundle model
- [Using Minder](using-minder.md) — the in-browser platform **Status** page
- [AI setup](ai-setup.md) — inference and model configuration
- [Troubleshooting](troubleshooting.md) — service health and healthcheck notes
