# Agentic AI ICM System — Task Tracker

## Phase 1: Architecture & Design
- [x] Analyze architecture diagrams (panorama screenshots + SVG)
- [x] Create architecture analysis (`architecture_analysis.md`)
- [x] Write GenAI Architecture Design Document (`genai_architecture_design.md`)
- [x] Style all 16 Mermaid diagrams (dark-mode, high-contrast, emoji)
- [x] Update main overview flowchart to full-featured version with cross-cutting concerns
- [x] Migrate framework: Semantic Kernel → **MAF + Azure AI Foundry**
- [x] Write master implementation plan (`implementation_plan.md`) — all 11 phases
- [x] Update architecture analysis tech stack & folder structure
- [/] User review of all docs

---

## Phase 2: Implementation

### Phase 0 — Project Foundation (Week 1)
- [ ] `pyproject.toml` + `.env.example` + `.gitignore` + `README.md`
- [ ] `config.py` — Pydantic Settings with demo-mode fallback
- [ ] `client.py` — `AIProjectClient` factory (Foundry SDK)
- [ ] Pydantic data contracts (`models/incident.py`, `results.py`, `outputs.py`)
- [ ] `BaseAgent` abstraction (Foundry SDK lifecycle)
- [ ] Stub agents: Supervisor, Noise, Impact, Mitigation
- [ ] `PipelineRunner` with `--demo` flag
- [ ] `BaseGuardrail` ABC
- [ ] Tests: `test_models.py`, `test_config.py`, `test_pipeline_demo.py`

### Phase 1 — Input Layer + Context Manager (Week 2)
- [ ] Input adapters: Email, Chat, Log → `IncidentPayload`
- [ ] PII pre-scan with Presidio
- [ ] Context Manager — session state with Redis
- [ ] Data normalization pipeline

### Phase 2 — Tool Router + Summarizer Agent (Week 3)
- [ ] Tool Router — agent/tool registry, deterministic dispatch
- [ ] Summarizer Agent as Foundry Hosted Agent (GPT-5.2)
- [ ] MAF hosting adapter for local testing
- [ ] Bidirectional Memory/Vector Store integration

### Phase 3 — Memory Manager + Vector Store (Week 4)
- [ ] Memory Manager: Redis (short-term) + Cosmos DB (long-term)
- [ ] Stage state machine + checkpoint/rollback
- [ ] Vector Store: Azure AI Search + embedding pipeline
- [ ] Hybrid search: vector + BM25 + re-rank

### Phase 4 — Aggregated Results + Supervisor (Week 5)
- [ ] Aggregated Results Layer with re-ranking algorithm
- [ ] Supervisor Agent — MAF Workflow Orchestration (YAML)
- [ ] Concurrent + sequential orchestration patterns
- [ ] Connected agents for cross-agent coordination

### Phase 5 — Guardrails & Gatekeeping (Week 6)
- [ ] Noise Guardrail, Impact Guardrail, Mitigation Guardrail
- [ ] Rejection flow → structured error → halt gracefully
- [ ] Azure AI Content Safety integration

### Phase 6 — Workflow Agents (Weeks 7–9)
- [ ] WF-9 Noise Agent — multi-pass filter → `FilteredTimeline`
- [ ] WF-10 Impact Agent — severity scoring → `ImpactSummary`
- [ ] WF-25 Mitigation Agent — action plan → `MitigationWorkflow` (bypasses Evaluator)
- [ ] All deployed as Foundry Hosted Agents

### Phase 7 — Evaluator + Output Layer (Week 10)
- [ ] Evaluator: multi-dimensional scoring, feedback loop
- [ ] Output Layer: JSON, HTML, Email, Teams
- [ ] Delivery channel config per environment

### Phase 8 — Safety & Governance (Week 11)
- [ ] AI Governance: PII redaction, data masking, encryption
- [ ] Responsible AI: prompt injection, jailbreak defense, content safety
- [ ] Foundry content filter integration

### Phase 9 — Observability & Control (Week 12)
- [ ] Error Handling: retries, circuit breaker, dead-letter queue
- [ ] Run History: audit trail
- [ ] Health Check: `/health` endpoint
- [ ] Cost Control: token budgets, daily/monthly limits

### Phase 10 — Integration & Deployment (Weeks 13–14)
- [ ] E2E integration tests
- [ ] Performance/load testing
- [ ] Hosted agent packaging (Docker → ACR → Foundry)
- [ ] CI/CD: GitHub Actions + `azd`
- [ ] IaC: Terraform + Bicep
- [ ] Dev → Staging → Prod deployment
