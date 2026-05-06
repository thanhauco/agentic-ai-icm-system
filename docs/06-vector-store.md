# 🔍 Vector Store — Deep Dive

> **Purpose**: Semantic search layer using Azure AI Search + Azure OpenAI embeddings. Stores and retrieves incident embeddings for knowledge retrieval, hybrid search (vector + BM25), and re-ranking.

---

## Architecture Overview

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#AAAAAA'}}}%%
graph LR
 A["Text"] --> B["text-embedding-3-large<br>(Azure OpenAI)"]
 B --> C["1536-dim vector"]
 C --> D["Azure AI Search<br>(Vector Index)"]
 D --> E["Hybrid: Vector + BM25"]
 E --> F["Semantic Ranker"]
 F --> G["Top-K Results"]

 style A fill:#FFD600,stroke:#FFC107,stroke-width:3px,color:#000
 style B fill:#FF1744,stroke:#F50057,stroke-width:3px,color:#fff
 style C fill:#00E5FF,stroke:#00B8D4,stroke-width:3px,color:#000
 style D fill:#EA80FC,stroke:#D500F9,stroke-width:3px,color:#000
 style E fill:#FFEB3B,stroke:#F9A825,stroke-width:3px,color:#000
 style F fill:#7C4DFF,stroke:#651FFF,stroke-width:3px,color:#fff
 style G fill:#B2FF59,stroke:#76FF03,stroke-width:3px,color:#000
```

---

## Azure Service Mapping

| Component | Azure Service | Config |
|---|---|---|
| Vector index | **Azure AI Search** | Standard S1, vector search profile, HNSW algorithm |
| Embedding model | **Azure OpenAI** `text-embedding-3-large` | 1536 dimensions, deployed via Foundry model catalog |
| Source document store | **Azure Cosmos DB** | NoSQL, `incidents` and `runbooks` containers |
| Foundry integration | **Azure AI Search as Foundry Knowledge Tool** | Registered in Foundry Agent Service |

---

## Implementation

```python
# src/icm_agents/core/vector_store.py

import os
from typing import Optional
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex, SearchField, SearchFieldDataType,
    VectorSearch, HnswAlgorithmConfiguration, VectorSearchProfile,
    SemanticConfiguration, SemanticSearch, SemanticPrioritizedFields, SemanticField,
)
from azure.search.documents.models import VectorizedQuery
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI
from opentelemetry import trace

tracer = trace.get_tracer("icm.vector_store")

INDEX_NAME = "icm-incidents"
EMBEDDING_MODEL = "text-embedding-3-large"
EMBEDDING_DIM = 1536


class VectorStore:
    """
    Azure AI Search vector store with hybrid search (vector + BM25)
    and semantic re-ranking. Uses Azure OpenAI for embedding generation.
    
    Can also be registered as a Foundry Knowledge Tool for agent access.
    """

    def __init__(self):
        credential = DefaultAzureCredential()
        self.search_client = SearchClient(
            endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
            index_name=INDEX_NAME,
            credential=credential,
        )
        self.index_client = SearchIndexClient(
            endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
            credential=credential,
        )
        self.openai = AzureOpenAI(
            azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
            api_version="2024-12-01-preview",
            azure_ad_token_provider=credential.get_token(
                "https://cognitiveservices.azure.com/.default"
            ).token,
        )

    def create_index(self) -> None:
        """Create the AI Search index with vector and semantic config."""
        index = SearchIndex(
            name=INDEX_NAME,
            fields=[
                SearchField(name="id", type=SearchFieldDataType.String, key=True),
                SearchField(name="incident_id", type=SearchFieldDataType.String, filterable=True),
                SearchField(name="category", type=SearchFieldDataType.String, filterable=True),
                SearchField(name="severity", type=SearchFieldDataType.String, filterable=True),
                SearchField(name="content", type=SearchFieldDataType.String, searchable=True),
                SearchField(name="summary", type=SearchFieldDataType.String, searchable=True),
                SearchField(
                    name="content_vector",
                    type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
                    searchable=True,
                    vector_search_dimensions=EMBEDDING_DIM,
                    vector_search_profile_name="icm-vector-profile",
                ),
                SearchField(name="source_type", type=SearchFieldDataType.String, filterable=True),
                SearchField(name="created_at", type=SearchFieldDataType.DateTimeOffset, sortable=True),
            ],
            vector_search=VectorSearch(
                algorithms=[
                    HnswAlgorithmConfiguration(
                        name="icm-hnsw",
                        parameters={"m": 4, "efConstruction": 400, "efSearch": 500, "metric": "cosine"},
                    ),
                ],
                profiles=[
                    VectorSearchProfile(name="icm-vector-profile", algorithm_configuration_name="icm-hnsw"),
                ],
            ),
            semantic_search=SemanticSearch(
                configurations=[
                    SemanticConfiguration(
                        name="icm-semantic",
                        prioritized_fields=SemanticPrioritizedFields(
                            content_fields=[SemanticField(field_name="content")],
                            title_field=SemanticField(field_name="summary"),
                        ),
                    ),
                ],
            ),
        )
        self.index_client.create_or_update_index(index)

    def _embed(self, text: str) -> list[float]:
        """Generate embedding via Azure OpenAI."""
        response = self.openai.embeddings.create(
            model=EMBEDDING_MODEL,
            input=text,
        )
        return response.data[0].embedding

    def index_document(self, doc_id: str, content: str, metadata: dict) -> None:
        """Generate embedding and upsert into AI Search index."""
        with tracer.start_as_current_span("vector.index"):
            vector = self._embed(content)
            document = {
                "id": doc_id,
                "content": content,
                "content_vector": vector,
                **metadata,
            }
            self.search_client.upload_documents(documents=[document])

    def search(self, query_text: str, top_k: int = 5, filters: Optional[str] = None) -> list[dict]:
        """
        Hybrid search: vector similarity + BM25 keyword + semantic re-rank.
        This is the recommended Azure AI Search pattern for best recall+precision.
        """
        with tracer.start_as_current_span("vector.search") as span:
            query_vector = self._embed(query_text)
            span.set_attribute("query_length", len(query_text))

            results = self.search_client.search(
                search_text=query_text,                          # BM25 keyword
                vector_queries=[
                    VectorizedQuery(
                        vector=query_vector,                     # Vector similarity
                        k_nearest_neighbors=top_k * 2,
                        fields="content_vector",
                    ),
                ],
                filter=filters,                                  # OData filter
                query_type="semantic",                           # Semantic re-rank
                semantic_configuration_name="icm-semantic",
                top=top_k,
            )

            hits = []
            for r in results:
                hits.append({
                    "id": r["id"],
                    "incident_id": r.get("incident_id"),
                    "content": r.get("content", "")[:500],
                    "summary": r.get("summary", ""),
                    "score": r["@search.score"],
                    "reranker_score": r.get("@search.reranker_score"),
                })

            span.set_attribute("result_count", len(hits))
            return hits

    def batch_index(self, documents: list[dict]) -> None:
        """Bulk indexing for historical data backfill."""
        for doc in documents:
            doc["content_vector"] = self._embed(doc["content"])
        self.search_client.upload_documents(documents=documents)

    def delete(self, doc_id: str) -> None:
        """Remove document from index."""
        self.search_client.delete_documents(documents=[{"id": doc_id}])
```

---

## Foundry Knowledge Tool Registration

Azure AI Search can be registered as a **Foundry Knowledge Tool**, allowing agents to search the index directly:

```python
from azure.ai.projects import AIProjectClient
from azure.ai.agents.models import AzureAISearchTool

# Register Azure AI Search as a Foundry knowledge tool
search_tool = AzureAISearchTool(
    index_connection_id="<ai-search-connection-id>",  # Foundry connection
    index_name="icm-incidents",
    query_type="vector_semantic_hybrid",
    top_k=5,
)

# Attach to any Foundry agent
agent = client.agents.create_agent(
    model="gpt-5.2",
    name="summarizer-agent",
    tools=search_tool.definitions,
    tool_resources=search_tool.resources,
)
```

---

## Index Schema Summary

| Field | Type | Searchable | Filterable | Purpose |
|---|---|---|---|---|
| `id` | String | — | Key | Unique document ID |
| `incident_id` | String | — | ✅ | Filter by incident |
| `category` | String | — | ✅ | noise / impact / mitigation |
| `severity` | String | — | ✅ | sev0 – sev3 |
| `content` | String | ✅ (BM25) | — | Full text for keyword search |
| `content_vector` | Vector (1536) | ✅ (HNSW) | — | Cosine similarity search |
| `summary` | String | ✅ | — | Semantic re-rank title field |

---

## Environment Variables

```env
AZURE_SEARCH_ENDPOINT=https://icm-search.search.windows.net
AZURE_OPENAI_ENDPOINT=https://icm-openai.openai.azure.com
```
