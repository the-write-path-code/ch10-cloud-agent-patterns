# Chapter 10 workflow diagrams

This appendix-style file gathers every Mermaid diagram for Chapter 10 in one
place, in chapter order. Use it for editorial review, figure planning, or
manuscript handoff.

---

## 10.1 Local success, cloud failure

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

**Figure intent:** Show that local success can hide the loss of in-process state
once requests begin landing on separate cloud containers.

---

## 10.2 Concurrent workers do not share memory

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

**Figure intent:** Show that concurrent requests multiply isolated worker
memories rather than creating a shared stateful system.

---

## 10.3 The stateless fix

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

**Figure intent:** Show that the fix is architectural. Retrieval becomes an
external shared capability instead of a local worker detail.

---

## 10.4 Broken vs fixed comparison

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

**Figure intent:** Provide one summary figure for the chapter's central
architectural contrast.

---

## 10.5 FastAPI as the retrieval boundary

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

**Figure intent:** Show that the agent no longer tries to remember knowledge in
its own process. Instead, it asks a dedicated retrieval service for document
backing every time it needs evidence.

---

## 10.6 Deployment environment configuration

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TD
    ENV["<div style='min-width: 640px;'><b>Deployment Configuration Source</b><br/>Environment Variables (<code>.env</code>) or <code>gcloud run deploy</code> Flags</div>"]

    subgraph SVA ["Service A Environment (policy-retrieval)"]
        direction TB
        A1["<div style='min-width: 270px;'><b>RAG_CORPUS</b><br/>Vertex AI corpus resource name</div>"]
        A2["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_PROJECT</b><br/>GCP project identifier</div>"]
        A3["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_LOCATION</b><br/>GCP deployment region</div>"]
        A1 --- A2 --- A3
    end

    subgraph SVB ["Service B Environment (policy-agent)"]
        direction TB
        B1["<div style='min-width: 270px;'><b>RETRIEVAL_SERVICE_URL</b><br/>Service A Cloud Run endpoint</div>"]
        B2["<div style='min-width: 270px;'><b>AGENT_MODEL</b><br/>Gemini model identifier</div>"]
        B3["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_PROJECT</b><br/>GCP project identifier</div>"]
        B4["<div style='min-width: 270px;'><b>GOOGLE_CLOUD_LOCATION</b><br/>GCP deployment region</div>"]
        B1 --- B2 --- B3 --- B4
    end

    ENV ==> SVA
    ENV ==> SVB

    classDef env fill:#F3F4F6,stroke:#4B5563,stroke-width:1.5px,color:#000000
    classDef sva fill:#EDE9FE,stroke:#7C3AED,stroke-width:1.5px,color:#000000
    classDef svb fill:#EFF6FF,stroke:#2563EB,stroke-width:1.5px,color:#000000
    classDef subg_a fill:#FFFFFF,stroke:#7C3AED,stroke-width:1.5px,color:#000000
    classDef subg_b fill:#FFFFFF,stroke:#2563EB,stroke-width:1.5px,color:#000000

    class ENV env
    class SVA subg_a
    class SVB subg_b
    class A1,A2,A3 sva
    class B1,B2,B3,B4 svb
```

**Figure intent:** Show that service discovery and cloud configuration are set
at deployment time rather than embedded in source code.
