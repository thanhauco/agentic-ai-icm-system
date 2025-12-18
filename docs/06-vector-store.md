# 🔍 Vector Store

> **Purpose**: Stores and retrieves semantic embeddings for similarity search across incident data, enabling knowledge retrieval and contextual enrichment.

---

## What It Does

The Vector Store maintains a searchable index of **embedding vectors** generated from past incidents, runbooks, and mitigation playbooks. It enables agents to find semantically similar content rather than relying on exact keyword matches.

## Embedding Pipeline

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#AAAAAA'}}}%%
graph LR
    A["📄 Text"] --> B["🤖 Embeddings"]
    B --> C["🔢 1536-dim"]
    C --> D["🗄️ AI Search"]
    D --> E["🔍 Cosine Sim"]
    E --> F["🎯 Top-K"]

    style A fill:#FFD600,stroke:#FFC107,stroke-width:3px,color:#000
    style B fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
    style C fill:#00E5FF,stroke:#00B8D4,stroke-width:3px,color:#000
    style D fill:#EA80FC,stroke:#D500F9,stroke-width:3px,color:#000
    style E fill:#FFEB3B,stroke:#F9A825,stroke-width:3px,color:#000
    style F fill:#B2FF59,stroke:#76FF03,stroke-width:3px,color:#000
```

## Key Operations

| Operation | Description |
|---|---|
| `index(doc_id, text, metadata)` | Generate embedding and upsert into vector index |
| `search(query_text, top_k, filters)` | Embed query, find top-K similar documents by cosine similarity |
| `search_with_rerank(query, top_k)` | Hybrid search: vector similarity + BM25 keyword, then re-rank |
| `delete(doc_id)` | Remove document from index |
| `batch_index(documents[])` | Bulk indexing for historical data backfill |

## Indexed Content Types

| Content Type | Fields Indexed | Metadata |
|---|---|---|
| Past incidents | Summary, root cause, resolution | `incident_id`, `severity`, `service`, `date` |
| Runbooks | Steps, prerequisites, commands | `runbook_id`, `service`, `version` |
| Mitigation playbooks | Action sequences, rollback steps | `playbook_id`, `scenario_type` |
| Categorized signals | Noise/Impact/Mitigation summaries | `session_id`, `category`, `confidence` |

## Azure Mapping

| Component | Azure Service |
|---|---|
| Vector index | Azure AI Search (vector search profile) |
| Embedding model | Azure OpenAI `text-embedding-3-large` |
| Document store | Azure Cosmos DB (source docs) |
