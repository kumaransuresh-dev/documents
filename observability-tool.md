# Local Observability Stack Setup Guide
## VirtualBox Ubuntu — OTel Collector, Grafana, Tempo, Loki, Prometheus, Jaeger

---

## Prerequisites

### VirtualBox VM Recommended Specs
- OS: Ubuntu 22.04 LTS
- RAM: At least 6 GB (8 GB recommended)
- CPU: 2+ cores
- Disk: 30 GB+
- Network: Bridged Adapter or NAT with port forwarding

### Software Required on Ubuntu VM
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git docker.io docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker is working:
```bash
docker --version
docker compose version
```

---

## Architecture Overview

```
Your App / Test Client
        |
        v
OpenTelemetry Collector  (ports 4317 gRPC, 4318 HTTP)
        |
   _____|_______________________
   |           |               |
   v           v               v
Tempo       Loki          Prometheus
(traces)    (logs)        (metrics)
   |           |               |
   |___________|_______________|
                |
                v
            Grafana  (port 3000)
                |
            Jaeger UI (port 16686)  ← also receives traces
```

---

## Step 1 — Create Project Directory

```bash
mkdir ~/observability && cd ~/observability
```

---

## Step 2 — Create Docker Compose File

Create the file `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
version: "3.8"

networks:
  observability:
    driver: bridge

volumes:
  grafana-data:
  prometheus-data:
  loki-data:
  tempo-data:

services:

  # ─── Grafana ────────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_AUTH_DISABLE_LOGIN_FORM=true
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - observability
    depends_on:
      - prometheus
      - loki
      - tempo

  # ─── Prometheus ─────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
    networks:
      - observability

  # ─── Loki ───────────────────────────────────────────────────
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - observability

  # ─── Tempo ──────────────────────────────────────────────────
  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    ports:
      - "3200:3200"   # Tempo HTTP API
      - "4317:4317"   # OTLP gRPC ingest  ← collector sends traces here
      - "4318:4318"   # OTLP HTTP ingest
    volumes:
      - ./tempo/tempo-config.yml:/etc/tempo/tempo-config.yml
      - tempo-data:/var/tempo
    command: ["-config.file=/etc/tempo/tempo-config.yml"]
    networks:
      - observability

  # ─── Jaeger ─────────────────────────────────────────────────
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # Jaeger gRPC
      - "14268:14268"  # Jaeger HTTP thrift
      - "6831:6831/udp"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - observability

  # ─── OTel Collector ─────────────────────────────────────────
  otelcol:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otelcol
    ports:
      - "4319:4317"   # OTLP gRPC  (host→collector; collector→tempo uses internal network)
      - "4320:4318"   # OTLP HTTP
      - "8888:8888"   # Collector self metrics
      - "13133:13133" # Health check
      - "55679:55679" # zPages
    volumes:
      - ./otelcol/otelcol-config.yml:/etc/otelcol-contrib/config.yaml
    depends_on:
      - tempo
      - loki
      - jaeger
      - prometheus
    networks:
      - observability
```

Save and exit (Ctrl+O, Enter, Ctrl+X in nano).

---

## Step 3 — Create Config Files

### 3a. OTel Collector Config

```bash
mkdir -p otelcol
nano otelcol/otelcol-config.yml
```

Paste:

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  zpages:
    endpoint: 0.0.0.0:55679

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          scrape_interval: 10s
          static_configs:
            - targets: ['localhost:8888']

processors:
  batch:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  # Traces → Tempo
  otlp/tempo:
    endpoint: "tempo:4317"
    tls:
      insecure: true

  # Traces → Jaeger (also receives traces)
  otlp/jaeger:
    endpoint: "jaeger:4317"
    tls:
      insecure: true

  # Logs → Loki
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

  # Metrics → Prometheus (via remote write)
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"
    tls:
      insecure: true

  # Debug output in collector logs
  debug:
    verbosity: basic

service:
  extensions: [health_check, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo, otlp/jaeger, debug]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki, debug]
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite, debug]
  telemetry:
    logs:
      level: "info"
```

### 3b. Prometheus Config

```bash
mkdir -p prometheus
nano prometheus/prometheus.yml
```

Paste:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otelcol:8888']

  - job_name: 'jaeger'
    static_configs:
      - targets: ['jaeger:14269']
```

### 3c. Loki Config

```bash
mkdir -p loki
nano loki/loki-config.yml
```

Paste:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093
```

### 3d. Tempo Config

```bash
mkdir -p tempo
nano tempo/tempo-config.yml
```

Paste:

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 1h

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal

metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /var/tempo/generator/wal

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
```

### 3e. Grafana Datasource Provisioning

```bash
mkdir -p grafana/provisioning/datasources
nano grafana/provisioning/datasources/datasources.yml
```

Paste:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
      serviceMap:
        datasourceUid: prometheus
      lokiSearch:
        datasourceUid: loki
```

---

## Step 4 — Start the Stack

```bash
cd ~/observability
docker compose up -d
```

Check all containers are running:

```bash
docker compose ps
```

Check logs for any errors:

```bash
docker compose logs otelcol
docker compose logs tempo
docker compose logs loki
```

---

## Step 5 — Verify Each Service

| Service        | URL                          | What to check                     |
|----------------|------------------------------|-----------------------------------|
| Grafana        | http://localhost:3000        | Dashboards, Explore                |
| Prometheus     | http://localhost:9090        | Targets should show UP             |
| Jaeger UI      | http://localhost:16686       | Services tab                       |
| OTel zPages    | http://localhost:55679       | Pipeline stats                     |
| OTel Health    | http://localhost:13133       | Should return HTTP 200             |
| Loki           | http://localhost:3100/ready  | Should return "ready"              |
| Tempo          | http://localhost:3200/ready  | Should return "ready"              |

---

## Step 6 — Test with a Sample Trace

Install the OTEL CLI tool to quickly send a test trace:

```bash
# Send a test trace to the collector (gRPC on port 4319)
docker run --rm --network host \
  otel/opentelemetry-collector-contrib:latest \
  telemetrygen traces \
  --otlp-insecure \
  --otlp-endpoint localhost:4319 \
  --duration 5s
```

Or install `telemetrygen` directly:

```bash
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest
telemetrygen traces --otlp-insecure --otlp-endpoint localhost:4319 --duration 5s
telemetrygen logs   --otlp-insecure --otlp-endpoint localhost:4319 --duration 5s
telemetrygen metrics --otlp-insecure --otlp-endpoint localhost:4319 --duration 5s
```

---

## Step 7 — View Results in Grafana

1. Open http://localhost:3000
2. Go to **Explore** (compass icon on the left)
3. Select **Tempo** from the datasource dropdown → search for traces
4. Select **Loki** → query `{exporter="OTLP"}`
5. Select **Prometheus** → query `otelcol_receiver_accepted_spans_total`

---

## Step 8 — Access from Windows Host

If using NAT in VirtualBox, set up port forwarding:

1. In VirtualBox, go to your VM Settings → Network → Adapter 1 → Advanced → Port Forwarding
2. Add rules:

| Name        | Host Port | Guest Port |
|-------------|-----------|------------|
| Grafana     | 3000      | 3000       |
| Prometheus  | 9090      | 9090       |
| Jaeger      | 16686     | 16686      |
| OTel gRPC   | 4319      | 4319       |
| OTel HTTP   | 4320      | 4320       |

Then access from Windows at `http://localhost:<host_port>`.

If using Bridged Adapter, get your VM's IP:
```bash
ip addr show | grep inet
```
Then use `http://<vm-ip>:3000` etc. from your Windows browser.

---

## Useful Commands

```bash
# Stop everything
docker compose down

# Stop and remove volumes (fresh start)
docker compose down -v

# Restart a single service
docker compose restart otelcol

# Stream logs from all services
docker compose logs -f

# Stream logs from one service
docker compose logs -f tempo

# Check resource usage
docker stats
```

---

## Sending Data from Your Own App

Point your application's OTLP exporter to:

| Protocol   | Endpoint                          |
|------------|-----------------------------------|
| gRPC       | `http://localhost:4319`           |
| HTTP/proto | `http://localhost:4320`           |

Example environment variables for most OTEL SDKs:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4319
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_SERVICE_NAME=my-test-app
```

---

## Troubleshooting

**Collector fails to start:**
```bash
docker compose logs otelcol
# Validate config manually:
docker run --rm -v $(pwd)/otelcol:/etc/otelcol-contrib \
  otel/opentelemetry-collector-contrib:latest \
  validate --config /etc/otelcol-contrib/otelcol-config.yml
```

**Tempo not receiving traces:**
- Confirm collector exporter endpoint is `tempo:4317` (internal docker network)
- Check `docker compose logs tempo`

**Grafana datasources not connecting:**
- Check container names match the URLs in datasources.yml
- All services must be on the same docker network (`observability`)

**Port conflict on host:**
```bash
sudo lsof -i :<port>
```
Change the host-side port mapping in docker-compose.yml if needed (left side of `host:container`).
