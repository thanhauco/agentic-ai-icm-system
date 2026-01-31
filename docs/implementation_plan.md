# Agentic AI ICM System — Master Implementation Plan

> **Platform**: Azure AI Foundry Agent Service + Microsoft Agent Framework (MAF)
> **Language**: Python 3.12+ | **LLM**: Azure OpenAI GPT-5.2 via Foundry

---

## Phase 0: Project Foundation (Week 1)

> Scaffold project, configure Azure AI Foundry client, and validate local demo mode.

| Deliverable | Details |
|---|---|
| `pyproject.toml` | PEP 621 config with deps: `azure-ai-projects`, `azure-identity`, `pydantic`, `presidio-analyzer`, `redis`, `opentelemetry-api` |
| `.env.example` | `PROJECT_ENDPOINT`, `MODEL_DEPLOYMENT_NAME`, `REDIS_URL`, `COSMOS_CONNECTION_STRING` |
| `src/icm_agents/config.py` | Pydantic Settings with demo-mode fallback |
| `src/icm_agents/client.py` | `get_foundry_client() → AIProjectClient` factory |
| Pydantic data contracts | `IncidentPayload`, `ContextualizedPayload`, `PipelineEnvelope`, `CategorizedResults`, `RankedResultSet`, `DelegationRequest`, `ValidatedInput`, `FilteredTimeline`, `ImpactSummary`, `MitigationWorkflow`, `EvaluationResult` |
| `BaseAgent` abstraction | ABC using Foundry SDK: `create_agent()`, `threads.create()`, `runs.create_and_process()` |
| Stub agents | `SupervisorAgent`, `NoiseAgent`, `ImpactAgent`, `MitigationAgent` |
| `PipelineRunner` | End-to-end orchestration with `--demo` flag (mock data, no Azure) |
| `BaseGuardrail` | ABC with `validate() → ValidatedInput \| RejectionResult` |
| Tests | `test_models.py`, `test_config.py`, `test_pipeline_demo.py` |

**Verification**: `pip install -e ".[dev]"` → `pytest tests/ -v` → `python -m icm_agents --demo`

---

## Phase 1: Input Layer + Context Manager (Week 2)

> Ingest raw incident data and manage session context.

| Deliverable | Details |
|---|---|
| Input adapters | `EmailAdapter`, `ChatAdapter`, `LogAdapter` — parse raw formats into `IncidentPayload` |
| PII pre-scan | Presidio integration — detect and tag PII fields at ingestion (redact before downstream) |
| Context Manager | Session state init, `session_id` assignment, stage tracking |
| Redis integration | Azure Cache for Redis — short-term session state (TTL: 24h) |
| Data normalization | Raw heterogeneous text → unified `ContextualizedPayload` schema |

**Azure Services**: Azure Cache for Redis, Microsoft Presidio

---

## Phase 2: Tool Router + Summarizer Agent (Week 3)

> Route data and perform first-pass AI categorization.

| Deliverable | Details |
|---|---|
| Tool Router | Agent/tool registry with deterministic dispatch (no LLM in router) |
| Summarizer Agent | **Foundry Hosted Agent** — GPT-5.2 via Foundry for normalize, categorize, cluster |
| MAF hosting adapter | `azure-ai-agentserver-agentframework` for local testing on `localhost:8088` |
| Output contract | `CategorizedResults` with `noise_signals[]`, `impact_signals[]`, `mitigation_signals[]` |
| Bidirectional Memory | Summarizer reads from and writes to Memory Manager + Vector Store |

**Azure Services**: Azure AI Foundry Agent Service, Azure OpenAI

---

## Phase 3: Memory Manager + Vector Store (Week 4)

> Persistent memory and semantic search.

| Deliverable | Details |
|---|---|
| Memory Manager | Two-tier: Redis (short-term session) + Cosmos DB (long-term cross-incident) |
| Stage state machine | `ingested → context_ready → summarizing → categorized → delegating → processing_* → evaluated → output_ready` |
| Checkpoint/rollback | Azure Blob Storage snapshots (TTL: 7 days) |
| Vector Store | Azure AI Search with vector profile, `text-embedding-3-large` (1536-dim) |
| Embedding pipeline | Text → Embeddings → Vector Index → Cosine Similarity → Top-K retrieval |
| Indexed content | Past incidents, runbooks, mitigation playbooks, categorized signals |
| Hybrid search | Vector similarity + BM25 keyword + re-rank |

**Azure Services**: Azure Cache for Redis, Azure Cosmos DB, Azure Blob Storage, Azure AI Search, Azure OpenAI Embeddings

---

## Phase 4: Aggregated Results + Supervisor Agent (Week 5)

> Central orchestration and result ranking.

| Deliverable | Details |
|---|---|
| Aggregated Results Layer | Group by category, re-rank by composite score, filter low-confidence |
| Re-ranking algorithm | `score = (0.35 × confidence) + (0.30 × severity) + (0.20 × recency_decay) + (0.15 × source_reliability)` |
| Supervisor Agent | **MAF Workflow Orchestration** with declarative YAML |
| Orchestration patterns | Concurrent (default: all 3 agents in parallel) or Sequential (Impact → Mitigation when dependent) |
| Connected agents | Foundry connected agents for cross-agent coordination |
| Delegation routing | Schema validation → classify into workflow domains → route through guardrails |

**Azure Services**: Azure AI Foundry Agent Service (workflows, connected agents)

---

## Phase 5: Guardrails & Gatekeeping (Week 6)

> Safety gates before each workflow agent.

| Deliverable | Details |
|---|---|
| `BaseGuardrail` framework | Reusable validation engine with configurable rules |
| Noise Guardrail | Min signal count, confidence threshold, source diversity check |
| Impact Guardrail | Schema completeness, severity field validation, scope bounds |
| Mitigation Guardrail | Safety checks, authorization gates, runbook reference validation |
| Rejection flow | On reject → structured error → halt pipeline gracefully |
| Content Safety integration | Azure AI Content Safety screens inputs before reaching agents |

**Azure Services**: Azure AI Content Safety

---

## Phase 6: Workflow Agents (Weeks 7–9)

> The three core AI agents — all deployed as **Foundry Hosted Agents**.

### WF-9 Noise Agent (Week 7)
| Deliverable | Details |
|---|---|
| Multi-pass filtering | Deduplication → relevance scoring → false positive check → chronological ordering |
| Output | `FilteredTimeline` — noise-free chronological incident narrative |
| Custom tools | Foundry tool definitions for signal classification |

### WF-10 Impact Agent (Week 8)
| Deliverable | Details |
|---|---|
| Impact analysis | Customer impact, service impact, SLA evaluation, blast radius |
| Severity scoring | Composite: `(0.30 × customer) + (0.25 × service) + (0.25 × sla) + (0.20 × blast_radius)` |
| Output | `ImpactSummary` — factual report with severity scoring |

### WF-25 Mitigation Agent (Week 9)
| Deliverable | Details |
|---|---|
| Action plan generation | Retrieve runbooks from Vector Store, generate step-by-step plan |
| Azure Functions tools | Foundry tool bindings to Azure Functions for real-world actions |
| Output | `MitigationWorkflow` — action plan (bypasses Evaluator, direct to Output) |

**Azure Services**: Azure AI Foundry Hosted Agents, Azure Functions

---

## Phase 7: Evaluator + Output Layer (Week 10)

> Quality assessment and final output delivery.

| Deliverable | Details |
|---|---|
| Evaluator Agent | Scores: factual accuracy, completeness, hallucination check, consistency, format compliance |
| Composite scoring | Weighted average → `approved` / `revision_needed` / `rejected` verdict |
| Feedback loop | On `revision_needed` → re-invoke source agent with feedback |
| Output Layer | Formatting + delivery (no AI) — JSON, HTML/Markdown, Email, Teams/Slack |
| Delivery channels | REST API, ICM Portal, Email notification, Teams Adaptive Card |

---

## Phase 8: Safety & Governance (Week 11)

> Cross-cutting security wrapping the entire pipeline.

| Deliverable | Details |
|---|---|
| AI Governance | Data classification, PII redaction (Presidio), data masking (configurable), encryption (AES-256 + TLS 1.3) |
| Responsible AI | Prompt injection defense, jailbreak detection, content safety filters, bias auditing |
| Foundry content filters | Integrated with Azure AI Foundry built-in content filters |
| Key management | Azure Key Vault for secrets, HSM-backed |

**Azure Services**: Microsoft Presidio, Azure AI Content Safety, Azure Key Vault

---

## Phase 9: Observability & Control (Week 12)

> Production-readiness monitoring and cost management.

| Deliverable | Details |
|---|---|
| Error Handling | Structured error types, retry with exponential backoff, circuit breaker, dead-letter queue |
| Run History | Full audit trail — every agent invocation with input/output, timestamps, decisions |
| Health Check | `/health` endpoint — module availability, API response times, agent readiness |
| Cost Control | Token usage per request/agent/incident, daily/monthly budget limits, alert threshold (80%) |
| Foundry tracing | Application Insights integration via Foundry Agent observability |

**Azure Services**: Azure Monitor, Application Insights, Azure Workbooks

---

## Phase 10: Integration, Testing & Deployment (Weeks 13–14)

> End-to-end validation and production deployment.

| Deliverable | Details |
|---|---|
| E2E integration tests | Full pipeline: Input → Context → Summarizer → Supervisor → Agents → Evaluator → Output |
| Performance testing | Load tests with synthetic incidents, latency benchmarking |
| Hosted agent packaging | Docker containers via MAF hosting adapter, local test → push to ACR |
| Deployment | Foundry Hosted Agents (primary) + Azure Container Apps (auxiliary microservices) |
| CI/CD | GitHub Actions + `azd` (Azure Developer CLI) for Foundry agent deployment |
| IaC | Terraform + Bicep for multi-environment infrastructure |
| Environments | Dev (GPT-4o, synthetic data) → Staging (GPT-5.2, anonymized) → Prod (GPT-5.2, live) |

**Azure Services**: Azure Container Registry, Foundry Hosted Agents, Azure Container Apps, GitHub Actions

---

## Project Structure

```
agentic-ai-icm-system/
├── src/icm_agents/
│   ├── __init__.py
│   ├── config.py                    # Pydantic Settings + demo mode
│   ├── client.py                    # AIProjectClient factory
│   ├── main.py                      # CLI entry point
│   ├── models/                      # Pydantic data contracts
│   │   ├── incident.py              # IncidentPayload, ContextualizedPayload, PipelineEnvelope
│   │   ├── results.py               # CategorizedResults, RankedResultSet, DelegationRequest
│   │   └── outputs.py               # FilteredTimeline, ImpactSummary, MitigationWorkflow
│   ├── agents/                      # Foundry Hosted Agents
│   │   ├── base.py                  # BaseAgent (Foundry SDK lifecycle)
│   │   ├── supervisor.py            # MAF Workflow Orchestration
│   │   ├── summarizer.py            # Normalize & Categorize
│   │   ├── noise_agent.py           # WF-9 Filter Noise
│   │   ├── impact_agent.py          # WF-10 Assess Impact
│   │   ├── mitigation_agent.py      # WF-25 Orchestrate Mitigation
│   │   └── evaluator.py             # Quality Assessment
│   ├── core/                        # Non-agent modules
│   │   ├── input_layer.py           # Input adapters (Email, Chat, Log)
│   │   ├── context_manager.py       # Session state (Redis)
│   │   ├── tool_router.py           # Deterministic dispatch
│   │   ├── memory_manager.py        # Short/Long-term memory
│   │   ├── vector_store.py          # Azure AI Search integration
│   │   └── aggregated_results.py    # Re-ranker
│   ├── guardrails/                  # Input validation gates
│   │   ├── base.py                  # BaseGuardrail ABC
│   │   ├── noise_guardrail.py
│   │   ├── impact_guardrail.py
│   │   └── mitigation_guardrail.py
│   ├── output/                      # Output formatters
│   │   ├── timeline.py
│   │   ├── impact_report.py
│   │   └── action_plan.py
│   ├── governance/                  # Safety & Governance
│   │   ├── pii_masking.py           # Presidio integration
│   │   ├── prompt_safety.py         # Injection defense
│   │   └── content_filter.py        # Azure AI Content Safety
│   ├── observability/               # Monitoring & Cost
│   │   ├── error_handler.py
│   │   ├── run_history.py
│   │   ├── health_check.py
│   │   └── cost_control.py
│   └── pipeline/
│       └── runner.py                # PipelineRunner (demo + live)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── deploy/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── terraform/
│   └── bicep/
├── workflows/                       # MAF declarative YAML workflows
│   └── supervisor_workflow.yaml
├── pyproject.toml
├── .env.example
├── .gitignore
└── README.md
```

---

## Technology Stack

| Layer | Technology |
|---|---|
| **Language** | Python 3.12+ |
| **LLM** | Azure OpenAI GPT-5.2 (via Foundry) |
| **Embeddings** | Azure OpenAI `text-embedding-3-large` |
| **Agent Platform** | Azure AI Foundry Agent Service |
| **Agent Framework** | Microsoft Agent Framework (MAF) |
| **Agent Hosting** | Foundry Hosted Agents (containerized, auto-scaling) |
| **Interoperability** | A2A Protocol + MCP |
| **Vector Search** | Azure AI Search |
| **Short-term Memory** | Azure Cache for Redis |
| **Long-term Memory** | Azure Cosmos DB (NoSQL) |
| **PII Detection** | Microsoft Presidio |
| **Content Safety** | Azure AI Content Safety |
| **Observability** | Azure Monitor + Application Insights |
| **Secrets** | Azure Key Vault |
| **Compute** | Foundry Hosted Agents + Azure Container Apps |
| **CI/CD** | GitHub Actions + Azure Developer CLI (`azd`) |
| **IaC** | Terraform + Bicep |

---

## Risk Mitigations

| Risk | Mitigation |
|---|---|
| Foundry Hosted Agents in preview | Fallback to Azure Container Apps if stability issues arise |
| LLM hallucination in agent outputs | Evaluator with multi-dimensional scoring + guardrails before every agent |
| Token cost overruns | Cost Control module with per-incident budgets + daily/monthly limits |
| PII leakage | Defense in depth: Presidio at ingestion + content filters at every stage |
| Agent response latency | Concurrent orchestration by default; circuit breaker for timeouts |
