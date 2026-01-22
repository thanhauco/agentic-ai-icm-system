# 🔭 Observability & Control

> **Purpose**: Full observability into system behavior, agent performance, cost management, and operational health. Connected to all modules via dotted-line (non-blocking) connections.

---

## Four Pillars

| Pillar | Module | What It Tracks |
|---|---|---|
| **Error Handling** | Structured errors | Error types, retry counts, circuit breaker states, dead-letter queue |
| **Run History** | Audit trail | Every agent invocation with full input/output, timestamps, latency, decisions made |
| **Health Check** | System health | Module availability, API response times, queue depths, agent readiness |
| **Cost Control** | Token/cost tracking | Token usage per request, per agent, per incident. Budget alerts, rate limiting |

## Error Handling Strategy

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#AAAAAA'}}}%%
graph TD
    A["⚠️ Error"] --> B{"🔎 Type?"}
    B -->|Transient| C["🔄 Retry"]
    B -->|Rate Limit| D["⏳ Queue"]
    B -->|Validation| E["📋 Structured Error"]
    B -->|Fatal| F["🚨 Circuit Breaker"]
    C -->|Still failing| F

    style A fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
    style B fill:#FFEB3B,stroke:#F9A825,stroke-width:3px,color:#000
    style C fill:#00E5FF,stroke:#00B8D4,stroke-width:3px,color:#000
    style D fill:#EA80FC,stroke:#D500F9,stroke-width:3px,color:#000
    style E fill:#FFD600,stroke:#FFC107,stroke-width:3px,color:#000
    style F fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
```

## Cost Control Model

```json
{
  "cost_tracking": {
    "per_request": {
      "incident_id": "INC-2026-001234",
      "total_tokens": 14200,
      "total_cost_usd": 0.087,
      "breakdown": {
        "summarizer": { "tokens": 3200, "cost": 0.020 },
        "noise_agent": { "tokens": 2800, "cost": 0.017 },
        "impact_agent": { "tokens": 4100, "cost": 0.025 },
        "mitigation_agent": { "tokens": 2500, "cost": 0.015 },
        "evaluator": { "tokens": 1600, "cost": 0.010 }
      }
    },
    "budget": {
      "daily_limit_usd": 500,
      "monthly_limit_usd": 10000,
      "alert_threshold": 0.80
    }
  }
}
```

## Azure Mapping

| Pillar | Azure Service |
|---|---|
| Telemetry & Logging | Azure Monitor + Application Insights |
| Alerting | Azure Monitor Alerts |
| Dashboard | Azure Workbooks / Grafana |
| Cost tracking | Custom metrics in Application Insights |
