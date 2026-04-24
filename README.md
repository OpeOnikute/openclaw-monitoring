# OpenClaw Monitoring with Prometheus

This project uses Prometheus in Docker to scrape metrics from an OpenClaw instance already running on the host machine.

## What was configured

- `docker-compose.yml` runs Prometheus on port `9090`.
- `docker-compose.yml` runs **Grafana** on port `3000` with a provisioned **OpenClaw overview** dashboard (folder **OpenClaw**, UID **openclaw-overview**).
- `docker-compose.yml` runs **Grafana Alloy** with OTLP gRPC `4317`, OTLP HTTP `4318`, and Alloy UI `12345`.
- `docker-compose.yml` runs **Loki** on `3100` for logs and **Tempo** on `3200` for traces.
- Optional: `prometheus/prometheus.yml` can scrape OpenClaw on `host.docker.internal:18789` with bearer auth (`secrets/openclaw_gateway_token`). If you rely only on OTLP → Alloy → remote_write, scrape jobs can stay empty.
- Prometheus is started with `--web.enable-remote-write-receiver` so Alloy can **remote-write** OTLP-derived metrics into the same TSDB.

## Grafana

- URL: [http://localhost:3000](http://localhost:3000) — default login **admin** / **admin** (change via compose env for anything beyond local use).
- Datasource: **Prometheus** at `http://prometheus:9090` (UID `prometheus`), provisioned from `grafana/provisioning/datasources/prometheus.yml`.
- Dashboard JSON: `grafana/provisioning/dashboards/json/openclaw.json` (tokens, cost, message counters, session state rate, histogram quantiles for run duration / context tokens / queue depth).

## Grafana Alloy (OTLP → Prometheus/Loki/Tempo)

This matches the common stack pattern:

```text
OpenClaw → OTLP/HTTP or OTLP/gRPC → Grafana Alloy
         → Prometheus (metrics via remote_write)
         → Loki (logs)
         → Tempo (traces)
```

- **Alloy config:** `alloy/config.alloy` — `otelcol.receiver.otlp` listens on `0.0.0.0:4317` (gRPC) and `0.0.0.0:4318` (HTTP), batches telemetry, then exports:
  - metrics via `otelcol.exporter.prometheus` to `http://prometheus:9090/api/v1/write`
  - logs via `otelcol.exporter.loki` to `http://loki:3100/loki/api/v1/push`
  - traces via `otelcol.exporter.otlp` to `tempo:4317`

**Configure OpenClaw on the host** to send OTLP to Alloy’s published ports (OpenClaw is not inside Docker in this setup):

- OTLP over HTTP/protobuf: base URL **`http://127.0.0.1:4318`** (standard OTLP paths such as `/v1/metrics` are used by the exporter; follow OpenClaw’s exact setting name in `openclaw.json` / docs).
- OTLP over gRPC: **`127.0.0.1:4317`**.

**Verify Alloy**

- Alloy UI: [http://localhost:12345](http://localhost:12345)
- After OpenClaw is exporting OTLP metrics, query Prometheus for a known OTLP metric name or use **Status → TSDB stats** to confirm samples ingested.

**Overlap with scrape**

If OpenClaw exposes the **same** metrics both as Prometheus scrape on `:18789` **and** via OTLP, you may see duplicates or conflicting series. Prefer one path (typically OTLP through Alloy **or** scrape, not both) unless you deliberately want both.

**Logs and traces**

- Logs are now wired end-to-end: OpenClaw OTLP logs → Alloy → Loki.
- Traces are now wired end-to-end: OpenClaw OTLP traces → Alloy → Tempo.

## What to verify in OpenClaw

OpenClaw should expose a Prometheus endpoint at:

- `http://localhost:18789/metrics`

If that URL returns the OpenClaw Control HTML login page, your gateway is protecting the endpoint and you must scrape with auth.

Create the token file used by Prometheus:

1. Copy your gateway token into `secrets/openclaw_gateway_token` (single line, no quotes).
2. Keep `secrets/openclaw_gateway_token` out of git (already ignored).
3. `secrets/openclaw_gateway_token.example` is included as a template only.

## Expected metric types

You should typically see:

- Runtime/process metrics (CPU, memory, heap, event loop lag).
- OpenClaw-specific counters/histograms/gauges for message processing and session state.

Example metric names seen in project discussions:

- `process_cpu_user_seconds_total`
- `nodejs_heap_size_used_bytes`
- `nodejs_eventloop_lag_seconds`
- `iagente_messages_processed_total`
- `iagente_message_duration_seconds`

Note: metric names can vary slightly by OpenClaw version/distribution.

## Run and test

1. Start Prometheus:
   - `docker compose up -d`
2. Check OpenClaw metrics from host with auth:
   - `curl -H "Authorization: Bearer $(cat ~/.openclaw/openclaw.json | jq -r '.gateway.auth.token')" http://localhost:18789/metrics`
3. Open Prometheus UI:
   - `http://localhost:9090`
4. In Prometheus UI, test query:
   - `up{job="openclaw"}`
5. Verify logs path:
   - `curl -s http://localhost:12345/metrics | rg 'otelcol_receiver_accepted_log_records'`
6. Verify traces path:
   - `curl -s http://localhost:12345/metrics | rg 'otelcol_receiver_accepted_spans'`
   - `curl -s http://localhost:3200/metrics | rg 'tempo_distributor_spans_received_total'`

If `up{job="openclaw"} == 1`, scraping is working.

## Why this is necessary

If `/metrics` returns `text/html` (OpenClaw Control app) instead of Prometheus text format, Prometheus marks the target as down or parse-error. Supplying the gateway bearer token lets Prometheus access the real metrics payload.

## Research summary

Research indicates OpenClaw has active work and shipped support around Prometheus observability, including:

- A native `/metrics` endpoint in Prometheus text format.
- Config flags for enabling/disabling Prometheus under diagnostics/observability config.
- Coverage of both Node.js runtime metrics and OpenClaw application metrics.

Because OpenClaw evolves quickly, always confirm the exact endpoint port and config keys in the version you are currently running.
