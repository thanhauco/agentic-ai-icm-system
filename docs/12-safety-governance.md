# 🔐 Safety & Governance

> **Purpose**: Ensure all data processing complies with AI governance policies. Protect against data leaks, PII exposure, prompt injection attacks, and jailbreak attempts. This layer wraps the **entire pipeline**.

---

## Two Pillars

### 1. AI Governance

| Concern | Implementation |
|---|---|
| **Data Control** | All incident data classified by sensitivity level. Access control enforced per classification. |
| **PII Redaction** | Microsoft Presidio detects and masks PII (names, emails, phone numbers, IP addresses) at ingestion. Downstream agents never see raw PII. |
| **Data Masking** | Configurable masking rules: full redaction, partial masking (show last 4 digits), or tokenized replacement. |
| **Data Protection** | Encryption at rest (AES-256) and in transit (TLS 1.3). Azure Key Vault for secrets management. |

### PII Redaction Pipeline

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#AAAAAA'}}}%%
graph LR
    A["📄 Raw Text"] --> B["🔍 Analyzer"]
    B --> C["🔒 Anonymizer"]
    C --> D["✅ Redacted"]
    D --> E["🤖 Agents"]

    style A fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
    style B fill:#FFEB3B,stroke:#F9A825,stroke-width:3px,color:#000
    style C fill:#EA80FC,stroke:#D500F9,stroke-width:3px,color:#000
    style D fill:#B2FF59,stroke:#76FF03,stroke-width:3px,color:#000
    style E fill:#00E5FF,stroke:#00B8D4,stroke-width:3px,color:#000
```

### 2. Responsible AI

| Concern | Implementation |
|---|---|
| **Prompt Injection Defense** | Input sanitization layer strips known injection patterns. Delimiter-based prompt isolation. System prompts marked as privileged. |
| **Jailbreak Defense** | Output classification detects jailbreak attempts. Rate limiting on unusual input patterns. Canary tokens in system prompts. |
| **Content Safety** | Azure AI Content Safety screens inputs and outputs for harmful content categories. |
| **Bias Detection** | Periodic model output audits for decision bias across incident severity and service categories. |
