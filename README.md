# Elastic APM — RCA Automation Agent

An OpenAPI specification and agent directive for automating **Root Cause Analysis (RCA)**, **Performance Audits**, and **SRE alignment** against services monitored by [Elastic APM](https://www.elastic.co/observability/apm).

> **Primary use case:** Load `elastic-apm-rca-v2.yaml` directly into any MCP-compatible AI agent — no glue code, no integration work. Your agent instantly gains full, structured access to every Elastic APM endpoint and can perform autonomous RCA investigations end-to-end.

---

## The Expected Use Case — MCP Integration

This repository is purpose-built to be consumed by AI agents via the **Model Context Protocol (MCP)**. The OpenAPI spec is the tool source; `AGENTS.md` is the system directive. Together they turn any MCP-capable agent (Claude, GPT, Cursor, Continue, etc.) into a fully autonomous APM investigator.

```
┌──────────────────────┐       OpenAPI spec       ┌──────────────────────┐
│   Your APM / Kibana  │ ◄──────────────────────► │   MCP Tool Server    │
│   (8200 / 5601)      │                           │  (e.g. MCE, any      │
└──────────────────────┘                           │   OpenAPI→MCP bridge)│
                                                   └──────────┬───────────┘
                                                              │  MCP protocol
                                                   ┌──────────▼───────────┐
                                                   │   AI Agent           │
                                                   │  (Claude / GPT /     │
                                                   │   Cursor / Continue) │
                                                   │                      │
                                                   │  guided by AGENTS.md │
                                                   └──────────────────────┘
```

### Why this matters

| Without MCP | With this repo + MCP |
|-------------|----------------------|
| Write custom API clients per endpoint | Drop in the YAML — done |
| Manually trace errors across dashboards | Ask the agent: *"Why is my service slow?"* |
| Build alert logic from scratch | Agent proposes SLOs and KQL alerts automatically |
| Context-switch between APM UI tabs | Single conversational investigation |

### Quickstart with [MCE (MCP Code Execution)](https://github.com/hypen-code/mcp-code-execution)

MCE compiles OpenAPI/Swagger specs into sandboxed, executable tools — it is the recommended way to connect this spec to an AI agent.

**1. Install MCE:**
```bash
git clone https://github.com/hypen-code/mcp-code-execution.git
cd mcp-code-execution
pip install -e ".[dev]"
```

**2. Register this spec in `config/swaggers.yaml`:**
```yaml
servers:
  elastic-apm:
    swagger_path: /path/to/elastic-apm-rca-v2.yaml
    auth:
      type: static
      headers:
        Authorization: "Basic <base64(user:pass)>"
        kbn-version: "8.19.10"
```

**3. Build sandbox and compile:**
```bash
docker build -t mce-sandbox:latest sandbox/
docker network create mce_network
mce compile
mce serve
```

**4. Connect your AI agent to the MCP server and paste `AGENTS.md` as the system prompt.**

Your agent can now autonomously run full RCA investigations — listing services, ranking error groups, tracing dependencies, and proposing SLOs — all through natural language.

### Works with any OpenAPI → MCP bridge

MCE is one option. This spec works with any tool that bridges OpenAPI to MCP:

- [MCE](https://github.com/hypen-code/mcp-code-execution) — sandboxed Python execution, SIMD caching, typed return types
- Any other MCP server that accepts an OpenAPI spec as a tool source

---

## What's in this repo

| File | Purpose |
|------|---------|
| `elastic-apm-rca-v2.yaml` | Full OpenAPI 3.0 spec — APM Server (port 8200) + Kibana Internal APM APIs (port 5601) |
| `AGENTS.md` | AI agent system directive — RCA workflows, dependency mapping, SLO/alerting protocols |

The spec reverse-engineers the internal `/internal/apm/*` endpoints from live Kibana browser traffic. These are undocumented by Elastic and have been validated against **Kibana 8.19.10**.

---

## Typical RCA Workflow

Once connected via MCP, the agent follows this investigation chain autonomously:

```
1. GET /internal/apm/has_data                                       → verify data available
2. GET /internal/apm/services                                       → list all services
3. GET /internal/apm/services/{name}/transactions/charts/error_rate → spot elevated error rate
4. GET /internal/apm/services/{name}/errors/groups/main_statistics  → rank error groups
5. GET /internal/apm/services/{name}/errors/{groupId}/samples       → pick a sample trace
6. GET /internal/apm/services/{name}/errors/{groupId}/error/{id}    → full error + trace ID
7. GET /internal/apm/services/{name}/transactions/charts/latency    → correlate latency spike
8. GET /internal/apm/services/{name}/dependencies                   → identify bottleneck
9. GET /internal/apm/services/{name}/metrics/charts                 → JVM / host metrics
```

**Example agent prompt:**
```
Investigate the service "my-service" for the past 1 hour.
Identify the top error group, trace the latency spike to its downstream dependency,
and propose an SLO + Kibana alert to prevent recurrence.
```

---

## Setup & Configuration

### 1. Set Your APM Host

Replace `YOUR_APM_HOST` in `elastic-apm-rca-v2.yaml` with your actual Elastic APM / Kibana host:

```yaml
servers:
  - url: http://YOUR_APM_HOST:8200   # APM Server
  - url: http://YOUR_APM_HOST:5601   # Kibana
```

### 2. Authentication

The spec supports four authentication methods:

| Scheme | When to use |
|--------|-------------|
| `BasicAuth` | Username/password against Kibana (port 5601) |
| `KibanaSession` | Browser session cookie (`sid`) — not recommended for automation |
| `ApmBearerToken` | APM Server shared secret token (port 8200) |
| `ApmApiKey` | APM Server API key, header: `ApiKey base64(id:api_key)` (port 8200) |

### 3. Required Request Parameters

Every Kibana APM call **must** include these parameters:

```
kbn-version: "8.19.10"    # must match your deployed Kibana version exactly
environment: ENVIRONMENT_ALL
kuery: ""
probability: 1.0
document_type: serviceTransactionMetric   # for service-level calls
             | transactionMetric          # for transaction group calls
```

---

## Agent Capabilities (AGENTS.md)

The `AGENTS.md` system directive enables the following multidimensional workflows:

### Cascading Dependency Mapping
- **Downstream** — find who the service calls; traverse Level 2 to locate the ultimate blocker
- **Upstream** — identify which services will go down if the target service fails

### Business Workload Translation
- Maps technical service names to user-facing business workloads
- Explains failures from the end-user perspective, not just stack traces

### SRE Protocol — SLOs and Alerts
- Defines availability and latency SLOs for critical paths
- Generates KQL-based Kibana alerts targeting specific error groups to minimize false positives

---

## API Surface

### APM Server (port 8200)

| Endpoint | Description |
|----------|-------------|
| `GET /` | Health check — returns version and `publish_ready` |
| `GET /config/v1/agents` | Dynamic agent configuration |

### Kibana Internal APM (port 5601)

| Tag | Endpoints |
|-----|-----------|
| Discovery | `has_data`, `environments` |
| Services | list, stats, throughput, error rate, latency, instances |
| Transactions | groups, charts, breakdown |
| Errors | groups, samples, individual error detail |
| Dependencies | downstream metrics, upstream blast radius |
| Metrics | JVM heap, GC, CPU, system memory |
| Infrastructure | Kubernetes pods, container IDs, host attributes |
| Traces | individual trace waterfall |

---

## Common Red Flag Patterns

| Pattern | Likely Cause |
|---------|-------------|
| `Broken pipe / ClientAbortException` | Massive serialization or client timeout |
| `NaN in NumberFormatException` | Frontend passing uninitialized variables |
| `APP_ERR_XXX` | Application validation / configuration gap |
| `The associated entity manager is closed` | JPA transaction boundary or connection leak |
| Latency > 1000ms | Legacy system blocking or heavy database lock |

---

## RCA Report Format

Each investigation produces a structured report covering:

1. **Business Impact** — which user-facing flow is affected
2. **Technical Root Cause** — specific endpoint and exception group
3. **Blast Radius** — which other services fail if this is not resolved
4. **Actionable SLO/Alert** — how to measure and catch this automatically in the future

---

## Compatibility

| Component | Version |
|-----------|---------|
| Elastic APM Server | 8.19.x |
| Kibana | 8.19.x |
| OpenAPI spec | 3.0.3 |

> **Note:** `/internal/apm/*` endpoints are undocumented and reverse-engineered from live browser traffic. They may change between Elastic minor versions. Always validate against your deployed Kibana version.

---

## License

MIT
