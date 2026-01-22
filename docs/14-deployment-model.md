# 🚀 Deployment Model

> **Purpose**: Multi-environment deployment strategy ensuring environment parity and safe progressive rollout.

---

## Environments

| Environment | Purpose | LLM Endpoint | Data |
|---|---|---|---|
| **Dev** | Development and experimentation | GPT-4o (cost-optimized) | Synthetic incidents |
| **Staging** | Pre-production validation | GPT-5.2 (same as prod) | Anonymized real incidents |
| **Prod** | Live production | GPT-5.2 (production SKU) | Real incidents |

## Overlays Architecture

The Deployment Model connects to both Input Layer and Output Layer via **Overlays** connections:

- **Input Overlay**: Controls which data sources are active (dev uses synthetic, prod uses real)
- **Output Overlay**: Controls which delivery channels are active (dev uses logging only, prod uses all channels)

## Deployment Strategy

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#AAAAAA'}}}%%
graph LR
    A["💻 Push"] --> B["⚙️ CI"]
    B --> C["🧪 Dev"]
    C --> D["🔬 Staging"]
    D --> E{"✅ Gate?"}
    E -->|Yes| F["🚀 Prod"]
    E -->|No| G["🚫 Block"]

    style A fill:#00E5FF,stroke:#00B8D4,stroke-width:3px,color:#000
    style B fill:#FFEB3B,stroke:#F9A825,stroke-width:3px,color:#000
    style C fill:#00FF88,stroke:#00E676,stroke-width:3px,color:#000
    style D fill:#EA80FC,stroke:#D500F9,stroke-width:3px,color:#000
    style E fill:#FFD600,stroke:#FFC107,stroke-width:3px,color:#000
    style F fill:#B2FF59,stroke:#76FF03,stroke-width:3px,color:#000
    style G fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
```

## Deployment Toolchain

| Tool | Purpose |
|---|---|
| **GitHub Actions** | CI/CD pipeline automation |
| **Azure Developer CLI (`azd`)** | Foundry agent deployment, environment provisioning |
| **Terraform** | Multi-environment infrastructure management |
| **Bicep** | Azure-native resource templates |
| **Docker + ACR** | Hosted agent packaging (Docker → Azure Container Registry → Foundry) |
