---
title: Observability Setup
---

# Observability Setup

ICP provides centralized observability for BI runtimes. Logs and metrics are collected via Fluent Bit, stored in OpenSearch, and displayed in the ICP Console.

For MI runtimes, see [MI Observability Setup](observability-setup-mi.md).

## Architecture

```
┌──────────┐   log files   ┌───────────┐   HTTP    ┌────────────┐
│    BI    │──────────────▶│ Fluent Bit │─────────▶│ OpenSearch  │
│ Runtime  │               └───────────┘           └─────┬──────┘
└────┬─────┘                                             │
     │ heartbeat                                         │ query
     ▼                                                   ▼
┌─────────┐                                     ┌──────────────┐
│   ICP   │◀────────────────────────────────────│ ICP Console  │
│  Server │          GraphQL / REST              │  (Browser)   │
└─────────┘                                     └──────────────┘
```

1. The BI runtime writes structured logs to two separate files: application logs and metrics logs.
2. Fluent Bit tails both files and ships each to its own OpenSearch index.
3. ICP Server queries OpenSearch when a user opens the Logs or Metrics page in the Console.

## Prerequisites

| Component | Purpose |
|-----------|---------|
| OpenSearch | Log and metrics storage |
| Fluent Bit | Log collection and forwarding |
| ICP Server | Observability API layer |
| ICP Console secret | Created in the Console under the organization's environment settings. Format: `<keyId>.<keyMaterial>`. |

## Step 1: Deploy OpenSearch

Any single-node or clustered OpenSearch deployment works. ICP needs HTTP(S) access to the OpenSearch REST API.

### Docker (recommended for evaluation)

The bundled Docker Compose stack in `icp_server/resources/observability/opensearch-observability-dashboard/` starts OpenSearch, OpenSearch Dashboards, Fluent Bit, and Data Prepper together:

```bash
cd icp_server/resources/observability/opensearch-observability-dashboard/

# Create the external Docker network (required — the compose file references it)
docker network create observability_network

# Edit config/.env and set a strong admin password
vim config/.env   # set OPENSEARCH_INITIAL_ADMIN_PASSWORD=YourStr0ng!Pass

docker compose up -d
```

### Existing OpenSearch cluster

If you already have OpenSearch running, skip to Step 2. Note the host, port, and credentials — you will configure them in the ICP Server.

## Step 2: Create Index Templates

Index templates ensure OpenSearch maps fields with the correct types before any data arrives. Apply them once per cluster.

The shipped template file is at:

```
icp_server/resources/observability/opensearch-observability-dashboard/setup/index-template-request.json
```

Apply it:

```bash
curl -X PUT 'https://<opensearch-host>:9200/_index_template/wso2_integration_application_log_template' \
  -ku admin:<password> \
  -H 'Content-Type: application/json' \
  -d @setup/index-template-request.json
```

This template covers `ballerina-application-logs-*` and `mi-application-logs-*` index patterns.

For metrics indices, a separate template is recommended to ensure correct numeric types:

```bash
curl -X PUT 'https://<opensearch-host>:9200/_index_template/wso2_integration_metrics_log_template' \
  -ku admin:<password> \
  -H 'Content-Type: application/json' \
  -d '{
    "index_patterns": ["ballerina-metrics-logs-*", "mi-metrics-logs-*"],
    "template": {
      "mappings": {
        "properties": {
          "time":                   { "type": "date" },
          "message":                { "type": "text" },
          "response_time_seconds":  { "type": "float" },
          "response_time":          { "type": "long" }
        }
      }
    }
  }'
```

If you use the bundled Docker Compose, the `opensearch-setup` service applies the application-log template automatically. You still need to apply the metrics template manually.

## Step 3: Configure ICP Server

Add the OpenSearch connection to ICP Server's `deployment.toml`:

```toml
opensearchUrl = "https://localhost:9200"
opensearchUsername = "admin"
opensearchPassword = "<your-opensearch-password>"
```

ICP Server exposes the observability adapter on port `9449` by default. The Console calls it through the main ICP port — no additional port needs to be exposed to users.

For self-signed TLS on OpenSearch, also set:

```toml
[observabilitySecureSocket]
allowInsecureTLS = true   # accepts self-signed certs (non-production)

# For production, use a proper truststore instead:
# [observabilitySecureSocket]
# allowInsecureTLS = false
# truststorePath = "/path/to/truststore.p12"
# truststorePassword = "changeit"
```

## Step 4: Configure the Integration

Observability requires changes in the integration project itself — these are not server-side settings.

### 1. Add the runtime bridge and metrics dependencies

In your Ballerina project's `main.bal` (or any `.bal` file), import the ICP runtime bridge and metrics logger:

```ballerina
import wso2/icp.runtime.bridge as _;
import ballerinax/metrics.logs as _;
```

Both are blank imports (`as _`) — they activate automatically at startup.

### 2. Enable logging and metrics in `Config.toml`

```toml
[ballerina.observe]
metricsLogsEnabled = true

[ballerina.log]
format = "logfmt"

[[ballerina.log.destinations]]
path = "./logs/app.log"

[ballerinax.metrics.logs]
logFilePath = "./logs/metrics.log"
```

This produces two separate log files:

| File | Content | OpenSearch index |
|------|---------|------------------|
| `logs/app.log` | Application logs (startup, errors, user log statements) | `ballerina-application-logs-*` |
| `logs/metrics.log` | Per-request metrics (response times, status codes, endpoints) | `ballerina-metrics-logs-*` |

| Setting | Purpose |
|---------|---------|
| `metricsLogsEnabled = true` | Enables the Ballerina runtime to emit per-request metrics |
| `format = "logfmt"` | Structured log output that Fluent Bit's `bal_logfmt_parser` can parse |
| `path = "./logs/app.log"` | Application log destination |
| `logFilePath = "./logs/metrics.log"` | Metrics log destination (separate file via `ballerinax/metrics.logs`) |

The log file paths must match the Fluent Bit input `Path` patterns. Adjust both sides if you change the directory layout.

### 3. Configure the ICP runtime bridge

Also in `Config.toml`, configure the bridge so the runtime registers with ICP and sends heartbeats:

```toml
[wso2.icp.runtime.bridge]
serverUrl = "https://<icp-server-host>:9445"
secret = "<key-id>.<key-material>"
project = "my-project"
integration = "my-integration"
environment = "dev"
heartbeatInterval = 10
```

The `secret` must be created **before** starting the BI runtime. See [Connect an Integration to ICP](connect-runtime.md) for details.

## Step 5: Configure Fluent Bit

Fluent Bit tails the BI log files and ships them to OpenSearch. The full reference configuration is in:

```
icp_server/resources/observability/opensearch-observability-dashboard/config/fluent-bit/
```

### Key files

- **`fluent-bit.conf`** — main pipeline config (inputs, filters, outputs)
- **`parsers.conf`** — log format parsers
- **`scripts/scripts.lua`** — Lua enrichment and routing logic

### Inputs and outputs

Since the integration writes app logs and metrics to separate files, Fluent Bit uses two independent inputs — no tag-rewriting or filtering needed to separate them:

| Input `Path` | Tag | Parser | Output index prefix | Content |
|-------------|-----|--------|---------------------|---------|
| `<bi-logs>/app.log` | `ballerina_app_logs` | `bal_logfmt_parser` | `ballerina-application-logs-` | Application logs |
| `<bi-logs>/metrics.log` | `ballerina_metrics` | `bal_logfmt_parser` | `ballerina-metrics-logs-` | Per-request metrics |

### Minimal Fluent Bit config

The shipped `scripts/scripts.lua` provides these Lua functions used by the filters below:

- `extract_app_from_path` — derives `app_name` from the log file path
- `enrich_bal_logs` — adds `product` and `app_module` fields
- `construct_bal_app_name` — builds the `app` and `deployment` fields
- `extract_bal_metrics_data` — parses metrics-specific fields (response time, status, etc.)
- `generate_document_id` — creates a hash-based `doc_id` for deduplication

```ini
[SERVICE]
    Flush        1
    Parsers_File parsers.conf

# ── App logs ──
[INPUT]
    Name         tail
    Path         /path/to/bi/logs/app.log
    Parser       bal_logfmt_parser
    Tag          ballerina_app_logs
    Read_from_Head On
    Path_Key     log_file_path

# ── Metrics logs (separate file) ──
[INPUT]
    Name         tail
    Path         /path/to/bi/logs/metrics.log
    Parser       bal_logfmt_parser
    Tag          ballerina_metrics
    Read_from_Head On
    Path_Key     log_file_path

# ── Enrich app logs ──
[FILTER]
    Name    lua
    Match   ballerina_app_logs
    Script  scripts/scripts.lua
    Call    extract_app_from_path

[FILTER]
    Name    lua
    Match   ballerina_app_logs
    Script  scripts/scripts.lua
    Call    enrich_bal_logs

[FILTER]
    Name    lua
    Match   ballerina_app_logs
    Script  scripts/scripts.lua
    Call    construct_bal_app_name

# ── Enrich metrics logs ──
[FILTER]
    Name    lua
    Match   ballerina_metrics
    Script  scripts/scripts.lua
    Call    extract_app_from_path

[FILTER]
    Name    lua
    Match   ballerina_metrics
    Script  scripts/scripts.lua
    Call    enrich_bal_logs

[FILTER]
    Name    lua
    Match   ballerina_metrics
    Script  scripts/scripts.lua
    Call    construct_bal_app_name

[FILTER]
    Name    lua
    Match   ballerina_metrics
    Script  scripts/scripts.lua
    Call    extract_bal_metrics_data

# ── Document IDs ──
[FILTER]
    Name    lua
    Match   ballerina_app_logs
    Script  scripts/scripts.lua
    Call    generate_document_id
    time_as_table true

[FILTER]
    Name    lua
    Match   ballerina_metrics
    Script  scripts/scripts.lua
    Call    generate_document_id
    time_as_table true

# ── Outputs ──
[OUTPUT]
    Name            opensearch
    Match           ballerina_app_logs
    Host            localhost
    Port            9200
    Logstash_Format On
    Logstash_Prefix ballerina-application-logs
    Replace_Dots    On
    Suppress_Type_Name On
    Id_Key          doc_id
    tls             On
    tls.verify      Off
    HTTP_User       admin
    HTTP_Passwd     <password>

[OUTPUT]
    Name            opensearch
    Match           ballerina_metrics
    Host            localhost
    Port            9200
    Logstash_Format On
    Logstash_Prefix ballerina-metrics-logs
    Replace_Dots    On
    Suppress_Type_Name On
    Id_Key          doc_id
    tls             On
    tls.verify      Off
    HTTP_User       admin
    HTTP_Passwd     <password>
```

`Replace_Dots On` is important — Ballerina logfmt fields contain dots (e.g. `src.module`, `http.method`) which OpenSearch rejects as field names. This setting converts them to underscores.

## Verification

### Check OpenSearch indices

After the BI runtime has been running for a minute or two:

```bash
curl -ku admin:<password> https://localhost:9200/_cat/indices?v
```

You should see:

```
ballerina-application-logs-2026-04-28
ballerina-metrics-logs-2026-04-28
```

### Check Fluent Bit health

```bash
curl http://localhost:2020/api/v1/metrics
```

Look for non-zero `proc_records` in the output sections and zero `errors`.

### Check ICP Console

1. Log into the ICP Console.
2. Navigate to a project that has a connected BI runtime.
3. Open **Logs** — you should see runtime log entries.
4. Open **Metrics** — you should see request counts and latency charts.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Metrics page shows "No metrics data" | BI runtime has no inbound HTTP requests | Metrics are generated per-request — send traffic first |
| Metrics page shows "No metrics data" | `metricsLogsEnabled` not set | Add `metricsLogsEnabled = true` to `[ballerina.observe]` in `Config.toml` |
| Metrics page shows "No metrics data" | Metrics log file not configured | Set `logFilePath` in `[ballerinax.metrics.logs]` |
| Logs page shows "Observability service is unavailable" | ICP Server can't reach OpenSearch | Verify `opensearchUrl` in ICP Server's `deployment.toml` |
| OpenSearch rejects documents with "total fields [1000] exceeded" | Deeply nested JSON in log messages | Increase limit: `curl -X PUT '.../_settings' -d '{"index.mapping.total_fields.limit": 2000}'` or add to the index template |

## Index Lifecycle

Indices are created daily with a date suffix (e.g. `ballerina-metrics-logs-2026-04-28`). To manage disk usage:

- Use [OpenSearch Index State Management (ISM)](https://opensearch.org/docs/latest/im-plugin/ism/index/) policies to automatically delete or roll over old indices.
- A typical retention policy keeps 30 days of logs and 90 days of metrics.

## Security Notes

- In production, enable TLS on OpenSearch and set `tls.verify On` in Fluent Bit.
- Use dedicated OpenSearch credentials for Fluent Bit (write-only) and ICP Server (read-only).
- The ICP Server generates short-lived JWTs (2 min) for internal communication between its observability service and its OpenSearch adapter — no user configuration needed.
