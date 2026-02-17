# New Relic OpenTelemetry Collector

A GitHub Codespace-ready OpenTelemetry Collector that receives traces, logs, and metrics via OTLP and forwards them to New Relic.

## Architecture

```
Your App / FluentBit
        │
        │  OTLP (gRPC :4317 or HTTP :4318)
        ▼
┌─────────────────────┐
│   OTel Collector    │  ← runs in this Codespace
│  (this repo)        │
└─────────────────────┘
        │
        │  OTLP HTTP :443
        ▼
  New Relic Backend
  (Traces / Logs / Metrics)
```

## Quick Start

### 1. Open in Codespace

Click **Code → Codespaces → Create codespace on main**.

The Codespace will automatically:
- Install Docker
- Start the OTel Collector via `docker compose up -d`
- Forward ports 4317 and 4318

### 2. Add Your New Relic License Key

Option A — via Codespace Secrets (recommended):
1. Go to **github.com → Settings → Codespaces → Secrets**
2. Add secret: `NEW_RELIC_LICENSE_KEY` = your license key
3. Rebuild the Codespace

Option B — via `.env` file (local only, never committed):
```bash
cp .env.example .env
# Edit .env and add your license key
nano .env
# Restart the collector
docker compose down && docker compose up -d
```

Get your license key from [New Relic API Keys](https://one.newrelic.com/admin-portal/api-keys/home). Use the **License Key** (starts with `NRAK-`), not the User API key.

### 3. Verify the Collector is Running

```bash
# Check container status
docker compose ps

# Check health endpoint
curl http://localhost:13133/

# Watch collector logs
docker compose logs -f
```

### 4. Send a Test Trace

```bash
chmod +x send-test-trace.sh
./send-test-trace.sh
```

Then check **New Relic → APM & Services → Distributed Tracing** for your test trace.

---

## Endpoints

| Port | Protocol | Purpose |
|------|----------|---------|
| `4317` | gRPC | OTLP traces, logs, metrics |
| `4318` | HTTP | OTLP traces, logs, metrics |
| `8888` | HTTP | Collector's own Prometheus metrics |
| `13133` | HTTP | Health check |

### OTLP HTTP paths

| Signal | Path |
|--------|------|
| Traces | `POST /v1/traces` |
| Logs | `POST /v1/logs` |
| Metrics | `POST /v1/metrics` |

---

## Sending Data to the Collector

### From FluentBit (logs)

Update your FluentBit ConfigMap output section:

```ini
[OUTPUT]
    name          opentelemetry
    match         *
    host          <your-codespace-name>-4318.app.github.dev
    port          443
    logs_uri      /v1/logs
    tls           on
    tls.verify    on
```

> **Note:** Get the forwarded URL from the **Ports** tab in your Codespace. Make the port **public** so external services can reach it.

### From an Application (OTel SDK)

Set these environment variables in your app:

```bash
# gRPC
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# HTTP
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

From outside the Codespace, use the forwarded URL from the **Ports** tab (HTTPS, port 443).

### From curl (manual test)

```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "resourceSpans": [{
    "resource": {
      "attributes": [{
        "key": "service.name",
        "value": { "stringValue": "my-service" }
      }]
    },
    "scopeSpans": [{
      "spans": [{
        "traceId": "5b8aa5a2d2c872e8321cf37308d69df2",
        "spanId": "051581bf3cb55c13",
        "name": "my-operation",
        "kind": 1,
        "startTimeUnixNano": "1544712660000000000",
        "endTimeUnixNano":   "1544712661000000000",
        "status": { "code": 1 }
      }]
    }]
  }]
}
EOF
```

---

## Configuration

The collector is configured in `otel-collector-config.yaml`. It has three pipelines:

- **traces** — receives and forwards spans to New Relic Distributed Tracing
- **logs** — receives and forwards log records to New Relic Logs
- **metrics** — receives and forwards metrics to New Relic Metrics

The `logging` exporter on the traces and logs pipelines prints received data to stdout, useful for debugging. Remove it from `otel-collector-config.yaml` if you want quieter output.

### Modifying the Config

After any change to `otel-collector-config.yaml`:

```bash
docker compose down && docker compose up -d
docker compose logs -f
```

---

## Viewing Data in New Relic

| Signal | Location in New Relic |
|--------|----------------------|
| Traces | APM & Services → Distributed Tracing |
| Logs | Logs |
| Metrics | Metrics & Events |

---

## Troubleshooting

**Collector won't start:**
```bash
docker compose logs otel-collector
```

**License key not set:**
```bash
echo $NEW_RELIC_LICENSE_KEY  # should print your key
docker compose down && docker compose up -d
```

**Port not reachable from outside Codespace:**
- Go to the **Ports** tab in VS Code / Codespace UI
- Right-click port 4318 → **Port Visibility → Public**

**New Relic not receiving data:**
- Confirm HTTP 200 responses in `docker compose logs -f`
- Check you're using the **License Key** (NRAK-...), not a User API key
- Verify your New Relic account region — EU accounts should use `otlp.eu.nr-data.net`

---

## Files

```
.
├── .devcontainer/
│   └── devcontainer.json       # Codespace configuration
├── .env.example                # Environment variable template
├── .gitignore
├── docker-compose.yml          # Collector service definition
├── otel-collector-config.yaml  # OTel Collector pipeline config
├── send-test-trace.sh          # Script to send a test trace
└── README.md
```
