# 🛡️ Guardrails & Gatekeeping

> **Purpose**: Validate, sanitize, and gate data before it reaches each specialized workflow agent. Prevents malformed, unsafe, or irrelevant inputs from consuming expensive LLM resources.

---

## What It Does

Each workflow agent (Noise, Impact, Mitigation) has a dedicated guardrail in front of it. Guardrails are **non-AI, rule-based validation layers** that enforce input quality, safety, and compliance.

## Three Guardrails

| Guardrail | Protects | Key Checks |
|---|---|---|
| **Noise Guardrails & Gatekeeper** | WF-9 Noise Agent | Input is categorized, contains noise signals, not empty payload, PII masked |
| **Impact Guardrails & Gatekeeper** | WF-10 Impact Agent | Impact signals present, severity field populated, affected service identified, data freshness < 24h |
| **Mitigation Guardrails & Gatekeeper** | WF-25 Mitigation Agent | Mitigation signals present, not a duplicate request, authorized caller, action safety check |

## Guardrail Processing Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#AAAAAA'}}}%%
graph TD
    A["📥 From Supervisor"] --> B{"🔎 Schema?"}
    B -->|Fail| C["❌ REJECT"]
    B -->|Pass| D["🔒 Content Check"]
    D -->|Fail| C
    D -->|Pass| E["📏 Biz Rules"]
    E -->|Fail| C
    E -->|Pass| F["✅ APPROVE"]
    C --> G["📊 Log Rejection"]

    style A fill:#40C4FF,stroke:#0091EA,stroke-width:3px,color:#000
    style B fill:#FFEB3B,stroke:#F9A825,stroke-width:3px,color:#000
    style C fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
    style D fill:#EA80FC,stroke:#D500F9,stroke-width:3px,color:#000
    style E fill:#00E5FF,stroke:#00B8D4,stroke-width:3px,color:#000
    style F fill:#B2FF59,stroke:#76FF03,stroke-width:3px,color:#000
    style G fill:#7C4DFF,stroke:#651FFF,stroke-width:3px,color:#fff
```

## Guardrail Rejection Response

```json
{
  "status": "rejected",
  "guardrail": "impact_guardrail",
  "reason": "missing_field",
  "details": "Field 'affected_service' is required for impact assessment",
  "suggestion": "Re-run Summarizer with service extraction emphasis",
  "timestamp": "2026-02-10T17:05:00Z"
}
```
