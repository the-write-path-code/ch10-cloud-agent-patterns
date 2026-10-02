# 07. Broken architecture vs fixed architecture

## Caption

Broken architecture vs. fixed architecture. On the left, workers attempt to retain retrieval state locally in container RAM, causing cache loss across autoscaled instances. On the right, workers remain strictly stateless and delegate document lookup to a centralized FastAPI service backed by Vertex AI RAG.

## Mermaid

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart LR
    subgraph BROKEN ["Broken: Stateful In-Process Memory"]
        direction TB
        B1["<div style='min-width: 250px;'><b>Request 1 ➔ Container A</b><br/>Populates local heap cache</div>"]
        B2["<div style='min-width: 250px;'><b>Request 2 ➔ Container B</b><br/>Cold heap; earlier cache unreachable</div>"]
        B1 -.->|"<b>No Shared Memory</b><br/>Cache hit count = 0"| B2
    end

    subgraph FIXED ["Fixed: Shared Retrieval Service"]
        direction TB
        F1["<div style='min-width: 250px;'><b>Stateless Agent Worker</b><br/>Cloud Run holds zero memory state</div>"]
        F2["<div style='min-width: 250px;'><b>FastAPI Retrieval Service A</b><br/>Shared retrieval boundary</div>"]
        F3["<div style='min-width: 250px;'><b>Vertex AI RAG Corpus</b><br/>Centralized enterprise knowledge</div>"]
        F1 -->|"HTTP POST /retrieve"| F2
        F2 -->|"rag.retrieval_query"| F3
        F3 -->|"Contexts"| F2
        F2 -->|"JSON"| F1
    end

    BROKEN ~~~ FIXED

    classDef broken fill:#FEF2F2,stroke:#DC2626,color:#000000,stroke-width:1.5px
    classDef fixed fill:#F0FDF4,stroke:#16A34A,color:#000000,stroke-width:1.5px
    classDef bNode fill:#FFFFFF,stroke:#DC2626,color:#000000,stroke-width:1px
    classDef fNode fill:#FFFFFF,stroke:#16A34A,color:#000000,stroke-width:1px

    class BROKEN broken
    class FIXED fixed
    class B1,B2 bNode
    class F1,F2,F3 fNode
```

## What the reader should notice

- In the broken design, memory stays trapped inside one worker.
- In the fixed design, retrieval moves to a shared service boundary.
- Every worker can now reach the same retrieval system.
- The key change is architectural, not cosmetic.
