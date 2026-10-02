# 01. Local success, cloud failure

## Caption

Request 1 arrives at Container A. Container A fills its in-memory cache. Request 2 arrives at Container B, which has a cold cache. Container B has no access to Container A's heap. This locally-successful pattern of cache accumulation has no analog in a multi-instance deployment.

## Mermaid

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TD
    A["<div style='min-width: 320px;'><b>Request 1 from Reader</b><br/>Initial query arrives at service</div>"] --> B["<div style='min-width: 320px;'><b>Cloud Run Container A</b><br/>Active worker process loads heap</div>"]
    B --> C["<div style='min-width: 320px;'><b>Retrieval Result Cached</b><br/>In-process heap stores RAG response</div>"]

    D["<div style='min-width: 320px;'><b>Request 2 from Reader</b><br/>Follow-up query routed by load balancer</div>"] --> E["<div style='min-width: 320px;'><b>Cloud Run Container B</b><br/>Independent worker process instance</div>"]
    E --> F["<div style='min-width: 320px;'><b>Process Memory Starts Empty</b><br/>Cold heap; earlier cache unreachable</div>"]

    C -.->|"<b>No Shared Memory</b><br/>Heap state is isolated per container"| F

    classDef req fill:#F3F4F6,stroke:#4B5563,color:#000000,stroke-width:1.5px
    classDef worker fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef warm fill:#DCFCE7,stroke:#15803D,color:#000000,stroke-width:1.5px
    classDef cold fill:#FEE2E2,stroke:#DC2626,color:#000000,stroke-width:1.5px

    class A,D req
    class B,E worker
    class C warm
    class F cold
```

## What the reader should notice

- Local success can hide a cloud deployment flaw.
- Each Cloud Run container owns its own process memory.
- The second request does not inherit the first request's cached state.
- The failure comes from the deployment model, not from Python itself.
