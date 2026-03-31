# Elastic APM — RCA Automation Agent

An OpenAPI specification and agent directive for automating **Root Cause Analysis (RCA)**, **Performance Audits**, and **SRE alignment** against services monitored by [Elastic APM](https://www.elastic.co/observability/apm).

---

## Overview

This project provides two files:

| File | Purpose |
|------|---------|
| `elastic-apm-rca-v2.yaml` | Full OpenAPI 3.0 spec covering Elastic APM Server (port 8200) and Kibana Internal APM APIs (port 5601) |
| `AGENTS.md` | AI agent directives for systematic RCA, dependency mapping, and SRE-aligned SLO/alerting workflows |

The spec targets **Kibana 8.19.x** and reverse-engineers the internal `/internal/apm/*` endpoints from live browser traffic. These endpoints are undocumented by Elastic and may change between minor versions.

---

## Typical RCA Workflow

```
1. GET /internal/apm/has_data                                      → verify data available
2. GET /internal/apm/services                                      → list all services
3. GET /internal/apm/services/{name}/transactions/charts/error_rate → spot elevated error rate
4. GET /internal/apm/services/{name}/errors/groups/main_statistics → rank error groups
5. GET /internal/apm/services/{name}/errors/{groupId}/samples      → pick a sample trace
6. GET /internal/apm/services/{name}/errors/{groupId}/error/{id}   → full error + trace ID
7. GET /internal/apm/services/{name}/transactions/charts/latency   → correlate latency spike
8. GET /internal/apm/services/{name}/dependencies                  → identify bottleneck
9. GET /internal/apm/services/{name}/metrics/charts                → JVM / host metrics
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

The spec supports four authentication methods — choose the one that matches your deployment:

| Scheme | When to use |
|--------|-------------|
| `BasicAuth` | Username/password against Kibana (port 5601) |
| `KibanaSession` | Browser session cookie (`sid`) — not recommended for automation |
| `ApmBearerToken` | APM Server shared secret token (port 8200) |
| `ApmApiKey` | APM Server API key, header: `ApiKey base64(id:api_key)` (port 8200) |

### 3. Required Request Parameters

Every Kibana APM call **must** include:

```
kbn-version: "8.19.10"    # must match your deployed Kibana version
environment: ENVIRONMENT_ALL
kuery: ""
probability: 1.0
document_type: serviceTransactionMetric   # for service-level calls
             | transactionMetric          # for transaction group calls
```

---

## Using with an AI Agent / MCP

Load `elastic-apm-rca-v2.yaml` as an MCP (Model Context Protocol) tool source or any OpenAPI-compatible agent framework. The `AGENTS.md` file contains directives that guide the agent through:

- **Cascading Dependency Mapping** — upstream and downstream blast-radius analysis
- **Business Workload Translation** — mapping service names to user-facing impact
- **SRE Protocol** — defining SLOs and high-signal KQL-based alerts

### Example prompt

```
Investigate the service "my-service" for the past 1 hour.
Identify the top error group, trace the latency spike to its downstream dependency,
and propose an SLO + Kibana alert to prevent recurrence.
```

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

## Reporting Format

Each RCA report produced by the agent covers:

1. **Business Impact** — which user-facing flow is affected
2. **Technical Root Cause** — specific endpoint and exception group
3. **Blast Radius** — which other services fail if this is not resolved
4. **Actionable SLO/Alert** — how to measure and catch this automatically

---

## Compatibility

| Component | Version |
|-----------|---------|
| Elastic APM Server | 8.19.x |
| Kibana | 8.19.x |
| OpenAPI spec | 3.0.3 |

> **Note:** `/internal/apm/*` endpoints are undocumented and reverse-engineered. They may break on Elastic upgrades. Always validate against your deployed version.

---

## License

MIT
