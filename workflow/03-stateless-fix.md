# 03. The stateless fix

## Caption

Stateless Agent Delegates Retrieval to Service A. The agent worker holds zero in-memory retrieval cache, delegating evidence retrieval via HTTP to an independent microservice backed by Vertex AI RAG.

## Mermaid

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TD
    A["<div style='min-width: 300px;'><b>Reader Request</b><br/>User submits question via API / UI</div>"] --> B["<div style='min-width: 300px;'><b>Stateless Agent Worker</b><br/>Cloud Run instance holds zero in-memory cache</div>"]

    B -->|"1. HTTP POST /retrieve"| C["<div style='min-width: 280px;'><b>HTTP Retrieval Service</b><br/>FastAPI Service A looks up policy evidence</div>"]
    C -->|"2. rag.retrieval_query()"| D["<div style='min-width: 280px;'><b>Vertex AI RAG Corpus</b><br/>Shared external knowledge base</div>"]
    D -->|"3. Grounded contexts"| C
    C -->|"4. JSON evidence payload"| B

    B -->|"5. Formulate final answer"| E["<div style='min-width: 280px;'><b>Grounded Response</b><br/>Accurate policy answer returned to reader</div>"]

    classDef req fill:#F3F4F6,stroke:#4B5563,color:#000000,stroke-width:1.5px
    classDef agent fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef srv fill:#EBF5FF,stroke:#2563EB,color:#000000,stroke-width:1.5px
    classDef rag fill:#FEF9C3,stroke:#CA8A04,color:#000000,stroke-width:1.5px
    classDef resp fill:#DCFCE7,stroke:#15803D,color:#000000,stroke-width:1.5px

    class A req
    class B agent
    class C srv
    class D rag
    class E resp
```

## What the reader should notice

- The agent worker keeps no mutable retrieval state between requests.
- Every request performs retrieval through the same external path.
- Consistency comes from shared external infrastructure, not from local memory.
- Any Cloud Run instance can now answer the request correctly.
