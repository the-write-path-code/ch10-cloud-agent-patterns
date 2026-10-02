# 04. FastAPI as the retrieval boundary

## Caption

FastAPI Retrieval Service as the Stable Boundary. Service B (policy-agent) delegates retrieval to Service A (policy-retrieval) via HTTP POST /retrieve. Service A coordinates directly with the Vertex AI RAG Engine, keeping agent instances stateless.

## Mermaid

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "18px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TD
    A["<div style='min-width: 550px;'><b>Service B: policy-agent</b><br/>Cloud Run agent worker receives question and manages user conversation session</div>"]
    B["<div style='min-width: 550px;'><b>Service A: policy-retrieval</b><br/>FastAPI microservice validates query and coordinates external RAG retrieval</div>"]
    C["<div style='min-width: 550px;'><b>Vertex AI RAG Engine</b><br/>Centralized vector index and document repository storing enterprise knowledge</div>"]

    A -->|"<b>1. POST /retrieve</b> { query: '...' }"| B
    B -->|"<b>2. rag.retrieval_query()</b>"| C
    C -->|"<b>3. Retrieved contexts</b> (chunks & scores)"| B
    B -->|"<b>4. Returns JSON contexts</b>"| A

    classDef b fill:#EFF6FF,stroke:#2563EB,stroke-width:1.5px,color:#000000
    classDef a fill:#EDE9FE,stroke:#7C3AED,stroke-width:1.5px,color:#000000
    classDef c fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#000000

    class A b
    class B a
    class C c
```

## What the reader should notice

- Service A is the place that knows how to find relevant documents.
- Service B is the place that knows how to answer the user.
- The agent no longer depends on remembering earlier retrievals in local memory.
- Every Cloud Run worker can ask the same retrieval service for the same evidence.
