# 02. Concurrent workers do not share memory

## Caption

Concurrent Workers Do Not Share Memory. Concurrent requests multiply isolated worker memories rather than creating a shared stateful system.

## Mermaid

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TD
    A["<div style='min-width: 320px;'><b>Concurrent Request A</b><br/>First client query arrives</div>"] --> B["<div style='min-width: 320px;'><b>Worker 1 in Container X</b><br/>Isolated OS process executes agent</div>"]
    B --> E["<div style='min-width: 320px;'><b>Local Cache Inside X</b><br/>Private heap stores retrieval state</div>"]

    C["<div style='min-width: 320px;'><b>Concurrent Request B</b><br/>Parallel client query arrives</div>"] --> D["<div style='min-width: 320px;'><b>Worker 2 in Container Y</b><br/>Isolated OS process executes agent</div>"]
    D --> F["<div style='min-width: 320px;'><b>Local Cache Inside Y</b><br/>Private heap has no access to X</div>"]

    E <-.->|"<b>No Shared Memory Boundary</b><br/>Concurrent workers cannot access peer heaps"| F

    classDef req fill:#F3F4F6,stroke:#4B5563,color:#000000,stroke-width:1.5px
    classDef worker fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef cache fill:#FEF9C3,stroke:#CA8A04,color:#000000,stroke-width:1.5px

    class A,C req
    class B,D worker
    class E,F cache
```

## What the reader should notice

- Concurrency increases the number of isolated worker memories.
- Cached retrievals and session state stay trapped inside one worker.
- A multi-worker backend is not a shared-memory system.
- This is why stateful agent servers become unreliable under real load.
